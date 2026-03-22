```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace, ALIKED/DISK, and LightGlue to identify GPS coordinates from any street photo
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - identify location from photo
  - build a street view index
  - run netryx geolocation
  - osint image geolocation
  - match street photo to coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable index of visual fingerprints, and matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and spatial refinement.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required — LightGlue must be installed from source
pip install git+https://github.com/cvg/LightGlue.git

# Optional — enables Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Optional: Gemini API key for AI Coarse location guessing

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

The GUI is the primary interface. It handles indexing, searching, and result visualization.

---

## Core Workflow

### 1. Create an Index

Index a geographic area before searching. The indexer crawls street-view panoramas and stores 512-dim CosPlace fingerprints.

**GUI steps:**
1. Select **Create** mode
2. Enter center lat/lon
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time and storage estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is incremental — safe to interrupt and resume.

### 2. Search

**GUI steps:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose **Manual** (enter lat/lon + radius) or **AI Coarse** (Gemini guesses region)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### 3. Ultra Mode

Enable the **Ultra Mode** checkbox for difficult images (night, blur, low texture). Adds:
- LoFTR detector-free dense matching
- Descriptor hopping (re-search from matched panorama's clean descriptor)
- Neighborhood expansion (searches panoramas within 100m of best match)

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├─ CosPlace descriptor (512-dim)
    ├─ Flipped descriptor (catches reversed perspectives)
    │
    ▼
Index cosine similarity search (radius-filtered via haversine)
    │
    └─ Top 500–1000 candidates
    │
    ▼
Download panoramas (8 tiles, stitched) → Rectilinear crops at 3 FOVs (70°/90°/110°)
    │
    ├─ ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    ├─ LightGlue deep feature matching
    └─ RANSAC geometric verification
    │
    ▼
Heading refinement: ±45° at 15° steps, top 15 candidates
    │
    ├─ Spatial consensus clustering (50m cells)
    └─ Confidence scoring (cluster density + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Key Code Patterns

### Extract a CosPlace descriptor from an image

```python
from cosplace_utils import get_cosplace_model, extract_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = get_cosplace_model(device=device)

img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device=device)
# descriptor.shape == (512,)
```

### Build index from existing cosplace_parts chunks

```python
# Run standalone index builder for large datasets
python build_index.py
# Reads cosplace_parts/*.npz → writes index/cosplace_descriptors.npy + index/metadata.npz
```

### Cosine similarity search with radius filter (manual usage)

```python
import numpy as np

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]

# Radius filter
center_lat, center_lon = 48.8566, 2.3522   # Paris
radius_km = 2.0
distances = haversine_km(center_lat, center_lon, lats, lons)
mask = distances <= radius_km

# Cosine similarity search
query_desc = descriptor / np.linalg.norm(descriptor)
filtered_descs = descriptors[mask]
filtered_descs_normed = filtered_descs / np.linalg.norm(filtered_descs, axis=1, keepdims=True)
similarities = filtered_descs_normed @ query_desc
top_k = np.argsort(similarities)[::-1][:500]

# Map back to full index
filtered_indices = np.where(mask)[0]
top_candidates = filtered_indices[top_k]
```

### Feature matching with LightGlue (ALIKED on CUDA)

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(kpts0)}")
```

### Feature matching with DISK on MPS (Mac)

```python
from lightglue import LightGlue, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("mps")

extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher = LightGlue(features="disk").eval().to(device)

image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

result = matcher({"image0": feats0, "image1": feats1})
result = rbd(result)
print(f"Inliers after LightGlue: {result['matches'].shape[0]}")
```

### RANSAC verification (inlier count = confidence)

```python
import cv2
import numpy as np

def ransac_inliers(kpts0_np, kpts1_np, threshold=4.0):
    """Returns number of geometrically consistent matches."""
    if len(kpts0_np) < 4:
        return 0
    _, mask = cv2.findFundamentalMat(
        kpts0_np, kpts1_np,
        cv2.FM_RANSAC,
        ransacReprojThreshold=threshold,
        confidence=0.99
    )
    if mask is None:
        return 0
    return int(mask.sum())

inliers = ransac_inliers(
    kpts0.cpu().numpy(),
    kpts1.cpu().numpy()
)
print(f"RANSAC inliers: {inliers}")
# >50 inliers = confident match; >100 = strong match
```

### LoFTR dense matching (Ultra Mode)

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda")
loftr = kornia.feature.LoFTR(pretrained="outdoor").eval().to(device)

def load_gray_tensor(path, size=(640, 480)):
    img = cv2.imread(path, cv2.IMREAD_GRAYSCALE)
    img = cv2.resize(img, size)
    return torch.from_numpy(img).float()[None, None] / 255.0

img0 = load_gray_tensor("query.jpg").to(device)
img1 = load_gray_tensor("candidate.jpg").to(device)

