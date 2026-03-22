```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision pipelines.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - identify location from photo
  - netryx geolocation
  - build street view index
  - visual place recognition locally
  - osint geolocation from image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls street-view panoramas, indexes them as 512-dim CosPlace fingerprints, then matches query photos using local feature extraction (ALIKED/DISK) and deep feature matching (LightGlue) — no internet image database needed.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (install from GitHub, not PyPI)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

**GPU backend auto-selection:**
- NVIDIA → CUDA (ALIKED, 1024 keypoints)
- Apple Silicon → MPS (DISK, 768 keypoints)
- Fallback → CPU (significantly slower)

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loader & descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Pipeline

### Stage 1 — Global Retrieval (CosPlace)

Extracts a 512-dim visual fingerprint from the query image and cosine-similarity searches the index. Also searches a horizontally flipped version (handles reversed perspectives). Returns top 500–1000 candidates filtered by haversine radius.

**Runs in under 1 second** — single matrix multiplication regardless of index size.

### Stage 2 — Local Geometric Verification (ALIKED/DISK + LightGlue)

For each candidate:
1. Downloads Google Street View panorama (8 tiles, stitched)
2. Crops rectilinear view at indexed heading
3. Generates multi-FOV crops (70°, 90°, 110°) to handle zoom mismatches
4. ALIKED/DISK extracts keypoints + descriptors
5. LightGlue deep-matches query vs candidate keypoints
6. RANSAC filters geometrically consistent matches (inliers)

**Processes 300–500 candidates in 2–5 minutes** on modern hardware.

### Stage 3 — Refinement

- **Heading refinement**: Tests ±45° at 15° steps across 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; clusters beat single outliers
- **Confidence scoring**: Evaluates clustering density + uniqueness ratio (best vs runner-up)

### Ultra Mode

Enable for night shots, blurry images, low-texture scenes:
- **LoFTR**: Detector-free dense matching (handles blur/low contrast)
- **Descriptor hopping**: Re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Workflow

### Step 1: Create an Index

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of target area
3. Set radius (start 0.5–1km for testing, 5–10km production)
4. Set grid resolution (default 300 — do not change)
5. Click **Create Index**

Index builds incrementally — safe to interrupt and resume.

**Time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

### Step 2: Search

1. Select **Search** mode
2. Upload street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues (signs, architecture, vegetation) — no prior knowledge needed
4. Click **Run Search** → **Start Full Search**
5. Watch real-time scanning visualization
6. Result: GPS coordinates + confidence score on map

---

## Multi-Area Index Strategy

The index is unified — all areas share one index file. Radius filtering isolates searches.

```
# Index Paris
Center: 48.8566, 2.3522 | Radius: 5km

# Index London  
Center: 51.5074, -0.1278 | Radius: 5km

# Search only Paris (London data ignored automatically)
Search center: 48.8566, 2.3522 | Radius: 5km
```

No city selection needed. Coordinates + radius handle everything.

---

## Using cosplace_utils.py Directly

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model()
model.eval()

# Extract descriptor from a query image
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img)  # returns torch.Tensor shape (512,)

print(f"Descriptor shape: {descriptor.shape}")
print(f"Descriptor norm: {torch.norm(descriptor):.4f}")
```

---

## Using build_index.py for Large Datasets

For areas >5km radius, use the standalone high-performance index builder:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5000 \        # meters
  --resolution 300 \     # grid density (default, don't change)
  --output ./index
```

This writes to `cosplace_parts/` incrementally, then auto-compiles to `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Programmatic Search (Advanced)

The main logic lives in `test_super.py`. For scripted/headless use, extract the search function:

```python
import numpy as np
import torch
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

# Load pre-built index
descriptors = np.load("index/cosplace_descriptors.npy")       # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]        # (N,)
lons = meta["lons"]        # (N,)
headings = meta["headings"] # (N,)
panoids = meta["panoids"]   # (N,)

# Load model
model = load_cosplace_model()
model.eval()

# Query
img = Image.open("query.jpg").convert("RGB")
q_desc = extract_descriptor(model, img).numpy()          # (512,)
q_desc_flip = extract_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT)).numpy()

# Cosine similarity search
desc_norm = descriptors / (np.linalg.norm(descriptors, axis=1, keepdims=True) + 1e-8)
q_norm = q_desc / (np.linalg.norm(q_desc) + 1e-8)
q_flip_norm = q_desc_flip / (np.linalg.norm(q_desc_flip) + 1e-8)

sims = np.maximum(desc_norm @ q_norm, desc_norm @ q_flip_norm)

# Radius filter (haversine) — e.g., 2km around Paris center
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * asin(sqrt(a))

center_lat, center_lon, radius_m = 48.8566, 2.3522, 2000
mask = np.array([haversine(center_lat, center_lon, la, lo) <= radius_m
                 for la, lo in zip(lats, lons)])

sims_filtered = np.where(mask, sims, -1)
top_k = np.argsort(sims_filtered)[::-1][:500]

print("Top 5 candidates:")
for i in top_k[:5]:
    print(f"  panoid={panoids[i]}  lat={lats[i]:.6f}  lon={lons[i]:.6f}  "
          f"heading={headings[i]}  sim={sims[i]:.4f}")
```

