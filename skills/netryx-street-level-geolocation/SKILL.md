```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - netryx geolocation
  - index street view panoramas
  - locate where a photo was taken
  - open source geolocation engine
  - visual place recognition coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images against that index using a three-stage computer vision pipeline: CosPlace (global retrieval) → ALIKED/DISK + LightGlue (geometric verification) → spatial refinement. Sub-50m accuracy, no landmark recognition needed, runs entirely on your hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (install from source)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR dense matching for Ultra Mode
pip install kornia
```

### Environment Variables

```bash
# Optional — only needed for AI Coarse blind geolocation via Gemini
export GEMINI_API_KEY="your_key_here"
```

### macOS tkinter Fix

If the GUI renders blank on macOS:

```bash
brew install python-tk@3.11   # match your Python version
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (index an area) and **Search** (geolocate a photo).

---

## Core Workflow

### Step 1 — Build an Index

Index a geographic area before searching. The indexer crawls Street View panoramas, extracts CosPlace descriptors, and saves them to `cosplace_parts/`.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon of the target area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is **resumable** — if interrupted, it continues from the last saved checkpoint on next run.

**Index file layout:**

```
cosplace_parts/          # Raw per-chunk .npz files written during indexing
index/
├── cosplace_descriptors.npy   # All 512-dim CosPlace vectors (N × 512 float32)
└── metadata.npz               # lat, lon, heading, panoid per descriptor
```

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual** — enter approximate center coordinates + radius
   - **AI Coarse** — Gemini infers region from visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on a live map

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace → 512-dim descriptor
    ├── Flipped CosPlace → 512-dim descriptor
    │
    ▼
Index cosine similarity search (radius-filtered via haversine)
    │
    └── Top 500–1000 candidates
    │
    ▼
Per candidate:
    ├── Download panorama (8 GSV tiles, stitched)
    ├── Crop at indexed heading
    ├── Multi-FOV crops: 70°, 90°, 110°
    ├── ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    └── LightGlue matching → RANSAC → inlier count
    │
    ▼
Refinement:
    ├── Heading sweep ±45° @ 15° steps × 3 FOVs (top 15 candidates)
    ├── Spatial consensus clustering (50m cells)
    └── Confidence scoring (cluster density + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable **Ultra Mode** for degraded images (night, blur, low texture, fog). Adds three enhancements:

| Enhancement | What it does |
|-------------|-------------|
| **LoFTR** | Detector-free dense matching — works without keypoints on blurry/dark images |
| **Descriptor hopping** | If best match < 50 inliers, extracts a fresh CosPlace descriptor from the matched panorama and re-searches |
| **Neighborhood expansion** | Searches all panoramas within 100m of the best match candidate |

Enable via the **Ultra Mode** checkbox in the GUI before running a search.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `test_super.py` | Main application — GUI, indexing orchestration, search pipeline |
| `cosplace_utils.py` | CosPlace model loading and descriptor extraction utilities |
| `build_index.py` | Standalone high-performance index builder for large datasets |
| `requirements.txt` | All Python dependencies |

---

## Code Examples

### Extract a CosPlace Descriptor Programmatically

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model, transform = load_cosplace_model(device="cuda")  # or "mps", "cpu"

# Extract descriptor from an image file
img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, transform, img, device="cuda")
# descriptor.shape → (512,) float32 numpy array
```

### Search the Index Directly

```python
import numpy as np

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz")
lats, lons, headings, panoids = meta["lats"], meta["lons"], meta["headings"], meta["panoids"]

# Haversine radius filter
def haversine_mask(lats, lons, center_lat, center_lon, radius_km):
    R = 6371.0
    dlat = np.radians(lats - center_lat)
    dlon = np.radians(lons - center_lon)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
    return 2 * R * np.arcsin(np.sqrt(a)) <= radius_km

# Filter to Paris, 2km radius
mask = haversine_mask(lats, lons, 48.8566, 2.3522, radius_km=2.0)
local_descriptors = descriptors[mask]
local_meta_idx = np.where(mask)[0]

# Cosine similarity search
query_desc = descriptor / np.linalg.norm(descriptor)          # normalize query
db_norms = local_descriptors / np.linalg.norm(local_descriptors, axis=1, keepdims=True)
scores = db_norms @ query_desc                                 # (M,) cosine similarities

# Top 10 candidates
top_k = np.argsort(scores)[::-1][:10]
for rank, idx in enumerate(top_k):
    global_idx = local_meta_idx[idx]
    print(f"Rank {rank+1}: lat={lats[global_idx]:.6f}, lon={lons[global_idx]:.6f}, "
          f"score={scores[idx]:.4f}, panoid={panoids[global_idx]}")
```

### Run LightGlue Matching Between Two Images

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

img0 = load_image("query.jpg").to(device)
img1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(img0)
feats1 = extractor.extract(img1)

matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

