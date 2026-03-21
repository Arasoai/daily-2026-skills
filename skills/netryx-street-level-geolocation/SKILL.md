```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - reverse image geolocation
  - netryx geolocation
  - index street view panoramas
  - match photo to location
  - osint geolocation tool
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images using a three-stage CV pipeline: CosPlace global retrieval → ALIKED/DISK + LightGlue geometric verification → spatial refinement. No cloud APIs required for searching — runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must install from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Fix blank GUI on macOS
```bash
brew install python-tk@3.11   # match your Python version
```

### Gemini API Key (optional, for AI Coarse blind geolocation)
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. It handles:
- Index creation (crawling + embedding)
- Search (upload photo → get GPS result)
- Real-time scanning visualization

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone index builder for large datasets
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (built during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Key Concepts

### The Three-Stage Pipeline

**Stage 1 — Global Retrieval (CosPlace)**
- Extracts a 512-dim fingerprint from the query image (+ horizontally flipped version)
- Cosine similarity search against the index, filtered by haversine radius
- Returns top 500–1000 visually similar panorama candidates
- Runs in < 1 second regardless of index size

**Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)**
- Downloads each candidate panorama (8 tiles, stitched)
- Crops at 3 fields of view: 70°, 90°, 110° (handles zoom mismatch)
- ALIKED (CUDA) or DISK (MPS/CPU) extracts keypoints
- LightGlue matches keypoints between query and candidate
- RANSAC filters to geometrically consistent inliers
- Takes 2–5 minutes for 300–500 candidates

**Stage 3 — Refinement**
- Heading refinement: tests ±45° offsets at 15° steps for top 15 candidates
- Spatial consensus: clusters matches into 50m cells, prefers clusters over outliers
- Confidence scoring: evaluates geographic clustering + uniqueness ratio

### Ultra Mode
Enable for night shots, blur, low texture:
- **LoFTR**: detector-free dense matcher (handles blur/low-contrast)
- **Descriptor hopping**: re-searches index using descriptor from matched panorama
- **Neighborhood expansion**: searches all panoramas within 100m of best match

### Index Design
- Single unified index for all regions
- Search is region-agnostic: coordinates + radius filter handles city selection
- Index multiple cities into the same `cosplace_descriptors.npy`

---

## Creating an Index

### Via GUI
1. Select **Create** mode
2. Enter center lat/lon of area to index
3. Set radius (start: 0.5–1km; production: 5–10km)
4. Grid resolution: 300 (default, don't change)
5. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

### Indexing Time Estimates

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

### For Large Datasets — Standalone Builder
```bash
python build_index.py
```

---

## Searching

### Via GUI
1. Select **Search** mode
2. Upload a street-level photo
3. Choose method:
   - **Manual**: Enter center lat/lon + radius (use if you know approximate region)
   - **AI Coarse**: Gemini analyzes signs/architecture to guess region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Code Examples

### Extract a CosPlace Descriptor

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"

model = load_cosplace_model(device=device)

img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device=device)
# descriptor.shape == (512,)
print(f"Descriptor shape: {descriptor.shape}")
```

### Load the Index and Run a Cosine Similarity Search

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity
from math import radians, sin, cos, sqrt, atan2

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
panoids = meta["panoids"]
headings = meta["headings"]

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

def search_index(query_descriptor, center_lat, center_lon, radius_km=2.0, top_k=500):
    """
    Returns indices of top_k candidates within radius_km of center.
    """
    # Radius filter
    distances = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i])
        for i in range(len(lats))
    ])
    in_radius = np.where(distances <= radius_km)[0]

    if len(in_radius) == 0:
        return []

    # Cosine similarity
    q = query_descriptor.reshape(1, -1)
    sims = cosine_similarity(q, descriptors[in_radius])[0]

    # Sort by similarity descending
    ranked = in_radius[np.argsort(sims)[::-1][:top_k]]
    return ranked

