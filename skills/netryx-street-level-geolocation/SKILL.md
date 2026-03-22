```markdown
---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photograph to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - run netryx geolocation
  - identify location from photo
  - visual place recognition pipeline
  - where was this photo taken
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted street-level geolocation engine that identifies precise GPS coordinates from any street photograph. It crawls street-view panoramas into a searchable index, then matches query images using a three-stage pipeline: CosPlace global retrieval → ALIKED/DISK + LightGlue local verification → heading refinement and spatial consensus clustering. Sub-50m accuracy, no landmarks required, runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # required
pip install kornia                                       # optional: Ultra Mode (LoFTR)
```

### Optional: Gemini AI coarse localisation

```bash
export GEMINI_API_KEY="your_key_from_aistudio_google_com"
```

### macOS blank GUI fix

```bash
brew install python-tk@3.11   # match your Python version
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM  | 4 GB    | 8 GB+       |
| RAM       | 8 GB    | 16 GB+      |
| Storage   | 10 GB   | 50 GB+      |
| Python    | 3.9+    | 3.10+       |

GPU backends: NVIDIA → CUDA (ALIKED), Apple Silicon → MPS (DISK), fallback → CPU.

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface for both indexing and searching.

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, and search
├── cosplace_utils.py      # CosPlace model loading and descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panoid IDs
```

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. Netryx crawls street-view panoramas, extracts CosPlace fingerprints, and writes them incrementally to `cosplace_parts/`.

**In the GUI:**
1. Select **Create** mode
2. Enter centre latitude/longitude
3. Set radius (km) and grid resolution (default 300)
4. Click **Create Index**

**Indexing time estimates:**

| Radius | Panoramas | Time (M2 Max) | Index size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2 000    | 1–2 h         | ~250 MB    |
| 5 km   | ~30 000   | 8–12 h        | ~3 GB      |
| 10 km  | ~100 000  | 24–48 h       | ~7 GB      |

Indexing is resumable — re-run after interruption and it picks up where it left off.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual** — enter approximate centre + radius if known
   - **AI Coarse** — Gemini analyses visual clues (signs, architecture) to guess region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score displayed on map

### Ultra Mode

Tick **Ultra Mode** for difficult images (night, blur, low texture). Adds:
- **LoFTR** dense matching (detector-free, handles blur)
- **Descriptor hopping** — re-searches index using the matched panorama's clean descriptor
- **Neighbourhood expansion** — checks all panoramas within 100 m of the best match

---

## Pipeline Deep-Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py usage pattern
from cosplace_utils import load_cosplace_model, extract_descriptor
import torch
from PIL import Image

model = load_cosplace_model(device="cuda")   # or "mps" / "cpu"

img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, img)           # shape: (512,)
flipped_desc = extract_descriptor(model, img, flip=True)

# Cosine similarity search against the index
import numpy as np

index_descs = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta        = np.load("index/metadata.npz", allow_pickle=True)
lats        = meta["lats"]
lons        = meta["lons"]

sims = index_descs @ descriptor           # dot product == cosine sim (unit vectors)
top_k = np.argsort(sims)[::-1][:500]     # top 500 candidates
```

### Stage 2 — Local Geometric Verification

```python
# Simplified version of what test_super.py does per candidate
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Feature extractor (ALIKED on CUDA, DISK on MPS/CPU)
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

def match_pair(query_path: str, candidate_path: str) -> int:
    """Return number of RANSAC-verified inliers between two images."""
    img0 = load_image(query_path).to(device)
    img1 = load_image(candidate_path).to(device)

    feats0 = extractor.extract(img0)
    feats1 = extractor.extract(img1)

    matches01 = matcher({"image0": feats0, "image1": feats1})
    feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

    kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
    kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]

    if len(kpts0) < 4:
        return 0

    import cv2
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, 3.0)
    inliers = int(mask.sum()) if mask is not None else 0
    return inliers

inliers = match_pair("query.jpg", "candidate_crop.jpg")
print(f"Verified inliers: {inliers}")   # >30 inliers = strong match
```

### Stage 3 — Heading Refinement & Spatial Consensus

```python
import numpy as np
from math import radians, sin, cos, sqrt, atan2

def haversine_m(lat1, lon1, lat2, lon2) -> float:
    """Distance in metres between two GPS points."""
    R = 6_371_000
    φ1, φ2 = radians(lat1), radians(lat2)
    dφ = radians(lat2 - lat1)
    dλ = radians(lon2 - lon1)
    a = sin(dφ/2)**2 + cos(φ1)*cos(φ2)*sin(dλ/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1-a))

def spatial_consensus(candidates: list[dict], cell_m: float = 50.0) -> dict:
    """
    Cluster candidates into 50 m cells.
    Return the candidate in the largest cluster (most spatial support).
    candidates: list of {"lat": float, "lon": float, "inliers": int}
    """
    clusters: dict[int, list[dict]] = {}
    for i, c in enumerate(candidates):
        placed = False
        for key, members in clusters.items():
            ref = members[0]
            if haversine_m(c["lat"], c["lon"], ref["lat"], ref["lon"]) <= cell_m:
                members.append(c)
                placed = True
                break
        if not placed:
            clusters[i] = [c]

    best_cluster = max(clusters.values(), key=lambda m: (len(m), sum(x["inliers"] for x in m)))
    return max(best_cluster, key=lambda x: x["inliers"])
