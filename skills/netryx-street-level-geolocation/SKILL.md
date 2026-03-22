---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - reverse geolocate image
  - identify location from photo
  - build street view index
  - run netryx geolocation
  - match photo to map coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted street-level geolocation engine. Upload any street photo → get precise GPS coordinates. It crawls street-view panoramas, indexes them with CosPlace visual fingerprints, then matches query images via ALIKED/DISK keypoints + LightGlue feature matching. Sub-50m accuracy. Runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must install from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

**GPU support:**
- NVIDIA: CUDA (uses ALIKED, 1024 keypoints)
- Apple Silicon: MPS (uses DISK, 768 keypoints)
- CPU: works, significantly slower

**Optional — Gemini API for AI Coarse location guessing:**
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

### 1. Create an Index (crawl + fingerprint an area)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Index is saved incrementally to `cosplace_parts/` — safe to interrupt and resume.

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

### 2. Search (geolocate a query image)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose:
   - **Manual**: provide center lat/lon + radius
   - **AI Coarse**: let Gemini estimate the region from visual clues
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Index Structure

```
netryx/
├── test_super.py              # Main GUI app (indexing + search)
├── cosplace_utils.py          # CosPlace model loading + descriptor extraction
├── build_index.py             # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/            # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # lat, lon, heading, panorama IDs
```

All areas share one unified index. Radius filtering at search time scopes results — no per-city separation needed.

---

## Pipeline Internals

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py pattern
import torch
from cosplace_utils import load_cosplace_model, get_descriptor

model = load_cosplace_model(device="cuda")  # or "mps" / "cpu"

# Extract 512-dim fingerprint from query image
descriptor = get_descriptor(model, image_path, device="cuda")

# Also extract flipped descriptor (catches reversed perspectives)
descriptor_flipped = get_descriptor(model, image_path, device="cuda", flip=True)

# Search index: cosine similarity + haversine radius filter
# Returns top 500-1000 candidate panorama IDs
```

### Stage 2 — Local Feature Matching (ALIKED/DISK + LightGlue)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

# CUDA → ALIKED; MPS/CPU → DISK
if torch.cuda.is_available():
    extractor = ALIKED(max_num_keypoints=1024).eval().cuda()
else:
    extractor = DISK(max_num_keypoints=768).eval()

matcher = LightGlue(features="aliked").eval()  # or "disk"

# Load images
query = load_image("query.jpg")
candidate = load_image("candidate_crop.jpg")

# Extract keypoints
feats0 = extractor.extract(query.unsqueeze(0))
feats1 = extractor.extract(candidate.unsqueeze(0))

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
matches01 = rbd(matches01)  # remove batch dimension

matches = matches01["matches"]           # [N, 2] matched keypoint indices
scores  = matches01["matching_scores0"]  # confidence per match
```

### Stage 3 — RANSAC Geometric Verification

```python
import cv2
import numpy as np

# After LightGlue matching
kpts0 = feats0["keypoints"][0][matches[:, 0]].cpu().numpy()
kpts1 = feats1["keypoints"][0][matches[:, 1]].cpu().numpy()

# RANSAC filters geometrically inconsistent matches
_, inlier_mask = cv2.findFundamentalMat(
    kpts0, kpts1,
    method=cv2.FM_RANSAC,
    ransacReprojThreshold=3.0,
    confidence=0.999
)

inliers = inlier_mask.ravel().astype(bool)
num_inliers = inliers.sum()
# Higher inlier count → better match
```

---

## Multi-FOV Crop Strategy

Netryx tests 3 fields of view per candidate heading to handle zoom mismatches:

```python
FOV_OPTIONS = [70, 90, 110]  # degrees

for fov in FOV_OPTIONS:
    crop = extract_rectilinear_crop(panorama, heading=heading_deg, fov=fov)
    feats = extractor.extract(crop)
    matches = matcher({"image0": query_feats, "image1": feats})
    # Keep the FOV that yields the most inliers
```

---

