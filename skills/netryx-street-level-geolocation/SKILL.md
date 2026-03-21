```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - netryx geolocation
  - index street view panoramas
  - locate where a photo was taken
  - open source geolocation pipeline
  - reverse geolocate image coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that finds the precise GPS coordinates of any street-level photograph. It crawls street-view panoramas, indexes them with CosPlace visual fingerprints, then uses LightGlue feature matching and RANSAC verification to narrow millions of candidates down to a single location — achieving sub-50m accuracy with no landmarks required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                        # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

**GPU backends:** CUDA (NVIDIA) → ALIKED extractor · MPS (Apple Silicon) → DISK extractor · CPU → DISK (slow)

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The indexer crawls Street View panoramas and extracts CosPlace 512-dim fingerprints.

**GUI:**
1. Select **Create** mode
2. Enter center lat/lng
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time reference:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hrs       | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hrs      | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hrs     | ~7 GB      |

Indexing is resumable — interrupt and restart without data loss.

### Step 2 — Search

**GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Enter approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loading & descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), written during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Lat/lng, headings, panorama IDs
```

---

## Pipeline Deep-Dive

### Stage 1 — Global Retrieval (CosPlace)

```
Query Image → 512-dim CosPlace descriptor
            + flipped descriptor (catches reversed perspectives)
            → cosine similarity against full index
            → haversine radius filter
            → top 500–1000 candidates
```

Runs in under 1 second — single matrix multiplication regardless of index size.

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```
For each candidate:
  Download panorama (8 tiles, stitched)
  → Crop at indexed heading
  → Multi-FOV crops: 70°, 90°, 110°
  → ALIKED (CUDA) / DISK (MPS/CPU) keypoint extraction
  → LightGlue deep feature matching vs query keypoints
  → RANSAC: filter to geometrically consistent inliers
  → Rank by inlier count
```

Processes 300–500 candidates in 2–5 minutes on modern hardware.

### Stage 3 — Refinement

- **Heading refinement**: Tests ±45° offsets at 15° steps across 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; clusters beat single high-inlier outliers
- **Confidence scoring**: Evaluates geographic clustering + uniqueness ratio (best vs runner-up)

### Ultra Mode

Enable for difficult images (night, blur, low texture):

- **LoFTR**: Detector-free dense matching — works without keypoints
- **Descriptor hopping**: Re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of the best match

---

## Using CosPlace Utilities Directly

```python
# cosplace_utils.py exposes model loading and descriptor extraction
from cosplace_utils import load_cosplace_model, get_cosplace_descriptor
from PIL import Image
import torch

device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"

model = load_cosplace_model(device=device)

img = Image.open("query_photo.jpg")
descriptor = get_cosplace_descriptor(model, img, device=device)
# descriptor shape: (512,) — float32 numpy array
```

### Manual Index Search (bypassing GUI)

```python
import numpy as np

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512)
meta        = np.load("index/metadata.npz", allow_pickle=True)
lats        = meta["lats"]      # shape: (N,)
lngs        = meta["lngs"]      # shape: (N,)
headings    = meta["headings"]  # shape: (N,)
panoids     = meta["panoids"]   # shape: (N,)

def cosine_similarity_search(query_desc, descriptors, top_k=500):
    """Return indices of top_k most similar descriptors."""
    query_norm = query_desc / (np.linalg.norm(query_desc) + 1e-8)
    db_norms   = descriptors / (np.linalg.norm(descriptors, axis=1, keepdims=True) + 1e-8)
    scores     = db_norms @ query_norm
    return np.argsort(scores)[::-1][:top_k]

def haversine_filter(lats, lngs, center_lat, center_lng, radius_km):
    """Return boolean mask for entries within radius_km of center."""
    R = 6371.0
    dlat = np.radians(lats - center_lat)
    dlng = np.radians(lngs - center_lng)
    a    = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlng/2)**2
    return 2 * R * np.arcsin(np.sqrt(a)) <= radius_km

# Example: search near Paris within 2 km
center_lat, center_lng, radius_km = 48.8566, 2.3522, 2.0
mask      = haversine_filter(lats, lngs, center_lat, center_lng, radius_km)
local_idx = np.where(mask)[0]

top_indices = cosine_similarity_search(descriptor, descriptors[local_idx])
top_global  = local_idx[top_indices]

print(f"Top match: lat={lats[top_global[0]]:.6f}, lng={lngs[top_global[0]]:.6f}")
print(f"Panorama ID: {panoids[top_global[0]]}, Heading: {headings[top_global[0]]}")
```