```

---

## Multi-Index Strategy

All city indexes are merged into a single `index/` directory. Coordinate + radius filtering isolates the right subset at query time — no city selection needed.

```python
# Radius filter applied before cosine search
def radius_filter(lats, lons, centre_lat, centre_lon, radius_km):
    mask = np.array([
        haversine_m(centre_lat, centre_lon, lat, lon) <= radius_km * 1000
        for lat, lon in zip(lats, lons)
    ], dtype=bool)
    return np.where(mask)[0]

# Example: search only within Paris 5 km radius
indices_in_radius = radius_filter(lats, lons, 48.8566, 2.3522, radius_km=5.0)
sims_filtered     = sims[indices_in_radius]
top_k             = indices_in_radius[np.argsort(sims_filtered)[::-1][:500]]
```

---

## Standalone Index Builder (Large Datasets)

For large areas, use `build_index.py` directly instead of the GUI:

```bash
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --resolution 300
```

This writes incremental chunks to `cosplace_parts/` and auto-compiles the final `index/cosplace_descriptors.npy` and `index/metadata.npz`.

---

## Ultra Mode with LoFTR

```python
import kornia.feature as KF
import torch

def loftr_match(img0_tensor: torch.Tensor, img1_tensor: torch.Tensor, device) -> int:
    """Dense detector-free matching for blurry/dark images."""
    matcher = KF.LoFTR(pretrained="outdoor").eval().to(device)

    # LoFTR expects grayscale tensors: (1, 1, H, W)
    def to_gray(t):
        return t.mean(dim=1, keepdim=True)

    data = {
        "image0": to_gray(img0_tensor).to(device),
        "image1": to_gray(img1_tensor).to(device),
    }
    with torch.no_grad():
        correspondences = matcher(data)

    confidence = correspondences["confidence"].cpu().numpy()
    # Count high-confidence matches as inliers
    inliers = int((confidence > 0.9).sum())
    return inliers
```

---

## Common Patterns

### Confidence Scoring

```python
def confidence_score(best_inliers: int, second_inliers: int, cluster_size: int) -> float:
    """
    Composite confidence: uniqueness ratio × cluster support bonus.
    Returns value in [0, 1].
    """
    if second_inliers == 0:
        uniqueness = 1.0
    else:
        uniqueness = min(best_inliers / second_inliers, 2.0) / 2.0   # cap at 2×

    cluster_bonus = min(cluster_size / 5, 1.0)   # saturates at 5 clustered matches
    return round(0.7 * uniqueness + 0.3 * cluster_bonus, 3)
```

### Batch Descriptor Extraction

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from pathlib import Path
import numpy as np
from PIL import Image

model = load_cosplace_model(device="cuda")
image_dir = Path("panorama_crops/")

descriptors = []
for img_path in sorted(image_dir.glob("*.jpg")):
    img = Image.open(img_path).convert("RGB")
    desc = extract_descriptor(model, img)
    descriptors.append(desc)

descriptors = np.stack(descriptors)          # (N, 512)
np.save("my_partial_index.npy", descriptors)
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| GUI appears blank on macOS | Bundled tkinter bug | `brew install python-tk@3.11` |
| `ModuleNotFoundError: lightglue` | LightGlue not installed from git | `pip install git+https://github.com/cvg/LightGlue.git` |
| CUDA out of memory | Too many keypoints | Lower `max_num_keypoints` to 512 |
| Low inliers on every candidate | Wrong search area | Verify centre lat/lon and increase radius |
| LoFTR import error | kornia not installed | `pip install kornia` |
| Indexing stalls | API rate limit | Reduce grid resolution; indexing auto-resumes |
| `index/` files not found | Index not compiled yet | Re-run index creation to trigger auto-build |
| Weak match (<30 inliers) | Degraded query image | Enable Ultra Mode |

### Device Detection

```python
import torch

def get_device() -> torch.device:
    if torch.cuda.is_available():
        return torch.device("cuda")
    if torch.backends.mps.is_available():
        return torch.device("mps")
    return torch.device("cpu")

device = get_device()
print(f"Using: {device}")   # cuda / mps / cpu
```

### Verify Index Integrity

```python
import numpy as np

descs = np.load("index/cosplace_descriptors.npy")
meta  = np.load("index/metadata.npz", allow_pickle=True)

assert descs.shape[1] == 512, "Unexpected descriptor dimension"
assert len(meta["lats"]) == descs.shape[0], "Descriptor/metadata count mismatch"
print(f"Index OK: {descs.shape[0]:,} panoramas indexed")
```

---

## Key Parameters Reference

| Parameter | Default | Effect |
|-----------|---------|--------|
| Grid resolution | 300 | Density of panorama sampling — do not change |
| Top-k candidates | 500 | More = higher recall, slower Stage 2 |
| ALIKED keypoints | 1024 | CUDA; reduce to 512 for low VRAM |
| DISK keypoints | 768 | MPS/CPU |
| RANSAC threshold | 3.0 px | Geometric consistency tolerance |
| Heading offsets | ±45° at 15° steps | Refinement sweep range |
| Refinement candidates | Top 15 | Heading refinement scope |
| Cluster cell size | 50 m | Spatial consensus granularity |
| Neighbourhood expansion | 100 m | Ultra Mode only |
| LoFTR confidence threshold | 0.9 | Ultra Mode inlier cutoff |
```
