```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - visual place recognition locally
  - netryx geolocation
  - index street view panoramas
  - identify location from street photo
  - osint geolocation from image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). Sub-50m accuracy. No landmarks needed. Runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: install LightGlue from source
pip install git+https://github.com/cvg/LightGlue.git

# Optional: install kornia for Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Platform GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works but significantly slower |

### Optional: Gemini API Key (AI Coarse mode)

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

Index a geographic area by crawling street-view panoramas and extracting CosPlace fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon of the target area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Index saves incrementally — safe to interrupt and resume.

### Step 2: Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose **Manual** (enter lat/lon + radius) or **AI Coarse** (Gemini guesses region)
4. Click **Run Search** → **Start Full Search**
5. Result shows on map with GPS coordinates and confidence score

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, and search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

- Extracts a 512-dim descriptor from the query image
- Also extracts descriptor from horizontally-flipped version (catches reversed perspectives)
- Cosine similarity against all indexed descriptors
- Haversine radius filter restricts to specified search area
- Returns top 500–1000 candidates
- **Speed: < 1 second** (single matrix multiply)

### Stage 2 — Local Geometric Verification (ALIKED/DISK + LightGlue)

For each candidate:
1. Downloads panorama from Street View (8 tiles, stitched)
2. Crops at indexed heading angle
3. Generates multi-FOV crops: 70°, 90°, 110°
4. Extracts keypoints with ALIKED (CUDA) or DISK (MPS/CPU)
5. LightGlue deep feature matching against query keypoints
6. RANSAC filters to geometrically consistent inliers
7. Candidate with most verified inliers = best match
- **Speed: 2–5 minutes for 300–500 candidates**

### Stage 3 — Refinement

- **Heading refinement**: Tests ±45° offsets at 15° steps across 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; prefers clusters over outliers
- **Confidence scoring**: Evaluates geographic clustering + uniqueness ratio (best vs. runner-up)

### Ultra Mode

Enable for difficult images (night, blur, low texture):
- **LoFTR**: Detector-free dense matcher — handles blur and low-contrast
- **Descriptor hopping**: Re-searches index using descriptor from matched panorama
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Code Examples

### Extract a CosPlace Descriptor Programmatically

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
model, device = load_cosplace_model()

# Load your query image
image = Image.open("query.jpg").convert("RGB")

# Extract 512-dim descriptor
descriptor = get_descriptor(model, image, device)
print(f"Descriptor shape: {descriptor.shape}")  # (512,)
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image

def haversine(lat1, lon1, lat2, lon2):
    R = 6371000  # meters
    lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = sin(dlat/2)**2 + cos(lat1)*cos(lat2)*sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # shape: (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]

# Load model and extract query descriptor
model, device = load_cosplace_model()
image = Image.open("query.jpg").convert("RGB")
query_desc = get_descriptor(model, image, device)  # shape: (512,)

# Also try flipped version
query_desc_flip = get_descriptor(model, image.transpose(Image.FLIP_LEFT_RIGHT), device)

# Cosine similarity search
scores = descriptors @ query_desc          # shape: (N,)
scores_flip = descriptors @ query_desc_flip

# Radius filter (e.g., search within 2km of Paris center)
center_lat, center_lon = 48.8566, 2.3522
radius_m = 2000

mask = np.array([
    haversine(center_lat, center_lon, lats[i], lons[i]) <= radius_m
    for i in range(len(lats))
])

# Apply mask and get top candidates
scores[~mask] = -np.inf
scores_flip[~mask] = -np.inf
combined = np.maximum(scores, scores_flip)
top_indices = np.argsort(combined)[::-1][:500]

print(f"Top match: lat={lats[top_indices[0]]:.6f}, lon={lons[top_indices[0]]:.6f}")
print(f"Score: {combined[top_indices[0]]:.4f}")
```

### Build Index for a Large Area (CLI)

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --grid_resolution 300
```

### Using Multiple City Indexes (Unified Index Pattern)

```python
# All cities index into the SAME index — radius filter handles separation
# Index Paris
# python build_index.py --lat 48.8566 --lon 2.3522 --radius 5.0

# Index London  
# python build_index.py --lat 51.5074 --lon -0.1278 --radius 5.0

# Search only Paris (radius=5km around Paris center)
center_lat, center_lon = 48.8566, 2.3522
radius_m = 5000  # only Paris results returned

