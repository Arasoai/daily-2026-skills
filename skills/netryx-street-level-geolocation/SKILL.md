```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - reverse geolocate photo
  - netryx geolocation
  - index street view panoramas
  - osint image geolocation
  - locate where a photo was taken
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable visual index using CosPlace embeddings, then matches query images via LightGlue feature matching — achieving sub-50m accuracy with no internet presence required for the target location.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                        # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API key for AI Coarse mode
```bash
export GEMINI_API_KEY="your_key_here"
```

### Requirements
- Python 3.9+ (3.10+ recommended)
- GPU: NVIDIA (CUDA, 4GB+ VRAM) or Apple Silicon (MPS) or CPU (slow)
- RAM: 8GB minimum, 16GB+ recommended
- Storage: 10GB minimum; 50GB+ for large indexed areas

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The GUI walks through this, but the key parameters are:

| Parameter | Description | Default |
|-----------|-------------|---------|
| Center lat/lon | Geographic center of area to index | — |
| Radius (km) | Search radius from center | 0.5–10 km |
| Grid resolution | Panorama density (don't change) | 300 |

**Indexing time estimates:**

| Radius | Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

Indexing is **incremental** — safe to interrupt and resume.

### Step 2 — Search

1. Upload a street-level photo
2. Choose search method:
   - **Manual**: Provide center coordinates + radius (faster, recommended)
   - **AI Coarse**: Gemini analyzes visual cues to estimate region automatically
3. Click **Run Search** → **Start Full Search**
4. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panoid IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)
- Extracts a 512-dim descriptor from the query image
- Also extracts from a horizontally-flipped version (handles reversed perspectives)
- Cosine similarity search against full index, radius-filtered via haversine distance
- Returns top 500–1000 candidates
- **Speed: <1 second** (single matrix multiply)

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)
- Downloads each candidate panorama (8 tiles, stitched)
- Crops at 3 FOVs: 70°, 90°, 110° (handles zoom mismatch)
- Extracts keypoints: **ALIKED** on CUDA, **DISK** on MPS/CPU
- LightGlue deep feature matching + RANSAC filtering
- **Speed: 2–5 min for 300–500 candidates**

### Stage 3 — Refinement
- Heading refinement: ±45° at 15° steps, top 15 candidates
- Spatial consensus: clusters matches into 50m cells
- Confidence scoring: uniqueness ratio + geographic clustering

### Ultra Mode (difficult images)
Enable for: night photos, motion blur, low texture, degraded images.
Adds:
- **LoFTR**: detector-free dense matching (handles blur/low contrast)
- **Descriptor hopping**: re-searches index using matched panorama's clean descriptor
- **Neighborhood expansion**: searches all panoramas within 100m of best match

---

## Using CosPlace Utils Directly

```python
# cosplace_utils.py exposes descriptor extraction
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"

model = get_cosplace_model(device=device)

# Extract 512-dim descriptor from an image file
descriptor = get_descriptor(model, "path/to/street_photo.jpg", device=device)
print(descriptor.shape)  # (512,)

# Also extract flipped version for better retrieval
import PIL.Image
import torchvision.transforms as T

