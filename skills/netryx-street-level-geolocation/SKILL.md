```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source locally-hosted street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - identify GPS coordinates from image
  - street level geolocation
  - use netryx to find location
  - index street view panoramas
  - find where a photo was taken
  - osint geolocation from photo
  - run netryx geolocation pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable visual index using CosPlace descriptors, and matches query images using ALIKED/DISK keypoints + LightGlue deep feature matching. Sub-50m accuracy, no landmarks required, runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must be installed from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Optional: Gemini API for AI-assisted region detection

```bash
export GEMINI_API_KEY="your_key_from_aistudio_google_com"
```

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### Step 1 — Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Index is saved incrementally — safe to interrupt and resume.

**Index size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Disk |
|--------|------------|---------------|------|
| 0.5 km | ~500       | 30 min        | ~60 MB |
| 1 km   | ~2,000     | 1–2 hrs       | ~250 MB |
| 5 km   | ~30,000    | 8–12 hrs      | ~3 GB |
| 10 km  | ~100,000   | 24–48 hrs     | ~7 GB |

### Step 2 — Search

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**

---

## Project Structure

```
netryx/
├── test_super.py          # Main GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace → 512-dim descriptor
    ├── Flipped image → 512-dim descriptor (catches reversed perspectives)
    │
    ▼
Cosine similarity search over index (radius-filtered via haversine)
    │
    └── Top 500–1000 candidates
    │
    ▼
Download panoramas → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) or DISK (MPS/CPU) → keypoints + descriptors
    ├── LightGlue → deep feature matching
    ├── RANSAC → geometric verification (inlier count)
    │
    ▼
Heading refinement (±45°, 15° steps, top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    ├── Confidence scoring (uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Using CosPlace Utilities Directly

```python
# cosplace_utils.py exposes model loading and descriptor extraction
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model()

# Extract 512-dim descriptor from an image
img = Image.open("query_photo.jpg")
descriptor = get_descriptor(model, img)  # returns torch.Tensor shape [512]

# Also extract flipped version for robustness
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = get_descriptor(model, img_flipped)
```

---

## Building a Large Index Programmatically

```python
# Use build_index.py for large-scale indexing without GUI overhead
# Run directly as a script with your target area

# Example: index a 5km radius around central Paris
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5 \
    --resolution 300
```

Or call internally if build_index exposes functions:

```python
import build_index

build_index.run(
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=5,
    grid_resolution=300,
    output_dir="cosplace_parts/"
)
```

---

## Loading and Searching the Index Manually

```python
import numpy as np
from math import radians, sin, cos, sqrt, atan2

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")  # shape [N, 512]
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # shape [N]
lons = meta["lons"]       # shape [N]
headings = meta["headings"]
panoids = meta["panoids"]

# Haversine filter — restrict to radius around a center point
def haversine_mask(lats, lons, center_lat, center_lon, radius_km):
    R = 6371.0
    dlat = np.radians(lats - center_lat)
    dlon = np.radians(lons - center_lon)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
    return 2 * R * np.arcsin(np.sqrt(a)) <= radius_km

# Cosine similarity search
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image

model = load_cosplace_model()
query = Image.open("query.jpg")
q_desc = get_descriptor(model, query).numpy()                   # [512]
q_desc_flip = get_descriptor(model, query.transpose(Image.FLIP_LEFT_RIGHT)).numpy()

# Apply radius filter
mask = haversine_mask(lats, lons, center_lat=48.8566, center_lon=2.3522, radius_km=1.0)
filtered_desc = descriptors[mask]           # [M, 512]

# Cosine similarity (descriptors assumed L2-normalized)
scores_orig = filtered_desc @ q_desc        # [M]
scores_flip = filtered_desc @ q_desc_flip   # [M]
scores = np.maximum(scores_orig, scores_flip)

# Top 500 candidates
top_k = min(500, len(scores))
top_indices = np.argpartition(scores, -top_k)[-top_k:]
top_indices = top_indices[np.argsort(scores[top_indices])[::-1]]

# Map back to original index positions
original_indices = np.where(mask)[0][top_indices]
candidates = [
    {
        "lat": lats[i],
        "lon": lons[i],
        "heading": headings[i],
        "panoid": panoids[i],
        "score": scores[top_indices[j]]
    }
    for j, i in enumerate(original_indices)
]
```

---

## GPU / Device Configuration

Netryx auto-detects the best available device:

| Platform     | Feature Extractor | Notes |
|-------------|-------------------|-------|
| NVIDIA CUDA | ALIKED (1024 kp)  | Best accuracy |
| Apple MPS   | DISK (768 kp)     | M1/M2/M3/M4 |
| CPU         | DISK              | Slow but works |

```python
import torch

# Check what Netryx will use
if torch.cuda.is_available():
    device = "cuda"
    extractor_type = "aliked"
elif torch.backends.mps.is_available():
    device = "mps"
    extractor_type = "disk"
else:
    device = "cpu"
    extractor_type = "disk"

print(f"Device: {device}, Extractor: {extractor_type}")
```

---

## Ultra Mode (Difficult Images)

Enable for blurry, dark, low-contrast, or heavily compressed images. Activates:

1. **LoFTR** — detector-free dense matching (handles blur/low-contrast)
2. **Descriptor hopping** — re-searches index using matched panorama's clean descriptor
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

In the GUI: check the **Ultra Mode** checkbox before running search.

Requires kornia:
```bash
pip install kornia
```

---

## Multi-City Index Strategy

The index is unified — all cities share one index. Use coordinates + radius to scope searches:

```python
# Index Paris and London into the SAME index
# Paris indexing run: center=(48.8566, 2.3522), radius=5km
# London indexing run: center=(51.5074, -0.1278), radius=5km

# Searching Paris only:
mask_paris = haversine_mask(lats, lons, 48.8566, 2.3522, radius_km=5)

# Searching London only:
mask_london = haversine_mask(lats, lons, 51.5074, -0.1278, radius_km=5)

# No need to maintain separate index files per city
```

---

## Common Patterns

### Pattern: Batch geolocation of multiple images

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import numpy as np

model = load_cosplace_model()
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

image_paths = ["img1.jpg", "img2.jpg", "img3.jpg"]
results = []

for path in image_paths:
    img = Image.open(path)
    q = get_descriptor(model, img).numpy()
    q_f = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT)).numpy()
    scores = np.maximum(descriptors @ q, descriptors @ q_f)
    best_idx = scores.argmax()
    results.append({
        "image": path,
        "lat": float(meta["lats"][best_idx]),
        "lon": float(meta["lons"][best_idx]),
        "confidence": float(scores[best_idx])
    })

