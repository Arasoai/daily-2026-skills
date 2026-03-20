```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - identify location from street view photo
  - build a geolocation index
  - reverse image search physical world
  - netryx geolocation
  - find where a photo was taken
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It indexes street-view panoramas and matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). No internet image database — it searches the physical world.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode support (LoFTR dense matching)
pip install kornia
```

### Requirements
- Python 3.9+ (3.10+ recommended)
- GPU: NVIDIA CUDA (4GB+ VRAM) or Apple Silicon MPS — CPU works but is slow
- RAM: 8GB minimum, 16GB+ recommended
- Storage: 10GB+ (index size depends on coverage area)
- Internet: Required for indexing (Street View panorama downloads) and searching

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

### Step 1: Create an Index

Index an area before searching. Netryx crawls Street View panoramas and extracts CosPlace fingerprints into a searchable index.

**Via GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius (0.5–10 km)
4. Set grid resolution (default: 300 — don't change)
5. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is incremental — interruptions resume automatically on next run.

### Step 2: Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Let Gemini estimate region from visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main GUI application (indexing + search)
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panoid IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace descriptor (512-dim fingerprint)
    ├── Flipped descriptor (catches reversed perspectives)
    │
    ▼
Index Search — cosine similarity, radius-filtered (haversine)
    │
    └── Top 500 candidates
    │
    ▼
Download Panoramas → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    ├── LightGlue deep feature matching
    ├── RANSAC geometric verification (inlier filtering)
    │
    ▼
Heading Refinement — ±45° at 15° steps, top 15 candidates
    │
    ├── Spatial consensus clustering (50m cells)
    ├── Confidence scoring (clustering + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Key Models

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace | Global visual place recognition (retrieval) | Any |
| ALIKED | Local keypoint extraction | CUDA only |
| DISK | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | Any |
| LoFTR | Dense detector-free matching (Ultra Mode) | Any |

**Platform-specific extractor selection:**
- NVIDIA GPU → ALIKED (1024 keypoints)
- Apple MPS → DISK (768 keypoints)
- CPU → DISK (768 keypoints)

---

## Code Examples

### Extract a CosPlace Descriptor Programmatically

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model()
device = torch.device("cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu")
model = model.to(device)

# Extract descriptor from an image
img = Image.open("street_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)
# Returns: numpy array of shape (512,)
print(f"Descriptor shape: {descriptor.shape}")  # (512,)
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    """Return distance in meters between two GPS points."""
    R = 6371000
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
panoids = meta["panoids"]
headings = meta["headings"]

# Your query descriptor (from extract_descriptor above)
query_desc = descriptor  # shape (512,)

# Cosine similarity search
query_norm = query_desc / (np.linalg.norm(query_desc) + 1e-8)
db_norms = descriptors / (np.linalg.norm(descriptors, axis=1, keepdims=True) + 1e-8)
similarities = db_norms @ query_norm  # (N,)

# Radius filter (e.g., search within 2km of Paris center)
center_lat, center_lon = 48.8566, 2.3522
radius_m = 2000

mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

# Apply mask and get top 500 candidates
similarities[~mask] = -1
top_indices = np.argsort(similarities)[::-1][:500]

print(f"Top match: lat={lats[top_indices[0]]:.6f}, lon={lons[top_indices[0]]:.6f}")
print(f"Similarity: {similarities[top_indices[0]]:.4f}")
print(f"Panoid: {panoids[top_indices[0]]}, Heading: {headings[top_indices[0]]}°")
```

### Build Index from Scratch (Standalone)

```python
# Use build_index.py for large datasets (more performant than GUI)
# Run from command line:
# python build_index.py
#
# Or integrate programmatically — check build_index.py for the API,
# but the typical pattern is:

import subprocess
result = subprocess.run(
    ["python", "build_index.py",
     "--lat", "48.8566",
     "--lon", "2.3522",
     "--radius", "1000",   # meters
     "--resolution", "300"],
    capture_output=True, text=True
)
print(result.stdout)
```

### Ultra Mode Usage Pattern

```python
# Ultra Mode is toggled via the GUI checkbox.
# It activates three additional strategies:
# 1. LoFTR dense matching (handles blur/low-texture images)
# 2. Descriptor hopping (re-searches index using matched panorama's descriptor)
# 3. Neighborhood expansion (searches within 100m of best match)

# When to use Ultra Mode:
# - Night photography
# - Motion blur or low resolution
# - Low-texture scenes (empty roads, foggy conditions)
# - Partial occlusion (crowd, vehicles blocking scene)

# Ultra Mode is significantly slower — use Standard mode first,
# only escalate to Ultra if confidence score is low (<50 inliers).
```

### Multi-Area Index Strategy

```python
# Netryx uses a single unified index — all cities share one index.
# Searching is always radius-filtered, so no city selection needed.

# Index Paris:
# GUI: Create, center=(48.8566, 2.3522), radius=5km

# Index London:
# GUI: Create, center=(51.5074, -0.1278), radius=5km

# Both are stored in the same cosplace_parts/ and index/ directories.

# Search Paris only:
# GUI: Search, center=(48.8566, 2.3522), radius=5km
# → Only returns Paris results despite London being in the index

# Search London only:
# GUI: Search, center=(51.5074, -0.1278), radius=5km
# → Only returns London results
```

---

## Common Patterns

### Pattern: Geolocating OSINT/Conflict Imagery

```
1. Examine image for regional clues (script on signs, vegetation, architecture)
2. Use AI Coarse mode OR manually estimate region from visual context
3. Index the suspected region (1–5 km radius)
4. Search with Manual mode, center on suspected area
5. If confidence is low, enable Ultra Mode and re-run
6. Cross-validate result with satellite imagery (Google Maps, etc.)
```

### Pattern: Incremental City Coverage

```
1. Start with 0.5km radius around a known landmark to validate setup
2. Expand radius incrementally (1km → 5km → 10km)
3. Index runs in background — each run adds to existing index
4. Interrupted runs resume automatically (incremental save)
```

### Pattern: Improving Low-Confidence Results

```
1. Run standard search first
2. If top match has <50 inliers:
   a. Enable Ultra Mode
   b. Check if query image has strong blur — try sharpening preprocessed version
   c. Try a tighter radius (eliminate distant false-positive clusters)
   d. Expand search area if location assumption might be wrong
3. Spatial consensus: if multiple candidates cluster in one area, that cluster wins over a single high-inlier outlier
```

---

## Configuration Reference

### Search Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| Top candidates | 500–1000 | Returned from cosine similarity search |
| Heading refinement range | ±45° | Step size: 15°, tested on top 15 candidates |
| Spatial clustering cell | 50m | Candidates within 50m grouped into consensus clusters |
| Ultra Mode neighbor radius | 100m | Neighborhood expansion around best match |
| Multi-FOV crops | 70°, 90°, 110° | Handles zoom mismatches between query and index |

### Index Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| Grid resolution | 300 | Panoramas per grid unit — do not change |
| Descriptor size | 512-dim | CosPlace output, stored as float32 |
| Save format | `.npz` chunks | Incremental, merged into `cosplace_descriptors.npy` |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # Match your Python version
```

### CUDA out of memory
- Reduce number of candidates (`top_k`) via GUI settings
- Use DISK instead of ALIKED (fewer keypoints: 768 vs 1024)
- Disable Ultra Mode (LoFTR is VRAM-heavy)

### Indexing stalls / no panoramas found
- Verify center coordinates are in an area with Street View coverage
- Check internet connection (panorama tiles fetched live during indexing)
- Reduce radius — some regions have sparse Street View data

### Low confidence scores consistently
1. Enable Ultra Mode
2. Verify the indexed area actually contains the query location
3. Expand search radius
4. Ensure query image is street-level (aerial/indoor images won't match)

### Import errors for LightGlue
```bash
# LightGlue must be installed from GitHub, not PyPI
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
```

### Index not found on search
- Run **Create Index** first, or rebuild from parts:
```bash
python build_index.py  # Compiles cosplace_parts/*.npz → index/
```

### Resuming interrupted indexing
- Simply re-run **Create Index** with the same parameters
- Already-processed panoramas are skipped automatically

---

## Key File Reference

| File | Purpose |
|------|---------|
| `test_super.py` | Main entry point — GUI, indexing, full search pipeline |
| `cosplace_utils.py` | CosPlace model loading, descriptor extraction utilities |
| `build_index.py` | High-performance standalone index builder for large areas |
| `index/cosplace_descriptors.npy` | Compiled descriptor matrix (N × 512) |
| `index/metadata.npz` | Parallel arrays: lats, lons, headings, panoids |
| `cosplace_parts/*.npz` | Raw incremental chunks (pre-compilation) |
```
