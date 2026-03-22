```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use Netryx to locate an image
  - index street view panoramas
  - run netryx geolocation search
  - visual place recognition pipeline
  - identify location from street photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies the exact GPS coordinates of any street-level photograph. It crawls and indexes street-view panoramas, then uses a three-stage computer vision pipeline (CosPlace global retrieval → ALIKED/DISK+LightGlue geometric verification → refinement) to match a query image against the physical world — not the internet.

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

### Optional: Gemini AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

### macOS tkinter fix (blank GUI)

```bash
brew install python-tk@3.11   # match your Python version
```

---

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4 GB | 8 GB+ |
| RAM | 8 GB | 16 GB+ |
| Storage | 10 GB | 50 GB+ |

**GPU backends:**
- NVIDIA → CUDA (uses ALIKED, 1024 keypoints)
- Apple Silicon → MPS (uses DISK, 768 keypoints)
- No GPU → CPU (works, significantly slower)

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (build an index) and **Search** (geolocate a photo).

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim global descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Workflow

### Step 1: Create an Index

Index a geographic area before searching. This crawls street-view panoramas and stores CosPlace fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon of target area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|------------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

Indexing is **incremental** — safe to interrupt and resume.

Multiple cities can coexist in the same index. The radius filter at search time isolates the correct region automatically.

### Step 2: Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini infers region from visual cues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on a map

---

## The Three-Stage Pipeline

### Stage 1 — Global Retrieval (CosPlace)

```
Query image → 512-dim CosPlace descriptor
             + flipped image descriptor
             → cosine similarity against full index
             → haversine radius filter
             → top 500–1000 candidates
```

Runs in under 1 second (single matrix multiply).

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```
For each candidate:
  Download panorama (8 tiles, stitched)
  → rectilinear crop at indexed heading
  → 3 FOV crops (70°, 90°, 110°)
  → ALIKED (CUDA) or DISK (MPS/CPU) keypoints
  → LightGlue matching vs query keypoints
  → RANSAC inlier filtering
  → ranked by verified inlier count
```

Processes 300–500 candidates in 2–5 minutes depending on hardware.

### Stage 3 — Refinement

- **Heading refinement**: ±45° at 15° steps, 3 FOVs, top 15 candidates
- **Spatial consensus**: 50 m grid clustering; clusters beat single high-inlier outliers
- **Confidence scoring**: clustering quality + uniqueness ratio vs. runner-up

### Ultra Mode (difficult images)

Enable the **Ultra Mode** checkbox for night shots, blurry images, or low-texture scenes.

Adds:
- **LoFTR**: detector-free dense matching (handles blur/low contrast)
- **Descriptor hopping**: re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: searches all panoramas within 100 m of the best match

Requires `kornia`: `pip install kornia`

---

## Code Examples

### Extract a CosPlace descriptor programmatically

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = get_cosplace_model(device)

img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)
# descriptor.shape → (512,)
print("Descriptor shape:", descriptor.shape)
```

### Search the index against a query descriptor

```python
import numpy as np

# Load the prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]    # (N,)
lons = meta["lons"]    # (N,)
headings = meta["headings"]
panoids = meta["panoids"]

# Cosine similarity search
query = descriptor / np.linalg.norm(descriptor)
index_norm = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = index_norm @ query   # (N,)

# Radius filter (haversine) — restrict to 5 km around Paris
from math import radians, sin, cos, sqrt, atan2

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1))*cos(radians(lat2))*sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522
radius_m = 5000

mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

filtered_scores = np.where(mask, scores, -1)
top_indices = np.argsort(filtered_scores)[::-1][:500]

print("Top match:", lats[top_indices[0]], lons[top_indices[0]])
print("Score:", filtered_scores[top_indices[0]])
```

### Build index from the command line (large datasets)

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 2.0 \
  --resolution 300
```

### Use flipped descriptor for both orientations

```python
import torchvision.transforms.functional as TF

img_pil = Image.open("query.jpg").convert("RGB")
img_flipped = TF.hflip(img_pil)

desc_orig = get_descriptor(model, img_pil, device)
desc_flip = get_descriptor(model, img_flipped, device)