# Search only London
center_lat, center_lon = 51.5074, -0.1278
radius_m = 5000  # only London results returned
```

### Feature Matching with LightGlue

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

# Detect device
device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU (Netryx convention)
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

# Load images
image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

# Extract features
feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(kpts0)}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kpts0, kpts1, threshold=3.0):
    """
    Filter matches to geometrically consistent inliers.
    Returns number of inliers (higher = better match).
    """
    if len(kpts0) < 8:
        return 0
    
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    
    _, mask = cv2.findFundamentalMat(
        pts0, pts1,
        cv2.FM_RANSAC,
        ransacReprojThreshold=threshold,
        confidence=0.999
    )
    
    if mask is None:
        return 0
    
    inliers = int(mask.sum())
    return inliers

# Usage
inliers = ransac_verify(kpts0, kpts1)
print(f"Inliers after RANSAC: {inliers}")
# >50 inliers = strong match, >100 = very confident
```

### Ultra Mode: LoFTR Dense Matching

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
matcher = kornia.feature.LoFTR(pretrained="outdoor").to(device)

def loftr_match(img0_path, img1_path, device):
    img0 = cv2.imread(img0_path, cv2.IMREAD_GRAYSCALE)
    img1 = cv2.imread(img1_path, cv2.IMREAD_GRAYSCALE)
    
    img0 = cv2.resize(img0, (640, 480))
    img1 = cv2.resize(img1, (640, 480))
    
    t0 = torch.from_numpy(img0).float()[None, None] / 255.0
    t1 = torch.from_numpy(img1).float()[None, None] / 255.0
    
    with torch.no_grad():
        output = matcher({"image0": t0.to(device), "image1": t1.to(device)})
    
    kpts0 = output["keypoints0"].cpu().numpy()
    kpts1 = output["keypoints1"].cpu().numpy()
    confidence = output["confidence"].cpu().numpy()
    
    # Filter by confidence
    good = confidence > 0.5
    return kpts0[good], kpts1[good]
```

---

## Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `grid_resolution` | 300 | Grid density for panorama crawling — do not lower |
| `radius_km` | 1.0 | Search/index radius in kilometers |
| `top_candidates` | 500 | CosPlace retrieval candidates for Stage 2 |
| `heading_offsets` | ±45° @ 15° | Heading refinement sweep range/step |
| `fov_list` | [70, 90, 110] | Multi-FOV crops for zoom mismatch handling |
| `cluster_cell_m` | 50 | Spatial consensus grid cell size in meters |
| `top_refine` | 15 | Number of candidates for heading refinement |
| `neighborhood_m` | 100 | Ultra Mode neighborhood expansion radius |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # match your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED (try 512 instead of 1024)
- Reduce `top_candidates` from 500 to 200
- Enable `torch.cuda.empty_cache()` between candidates

### MPS errors on Apple Silicon
- DISK is used automatically over ALIKED on MPS — this is expected
- If MPS errors occur: `export PYTORCH_ENABLE_MPS_FALLBACK=1`

### Index not resuming after interrupt
- Check `cosplace_parts/` directory — parts are saved incrementally
- Re-run with same parameters; already-indexed panoramas are skipped

### Poor match quality (< 20 inliers)
1. Enable **Ultra Mode** for low-quality/blurry images
2. Increase index density (lower grid_resolution slightly)
3. Expand search radius — correct location may be near radius boundary
4. Try AI Coarse mode if region is completely unknown

### Gemini API key not working
```bash
# Verify it's set
echo $GEMINI_API_KEY

# Set in current session
export GEMINI_API_KEY="your_key_here"

# Or add to .env and load with python-dotenv
```

---

## Key Models Reference

| Model | Used For | Hardware |
|-------|----------|----------|
| CosPlace (512-dim) | Global retrieval fingerprinting | All |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS/CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | CUDA/CPU |

---

## Data Flow Summary

```
# Indexing
Grid points → Street View API → Panoramas → CosPlace → cosplace_parts/*.npz
cosplace_parts/*.npz → index/cosplace_descriptors.npy + index/metadata.npz

# Searching  
Query image → CosPlace (512-dim) → cosine similarity (radius-filtered)
→ top 500 candidates → download panoramas → ALIKED/DISK + LightGlue
→ RANSAC inlier count → heading refinement → spatial consensus
→ GPS coordinates + confidence score
```
```
