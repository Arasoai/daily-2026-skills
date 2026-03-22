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
  - run netryx geolocation pipeline
  - match street photo to coordinates
  - visual place recognition geolocation
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable embedding index, and matches query images using a three-stage computer vision pipeline (global retrieval → geometric verification → spatial refinement). No internet presence of the target image is required — it searches the physical world, not the web.

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

# Optional: LoFTR dense matcher for Ultra Mode
pip install kornia
```

### Requirements at a Glance

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4 GB | 8 GB+ |
| RAM | 8 GB | 16 GB+ |
| Storage | 10 GB | 50 GB+ |

**GPU backends:** CUDA (NVIDIA) → ALIKED extractor · MPS (Apple Silicon) → DISK extractor · CPU → DISK (slow)

### Optional: Gemini API Key (AI Coarse mode)

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

### Step 1 — Create an Index

Index a geographic area before searching. This crawls Street View panoramas and extracts CosPlace (512-dim) fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon, radius (km), grid resolution (default 300)
3. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

Indexing is **resumable** — interrupted runs pick up where they left off.

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini infers region from visual cues (signs, architecture)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on live map

---

## Project Structure

```
netryx/
├── test_super.py          # Main entrypoint: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading & descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (N × 512 float32)
    └── metadata.npz               # lat, lon, heading, panoid per entry
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py exposes:
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model(device="cuda")   # or "mps" / "cpu"

# Extract 512-dim fingerprint from a PIL image or file path
descriptor = get_descriptor(model, "query.jpg", device="cuda")
# Returns: np.ndarray shape (512,), float32, L2-normalized

# Flipped version (catches reversed perspectives)
descriptor_flipped = get_descriptor(model, "query.jpg", device="cuda", flip=True)
```

**Index search (cosine similarity + haversine radius filter):**

```python
import numpy as np

def search_index(query_desc, index_desc, metadata, center_lat, center_lon, radius_km, top_k=500):
    """
    query_desc:  np.ndarray (512,)
    index_desc:  np.ndarray (N, 512) — cosplace_descriptors.npy
    metadata:    dict with keys 'lat', 'lon', 'heading', 'panoid'
    Returns:     list of dicts sorted by similarity score
    """
    from math import radians, sin, cos, sqrt, atan2

    def haversine(lat1, lon1, lat2, lon2):
        R = 6371.0
        dlat = radians(lat2 - lat1)
        dlon = radians(lon2 - lon1)
        a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
        return R * 2 * atan2(sqrt(a), sqrt(1 - a))

    lats = metadata['lat']
    lons = metadata['lon']

    # Haversine radius filter
    mask = np.array([
        haversine(center_lat, center_lon, lat, lon) <= radius_km
        for lat, lon in zip(lats, lons)
    ])

    filtered_desc = index_desc[mask]
    filtered_meta_idx = np.where(mask)[0]

    # Cosine similarity (descriptors are L2-normalized → dot product = cosine)
    scores = filtered_desc @ query_desc  # shape (M,)

    top_idx = np.argsort(scores)[::-1][:top_k]

    results = []
    for rank, i in enumerate(top_idx):
        orig_idx = filtered_meta_idx[i]
        results.append({
            "rank": rank,
            "score": float(scores[i]),
            "lat": float(lats[orig_idx]),
            "lon": float(lons[orig_idx]),
            "heading": float(metadata['heading'][orig_idx]),
            "panoid": str(metadata['panoid'][orig_idx]),
        })
    return results
```

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")

# Choose extractor based on device
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def extract_features(image_path):
    image = load_image(image_path).to(device)
    return extractor.extract(image)

def match_pair(feats0, feats1):
    """Returns number of RANSAC-verified inliers."""
    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0_, feats1_, matches01_ = [rbd(x) for x in [feats0, feats1, matches01]]

    kpts0 = feats0_["keypoints"][matches01_["matches"][..., 0]]
    kpts1 = feats1_["keypoints"][matches01_["matches"][..., 1]]

    if len(kpts0) < 8:
        return 0

    import cv2
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, inlier_mask = cv2.findFundamentalMat(pts0, pts1, cv2.FM_RANSAC, 3.0, 0.99)
    inliers = int(inlier_mask.sum()) if inlier_mask is not None else 0
    return inliers
```

**Multi-FOV cropping (handles zoom mismatch):**

```python
# The pipeline tests 3 FOVs per candidate: 70°, 90°, 110°
# For each candidate panorama, crops are generated at the indexed heading
# Best-scoring FOV crop is kept per candidate

FOV_LIST = [70, 90, 110]
```

### Stage 3 — Refinement

```python
# Heading refinement: ±45° at 15° steps for top 15 candidates
HEADING_OFFSETS = list(range(-45, 46, 15))   # [-45, -30, -15, 0, 15, 30, 45]

# Spatial consensus: cluster matches into 50m cells
# Prefer clusters over isolated high-inlier outliers

# Confidence scoring:
# - Geographic clustering of top matches (tight cluster → high confidence)
# - Uniqueness ratio: best_inliers / second_best_inliers (same location)
```

### Ultra Mode (Difficult Images)

Enable for blurry, night, or low-texture images:

```python
# Ultra Mode adds three mechanisms:

# 1. LoFTR dense matching (detector-free, handles blur/low-contrast)
import kornia.feature as KF
loftr = KF.LoFTR(pretrained="outdoor").eval().to(device)

# 2. Descriptor hopping: if best match has <50 inliers,
#    extract CosPlace from the matched panorama and re-search index

# 3. Neighborhood expansion: search all panoramas within 100m of best match
NEIGHBORHOOD_RADIUS_M = 100
```

