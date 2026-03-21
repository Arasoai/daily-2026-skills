```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use netryx to locate an image
  - index street view panoramas
  - identify location from street photo
  - osint geolocation from photo
  - run netryx geolocation search
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies the precise GPS coordinates of any street-level photograph. It crawls and indexes street-view panoramas, then matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). Sub-50m accuracy. No internet presence of the target image required. Runs entirely on local hardware.

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

# 2. Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate          # Linux/macOS
# venv\Scripts\activate           # Windows

# 3. Install core dependencies
pip install -r requirements.txt

# 4. Install LightGlue (required — not on PyPI)
pip install git+https://github.com/cvg/LightGlue.git

# 5. Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### macOS tkinter fix (blank GUI)
```bash
brew install python-tk@3.11   # replace 3.11 with your Python version
```

### Optional: Gemini API key (AI Coarse location mode)
```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

GPU backends automatically selected:
- **NVIDIA** → CUDA → ALIKED (1024 keypoints)
- **Apple Silicon** → MPS → DISK (768 keypoints)
- **CPU** → DISK (slowest)

---

## Launch the GUI

```bash
python test_super.py
```

All functionality (indexing, searching, visualization) is in this single entry point.

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The indexer crawls street-view panoramas, extracts CosPlace 512-dim fingerprints, and saves them to disk.

**In the GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hr        | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hr       | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hr      | ~7 GB      |

Indexing is **resumable** — safe to interrupt and restart.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center lat/lon + radius
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### Ultra Mode

Enable the **Ultra Mode** checkbox for:
- Blurry or low-resolution images
- Night photos
- Low-texture environments

Ultra Mode adds:
- **LoFTR** detector-free dense matching
- **Descriptor hopping** (re-searches from matched panorama descriptor)
- **Neighborhood expansion** (searches all panoramas within 100m of best match)

Significantly slower, but catches matches the standard pipeline misses.

---

## Project Structure

```
netryx/
├── test_super.py           # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py       # CosPlace model loading & descriptor extraction
├── build_index.py          # Standalone index builder (large datasets)
├── requirements.txt
├── cosplace_parts/         # Raw embedding chunks (.npz), written during indexing
└── index/
    ├── cosplace_descriptors.npy   # Compiled 512-dim descriptor matrix
    └── metadata.npz               # Lat/lon, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

```
Query image
  → CosPlace 512-dim descriptor
  → Also extract descriptor from horizontally flipped image
  → Cosine similarity against entire index (single matrix multiply, <1 sec)
  → Haversine radius filter
  → Top 500–1000 candidates
```

### Stage 2 — Local Geometric Verification

```
For each candidate panorama:
  → Download from Street View (8 tiles, stitched)
  → Crop at indexed heading angle
  → Generate multi-FOV crops: 70°, 90°, 110°
  → ALIKED (CUDA) or DISK (MPS/CPU): extract keypoints + descriptors
  → LightGlue: deep feature matching vs query keypoints
  → RANSAC: keep geometrically consistent inliers
  → Rank by inlier count
```

Processes 300–500 candidates in **2–5 minutes** depending on hardware.

### Stage 3 — Refinement

```
Top 15 candidates:
  → Heading refinement: test ±45° at 15° steps × 3 FOVs
  → Spatial consensus: cluster into 50m cells (prefer clusters over outliers)
  → Confidence scoring: clustering density + uniqueness ratio vs runner-up
```

---

## Index Architecture

The index is **unified and coordinate-filtered** — index multiple cities into one index, search by specifying center + radius.

```python
# Conceptual index lookup (internal to test_super.py)
import numpy as np

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]      # shape: (N,)
lons = meta["lons"]
headings = meta["headings"]
panoids = meta["panoids"]

# Cosine similarity search
query_desc = extract_cosplace_descriptor(query_image)      # (512,)
similarities = descriptors @ query_desc                    # (N,) dot products

# Radius filter (haversine)
from math import radians, sin, cos, sqrt, atan2

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000  # meters
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522  # Paris
radius_m = 1000

mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

filtered_sims = similarities.copy()
filtered_sims[~mask] = -1

