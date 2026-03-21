```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - identify coordinates from an image
  - street level geolocation
  - build a street view index
  - use netryx to find location
  - reverse geolocate image
  - find GPS from street photo
  - run netryx geolocation pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, then uses a three-stage computer vision pipeline (global retrieval → geometric verification → refinement) to match a query image to a physical location with sub-50m accuracy — no landmarks or internet image presence required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git  # required
pip install kornia                                       # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API key for AI Coarse location mode

```bash
export GEMINI_API_KEY="your_key_here"
```

### macOS tkinter fix (blank GUI)

```bash
brew install python-tk@3.11   # match your Python version
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

GPU backend selected automatically:
- **NVIDIA**: CUDA + ALIKED (1024 keypoints)
- **Apple Silicon**: MPS + DISK (768 keypoints)
- **CPU**: DISK (slow but functional)

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface for both indexing and searching.

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Workflow

### Step 1 — Create an Index

Index an area before searching. The indexer crawls Street View panoramas, extracts CosPlace descriptors, and saves them.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius  | ~Panoramas | Time (M2 Max) | Index Size |
|---------|-----------|---------------|------------|
| 0.5 km  | 500       | 30 min        | ~60 MB     |
| 1 km    | 2,000     | 1–2 hrs       | ~250 MB    |
| 5 km    | 30,000    | 8–12 hrs      | ~3 GB      |
| 10 km   | 100,000   | 24–48 hrs     | ~7 GB      |

Indexing is **incremental** — safe to interrupt and resume.

**Via standalone builder (large datasets):**

```bash
python build_index.py
```

---

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: provide approximate center lat/lon + radius
   - **AI Coarse**: Gemini analyzes the image and guesses the region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score displayed on map

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py pattern
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model(device="cuda")   # or "mps" / "cpu"

# Extract 512-dim fingerprint from query image
descriptor = get_descriptor(model, "query.jpg", device="cuda")

# Also extract flipped version to catch reversed perspectives
import PIL.Image, torchvision.transforms as T, torch
img = PIL.Image.open("query.jpg").convert("RGB")
flipped = img.transpose(PIL.Image.FLIP_LEFT_RIGHT)
descriptor_flipped = get_descriptor(model, flipped, device="cuda")
```

Index search is a **single matrix multiplication** (cosine similarity) — runs in <1 second regardless of index size.

```python
import numpy as np

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons, headings, panoids = meta["lats"], meta["lons"], meta["headings"], meta["panoids"]

# Cosine similarity search
query_vec = descriptor / np.linalg.norm(descriptor)
index_vecs = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = index_vecs @ query_vec                            # (N,)

# Radius filter (haversine)
from math import radians, sin, cos, sqrt, atan2

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522   # Paris example
radius_km = 2.0

mask = np.array([
    haversine_km(center_lat, center_lon, lat, lon) <= radius_km
    for lat, lon in zip(lats, lons)
])

filtered_scores = np.where(mask, scores, -1)
top_indices = np.argsort(filtered_scores)[::-1][:500]   # top 500 candidates
```

---

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")

# Choose extractor by device
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def match_images(path_query, path_candidate):
    img0 = load_image(path_query).to(device)
    img1 = load_image(path_candidate).to(device)

    feats0 = extractor.extract(img0)
    feats1 = extractor.extract(img1)

    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0, feats1, matches01 = [rbd(x) for x in (feats0, feats1, matches01)]

    matched_kps0 = feats0["keypoints"][matches01["matches"][..., 0]]
    matched_kps1 = feats1["keypoints"][matches01["matches"][..., 1]]
    return matched_kps0, matched_kps1, matches01["matching_scores0"]
```

**RANSAC geometric verification** (filters false matches):

```python
import cv2

def count_inliers(kps0, kps1):
    if len(kps0) < 4:
        return 0
    pts0 = kps0.cpu().numpy()
    pts1 = kps1.cpu().numpy()
    _, mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, ransacReprojThreshold=4.0)
    return int(mask.sum()) if mask is not None else 0
```

**Multi-FOV crops** are tested at 70°, 90°, 110° per candidate to handle zoom mismatches.

