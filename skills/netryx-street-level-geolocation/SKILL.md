```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace + LightGlue to identify GPS coordinates from any street photo
triggers:
  - geolocate a street photo
  - identify location from image
  - street view geolocation
  - find GPS coordinates from photo
  - osint geolocation tool
  - reverse geolocation street image
  - netryx geolocation setup
  - index street view panoramas
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It combines CosPlace (global visual retrieval), ALIKED/DISK (local feature extraction), and LightGlue (deep feature matching) to achieve sub-50m accuracy without needing internet presence of the query image.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode dense matching
pip install kornia
```

**macOS tkinter fix (blank GUI):**
```bash
brew install python-tk@3.11  # match your Python version
```

**Optional Gemini AI key for AI Coarse mode:**
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

GPU backends: CUDA (NVIDIA), MPS (Apple Silicon M1+), CPU (slow).

---

## Launch the GUI

```bash
python test_super.py
```

The GUI has two modes: **Create** (build an index) and **Search** (geolocate a photo).

---

## Core Workflow

### Step 1: Create an Index

Index a geographic area by crawling Street View panoramas and extracting CosPlace fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon of the target area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index** — saves incrementally to `cosplace_parts/`

**Via standalone builder (large datasets):**
```bash
python build_index.py
```

Index size estimates:

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Indexing is resumable — interrupted runs continue from the last checkpoint.

### Step 2: Search

1. Select **Search** mode
2. Upload a street-level image
3. Choose method:
   - **Manual**: provide approximate center lat/lon + radius
   - **AI Coarse**: Gemini analyzes visual clues to guess region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. View result on map with GPS coordinates and confidence score

**Ultra Mode** (checkbox): enables LoFTR dense matching, descriptor hopping, and 100m neighborhood expansion. Use for blurry, nighttime, or low-texture images.

---

## Project Structure

```
netryx/
├── test_super.py          # Main GUI application
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace descriptor (512-dim)
    ├── Flipped descriptor (catches reversed perspectives)
    │
    ▼
Index Search — cosine similarity, haversine radius filter → top 500 candidates
    │
    ▼
Download panoramas → 3-FOV crops (70°/90°/110°) → ALIKED/DISK keypoints
    │
    ├── LightGlue matching
    ├── RANSAC geometric verification
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

## Code Examples

### Extract a CosPlace Descriptor Manually

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
model, device = load_cosplace_model()

# Extract descriptor from an image file
image = Image.open("street_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, image, device)
# descriptor.shape == (512,)
print(f"Descriptor shape: {descriptor.shape}")
print(f"Device used: {device}")
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512) float32
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
panoids = meta["panoids"]
headings = meta["headings"]

def search_index(query_descriptor, center_lat, center_lon, radius_km=2.0, top_k=500):
    """
    Returns top-K candidates within radius, sorted by cosine similarity.
    """
    # Radius filter
    mask = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
        for i in range(len(lats))
    ])
    filtered_desc = descriptors[mask]
    filtered_indices = np.where(mask)[0]

    if len(filtered_indices) == 0:
        return []

    # Cosine similarity (descriptors assumed L2-normalized)
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    sims = filtered_desc @ q  # (M,)

    # Top-K
    top_k_local = min(top_k, len(sims))
    top_idx = np.argpartition(sims, -top_k_local)[-top_k_local:]
    top_idx = top_idx[np.argsort(sims[top_idx])[::-1]]

    results = []
    for local_i in top_idx:
        global_i = filtered_indices[local_i]
        results.append({
            "lat": float(lats[global_i]),
            "lon": float(lons[global_i]),
            "panoid": str(panoids[global_i]),
            "heading": float(headings[global_i]),
            "similarity": float(sims[local_i]),
        })
    return results

# Usage
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

model, device = load_cosplace_model()
image = Image.open("query.jpg").convert("RGB")
desc = extract_descriptor(model, image, device)

candidates = search_index(
    query_descriptor=desc,
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=2.0,
    top_k=500,
)
print(f"Found {len(candidates)} candidates")
print(f"Top match: {candidates[0]}")
```

### LightGlue Feature Matching (Stage 2)

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

