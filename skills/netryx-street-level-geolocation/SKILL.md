```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace, ALIKED/DISK, and LightGlue to identify GPS coordinates from any street photo.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - visual place recognition locally
  - index street view panoramas
  - run netryx geolocation
  - match street photo to coordinates
  - open source geoguessr tool
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas into a searchable index, then uses a three-stage computer vision pipeline (global retrieval → local feature matching → refinement) to find the exact location. Sub-50m accuracy. No cloud dependency after indexing.

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

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Gemini API Key (optional — AI Coarse region guessing)

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

All indexing, searching, and configuration happens inside the GUI. No separate server needed.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim visual fingerprints
    └── metadata.npz               # Lat/lng, heading, panorama IDs
```

---

## Core Workflow

### Step 1 — Create an Index

In the GUI, select **Create** mode and enter:

| Field | Description | Default |
|-------|-------------|---------|
| Center latitude | Center of area to index | — |
| Center longitude | Center of area to index | — |
| Radius (km) | Coverage radius | 1 km |
| Grid resolution | Panorama density | 300 |

Indexing crawls Street View panoramas, extracts CosPlace 512-dim descriptors, and saves them to `cosplace_parts/`. The process is resumable — it picks up where it left off if interrupted.

**Time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

### Step 2 — Search

Select **Search** mode, upload a street photo, then choose:

- **Manual**: provide approximate center coordinates + radius
- **AI Coarse**: Gemini analyzes visual clues (signs, architecture) to guess the region (requires `GEMINI_API_KEY`)

Click **Run Search → Start Full Search**. Results appear on the map with a confidence score.

---

## Pipeline Internals

### Stage 1: Global Retrieval (CosPlace)

```python
# cosplace_utils.py pattern
from cosplace_utils import load_cosplace_model, extract_descriptor
import numpy as np

model = load_cosplace_model(device="cuda")  # or "mps" / "cpu"

# Extract 512-dim fingerprint from query image
descriptor = extract_descriptor(model, "query.jpg", device="cuda")

# Also extract flipped version (catches reversed perspectives)
descriptor_flipped = extract_descriptor(model, "query.jpg", device="cuda", flip=True)

# Load pre-built index
descriptors = np.load("index/cosplace_descriptors.npy")       # shape: (N, 512)
metadata    = np.load("index/metadata.npz", allow_pickle=True)

# Cosine similarity search (single matrix multiply — sub-1s regardless of index size)
scores = descriptors @ descriptor.T          # shape: (N,)
top_k  = np.argsort(scores)[::-1][:500]     # top 500 candidates

# Radius filter using haversine distance
from math import radians, sin, cos, sqrt, atan2

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000  # metres
    phi1, phi2 = radians(lat1), radians(lat2)
    dphi       = radians(lat2 - lat1)
    dlambda    = radians(lon2 - lon1)
    a = sin(dphi/2)**2 + cos(phi1)*cos(phi2)*sin(dlambda/2)**2
    return 2 * R * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522
radius_m = 2000

lats = metadata["lats"]
lons = metadata["lons"]
mask = np.array([haversine(center_lat, center_lon, lats[i], lons[i]) < radius_m
                 for i in top_k])
candidates = top_k[mask]
```

### Stage 2: Local Feature Matching (ALIKED/DISK + LightGlue)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps"  if torch.backends.mps.is_available() else "cpu")

# ALIKED on CUDA, DISK on MPS/CPU
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def match_images(img0_path, img1_path):
    img0 = load_image(img0_path).to(device)
    img1 = load_image(img1_path).to(device)

    feats0 = extractor.extract(img0)
    feats1 = extractor.extract(img1)

    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0, feats1, matches01 = [rbd(x) for x in (feats0, feats1, matches01)]

    matched_kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
    matched_kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
    return matched_kpts0, matched_kpts1, matches01["matching_scores0"]

# RANSAC geometric verification
import cv2