---

### Stage 3 — Refinement

**Heading refinement** — for top 15 candidates, test ±45° in 15° steps across 3 FOVs:

```python
heading_offsets = range(-45, 46, 15)   # -45, -30, -15, 0, 15, 30, 45
fovs = [70, 90, 110]

best_inliers = 0
best_heading = base_heading

for offset in heading_offsets:
    for fov in fovs:
        candidate_crop = download_streetview_crop(panoid, base_heading + offset, fov)
        kps0, kps1, _ = match_images(query_path, candidate_crop)
        inliers = count_inliers(kps0, kps1)
        if inliers > best_inliers:
            best_inliers = inliers
            best_heading = base_heading + offset
```

**Spatial consensus clustering** — group matches into 50m cells, prefer clusters over isolated outliers:

```python
from collections import defaultdict

def cluster_candidates(candidates, cell_size_m=50):
    """candidates: list of (lat, lon, inliers)"""
    cells = defaultdict(list)
    for lat, lon, inliers in candidates:
        # ~0.00045 deg ≈ 50m
        cell_key = (round(lat / 0.00045), round(lon / 0.00045))
        cells[cell_key].append((lat, lon, inliers))

    # Score each cell: sum of inliers * count bonus
    best_cell = max(cells.values(), key=lambda grp: sum(x[2] for x in grp) * len(grp))
    best_match = max(best_cell, key=lambda x: x[2])
    return best_match
```

---

## Ultra Mode

Enable for difficult images (night, blur, low texture). Adds three techniques:

### 1. LoFTR Dense Matching

```python
import kornia.feature as KF
import kornia
import torch

loftr = KF.LoFTR(pretrained="outdoor").eval().to(device)

def loftr_match(img0_path, img1_path):
    def load_gray(path, size=(640, 480)):
        img = PIL.Image.open(path).convert("L").resize(size)
        return torch.tensor(np.array(img) / 255.0, dtype=torch.float32)[None, None].to(device)

    img0 = load_gray(img0_path)
    img1 = load_gray(img1_path)

    with torch.no_grad():
        out = loftr({"image0": img0, "image1": img1})

    kps0 = out["keypoints0"].cpu().numpy()
    kps1 = out["keypoints1"].cpu().numpy()
    conf = out["confidence"].cpu().numpy()

    # Filter by confidence
    mask = conf > 0.5
    return kps0[mask], kps1[mask]
```

### 2. Descriptor Hopping

```python
# If best match has < 50 inliers, re-search using the matched panorama's descriptor
if best_inliers < 50:
    matched_panorama_path = download_panorama(best_panoid)
    new_query_descriptor = get_descriptor(model, matched_panorama_path, device=str(device))
    # Re-run Stage 1 with new_query_descriptor
    new_scores = index_vecs @ (new_query_descriptor / np.linalg.norm(new_query_descriptor))
    new_top_indices = np.argsort(np.where(mask, new_scores, -1))[::-1][:500]
```

### 3. Neighborhood Expansion

```python
# Search all panoramas within 100m of best match
best_lat, best_lon = lats[best_idx], lons[best_idx]
neighborhood_mask = np.array([
    haversine_km(best_lat, best_lon, lat, lon) <= 0.1   # 100m
    for lat, lon in zip(lats, lons)
])
neighborhood_indices = np.where(neighborhood_mask)[0]
# Re-run Stage 2 on all neighborhood panoramas
```

---

## Index Management

The index is **source-agnostic** and **unified** — index multiple cities into one index, filter by coordinates at search time.

```python
# Load index parts and inspect coverage
import numpy as np
import glob

parts = glob.glob("cosplace_parts/*.npz")
print(f"Total parts: {len(parts)}")

meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]
print(f"Total panoramas indexed: {len(lats)}")
print(f"Lat range: {lats.min():.4f} – {lats.max():.4f}")
print(f"Lon range: {lons.min():.4f} – {lons.max():.4f}")
```

**Rebuild compiled index from parts:**

```bash
python build_index.py
```

This merges all `cosplace_parts/*.npz` into `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Common Patterns

### Programmatic search (no GUI)

```python
import numpy as np
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

