```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - match photo to location
  - visual place recognition pipeline
  - run netryx geolocation search
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images against them using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature extraction (ALIKED/DISK), and deep feature matching (LightGlue + RANSAC). No landmarks required. No internet search. Runs entirely on local hardware.

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

# 2. Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# 3. Install core dependencies
pip install -r requirements.txt

# 4. Install LightGlue (required — must be installed from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# 5. (Optional) Install kornia for Ultra Mode (LoFTR dense matching)
pip install kornia
```

### macOS tkinter fix (blank GUI issue)
```bash
brew install python-tk@3.11   # match your Python version
```

### Gemini API key (optional — AI Coarse region estimation)
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

**GPU backends:** CUDA (NVIDIA) → ALIKED extractor | MPS (Apple Silicon) → DISK extractor | CPU → DISK (slow)

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (index an area) and **Search** (geolocate a photo).

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The process crawls street-view panoramas, extracts CosPlace fingerprints, and saves them incrementally (safe to interrupt and resume).

**Via GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius (km) and grid resolution (default 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius  | ~Panoramas | Time (M2 Max) | Index Size |
|---------|-----------|---------------|------------|
| 0.5 km  | ~500      | 30 min        | ~60 MB     |
| 1 km    | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km    | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km   | ~100,000  | 24–48 hours   | ~7 GB      |

**Programmatic index building (large datasets):**
```bash
python build_index.py
```

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coords + radius
   - **AI Coarse**: Let Gemini estimate region from visual clues
4. (Optional) Enable **Ultra Mode** for difficult images
5. Click **Run Search** → **Start Full Search**

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loader and descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Lat/lon, headings, panorama IDs
```

---

## How the Pipeline Works

```
Query Image
    │
    ├─ CosPlace → 512-dim descriptor
    ├─ Flipped image → 512-dim descriptor (catches reversed perspectives)
    │
    ▼
Index search: cosine similarity, radius-filtered (haversine)
    │
    └─ Top 500 candidates
         │
         ▼
    Download panorama tiles → stitch → rectilinear crop
    Multi-FOV crops: 70°, 90°, 110°
         │
         ├─ ALIKED (CUDA) or DISK (MPS/CPU) → keypoints + descriptors
         ├─ LightGlue → feature matches
         └─ RANSAC → geometric inliers
              │
              ▼
    Heading refinement: ±45° at 15° steps, top 15 candidates
    Spatial consensus: cluster matches into 50m cells
    Confidence scoring: inlier count + uniqueness ratio
              │
              ▼
    📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable for night shots, motion blur, low-texture scenes, or when the standard pipeline returns low confidence.

Adds three enhancements:
1. **LoFTR** — detector-free dense matching (handles blur/low contrast)
2. **Descriptor hopping** — re-searches the index using the matched panorama's clean descriptor if inliers < 50
3. **Neighborhood expansion** — searches all panoramas within 100m of the best match

```python
# Ultra Mode is toggled via the GUI checkbox.
# Internally it activates LoFTR and re-ranking passes.
# Requires: pip install kornia
```

---

## Index Architecture

All city indexes are unified. Radius filtering at search time isolates the correct area — no per-city index files needed.

```python
# Index stored as:
#   index/cosplace_descriptors.npy  — shape: (N, 512), float32
#   index/metadata.npz              — arrays: lat, lon, heading, panoid

import numpy as np

# Load and inspect the index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz")
lats = meta["lat"]       # (N,)
lons = meta["lon"]       # (N,)
headings = meta["heading"]
panoids = meta["panoid"]

print(f"Index contains {len(lats):,} panoramas")
```

---

## CosPlace Descriptor Extraction

```python
# cosplace_utils.py exposes the model loader and extractor
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

model = get_cosplace_model(device)

img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)  # shape: (512,), numpy float32

# Also extract flipped descriptor for reversed-perspective robustness
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = get_descriptor(model, img_flipped, device)
```

---

## Manual Cosine Similarity Search

```python
import numpy as np

def cosine_search(query_desc, descriptors, top_k=500):
    """Return indices of top-k most similar descriptors."""
    # Normalize
    q = query_desc / (np.linalg.norm(query_desc) + 1e-8)
    d = descriptors / (np.linalg.norm(descriptors, axis=1, keepdims=True) + 1e-8)
    scores = d @ q                          # (N,)
    top_indices = np.argsort(scores)[::-1][:top_k]
    return top_indices, scores[top_indices]

descriptors = np.load("index/cosplace_descriptors.npy")
top_idx, top_scores = cosine_search(descriptor, descriptors)
```

---

## Haversine Radius Filter

```python
import numpy as np

def haversine_filter(lats, lons, center_lat, center_lon, radius_km):
    """Return boolean mask of panoramas within radius_km of center."""
    R = 6371.0
    dlat = np.radians(lats - center_lat)
    dlon = np.radians(lons - center_lon)
    a = (np.sin(dlat / 2) ** 2 +
         np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) *
         np.sin(dlon / 2) ** 2)
    dist_km = 2 * R * np.arcsin(np.sqrt(a))
    return dist_km <= radius_km

meta = np.load("index/metadata.npz")
mask = haversine_filter(meta["lat"], meta["lon"],
                        center_lat=48.8566, center_lon=2.3522,
                        radius_km=2.0)
print(f"Candidates in radius: {mask.sum():,}")
```

---

## Combined Retrieval Pattern

```python
import numpy as np
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz")

# Load model and extract query descriptor
model = get_cosplace_model(device)
img = Image.open("street_photo.jpg").convert("RGB")
q_desc = get_descriptor(model, img, device)
q_desc_flip = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT), device)

# Radius filter
CENTER_LAT, CENTER_LON, RADIUS_KM = 48.8566, 2.3522, 3.0
R = 6371.0
dlat = np.radians(meta["lat"] - CENTER_LAT)
dlon = np.radians(meta["lon"] - CENTER_LON)
a = np.sin(dlat/2)**2 + np.cos(np.radians(CENTER_LAT))*np.cos(np.radians(meta["lat"]))*np.sin(dlon/2)**2
dist_km = 2 * R * np.arcsin(np.sqrt(a))
mask = dist_km <= RADIUS_KM

local_descs = descriptors[mask]             # (M, 512)
local_lats  = meta["lat"][mask]
local_lons  = meta["lon"][mask]
local_panoids = meta["panoid"][mask]

# Score = max of normal and flipped similarity
norm = lambda v: v / (np.linalg.norm(v) + 1e-8)
scores = local_descs @ norm(q_desc)
scores_flip = local_descs @ norm(q_desc_flip)
combined = np.maximum(scores, scores_flip)

top500_idx = np.argsort(combined)[::-1][:500]
print("Top match:", local_panoids[top500_idx[0]],
      "at", local_lats[top500_idx[0]], local_lons[top500_idx[0]])
```

---

## Configuration Tips

| Setting | Recommendation |
|---------|---------------|
| Grid resolution | 300 (default) — don't change unless you know why |
| Search radius | Start small (0.5–1 km) for testing |
| Top candidates | 500 standard; reduce to 200 for speed on CPU |
| Ultra Mode | Enable only for difficult images; 2–3× slower |
| FOV crops | 70°/90°/110° (built-in) — covers zoom mismatches |
| Heading refinement | ±45° at 15° steps, top 15 candidates (built-in) |

---

## Troubleshooting

### GUI appears blank (macOS)
```bash
brew install python-tk@3.11   # match your Python version exactly
```

### LightGlue import error
```bash
# Must be installed from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
```

### CUDA out of memory
- Reduce batch size of candidates processed simultaneously in `test_super.py`
- Switch to DISK instead of ALIKED (lower keypoint count: 768 vs 1024)
- Use CPU fallback (slower but always works)

### Low confidence results / wrong location
1. Enable **Ultra Mode** — adds LoFTR and descriptor hopping
2. Verify the area is indexed: check `index/metadata.npz` lat/lon bounds
3. Increase search radius — the correct location may be near the edge
4. Try a tighter radius centered closer to the true location if you have a hint
5. Check that the query image is street-level and not aerial/indoor

### Index build interrupted
Safe to restart — `build_index.py` and `test_super.py` resume from the last saved `cosplace_parts/*.npz` chunk.

### MPS (Apple Silicon) slower than expected
DISK on MPS uses 768 keypoints vs ALIKED's 1024 on CUDA — this is expected. The matching quality is comparable; the speed difference is hardware-bound.

### AI Coarse mode not working
```bash
export GEMINI_API_KEY="your_key_here"
# Verify it's set:
echo $GEMINI_API_KEY
```

---

## Models Reference

| Model | Role | Backend |
|-------|------|---------|
| CosPlace (512-dim) | Global retrieval — image fingerprint | CUDA / MPS / CPU |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | CUDA / MPS / CPU |
| RANSAC | Geometric verification (inlier filtering) | CPU |
| LoFTR | Dense matching for difficult images (Ultra) | CUDA / MPS |

---

## Key Files Quick Reference

| File | Purpose |
|------|---------|
| `test_super.py` | **Main entry point** — launch this |
| `cosplace_utils.py` | Import `get_cosplace_model`, `get_descriptor` |
| `build_index.py` | Run standalone for large-area indexing |
| `index/cosplace_descriptors.npy` | Descriptor matrix — shape `(N, 512)` |
| `index/metadata.npz` | Keys: `lat`, `lon`, `heading`, `panoid` |
| `cosplace_parts/` | Intermediate chunks — auto-managed |
```
