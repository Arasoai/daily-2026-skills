```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - "geolocate a street photo"
  - "find coordinates from street image"
  - "use netryx to locate"
  - "street level geolocation"
  - "index street view panoramas"
  - "run netryx search"
  - "build a geolocation index"
  - "match street photo to GPS coordinates"
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that finds precise GPS coordinates (sub-50m accuracy) for any street-level photograph. It works by indexing street-view panoramas into a searchable embedding database, then matching query images against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). No landmark recognition required — it matches raw visual structure against crawled panoramas.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must be installed from source)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode dense matching
pip install kornia
```

### Optional: Gemini API for AI Coarse region estimation

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

GPU backend is auto-detected:
- **NVIDIA**: CUDA — uses ALIKED (1024 keypoints)
- **Apple Silicon**: MPS — uses DISK (768 keypoints)
- **CPU**: DISK — significantly slower

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11` (match your Python version)

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone CLI index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (compiled index)
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area by crawling Street View panoramas and extracting CosPlace fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center latitude/longitude
3. Set radius (start with 0.5–1 km for testing)
4. Set grid resolution (default: 300 — do not change)
5. Click **Create Index**

**Indexing time estimates:**

| Radius  | ~Panoramas | Time (M2 Max) | Index Size |
|---------|------------|---------------|------------|
| 0.5 km  | ~500       | 30 min        | ~60 MB     |
| 1 km    | ~2,000     | 1–2 hours     | ~250 MB    |
| 5 km    | ~30,000    | 8–12 hours    | ~3 GB      |
| 10 km   | ~100,000   | 24–48 hours   | ~7 GB      |

Indexing is **incremental** — safe to interrupt and resume. Parts are written to `cosplace_parts/` and compiled into `index/` automatically.

**Multiple cities in one index:** Index Paris, then London, then Tokyo — all share the same index. Radius filtering at search time handles separation automatically.

---

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Enter known approximate coordinates + radius
   - **AI Coarse**: Gemini analyzes visual cues (signs, architecture, vegetation) to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Pipeline Deep-Dive

### Stage 1: Global Retrieval (CosPlace)

- Extracts a 512-dim descriptor from the query image
- Also extracts descriptor from horizontally flipped version (catches reversed perspectives)
- Cosine similarity against full index (single matrix multiply — <1 second regardless of index size)
- Haversine radius filter applied
- Returns top 500–1000 candidate panoramas

### Stage 2: Local Geometric Verification (ALIKED/DISK + LightGlue)

For each candidate:
1. Download panorama from Street View (8 tiles, stitched)
2. Extract rectilinear crop at indexed heading
3. Generate multi-FOV crops at 70°, 90°, 110° (handles zoom mismatch)
4. Extract keypoints with ALIKED (CUDA) or DISK (MPS/CPU)
5. Match keypoints with LightGlue
6. Filter with RANSAC (removes geometrically inconsistent matches)
7. Inlier count = match quality score

Runtime: 300–500 candidates in 2–5 minutes depending on hardware.

### Stage 3: Refinement

- **Heading refinement**: Top 15 candidates re-tested at ±45° offsets (15° steps, 3 FOVs)
- **Spatial consensus**: Matches clustered into 50m cells — clusters preferred over lone outliers
- **Confidence scoring**: Geographic clustering density + uniqueness ratio (best vs. runner-up at different location)

### Ultra Mode

Enable for difficult images (night, blur, low texture):

- **LoFTR**: Detector-free dense matcher — works without distinct keypoints
- **Descriptor hopping**: If best match has <50 inliers, re-searches index using descriptor extracted from the matched panorama (clean image) instead of the degraded query
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Code Examples

### Extract a CosPlace Descriptor Manually

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")
model = load_cosplace_model(device=device)

# Extract descriptor from image
img = Image.open("street_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device=device)
# descriptor.shape == (512,)
print(f"Descriptor shape: {descriptor.shape}")
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, sin, cos, sqrt, atan2

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
headings = meta["headings"]
panoids = meta["panoids"]

# Query descriptor (from cosplace_utils)
query_desc = extract_descriptor(model, img, device=device)  # shape: (512,)

# Radius filter (e.g., center=Paris 1km)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 1.0

mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
])
filtered_descriptors = descriptors[mask]
filtered_indices = np.where(mask)[0]

# Cosine similarity search
query_norm = query_desc / (np.linalg.norm(query_desc) + 1e-8)
db_norms = filtered_descriptors / (np.linalg.norm(filtered_descriptors, axis=1, keepdims=True) + 1e-8)
similarities = db_norms @ query_norm  # shape: (M,)

# Top 500 candidates
top_k = min(500, len(similarities))
top_local_indices = np.argsort(similarities)[::-1][:top_k]
top_global_indices = filtered_indices[top_local_indices]

for rank, idx in enumerate(top_global_indices[:5]):
    print(f"Rank {rank+1}: lat={lats[idx]:.6f}, lon={lons[idx]:.6f}, "
          f"heading={headings[idx]}, sim={similarities[top_local_indices[rank]]:.4f}")
```

### Build Index from CLI (Large Datasets)

```bash
# build_index.py is the high-performance standalone builder
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --resolution 300
```

### Check Index Stats

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

print(f"Total indexed panoramas: {len(descriptors)}")
print(f"Descriptor shape: {descriptors.shape}")
print(f"Lat range: {meta['lats'].min():.4f} — {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} — {meta['lons'].max():.4f}")
```

### Inspect Raw Embedding Chunks

```python
import numpy as np
import glob