device = "cuda"  # or "mps" / "cpu"
model = get_cosplace_model(device=device)

# 1. Load index
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons, headings, panoids = (
    meta["lats"], meta["lons"], meta["headings"], meta["panoids"]
)

# 2. Extract query descriptor
query_desc = get_descriptor(model, "my_street_photo.jpg", device=device)
query_vec = query_desc / np.linalg.norm(query_desc)

# 3. Normalize index
index_vecs = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)

# 4. Score + radius filter
scores = index_vecs @ query_vec
center_lat, center_lon, radius_km = 48.8566, 2.3522, 3.0

# haversine vectorized
def haversine_vec(lat1, lon1, lat2_arr, lon2_arr):
    R = 6371.0
    dlat = np.radians(lat2_arr - lat1)
    dlon = np.radians(lon2_arr - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2_arr)) * np.sin(dlon/2)**2
    return R * 2 * np.arctan2(np.sqrt(a), np.sqrt(1 - a))

dists = haversine_vec(center_lat, center_lon, lats, lons)
mask = dists <= radius_km
filtered_scores = np.where(mask, scores, -1.0)
top_k = np.argsort(filtered_scores)[::-1][:500]

print("Top candidate:")
best = top_k[0]
print(f"  lat={lats[best]:.6f}, lon={lons[best]:.6f}, heading={headings[best]}, panoid={panoids[best]}")
print(f"  CosPlace score: {filtered_scores[best]:.4f}")
```

### Batch index a list of coordinates

```python
# Useful when you have custom lat/lon lists instead of grid crawling
coords = [
    (48.8584, 2.2945),  # Eiffel Tower area
    (48.8530, 2.3499),  # Notre-Dame area
]

for lat, lon in coords:
    desc = get_descriptor(model, download_streetview_image(lat, lon), device=device)
    # save to cosplace_parts/
    np.savez(f"cosplace_parts/{lat}_{lon}.npz", descriptor=desc, lat=lat, lon=lon)
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| GUI appears blank on macOS | Bundled tkinter bug | `brew install python-tk@3.11` |
| `ModuleNotFoundError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| `ModuleNotFoundError: kornia` | Ultra Mode dependency missing | `pip install kornia` |
| CUDA out of memory | Too many keypoints | Reduce `max_num_keypoints` in extractor init |
| Index search returns no results | Radius too small or area not indexed | Increase radius or create index for that area |
| Low inlier counts everywhere | Query photo is heavily blurred/dark | Enable Ultra Mode; use LoFTR |
| Indexing stops mid-way | Network timeout or API rate limit | Safe to restart — resumes from last saved part |
| Wrong city matched | Insufficient radius filter | Tighten center coordinates and radius |
| `metadata.npz` not found | Index not compiled yet | Run `python build_index.py` |

### Check device detection

```python
import torch
print("CUDA:", torch.cuda.is_available())
print("MPS:", torch.backends.mps.is_available())
device = (
    "cuda" if torch.cuda.is_available() else
    "mps" if torch.backends.mps.is_available() else
    "cpu"
)
print("Using:", device)
```

### Verify LightGlue installation

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
print("LightGlue OK")
```

### Verify index integrity

```python
import numpy as np
desc = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)
assert desc.shape[1] == 512, "Expected 512-dim CosPlace descriptors"
assert len(desc) == len(meta["lats"]), "Descriptor/metadata count mismatch"
print(f"Index OK: {len(desc)} panoramas, shape {desc.shape}")
```

---

## Models Reference

| Model | Role | Loaded on |
|-------|------|-----------|
| CosPlace (CVPR 2022) | 512-dim global descriptor | Indexing + search Stage 1 |
| ALIKED (IEEE TIP 2023) | Local keypoints — CUDA | Search Stage 2 |
| DISK (NeurIPS 2020) | Local keypoints — MPS/CPU | Search Stage 2 |
| LightGlue (ICCV 2023) | Deep feature matching | Search Stage 2 |
| LoFTR (CVPR 2021) | Dense matching — Ultra Mode | Search Stage 2 (optional) |

All models are downloaded automatically on first use via their respective library loaders.
```