top_k = np.argsort(filtered_sims)[-500:][::-1]
```

---

## CosPlace Descriptor Extraction

```python
# cosplace_utils.py usage pattern
from cosplace_utils import load_cosplace_model, get_descriptor
import torch
from PIL import Image

device = (
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)

model = load_cosplace_model(device=device)

# Extract descriptor from a query image
img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device=device)  # numpy array, shape (512,)

# Also extract flipped variant (catches reversed perspectives)
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = get_descriptor(model, img_flipped, device=device)

# Use both for retrieval
combined_score = 0.6 * sim_normal + 0.4 * sim_flipped
```

---

## Using the Standalone Index Builder

For large areas (5km+), use `build_index.py` instead of the GUI to avoid memory issues and get better throughput:

```bash
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5000 \
    --resolution 300 \
    --output cosplace_parts/
```

After building parts, the GUI auto-compiles them into `index/` on first search run, or trigger manually by opening the app and selecting **Search** mode.

---

## Common Patterns

### Pattern 1: Index a city, search a specific neighborhood

```
Index:  center=(48.8566, 2.3522), radius=10km  → indexes all of central Paris
Search: center=(48.8738, 2.2950), radius=1km   → searches only 17th arrondissement
```

No re-indexing needed. Radius filter handles it at search time.

### Pattern 2: Multi-city index

```
Index run 1: center=(48.8566, 2.3522), radius=5km   # Paris
Index run 2: center=(51.5074, -0.1278), radius=5km  # London
Index run 3: center=(40.7128, -74.0060), radius=5km # New York

All go into the same cosplace_parts/ and index/
Search with the right center+radius to target the right city
```

### Pattern 3: Difficult image → Ultra Mode

For blurry, night, or low-texture images:
1. Enable **Ultra Mode** in the GUI checkbox
2. Expect 3–10x longer runtime
3. LoFTR requires `pip install kornia`

### Pattern 4: Interrupted indexing

Indexing saves incrementally to `cosplace_parts/*.npz`. To resume:
- Simply re-run with the same parameters
- Already-indexed panoramas are skipped

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your Python version
# Then relaunch:
python test_super.py
```

### LightGlue import error
```bash
# LightGlue is NOT on PyPI — must install from GitHub
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not working
```bash
pip install kornia
# Verify:
python -c "import kornia; print(kornia.__version__)"
```

### CUDA out of memory
- Reduce candidate count in GUI settings
- Use DISK instead of ALIKED (lower keypoint count: 768 vs 1024)
- Ensure no other GPU processes are running

### MPS (Apple Silicon) errors
```bash
# Ensure PyTorch with MPS support
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
# MPS is bundled in standard macOS PyTorch builds (1.12+)
python -c "import torch; print(torch.backends.mps.is_available())"
```

### Slow indexing
- Use `build_index.py` instead of the GUI for large areas
- Ensure GPU is active: check that CUDA/MPS is detected at startup
- Start with small radius (0.5km) to validate setup before large runs

### Poor match accuracy
- Verify the indexed area covers the query image location
- Try Ultra Mode for degraded images
- Increase candidate count (top-K) if available in settings
- Ensure query image is genuine street-level (not aerial, not indoor)

### Gemini AI Coarse mode fails
```bash
export GEMINI_API_KEY="your_key_here"
# Verify:
python -c "import os; print(os.environ.get('GEMINI_API_KEY', 'NOT SET'))"
```
Note: AI Coarse mode is optional. Manual coordinate entry (if you know the rough region) works better and is the recommended approach.

---

## Model Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace | Global 512-dim visual fingerprint | All |
| ALIKED | Local keypoint extraction | CUDA only |
| DISK | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Detector-free dense matching (Ultra) | All (slow on CPU) |

---

## Key Files for Programmatic Use

| File | Purpose |
|------|---------|
| `test_super.py` | All-in-one: GUI, indexer, search pipeline — start here |
| `cosplace_utils.py` | Import for descriptor extraction in custom scripts |
| `build_index.py` | CLI index builder for large-scale indexing |
| `index/cosplace_descriptors.npy` | The compiled descriptor matrix |
| `index/metadata.npz` | Coordinates + panorama metadata |
| `cosplace_parts/*.npz` | Raw per-batch embedding chunks |
```
