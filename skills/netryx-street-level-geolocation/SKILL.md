```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - identify location from street view photo
  - netryx geolocation
  - build a street view index
  - visual place recognition pipeline
  - osint geolocation from photo
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that finds exact GPS coordinates from any street-level photograph. It crawls Street View panoramas, indexes them as 512-dim CosPlace fingerprints, then uses LightGlue feature matching + RANSAC to verify candidates — achieving sub-50m accuracy with no internet presence required for the query image.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                        # optional: Ultra Mode (LoFTR)
```

**macOS blank GUI fix:**
```bash
brew install python-tk@3.11   # match your Python version
```

**Optional — Gemini AI coarse location (not required for most use):**
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

GPU backend auto-selected: **CUDA** (NVIDIA) → **MPS** (Apple Silicon) → **CPU**.

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (build an index) and **Search** (geolocate a photo).

---

## Core Workflow

### Step 1 — Build an Index

Index a geographic area before searching. The indexer crawls Street View panoramas in a grid and stores CosPlace descriptors.

```
GUI → Create mode
  Center lat/lon: 48.8566, 2.3522   (Paris)
  Radius: 1000 (meters)
  Grid resolution: 300 (default, don't change)
→ Click "Create Index"
```

**Indexing time estimates:**

| Radius  | ~Panoramas | Time (M2 Max) | Index size |
|---------|-----------|---------------|------------|
| 0.5 km  | ~500      | 30 min        | ~60 MB     |
| 1 km    | ~2,000    | 1–2 hrs       | ~250 MB    |
| 5 km    | ~30,000   | 8–12 hrs      | ~3 GB      |
| 10 km   | ~100,000  | 24–48 hrs     | ~7 GB      |

Indexing is **resumable** — if interrupted, restart and it picks up from the last saved chunk in `cosplace_parts/`.

The auto-build step compiles chunks into the searchable index:
```
cosplace_parts/*.npz  →  index/cosplace_descriptors.npy
                      →  index/metadata.npz
```

### Step 2 — Search

```
GUI → Search mode
  Upload street photo
  Method: Manual (enter approx lat/lon + radius)
       or AI Coarse (Gemini guesses region from visual clues)
→ "Run Search" → "Start Full Search"
```

Results appear on an embedded map with a confidence score.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (matrix)
    └── metadata.npz               # lat, lon, heading, panoid per descriptor
```

---

## Pipeline Deep-Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py exposes these helpers
from cosplace_utils import load_cosplace_model, extract_descriptor

model = load_cosplace_model()  # loads pretrained CosPlace weights

# Extract 512-dim fingerprint from a PIL image
descriptor = extract_descriptor(model, pil_image)           # shape: (512,)
flipped_desc = extract_descriptor(model, pil_image.transpose(Image.FLIP_LEFT_RIGHT))

# Index search — cosine similarity, radius-filtered (haversine)
# Returns top 500–1000 candidate panoramas
```

Internally, the search is a single matrix multiply:

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz")

scores = descriptors @ query_descriptor                    # cosine similarity
top_k = np.argsort(scores)[::-1][:1000]
```

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

For each candidate:
1. Download 8 Street View tiles → stitch panorama
2. Crop at indexed heading, 3 FOVs: **70°, 90°, 110°**
3. Extract keypoints with **ALIKED** (CUDA) or **DISK** (MPS/CPU)
4. Match with **LightGlue**
5. Filter with **RANSAC** → count inliers
6. Candidate with most inliers wins

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

# Feature extractor (auto-selected by device)
if device == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked").eval().to(device)  # or "disk"

# Extract features
feats0 = extractor.extract(query_tensor)
feats1 = extractor.extract(candidate_tensor)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
matches01 = rbd(matches01)  # remove batch dimension

matched_kps0 = feats0["keypoints"][matches01["matches"][..., 0]]
matched_kps1 = feats1["keypoints"][matches01["matches"][..., 1]]

# RANSAC verification
_, inlier_mask = cv2.findFundamentalMat(
    matched_kps0.cpu().numpy(),
    matched_kps1.cpu().numpy(),
    cv2.FM_RANSAC,
    ransacReprojThreshold=3.0,
)
inlier_count = int(inlier_mask.sum()) if inlier_mask is not None else 0
```

### Stage 3 — Refinement

```
Top 15 candidates:
  → Heading sweep: ±45° in 15° steps × 3 FOVs
  → Spatial consensus: cluster into 50m cells, prefer clusters over lone outliers
  → Confidence score: clustering quality + uniqueness ratio (best vs. runner-up)
```

### Ultra Mode (difficult images: night, blur, low texture)

Enable the **Ultra Mode** checkbox in GUI, or understand what it adds:

1. **LoFTR** — detector-free dense matching (handles blur/low contrast):

```python
import kornia.feature as KF

matcher = KF.LoFTR(pretrained="outdoor")
# Operates on grayscale image pairs, no keypoint detection needed
input_dict = {
    "image0": query_gray,     # (1, 1, H, W)
    "image1": candidate_gray,
}
correspondences = matcher(input_dict)
mkpts0 = correspondences["keypoints0"]
mkpts1 = correspondences["keypoints1"]
```

