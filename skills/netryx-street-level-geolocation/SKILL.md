```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for Netryx, an open-source local-first street-level geolocation engine using CosPlace, ALIKED/DISK, and LightGlue to identify GPS coordinates from street photos.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - reverse geolocate image
  - build netryx index
  - run netryx search
  - osint geolocation tool
  - identify location from street photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas into a searchable index, then matches query images against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). Achieves sub-50m accuracy with no internet presence required for matched locations.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue matcher
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | Best performance; uses ALIKED (1024 keypoints) |
| Apple Silicon | MPS | M1/M2/M3/M4; uses DISK (768 keypoints) |
| CPU | CPU | Works but slow |

### Optional: Gemini API for AI Coarse mode

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

### 1. Create an Index

Index a geographic area before searching. The index stores CosPlace 512-dim fingerprints of all crawled street-view panoramas.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon, search radius (km), and grid resolution (default: 300)
3. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

Indexing is **resumable** — if interrupted, it continues from where it left off.

### 2. Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes the image to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result appears on the map with GPS coordinates and confidence score

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace 512-dim descriptor extraction
    ├── Flipped descriptor extraction (catches reversed perspectives)
    │
    ▼
Index cosine similarity search → radius filter (haversine) → top 500 candidates
    │
    ▼
Download panorama tiles → stitch → rectilinear crop at heading
    │
    ├── Multi-FOV crops: 70°, 90°, 110°
    ├── ALIKED (CUDA) / DISK (MPS/CPU) keypoint extraction
    ├── LightGlue deep feature matching
    └── RANSAC geometric verification → inlier count
    │
    ▼
Heading refinement: ±45° offsets × 15° steps × 3 FOVs (top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    └── Confidence scoring (uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), written during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (stacked)
    └── metadata.npz               # lat, lon, heading, panoid per descriptor
```

---

## Key Code Patterns

### Load CosPlace and Extract a Descriptor

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-selects CUDA > MPS > CPU)
device = (
    torch.device("cuda") if torch.cuda.is_available()
    else torch.device("mps") if torch.backends.mps.is_available()
    else torch.device("cpu")
)
model = load_cosplace_model(device)

# Extract 512-dim descriptor from an image
img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)  # shape: (512,)

# Also extract flipped version for better recall
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = extract_descriptor(model, img_flipped, device)
```

### Search the Index Manually

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * asin(sqrt(a))

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz")
lats, lons = meta["lats"], meta["lons"]
panoids, headings = meta["panoids"], meta["headings"]

# Radius filter
center_lat, center_lon = 48.8566, 2.3522  # Paris
radius_km = 2.0

mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
])
filtered_desc = descriptors[mask]
filtered_idx = np.where(mask)[0]

# Cosine similarity search
query_norm = descriptor / (np.linalg.norm(descriptor) + 1e-8)
db_norms = filtered_desc / (np.linalg.norm(filtered_desc, axis=1, keepdims=True) + 1e-8)
scores = db_norms @ query_norm  # (M,)

# Top-K candidates
top_k = 500
top_local = np.argsort(scores)[::-1][:top_k]
top_global_idx = filtered_idx[top_local]
top_scores = scores[top_local]

for rank, (idx, score) in enumerate(zip(top_global_idx[:5], top_scores[:5])):
    print(f"Rank {rank+1}: lat={lats[idx]:.6f}, lon={lons[idx]:.6f}, "
          f"panoid={panoids[idx]}, heading={headings[idx]}, score={score:.4f}")
```

### LightGlue Feature Matching (ALIKED, CUDA)

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda")

# Initialize extractor and matcher
extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

# Load images as tensors [C, H, W] in [0,1]
image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

# Extract features
feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

kpts0 = feats0["keypoints"]
kpts1 = feats1["keypoints"]
matches = matches01["matches"]  # (M, 2) indices

matched_kpts0 = kpts0[matches[:, 0]]
matched_kpts1 = kpts1[matches[:, 1]]

print(f"Matched keypoints: {len(matches)}")
```

### DISK Extractor (MPS/CPU fallback)

```python
from lightglue import DISK

device = torch.device("mps")  # or "cpu"
extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher = LightGlue(features="disk").eval().to(device)
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_inliers(kpts0, kpts1):
    """
    Returns number of RANSAC inliers between two sets of matched keypoints.
    kpts0, kpts1: numpy arrays of shape (N, 2)
    """
    if len(kpts0) < 4:
        return 0
    _, mask = cv2.findHomography(
        kpts0, kpts1,
        cv2.RANSAC,
        ransacReprojThreshold=4.0,
        maxIters=2000,
        confidence=0.999
    )
    if mask is None:
        return 0
    return int(mask.sum())

# Usage after LightGlue matching
pts0 = matched_kpts0.cpu().numpy()
pts1 = matched_kpts1.cpu().numpy()
inliers = ransac_inliers(pts0, pts1)
print(f"Verified inliers: {inliers}")
```

### Spatial Consensus Clustering

```python
from collections import defaultdict

