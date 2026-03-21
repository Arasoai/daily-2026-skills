```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - identify location from street photo
  - run netryx geolocation pipeline
  - reverse geolocate image coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes Street View panoramas into a searchable vector index, then matches query images through a three-stage pipeline: global retrieval (CosPlace), local geometric verification (ALIKED/DISK + LightGlue), and spatial refinement with confidence scoring. Sub-50m accuracy. No landmarks required. Runs entirely on local hardware.

---

## Installation

```bash
# Clone
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

# Virtual environment
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# Core dependencies
pip install -r requirements.txt

# LightGlue (required — install from GitHub, not PyPI)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode dense matching
pip install kornia
```

### Optional: Gemini AI coarse localization

```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

### GPU support matrix

| Hardware | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU (4GB+ VRAM) | CUDA | ALIKED extractor (1024 keypoints) |
| Apple Silicon M1+ | MPS | DISK extractor (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix**: `brew install python-tk@3.11` (match your Python version)

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI, indexing, search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Workflow

### 1. Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Index is saved incrementally — safe to interrupt and resume.

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

### 2. Search (geolocate a query image)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose **Manual** (enter lat/lon + radius) or **AI Coarse** (Gemini guesses region)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### 3. Enable Ultra Mode (difficult images)

Check the **Ultra Mode** checkbox before searching. Adds:
- **LoFTR** dense matching (handles blur, low contrast, night images)
- **Descriptor hopping** (re-searches index from matched panorama's clean descriptor)
- **Neighborhood expansion** (checks all panoramas within 100m of best match)

---

## Pipeline Internals

### Stage 1 — Global Retrieval

```python
# cosplace_utils.py pattern
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()  # loads CosPlace backbone (512-dim output)

# Extract descriptor from query image
descriptor = get_descriptor(model, "query.jpg")          # shape: (512,)
flipped_desc = get_descriptor(model, "query.jpg", flip=True)  # horizontal flip

# Index search: cosine similarity against all indexed descriptors
# cosplace_descriptors.npy shape: (N, 512)
import numpy as np
index_descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)

# Cosine similarity (single matrix multiply — runs in <1s for millions of entries)
scores = index_descriptors @ descriptor  # (N,)

# Radius filter using haversine distance
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000  # meters
    phi1, phi2 = radians(lat1), radians(lat2)
    dphi = radians(lat2 - lat1)
    dlambda = radians(lon2 - lon1)
    a = sin(dphi/2)**2 + cos(phi1)*cos(phi2)*sin(dlambda/2)**2
    return 2 * R * asin(sqrt(a))

center_lat, center_lon = 48.8566, 2.3522   # Paris example
radius_m = 2000

lats = meta["lats"]
lons = meta["lons"]
mask = np.array([
    haversine(center_lat, center_lon, lat, lon) <= radius_m
    for lat, lon in zip(lats, lons)
])

filtered_scores = scores * mask   # zero out out-of-radius entries
top_candidates = np.argsort(filtered_scores)[::-1][:500]  # top 500
```

### Stage 2 — Local Geometric Verification

```python
# Conceptual representation of the matching stage in test_super.py
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available()
                       else "mps" if torch.backends.mps.is_available()
                       else "cpu")

# Choose extractor by device
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def match_images(img0_path, img1_path):
    image0 = load_image(img0_path).to(device)
    image1 = load_image(img1_path).to(device)

    feats0 = extractor.extract(image0)
    feats1 = extractor.extract(image1)

    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

    kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
    kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
    return kpts0, kpts1, matches01["matching_scores0"]

# RANSAC for geometric verification
import cv2

def count_inliers(kpts0, kpts1):
    if len(kpts0) < 4:
        return 0
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, mask = cv2.findFundamentalMat(pts0, pts1, cv2.FM_RANSAC, 3.0)
    if mask is None:
        return 0
    return int(mask.sum())
```

### Stage 3 — Multi-FOV Crops + Heading Refinement

```python
# The pipeline tests 3 FOVs to handle zoom mismatches
FOV_VARIANTS = [70, 90, 110]

# Heading refinement: test ±45° at 15° steps for top 15 candidates
HEADING_OFFSETS = range(-45, 46, 15)   # -45, -30, -15, 0, 15, 30, 45

# Spatial consensus: cluster matches into 50m grid cells
# If multiple candidates land in same cell → prefer cluster over isolated high-inlier match
CLUSTER_RADIUS_M = 50
```

---

## Configuration Reference

All configuration is done via the GUI. Key parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| Center Lat/Lon | — | Geographic center of search area |
| Radius (km) | — | Search radius for both indexing and querying |
| Grid Resolution | 300 | Panorama sampling density (don't change) |
| Ultra Mode | Off | Enables LoFTR + descriptor hopping + neighborhood expansion |
| Search Method | Manual | Manual (coords) or AI Coarse (Gemini guesses region) |

---

## Index Management

The index is unified across all indexed areas. Coordinates + radius handle region selection at search time — no city-selection step needed.

```python
# Index files
index/cosplace_descriptors.npy   # shape: (N, 512), float32
index/metadata.npz               # keys: lats, lons, headings, panoids

# Load index manually
import numpy as np
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats    = meta["lats"]      # (N,) float64
lons    = meta["lons"]      # (N,) float64
heading = meta["headings"]  # (N,) float32
panoids = meta["panoids"]   # (N,) str — Street View panorama IDs

print(f"Index contains {len(lats):,} panorama views")
```

### Build index from parts (large datasets)

```bash
# After indexing, compile parts into searchable index
python build_index.py
```

---

## Common Patterns

### Pattern: Programmatic search (no GUI)

```python
import numpy as np
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

# Setup
model = get_cosplace_model()
device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]

def geolocate(image_path, center_lat, center_lon, radius_m=2000, top_k=500):
    """Return top-k candidate (lat, lon, score) tuples for a query image."""
    # Extract descriptors (normal + flipped)
    desc  = get_descriptor(model, image_path)
    fdesc = get_descriptor(model, image_path, flip=True)
    combined = (descriptors @ desc + descriptors @ fdesc) / 2  # average scores

    # Radius mask
    from math import radians, cos, sin, asin, sqrt
    def hav(la1, lo1, la2, lo2):
        R = 6371000
        p1, p2 = radians(la1), radians(la2)
        a = sin((p2-p1)/2)**2 + cos(p1)*cos(p2)*sin(radians(lo2-lo1)/2)**2
        return 2*R*asin(sqrt(a))

    mask = np.array([hav(center_lat, center_lon, la, lo) <= radius_m
                     for la, lo in zip(lats, lons)])
    combined[~mask] = -1

    top_idx = np.argsort(combined)[::-1][:top_k]
    return [(lats[i], lons[i], combined[i]) for i in top_idx]

candidates = geolocate("mystery_street.jpg", 48.8566, 2.3522, radius_m=3000)
best_lat, best_lon, best_score = candidates[0]
print(f"Best match: {best_lat:.6f}, {best_lon:.6f}  (score: {best_score:.4f})")
```

### Pattern: Extract CosPlace descriptor from image bytes

```python
from PIL import Image
import io
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()

# From file path
desc = get_descriptor(model, "image.jpg")

# From PIL Image (if get_descriptor supports it — else save temp file)
import tempfile, os

def descriptor_from_pil(pil_image, model):
    with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as f:
        pil_image.save(f.name)
        desc = get_descriptor(model, f.name)
    os.unlink(f.name)
    return desc
```

### Pattern: Add new area to existing index

Just run **Create Index** again with different center coordinates — Netryx appends to `cosplace_parts/` and merges on next search. No manual merging required.

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your exact Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
# Verify:
python -c "from lightglue import LightGlue; print('OK')"
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED (default 1024 → try 512)
- Disable Ultra Mode (LoFTR adds ~2GB VRAM)
- Use CPU fallback: `CUDA_VISIBLE_DEVICES="" python test_super.py`

### Low match confidence / wrong location
1. Enable **Ultra Mode** for blurry/night/low-texture images
2. Widen search radius (the correct location may be near index boundary)
3. Ensure the area is indexed — run Create mode for the expected region
4. Try AI Coarse mode if you have no idea of the region

### Indexing stalls / no panoramas found
- Internet connection required for indexing (fetches Street View tiles)
- Some areas have no Street View coverage — try a denser urban area first
- Check that grid resolution is 300 (default) — lower values produce sparser coverage

### LoFTR not available (Ultra Mode)
```bash
pip install kornia
python -c "import kornia; print(kornia.__version__)"
```

### Index rebuild after crash
```bash
# Parts are saved incrementally — just re-run indexing, it resumes
# To force full recompile of parts into searchable index:
python build_index.py
```

---

## Models Reference

| Model | Role | Loaded on |
|-------|------|-----------|
| CosPlace (ResNet backbone) | 512-dim global place descriptor | CPU/GPU |
| ALIKED | Local keypoints + descriptors | CUDA only |
| DISK | Local keypoints + descriptors | MPS / CPU |
| LightGlue | Deep feature matching | Any device |
| LoFTR (via kornia) | Detector-free dense matching (Ultra) | Any device |

All models are downloaded automatically on first use via their respective libraries (torch.hub / HuggingFace / package weights).

---

## Key Facts for AI Agents

- **Entry point**: `python test_super.py` (GUI-only application)
- **No REST API** — interaction is through the Tkinter GUI or by calling Python functions directly
- **Index is source-agnostic**: works with Mapillary, KartaView, or any panorama provider
- **Index format**: `.npy` (descriptors) + `.npz` (metadata) — standard NumPy, easy to inspect
- **Incremental indexing**: always safe to interrupt; resumes on next run
- **Multi-area support**: one unified index, radius filter at query time — no per-city files
- **Confidence score**: based on inlier count, geographic clustering, and uniqueness ratio vs runner-up
- **Gemini API**: optional, only for AI Coarse region estimation; not needed for searching known regions
```