2. **Descriptor hopping** — if best match has <50 inliers, extract CosPlace from the *matched panorama* (clean image) and re-search.

3. **Neighborhood expansion** — search all panoramas within 100m of best match.

---

## Multi-City Index Pattern

All cities share one index — coordinate + radius filters isolate results:

```python
# Index Paris
# → index/cosplace_descriptors.npy  (Paris entries added)

# Index London
# → index/cosplace_descriptors.npy  (London entries appended)

# Search Paris only: center=(48.8566, 2.3522), radius=5000m
# Search London only: center=(51.5074, -0.1278), radius=10000m
# No city tag needed — haversine filter handles it
```

---

## Using the Standalone Index Builder (Large Areas)

For areas >5km radius, use `build_index.py` directly for better performance:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5000 \
  --grid 300
```

This writes chunks to `cosplace_parts/` and compiles the final index. Run in a `tmux`/`screen` session for overnight jobs.

---

## Common Patterns

### Pattern: Programmatic descriptor extraction

```python
from PIL import Image
from cosplace_utils import load_cosplace_model, extract_descriptor

model = load_cosplace_model()
img = Image.open("street_photo.jpg").convert("RGB")
desc = extract_descriptor(model, img)
print(desc.shape)   # (512,)
```

### Pattern: Manual index search (no GUI)

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000
    φ1, φ2 = radians(lat1), radians(lat2)
    dφ = radians(lat2 - lat1)
    dλ = radians(lon2 - lon1)
    a = sin(dφ/2)**2 + cos(φ1)*cos(φ2)*sin(dλ/2)**2
    return R * 2 * asin(sqrt(a))

# Load index
descs = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta  = np.load("index/metadata.npz")
lats, lons = meta["lats"], meta["lons"]

# Load query
model = load_cosplace_model()
query_img = Image.open("query.jpg").convert("RGB")
q_desc = extract_descriptor(model, query_img)       # (512,)

# Radius filter (e.g. Paris, 2km)
center_lat, center_lon, radius_m = 48.8566, 2.3522, 2000
mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

# Cosine similarity search within radius
filtered_descs = descs[mask]
scores = filtered_descs @ q_desc
top_local = np.argsort(scores)[::-1][:500]

# Map back to global indices
global_indices = np.where(mask)[0][top_local]
candidates = [(lats[i], lons[i], scores[top_local[k]]) 
              for k, i in enumerate(global_indices)]
print("Top candidate:", candidates[0])
```

### Pattern: Check GPU device

```python
import torch

if torch.cuda.is_available():
    device = "cuda"
    extractor_type = "aliked"
elif torch.backends.mps.is_available():
    device = "mps"
    extractor_type = "disk"
else:
    device = "cpu"
    extractor_type = "disk"

print(f"Using device: {device}, extractor: {extractor_type}")
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| GUI appears blank | macOS bundled tkinter bug | `brew install python-tk@3.11` |
| `ModuleNotFoundError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| `ModuleNotFoundError: kornia` | Ultra Mode dependency missing | `pip install kornia` |
| Indexing stops mid-run | Network timeout or interrupt | Just restart — resumes from `cosplace_parts/` |
| Low inlier count (<20) everywhere | Query image too blurry/dark | Enable Ultra Mode (LoFTR) |
| Search returns wrong country | Radius too large, wrong center | Tighten radius, verify center coordinates |
| MPS out of memory | DISK using too many keypoints | Reduce `max_num_keypoints` to 512 |
| CUDA OOM during LightGlue | Too many keypoints | Reduce ALIKED `max_num_keypoints` to 512 |
| `index/` folder empty | Index not compiled yet | Run a search once; auto-build triggers, or run `build_index.py` |
| Gemini AI Coarse not working | API key missing | `export GEMINI_API_KEY="..."` |

---

## Key Configuration Defaults

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Don't change — controls panorama density |
| CosPlace descriptor dim | 512 | Fixed by model |
| Retrieval top-k | 500–1000 | Narrows to Stage 2 candidates |
| ALIKED keypoints (CUDA) | 1024 | Reduce to 512 on low VRAM |
| DISK keypoints (MPS/CPU) | 768 | Reduce to 512 on low VRAM |
| Heading refinement steps | ±45° / 15° steps | 7 headings × 3 FOVs = 21 crops per candidate |
| Spatial cluster cell size | 50m | Consensus grouping |
| Neighborhood expansion (Ultra) | 100m | Searches all panoids within radius of best match |

---

## Models Reference

| Model | Role | Backend |
|-------|------|---------|
| CosPlace (CVPR 2022) | Global 512-dim fingerprint | All devices |
| ALIKED (IEEE TIP 2023) | Local keypoints | CUDA only |
| DISK (NeurIPS 2020) | Local keypoints | MPS / CPU |
| LightGlue (ICCV 2023) | Deep feature matching | All devices |
| LoFTR (CVPR 2021) | Dense matching, Ultra Mode | via `kornia` |
```