---

## Loading the Index Programmatically

```python
import numpy as np

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)

lats     = meta["lat"]      # float64 array (N,)
lons     = meta["lon"]      # float64 array (N,)
headings = meta["heading"]  # float64 array (N,)
panoids  = meta["panoid"]   # str array (N,)

print(f"Index contains {len(lats):,} panorama views")
```

---

## Building a Large Index (CLI / Standalone)

For large areas (5 km+), use the standalone high-performance builder:

```bash
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --grid-resolution 300 \
    --output-dir ./cosplace_parts
```

Then auto-compile parts into a searchable index via the GUI or by triggering the build step in `test_super.py`.

---

## Common Patterns

### Pattern 1: Programmatic Single-Image Geolocation

```python
import numpy as np
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

device = "cuda"  # or "mps" / "cpu"

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

# Extract query descriptor
model = get_cosplace_model(device=device)
query_desc = get_descriptor(model, "mystery_street.jpg", device=device)
query_desc_flip = get_descriptor(model, "mystery_street.jpg", device=device, flip=True)

# Combine normal + flipped (max pooling)
combined = np.maximum(query_desc, query_desc_flip)
combined /= np.linalg.norm(combined)

# Search index — Paris, 3km radius
results = search_index(
    query_desc=combined,
    index_desc=descriptors,
    metadata=meta,
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=3.0,
    top_k=500
)

print(f"Top match: {results[0]['lat']:.6f}, {results[0]['lon']:.6f} "
      f"(score: {results[0]['score']:.4f}, heading: {results[0]['heading']:.1f}°)")
```

### Pattern 2: Multi-City Index (No Partitioning Needed)

```python
# Index multiple cities into the SAME index — radius filter handles isolation
# Paris: center=(48.8566, 2.3522), radius=5km
# London: center=(51.5074, -0.1278), radius=5km
# Tel Aviv: center=(32.0853, 34.7818), radius=3km

# Search only returns results within your specified radius + center
# No city labels or partitions needed
```

### Pattern 3: Batch Geolocation Script

```python
import os
import json
from pathlib import Path

IMAGE_DIR = Path("./images_to_geolocate")
CENTER_LAT, CENTER_LON, RADIUS_KM = 48.8566, 2.3522, 5.0

model = get_cosplace_model(device="cuda")
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

results_all = {}

for img_path in IMAGE_DIR.glob("*.jpg"):
    desc = get_descriptor(model, str(img_path), device="cuda")
    desc_flip = get_descriptor(model, str(img_path), device="cuda", flip=True)
    combined = np.maximum(desc, desc_flip)
    combined /= np.linalg.norm(combined)

    top = search_index(combined, descriptors, meta, CENTER_LAT, CENTER_LON, RADIUS_KM, top_k=1)
    results_all[img_path.name] = {
        "lat": top[0]["lat"],
        "lon": top[0]["lon"],
        "confidence": top[0]["score"],
        "panoid": top[0]["panoid"],
    }
    print(f"{img_path.name}: ({top[0]['lat']:.6f}, {top[0]['lon']:.6f})")

with open("geolocation_results.json", "w") as f:
    json.dump(results_all, f, indent=2)
```

---

## Models Reference

| Model | Role | Device Preference |
|-------|------|------------------|
| CosPlace (512-dim) | Global retrieval fingerprint | CUDA → MPS → CPU |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU fallback |
| LightGlue | Deep feature matching | CUDA → MPS → CPU |
| LoFTR | Dense matching (Ultra Mode) | CUDA → MPS → CPU |

---

## Troubleshooting

### `ImportError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your Python version
```

### CUDA out of memory
- Reduce `max_num_keypoints` in the ALIKED/DISK extractor (try 512 instead of 1024)
- Reduce `top_k` candidates from 500 to 200
- Disable Ultra Mode

### Index search returns no results
- Verify your center coordinates and radius actually overlap with indexed areas
- Check `index/metadata.npz` lat/lon ranges:
  ```python
  meta = np.load("index/metadata.npz", allow_pickle=True)
  print(meta["lat"].min(), meta["lat"].max())
  print(meta["lon"].min(), meta["lon"].max())
  ```

### Low confidence / wrong location
- Increase radius slightly (the image may be near the edge of your indexed area)
- Enable **Ultra Mode** for difficult images (night, blur, low texture)
- Try **AI Coarse** mode if you have no prior location knowledge
- Ensure query image is street-level, not aerial or indoor

### Indexing stops mid-run
Indexing is resumable. Just re-run with the same parameters — already-processed panoramas are skipped via the incremental `cosplace_parts/*.npz` checkpoint system.

### `GEMINI_API_KEY` not found (AI Coarse mode)
```bash
export GEMINI_API_KEY="your_key_here"
# Then re-launch: python test_super.py
```

---

## Key Hyperparameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Panorama density during indexing; don't change |
| Top-K candidates | 500–1000 | Stage 1 retrieval pool size |
| Heading offsets | ±45° @ 15° steps | Applied to top 15 candidates in refinement |
| FOV list | 70°, 90°, 110° | Multi-FOV crops per candidate |
| Spatial cluster cell | 50 m | Consensus clustering granularity |
| Neighborhood expansion | 100 m | Ultra Mode spatial search radius |
| Descriptor hop threshold | 50 inliers | Ultra Mode: triggers re-search if below |
```
