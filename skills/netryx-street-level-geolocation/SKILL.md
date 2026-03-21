---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a locally-hosted open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation locally
  - use netryx to locate an image
  - index street view panoramas
  - reverse geolocate a photograph
  - run netryx geolocation pipeline
  - find where a photo was taken
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls street-view panoramas, indexes them as 512-dim CosPlace fingerprints, then matches query images using ALIKED/DISK keypoints + LightGlue deep feature matching. Achieves sub-50m accuracy with no landmark recognition — works on arbitrary street corners.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must install from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### GPU Support
- **NVIDIA**: CUDA — ALIKED extractor (1024 keypoints)
- **Apple Silicon**: MPS — DISK extractor (768 keypoints)
- **CPU**: Works, significantly slower

### Optional: Gemini API for AI Coarse Mode
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### 1. Create an Index (required before searching)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of target area
3. Set radius (start: 0.5–1 km for testing, 5–10 km for production)
4. Set grid resolution (default: 300 — don't change)
5. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hrs       | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hrs      | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hrs     | ~7 GB      |

### 2. Search

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide center coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues to guess region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on a map

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace (512-dim descriptor) + flipped version
    ▼
Index Search (cosine similarity, radius-filtered) → Top 500 candidates
    │
    ▼
Download panoramas → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) / DISK (MPS/CPU) keypoint extraction
    ├── LightGlue deep feature matching
    └── RANSAC geometric verification
    │
    ▼
Heading Refinement (±45°, 15° steps, top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    └── Confidence scoring (clustering + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), written during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (matrix)
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Key Code Patterns

### Extract a CosPlace Descriptor Programmatically

```python
# cosplace_utils.py provides the model loader
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model(device="cuda")  # or "mps" or "cpu"

# Extract 512-dim fingerprint from any image
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device="cuda")
# descriptor.shape → (512,)
```

### Load and Search an Existing Index

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,)
lons = meta["lons"]       # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

# Cosine similarity search
query_desc = descriptor.reshape(1, -1)  # (1, 512)
sims = cosine_similarity(query_desc, descriptors)[0]  # (N,)

# Radius filter using haversine
def haversine_km(lat1, lon1, lat2, lon2):
    from math import radians, sin, cos, sqrt, atan2
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522   # Paris
radius_km = 2.0

mask = np.array([
    haversine_km(center_lat, center_lon, lat, lon) <= radius_km
    for lat, lon in zip(lats, lons)
])

# Apply mask and get top candidates
filtered_sims = np.where(mask, sims, -1.0)
top_indices = np.argsort(filtered_sims)[::-1][:500]

for idx in top_indices[:5]:
    print(f"Similarity: {sims[idx]:.4f} | "
          f"Lat: {lats[idx]:.6f} | Lon: {lons[idx]:.6f} | "
          f"Heading: {headings[idx]}° | PanoID: {panoids[idx]}")
```

### Run LightGlue Feature Matching

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

# Load images
image0 = load_image("query_photo.jpg").to(device)       # your photo
image1 = load_image("candidate_panorama.jpg").to(device) # street view crop

# Extract keypoints
feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
matches01 = rbd(matches01)  # remove batch dimension

matches = matches01["matches"]           # (M, 2) matched keypoint indices
scores = matches01["matching_scores0"]   # (M,) match confidence scores

print(f"Matched keypoints: {len(matches)}")

# RANSAC geometric verification (via OpenCV)
import cv2
import numpy as np

kp0 = feats0["keypoints"][0][matches[:, 0]].cpu().numpy()
kp1 = feats1["keypoints"][0][matches[:, 1]].cpu().numpy()

if len(kp0) >= 4:
    _, inliers = cv2.findHomography(kp0, kp1, cv2.RANSAC, ransacReprojThreshold=8.0)
    num_inliers = int(inliers.sum()) if inliers is not None else 0
    print(f"RANSAC inliers: {num_inliers}")
```

### Ultra Mode — LoFTR Dense Matching

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Load LoFTR (detector-free — handles blur/low-texture)
matcher = kornia.feature.LoFTR(pretrained="outdoor").to(device).eval()

def load_gray_tensor(path, size=(480, 640)):
    img = cv2.imread(path, cv2.IMREAD_GRAYSCALE)
    img = cv2.resize(img, (size[1], size[0]))
    return torch.tensor(img / 255.0, dtype=torch.float32)[None, None].to(device)

img0 = load_gray_tensor("query_photo.jpg")
img1 = load_gray_tensor("candidate_crop.jpg")

with torch.no_grad():
    result = matcher({"image0": img0, "image1": img1})

kp0 = result["keypoints0"].cpu().numpy()
kp1 = result["keypoints1"].cpu().numpy()
conf = result["confidence"].cpu().numpy()

# Filter by confidence
high_conf = conf > 0.5
kp0_conf = kp0[high_conf]
kp1_conf = kp1[high_conf]

print(f"LoFTR matches (conf > 0.5): {len(kp0_conf)}")