# Usage
query_desc = extract_descriptor(model, img, device=device)
candidates = search_index(query_desc, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
print(f"Found {len(candidates)} candidates")
```

### Run LightGlue Matching Between Two Images

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

# Load images (LightGlue expects tensors in [0,1], shape CHW)
image0 = load_image("query_photo.jpg").to(device)
image1 = load_image("candidate_panorama_crop.jpg").to(device)

# Extract features
feats0 = extractor.extract(image0.unsqueeze(0))
feats1 = extractor.extract(image1.unsqueeze(0))

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

def ransac_verify(kpts0_np, kpts1_np, threshold=4.0):
    """
    Returns number of geometric inliers.
    kpts0_np, kpts1_np: (N, 2) float32 arrays of matched keypoints.
    """
    if len(kpts0_np) < 8:
        return 0

    _, mask = cv2.findFundamentalMat(
        kpts0_np, kpts1_np,
        cv2.FM_RANSAC,
        ransacReprojThreshold=threshold,
        confidence=0.999
    )
    if mask is None:
        return 0
    return int(mask.sum())

# Convert torch tensors to numpy
kpts0_np = kpts0.cpu().numpy()
kpts1_np = kpts1.cpu().numpy()
inliers = ransac_verify(kpts0_np, kpts1_np)
print(f"RANSAC inliers: {inliers}")
```

### Ultra Mode — LoFTR Dense Matching

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
matcher = kornia.feature.LoFTR(pretrained="outdoor").eval().to(device)

def loftr_match(img0_path, img1_path, device):
    def preprocess(path):
        img = cv2.imread(path, cv2.IMREAD_GRAYSCALE)
        img = cv2.resize(img, (640, 480))
        return torch.from_numpy(img).float()[None, None] / 255.0

    img0 = preprocess(img0_path).to(device)
    img1 = preprocess(img1_path).to(device)

    with torch.no_grad():
        out = matcher({"image0": img0, "image1": img1})

    kpts0 = out["keypoints0"].cpu().numpy()
    kpts1 = out["keypoints1"].cpu().numpy()
    conf = out["confidence"].cpu().numpy()

    # Filter by confidence
    mask = conf > 0.5
    return kpts0[mask], kpts1[mask]

kpts0, kpts1 = loftr_match("query.jpg", "candidate.jpg", device)
inliers = ransac_verify(kpts0, kpts1)
print(f"LoFTR inliers: {inliers}")
```

---

## Common Patterns

### Pattern: Index Multiple Cities, Search by Coordinates
```python
# Index Paris → index/  (center=48.8566, 2.3522, radius=5km)
# Index London → index/ (center=51.5074, -0.1278, radius=5km)
# Same index files — radius filter handles separation at search time

# Search Paris only:
candidates = search_index(query_desc, center_lat=48.8566, center_lon=2.3522, radius_km=5.0)

# Search London only:
candidates = search_index(query_desc, center_lat=51.5074, center_lon=-0.1278, radius_km=5.0)
```

### Pattern: Flip Augmentation for Retrieval
```python
from PIL import Image, ImageOps

img = Image.open("query.jpg").convert("RGB")
img_flipped = ImageOps.mirror(img)

desc_orig = extract_descriptor(model, img, device=device)
desc_flip = extract_descriptor(model, img_flipped, device=device)

# Average or take best of both similarity scores
combined_desc = (desc_orig + desc_flip) / 2
```

### Pattern: Multi-FOV Candidate Cropping
```python
# For each candidate panorama, test 3 FOVs to handle zoom mismatch
FOVS = [70, 90, 110]

best_inliers = 0
best_fov = None

for fov in FOVS:
    crop = extract_rectilinear_crop(panorama, heading=heading, fov=fov, output_size=(640, 480))
    # ... run LightGlue + RANSAC
    if inliers > best_inliers:
        best_inliers = inliers
        best_fov = fov
```

### Pattern: Spatial Consensus Clustering
```python
from collections import defaultdict

def cluster_matches(candidates_with_coords, cell_size_m=50):
    """Group candidates into 50m grid cells, prefer larger clusters."""
    cells = defaultdict(list)
    for cand in candidates_with_coords:
        # ~50m per 0.0005 degrees at mid-latitudes
        cell_lat = round(cand["lat"] / 0.0005)
        cell_lon = round(cand["lon"] / 0.0005)
        cells[(cell_lat, cell_lon)].append(cand)

    # Return cell with most candidates (consensus location)
    best_cell = max(cells.values(), key=lambda c: sum(x["inliers"] for x in c))
    return best_cell
```

---

## GPU / Device Selection

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|:---:|:---:|:---:|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| LightGlue | ✅ | ✅ | ✅ (slow) |
| LoFTR (Ultra) | ✅ | ✅ | ✅ (very slow) |
| Recommended VRAM | 8GB+ | unified 8GB+ | N/A |

```python
import torch

def get_device():
    if torch.cuda.is_available():
        return torch.device("cuda")
    elif torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| GUI appears blank on macOS | `brew install python-tk@3.11` (match your Python version) |
| `ModuleNotFoundError: lightglue` | Run `pip install git+https://github.com/cvg/LightGlue.git` |
| `ModuleNotFoundError: kornia` | Run `pip install kornia` (Ultra Mode only) |
| Indexing crashes midway | Safe to restart — indexing resumes from last saved chunk in `cosplace_parts/` |
| Zero candidates returned | Radius too small, or area not indexed. Increase radius or re-index. |
| Low confidence result | Enable Ultra Mode; try a larger search radius; ensure good lighting in query image |
| CUDA OOM | Reduce `max_num_keypoints` in ALIKED (e.g. 512); process fewer candidates in parallel |
| AI Coarse mode fails | Check `GEMINI_API_KEY` env var is set; fall back to Manual mode with approximate coordinates |
| Slow matching on CPU | Expected — use GPU. CPU processes ~10-20 candidates/min vs 100-200/min on GPU |

### Index Rebuild After Adding New Areas
After adding new `cosplace_parts/*.npz` files (new indexed areas), the searchable index must be rebuilt:
```bash
python build_index.py
# This regenerates index/cosplace_descriptors.npy and index/metadata.npz
```

---

## Model References

| Model | Role | Source |
|-------|------|--------|
| CosPlace | Global visual retrieval (512-dim descriptor) | [github.com/gmberton/cosplace](https://github.com/gmberton/cosplace) |
| ALIKED | Keypoint extraction on CUDA | [github.com/naver/alike](https://github.com/naver/alike) |
| DISK | Keypoint extraction on MPS/CPU | via LightGlue |
| LightGlue | Deep feature matching | [github.com/cvg/LightGlue](https://github.com/cvg/LightGlue) |
| LoFTR | Dense detector-free matching (Ultra Mode) | via kornia |
```
