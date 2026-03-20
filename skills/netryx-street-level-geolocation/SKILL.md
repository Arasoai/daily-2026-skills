```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - identify location from street view photo
  - netryx geolocation
  - reverse geolocate image
  - find where a photo was taken
  - osint geolocation from photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, extracts visual fingerprints using CosPlace, then verifies matches with local feature matching (ALIKED/DISK + LightGlue). Sub-50m accuracy, runs entirely on local hardware, no internet presence required for the target location.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

**Requirements:**
- Python 3.9+ (3.10+ recommended)
- GPU: NVIDIA (CUDA, 4GB+ VRAM) or Apple Silicon (MPS) — CPU works but is slow
- RAM: 8GB minimum, 16GB+ recommended
- Storage: 10GB+ (scales with index size)

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix:** `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### 1. Create an Index (Required Before Searching)

Index a geographic area by crawling street-view panoramas and computing CosPlace descriptors.

**In GUI:**
1. Select **Create** mode
2. Enter center lat/lon of target area
3. Set radius (km) and grid resolution (default 300)
4. Click **Create Index**

**Index size estimates:**

| Radius | Panoramas | Time (M2 Max) | Storage |
|--------|-----------|---------------|---------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Index saves incrementally — safe to interrupt and resume.

### 2. Search for a Location

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Uses Gemini to estimate region from visual cues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**

**Optional: Enable Ultra Mode** for difficult images (night, blur, low texture).

---

## Environment Variables

```bash
# Optional — only needed for AI Coarse search mode
export GEMINI_API_KEY="your_key_here"
```

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Using CosPlace Utilities Directly

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load the model (auto-detects CUDA / MPS / CPU)
device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"
model = load_cosplace_model(device=device)

# Extract a 512-dim descriptor from an image
img = Image.open("query_photo.jpg")
descriptor = get_descriptor(model, img, device=device)
# descriptor.shape == (512,)
```

---

## Building an Index Programmatically

```python
# Use the standalone high-performance index builder
# (better for large areas than the GUI)
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 1.0 \
    --resolution 300
```

Or trigger index build from within the app logic:

```python
# The index is auto-compiled from cosplace_parts/*.npz chunks
# After indexing completes, the searchable index is written to:
#   index/cosplace_descriptors.npy  — shape (N, 512)
#   index/metadata.npz             — lat, lon, heading, panoid arrays
```

---

## Loading and Searching the Index Manually

```python
import numpy as np
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz")
lats = meta["lats"]
lons = meta["lons"]
headings = meta["headings"]
panoids = meta["panoids"]

# Load model
device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"
model = load_cosplace_model(device=device)

# Extract query descriptor
query_img = Image.open("query.jpg")
query_desc = get_descriptor(model, query_img, device=device)  # (512,)

# Also extract flipped version (catches reversed perspectives)
query_img_flipped = query_img.transpose(Image.FLIP_LEFT_RIGHT)
query_desc_flipped = get_descriptor(model, query_img_flipped, device=device)

# Cosine similarity search
desc_tensor = torch.tensor(descriptors, dtype=torch.float32)
q = torch.tensor(query_desc, dtype=torch.float32)
q = q / q.norm()
desc_tensor = desc_tensor / desc_tensor.norm(dim=1, keepdim=True)

similarities = desc_tensor @ q  # (N,)
top_k = torch.topk(similarities, k=500)

top_indices = top_k.indices.numpy()
top_scores = top_k.values.numpy()

# Radius filter using haversine distance
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000  # Earth radius in meters
    lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

# Filter to search_center ± radius_m
search_lat, search_lon = 48.8566, 2.3522
radius_m = 2000  # 2km radius

filtered = [
    i for i in top_indices
    if haversine(search_lat, search_lon, lats[i], lons[i]) <= radius_m
]

