```markdown
---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue locally.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - run netryx geolocation
  - identify location from photo
  - use netryx to find where a photo was taken
  - open source geolocation pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It works by crawling street-view panoramas into a local index, then matching a query image against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and spatial refinement. Sub-50m accuracy. No internet presence of the target image required. Runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (deep feature matcher)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (handles blur/night images)
pip install kornia
```

### GPU support matrix

| Hardware | Backend | Notes |
|---|---|---|
| NVIDIA GPU (4GB+ VRAM) | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon M1–M4 | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"
```

Only needed if you want AI-assisted region guessing from visual clues (signs, architecture). Manual coordinate entry works without it.

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loading and descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # 512-dim descriptor matrix
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Launch the GUI

```bash
python test_super.py
```

**macOS blank GUI fix:**
```bash
brew install python-tk@3.11   # match your Python version
```

---

## Core Workflow

### Step 1 — Create an Index

Index an area before searching. The indexer crawls Street View panoramas and computes CosPlace fingerprints for every one.

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

**Time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|---|---|---|---|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

### Step 2 — Search

1. Select **Search** mode
2. Upload your street-level photo
3. Choose search method:
   - **Manual**: provide center lat/lon + radius (recommended)
   - **AI Coarse**: Gemini guesses the region from visual clues
4. Click **Run Search** → **Start Full Search**
5. GPS result + confidence score appear on the map

**Enable Ultra Mode** for difficult images (night, motion blur, low texture). Adds LoFTR dense matching, descriptor hopping, and 100m neighborhood expansion. Slower but more robust.

---

## Pipeline Internals

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py usage pattern
from cosplace_utils import load_cosplace_model, get_descriptor

model = load_cosplace_model(device="cuda")  # or "mps" / "cpu"

# Extract 512-dim fingerprint from query image
descriptor = get_descriptor(model, "query.jpg", device="cuda")

# Also extract flipped version (catches reversed perspectives)
descriptor_flipped = get_descriptor(model, "query.jpg", device="cuda", flip=True)
```

Index search is a single cosine similarity matrix multiply — runs in under 1 second regardless of index size. A haversine radius filter then restricts candidates to the specified area.

### Stage 2 — Local Feature Matching

```python
# Simplified representation of the matching stage
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

device = torch.device("cuda")   # or "mps"

# Feature extractor selection
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked").eval().to(device)   # or "disk"

# Load and extract keypoints
image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
matches = matches01["matches"]   # (N, 2) matched keypoint indices
```

For each of the top 500–1000 CosPlace candidates:
- Panorama is downloaded from Street View (8 tiles stitched)
- Rectilinear crop extracted at indexed heading
- **Three FOV crops** generated (70°, 90°, 110°) to handle zoom mismatches
- ALIKED/DISK keypoints extracted → LightGlue matched → RANSAC inlier count

### Stage 3 — Refinement

- **Heading refinement**: top 15 candidates tested at ±45° in 15° steps across 3 FOVs
- **Spatial consensus**: candidates clustered into 50m cells; clusters preferred over isolated high-inlier outliers
- **Confidence scoring**: geographic clustering + uniqueness ratio (best vs. runner-up at different location)

---

## Building a Large Index with build_index.py

For large areas, use the standalone high-performance index builder instead of the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --grid_resolution 300 \
  --output_dir ./cosplace_parts
```

Then compile the parts into a searchable index:

```python
import numpy as np
import os
import glob

parts = sorted(glob.glob("cosplace_parts/*.npz"))

all_descriptors = []
all_metadata = {"lat": [], "lon": [], "heading": [], "panoid": []}

for part in parts:
    data = np.load(part, allow_pickle=True)
    all_descriptors.append(data["descriptors"])
    for key in all_metadata:
        all_metadata[key].extend(data[key].tolist())

descriptors = np.vstack(all_descriptors).astype(np.float32)
np.save("index/cosplace_descriptors.npy", descriptors)
np.savez("index/metadata.npz", **{k: np.array(v) for k, v in all_metadata.items()})
```

---

## Multi-Area Index Strategy

All embeddings live in a single unified index. The radius filter handles region isolation at search time — no per-city indices needed.

```
Index once:
  Paris (48.8566, 2.3522), r=5km  → cosplace_parts/paris_*.npz
  London (51.5074, -0.1278), r=5km → cosplace_parts/london_*.npz
  NYC    (40.7128, -74.0060), r=5km → cosplace_parts/nyc_*.npz

Search Paris only:
  center=(48.8566, 2.3522), radius=5km  → only Paris results returned