# Average or max-pool for a combined signal
desc_combined = (desc_orig + desc_flip) / 2
desc_combined /= np.linalg.norm(desc_combined)
```

### Run LightGlue matching between two images

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

img0 = load_image("query.jpg").to(device)
img1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(img0)
feats1 = extractor.extract(img1)

matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

matched_kps0 = feats0["keypoints"][matches01["matches"][..., 0]]
matched_kps1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(matched_kps0)}")
```

### RANSAC geometric verification on matches

```python
import cv2
import numpy as np

# matched_kps0, matched_kps1 are (N, 2) numpy arrays
pts0 = matched_kps0.cpu().numpy()
pts1 = matched_kps1.cpu().numpy()

if len(pts0) >= 4:
    _, inlier_mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, 5.0)
    inlier_count = int(inlier_mask.sum()) if inlier_mask is not None else 0
    print(f"RANSAC inliers: {inlier_count}")
else:
    inlier_count = 0
```

---

## Configuration Reference

| Parameter | Where | Default | Notes |
|-----------|-------|---------|-------|
| Grid resolution | GUI / CLI | 300 | Higher = denser panorama coverage |
| Search radius | GUI / CLI | 1.0 km | Haversine filter at search time |
| Top candidates | Code | 500–1000 | CosPlace retrieval pool size |
| Heading refinement range | Code | ±45° @ 15° steps | Applied to top 15 candidates |
| Spatial cluster cell | Code | 50 m | Consensus grid size |
| FOV crops | Code | 70°, 90°, 110° | Three crops per heading |
| Ultra neighborhood | Code | 100 m | Expansion radius in Ultra Mode |

---

## Common Patterns

### Geolocating OSINT/conflict images
1. Index the suspected region (city or district level, 2–5 km radius)
2. Upload the image in Search mode with Manual coordinates + radius
3. If confidence is low, enable Ultra Mode
4. Cross-reference the returned coordinates with satellite imagery

### Multi-city index
Index each city separately (they share the same `index/` files). At search time, specify the correct center+radius — only matching panoramas are returned.

```
# Index Paris
python build_index.py --lat 48.8566 --lon 2.3522 --radius 5.0

# Index London (appends to same index)
python build_index.py --lat 51.5074 --lon -0.1278 --radius 5.0

# Search only in Paris
# GUI: center=48.8566,2.3522 radius=6km
```

### Handling degraded images (night/blur)
- Always try standard pipeline first (faster)
- Enable **Ultra Mode** for night photography, motion blur, heavy compression
- Ensure `kornia` is installed: `pip install kornia`

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| GUI appears blank | macOS bundled tkinter bug | `brew install python-tk@3.11` |
| `ImportError: lightglue` | LightGlue not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| `ImportError: kornia` | kornia not installed | `pip install kornia` (Ultra Mode only) |
| CUDA out of memory | Not enough VRAM | Reduce `max_num_keypoints` in ALIKED init; use DISK instead |
| Low confidence score (<0.3) | Query area not indexed, or degraded image | Re-index with denser grid; enable Ultra Mode |
| Index search returns 0 results | Radius too small or wrong center coords | Increase radius; verify lat/lon are correct |
| Indexing stalled | Network timeout | Safe to restart — resumes from last checkpoint |
| Wrong city matched | Index too broad + wrong search center | Narrow radius; set center near the expected city |
| MPS device errors | Metal not supported for all ops | Set `PYTORCH_ENABLE_MPS_FALLBACK=1` |

```bash
# MPS fallback for unsupported ops
export PYTORCH_ENABLE_MPS_FALLBACK=1
python test_super.py
```

---

## Key Dependencies

```
torch / torchvision       # Deep learning backend
lightglue                 # Feature matching (install from GitHub)
kornia                    # LoFTR dense matching (Ultra Mode)
numpy / scipy             # Array ops, clustering
Pillow                    # Image loading
opencv-python             # RANSAC, image stitching
requests                  # Street View tile downloads
tkinter                   # GUI (stdlib, may need brew upgrade on macOS)
```

---

## How the Index Files Work

```
cosplace_parts/*.npz          # Incremental chunks written during indexing
       ↓  (auto-build step)
index/cosplace_descriptors.npy    # Merged (N × 512) float32 matrix
index/metadata.npz                # lats, lons, headings, panoids arrays
```

All arrays are aligned by row index `i`:
- `descriptors[i]` ↔ `lats[i], lons[i]` ↔ `headings[i]` ↔ `panoids[i]`
```