# RANSAC
if len(kp0_conf) >= 4:
    _, inliers = cv2.findHomography(kp0_conf, kp1_conf, cv2.RANSAC, 8.0)
    print(f"LoFTR RANSAC inliers: {int(inliers.sum()) if inliers is not None else 0}")
```

### Spatial Consensus Clustering

```python
import numpy as np
from collections import defaultdict

def cluster_candidates(candidates, cell_size_m=50):
    """
    candidates: list of dicts with keys: lat, lon, inliers
    Returns best cluster center.
    """
    # Approx degrees per meter
    deg_per_m_lat = 1 / 111320
    deg_per_m_lon = lambda lat: 1 / (111320 * np.cos(np.radians(lat)))

    clusters = defaultdict(list)
    for c in candidates:
        cell_lat = round(c["lat"] / (cell_size_m * deg_per_m_lat))
        cell_lon = round(c["lon"] / (cell_size_m * deg_per_m_lon(c["lat"])))
        clusters[(cell_lat, cell_lon)].append(c)

    # Pick cluster with highest total inliers
    best_cluster = max(clusters.values(), key=lambda g: sum(x["inliers"] for x in g))
    best = max(best_cluster, key=lambda x: x["inliers"])
    return best["lat"], best["lon"], len(best_cluster)

# Example usage
candidates = [
    {"lat": 48.8566, "lon": 2.3522, "inliers": 120},
    {"lat": 48.8567, "lon": 2.3521, "inliers": 95},
    {"lat": 48.8600, "lon": 2.3600, "inliers": 140},  # outlier location
]
lat, lon, cluster_size = cluster_candidates(candidates)
print(f"Best match: {lat:.6f}, {lon:.6f} (cluster size: {cluster_size})")
```

---

## Index Architecture Notes

- All cities/regions share **one unified index** — no per-city selection needed
- The radius filter at search time handles geographic scoping
- Index can grow incrementally: index Paris, later add London, search works across both
- `cosplace_parts/*.npz` = raw chunk files written during crawling
- `index/cosplace_descriptors.npy` + `index/metadata.npz` = compiled searchable form (auto-built from parts)

```
# Index files layout
cosplace_parts/
  part_0000.npz    # chunk: descriptors + metadata
  part_0001.npz
  ...
index/
  cosplace_descriptors.npy   # stacked (N, 512) float32
  metadata.npz               # lats, lons, headings, panoids arrays
```

---

## Configuration Reference

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Panorama sampling density — don't change |
| Top candidates | 500–1000 | From CosPlace retrieval stage |
| Heading refinement steps | ±45° at 15° | Tests 7 headings × 3 FOVs = 21 crops per candidate |
| FOVs tested | 70°, 90°, 110° | Handles zoom mismatch between query and index |
| RANSAC threshold | 8.0 px | Reprojection error tolerance |
| Spatial cluster cell | 50 m | Consensus grouping granularity |
| Ultra Mode LoFTR conf | 0.5 | Minimum match confidence |
| Descriptor hop threshold | 50 inliers | Below this → trigger descriptor hopping |
| Neighborhood expansion | 100 m | Searched around best match in Ultra Mode |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # match your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory
```python
# Reduce keypoints in extractor
extractor = ALIKED(max_num_keypoints=512).eval().to(device)  # default 1024
```

### MPS (Apple Silicon) errors with ALIKED
```python
# ALIKED has MPS incompatibilities — use DISK instead
extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher = LightGlue(features="disk").eval().to(device)
```

### Indexing interrupted / partial index
Safe to re-run — `cosplace_parts/` accumulates incrementally and the build step deduplicates by panorama ID.

### Low inlier count / wrong location
1. Enable **Ultra Mode** (LoFTR + descriptor hopping + neighborhood expansion)
2. Increase candidate pool (search with larger radius if region is uncertain)
3. Check that the query image is street-level (not aerial, not interior)
4. Try **AI Coarse** mode if region is completely unknown

### LoFTR not available
```bash
pip install kornia
# Verify:
python -c "import kornia; print(kornia.__version__)"
```

### Slow search on CPU
Expected — feature matching is GPU-bound. Use the batch size wisely:
- Reduce top-K candidates from 500 to 100 for faster (less accurate) results
- Disable heading refinement for speed (modify `test_super.py`)

---

## Models Reference

| Model | Purpose | Source |
|-------|---------|--------|
| CosPlace | Global 512-dim visual fingerprint | [github.com/gmberton/cosplace](https://github.com/gmberton/cosplace) |
| ALIKED | Local keypoints — CUDA | [github.com/naver/alike](https://github.com/naver/alike) |
| DISK | Local keypoints — MPS/CPU | [github.com/cvlab-epfl/disk](https://github.com/cvlab-epfl/disk) |
| LightGlue | Deep feature matcher | [github.com/cvg/LightGlue](https://github.com/cvg/LightGlue) |
| LoFTR | Detector-free dense matcher (Ultra) | via `kornia.feature.LoFTR` |