with torch.no_grad():
    output = loftr({"image0": img0, "image1": img1})

kpts0 = output["keypoints0"].cpu().numpy()
kpts1 = output["keypoints1"].cpu().numpy()
confidence = output["confidence"].cpu().numpy()

# Filter by confidence
high_conf = confidence > 0.5
inliers = ransac_inliers(kpts0[high_conf], kpts1[high_conf])
print(f"LoFTR inliers: {inliers}")
```

---

## Configuration Reference

| Parameter | Where | Notes |
|-----------|-------|-------|
| Center lat/lon | GUI Create mode | Starting point for panorama crawl |
| Radius (km) | GUI Create/Search | `0.5–1` for testing, `5–10` for production |
| Grid resolution | GUI Create mode | Default `300` — controls density of crawl grid points |
| Search candidates | Code (`test_super.py`) | Top 500–1000 from CosPlace retrieval |
| Heading refinement steps | Code | ±45° at 15° increments, top 15 candidates |
| Spatial cluster cell size | Code | 50m cells for consensus |
| RANSAC threshold | Code | 4.0 pixels reprojection error |
| Max keypoints ALIKED | Code | 1024 (CUDA) |
| Max keypoints DISK | Code | 768 (MPS/CPU) |
| `GEMINI_API_KEY` | Environment variable | Required only for AI Coarse mode |

---

## Hardware & Device Selection

| Hardware | Feature Extractor | Speed |
|----------|------------------|-------|
| NVIDIA GPU (CUDA) | ALIKED (1024 kp) | Fastest |
| Apple Silicon (MPS) | DISK (768 kp) | Fast |
| CPU only | DISK | Slow (avoid for large searches) |

The pipeline auto-detects device:
```python
if torch.cuda.is_available():
    device = torch.device("cuda")
elif torch.backends.mps.is_available():
    device = torch.device("mps")
else:
    device = torch.device("cpu")
```

---

## Index Management

All areas share one unified index. The radius filter in search mode scopes results automatically — no per-city indexes needed.

```
# Index layout
index/
├── cosplace_descriptors.npy   # float32 (N, 512) — all descriptors
└── metadata.npz               # lats, lons, headings, panoids

# Raw chunks (written incrementally during indexing)
cosplace_parts/
├── part_0000.npz
├── part_0001.npz
└── ...
```

To rebuild the compiled index from parts (e.g. after adding a new area):
```bash
python build_index.py
```

---

## Common Patterns

### Pattern: Add a new city to existing index

1. Launch GUI → **Create** mode
2. Enter new city center coordinates
3. Set radius and click **Create Index**
4. New panoramas append to `cosplace_parts/`
5. Run `python build_index.py` to recompile (or let GUI auto-build)
6. Search any city by setting appropriate center + radius

### Pattern: Headless batch geolocation

```python
# Pseudocode — adapt from test_super.py internals
import numpy as np
from cosplace_utils import get_cosplace_model, extract_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = get_cosplace_model(device=device)

images = ["img1.jpg", "img2.jpg", "img3.jpg"]
center = (48.8566, 2.3522)   # Paris
radius_km = 3.0

for path in images:
    img = Image.open(path).convert("RGB")
    desc = extract_descriptor(model, img, device=device)
    # Pass desc + center + radius into the search pipeline from test_super.py
    # Extract run_search() or equivalent function for headless use
```

### Pattern: Evaluate match confidence

```python
def confidence_label(inliers: int) -> str:
    if inliers >= 100:
        return "HIGH"
    elif inliers >= 50:
        return "MEDIUM"
    elif inliers >= 20:
        return "LOW"
    else:
        return "NO_MATCH"
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your exact Python version
```

### `ImportError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### `kornia` not found (Ultra Mode disabled)
```bash
pip install kornia
```

### CUDA out of memory
- Reduce `max_num_keypoints` for ALIKED (try 512 instead of 1024)
- Reduce candidate count in the search pipeline
- Ensure no other processes are using GPU memory

### MPS errors on Apple Silicon
- Ensure macOS 12.3+ and PyTorch ≥ 2.0
- If MPS is unstable, force CPU: `export PYTORCH_ENABLE_MPS_FALLBACK=1`

### Index search returns no results
- Confirm the search center + radius overlaps your indexed area
- Verify `index/cosplace_descriptors.npy` exists (run `python build_index.py` if missing)
- Check `cosplace_parts/` is non-empty — if empty, indexing did not complete

### Very low inlier counts on valid images
- Enable **Ultra Mode** (LoFTR handles blur/night/low-texture)
- Try a wider search radius — the correct panorama may be just outside the area
- Verify the query image is street-level (aerial or indoor images will not match)

### Indexing interrupted mid-run
Safe to restart — indexing is incremental. The crawler skips already-processed grid points.
```
