---
name: netryx-street-level-geolocation
description: Use Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from any street photo using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - use netryx to locate a photo
  - index street view panoramas
  - run netryx geolocation search
  - identify location from street image
  - osint geolocation from photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature extraction (ALIKED/DISK), and deep feature matching (LightGlue). No cloud APIs required for searching — runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue from source
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Optional: Gemini API for AI Coarse region detection

```bash
export GEMINI_API_KEY="your_key_here"
```

### Platform GPU Support

| Platform | Accelerator | Feature Extractor Used |
|----------|-------------|----------------------|
| NVIDIA GPU | CUDA | ALIKED (1024 keypoints) |
| Apple Silicon | MPS | DISK (768 keypoints) |
| CPU only | None | DISK (slower) |

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix:** `brew install python-tk@3.11` (match your Python version)

The GUI has two primary modes: **Create** (index an area) and **Search** (geolocate a query image).

---

## Core Workflow

### Step 1 — Index an Area (Create Mode)

Before searching, you must crawl and index street-view panoramas for your target region.

**GUI steps:**
1. Select **Create** mode
2. Enter center latitude/longitude of target area
3. Set search radius (start with `0.5`–`1` km for testing)
4. Set grid resolution (default `300` — do not change unless you know what you're doing)
5. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Indexing is **resumable** — interrupting and restarting continues from the last checkpoint.

**Index output files:**
```
cosplace_parts/          # Raw embedding chunks (created incrementally)
index/
├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
└── metadata.npz               # Coordinates, headings, panorama IDs
```

### Step 2 — Search (Geolocate a Photo)

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Enter approximate center coordinates + radius (faster, more precise)
   - **AI Coarse**: Uses Gemini to infer region from visual cues (signs, architecture, vegetation)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on a map

---

## Pipeline Architecture

```
Query Image
    │
    ├─ CosPlace descriptor (512-dim fingerprint)
    ├─ Flipped descriptor (catches reversed perspectives)
    │
    ▼
Index Search (cosine similarity + haversine radius filter)
    │
    └─ Top 500–1000 candidates
    │
    ▼
For each candidate:
    ├─ Download panorama (8 tiles, stitched)
    ├─ Crop at indexed heading angle
    ├─ Multi-FOV crops (70°, 90°, 110°)
    ├─ ALIKED/DISK keypoint extraction
    ├─ LightGlue feature matching
    └─ RANSAC geometric verification (inlier count)
    │
    ▼
Heading Refinement
    ├─ Top 15 candidates × ±45° offsets × 3 FOVs
    │
    ▼
Spatial Consensus Clustering (50m cells)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable **Ultra Mode** for difficult images (night, motion blur, low texture). Adds three additional techniques:

| Technique | What it does |
|-----------|-------------|
| **LoFTR** | Detector-free dense matcher — works without clear keypoints |
| **Descriptor hopping** | If best match is weak (<50 inliers), re-searches index using the matched panorama's clean descriptor |
| **Neighborhood expansion** | Searches all panoramas within 100m of best match |

Ultra Mode is significantly slower but catches matches the standard pipeline misses.

---

## Multi-Area Index (Key Design Point)

All indexed areas share **one unified index**. The radius filter at search time restricts results geographically. You never need to select a "city" — coordinates + radius handle everything.

```
# Index Paris, then London, then Tokyo — all into the same index
# Search Paris:  center=(48.8566, 2.3522), radius=5km → returns only Paris results
# Search London: center=(51.5074, -0.1278), radius=5km → returns only London results
```

---

## Working with the Index Programmatically

### Load and Query the Index Directly

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # shape: (N, 512)
metadata = np.load("index/metadata.npz", allow_pickle=True)
lats = metadata["lats"]      # float array of latitudes
lons = metadata["lons"]      # float array of longitudes
headings = metadata["headings"]
panoids = metadata["panoids"]

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * asin(sqrt(a))

def radius_filter(center_lat, center_lon, radius_km):
    """Return boolean mask of index entries within radius."""
    mask = np.array([
        haversine_km(center_lat, center_lon, lat, lon) <= radius_km
        for lat, lon in zip(lats, lons)
    ])
    return mask

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """Search index with radius filter, return top-k candidate indices."""
    mask = radius_filter(center_lat, center_lon, radius_km)
    filtered_descriptors = descriptors[mask]
    filtered_indices = np.where(mask)[0]

    # Cosine similarity (descriptors should be L2-normalized)
    query_norm = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    sims = filtered_descriptors @ query_norm
    top_k_local = np.argsort(sims)[::-1][:top_k]
    return filtered_indices[top_k_local], sims[top_k_local]
```

### Extract CosPlace Descriptor from an Image

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

device = (
    torch.device("cuda") if torch.cuda.is_available()
    else torch.device("mps") if torch.backends.mps.is_available()
    else torch.device("cpu")
)

model = get_cosplace_model(device=device)

def extract_descriptor(image_path):
    img = Image.open(image_path).convert("RGB")
    descriptor = get_descriptor(model, img, device=device)  # returns (512,) numpy array
    # Also extract flipped version for robustness
    descriptor_flipped = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT), device=device)
    # Combine by averaging (or use max similarity during search)
    return descriptor, descriptor_flipped
```

### Run a Full Programmatic Search

```python
# Full search using index
query_image_path = "query.jpg"
center_lat, center_lon = 48.8566, 2.3522  # Paris
radius_km = 2.0

desc, desc_flipped = extract_descriptor(query_image_path)

