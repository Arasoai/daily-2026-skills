---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - index street view panoramas
  - use netryx to locate
  - reverse geolocate image
  - find location from photo
  - visual place recognition pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted open-source geolocation engine that takes any street-level photo and returns precise GPS coordinates (sub-50m accuracy). It works by indexing crawled Street View panoramas as 512-dim CosPlace fingerprints, then matching a query image through a three-stage pipeline: global retrieval → local geometric verification → spatial refinement. No landmarks or internet image presence required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                       # optional: Ultra Mode (LoFTR)
```

**macOS tkinter fix** (blank GUI):
```bash
brew install python-tk@3.11    # match your Python version
```

**Optional — Gemini API for AI Coarse mode:**
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

GPU backends auto-selected: CUDA (NVIDIA) → MPS (Apple Silicon) → CPU fallback.

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. All indexing, searching, and result visualization runs from here.

---

## Project Structure

```
netryx/
├── test_super.py          # Main GUI app — indexing + search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz, created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Lat/lon, headings, panorama IDs
```

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area by crawling Street View panoramas and extracting CosPlace descriptors.

**Via GUI:**
1. Select **Create** mode
2. Enter center `lat, lon` of the area
3. Set radius (km) and grid resolution (default 300 — don't change)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hr        | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hr       | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hr      | ~7 GB      |

The index saves incrementally — safe to interrupt and resume.

**All cities go into one unified index.** Radius filtering at search time handles city separation automatically.

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload any street-level photo
3. Choose search method:
   - **Manual**: provide approximate center `lat, lon` + radius
   - **AI Coarse**: Gemini infers region from visual cues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS pin on map + confidence score

**Enable Ultra Mode** checkbox for difficult images (night, blur, low texture). Adds LoFTR dense matching, descriptor hopping, and ±100m neighborhood expansion. Slower but more robust.

---

## Pipeline Internals

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py pattern
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model(device="cuda")  # or "mps" / "cpu"

# Extract 512-dim descriptor from a PIL image
descriptor = get_descriptor(model, pil_image, device="cuda")

# Also extracts from horizontally-flipped version for reversed perspectives
descriptor_flipped = get_descriptor(model, pil_image.transpose(Image.FLIP_LEFT_RIGHT), device="cuda")
```

Index search is a single cosine similarity matrix multiply + haversine radius filter → top 500–1000 candidates in <1 second.

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```python
# Pseudo-code reflecting internal pipeline
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

# Device-aware extractor selection
if device == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked").eval().to(device)   # or "disk"

# For each candidate: download panorama, crop at 3 FOVs (70°, 90°, 110°)
for fov in [70, 90, 110]:
    candidate_crop = get_rectilinear_crop(panorama, heading, fov)
    
    feats0 = extractor.extract(query_tensor)
    feats1 = extractor.extract(candidate_tensor)
    matches = matcher({"image0": feats0, "image1": feats1})
    
    # RANSAC filters to geometrically consistent inliers
    inliers = ransac_filter(matches)
```

300–500 candidates processed in 2–5 minutes depending on hardware.

### Stage 3 — Refinement

```python
# Heading refinement: test ±45° at 15° steps for top 15 candidates
for heading_offset in range(-45, 46, 15):
    refined_crop = get_rectilinear_crop(panorama, base_heading + heading_offset, fov)
    # re-run LightGlue, track best inlier count

# Spatial consensus: cluster matches into 50m cells
# Prefer clusters over isolated high-inlier outliers

# Confidence score based on:
#   - geographic clustering of top matches
#   - uniqueness ratio (best vs. runner-up at different location)
```

---

## Building a Large Index Programmatically

Use `build_index.py` for large areas outside the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300 \
  --device cuda
```

This writes chunks to `cosplace_parts/` and auto-builds the compiled index at `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Using the Index Programmatically

