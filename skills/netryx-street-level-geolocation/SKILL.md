```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - identify location from street view photo
  - netryx geolocation
  - build a street view index
  - match photo to map location
  - osint geolocation from image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable vector index of CosPlace descriptors, then uses LightGlue feature matching + RANSAC verification to pinpoint locations with sub-50m accuracy — no internet image presence required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                       # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

### Platform requirements

| Platform | GPU support | Notes |
|----------|------------|-------|
| macOS M1–M4 | MPS | Uses DISK extractor (768 kp) |
| Linux + NVIDIA | CUDA | Uses ALIKED extractor (1024 kp), fastest |
| CPU-only | None | Works, significantly slower |
| Min VRAM | 4 GB | 8 GB+ recommended |

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Project Structure

```
netryx/
├── test_super.py           # Main app: GUI + indexing + search
├── cosplace_utils.py       # CosPlace model loading + descriptor extraction
├── build_index.py          # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/         # Raw .npz embedding chunks (created at index time)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Pipeline

### Three-stage architecture

```
Query Image
    │
    ├── [Stage 1] CosPlace → 512-dim descriptor + flipped descriptor
    │             cosine similarity search against index (radius-filtered)
    │             → top 500–1000 candidates  (<1 second)
    │
    ├── [Stage 2] Download panoramas → rectilinear crops at 3 FOVs (70°/90°/110°)
    │             ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    │             LightGlue deep feature matching
    │             RANSAC geometric verification
    │             → best match by inlier count  (2–5 minutes)
    │
    └── [Stage 3] Heading refinement ±45° at 15° steps × 3 FOVs (top 15 candidates)
                  Spatial consensus clustering (50m cells)
                  Confidence scoring (uniqueness ratio)
                  → GPS coordinates + confidence score
```

---

## Step 1: Build an Index

In the GUI select **Create** mode, or understand the flow programmatically:

```python
# Conceptual equivalent of what test_super.py does during indexing
# The actual call happens through the GUI; this shows the data flow.

import numpy as np
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

model = load_cosplace_model()           # downloads weights on first run

# For each crawled panorama crop:
img = Image.open("panorama_crop.jpg")
descriptor = extract_descriptor(model, img)   # returns (512,) float32 numpy array

# Saved incrementally to cosplace_parts/chunk_XXXX.npz
np.savez("cosplace_parts/chunk_0001.npz",
         descriptors=descriptor[np.newaxis, :],   # shape (N, 512)
         lats=np.array([48.8566]),
         lons=np.array([2.3522]),
         headings=np.array([90.0]),
         panoids=np.array(["abc123xyz"]))
```

### Index build parameters (GUI fields)

| Field | Default | Notes |
|-------|---------|-------|
| Center lat/lon | — | Center of area to crawl |
| Radius (km) | 1 | 0.5–1 km for testing, 5–10 km for production |
| Grid resolution | 300 | Points per axis — don't change this |

### Indexing time estimates

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

Indexing is **resumable** — interrupted runs continue from the last saved chunk.

### Build the compiled index from parts

After crawling, the GUI auto-builds `index/cosplace_descriptors.npy` + `index/metadata.npz` from `cosplace_parts/*.npz`. You can also trigger this via `build_index.py`:

```bash
python build_index.py
```

---

## Step 2: Search for a Location

### Via GUI

1. Select **Search** mode
2. Upload a street-level photo (JPEG/PNG)
3. Choose search method:
   - **Manual**: provide center lat/lon + radius (km)
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Real-time candidate visualization updates as matches are evaluated
6. Final result: GPS pin on map + confidence score

### Understanding the search internals

```python
# Illustrative — actual search runs inside test_super.py pipeline
import numpy as np
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512) float32
meta        = np.load("index/metadata.npz")
lats        = meta["lats"]      # (N,) float64
lons        = meta["lons"]      # (N,) float64
headings    = meta["headings"]  # (N,) float32
panoids     = meta["panoids"]   # (N,) str

# Extract query descriptor
model = load_cosplace_model()
query_img  = Image.open("query.jpg")
query_desc = extract_descriptor(model, query_img)          # (512,)
flip_desc  = extract_descriptor(model, query_img.transpose(Image.FLIP_LEFT_RIGHT))

# Cosine similarity search (radius pre-filtered via haversine)
from numpy.linalg import norm

def cosine_sim(a, b_matrix):
    return (b_matrix @ a) / (norm(b_matrix, axis=1) * norm(a) + 1e-8)

scores = np.maximum(
    cosine_sim(query_desc, descriptors),
    cosine_sim(flip_desc,  descriptors)
)

# Haversine radius filter (example: 2 km around Paris center)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 2.0

dlat = np.radians(lats - center_lat)
dlon = np.radians(lons - center_lon)
a    = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
dist_km = 6371 * 2 * np.arcsin(np.sqrt(a))

mask        = dist_km <= radius_km
scores[~mask] = -1.0

top_indices = np.argsort(scores)[::-1][:500]   # top 500 candidates
```