# Search with both descriptors, merge results
indices_1, sims_1 = search_index(desc, center_lat, center_lon, radius_km, top_k=500)
indices_2, sims_2 = search_index(desc_flipped, center_lat, center_lon, radius_km, top_k=500)

# Combine and deduplicate (keep best similarity per index entry)
combined = {}
for idx, sim in zip(list(indices_1) + list(indices_2), list(sims_1) + list(sims_2)):
    if idx not in combined or sim > combined[idx]:
        combined[idx] = sim

top_candidates = sorted(combined.items(), key=lambda x: x[1], reverse=True)[:500]
print(f"Top candidate: panoid={panoids[top_candidates[0][0]]}, "
      f"lat={lats[top_candidates[0][0]]:.6f}, "
      f"lon={lons[top_candidates[0][0]]:.6f}, "
      f"similarity={top_candidates[0][1]:.4f}")
```

---

## Build Index Programmatically (Large Areas)

For large datasets, use the standalone high-performance index builder:

```bash
python build_index.py
```

Or integrate into a script:

```python
# build_index.py handles:
# - Merging all cosplace_parts/*.npz into the unified index
# - Building cosplace_descriptors.npy and metadata.npz
# Run after indexing completes or to rebuild after partial indexing
```

---

## Project Structure

```
netryx/
├── test_super.py           # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py       # CosPlace model loading + descriptor extraction
├── build_index.py          # Standalone index builder for large datasets
├── requirements.txt        # Python dependencies
├── cosplace_parts/         # Raw embedding chunks (incremental, resumable)
└── index/
    ├── cosplace_descriptors.npy   # Compiled 512-dim descriptor matrix
    └── metadata.npz               # lat, lon, heading, panoid per entry
```

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace | Global 512-dim visual fingerprint | All |
| ALIKED | Keypoint extraction (high quality) | CUDA only |
| DISK | Keypoint extraction (fallback) | MPS / CPU |
| LightGlue | Deep feature matching + RANSAC | All |
| LoFTR | Dense matching (Ultra Mode) | All (needs `kornia`) |

---

## Common Patterns

### Pattern: Check Device and Confirm GPU Active

```python
import torch

def get_device():
    if torch.cuda.is_available():
        print(f"Using CUDA: {torch.cuda.get_device_name(0)}")
        return torch.device("cuda")
    elif torch.backends.mps.is_available():
        print("Using MPS (Apple Silicon)")
        return torch.device("mps")
    else:
        print("Warning: Using CPU — search will be slow")
        return torch.device("cpu")

device = get_device()
```

### Pattern: Verify Index Integrity

```python
import numpy as np
import os

def verify_index():
    desc_path = "index/cosplace_descriptors.npy"
    meta_path = "index/metadata.npz"

    if not os.path.exists(desc_path) or not os.path.exists(meta_path):
        print("Index not found. Run indexing first via Create mode.")
        return False

    descriptors = np.load(desc_path)
    metadata = np.load(meta_path, allow_pickle=True)

    n_desc = descriptors.shape[0]
    n_meta = len(metadata["lats"])

    print(f"Descriptors: {n_desc} entries, shape {descriptors.shape}")
    print(f"Metadata: {n_meta} entries")

    if n_desc != n_meta:
        print("WARNING: Descriptor/metadata count mismatch — rebuild index with build_index.py")
        return False

    print("Index OK")
    return True

verify_index()
```

### Pattern: Batch Index Multiple Cities

```python
# Example: sequentially index multiple areas into the unified index
# via the GUI, or by calling the indexing functions programmatically

cities = [
    {"name": "Paris Center", "lat": 48.8566, "lon": 2.3522, "radius_km": 1.0},
    {"name": "London Center", "lat": 51.5074, "lon": -0.1278, "radius_km": 1.0},
    {"name": "Berlin Center", "lat": 52.5200, "lon": 13.4050, "radius_km": 1.0},
]

# All cities go into the same unified index.
# When searching, specify the correct center + radius to isolate the target city.
for city in cities:
    print(f"Index {city['name']}: center=({city['lat']}, {city['lon']}), radius={city['radius_km']}km")
    # Launch GUI with these params, or call indexing function directly from test_super.py
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # Replace 3.11 with your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
```

### Index and metadata count mismatch
```bash
# Rebuild the compiled index from raw parts
python build_index.py
```

### Search returns no results / low confidence
- Verify the indexed area covers your query image's location
- Expand the search radius
- Enable Ultra Mode for blurry or low-quality images
- Ensure the index was built with sufficient grid resolution (default: 300)

### CUDA out of memory
- Reduce top-k candidates in search (GUI setting)
- Use a GPU with more VRAM (8GB+ recommended)
- Fall back to MPS or CPU (automatic if CUDA fails)

### Slow indexing
- Use `build_index.py` for large areas — it is optimized for batch processing
- Indexing is resumable; restart picks up from last checkpoint automatically

### Gemini AI Coarse mode not working
```bash
# Ensure API key is exported before launching
export GEMINI_API_KEY="your_key_here"
python test_super.py
```

---

## Key Parameters Cheat Sheet

| Parameter | Recommended Default | Notes |
|-----------|--------------------|----|
| Grid resolution | 300 | Don't change — controls panorama density |
| Search radius (testing) | 0.5–1 km | Small area for fast iteration |
| Search radius (production) | 5–10 km | City-scale search |
| Top candidates (Stage 1) | 500–1000 | Higher = slower Stage 2 |
| Ultra Mode inlier threshold | 50 | Below this triggers descriptor hopping |
| Heading refinement range | ±45° @ 15° steps | Applied to top 15 candidates |
| Spatial clustering cell size | 50m | For consensus scoring |