### Building the Index Standalone (Large Datasets)

```bash
# Use build_index.py directly for headless/server environments
python build_index.py \
    --lat 48.8566 \
    --lng 2.3522 \
    --radius 5.0 \
    --resolution 300
```

### Descriptor Extraction for Custom Images

```python
from cosplace_utils import load_cosplace_model, get_cosplace_descriptor
from PIL import Image
import numpy as np
import torch

def extract_and_flip_descriptors(image_path: str, model, device: str):
    """Extract both normal and horizontally-flipped CosPlace descriptors."""
    img     = Image.open(image_path).convert("RGB")
    img_flp = img.transpose(Image.FLIP_LEFT_RIGHT)

    desc     = get_cosplace_descriptor(model, img,     device=device)
    desc_flp = get_cosplace_descriptor(model, img_flp, device=device)

    # Average or keep both for dual-search
    return desc, desc_flp

device = "cuda" if torch.cuda.is_available() else "cpu"
model  = load_cosplace_model(device=device)
d, d_flip = extract_and_flip_descriptors("street.jpg", model, device)
```

---

## Multi-Index Strategy (Multiple Cities)

All embeddings live in a single unified index. No city separation needed — use coordinates + radius to scope searches:

```python
# Index Paris
# python test_super.py → Create → lat=48.8566, lng=2.3522, radius=5

# Index London  
# python test_super.py → Create → lat=51.5074, lng=-0.1278, radius=5

# Search only Paris results
mask_paris  = haversine_filter(lats, lngs, 48.8566,  2.3522, 5.0)

# Search only London results
mask_london = haversine_filter(lats, lngs, 51.5074, -0.1278, 5.0)
```

---

## Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| Grid resolution | 300 | Sampling density for panorama crawl — don't modify |
| Top candidates (Stage 1) | 500–1000 | Returned by CosPlace cosine search |
| Heading offsets (refinement) | ±45° at 15° steps | Tests 7 angles × 3 FOVs = 21 crops per candidate |
| FOV values | 70°, 90°, 110° | Multi-FOV crops for zoom mismatch handling |
| Refinement top-N | 15 | Number of candidates sent to heading refinement |
| Spatial cluster cell | 50 m | Grid size for consensus clustering |
| Ultra neighborhood | 100 m | Expansion radius for neighborhood search |
| Descriptor dim | 512 | CosPlace output dimensionality |
| ALIKED keypoints | 1024 | Used on CUDA |
| DISK keypoints | 768 | Used on MPS/CPU |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match your Python version
```

### LightGlue import error
```bash
pip install git+https://github.com/cvg/LightGlue.git
# Not available on PyPI — must install from GitHub
```

### LoFTR / Ultra Mode unavailable
```bash
pip install kornia
```

### CUDA out of memory
- Reduce `top_k` candidates in Stage 1 (500 → 200)
- Disable Ultra Mode
- Use DISK instead of ALIKED (happens automatically on MPS/CPU)

### Index not found on startup
Ensure `cosplace_parts/` and `index/` directories exist. Run **Create Index** before **Search**.

### Slow indexing
- Indexing is I/O and network bound — broadband recommended
- Use `build_index.py` for headless server runs on large areas
- Process is resumable; `cosplace_parts/*.npz` are written incrementally

### Low confidence results
1. Enable **Ultra Mode**
2. Expand search radius
3. Re-index the target area at higher density (lower grid resolution value cautiously)
4. Try **AI Coarse** mode if region is unknown — requires `GEMINI_API_KEY`

### Gemini AI Coarse mode not working
```bash
export GEMINI_API_KEY="your_key_here"
# Verify it's set:
echo $GEMINI_API_KEY
```

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim visual fingerprint | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoints + descriptors | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints + descriptors | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Detector-free dense matching (Ultra) | All (via kornia) |

---

## Quick-Start Checklist

```
□ Clone repo and install dependencies
□ pip install git+https://github.com/cvg/LightGlue.git
□ python test_super.py
□ Create Index: enter lat/lng, radius, click Create Index
□ Wait for indexing to complete (or interrupt and resume later)
□ Search: upload photo, set coordinates + radius, click Run Search
□ Enable Ultra Mode for difficult/degraded images
□ Export GEMINI_API_KEY for AI Coarse blind geolocation
```
```
