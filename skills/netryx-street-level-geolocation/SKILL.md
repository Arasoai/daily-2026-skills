```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine using CosPlace, ALIKED/DISK, and LightGlue to identify GPS coordinates from street photos.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - reverse geolocate image
  - netryx geolocation
  - index street view panoramas
  - identify location from photo
  - osint image geolocation
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls and indexes street-view panoramas, then matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and geometric verification (RANSAC). Achieves sub-50m accuracy without landmarks, running entirely on local hardware.

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

### macOS tkinter fix (blank GUI)
```bash
brew install python-tk@3.11  # match your Python version
```

### Optional: Gemini API for AI Coarse mode
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. All indexing and searching is driven from here.

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. This crawls street-view panoramas and stores CosPlace fingerprints.

**GUI steps:**
1. Select **Create** mode
2. Enter center latitude/longitude
3. Set search radius (km)
4. Set grid resolution (default: 300 — don't change)
5. Click **Create Index**

**Index size reference:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Indexing is **resumable** — interrupt and restart safely.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide center coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### Ultra Mode

Enable **Ultra Mode** checkbox for:
- Night/blurry/low-texture images
- LoFTR dense matching (no keypoint detection needed)
- Descriptor hopping (re-search from matched panorama's clean descriptor)
- Neighborhood expansion (±100m around best match)

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

- Extracts 512-dim descriptor from query image
- Also extracts descriptor from horizontally-flipped version
- Cosine similarity search against entire index
- Haversine radius filter applied
- Returns top 500–1000 candidates
- Runtime: **< 1 second** (single matrix multiplication)

### Stage 2 — Local Geometric Verification

For each candidate:
1. Download Street View panorama (8 tiles, stitched)
2. Crop at indexed heading angle
3. Generate multi-FOV crops: 70°, 90°, 110°
4. Extract keypoints: **ALIKED** (CUDA) or **DISK** (MPS/CPU)
5. **LightGlue** deep feature matching
6. **RANSAC** geometric verification (filters false matches)
7. Winner = most verified inliers

Runtime: **2–5 minutes** for 300–500 candidates.

### Stage 3 — Refinement

- Heading refinement: ±45° at 15° steps, 3 FOVs, top 15 candidates
- Spatial consensus: cluster matches into 50m cells
- Confidence scoring: clustering density + uniqueness ratio

---

## Working with the Index Programmatically

### Extract a CosPlace descriptor

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-selects CUDA/MPS/CPU)
model, device = get_cosplace_model()

# Extract descriptor from an image file
img = Image.open("query.jpg")
descriptor = get_descriptor(model, img, device)
# descriptor shape: (512,) — numpy array
print(f"Descriptor shape: {descriptor.shape}")
```

### Search the index manually

```python
import numpy as np

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,)
lons = meta["lons"]       # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

# Cosine similarity search
query_desc = descriptor / np.linalg.norm(descriptor)
index_norms = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = index_norms @ query_desc  # (N,)

# Haversine radius filter (example: 2km around Paris center)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 2.0

def haversine(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

distances = haversine(center_lat, center_lon, lats, lons)
mask = distances <= radius_km

# Top 20 candidates within radius
masked_scores = np.where(mask, scores, -1)
top_indices = np.argsort(masked_scores)[::-1][:20]

for idx in top_indices:
    print(f"Score: {scores[idx]:.4f} | Lat: {lats[idx]:.6f} | Lon: {lons[idx]:.6f} | PanoID: {panoids[idx]}")
```

### Build index from parts (standalone)

```bash
python build_index.py
```

This compiles all `cosplace_parts/*.npz` chunks into the unified `index/` directory. Use after manual/partial indexing or when resuming interrupted runs.

---

## Platform-Specific Behavior

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|--------------|---------------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| Speed | Fastest | Fast | Slow |
| Min VRAM | 4GB | 4GB unified | N/A |

```python
# Device selection logic used internally
import torch

if torch.cuda.is_available():
    device = torch.device("cuda")
elif torch.backends.mps.is_available():
    device = torch.device("mps")
else:
    device = torch.device("cpu")
```

---

## Running LightGlue Matching (Standalone Example)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU
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

# Get matched keypoint coordinates
kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
num_matches = len(matches01["matches"])
print(f"Matched keypoints: {num_matches}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kpts0, kpts1, threshold=3.0):
    """
    Filter LightGlue matches with RANSAC homography.
    Returns number of inliers (higher = better geometric match).
    """
    if len(kpts0) < 4:
        return 0

    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()

    H, mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, threshold)
    if mask is None:
        return 0

    inliers = int(mask.sum())
    return inliers

inlier_count = ransac_verify(kpts0, kpts1)
print(f"RANSAC inliers: {inlier_count}")
# >= 50 inliers = strong match; < 20 = weak/false match
```

---

## Multi-City Index Strategy

The index is unified — all cities share one index file. Radius filtering handles separation at search time:

```python
# Index Paris (run in GUI or via build_index.py after crawling)
# center: 48.8566, 2.3522 — stored in metadata

# Index London separately
# center: 51.5074, -0.1278 — stored in metadata

# Search only Paris results:
# center_lat=48.8566, center_lon=2.3522, radius_km=5.0

# Search only London results:
# center_lat=51.5074, center_lon=-0.1278, radius_km=5.0
```

No city labels needed — coordinates + radius fully isolate results.

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # replace 3.11 with your Python version
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### Low match quality / wrong location
- Enable **Ultra Mode** for blurry or low-texture images
- Increase search radius if approximate location is uncertain
- Ensure the indexed area covers the query region
- Check that the query is a street-level photo (not aerial/indoor)

### CUDA out of memory
- Reduce `max_num_keypoints` in extractor init (try 512)
- Process fewer candidates per batch
- Use `torch.cuda.empty_cache()` between batches

### Indexing stalls / incomplete
- Interrupting is safe — resume by re-running **Create Index** with same parameters
- Run `python build_index.py` to recompile parts into the searchable index

### Confidence score is low but location looks correct
- Spatial consensus requires multiple candidates to cluster — works best with dense indexes
- Use manual mode with tight radius (0.5–1km) if you have a rough prior

### `cosplace_parts/` grows large
- Each `.npz` chunk is ~60MB per 500 panoramas
- Safe to delete after `build_index.py` compiles them into `index/`

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim descriptor | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoints | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra) | All (slow on CPU) |

---

## Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `GEMINI_API_KEY` | AI Coarse geolocation via Gemini | No (manual mode works without it) |
```