print(f"Top candidate: lat={lats[filtered[0]]:.6f}, lon={lons[filtered[0]]:.6f}")
print(f"Heading: {headings[filtered[0]]}°, PanoID: {panoids[filtered[0]]}")
```

---

## The Three-Stage Pipeline

### Stage 1 — Global Retrieval (CosPlace)
- Extracts 512-dim visual fingerprint from query + flipped query
- Cosine similarity search against full index
- Radius filter (haversine) narrows to search area
- Returns top 500–1000 candidates
- **Speed: <1 second** (single matrix multiply)

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)
- Downloads panorama tiles from Street View (8 tiles, stitched)
- Crops at indexed heading, 3 FOVs: 70°, 90°, 110°
- ALIKED (CUDA) or DISK (MPS/CPU) extracts keypoints
- LightGlue deep feature matching against query keypoints
- RANSAC filters geometrically inconsistent matches
- **Speed: 2–5 minutes for 300–500 candidates**

### Stage 3 — Refinement
- Tests ±45° heading offsets at 15° steps for top 15 candidates
- Spatial consensus: clusters matches in 50m cells
- Confidence scoring: clustering density + uniqueness ratio

### Ultra Mode (Optional)
- **LoFTR**: Dense detector-free matching for blur/low-texture images
- **Descriptor hopping**: Re-searches index using matched panorama's clean descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Multi-Region Index Strategy

All regions share a single unified index. Radius filtering handles separation:

```python
# Index Paris
# → stored in cosplace_parts/ and index/

# Index London  
# → appended to same index

# Search Paris only:
search_lat, search_lon = 48.8566, 2.3522
radius_m = 5000

# Search London only:
search_lat, search_lon = 51.5074, -0.1278
radius_m = 10000

# No city selection needed — coordinates + radius handle everything
```

---

## Platform-Specific Notes

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|--------------|---------------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| Speed | Fastest | Fast | Slow |
| Ultra Mode (LoFTR) | Full support | Partial | Slow |

```python
# Device detection pattern used throughout Netryx
import torch

if torch.cuda.is_available():
    device = "cuda"
elif torch.backends.mps.is_available():
    device = "mps"
else:
    device = "cpu"
```

---

## Common Patterns

### Batch Index Multiple Cities

```python
import subprocess

cities = [
    ("48.8566", "2.3522", "1.0"),   # Paris
    ("51.5074", "-0.1278", "1.0"),  # London
    ("40.7128", "-74.0060", "1.0"), # New York
]

for lat, lon, radius in cities:
    subprocess.run([
        "python", "build_index.py",
        "--lat", lat,
        "--lon", lon,
        "--radius", radius,
        "--resolution", "300"
    ])
```

### Check Index Size

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz")

print(f"Total indexed panoramas: {len(descriptors):,}")
print(f"Descriptor array shape: {descriptors.shape}")
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
```

### Inspect Top Candidates with Google Maps Links

```python
for rank, idx in enumerate(filtered[:10]):
    lat, lon = lats[idx], lons[idx]
    score = similarities[idx].item()
    maps_url = f"https://www.google.com/maps?q={lat},{lon}"
    sv_url = f"https://www.google.com/maps/@{lat},{lon},3a,75y,{headings[idx]}h,90t/data=!3m1!1e1"
    print(f"#{rank+1} Score={score:.4f} | {lat:.6f}, {lon:.6f} | {sv_url}")
```

---

## Troubleshooting

**GUI appears blank on macOS**
```bash
brew install python-tk@3.11  # Match your Python version
```

**`ImportError: No module named 'lightglue'`**
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

**CUDA out of memory**
- Reduce candidates from 500 to 200 in the GUI settings
- Use DISK instead of ALIKED (fewer keypoints: 768 vs 1024)
- Enable `torch.cuda.empty_cache()` between candidates

**Index not found / empty results**
- Ensure `index/cosplace_descriptors.npy` and `index/metadata.npz` exist
- If only `cosplace_parts/*.npz` exist, trigger index auto-build by running a search — it compiles automatically
- Or run `build_index.py` explicitly

**Indexing stalls / interrupted**
- Safe to re-run — indexing resumes from last checkpoint
- Check `cosplace_parts/` for existing `.npz` chunks

**Poor match accuracy**
- Increase search radius
- Enable Ultra Mode for degraded images
- Try AI Coarse mode if unsure of region (requires `GEMINI_API_KEY`)
- Ensure the area is indexed at sufficient density (radius ≤ 5km, resolution = 300)

**MPS errors on Apple Silicon**
```bash
# Fallback to CPU if MPS causes issues
export PYTORCH_ENABLE_MPS_FALLBACK=1
python test_super.py
```

**Slow indexing**
- Use `build_index.py` instead of the GUI for large areas (more efficient)
- Reduce radius and increase resolution gradually
- Ensure you're using GPU (check device detection output at startup)
```