Search London only:
  center=(51.5074, -0.1278), radius=5km → only London results returned
```

---

## Ultra Mode Internals

Enable when standard pipeline yields low inlier counts (<50) or on degraded images.

```python
# Ultra Mode adds three mechanisms:

# 1. LoFTR dense matching (detector-free — works without keypoints)
from kornia.feature import LoFTR
loftr = LoFTR(pretrained="outdoor").eval().to(device)
correspondences = loftr({"image0": gray0, "image1": gray1})
# Returns dense matches including in low-texture/blurry regions

# 2. Descriptor hopping
# If best match inliers < threshold:
#   Extract CosPlace descriptor from the MATCHED panorama (clean image)
#   Re-search index with that clean descriptor
#   Often finds the correct panorama the degraded query missed

# 3. Neighborhood expansion
# Search all panoramas within 100m of the best match coordinates
# Correct location is often one street node away from top CosPlace hit
```

---

## Common Patterns

### Pattern: Programmatic search (no GUI)

```python
import numpy as np
from cosplace_utils import load_cosplace_model, get_descriptor

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512) float32
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lat"]
lons = meta["lon"]

# Load model
model = load_cosplace_model(device="cuda")

# Extract query descriptor
q = get_descriptor(model, "query.jpg", device="cuda")       # (512,)
q_flip = get_descriptor(model, "query.jpg", device="cuda", flip=True)

# Cosine similarity search
q_norm = q / np.linalg.norm(q)
desc_norm = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = desc_norm @ q_norm                                  # (N,)

# Radius filter (haversine) — restrict to Paris 5km
from math import radians, sin, cos, sqrt, atan2

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1))*cos(radians(lat2))*sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1-a))

center_lat, center_lon = 48.8566, 2.3522
radius_m = 5000

mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

scores[~mask] = -1   # exclude out-of-radius entries
top_indices = np.argsort(scores)[::-1][:500]

# top_indices now contains the 500 best candidates within the radius
for idx in top_indices[:5]:
    print(f"Candidate: lat={lats[idx]:.6f}, lon={lons[idx]:.6f}, score={scores[idx]:.4f}")
```

### Pattern: Check device and select feature extractor

```python
import torch

def get_device():
    if torch.cuda.is_available():
        return torch.device("cuda"), "aliked"
    elif torch.backends.mps.is_available():
        return torch.device("mps"), "disk"
    else:
        return torch.device("cpu"), "disk"

device, features = get_device()
print(f"Using device: {device}, feature extractor: {features}")
```

### Pattern: Verify LightGlue installation

```python
try:
    from lightglue import LightGlue, ALIKED, DISK
    from lightglue.utils import load_image, rbd
    print("LightGlue OK")
except ImportError:
    print("Run: pip install git+https://github.com/cvg/LightGlue.git")
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11   # match your exact Python version
```

### CUDA out of memory

Reduce keypoints or process candidates in smaller batches:
```python
extractor = ALIKED(max_num_keypoints=512)   # default 1024, reduce if OOM
```

### Index search returns no results

- Confirm `index/cosplace_descriptors.npy` and `index/metadata.npz` both exist
- Check that search radius covers the indexed area — use the same or smaller radius than used during indexing
- Re-run index compilation if you added new `cosplace_parts/*.npz` files

### LightGlue import error

```bash
pip install git+https://github.com/cvg/LightGlue.git
```
Do not install from PyPI — the correct package is GitHub-only.

### LoFTR / Ultra Mode unavailable

```bash
pip install kornia
```

### Low match confidence on all candidates

- Enable Ultra Mode (adds LoFTR + descriptor hopping + neighborhood expansion)
- Try a tighter search radius if you have approximate knowledge of location
- Ensure the indexed area is dense enough — increase grid resolution or reduce radius

### Indexing interrupted — resume safely

Re-run the same Create Index command. The indexer checks existing `cosplace_parts/*.npz` files and skips already-processed panoramas.

---

## Key Configuration Parameters

| Parameter | Default | Notes |
|---|---|---|
| Grid resolution | 300 | Panorama sampling density — don't change without understanding coverage tradeoffs |
| Top candidates | 500–1000 | Retrieved from CosPlace before geometric verification |
| Heading refinement steps | ±45° at 15° intervals | Applied to top 15 candidates |
| FOV crops | 70°, 90°, 110° | Three crops per candidate to handle zoom mismatches |
| Spatial cluster cell | 50m | For consensus scoring |
| Ultra Mode neighborhood | 100m | Panoramas searched around best match |
| Ultra Mode inlier threshold | 50 | Below this triggers descriptor hopping |
```
