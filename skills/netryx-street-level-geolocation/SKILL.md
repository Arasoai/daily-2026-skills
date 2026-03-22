```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - use netryx to locate
  - build a geolocation index
  - identify location from street view photo
  - osint geolocation with computer vision
  - where was this photo taken
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls street-view panoramas, builds a searchable index of visual fingerprints, then matches query images using a three-stage computer vision pipeline: global retrieval (CosPlace) → local geometric verification (ALIKED/DISK + LightGlue) → spatial refinement.

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

**macOS tkinter fix (blank GUI):**
```bash
brew install python-tk@3.11   # match your Python version
```

**Optional — Gemini AI Coarse mode:**
```bash
export GEMINI_API_KEY="your_key_here"
```

### Hardware requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

GPU backends: CUDA (NVIDIA), MPS (Apple Silicon M1+), or CPU (slow).

---

## Launch the GUI

```bash
python test_super.py
```

The GUI exposes two modes: **Create** (build an index for an area) and **Search** (geolocate a query image).

---

## Core Workflow

### 1. Build an index for a geographic area

In the GUI:
1. Select **Create** mode
2. Enter center lat/lng of the target area
3. Set radius (km) and grid resolution (default 300 — do not change)
4. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

**Index size reference:**

| Radius | Panoramas | Time (M2 Max) | Disk |
|--------|-----------|---------------|------|
| 0.5 km | ~500      | 30 min        | ~60 MB |
| 1 km   | ~2 000    | 1–2 h         | ~250 MB |
| 5 km   | ~30 000   | 8–12 h        | ~3 GB |
| 10 km  | ~100 000  | 24–48 h       | ~7 GB |

### 2. Search (geolocate an image)

1. Select **Search** mode
2. Upload a street photo
3. Choose method:
   - **Manual** — enter approximate center coords + radius
   - **AI Coarse** — Gemini infers region from visual cues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on an interactive map

### 3. Ultra Mode (hard images)

Enable the **Ultra Mode** checkbox before searching. Adds:
- **LoFTR** dense matching (handles blur, low texture, night)
- **Descriptor hopping** — re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion** — checks all panoramas within 100 m of best match

Significantly slower; use when standard mode returns low confidence.

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim visual fingerprints
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep-Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py pattern
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()          # loads pretrained CosPlace (512-dim)
descriptor = get_descriptor(model, image_tensor)          # shape: (512,)
flipped_descriptor = get_descriptor(model, flipped_tensor) # catches reversed perspectives

# Index search: cosine similarity matrix multiplication
# scores = descriptors_matrix @ query_descriptor  → top-k candidates
# Radius filter via haversine distance on metadata lat/lng
```

- One matrix multiply against the entire index → sub-1-second retrieval
- Returns top 500–1000 candidates ranked by visual similarity

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```python
# Pseudocode matching the internal pipeline
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps"  if torch.backends.mps.is_available() else "cpu")

# Device-aware extractor selection
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

query_img = load_image("query.jpg").to(device)
candidate_img = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(query_img)
feats1 = extractor.extract(candidate_img)

matches = matcher({"image0": feats0, "image1": feats1})
matches = rbd(matches)   # remove batch dimension

# RANSAC geometric verification
matched_kpts0 = feats0["keypoints"][matches["matches"][..., 0]]
matched_kpts1 = feats1["keypoints"][matches["matches"][..., 1]]

# inlier_count drives candidate ranking
```

For each candidate, the system tests **3 FOVs** (70°, 90°, 110°) against the indexed heading to handle zoom mismatches.

### Stage 3 — Refinement

```python
# Heading refinement: ±45° at 15° steps, 3 FOVs, top-15 candidates
heading_offsets = range(-45, 46, 15)   # [-45, -30, -15, 0, 15, 30, 45]
fovs = [70, 90, 110]

# Spatial consensus: cluster matches into 50m cells
# Prefer clusters of multiple agreeing candidates over single high-inlier outliers

# Confidence scoring considers:
# - inlier count of best match
# - uniqueness ratio: best_inliers / runner_up_inliers
# - geographic clustering of top-N results
```

---

## Code Examples

### Extract a CosPlace descriptor from a local image

```python
import torch
from PIL import Image
from torchvision import transforms
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()
model.eval()

transform = transforms.Compose([
    transforms.Resize((512, 512)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])

img = Image.open("street_photo.jpg").convert("RGB")
tensor = transform(img).unsqueeze(0)   # (1, 3, 512, 512)

with torch.no_grad():
    descriptor = get_descriptor(model, tensor)   # (512,)

print(descriptor.shape)  # torch.Size([512])
```

### Search the compiled index manually

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lngs = meta["lngs"]

