```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from any street photo using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find coordinates from street image
  - street level geolocation
  - reverse geolocate image
  - netryx geolocation
  - index street view area
  - identify location from photo
  - osint image geolocation
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls street-view panoramas, builds a searchable visual index using CosPlace embeddings, then matches query photos via ALIKED/DISK keypoint extraction + LightGlue deep feature matching — all running on your local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # Required
pip install kornia                                        # Optional: Ultra Mode (LoFTR)
```

### Requirements
- Python 3.9+ (3.10+ recommended)
- GPU: NVIDIA (CUDA, 4GB+ VRAM) or Apple Silicon (MPS) or CPU (slow)
- RAM: 8GB min, 16GB+ recommended
- Storage: 10GB min; scales with indexed area (1km radius ≈ 250MB)

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

### 1. Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (start 0.5–1km for testing; 5–10km for production)
4. Set grid resolution (default: 300 — don't change)
5. Click **Create Index**

Indexing is incremental — interruptions resume automatically.

**Time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hrs       | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hrs      | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hrs     | ~7 GB      |

### 2. Search (geolocate a photo)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: enter center coords + radius (fastest, use when you know the region)
   - **AI Coarse**: Gemini analyzes visual clues to guess region (no prior knowledge needed)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py           # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py       # CosPlace model loading + descriptor extraction
├── build_index.py          # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/         # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace 512-dim descriptor + flipped descriptor
    ▼
Index Search (cosine similarity, haversine radius filter)
    → Top 500–1000 candidates
    │
    ▼
Download panoramas → Rectilinear crops at 3 FOVs (70°, 90°, 110°)
    → ALIKED (CUDA) / DISK (MPS/CPU) keypoint extraction
    → LightGlue deep feature matching
    → RANSAC geometric verification (inlier count)
    │
    ▼
Heading Refinement: ±45° at 15° steps, 3 FOVs, top 15 candidates
    → Spatial consensus clustering (50m cells)
    → Confidence scoring (clustering + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable **Ultra Mode** checkbox in GUI for difficult images (night, blur, low texture). Adds:

- **LoFTR**: detector-free dense matching (handles blur/low-contrast)
- **Descriptor hopping**: re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: searches all panoramas within 100m of best match

Requires `kornia`: `pip install kornia`

---

## Key Models

| Model | Role | Device |
|-------|------|--------|
| CosPlace | Global visual fingerprint (512-dim) | Any |
| ALIKED | Local keypoints + descriptors | CUDA |
| DISK | Local keypoints + descriptors | MPS / CPU |
| LightGlue | Deep feature matching | Any |
| LoFTR | Dense detector-free matching (Ultra) | Any |

---

## Working with the Index Programmatically

```python
import numpy as np
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512) float32
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,) float64
lons = meta["lons"]       # (N,) float64
headings = meta["headings"]  # (N,) float
panoids = meta["panoids"]    # (N,) str

# Load CosPlace model
device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"
model = get_cosplace_model(device=device)

# Extract descriptor from a query image
img = Image.open("query.jpg").convert("RGB")
query_desc = get_descriptor(model, img, device=device)  # (512,) numpy array

# Cosine similarity search
from numpy.linalg import norm
sims = (descriptors @ query_desc) / (norm(descriptors, axis=1) * norm(query_desc) + 1e-8)
top_idx = np.argsort(sims)[::-1][:500]

# Haversine radius filter (e.g., 5km around Paris)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 5.0

def haversine(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

filtered = [i for i in top_idx if haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_km]
print(f"Top candidate: lat={lats[filtered[0]]:.6f}, lon={lons[filtered[0]]:.6f}")
```

---

## Building the Index Standalone (Large Areas)

For large-scale indexing without the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300
```

The index auto-rebuilds from `cosplace_parts/*.npz` chunks into `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Multi-City Index Pattern

Netryx uses a single unified index — add multiple cities and search by coordinates + radius:

```python
# All cities indexed into the same cosplace_parts/ folder
# Search Paris only:
search(center_lat=48.8566, center_lon=2.3522, radius_km=5)

# Search London only:
search(center_lat=51.5074, center_lon=-0.1278, radius_km=10)

# No city selection — radius filter handles isolation automatically
```

---

## Troubleshooting

### GUI appears blank (macOS)
```bash
brew install python-tk@3.11   # match your Python version
```

### CUDA out of memory
- Reduce candidates: lower radius or use a smaller index area
- Use DISK instead of ALIKED (lower keypoint count): handled automatically on MPS/CPU
- Disable Ultra Mode if enabled

### LightGlue import error
```bash
pip install git+https://github.com/cvg/LightGlue.git
# NOT from PyPI — must install from GitHub
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
```

### Indexing stalls / API rate limits
- Indexing is incremental; just re-run — it resumes from last saved chunk
- Check `cosplace_parts/` for partial `.npz` files to verify progress

### Low confidence / wrong result
- Increase search radius to cover more candidates
- Enable **Ultra Mode** for degraded images (blur, night, low texture)
- Verify the area is indexed: check `index/metadata.npz` lat/lon bounds
- Try **AI Coarse** mode if unsure of the region — Gemini estimates from visual cues

### Slow performance (CPU-only)
- Standard pipeline: 2–5 min on GPU → 20–60 min on CPU for 500 candidates
- Reduce candidates by tightening radius
- Avoid Ultra Mode on CPU

---

## Configuration Reference

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Panorama density during indexing — don't change |
| Top candidates (retrieval) | 500–1000 | More = better recall, slower matching |
| Heading refinement steps | ±45° at 15° | Applied to top 15 candidates |
| Spatial consensus cell | 50m | Clustering granularity for false-positive rejection |
| Ultra neighborhood | 100m | Expansion radius for descriptor hopping |
| ALIKED keypoints (CUDA) | 1024 | |
| DISK keypoints (MPS/CPU) | 768 | |
| FOVs tested per candidate | 3 | 70°, 90°, 110° |

---

## Environment Variables

```bash
export GEMINI_API_KEY="..."   # Required only for AI Coarse search mode
```

No other secrets required — Street View panorama fetching is handled internally.
```