def cluster_candidates(candidates, cell_size_m=50):
    """
    candidates: list of dicts with keys 'lat', 'lon', 'inliers'
    Returns the cluster center with the most total inliers.
    """
    # ~50m per degree latitude ≈ 0.00045 degrees
    cell_deg = cell_size_m / 111_000

    clusters = defaultdict(list)
    for c in candidates:
        cell = (
            round(c["lat"] / cell_deg),
            round(c["lon"] / cell_deg)
        )
        clusters[cell].append(c)

    best_cell = max(clusters, key=lambda k: sum(c["inliers"] for c in clusters[k]))
    best_candidates = clusters[best_cell]
    best = max(best_candidates, key=lambda c: c["inliers"])
    return best, len(best_candidates)

result, cluster_size = cluster_candidates(top_candidates)
print(f"Best match: {result['lat']:.6f}, {result['lon']:.6f} "
      f"(cluster size: {cluster_size}, inliers: {result['inliers']})")
```

### Build Index from Standalone Script (Large Areas)

```bash
# For large-scale indexing, use build_index.py directly
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --grid-resolution 300
```

---

## Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `grid_resolution` | 300 | Sampling density for panorama crawl — don't change |
| `radius_km` | user-set | Search radius in km; filters index by haversine distance |
| Top-K candidates | 500–1000 | Number of CosPlace candidates passed to Stage 2 |
| Heading refinement offsets | ±45° @ 15° steps | Angles tested per top-15 candidate |
| Multi-FOV crops | 70°, 90°, 110° | Field-of-view variants per candidate crop |
| RANSAC threshold | 4.0 px | Reprojection error for inlier classification |
| Spatial cluster cell | 50 m | Cell size for geographic consensus clustering |
| Ultra neighborhood expand | 100 m | Radius for neighbor search in Ultra Mode |

---

## Ultra Mode

Enable Ultra Mode for difficult images (night, blur, low texture, heavy compression).

**Activates three additional strategies:**

1. **LoFTR dense matching** — detector-free matching; handles scenes where keypoints can't be detected reliably
2. **Descriptor hopping** — extracts a CosPlace descriptor from the *matched clean panorama* and re-searches the index, bypassing noise in the degraded query
3. **Neighborhood expansion** — searches all panoramas within 100m of the best match to catch off-by-one-node errors

```python
# Conceptual usage (controlled via GUI checkbox)
search_config = {
    "ultra_mode": True,       # enables LoFTR + hopping + expansion
    "loftr_enabled": True,    # requires: pip install kornia
    "descriptor_hop": True,
    "expand_radius_m": 100,
}
```

---

## Index Layout

The index is **unified across all indexed cities**. Radius filtering at search time handles geographic isolation — no per-city organization needed.

```
index/
├── cosplace_descriptors.npy   # float32, shape (N, 512)
└── metadata.npz
    ├── lats      # float64, shape (N,)
    ├── lons      # float64, shape (N,)
    ├── panoids   # str array, shape (N,)
    └── headings  # float32, shape (N,) — degrees 0–360
```

Raw chunks before compilation live in `cosplace_parts/*.npz` — auto-merged into the searchable index on each run.

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # match your Python version
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED: `ALIKED(max_num_keypoints=512)`
- Process fewer candidates per batch
- Check VRAM: `nvidia-smi`

### LightGlue import error
```bash
pip install git+https://github.com/cvg/LightGlue.git
# Must be installed from GitHub, not PyPI
```

### LoFTR not available (Ultra Mode)
```bash
pip install kornia
# kornia includes LoFTR implementation
```

### Low inlier count / wrong location
- Enable Ultra Mode for degraded images
- Increase search radius if approximate location is uncertain
- Ensure the query image is street-level (not aerial or indoor)
- Try AI Coarse mode if region is unknown (requires `GEMINI_API_KEY`)

### Indexing stalls or crashes
- Indexing is resumable — restart and it continues from the last saved chunk in `cosplace_parts/`
- Check disk space: large radius indexes can exceed 10GB
- Reduce radius or use `build_index.py` for more control over large jobs

### Descriptor shape mismatch
```python
# Verify index integrity
import numpy as np
desc = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz")
assert desc.shape[0] == len(meta["lats"]), "Index/metadata row count mismatch — rebuild index"
assert desc.shape[1] == 512, f"Expected 512-dim descriptors, got {desc.shape[1]}"
```

---

## Models Reference

| Model | Role | Paper |
|-------|------|-------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global place recognition descriptor | CVPR 2022 |
| [ALIKED](https://github.com/naver/alike) | Local keypoints — CUDA | IEEE TIP 2023 |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints — MPS/CPU | NeurIPS 2020 |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | ICCV 2023 |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra Mode) | CVPR 2021 |
```
