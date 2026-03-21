```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - use netryx to locate
  - index street view area
  - run netryx search
  - visual place recognition geolocation
  - locate image coordinates from photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls and indexes Street View panoramas, extracts 512-dimensional visual fingerprints using CosPlace, then verifies matches with local feature matching (ALIKED/DISK + LightGlue). No internet-presence required for the target location — it searches the physical world, not the web.

**Key capabilities:**
- Sub-50m accuracy on clear street photos
- Fully local — no geolocation API calls, no data sent to third parties
- Supports CUDA (NVIDIA), MPS (Apple Silicon), and CPU
- Optional Ultra Mode for degraded images (night, blur, low texture)
- Optional Gemini API integration for AI-assisted region guessing

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (deep feature matcher)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode
pip install kornia
```

**macOS tkinter fix** (if GUI appears blank):
```bash
brew install python-tk@3.11   # match your Python version
```

**Environment variables:**
```bash
# Optional — only needed for AI Coarse region-guessing mode
export GEMINI_API_KEY="your_key_from_aistudio_google_com"
```

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
├── index/
│   ├── cosplace_descriptors.npy   # Compiled 512-dim descriptor matrix
│   └── metadata.npz               # Coordinates, headings, panorama IDs
└── README.md
```

---

## Launching the GUI

```bash
python test_super.py
```

The GUI is the primary interface. It exposes all pipeline stages through a single window with mode selection (Create / Search), map visualization, and real-time scanning output.

---

## Core Workflow

### Step 1 — Index an Area (Create Mode)

Indexing crawls Street View panoramas in a geographic region and stores CosPlace fingerprints. This is a one-time step per area.

**GUI workflow:**
1. Select **Create** mode
2. Enter center lat/lon (e.g. `48.8566, 2.3522` for Paris)
3. Set radius in km (start with `0.5`–`1` for testing)
4. Set grid resolution (default `300` — do not change)
5. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

Indexing is **resumable** — interrupt and restart; it picks up from the last saved chunk in `cosplace_parts/`.

Multiple cities can share one index. Radius filtering at search time handles separation automatically.

---

### Step 2 — Search (Search Mode)

**GUI workflow:**
1. Select **Search** mode
2. Upload a street-level image (JPG/PNG)
3. Choose search method:
   - **Manual**: Enter approximate center lat/lon + radius
   - **AI Coarse**: Gemini analyzes image for visual clues → auto-suggests region (requires `GEMINI_API_KEY`)
4. Optionally enable **Ultra Mode** (slower, better for degraded images)
5. Click **Run Search** → **Start Full Search**
6. Result: GPS coordinates + confidence score displayed on map

---

## Pipeline Internals

### Stage 1 — Global Retrieval (CosPlace)

```python
# From cosplace_utils.py pattern — extract descriptor from an image
import torch
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()  # loads CosPlace ResNet-50, 512-dim output

# Returns a (512,) numpy array — the visual fingerprint
descriptor = get_descriptor(model, "path/to/query_image.jpg")

# Index search: cosine similarity against all stored descriptors
# descriptors matrix shape: (N, 512)
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)

query = descriptor / np.linalg.norm(descriptor)
db = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
similarities = db @ query                                   # cosine sim, shape (N,)

top_indices = np.argsort(similarities)[::-1][:500]         # top 500 candidates
```

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)

```python
# Platform detection — ALIKED on CUDA, DISK on MPS/CPU
import torch

device = (
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)

# LightGlue setup
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

if device == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device == "cuda" else "disk").eval().to(device)

# Extract keypoints from query image
query_img = load_image("query.jpg").to(device)
query_feats = extractor.extract(query_img)

# Extract keypoints from candidate panorama crop
cand_img = load_image("candidate_crop.jpg").to(device)
cand_feats = extractor.extract(cand_img)

# Match
matches_data = matcher({"image0": query_feats, "image1": cand_feats})
query_feats, cand_feats, matches_data = [
    rbd(x) for x in [query_feats, cand_feats, matches_data]
]

matches = matches_data["matches"]           # (M, 2) matched keypoint indices
inliers = matches_data["matching_scores0"]  # confidence scores

