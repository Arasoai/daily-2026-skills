```markdown
---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - run netryx search
  - use netryx to locate a photo
  - open source geolocation tool
  - osint geolocation from image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that takes any street-level photograph and returns precise GPS coordinates (sub-50m accuracy). It indexes Street View panoramas into a searchable vector database using CosPlace embeddings, then verifies candidates using local feature matching (ALIKED/DISK + LightGlue). No internet lookups of the query image — it matches against the physical world.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                        # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"   # get from aistudio.google.com
```

### GPU support

| Hardware | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU (4GB+ VRAM) | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon (M1+) | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix:** `brew install python-tk@3.11` (match your Python version)

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area by crawling Street View panoramas and extracting CosPlace fingerprints.

In the GUI:
1. Select **Create** mode
2. Enter center `lat, lon`
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Indexing is **resumable** — safe to interrupt and restart.

Multiple regions (Paris, London, Tel Aviv) can share one unified index. The radius filter in search isolates the correct region automatically.

### Step 2 — Search

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: provide approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes the image for visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### Ultra Mode

Enable **Ultra Mode** for difficult images (night shots, motion blur, low texture). Adds:
- **LoFTR** dense detector-free matching
- **Descriptor hopping** (re-searches index using matched panorama's clean descriptor)
- **Neighborhood expansion** (searches all panoramas within 100m of best match)

Slower but catches matches that the standard pipeline misses.

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace descriptor (512-dim)
    ├── Flipped image descriptor (catches reversed perspectives)
    │
    ▼
Index Search
    ├── Cosine similarity against all indexed descriptors
    ├── Haversine radius filter (your specified area)
    └── Top 500–1000 candidates returned
    │
    ▼
Geometric Verification (per candidate)
    ├── Download panorama (8 tiles, stitched)
    ├── Rectilinear crops at 3 FOVs: 70°, 90°, 110°
    ├── ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    ├── LightGlue feature matching vs. query keypoints
    └── RANSAC inlier filtering
    │
    ▼
Refinement
    ├── Heading refinement: ±45° at 15° steps, top 15 candidates
    ├── Spatial consensus: cluster matches into 50m cells
    └── Confidence scoring: clustering density + uniqueness ratio
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Using CosPlace Utils Directly

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA / MPS / CPU)
model = load_cosplace_model()

# Extract a 512-dim descriptor from an image file
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = get_descriptor(model, img)   # shape: (512,)

