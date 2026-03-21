---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source locally-hosted street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - identify location from photo
  - osint geolocation tool
  - reverse image geolocation
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature extraction (ALIKED/DISK), and deep feature matching (LightGlue + RANSAC).

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: install LightGlue from GitHub
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Platform Requirements

| Platform | GPU Backend | Notes |
|----------|-------------|-------|
| Mac M1/M2/M3/M4 | MPS | Uses DISK extractor |
| NVIDIA GPU (4GB+ VRAM) | CUDA | Uses ALIKED extractor |
| CPU only | None | Functional but slow |

### Optional: Gemini API for AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### 1. Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon (e.g. `48.8566, 2.3522` for Paris)
3. Set radius (start with `0.5`–`1` km for testing)
4. Set grid resolution (`300` recommended — don't change this)
5. Click **Create Index**

Indexing writes to `cosplace_parts/*.npz` incrementally (safe to interrupt/resume).

**Index size reference:**

| Radius | ~Panoramas | Time (M2 Max) | Disk |
|--------|-----------|---------------|------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

### 2. Search (geolocate a query image)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini infers region from visual clues automatically
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main application (GUI + indexing + search logic)
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

- Extracts a 512-dim descriptor from the query image
- Also extracts descriptor from horizontally-flipped version (handles reversed perspectives)
- Cosine similarity search against all indexed descriptors
- Haversine radius filter narrows to specified area
- Returns top 500–1000 candidates
- Runs in **< 1 second** (single matrix multiplication)

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

- Downloads Street View panorama (8 tiles, stitched)
- Crops at indexed heading angle
- Generates **multi-FOV crops** at 70°, 90°, 110° (handles zoom mismatch)
- ALIKED (CUDA) or DISK (MPS/CPU) extracts keypoints + descriptors
- LightGlue matches keypoints between query and candidate
- RANSAC filters to geometrically consistent inliers
- Processes 300–500 candidates in **2–5 minutes**

### Stage 3 — Refinement

- **Heading refinement**: Tests ±45° offsets at 15° steps × 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; clusters beat lone high-inlier outliers
- **Confidence scoring**: Geographic clustering density + uniqueness ratio (best vs. runner-up)

### Ultra Mode (difficult images: night, blur, low texture)

Enable via checkbox in GUI. Adds:
- **LoFTR**: Detector-free dense matching (handles blur/low contrast)
- **Descriptor hopping**: Re-searches index using descriptor from the matched (clean) panorama
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Using the Index Programmatically

```python
import numpy as np
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model
device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"
model = get_cosplace_model(device=device)

# Load your query image
image = Image.open("query_photo.jpg").convert("RGB")

# Extract CosPlace descriptor
descriptor = get_descriptor(model, image, device=device)  # shape: (512,)

# Load the index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
headings = meta["headings"]
panoids = meta["panoids"]

# Cosine similarity search
desc_norm = descriptor / (np.linalg.norm(descriptor) + 1e-8)
index_norm = descriptors / (np.linalg.norm(descriptors, axis=1, keepdims=True) + 1e-8)
similarities = index_norm @ desc_norm  # shape: (N,)

# Radius filter (haversine) — restrict to 2km around Paris center
from math import radians, sin, cos, sqrt, atan2

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522
radius_km = 2.0

mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
])

# Apply mask and get top candidates
masked_sims = np.where(mask, similarities, -1.0)
top_indices = np.argsort(masked_sims)[::-1][:500]

print("Top match:")
best = top_indices[0]
print(f"  lat={lats[best]:.6f}, lon={lons[best]:.6f}")
print(f"  heading={headings[best]}, panoid={panoids[best]}")
print(f"  similarity={similarities[best]:.4f}")
```

---

## Building a Large Index via CLI

For large areas, use the standalone builder instead of the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300
```

This writes chunks to `cosplace_parts/` and auto-compiles to `index/` when complete.

---

## Multi-City Index Pattern

Netryx uses a **single unified index** — all cities coexist and are separated at search time by coordinates + radius.

```python
# Index Paris (run once)
# python build_index.py --lat 48.8566 --lon 2.3522 --radius 5.0

# Index London (appends to same index)
# python build_index.py --lat 51.5074 --lon -0.1278 --radius 5.0

# Search only in Paris — London entries never appear
search_center = (48.8566, 2.3522)
search_radius_km = 5.0
# Apply haversine mask as shown above
```

---

## Common Patterns

### Check available GPU backend

```python
import torch

if torch.cuda.is_available():
    device = "cuda"
    extractor_type = "ALIKED"
elif torch.backends.mps.is_available():
    device = "mps"
    extractor_type = "DISK"
else:
    device = "cpu"
    extractor_type = "DISK"

print(f"Using device: {device}, extractor: {extractor_type}")
```

### Verify index integrity

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

print(f"Index contains {len(descriptors)} panorama views")
print(f"Descriptor shape: {descriptors.shape}")   # (N, 512)
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
```

### Batch descriptor extraction

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch
import numpy as np
from pathlib import Path

device = "cuda" if torch.cuda.is_available() else "cpu"
model = get_cosplace_model(device=device)

image_paths = list(Path("query_images/").glob("*.jpg"))
descriptors = []

for path in image_paths:
    img = Image.open(path).convert("RGB")
    desc = get_descriptor(model, img, device=device)
    descriptors.append(desc)
    print(f"Processed: {path.name}")

descriptors = np.stack(descriptors)  # shape: (N, 512)
np.save("query_descriptors.npy", descriptors)
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| GUI appears blank on macOS | Bundled tkinter bug | `brew install python-tk@3.11` |
| `ModuleNotFoundError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| `ModuleNotFoundError: kornia` | Ultra Mode dependency missing | `pip install kornia` |
| Low match confidence (<30 inliers) | Degraded image or heading mismatch | Enable **Ultra Mode** |
| Index search returns 0 candidates | Radius too small or wrong coordinates | Increase radius or verify center coordinates |
| MPS out of memory | VRAM pressure on Mac | Reduce candidates from 500 to 200 in GUI |
| Indexing stalls / never resumes | Partial `.npz` chunk | Delete the last incomplete file in `cosplace_parts/` and restart |
| CUDA out of memory | VRAM too small | Lower keypoint count (ALIKED default 1024 → try 512) |
| `cosplace_descriptors.npy` not found | Index not compiled yet | Re-run indexing or run `build_index.py` to completion |

### Debug: confirm LightGlue import works

```python
try:
    from lightglue import LightGlue, SuperPoint, DISK, ALIKED
    from lightglue.utils import load_image, rbd
    print("LightGlue import OK")
except ImportError as e:
    print(f"LightGlue import failed: {e}")
    print("Run: pip install git+https://github.com/cvg/LightGlue.git")
```

---

## Model Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace | Global 512-dim visual fingerprint | All |
| ALIKED (1024 kp) | Local keypoints + descriptors | CUDA only |
| DISK (768 kp) | Local keypoints + descriptors | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Detector-free dense matching (Ultra Mode) | All (requires `kornia`) |

---

## Key Parameters (GUI)

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Don't change — affects panorama crawl density |
| Search radius | 1–5 km | Smaller = faster search |
| Top candidates | 500 | Reduce to 200 for speed, increase to 1000 for recall |
| Ultra Mode | Off | Enable for night/blur/low-texture images |
| Heading refinement steps | ±45° @ 15° | Catches viewing direction mismatches |
| Spatial consensus cell | 50m | Clusters nearby matches to reduce false positives |