def ransac_inliers(kpts0, kpts1, threshold=4.0):
    if len(kpts0) < 4:
        return 0
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, mask = cv2.findFundamentalMat(pts0, pts1,
                                      cv2.FM_RANSAC,
                                      ransacReprojThreshold=threshold)
    return int(mask.sum()) if mask is not None else 0
```

### Stage 3: Heading Refinement + Spatial Consensus

```python
import numpy as np
from collections import defaultdict

def cluster_candidates(candidates, lats, lons, cell_size_m=50):
    """Group candidates into 50m spatial cells; prefer clusters over outliers."""
    clusters = defaultdict(list)
    for idx, inliers in candidates:
        # Quantize to cell
        cell_lat = round(lats[idx] / (cell_size_m / 111320), 4)
        cell_lon = round(lons[idx] / (cell_size_m / (111320 * np.cos(np.radians(lats[idx])))), 4)
        clusters[(cell_lat, cell_lon)].append((idx, inliers))

    # Score each cluster by total inliers
    best_cluster = max(clusters.values(), key=lambda c: sum(i for _, i in c))
    best_match   = max(best_cluster, key=lambda x: x[1])
    return best_match

# Confidence scoring
def confidence_score(best_inliers, runner_up_inliers, cluster_size):
    uniqueness = best_inliers / max(runner_up_inliers, 1)
    consensus  = min(cluster_size / 3, 1.0)         # saturates at 3+ cluster matches
    return min((uniqueness * 0.6 + consensus * 0.4), 1.0)
```

### Ultra Mode: LoFTR Dense Matching

```python
import kornia.feature as KF
import kornia
import torch

def loftr_match(img0_path, img1_path, device):
    matcher = KF.LoFTR(pretrained="outdoor").eval().to(device)

    img0 = kornia.io.load_image(img0_path, kornia.io.ImageLoadType.GRAY32).unsqueeze(0).to(device)
    img1 = kornia.io.load_image(img1_path, kornia.io.ImageLoadType.GRAY32).unsqueeze(0).to(device)

    # Resize to multiple of 8 (LoFTR requirement)
    img0 = kornia.geometry.resize(img0, (480, 640))
    img1 = kornia.geometry.resize(img1, (480, 640))

    with torch.no_grad():
        input_dict = {"image0": img0, "image1": img1}
        correspondences = matcher(input_dict)

    kpts0 = correspondences["keypoints0"].cpu().numpy()
    kpts1 = correspondences["keypoints1"].cpu().numpy()
    conf  = correspondences["confidence"].cpu().numpy()

    # Filter by confidence
    mask  = conf > 0.7
    return kpts0[mask], kpts1[mask]
```

### Ultra Mode: Descriptor Hopping

```python
# If best match has < 50 inliers, re-search from the matched panorama's clean image
def descriptor_hop(matched_panorama_path, index_descriptors, index_metadata,
                   center_lat, center_lon, radius_m, model, device):
    """Extract descriptor from clean matched panorama, re-search index."""
    clean_descriptor = extract_descriptor(model, matched_panorama_path, device=device)
    scores    = index_descriptors @ clean_descriptor.T
    top_k     = np.argsort(scores)[::-1][:500]
    # apply radius filter, return new candidates
    return top_k
```

---

## Building a Large Index (CLI)

For areas larger than ~5km, use the standalone builder instead of the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 10 \
  --resolution 300 \
  --output ./index
```

This runs without the GUI, is more memory-efficient, and supports parallel workers for large-scale indexing.

---

## Configuration Reference