# Compare two descriptors
from torch.nn.functional import cosine_similarity
sim = cosine_similarity(
    torch.tensor(desc_a).unsqueeze(0),
    torch.tensor(desc_b).unsqueeze(0)
)
print(f"Similarity: {sim.item():.4f}")   # 1.0 = identical
```

---

## Building the Index Programmatically

```python
# build_index.py is a standalone high-performance builder
# Run directly for large datasets:
python build_index.py
```

Or trigger index auto-build after chunked indexing completes (merges `cosplace_parts/*.npz` into the searchable index):

```python
import numpy as np
import os
from pathlib import Path

# After cosplace_parts/ is populated, merge into searchable index:
parts_dir = Path("cosplace_parts")
all_descriptors = []
all_metadata = {"lats": [], "lons": [], "headings": [], "panoids": []}

for npz_file in sorted(parts_dir.glob("*.npz")):
    data = np.load(npz_file, allow_pickle=True)
    all_descriptors.append(data["descriptors"])
    all_metadata["lats"].extend(data["lats"])
    all_metadata["lons"].extend(data["lons"])
    all_metadata["headings"].extend(data["headings"])
    all_metadata["panoids"].extend(data["panoids"])

descriptors = np.vstack(all_descriptors)
os.makedirs("index", exist_ok=True)
np.save("index/cosplace_descriptors.npy", descriptors)
np.savez("index/metadata.npz", **{k: np.array(v) for k, v in all_metadata.items()})
print(f"Index built: {len(descriptors)} panoramas")
```

---

## Searching the Index Programmatically

```python
import numpy as np
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image

def haversine_km(lat1, lon1, lat2, lon2):
    """Returns distance in km between two lat/lon points."""
    import math
    R = 6371
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * \
        math.cos(math.radians(lat2)) * math.sin(dlon/2)**2
    return R * 2 * math.asin(math.sqrt(a))

def search_index(query_image_path, center_lat, center_lon, radius_km=2.0, top_k=500):
    # Load index
    descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
    meta = np.load("index/metadata.npz", allow_pickle=True)
    lats, lons = meta["lats"], meta["lons"]

    # Extract query descriptor
    model = load_cosplace_model()
    img = Image.open(query_image_path).convert("RGB")
    query_desc = get_descriptor(model, img)                    # (512,)

    # Also get flipped descriptor
    query_desc_flipped = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT))

    # Cosine similarity (both orientations, take max)
    sims = descriptors @ query_desc                            # (N,)
    sims_flipped = descriptors @ query_desc_flipped
    sims = np.maximum(sims, sims_flipped)

    # Radius filter
    in_radius = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
        for i in range(len(lats))
    ])

    # Rank candidates within radius
    masked_sims = np.where(in_radius, sims, -1)
    top_indices = np.argsort(masked_sims)[::-1][:top_k]

    candidates = [
        {
            "index": int(i),
            "lat": float(lats[i]),
            "lon": float(lons[i]),
            "heading": float(meta["headings"][i]),
            "panoid": str(meta["panoids"][i]),
            "similarity": float(masked_sims[i]),
        }
        for i in top_indices if masked_sims[i] > 0
    ]
    return candidates

# Example usage
candidates = search_index(
    query_image_path="mystery_street.jpg",
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=3.0,
    top_k=500
)
print(f"Found {len(candidates)} candidates")
print(f"Top match: {candidates[0]}")
```

---

## Common Patterns

### Pattern 1: Batch geolocate multiple images

```python
from pathlib import Path

image_dir = Path("images_to_geolocate")
results = {}

for img_path in image_dir.glob("*.jpg"):
    candidates = search_index(
        query_image_path=str(img_path),
        center_lat=48.8566,
        center_lon=2.3522,
        radius_km=5.0,
        top_k=300,
    )
    if candidates:
        best = candidates[0]
        results[img_path.name] = {
            "lat": best["lat"],
            "lon": best["lon"],
            "confidence": best["similarity"],
        }
        print(f"{img_path.name} → {best['lat']:.6f}, {best['lon']:.6f} (sim={best['similarity']:.3f})")

import json
with open("geolocation_results.json", "w") as f:
    json.dump(results, f, indent=2)
```

### Pattern 2: Check index health

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

print(f"Total indexed panoramas : {len(descriptors)}")
print(f"Descriptor shape        : {descriptors.shape}")
print(f"Lat range               : {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range               : {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
print(f"Unique panoids          : {len(set(meta['panoids']))}")
```

### Pattern 3: Resume interrupted indexing

Indexing is automatically resumable. Chunks are saved to `cosplace_parts/` incrementally. Simply re-run:

```bash
python test_super.py   # Select Create mode, same params — resumes from last chunk
```

Or rebuild the merged index from existing parts without re-crawling:

```bash
python build_index.py
```

---

## Configuration Reference

All configuration is set through the GUI or directly in `test_super.py`. Key parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `center_lat`, `center_lon` | — | Center of search/index area |
| `radius_km` | 1.0 | Search/index radius in km |
| `grid_resolution` | 300 | Grid density for indexing (don't change) |
| `top_k_candidates` | 500–1000 | CosPlace retrieval candidates |
| `top_k_refine` | 15 | Candidates for heading refinement |
| `heading_steps` | ±45° @ 15° | Heading offsets tested in refinement |
| `fovs` | [70, 90, 110] | Fields of view for panorama crops |
| `cluster_radius_m` | 50 | Spatial consensus cell size |
| `ultra_mode` | False | Enable LoFTR + descriptor hopping |
| `neighbor_radius_m` | 100 | Neighborhood expansion in Ultra Mode |

---

## Troubleshooting

### GUI appears blank (macOS)
```bash
brew install python-tk@3.11   # match your Python version exactly
```

### `No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### `No module named 'kornia'` (Ultra Mode only)
```bash
pip install kornia
```

### CUDA out of memory
- Reduce `top_k_candidates` to 200–300 in the GUI
- Use CPU fallback: the pipeline auto-detects and downgrades gracefully

### Index not found / empty results
```bash
# Verify index files exist
ls -lh index/cosplace_descriptors.npy index/metadata.npz

# If missing, rebuild from parts:
python build_index.py
```

### Low confidence scores (<0.3) / wrong location
1. Enable **Ultra Mode** for degraded images
2. Expand search radius — the correct area may not be fully indexed
3. Verify the indexed area actually covers the photo's location
4. Use **AI Coarse** mode if region is unknown (requires `GEMINI_API_KEY`)

### Slow indexing
- Indexing is I/O-bound (Street View API calls), not GPU-bound — expected behavior
- Use `build_index.py` standalone for large datasets (more efficient than GUI)
- Indexing resumes automatically on restart — safe to run overnight

### `GEMINI_API_KEY` not found
```bash
export GEMINI_API_KEY="your_key_here"
# Or add to .env and load with python-dotenv
```

---

## Model Reference

| Model | Role | Used when |
|-------|------|-----------|
| CosPlace | Global 512-dim visual fingerprint | Always (retrieval stage) |
| ALIKED | Local keypoint extraction | CUDA hardware |
| DISK | Local keypoint extraction | MPS / CPU hardware |
| LightGlue | Deep feature matching | Always (verification stage) |
| LoFTR | Dense detector-free matching | Ultra Mode only |

CosPlace model weights are downloaded automatically on first run via `cosplace_utils.py`.
```