img = PIL.Image.open("path/to/street_photo.jpg")
img_flipped = img.transpose(PIL.Image.FLIP_LEFT_RIGHT)
desc_flipped = get_descriptor(model, img_flipped, device=device)
```

---

## Building a Large Index (CLI)

For large areas, use the standalone builder instead of the GUI:

```bash
python build_index.py
```

This compiles all `cosplace_parts/*.npz` chunks into the searchable `index/` directory:
- `index/cosplace_descriptors.npy` — stacked descriptor matrix
- `index/metadata.npz` — lat, lon, heading, panoid per entry

---

## Index Architecture

All cities share **one unified index**. Radius filtering at search time handles isolation:

```python
# Conceptual: how radius filtering works internally
import numpy as np

def haversine_filter(meta_lats, meta_lons, center_lat, center_lon, radius_km):
    """Filter index entries within radius_km of center."""
    R = 6371.0
    dlat = np.radians(meta_lats - center_lat)
    dlon = np.radians(meta_lons - center_lon)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * \
        np.cos(np.radians(meta_lats)) * np.sin(dlon/2)**2
    distances = 2 * R * np.arcsin(np.sqrt(a))
    return distances <= radius_km

# Index multiple cities, search independently
# Index Paris → same index
# Index Tokyo → same index
# Search: center=(48.85, 2.35), radius=5 → only Paris results returned
```

---

## GPU / Device Behavior

| Feature | CUDA (NVIDIA) | MPS (Apple) | CPU |
|---------|--------------|-------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| CosPlace | ✅ Full speed | ✅ Full speed | ✅ Slower |
| LightGlue | ✅ Fast | ✅ Good | ⚠️ Slow |
| LoFTR (Ultra) | ✅ | ✅ | ⚠️ Very slow |

```python
import torch

def get_device():
    if torch.cuda.is_available():
        return "cuda"
    elif hasattr(torch.backends, "mps") and torch.backends.mps.is_available():
        return "mps"
    return "cpu"
```

---

## Real Code Examples

### Load and search the index manually

```python
import numpy as np
from cosplace_utils import get_cosplace_model, get_descriptor

# Load pre-built index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz")
lats = meta["lats"]       # (N,)
lons = meta["lons"]       # (N,)
headings = meta["headings"]
panoids = meta["panoids"]

device = "cuda"  # or "mps" / "cpu"
model = get_cosplace_model(device=device)

# Extract query descriptor
query_desc = get_descriptor(model, "query_photo.jpg", device=device)  # (512,)

# Radius filter (e.g., Paris, 2km radius)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 2.0

R = 6371.0
dlat = np.radians(lats - center_lat)
dlon = np.radians(lons - center_lon)
a = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
distances_km = 2 * R * np.arcsin(np.sqrt(a))
mask = distances_km <= radius_km

filtered_desc = descriptors[mask]         # (M, 512)
filtered_lats = lats[mask]
filtered_lons = lons[mask]
filtered_panoids = panoids[mask]

# Cosine similarity search
query_norm = query_desc / np.linalg.norm(query_desc)
db_norms = filtered_desc / np.linalg.norm(filtered_desc, axis=1, keepdims=True)
similarities = db_norms @ query_norm      # (M,)

# Top 10 candidates
top_k = 10
top_indices = np.argsort(similarities)[::-1][:top_k]

for i, idx in enumerate(top_indices):
    print(f"Rank {i+1}: ({filtered_lats[idx]:.6f}, {filtered_lons[idx]:.6f}) "
          f"sim={similarities[idx]:.4f} panoid={filtered_panoids[idx]}")
```

### Using flipped descriptor for better retrieval

```python
import PIL.Image
import numpy as np

def get_combined_score(model, image_path, db_descriptors, device):
    """Score combining normal + flipped descriptor."""
    img = PIL.Image.open(image_path).convert("RGB")
    img_flipped = img.transpose(PIL.Image.FLIP_LEFT_RIGHT)

    desc = get_descriptor(model, img, device=device)
    desc_flip = get_descriptor(model, img_flipped, device=device)

    # Normalize
    db_norm = db_descriptors / np.linalg.norm(db_descriptors, axis=1, keepdims=True)
    d1 = desc / np.linalg.norm(desc)
    d2 = desc_flip / np.linalg.norm(desc_flip)

    # Average similarity scores
    sim1 = db_norm @ d1
    sim2 = db_norm @ d2
    return (sim1 + sim2) / 2
```

---

## Common Patterns

### Pattern: Known region, precise search
Use **Manual** mode. Provide tight radius (0.5–2 km) for faster, more accurate results.

### Pattern: Unknown region (blind geolocation)
Use **AI Coarse** mode with `GEMINI_API_KEY` set. Gemini estimates country/city from visual cues, then Manual search runs in that region.

### Pattern: Difficult image (night / blur)
Enable **Ultra Mode** checkbox. Expect 3–10× longer search time but significantly better recall.

### Pattern: Multiple city coverage
Index each city separately (same index file), search with appropriate center coordinates per query. No city tagging needed.

### Pattern: Interrupted indexing
Just re-run — `cosplace_parts/` chunks are saved incrementally. The builder skips already-processed panoramas.

---

## Troubleshooting

### `ImportError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### GUI renders blank on macOS
```bash
brew install python-tk@3.11   # match your exact Python version
```

### CUDA out of memory
- Reduce candidate count in GUI (lower top-K from 500 to 200)
- Use a smaller index radius
- Switch from ALIKED (1024 kp) to fewer keypoints by editing the extractor config in `test_super.py`

### MPS errors on Apple Silicon
```bash
# Force CPU if MPS is unstable
export PYTORCH_ENABLE_MPS_FALLBACK=1
```

### Index search returns no results
- Verify the index was built: check `index/cosplace_descriptors.npy` exists
- Confirm your search center/radius actually overlaps the indexed area
- Re-run `python build_index.py` to recompile parts into the searchable index

### Very low confidence scores (<30 inliers)
- Try Ultra Mode
- Widen search radius
- The query area may not be indexed — run indexing for that region first

### Slow indexing
- Indexing is I/O and network-bound (Street View API calls), not GPU-bound
- Running overnight for 5–10 km radius areas is normal
- Use `build_index.py` (not GUI) for large-scale indexing

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim visual fingerprint | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoint extraction | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoint extraction | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra Mode) | All (via kornia) |
```