```python
import numpy as np
from math import radians, sin, cos, sqrt, atan2

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # shape: (N,)
lons = meta["lons"]       # shape: (N,)
headings = meta["headings"]
panoids = meta["panoids"]

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1))*cos(radians(lat2))*sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """
    Returns indices of top_k candidates within radius sorted by cosine similarity.
    query_descriptor: np.ndarray shape (512,), already L2-normalized
    """
    # Radius filter
    distances = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i])
        for i in range(len(lats))
    ])
    mask = distances <= radius_km
    
    # Cosine similarity (descriptors assumed normalized)
    sims = descriptors[mask] @ query_descriptor
    
    # Get top_k
    masked_indices = np.where(mask)[0]
    top_local = np.argsort(sims)[::-1][:top_k]
    top_global = masked_indices[top_local]
    
    return [
        {
            "index": int(top_global[i]),
            "similarity": float(sims[top_local[i]]),
            "lat": float(lats[top_global[i]]),
            "lon": float(lons[top_global[i]]),
            "heading": float(headings[top_global[i]]),
            "panoid": str(panoids[top_global[i]]),
        }
        for i in range(len(top_global))
    ]

# Example usage
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image

model = get_cosplace_model(device="cuda")
img = Image.open("query.jpg")
desc = get_descriptor(model, img, device="cuda")          # (512,) normalized
desc_flip = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT), device="cuda")

# Average normal + flipped descriptor
combined = (desc + desc_flip) / 2
combined /= np.linalg.norm(combined)

candidates = search_index(combined, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
print(f"Top candidate: {candidates[0]}")
```

---

## Common Patterns

### Pattern: Geolocate with Known Approximate Region

```python
# 1. Index the region first (run once)
# python test_super.py → Create mode → lat/lon/radius → Create Index

# 2. Search with known region
# GUI: Search mode → Manual → enter approximate center + radius
# Result: GPS coordinates + confidence score shown on map
```

### Pattern: Geolocate with No Prior Knowledge (AI Coarse)

```bash
export GEMINI_API_KEY=$GEMINI_API_KEY   # set from environment, never hardcode

# GUI: Search mode → AI Coarse → upload photo → Run Search
# Gemini analyzes visual cues (signs, architecture, vegetation) → infers region
# Pipeline then runs normally within that region
```

### Pattern: Ultra Mode for Difficult Images

Enable in GUI, or note that it activates:
1. **LoFTR** dense matching (no keypoint detection needed — handles blur/night)
2. **Descriptor hopping** — re-indexes using the matched panorama's clean descriptor
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

Use Ultra Mode when standard search returns confidence < 50 inliers or obviously wrong location.

### Pattern: Multi-City Index

```python
# Index Paris
# GUI: Create → lat=48.8566, lon=2.3522, radius=5km

# Index London (same index, just run again with different coords)
# GUI: Create → lat=51.5074, lon=-0.1278, radius=5km

# Search is automatically scoped by center+radius at query time
# No city selection needed
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your exact Python version
```

### CUDA out of memory
- Reduce `max_num_keypoints` in extractor (1024 → 512)
- Process fewer candidates (top_k 500 → 200)
- Switch to MPS or CPU backend

### LightGlue import error
```bash
pip install git+https://github.com/cvg/LightGlue.git
# Note: NOT available on PyPI — must install from GitHub
```

### Poor match quality / wrong location
1. Enable **Ultra Mode**
2. Increase search radius (the correct panorama may be slightly outside initial radius)
3. Ensure the indexed area was crawled densely enough (radius 0.5–1km with resolution 300 for testing)
4. Try a different crop of the query image — remove sky/foreground, focus on architectural features

### Index build interrupted
Safe to re-run — index builds incrementally from `cosplace_parts/*.npz`. Already-processed panoramas are skipped.

### No panoramas found in area
Some areas have sparse Street View coverage. Verify coverage at [maps.google.com](https://maps.google.com) before indexing.

### LoFTR / kornia not found (Ultra Mode disabled)
```bash
pip install kornia
```

---

## Key Models Reference

| Model | Role | Used When |
|-------|------|-----------|
| CosPlace | Global 512-dim descriptor | Always — retrieval stage |
| ALIKED | Local keypoints + descriptors | CUDA devices |
| DISK | Local keypoints + descriptors | MPS / CPU devices |
| LightGlue | Deep feature matching | Always — verification stage |
| LoFTR | Detector-free dense matching | Ultra Mode only |