## Heading Refinement

After initial match, tests ±45° heading offsets in 15° steps:

```python
best_heading = initial_heading
best_inliers = 0

for delta in range(-45, 46, 15):
    test_heading = (initial_heading + delta) % 360
    for fov in [70, 90, 110]:
        crop = extract_rectilinear_crop(panorama, heading=test_heading, fov=fov)
        inliers = count_inliers(query_feats, crop, extractor, matcher)
        if inliers > best_inliers:
            best_inliers = inliers
            best_heading = test_heading
```

---

## Ultra Mode

Enable for night shots, blurry images, or low-texture scenes.

**What it adds:**
1. **LoFTR** — detector-free dense matching (handles blur/low-contrast)
2. **Descriptor hopping** — if best match < 50 inliers, re-searches index using the matched panorama's clean descriptor
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

```python
# Ultra Mode requires kornia
import kornia.feature as KF

matcher_loftr = KF.LoFTR(pretrained="outdoor")

img0 = kornia.io.load_image("query.jpg", kornia.io.ImageLoadType.GRAY32).unsqueeze(0)
img1 = kornia.io.load_image("candidate.jpg", kornia.io.ImageLoadType.GRAY32).unsqueeze(0)

input_dict = {"image0": img0, "image1": img1}
with torch.inference_mode():
    correspondences = matcher_loftr(input_dict)

kpts0 = correspondences["keypoints0"]
kpts1 = correspondences["keypoints1"]
conf  = correspondences["confidence"]
```

---

## Confidence Scoring

```python
# Spatial consensus: cluster top matches into 50m cells
# Prefer clusters over single high-inlier outliers

# Uniqueness ratio: how much better is the top match?
uniqueness = best_inliers / second_best_inliers  # from different location
# uniqueness > 1.5 → high confidence
# uniqueness < 1.1 → ambiguous, low confidence
```

---

## Standalone Index Builder (Large Datasets)

For large areas (5km+), use `build_index.py` instead of the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300 \
  --device cuda
```

This builds `cosplace_parts/*.npz` incrementally. After building parts, the GUI auto-compiles them into `index/cosplace_descriptors.npy` + `index/metadata.npz` on first search.

---

## Common Patterns

### Index multiple cities, search any of them

```python
# Index Paris (center: 48.8566, 2.3522, radius: 5km) → saved to shared index
# Index London (center: 51.5074, -0.1278, radius: 5km) → appended to same index

# Search Paris only:
search(query_image, center_lat=48.8566, center_lon=2.3522, radius_km=5.0)

# Search London only:
search(query_image, center_lat=51.5074, center_lon=-0.1278, radius_km=5.0)
```

### Interpreting results

- **>100 inliers** + **uniqueness >1.5** → high confidence match, likely <50m error
- **50–100 inliers** → moderate confidence, try Ultra Mode
- **<50 inliers** → low confidence, consider expanding search radius or enabling Ultra Mode

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| GUI appears blank on macOS | `brew install python-tk@3.11` |
| `ModuleNotFoundError: lightglue` | `pip install git+https://github.com/cvg/LightGlue.git` |
| CUDA out of memory | Reduce `max_num_keypoints` (ALIKED: 512, DISK: 512) |
| Indexing stops mid-run | Safe to restart — resumes from last saved chunk in `cosplace_parts/` |
| Low inlier counts everywhere | Try Ultra Mode; widen FOV range; ensure query is street-level (not aerial) |
| AI Coarse mode fails | Check `GEMINI_API_KEY` env var is set; falls back to Manual gracefully |
| MPS device errors on Mac | Ensure PyTorch ≥ 2.0; `pip install torch torchvision --upgrade` |
| `index/` directory missing | Run Create Index first; or run `build_index.py` then restart GUI |

---

## Model Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (512-dim) | Global descriptor / retrieval | All |
| ALIKED (1024 kp) | Local keypoints | CUDA only |
| DISK (768 kp) | Local keypoints | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | All (needs `kornia`) |