def haversine_km(lat1, lng1, lat2, lng2):
    R = 6371
    dlat = radians(lat2 - lat1)
    dlng = radians(lng2 - lng1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlng/2)**2
    return 2 * R * asin(sqrt(a))

def search(query_descriptor, center_lat, center_lng, radius_km, top_k=500):
    # Radius mask
    mask = np.array([
        haversine_km(center_lat, center_lng, la, lo) <= radius_km
        for la, lo in zip(lats, lngs)
    ])
    filtered_desc = descriptors[mask]
    filtered_idx  = np.where(mask)[0]

    # Cosine similarity (descriptors assumed L2-normalised)
    scores = filtered_desc @ query_descriptor
    top_local = np.argsort(scores)[::-1][:top_k]
    top_global = filtered_idx[top_local]

    return [
        {"lat": lats[i], "lng": lngs[i], "score": scores[top_local[j]]}
        for j, i in enumerate(top_global)
    ]

results = search(descriptor.numpy(), 48.8566, 2.3522, radius_km=2.0)
print(results[0])  # {'lat': 48.857..., 'lng': 2.351..., 'score': 0.94}
```

### Run LightGlue matching between two images

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher   = LightGlue(features="aliked").eval().to(device)

img0 = load_image("query.jpg").to(device)
img1 = load_image("candidate.jpg").to(device)

with torch.no_grad():
    f0 = extractor.extract(img0)
    f1 = extractor.extract(img1)
    result = rbd(matcher({"image0": f0, "image1": f1}))

matched_pairs = result["matches"]          # (M, 2) index pairs
scores        = result["matching_scores"]  # (M,) confidence per match
inlier_count  = matched_pairs.shape[0]

print(f"Matched keypoints: {inlier_count}")
```

### Build index from command line (large areas)

```bash
# build_index.py is the high-performance standalone builder
python build_index.py \
  --lat 48.8566 \
  --lng 2.3522 \
  --radius 5.0 \
  --resolution 300
```

---

## Common Patterns

### Multi-city index (single unified index)

```
# Index Paris
Create mode → lat=48.8566, lng=2.3522, radius=5

# Index London (appends to same index)
Create mode → lat=51.5074, lng=-0.1278, radius=5

# Search only Paris
Search mode → center=48.8566,2.3522, radius=5
# Search only London
Search mode → center=51.5074,-0.1278, radius=5
```

No city selection needed — coordinates + radius act as the filter.

### Choosing radius for search

- **Known city, unknown district**: 5–10 km
- **Known district**: 1–2 km
- **Unknown country (AI Coarse)**: AI estimates region → feeds as center
- **Conflict imagery with partial context**: start 5 km, tighten on result

### When to use Ultra Mode

| Symptom | Action |
|---------|--------|
| Confidence < 0.5, night photo | Enable Ultra Mode |
| Blurry / low-res query | Enable Ultra Mode |
| Top match looks right but score is low | Enable Ultra Mode |
| Standard mode returns wrong country | Use AI Coarse first to narrow region |

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # match exact Python version in venv
```

### `ModuleNotFoundError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### `ModuleNotFoundError: No module named 'kornia'` (Ultra Mode)
```bash
pip install kornia
```

### CUDA out of memory
- Reduce `max_num_keypoints` in extractor init (try 512 instead of 1024)
- Process fewer candidates per batch (modify `top_k` in search call)
- Fall back to MPS or CPU by unsetting CUDA device

### Indexing stalls / resumes from wrong point
- Indexing writes to `cosplace_parts/*.npz` incrementally
- Safe to `Ctrl+C` and re-run — existing chunks are skipped
- If index seems corrupt: delete `index/` (not `cosplace_parts/`) and re-run auto-build

### Low match quality on standard mode
1. Try Ultra Mode
2. Verify the indexed area actually covers the query location
3. Increase search radius
4. Try AI Coarse mode to get a better center estimate
5. Check that the index was built at resolution 300 (default)

### `GEMINI_API_KEY` not found
```bash
export GEMINI_API_KEY="your_key_here"   # Linux/macOS
set GEMINI_API_KEY=your_key_here        # Windows CMD
$env:GEMINI_API_KEY="your_key_here"     # PowerShell
```

---

## Models Reference

| Model | Role | Hardware preference |
|-------|------|---------------------|
| CosPlace (512-dim) | Global visual fingerprint + index search | Any |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching + RANSAC | Any (CUDA fastest) |
| LoFTR | Detector-free dense matching (Ultra) | CUDA / MPS |

---

## Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `GEMINI_API_KEY` | AI Coarse blind geolocation via Gemini | No (optional) |

No other secrets or API keys are needed for core functionality. Street View panoramas are fetched via the pipeline at search time.
```