# Select extractor based on available hardware
device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def match_images(query_path: str, candidate_path: str):
    """Returns number of RANSAC-verified inliers between two images."""
    image0 = load_image(query_path).to(device)
    image1 = load_image(candidate_path).to(device)

    feats0 = extractor.extract(image0)
    feats1 = extractor.extract(image1)

    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

    matched_kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
    matched_kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]

    # RANSAC verification
    if len(matched_kpts0) >= 4:
        import cv2
        pts0 = matched_kpts0.cpu().numpy()
        pts1 = matched_kpts1.cpu().numpy()
        _, inlier_mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, 5.0)
        inliers = int(inlier_mask.sum()) if inlier_mask is not None else 0
    else:
        inliers = 0

    return inliers

score = match_images("query.jpg", "candidate_panorama_crop.jpg")
print(f"Inlier matches: {score}")
```

### Spatial Consensus Clustering

```python
from collections import defaultdict

def cluster_candidates(candidates, cell_size_m=50):
    """
    Group candidates into ~50m geographic cells.
    Returns the largest cluster's representative candidate.
    """
    METERS_PER_DEG_LAT = 111_000
    
    def to_cell(lat, lon):
        cell_lat = round(lat * METERS_PER_DEG_LAT / cell_size_m)
        cell_lon = round(lon * METERS_PER_DEG_LAT * abs(cos(radians(lat))) / cell_size_m)
        return (cell_lat, cell_lon)

    from math import cos, radians
    clusters = defaultdict(list)
    for c in candidates:
        cell = to_cell(c["lat"], c["lon"])
        clusters[cell].append(c)

    # Return best candidate from largest cluster
    best_cluster = max(clusters.values(), key=lambda g: sum(x["inliers"] for x in g))
    return max(best_cluster, key=lambda x: x["inliers"])

# Usage after LightGlue scoring
for c in candidates[:50]:
    c["inliers"] = match_images("query.jpg", download_crop(c["panoid"], c["heading"]))

best = cluster_candidates([c for c in candidates if c["inliers"] > 0])
print(f"Best match: {best['lat']}, {best['lon']} ({best['inliers']} inliers)")
```

---

## Multi-Index Strategy

All areas are stored in a single unified index. Radius filtering handles separation:

```python
# Index Paris + London into the same index — no conflict
# Search Paris only:
paris_results = search_index(desc, center_lat=48.8566, center_lon=2.3522, radius_km=5.0)

# Search London only:
london_results = search_index(desc, center_lat=51.5074, center_lon=-0.1278, radius_km=5.0)
```

---

## Models Reference

| Model | Purpose | Hardware |
|-------|---------|----------|
| CosPlace | Global 512-dim visual fingerprint | All |
| ALIKED | Local keypoints + descriptors | CUDA only |
| DISK | Local keypoints + descriptors | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense detector-free matching (Ultra Mode) | All (slow on CPU) |

---

## Configuration Tips

- **Grid resolution 300** is the safe default — don't lower it (gaps in coverage)
- **top_k=500** for standard searches; reduce to 200 for speed in well-indexed areas
- **radius_km**: start at 0.5–1km for testing; 5–10km for production searches
- **Ultra Mode**: only enable for genuinely difficult images — adds 3–5× search time
- **RANSAC threshold**: 5.0 pixels is standard; lower to 3.0 for high-resolution query images

---

## Troubleshooting

**GUI appears blank on macOS:**
```bash
brew install python-tk@3.11  # match your exact Python version
```

**CUDA out of memory:**
- Reduce `max_num_keypoints` in ALIKED (try 512 instead of 1024)
- Process fewer candidates per search (lower `top_k`)

**LightGlue import error:**
```bash
pip install git+https://github.com/cvg/LightGlue.git --force-reinstall
```

**Index search returns 0 results:**
- Verify `index/cosplace_descriptors.npy` and `index/metadata.npz` exist
- Run index auto-build after creating parts: the GUI triggers this automatically, or run `build_index.py`
- Check that your search radius covers the indexed area

**Descriptor hopping not finding correct match:**
- Increase `neighborhood_expansion` radius to 200m
- Enable Ultra Mode for the LoFTR fallback path

**Slow indexing:**
- Use `build_index.py` (standalone builder) instead of the GUI for large areas
- Indexing is I/O-bound on Street View tile downloads — a faster connection helps more than GPU

**MPS device errors (Apple Silicon):**
```python
# Force CPU if MPS causes issues
import os
os.environ["PYTORCH_ENABLE_MPS_FALLBACK"] = "1"
```

---

## Environment Variables

```bash
export GEMINI_API_KEY="..."        # AI Coarse geolocation mode (optional)
export PYTORCH_ENABLE_MPS_FALLBACK="1"  # MPS op fallback to CPU (macOS)
```
```