print(f"Matched keypoints: {len(matches)}")
```

### Stage 3 — RANSAC Geometric Filter

```python
import cv2
import numpy as np

def count_ransac_inliers(kp0, kp1, matches):
    """Returns number of geometrically consistent match inliers."""
    if len(matches) < 8:
        return 0

    pts0 = np.float32([kp0[m[0]] for m in matches])
    pts1 = np.float32([kp1[m[1]] for m in matches])

    _, mask = cv2.findFundamentalMat(
        pts0, pts1,
        cv2.FM_RANSAC,
        ransacReprojThreshold=3.0,
        confidence=0.999
    )
    return int(mask.sum()) if mask is not None else 0
```

---

## Multi-FOV Cropping Pattern

Netryx tests three fields of view per candidate to handle zoom mismatches:

```python
# FOVs tested per candidate heading
FOV_VARIANTS = [70, 90, 110]   # degrees

# Heading refinement offsets for top-15 candidates
HEADING_OFFSETS = list(range(-45, 46, 15))  # -45 to +45 in 15° steps

# Best match = max inliers across all (FOV, heading) combinations
best_inliers = 0
best_config = None

for fov in FOV_VARIANTS:
    for offset in HEADING_OFFSETS:
        heading = base_heading + offset
        crop = get_panorama_crop(panoid, heading, fov)
        inliers = run_verification(query_feats, crop)
        if inliers > best_inliers:
            best_inliers = inliers
            best_config = (heading, fov, inliers)
```

---

## Ultra Mode Components

Enable Ultra Mode for night photos, motion blur, or low-texture scenes:

```python
# LoFTR — detector-free dense matching (requires kornia)
import kornia.feature as KF
import torch

loftr = KF.LoFTR(pretrained="outdoor").eval().to(device)

def loftr_match(img0_path, img1_path):
    img0 = load_as_gray_tensor(img0_path).to(device)   # (1,1,H,W)
    img1 = load_as_gray_tensor(img1_path).to(device)

    with torch.no_grad():
        out = loftr({"image0": img0, "image1": img1})

    kp0 = out["keypoints0"].cpu().numpy()
    kp1 = out["keypoints1"].cpu().numpy()
    conf = out["confidence"].cpu().numpy()

    # Filter by confidence threshold
    mask = conf > 0.5
    return kp0[mask], kp1[mask], conf[mask]
```

**Ultra Mode extras:**
- **Descriptor hopping**: Re-searches index using the matched panorama's clean descriptor if initial match has <50 inliers
- **Neighborhood expansion**: Searches all panoramas within 100m of the best match location

---

## Spatial Consensus Scoring

```python
from collections import defaultdict
import math

def haversine_m(lat1, lon1, lat2, lon2):
    """Distance in meters between two GPS points."""
    R = 6371000
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlambda = math.radians(lon2 - lon1)
    a = math.sin(dphi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(dlambda/2)**2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

def cluster_candidates(candidates, cell_size_m=50):
    """
    Group candidates into 50m spatial cells.
    Returns clusters sorted by total inliers descending.
    candidates: list of (lat, lon, inliers)
    """
    clusters = defaultdict(list)

    for lat, lon, inliers in candidates:
        # Quantize to grid cell
        cell_lat = round(lat * 111000 / cell_size_m) * cell_size_m / 111000
        cell_lon = round(lon * 111000 * math.cos(math.radians(lat)) / cell_size_m) \
                   * cell_size_m / (111000 * math.cos(math.radians(lat)))
        clusters[(round(cell_lat, 6), round(cell_lon, 6))].append((lat, lon, inliers))

    # Score each cluster by total inlier count
    scored = []
    for cell, members in clusters.items():
        total_inliers = sum(m[2] for m in members)
        best = max(members, key=lambda x: x[2])
        scored.append((total_inliers, len(members), best[0], best[1]))

    return sorted(scored, reverse=True)
```

---

## Building a Large Index (CLI)

For areas >5km radius, use the standalone high-performance builder instead of the GUI:

```bash
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --grid-resolution 300
```

This writes chunks to `cosplace_parts/` and auto-compiles to `index/` on completion. Resumable on restart.

---

## Confidence Score Interpretation

| Inliers | Confidence | Interpretation |
|---------|-----------|----------------|
| >150 | Very High | Almost certainly correct |
| 80–150 | High | Strong match |
| 40–80 | Medium | Likely correct, verify manually |
| 15–40 | Low | Weak match, consider Ultra Mode |
| <15 | Very Low | No reliable match found |

---

## Common Patterns

### Pattern: Programmatic search without GUI

```python
# Pseudo-code for headless search — adapt from test_super.py internals
import numpy as np
from cosplace_utils import get_cosplace_model, get_descriptor

# 1. Load model and index
model = get_cosplace_model()
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]
headings = meta["headings"]
panoids = meta["panoids"]