| Parameter | Where | Description |
|-----------|-------|-------------|
| `GEMINI_API_KEY` | env var | Enables AI Coarse region guessing |
| Grid resolution | GUI / `build_index.py` | Panorama grid density (default 300, don't increase) |
| Radius (km) | GUI | Search/index radius from center point |
| Ultra Mode | GUI checkbox | Enables LoFTR + descriptor hopping + neighborhood expansion |
| Max keypoints | Code | ALIKED: 1024 (CUDA), DISK: 768 (MPS/CPU) |
| RANSAC threshold | Code | 4.0px default; increase for noisy/distorted images |
| Cluster cell size | Code | 50m default spatial consensus grid |
| Candidate pool | Code | 500 candidates Stage 1 → top 15 refined in Stage 3 |

---

## Multi-City Index

All cities live in one unified index. The radius filter at search time handles isolation:

```python
# Index Paris
# → cosplace_parts/ gets Paris embeddings appended

# Index Tokyo  
# → cosplace_parts/ gets Tokyo embeddings appended

# Search Paris only
# center=(48.8566, 2.3522), radius=5km → only Paris results returned

# Search Tokyo only
# center=(35.6762, 139.6503), radius=5km → only Tokyo results returned
```

No city labels, no separate files — coordinates + radius handle everything.

---

## Hardware Behavior

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|--------------|---------------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK (768 kp) |
| Recommended VRAM | 8GB+ | Unified memory 8GB+ | — |
| Stage 2 speed (300 candidates) | ~2 min | ~4 min | ~15+ min |
| LoFTR (Ultra Mode) | ✅ | ✅ | ✅ (slow) |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # replace 3.11 with your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory during Stage 2
Reduce `max_num_keypoints` in the extractor:
```python
extractor = ALIKED(max_num_keypoints=512).eval().to(device)  # was 1024
```

### Low inlier count on valid match (<20 inliers)
- Enable **Ultra Mode** (adds LoFTR + descriptor hopping)
- Try increasing FOV range — the query may have a different zoom level than the indexed crop
- Check that the photo is street-level (aerial, indoor, and interior shots are not supported)

### Index build interrupted
Safe to restart — the builder checks `cosplace_parts/` for already-processed panoramas and skips them. Resume by running the same command/GUI settings.

### AI Coarse mode not working
```bash
# Verify key is exported
echo $GEMINI_API_KEY

# Re-export if missing
export GEMINI_API_KEY="your_key_here"
```

### CosPlace model not downloading
The model downloads automatically on first run (~500MB). Ensure internet access and sufficient disk space. Models cache to `~/.cache/torch/` by default.

---

## Common Patterns

### Batch geolocate a folder of images (scripted)

```python
import os
from pathlib import Path
from cosplace_utils import load_cosplace_model, extract_descriptor
import numpy as np

device = "cuda"  # or "mps" / "cpu"
model  = load_cosplace_model(device=device)

descriptors = np.load("index/cosplace_descriptors.npy")
metadata    = np.load("index/metadata.npz", allow_pickle=True)

images = list(Path("./query_images").glob("*.jpg"))

for img_path in images:
    desc   = extract_descriptor(model, str(img_path), device=device)
    scores = descriptors @ desc.T
    top_1  = np.argmax(scores)
    lat    = metadata["lats"][top_1]
    lon    = metadata["lons"][top_1]
    print(f"{img_path.name}: {lat:.6f}, {lon:.6f} (score={scores[top_1]:.3f})")
```

### Check index coverage

```python
import numpy as np

meta = np.load("index/metadata.npz", allow_pickle=True)
print(f"Total panoramas indexed: {len(meta['lats'])}")
print(f"Lat range: {meta['lats'].min():.4f} → {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} → {meta['lons'].max():.4f}")
```

### Merge two index parts manually

```python
import numpy as np

d1 = np.load("index_paris/cosplace_descriptors.npy")
d2 = np.load("index_tokyo/cosplace_descriptors.npy")
merged = np.vstack([d1, d2])
np.save("index_merged/cosplace_descriptors.npy", merged)

m1 = np.load("index_paris/metadata.npz",  allow_pickle=True)
m2 = np.load("index_tokyo/metadata.npz",  allow_pickle=True)
np.savez("index_merged/metadata.npz",
         lats=np.concatenate([m1["lats"], m2["lats"]]),
         lons=np.concatenate([m1["lons"], m2["lons"]]),
         headings=np.concatenate([m1["headings"], m2["headings"]]),
         panoids=np.concatenate([m1["panoids"],  m2["panoids"]]))
```
```
