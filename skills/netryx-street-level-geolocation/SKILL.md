```markdown
---
name: netryx-street-level-geolocation
description: Expertise in using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - run netryx search
  - osint geolocation from photo
  - identify location from street image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, then matches a query image against those panoramas using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric refinement. Sub-50m accuracy, runs entirely on local hardware — no cloud API required for searching.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (install from source)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode
pip install kornia
```

### Requirements Summary

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

**GPU backend auto-selection:**
- NVIDIA → CUDA (uses ALIKED, 1024 keypoints)
- Apple Silicon → MPS (uses DISK, 768 keypoints)
- Fallback → CPU (slow but functional)

### Optional: Gemini API Key (AI Coarse location mode)

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

### Step 1 — Create an Index

Index a geographic area before searching. The indexer crawls street-view panoramas, extracts CosPlace 512-dim descriptors, and saves them to disk.

**GUI:** Select **Create** mode → enter lat/lng center → set radius → set grid resolution (default: 300) → click **Create Index**.

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Index saves incrementally — safe to interrupt and resume.

**Index file layout:**

```
cosplace_parts/          # Raw per-chunk embeddings (.npz)
index/
  cosplace_descriptors.npy   # All 512-dim CosPlace vectors
  metadata.npz               # lat, lng, heading, panoid per entry
```

Multiple regions (Paris + London + Tel Aviv) can coexist in one index — the radius filter at search time handles isolation.

### Step 2 — Search

**GUI:** Select **Search** mode → upload photo → choose method:
- **Manual**: enter approximate center lat/lng + radius
- **AI Coarse**: Gemini analyzes visual cues to guess region (requires `GEMINI_API_KEY`)

Click **Run Search** → **Start Full Search** → watch real-time candidate evaluation → result appears on map with confidence score.

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace 512-dim descriptor (normal + horizontally flipped)
    │
    ▼
Index cosine similarity search (radius-filtered via haversine)
    │
    ├── Top 500–1000 candidates
    │
    ▼
Download panoramas → Rectilinear crops at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) / DISK (MPS/CPU) keypoint extraction
    ├── LightGlue deep feature matching
    ├── RANSAC geometric verification (inlier count)
    │
    ▼
Heading refinement: ±45° at 15° steps, 3 FOVs, top 15 candidates
    │
    ├── Spatial consensus clustering (50m cells)
    ├── Confidence scoring (uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable **Ultra Mode** checkbox in GUI for difficult images (night, blur, low texture). Adds:

1. **LoFTR** — detector-free dense matching, handles blur/low-contrast
2. **Descriptor hopping** — re-searches index using the matched panorama's clean descriptor when initial match has <50 inliers
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

Significantly slower; use only when standard pipeline gives low confidence.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading, descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created)
└── index/                 # Compiled searchable index (auto-created)
```

---

## Code Examples

### Extract a CosPlace Descriptor Programmatically

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-selects CUDA > MPS > CPU)
device = (
    torch.device("cuda") if torch.cuda.is_available()
    else torch.device("mps") if torch.backends.mps.is_available()
    else torch.device("cpu")
)
model = load_cosplace_model(device=device)

# Extract descriptor from a query image
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device=device)
# descriptor.shape == (512,)
print(f"Descriptor shape: {descriptor.shape}")
```

### Load the Index and Run a Cosine Similarity Search

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lng1, lat2, lng2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlng = radians(lng2 - lng1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlng/2)**2
    return 2 * R * asin(sqrt(a))

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512) float32
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]   # (N,)
lngs = meta["lngs"]   # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

# Normalize for cosine similarity
descriptors_norm = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)

def search_index(query_descriptor, center_lat, center_lng, radius_km, top_k=500):
    """Return top-k candidates within radius, sorted by cosine similarity."""
    # Radius filter
    distances = np.array([
        haversine_km(center_lat, center_lng, lat, lng)
        for lat, lng in zip(lats, lngs)
    ])
    mask = distances <= radius_km

    # Cosine similarity (query already normalized)
    q = query_descriptor / np.linalg.norm(query_descriptor)
    scores = descriptors_norm[mask] @ q

    # Get top-k within mask
    masked_indices = np.where(mask)[0]
    top_local = np.argsort(-scores)[:top_k]
    top_global = masked_indices[top_local]

    return [
        {
            "panoid": panoids[i],
            "lat": lats[i],
            "lng": lngs[i],
            "heading": headings[i],
            "score": float(scores[top_local[rank]])
        }
        for rank, i in enumerate(top_global)
    ]

# Example search: Paris 8th arrondissement, 1km radius
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = load_cosplace_model(device=device)

query_img = Image.open("mystery_street.jpg").convert("RGB")
q_desc = extract_descriptor(model, query_img, device=device)

