```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision pipelines.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - identify location from street photo
  - open source geolocation pipeline
  - run netryx geolocation
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes Street View panoramas, extracts visual fingerprints using CosPlace, then verifies matches geometrically using ALIKED/DISK + LightGlue. No landmark recognition required — it works on any random street corner.

**Pipeline summary:**
1. **Stage 1 — Global Retrieval**: CosPlace 512-dim descriptor → cosine similarity search over indexed panoramas
2. **Stage 2 — Local Verification**: ALIKED/DISK keypoints + LightGlue matching + RANSAC on top 500 candidates
3. **Stage 3 — Refinement**: Heading sweeps, spatial clustering, confidence scoring
4. **Ultra Mode** (optional): LoFTR dense matching + descriptor hopping + neighborhood expansion

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must be installed from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

**macOS tkinter fix** (if GUI renders blank):
```bash
brew install python-tk@3.11   # match your Python version
```

### Optional: Gemini API for AI Coarse region estimation

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM | 4 GB | 8 GB+ |
| RAM | 8 GB | 16 GB+ |
| Storage | 10 GB | 50 GB+ |
| Python | 3.9+ | 3.10+ |

- **Mac M1+**: Uses MPS (DISK extractor, 768 keypoints)
- **NVIDIA GPU**: Uses CUDA (ALIKED extractor, 1024 keypoints)
- **CPU**: Supported but significantly slower

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (index an area) and **Search** (geolocate a photo).

---

## Step 1: Creating an Index

An index must exist before searching. It crawls Street View panoramas in a geographic area and stores CosPlace descriptors.

### Via GUI

1. Select **Create** mode
2. Enter center lat/lon (e.g., `48.8566, 2.3522` for Paris)
3. Set radius in km (start with `0.5`–`1` for testing)
4. Set grid resolution (default `300` — do not change)
5. Click **Create Index**

The index saves incrementally to `cosplace_parts/` and resumes if interrupted.

### Index Size Estimates

| Radius | Panoramas | Time (M2 Max) | Disk |
|--------|-----------|---------------|------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

### Build Index from Existing Parts (Large Datasets)

Use the standalone high-performance builder after crawling:

```bash
python build_index.py
```

This compiles `cosplace_parts/*.npz` into the searchable `index/` directory:
- `index/cosplace_descriptors.npy` — all 512-dim descriptors
- `index/metadata.npz` — coordinates, headings, panorama IDs

---

## Step 2: Searching (Geolocating a Photo)

### Via GUI

1. Select **Search** mode
2. Click **Upload** and select a street-level image
3. Choose search method:
   - **Manual**: Enter known approximate center lat/lon + radius
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Results appear on the map with GPS coordinates and confidence score

### Ultra Mode

Enable the **Ultra Mode** checkbox before searching for:
- Night photos
- Blurry or low-contrast images
- Low-texture scenes

Ultra Mode adds LoFTR dense matching, descriptor hopping (re-searches using the matched panorama's clean descriptor), and 100m neighborhood expansion.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading and descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # Compiled descriptor matrix
    └── metadata.npz               # Panorama metadata (lat, lon, heading, panoid)
```

---

## Using CosPlace Utils Programmatically

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load the model (downloads weights on first run)
model = load_cosplace_model()
device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")
model = model.to(device)
model.eval()

# Extract a 512-dim descriptor from an image
img = Image.open("street_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)
# descriptor.shape == (512,)
```

---

## Loading and Querying the Index Directly

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# Load the prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # shape: (N,)
lons = meta["lons"]       # shape: (N,)
headings = meta["headings"]
panoids = meta["panoids"]

# Query: find top-K most similar panoramas
def search_index(query_descriptor, top_k=500):
    """
    query_descriptor: np.ndarray of shape (512,)
    Returns indices of top_k most similar entries.
    """
    query = query_descriptor.reshape(1, -1)
    sims = cosine_similarity(query, descriptors)[0]   # shape: (N,)
    top_indices = np.argsort(sims)[::-1][:top_k]
    return top_indices, sims[top_indices]

# Radius filter using haversine distance
def haversine_km(lat1, lon1, lat2, lon2):
    from math import radians, sin, cos, sqrt, atan2
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

def search_with_radius(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """Filter index to radius, then find top-K by cosine similarity."""
    # Build mask of panoramas within radius
    mask = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
        for i in range(len(lats))
    ])
    local_indices = np.where(mask)[0]
    local_descriptors = descriptors[local_indices]

    query = query_descriptor.reshape(1, -1)
    sims = cosine_similarity(query, local_descriptors)[0]
    ranked = np.argsort(sims)[::-1][:top_k]

    result_indices = local_indices[ranked]
    return result_indices, sims[ranked]
```

---

## Programmatic Pipeline Example

```python
"""
Full headless geolocation: extract descriptor, search index,
return top candidate coordinates.
"""
import numpy as np
import torch
from PIL import Image

from cosplace_utils import load_cosplace_model, extract_descriptor

def geolocate(image_path: str, center_lat: float, center_lon: float,
              radius_km: float = 2.0, top_k: int = 500):
    """
    Returns a list of dicts with candidate matches sorted by similarity.
    Each dict: {lat, lon, heading, panoid, similarity}
    """
    # Device setup
    if torch.cuda.is_available():
        device = torch.device("cuda")
    elif torch.backends.mps.is_available():
        device = torch.device("mps")
    else:
        device = torch.device("cpu")

    # Load model and image
    model = load_cosplace_model().to(device).eval()
    img = Image.open(image_path).convert("RGB")

    # Extract query descriptors (original + horizontally flipped)
    desc = extract_descriptor(model, img, device)
    desc_flip = extract_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT), device)
    query_desc = (desc + desc_flip) / 2.0   # average-pool both views
    query_desc /= np.linalg.norm(query_desc)

    # Load index
    descriptors = np.load("index/cosplace_descriptors.npy")
    meta = np.load("index/metadata.npz", allow_pickle=True)
    lats, lons = meta["lats"], meta["lons"]
    headings, panoids = meta["headings"], meta["panoids"]

    # Radius-filtered cosine search
    from math import radians, sin, cos, sqrt, atan2
    def hav(la1, lo1, la2, lo2):
        R = 6371.0
        dlat, dlon = radians(la2-la1), radians(lo2-lo1)
        a = sin(dlat/2)**2 + cos(radians(la1))*cos(radians(la2))*sin(dlon/2)**2
        return R * 2 * atan2(sqrt(a), sqrt(1-a))

    mask = np.array([hav(center_lat, center_lon, lats[i], lons[i]) <= radius_km
                     for i in range(len(lats))])
    local_idx = np.where(mask)[0]

    from sklearn.metrics.pairwise import cosine_similarity
    sims = cosine_similarity(query_desc.reshape(1, -1), descriptors[local_idx])[0]
    ranked = np.argsort(sims)[::-1][:top_k]

    results = []
    for rank_pos, idx in enumerate(ranked):
        global_idx = local_idx[idx]
        results.append({
            "rank": rank_pos + 1,
            "lat": float(lats[global_idx]),
            "lon": float(lons[global_idx]),
            "heading": float(headings[global_idx]),
            "panoid": str(panoids[global_idx]),
            "similarity": float(sims[idx]),
        })

    return results

# Example usage
if __name__ == "__main__":
    candidates = geolocate(
        image_path="mystery_street.jpg",
        center_lat=48.8566,
        center_lon=2.3522,
        radius_km=3.0,
        top_k=500,
    )
    best = candidates[0]
    print(f"Best match: {best['lat']:.6f}, {best['lon']:.6f} "
          f"(similarity={best['similarity']:.4f}, panoid={best['panoid']})")
```

---

## Multi-City Index Strategy

All cities share a single index. The radius filter at search time handles isolation:

```python
# Index Paris center
# center_lat=48.8566, center_lon=2.3522, radius=5km

# Index London center
# center_lat=51.5074, center_lon=-0.1278, radius=5km

# Search only London results (Paris entries are filtered out automatically)
candidates = geolocate("photo.jpg", center_lat=51.5074, center_lon=-0.1278, radius_km=5.0)

# Search only Paris results
candidates = geolocate("photo.jpg", center_lat=48.8566, center_lon=2.3522, radius_km=5.0)
```

---

## Common Patterns

### Batch Geolocation of Multiple Images

```python
import os
from pathlib import Path

image_dir = Path("images_to_geolocate/")
results_log = []

for img_path in image_dir.glob("*.jpg"):
    try:
        candidates = geolocate(
            image_path=str(img_path),
            center_lat=48.8566,
            center_lon=2.3522,
            radius_km=5.0,
            top_k=300,
        )
        best = candidates[0]
        results_log.append({
            "file": img_path.name,
            "lat": best["lat"],
            "lon": best["lon"],
            "confidence": best["similarity"],
        })
        print(f"{img_path.name}: {best['lat']:.5f}, {best['lon']:.5f}")
    except Exception as e:
        print(f"Failed on {img_path.name}: {e}")
```

### Export Results to GeoJSON

```python
import json

def results_to_geojson(results: list) -> dict:
    features = []
    for r in results:
        features.append({
            "type": "Feature",
            "geometry": {"type": "Point", "coordinates": [r["lon"], r["lat"]]},
            "properties": {
                "rank": r["rank"],
                "similarity": r["similarity"],
                "panoid": r["panoid"],
                "heading": r["heading"],
            }
        })
    return {"type": "FeatureCollection", "features": features}

candidates = geolocate("photo.jpg", 48.8566, 2.3522, radius_km=2.0)
geojson = results_to_geojson(candidates[:10])
with open("results.geojson", "w") as f:
    json.dump(geojson, f, indent=2)
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # match your Python version exactly
```

### `ImportError: No module named 'lightglue'`
LightGlue must be installed from GitHub, not PyPI:
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### Index search returns zero candidates
- The radius may be too small for the indexed area, or the center coordinates are outside any indexed region.
- Check that `index/cosplace_descriptors.npy` and `index/metadata.npz` exist and are non-empty.
- Run `python build_index.py` if `cosplace_parts/` has data but `index/` is missing/stale.

### CUDA out of memory during search
- Reduce `top_k` from 500 to 200–300
- Disable Ultra Mode
- Reduce GPU batch size inside `test_super.py` if configurable

### Low confidence / wrong result
1. Enable **Ultra Mode** for degraded images
2. Increase index density (smaller grid resolution or larger radius during indexing)
3. Try **AI Coarse** mode to get a better center estimate before searching
4. Ensure the query image is street-level and not aerial/indoor

### Indexing interrupted — resuming
No action needed. Netryx saves `cosplace_parts/*.npz` incrementally. Re-run **Create Index** with the same parameters and it will skip already-processed grid points.

### MPS (Apple Silicon) slower than expected
DISK is used on MPS instead of ALIKED. This is expected and normal — ALIKED requires CUDA. Performance on M1/M2/M3/M4 is still practical for search.

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace | Global descriptor (512-dim) | All |
| ALIKED | Local keypoints (1024 kp) | CUDA only |
| DISK | Local keypoints (768 kp) | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | All (slow on CPU) |
```