for r in results:
    print(f"{r['image']}: ({r['lat']:.6f}, {r['lon']:.6f}) conf={r['confidence']:.3f}")
```

### Pattern: Verify index coverage for an area

```python
import numpy as np
from your_script import haversine_mask  # or inline the function

meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]

center_lat, center_lon = 48.8566, 2.3522
radius_km = 1.0

mask = haversine_mask(lats, lons, center_lat, center_lon, radius_km)
print(f"Panoramas indexed within {radius_km}km of ({center_lat}, {center_lon}): {mask.sum()}")

# Rule of thumb: aim for >500 panoramas per km² for reliable matching
```

---

## Troubleshooting

### GUI appears blank (macOS)
```bash
brew install python-tk@3.11  # match your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory
- Reduce candidate count in the GUI (top-K setting)
- Use DISK instead of ALIKED (fewer keypoints: 768 vs 1024)
- Enable Ultra Mode only when necessary

### Low inlier counts / poor matches
- Increase search radius — query location may be at the edge of indexed area
- Enable Ultra Mode for degraded images
- Ensure the area is indexed at sufficient density (≥300 grid resolution)
- Check that the panorama coverage exists for that area (rural areas may have sparse Street View data)

### Index build interrupted
Safe to re-run — indexing saves incrementally to `cosplace_parts/*.npz` and resumes from where it left off.

### Slow performance on CPU
- MPS (Apple Silicon) is ~3–5× faster than CPU
- CUDA is fastest for large candidate sets
- Reduce top-K candidates (500→200) to speed up Stage 2 at some accuracy cost

### AI Coarse mode not working
```bash
# Ensure the environment variable is set in the same shell session
export GEMINI_API_KEY="your_key_here"
python test_super.py
```

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim descriptor (Stage 1) | CPU/GPU |
| [ALIKED](https://github.com/naver/alike) | Local keypoints (Stage 2, CUDA) | CUDA |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints (Stage 2, MPS/CPU) | MPS/CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching + RANSAC | CUDA/MPS/CPU |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching, Ultra Mode only | CUDA/MPS/CPU |
```