parts = sorted(glob.glob("cosplace_parts/*.npz"))
total = 0
for part in parts:
    data = np.load(part, allow_pickle=True)
    n = len(data["descriptors"])
    total += n
    print(f"{part}: {n} panoramas")
print(f"Total across all parts: {total}")
```

---

## Configuration Reference

All configuration is done through the GUI or by passing arguments to `build_index.py`. Key parameters:

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Controls panorama sampling density — do not change |
| Top candidates (Stage 1) | 500–1000 | Higher = more accurate, slower Stage 2 |
| Heading refinement steps | ±45° at 15° | Applied to top 15 candidates |
| Spatial clustering cell | 50m | For consensus scoring |
| Ultra Mode LoFTR threshold | 50 inliers | Below this, descriptor hopping activates |
| Neighborhood expansion | 100m | Panoramas within this radius of best match |

---

## Common Patterns

### Pattern: Index Multiple Cities, Search One

```python
# Index each city separately (they share the same index files)
# City 1: Paris
python build_index.py --lat 48.8566 --lon 2.3522 --radius 2.0

# City 2: London (appends to same index)
python build_index.py --lat 51.5074 --lon -0.1278 --radius 2.0

# Search only Paris results by specifying center + radius
center_lat, center_lon = 48.8566, 2.3522  # Paris
radius_km = 2.5  # Tight enough to exclude London
```

### Pattern: Batch Geolocation

```python
import os
from PIL import Image
from cosplace_utils import load_cosplace_model, extract_descriptor
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")
model = load_cosplace_model(device=device)

image_dir = "images_to_geolocate/"
results = {}

for fname in os.listdir(image_dir):
    if not fname.lower().endswith((".jpg", ".jpeg", ".png")):
        continue
    img = Image.open(os.path.join(image_dir, fname)).convert("RGB")
    desc = extract_descriptor(model, img, device=device)
    results[fname] = desc
    print(f"Extracted descriptor for {fname}")

# Then pass each descriptor through the index search logic above
```

### Pattern: Validate Index Coverage for a Region

```python
import numpy as np
from math import radians, sin, cos, sqrt, atan2

def count_in_radius(lats, lons, center_lat, center_lon, radius_km):
    count = 0
    for lat, lon in zip(lats, lons):
        dlat = radians(lat - center_lat)
        dlon = radians(lon - center_lon)
        a = sin(dlat/2)**2 + cos(radians(center_lat)) * cos(radians(lat)) * sin(dlon/2)**2
        dist = 6371.0 * 2 * atan2(sqrt(a), sqrt(1 - a))
        if dist <= radius_km:
            count += 1
    return count

meta = np.load("index/metadata.npz", allow_pickle=True)
n = count_in_radius(meta["lats"], meta["lons"], 48.8566, 2.3522, 1.0)
print(f"Panoramas indexed within 1km of Paris center: {n}")
# Rule of thumb: aim for >500 per km² for reliable matching
```

---

## Troubleshooting

### GUI is blank on macOS
```bash
brew install python-tk@3.11   # Replace 3.11 with your Python version
```

### LightGlue import error
```bash
# Must be installed from source, not PyPI
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
# kornia ships LoFTR — Ultra Mode checkbox will be greyed out without it
```

### CUDA out of memory during Stage 2
- Reduce batch size in `test_super.py` (look for candidate batch processing loops)
- Switch from ALIKED (1024 kp) to DISK (768 kp) by forcing MPS/CPU device
- Reduce top candidates from 500 to 300

### Index search returns no results
- Verify the index covers your search region: use the coverage validation snippet above
- Increase search radius
- Check that `index/cosplace_descriptors.npy` and `index/metadata.npz` both exist (auto-built from `cosplace_parts/`)

### Indexing stalled or crashed
Safe to restart — indexing resumes from existing `cosplace_parts/*.npz` files automatically.

### Low confidence / wrong match
1. Enable **Ultra Mode** — adds LoFTR and descriptor hopping
2. Increase indexed panorama density (re-index with same area, parts accumulate)
3. Use **AI Coarse** mode if the region is completely unknown (requires `GEMINI_API_KEY`)
4. Verify query image is genuinely street-level and not heavily cropped/filtered

### Slow Stage 2 (>10 min for 300 candidates)
- CUDA >> MPS >> CPU for speed
- On CPU, reduce candidates to 100–200 in GUI settings
- Avoid Ultra Mode on CPU — LoFTR is very slow without GPU

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (CVPR 2022) | Global 512-dim place descriptor | Any |
| ALIKED (IEEE TIP 2023) | Local keypoints + descriptors | CUDA only |
| DISK (NeurIPS 2020) | Local keypoints + descriptors | MPS / CPU |
| LightGlue (ICCV 2023) | Deep feature matching | Any |
| LoFTR (CVPR 2021) | Detector-free dense matching (Ultra) | Any (GPU recommended) |

---

## Key Files to Edit

| File | When to edit |
|------|-------------|
| `test_super.py` | Adjust candidate counts, FOV crops, RANSAC thresholds, clustering cell size |
| `cosplace_utils.py` | Swap CosPlace backbone, change descriptor dimension |
| `build_index.py` | Customize crawling strategy, grid resolution, API rate limits |
```