---

## Common Patterns

### Pattern: Testing index coverage

```python
import numpy as np

meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]

print(f"Total indexed panoramas: {len(lats)}")
print(f"Lat range: {lats.min():.4f} — {lats.max():.4f}")
print(f"Lon range: {lons.min():.4f} — {lons.max():.4f}")
```

### Pattern: Checking GPU backend

```python
import torch

if torch.cuda.is_available():
    print(f"CUDA: {torch.cuda.get_device_name(0)}")
    print("Feature extractor: ALIKED (1024 keypoints)")
elif torch.backends.mps.is_available():
    print("MPS: Apple Silicon")
    print("Feature extractor: DISK (768 keypoints)")
else:
    print("CPU only — expect slow performance")
```

### Pattern: Verify LightGlue installation

```python
try:
    from lightglue import LightGlue, SuperPoint, DISK, ALIKED
    from lightglue.utils import load_image, rbd
    print("LightGlue: OK")
except ImportError:
    print("LightGlue missing — run: pip install git+https://github.com/cvg/LightGlue.git")
```

### Pattern: Verify Ultra Mode (kornia/LoFTR)

```python
try:
    import kornia
    from kornia.feature import LoFTR
    print(f"kornia {kornia.__version__}: LoFTR available")
except ImportError:
    print("Ultra Mode unavailable — run: pip install kornia")
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11   # match your Python version
```

### `ImportError: No module named 'lightglue'`

LightGlue must be installed from GitHub, not PyPI:
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory

Reduce candidates in Stage 2 by tightening radius, or use a smaller index area. On 4GB VRAM, keep candidate count ≤300.

### Index build interrupted / partial index

Safe to rerun — indexing is incremental. The builder checks existing `cosplace_parts/*.npz` files and skips already-processed grid points.

### Very low inlier counts (<20) on all candidates

- Enable **Ultra Mode** (LoFTR + descriptor hopping)
- Verify the query image is street-level (not aerial/indoor)
- Ensure the indexed area actually covers the query location — expand radius
- Check image quality: extreme blur, night shots, or heavy occlusion reduce matching

### AI Coarse mode fails / returns no region

```bash
echo $GEMINI_API_KEY   # verify env var is set
```
Fall back to **Manual** mode — provide approximate coordinates from any map service.

### Slow indexing on CPU

Expected. Use GPU hardware. On M1 Mac with MPS, 1km radius (~2000 panoramas) takes ~1–2 hours. CPU-only systems should index small areas only (<0.5km).

### `metadata.npz` not found during search

The index hasn't been compiled yet. Run **Create Index** first, or manually trigger:
```bash
python build_index.py --compile-only --output ./index
```
(Check `build_index.py` for exact flag names — recompile from existing `cosplace_parts/`.)

---

## Search Method Decision Guide

| Scenario | Recommended Method |
|----------|-------------------|
| Know approximate city/neighborhood | Manual — enter center coords + tight radius |
| Unknown location entirely | AI Coarse (Gemini) → refine with Manual |
| Night/blurry/low-texture image | Manual + Ultra Mode enabled |
| OSINT / conflict zone monitoring | Manual with broad radius (10–20km) + Ultra Mode |
| Rapid testing of index quality | Manual with 0.5km radius around known location |

---

## Key Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (CVPR 2022) | Global 512-dim place fingerprint | All |
| ALIKED (IEEE TIP 2023) | Local keypoint extraction | CUDA only |
| DISK (NeurIPS 2020) | Local keypoint extraction | MPS / CPU |
| LightGlue (ICCV 2023) | Deep feature matching | All |
| LoFTR (CVPR 2021) | Dense matching, Ultra Mode | All (slow on CPU) |
```