matched_kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
matched_kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(matched_kpts0)}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_inliers(kpts0, kpts1, threshold=3.0):
    """Returns inlier count from RANSAC fundamental matrix estimation."""
    if len(kpts0) < 8:
        return 0
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, mask = cv2.findFundamentalMat(
        pts0, pts1,
        method=cv2.FM_RANSAC,
        ransacReprojThreshold=threshold,
        confidence=0.999
    )
    if mask is None:
        return 0
    return int(mask.sum())

inliers = ransac_inliers(matched_kpts0, matched_kpts1)
print(f"Geometric inliers: {inliers}")
```

### Use DISK on MPS (Mac) Instead of ALIKED

```python
from lightglue import DISK

device = torch.device("mps")  # Apple Silicon
extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher = LightGlue(features="disk").eval().to(device)
```

### LoFTR Dense Matching (Ultra Mode)

```python
import kornia
from kornia.feature import LoFTR
import torch
import cv2

device = torch.device("cuda")
loftr = LoFTR(pretrained="outdoor").eval().to(device)

def loftr_match(img0_path, img1_path, device):
    img0 = cv2.imread(img0_path, cv2.IMREAD_GRAYSCALE)
    img1 = cv2.imread(img1_path, cv2.IMREAD_GRAYSCALE)

    t0 = torch.from_numpy(img0).float()[None, None] / 255.0
    t1 = torch.from_numpy(img1).float()[None, None] / 255.0

    with torch.no_grad():
        out = loftr({"image0": t0.to(device), "image1": t1.to(device)})

    kpts0 = out["keypoints0"].cpu().numpy()
    kpts1 = out["keypoints1"].cpu().numpy()
    conf  = out["confidence"].cpu().numpy()

    # Keep high-confidence matches
    mask = conf > 0.5
    return kpts0[mask], kpts1[mask]
```

---

## Multi-Area Index Strategy

All areas are stored in a single unified index. The radius filter at search time scopes results to any region:

```python
# Index Paris (run once)
# GUI: Create mode, center=48.8566,2.3522, radius=5km

# Index London (run separately, merges into same index)
# GUI: Create mode, center=51.5074,-0.1278, radius=5km

# Search Paris only — London entries are excluded by radius filter
# GUI: Search mode, center=48.8566,2.3522, radius=5km
```

---

## Hardware & Device Selection

| Hardware | Feature Extractor | Notes |
|----------|------------------|-------|
| NVIDIA CUDA | ALIKED (1024 keypoints) | Fastest; recommended for large searches |
| Apple MPS (M1–M4) | DISK (768 keypoints) | Full pipeline works well |
| CPU | DISK | Works but 5–10× slower |

```python
# Device detection pattern used internally
import torch

if torch.cuda.is_available():
    device = torch.device("cuda")
elif torch.backends.mps.is_available():
    device = torch.device("mps")
else:
    device = torch.device("cpu")
```

---

## Confidence Score Interpretation

| Score | Meaning |
|-------|---------|
| > 80  | High confidence — strong geometric match, good cluster density |
| 50–80 | Moderate — likely correct, verify visually |
| 20–50 | Low — possible match, manual review recommended |
| < 20  | Very low — insufficient evidence, try Ultra Mode |

Confidence is computed from:
1. **Inlier ratio**: `best_inliers / second_best_inliers` (uniqueness)
2. **Cluster density**: how many of the top-50 candidates fall within 50m of the best match
3. **Inlier count**: absolute number of RANSAC-verified correspondences

---

## Troubleshooting

### GUI is blank on macOS

```bash
brew install python-tk@3.11   # or python-tk@3.12 for Python 3.12
```

### LightGlue import error

```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory

Reduce keypoints in the extractor:
```python
extractor = ALIKED(max_num_keypoints=512).eval().to(device)  # default 1024
```

### Indexing stalls or fails partway through

Indexing is resumable. Re-run the same Create command — it reads existing `cosplace_parts/*.npz` files and skips already-processed grid points.

### Search returns wrong location

1. Ensure the search radius covers the actual location
2. Enable **Ultra Mode** for difficult images
3. Increase candidate count by expanding the search radius
4. Verify the area is indexed — check `cosplace_parts/` for .npz files covering the target region

### MPS errors on Apple Silicon

```bash
# Ensure PyTorch MPS build is installed
pip install torch torchvision --index-url https://download.pytorch.org/whl/nightly/cpu
```

### Gemini AI Coarse mode not working

```bash
export GEMINI_API_KEY="your_key_here"
# Verify it's set:
python -c "import os; print(os.environ.get('GEMINI_API_KEY', 'NOT SET'))"
```

---

## Tips for Best Accuracy

- **Index density matters**: grid resolution 300 is the recommended default; don't increase it as it may cause API issues
- **Image quality**: clear, daylight, non-obstructed street views perform best; use Ultra Mode for degraded images
- **Search radius**: start tight (1–2km) if you have a rough idea of location; a smaller candidate pool = faster verification
- **Flip invariance**: Netryx automatically tests the horizontally-flipped query descriptor — useful for images taken from the opposite side of the road
- **Multiple panoramas nearby**: the spatial consensus stage handles cases where the correct panorama is adjacent to the top CosPlace hit
```
