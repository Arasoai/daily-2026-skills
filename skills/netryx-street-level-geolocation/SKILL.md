---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - run netryx geolocation
  - use netryx to find where a photo was taken
  - local geolocation engine setup
  - visual place recognition pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable visual index using CosPlace (512-dim descriptors), then matches query images via ALIKED/DISK keypoints and LightGlue feature matching — achieving sub-50m accuracy without landmarks or internet image search.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                        # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API key for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"
```

### macOS tkinter fix (blank GUI)

```bash
brew install python-tk@3.11   # match your Python version
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. All indexing and searching is done through it.

---

## Core Workflow

### 1. Create an Index

Before searching, index an area by crawling street-view panoramas and extracting CosPlace fingerprints.

**In the GUI:**
- Select **Create** mode
- Enter center lat/lon of the area
- Set radius (start with `0.5`–`1` km for testing)
- Set grid resolution (`300` recommended — do not change)
- Click **Create Index**

**Indexing time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is incremental — safe to interrupt and resume.

**Output files:**

```
cosplace_parts/          # raw embedding chunks (.npz files)
index/
  cosplace_descriptors.npy   # all 512-dim descriptors
  metadata.npz               # lat/lon, headings, panoid IDs
```

### 2. Search

- Select **Search** mode
- Upload a street-level photo
- Choose method:
  - **Manual**: provide approximate center lat/lon + radius
  - **AI Coarse**: let Gemini guess the region from visual clues
- Click **Run Search** → **Start Full Search**
- Result: GPS coordinates + confidence score shown on map

### 3. Ultra Mode

Enable **Ultra Mode** checkbox for difficult images (night, blur, low texture). Adds:
- LoFTR dense matching (detector-free)
- Descriptor hopping (re-searches from matched panorama descriptor)
- Neighborhood expansion (±100m around best match)

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone CLI index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created)
└── index/                 # Compiled searchable index (auto-created)
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace → 512-dim descriptor
    ├── Flipped CosPlace → 512-dim descriptor (catches reversed perspectives)
    │
    ▼
Index Search (cosine similarity, haversine radius filter)
    └── Top 500–1000 candidates
    │
    ▼
For each candidate:
    Download panorama (8 tiles, stitched)
    Crop at heading angle × 3 FOVs (70°, 90°, 110°)
    ALIKED (CUDA) or DISK (MPS/CPU) → keypoints + descriptors
    LightGlue → feature matches
    RANSAC → geometric inliers
    │
    ▼
Heading Refinement (±45°, 15° steps, top 15 candidates)
Spatial Consensus Clustering (50m cells)
Confidence Scoring (uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Using `cosplace_utils.py` Directly

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import numpy as np

# Load the CosPlace model (downloads weights on first run)
model = load_cosplace_model(device="cuda")  # or "mps" or "cpu"

# Extract a 512-dim descriptor from an image
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device="cuda")
# descriptor.shape == (512,)

# Also extract flipped version to catch reversed perspectives
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = extract_descriptor(model, img_flipped, device="cuda")
```

---

## Building the Index from Parts (Standalone)

Use `build_index.py` for large datasets where the GUI builder is too slow:

```bash
python build_index.py
```

This reads all `.npz` files in `cosplace_parts/` and compiles them into:
- `index/cosplace_descriptors.npy`
- `index/metadata.npz`

---

## Searching the Index Programmatically

```python
import numpy as np
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]      # shape: (N,)
lons = meta["lons"]      # shape: (N,)
headings = meta["headings"]  # shape: (N,)
panoids = meta["panoids"]    # shape: (N,)

# Load model and extract query descriptor
model = load_cosplace_model(device="cuda")
img = Image.open("query.jpg").convert("RGB")
query_desc = extract_descriptor(model, img, device="cuda")  # (512,)

# Cosine similarity search
query_norm = query_desc / np.linalg.norm(query_desc)
desc_norms = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
similarities = desc_norms @ query_norm  # (N,)

# Haversine radius filter (example: 5km around Paris)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 5.0

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

distances_km = haversine_km(center_lat, center_lon, lats, lons)
radius_mask = distances_km <= radius_km

# Apply mask and get top-500 candidates
masked_sims = np.where(radius_mask, similarities, -1.0)
top_indices = np.argsort(masked_sims)[::-1][:500]