candidates = search_index(q_desc, center_lat=48.8744, center_lng=2.2981, radius_km=1.0)
print(f"Top candidate: {candidates[0]}")
```

### Run a Full Search via the Main Application

The full pipeline (download panoramas, crop, ALIKED/LightGlue/RANSAC, refinement) lives in `test_super.py`. The cleanest way to invoke it programmatically is through the GUI, but you can trace the search function call pattern:

```python
# Pseudocode reflecting test_super.py internal structure
# (actual function names depend on your checkout version)

# 1. Load index
# 2. Extract CosPlace descriptor from query
# 3. Run index search → candidates list
# 4. For each candidate:
#    a. Download GSV panorama tiles, stitch
#    b. Crop rectilinear view at indexed heading, 3 FOVs
#    c. Extract ALIKED/DISK keypoints on query + candidate
#    d. LightGlue match → inliers via RANSAC
# 5. Heading refinement on top 15
# 6. Spatial consensus → final GPS output
```

### Build Index from Standalone Script (Large Areas)

```bash
# build_index.py is optimized for large-scale indexing
python build_index.py \
  --lat 48.8566 \
  --lng 2.3522 \
  --radius 5.0 \
  --grid-resolution 300
```

---

## Configuration Reference

All primary configuration is done through the GUI. Key parameters:

| Parameter | Description | Default | Notes |
|-----------|-------------|---------|-------|
| Center lat/lng | Geographic center of index/search area | — | Decimal degrees |
| Radius (km) | Search/index radius | — | Start with 0.5–1km for testing |
| Grid resolution | Density of panorama sampling grid | 300 | Don't modify without reason |
| Top candidates | Number of CosPlace matches to verify | 500 | Higher = slower, more accurate |
| Ultra Mode | Enable LoFTR + descriptor hopping | Off | For difficult images |
| AI Coarse | Use Gemini to estimate region | Off | Requires `GEMINI_API_KEY` |

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim place descriptor | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoints (1024) | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints (768) | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra Mode) | All (needs `kornia`) |

---

## Common Patterns

### Pattern: Index Multiple Cities, Search Any

```python
# Index Paris (no city label needed — coordinates handle it)
# GUI: Create, center=(48.8566, 2.3522), radius=5km

# Index London
# GUI: Create, center=(51.5074, -0.1278), radius=5km

# Search in Paris only
# GUI: Search, center=(48.8566, 2.3522), radius=6km
# → Only Paris panoramas returned via haversine filter
```

### Pattern: Resume Interrupted Indexing

The indexer writes `cosplace_parts/*.npz` incrementally. Simply re-run Create with the same parameters — already-processed chunks are skipped.

### Pattern: Low-Confidence Result → Try Ultra Mode

1. Run standard search → confidence < 0.5 or suspicious location
2. Enable **Ultra Mode** checkbox
3. Re-run search with same parameters
4. LoFTR + neighborhood expansion often recovers correct match

### Pattern: No Prior Knowledge of Location

1. Select **AI Coarse** mode (requires `GEMINI_API_KEY`)
2. Gemini reads visual cues (signs, architecture, vegetation, license plates)
3. Returns approximate region → system auto-sets center + radius
4. Full pipeline runs from there

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11  # match your Python version
```

### `import lightglue` fails

```bash
pip install git+https://github.com/cvg/LightGlue.git
```
Do **not** install from PyPI — only the GitHub source version is compatible.

### `kornia` not found (Ultra Mode crashes)

```bash
pip install kornia
```

### CUDA out of memory

- Reduce `top_k` candidates (try 200 instead of 500)
- Switch from ALIKED (1024 kp) to DISK manually by ensuring non-CUDA device is selected
- Close other GPU processes

### MPS errors on Apple Silicon

Ensure PyTorch nightly or ≥2.0 with MPS support:
```bash
pip install --upgrade torch torchvision
```

### Index search returns zero results

- Check that center coordinates and radius actually overlap your indexed area
- Confirm `index/cosplace_descriptors.npy` and `index/metadata.npz` exist (run **Create** first, or re-run if interrupted before the auto-build step)
- `cosplace_parts/` chunks exist but `index/` is missing → the auto-build from chunks didn't complete; re-run Create mode to trigger rebuild

### Low accuracy on all images

- Increase index density: lower grid resolution value (e.g., 200 instead of 300) — more panoramas sampled
- Increase search radius to ensure the correct location is within the indexed area
- Enable Ultra Mode
- Verify the query image is street-level (aerial/satellite images are not supported)

### Slow panorama downloads

Netryx downloads panorama tiles during search (not cached at index time). A slow connection will bottleneck Stage 2. Broadband recommended; no workaround except local caching (not built-in).
```