# 2. Extract query descriptor
query_desc = get_descriptor(model, "my_photo.jpg")
query_flipped_desc = get_descriptor(model, "my_photo_flipped.jpg")

# 3. Radius filter (haversine)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 1.0

def haversine_km(lat1, lon1, lat2, lon2):
    import math
    R = 6371
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi, dlambda = math.radians(lat2-lat1), math.radians(lon2-lon1)
    a = math.sin(dphi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(dlambda/2)**2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
])

filtered_descs = descriptors[mask]
filtered_lats = lats[mask]
filtered_lons = lons[mask]

# 4. Cosine similarity
q = query_desc / np.linalg.norm(query_desc)
db = filtered_descs / np.linalg.norm(filtered_descs, axis=1, keepdims=True)
sims = db @ q
top_idx = np.argsort(sims)[::-1][:500]

print(f"Top candidate: lat={filtered_lats[top_idx[0]]:.6f}, lon={filtered_lons[top_idx[0]]:.6f}")
```

### Pattern: Check GPU availability

```python
import torch

device = (
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)
print(f"Using device: {device}")
# CUDA → ALIKED (1024 kp)
# MPS  → DISK (768 kp)
# CPU  → DISK (768 kp, slow)
```

### Pattern: Flip augmentation for retrieval

```python
from PIL import Image
import numpy as np

def get_augmented_descriptor(model, image_path):
    """Get max-pooled descriptor from original + horizontally flipped."""
    from cosplace_utils import get_descriptor

    desc_orig = get_descriptor(model, image_path)

    img = Image.open(image_path)
    img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
    img_flipped.save("/tmp/_flipped_query.jpg")
    desc_flip = get_descriptor(model, "/tmp/_flipped_query.jpg")

    # Element-wise max (catches reversed perspectives)
    return np.maximum(desc_orig, desc_flip)
```

---

## Troubleshooting

**GUI appears blank on macOS**
```bash
brew install python-tk@3.11   # use your actual Python version
```

**`No module named 'lightglue'`**
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

**LoFTR / Ultra Mode not available**
```bash
pip install kornia
```

**MPS errors on Apple Silicon**
```bash
# Some ops fall back to CPU on MPS — this is expected
# Set if you encounter hard failures:
export PYTORCH_ENABLE_MPS_FALLBACK=1
```

**Indexing stalls / no panoramas found**
- Verify the coordinates are in an area with Street View coverage
- Check internet connection (indexing downloads panoramas)
- The index builder resumes automatically — restart `python test_super.py` or `python build_index.py`

**Low inlier counts (<15) on clear images**
- Expand search radius — the correct location may not be indexed yet
- Try enabling Ultra Mode (adds LoFTR + neighborhood expansion)
- Verify the indexed area overlaps with the query image location

**CUDA out of memory**
```python
# Reduce keypoints in extractor
extractor = ALIKED(max_num_keypoints=512).eval().to("cuda")  # default 1024
```

**Index search returns wrong city**
- Always specify a tight radius — the index is global, radius is the only geographic filter
- Example: searching Paris at 1km radius won't return London results

---

## Hardware Recommendations

| Setup | Expected Search Time (300 candidates) |
|-------|--------------------------------------|
| NVIDIA RTX 3080+ (CUDA) | 1–2 minutes |
| Apple M2 Max/M3 (MPS) | 2–3 minutes |
| Apple M1 (MPS) | 3–5 minutes |
| CPU only | 15–30 minutes |

Ultra Mode adds ~2× time overhead on top of standard search time.
```