for idx in top_indices[:10]:
    print(f"panoid={panoids[idx]}, lat={lats[idx]:.6f}, lon={lons[idx]:.6f}, "
          f"heading={headings[idx]}, sim={similarities[idx]:.4f}")
```

---

## LightGlue Matching Example

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
from PIL import Image
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU (matches Netryx behavior)
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

# Load and extract features from query image
query_tensor = load_image("query.jpg").to(device)
feats_query = extractor.extract(query_tensor)

# Load and extract features from candidate panorama crop
candidate_tensor = load_image("candidate_crop.jpg").to(device)
feats_candidate = extractor.extract(candidate_tensor)

# Match
with torch.no_grad():
    matches_result = matcher({"image0": feats_query, "image1": feats_candidate})

# Unpack (remove batch dim)
feats_query, feats_candidate, matches_result = [
    rbd(x) for x in [feats_query, feats_candidate, matches_result]
]

matches = matches_result["matches"]          # (M, 2) matched keypoint indices
scores  = matches_result["matching_scores0"] # (M,) confidence scores

kpts0 = feats_query["keypoints"][matches[:, 0]]     # matched kpts in query
kpts1 = feats_candidate["keypoints"][matches[:, 1]] # matched kpts in candidate

print(f"Raw matches: {len(matches)}")

# RANSAC geometric verification
import cv2
if len(matches) >= 4:
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, inlier_mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, 5.0)
    n_inliers = int(inlier_mask.sum()) if inlier_mask is not None else 0
    print(f"Inliers after RANSAC: {n_inliers}")
```

---

## Platform-Specific Behavior

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|--------------|---------------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK (768 kp) |
| LoFTR (Ultra Mode) | ✅ | ✅ | ✅ (slow) |
| Recommended VRAM | 8GB+ | 8GB+ unified | N/A |

Detect device in code:

```python
import torch

def get_device():
    if torch.cuda.is_available():
        return torch.device("cuda")
    elif torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")

device = get_device()
```

---

## Multi-City Indexing Pattern

All cities share one index. The radius filter at search time isolates results:

```python
# Index Paris (run Create mode, center=Paris)
# Index London (run Create mode, center=London)
# Index Tokyo  (run Create mode, center=Tokyo)
# All go into the same cosplace_parts/ → index/

# Search Paris only:
center_lat, center_lon = 48.8566, 2.3522
radius_km = 5.0

# Search London only:
center_lat, center_lon = 51.5074, -0.1278
radius_km = 10.0

# No city selection needed — coordinates + radius do the filtering
```

---

## Common Patterns & Tips

### Choosing search radius
- If you know the country/city: `1–10 km`
- If you know the district: `0.5–2 km`
- Blind search (AI Coarse mode): Gemini estimates a region first, then Manual search within that region

### Confidence scoring interpretation
- **High confidence**: top match has many inliers AND spatial clustering of top-N results in same location
- **Low confidence**: try Ultra Mode or expand search radius
- **False positive risk**: check uniqueness ratio — if runner-up is close, result is ambiguous

### Index freshness
Re-index an area if street-view data is more than ~1 year old (panoramas change).

### Resuming interrupted indexing
Just re-run Create mode with the same parameters. The system detects existing `.npz` chunks in `cosplace_parts/` and skips already-processed grid points.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| GUI appears blank on macOS | `brew install python-tk@3.11` (match your Python version) |
| `ModuleNotFoundError: lightglue` | `pip install git+https://github.com/cvg/LightGlue.git` |
| LoFTR not available | `pip install kornia` |
| CUDA out of memory | Reduce `max_num_keypoints` in extractor init, or switch to CPU |
| Index search returns 0 candidates | Radius too small or area not indexed — check center coords and re-index |
| Very low inlier counts (<10) | Enable Ultra Mode; try different FOV crops manually |
| Gemini AI Coarse fails | Check `GEMINI_API_KEY` env var; fall back to Manual mode with known region |
| Indexing hangs mid-way | Safe to Ctrl+C and restart — incremental saves resume from last chunk |
| MPS errors on Apple Silicon | Ensure PyTorch ≥ 2.0: `pip install --upgrade torch torchvision` |

---

## Key Dependencies

```
torch / torchvision      # deep learning backend
lightglue                # install from GitHub (see above)
kornia                   # optional, Ultra Mode LoFTR
pillow                   # image loading
numpy                    # descriptor math
opencv-python            # RANSAC, image stitching
requests                 # panorama tile fetching
tkinter                  # GUI (system package, not pip)
```