---

## Ultra Mode

Enable via the **Ultra Mode** checkbox in the GUI. Adds three enhancements for difficult images (night, blur, low texture):

| Enhancement | What it does |
|-------------|-------------|
| **LoFTR** | Detector-free dense matching — works without clear keypoints |
| **Descriptor hopping** | Re-searches index using the matched panorama's clean descriptor |
| **Neighborhood expansion** | Searches all panoramas within 100 m of best match |

Ultra Mode is significantly slower but improves recall on degraded images.

---

## Multi-Area Index (Single Index, Multiple Cities)

The index is location-agnostic. Index multiple areas and search any with radius filtering:

```python
# Index Paris, London, Tel Aviv — all stored together
# Search is scoped by coordinates + radius at query time

# Search Paris only:
search(center=(48.8566, 2.3522), radius_km=5)

# Search London only:
search(center=(51.5074, -0.1278), radius_km=10)

# No city selector needed — coordinates handle everything
```

---

## Confidence Score Interpretation

| Score range | Meaning |
|-------------|---------|
| High + tight cluster | Strong match, high accuracy |
| High + spread cluster | Visually similar but ambiguous area |
| Low uniqueness ratio | Runner-up nearly as good — treat as uncertain |
| < 50 inliers (Ultra Mode trigger) | Descriptor hopping activates automatically |

---

## Common Patterns

### Pattern 1: Geolocating OSINT imagery with known region

1. Index the suspected city/region (e.g., 5 km radius around city center)
2. Use **Manual** search mode with that region's coordinates
3. Enable Ultra Mode if image is from social media (compressed/low quality)

### Pattern 2: Blind geolocation (no region prior)

1. Use **AI Coarse** mode (requires `GEMINI_API_KEY`) — Gemini reads signs, architecture, vegetation
2. It returns a coarse bounding region
3. Index that region, then run standard search

### Pattern 3: Large-area coverage

Use `build_index.py` for areas > 5 km radius (better performance than GUI indexer):

```bash
# Edit build_index.py to set your center + radius, then:
python build_index.py
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11    # match your Python version
```

### CUDA out of memory

- Reduce candidate count (lower `top_k` in search settings)
- Use DISK instead of ALIKED (fewer keypoints)
- Disable Ultra Mode

### LightGlue import error

```bash
pip install git+https://github.com/cvg/LightGlue.git
# Must install from GitHub — PyPI version is outdated
```

### LoFTR not available (Ultra Mode fails)

```bash
pip install kornia
```

### Index not found on search

Ensure both files exist:
```
index/cosplace_descriptors.npy
index/metadata.npz
```
If missing, re-run `python build_index.py` to compile from `cosplace_parts/`.

### Slow indexing / search on CPU

No GPU workaround — indexing is I/O + compute bound. Options:
- Use a smaller radius (0.5 km for testing)
- Run on a machine with CUDA or Apple Silicon MPS
- Use `build_index.py` (more efficient than GUI path for bulk indexing)

### Resuming interrupted indexing

Just re-run — the indexer checks existing `cosplace_parts/*.npz` files and skips already-processed panoramas.

---

## Key Dependencies

| Package | Purpose | Install |
|---------|---------|---------|
| `lightglue` | Deep feature matching | `pip install git+https://github.com/cvg/LightGlue.git` |
| `kornia` | LoFTR dense matching (Ultra Mode) | `pip install kornia` |
| `torch` | Model inference | via `requirements.txt` |
| `numpy` | Index operations | via `requirements.txt` |
| `Pillow` | Image loading | via `requirements.txt` |
| `google-generativeai` | AI Coarse mode (Gemini) | via `requirements.txt` |

---

## Models Reference

| Model | Stage | Paper |
|-------|-------|-------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global retrieval | CVPR 2022 |
| [ALIKED](https://github.com/naver/alike) | Local features (CUDA) | IEEE TIP 2023 |
| [DISK](https://github.com/cvlab-epfl/disk) | Local features (MPS/CPU) | NeurIPS 2020 |
| [LightGlue](https://github.com/cvg/LightGlue) | Feature matching | ICCV 2023 |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra) | CVPR 2021 |
```
