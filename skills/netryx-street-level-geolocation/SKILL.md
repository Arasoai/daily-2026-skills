```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for Netryx, a local-first open-source street-level geolocation engine using CosPlace, ALIKED/DISK, and LightGlue to identify GPS coordinates from street photos.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - use Netryx to locate an image
  - run netryx geolocation pipeline
  - build a street view index
  - match street photo to coordinates
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls street-view panoramas, indexes them with CosPlace visual fingerprints, then verifies candidates using ALIKED/DISK keypoint extraction and LightGlue deep feature matching — all running on your own hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # Required
pip install kornia                                        # Optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API key (AI Coarse mode)

```bash
export GEMINI_API_KEY="your_key_here"
```

### Platform requirements

| Platform | GPU Backend | Min VRAM |
|----------|------------|----------|
| NVIDIA   | CUDA       | 4 GB     |
| Apple M1+ | MPS       | shared   |
| Any      | CPU        | —        |

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### 1. Create an Index

Index an area before searching. The index stores 512-dim CosPlace descriptors for every crawled panorama.

**In GUI:**
1. Select **Create** mode
2. Enter center lat/lon
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time / storage estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hr        | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hr       | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hr      | ~7 GB      |

Indexing is **resumable** — interrupted runs pick up where they left off.

### 2. Search

**In GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: provide center lat/lon + radius
   - **AI Coarse**: Gemini infers region from visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (searchable)
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

```python
# cosplace_utils.py usage pattern
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model(device="cuda")  # or "mps" / "cpu"

# Extract 512-dim fingerprint from a query image path
descriptor = get_descriptor(model, "query.jpg", device="cuda")

# Also extract flipped descriptor to catch reversed perspectives
import PIL.Image, torchvision.transforms as T, torch
img = PIL.Image.open("query.jpg")
flipped = img.transpose(PIL.Image.FLIP_LEFT_RIGHT)
descriptor_flipped = get_descriptor(model, flipped, device="cuda")
```

Index search is a single cosine-similarity matrix multiply filtered by haversine radius:

```python
import numpy as np

def cosine_search(query_desc, index_descs, top_k=1000):
    """Returns indices of top_k most similar panoramas."""
    # index_descs: (N, 512) float32, query_desc: (512,)
    sims = index_descs @ query_desc  # cosine sim (descriptors are L2-normalized)
    return np.argsort(sims)[::-1][:top_k]

def haversine_filter(candidates_meta, center_lat, center_lon, radius_km):
    """Filter candidates to those within radius_km of center."""
    from math import radians, sin, cos, sqrt, atan2
    R = 6371.0
    results = []
    for c in candidates_meta:
        dlat = radians(c['lat'] - center_lat)
        dlon = radians(c['lon'] - center_lon)
        a = sin(dlat/2)**2 + cos(radians(center_lat)) * cos(radians(c['lat'])) * sin(dlon/2)**2
        if R * 2 * atan2(sqrt(a), sqrt(1-a)) <= radius_km:
            results.append(c)
    return results
```

### Stage 2 — Local Geometric Verification (LightGlue)

```python
# ALIKED + LightGlue matching (CUDA path)
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher   = LightGlue(features="aliked").eval().to(device)

query_img     = load_image("query.jpg").to(device)
candidate_img = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(query_img)
feats1 = extractor.extract(candidate_img)

matches_out = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches_out = rbd(feats0), rbd(feats1), rbd(matches_out)

matches   = matches_out["matches"]           # (M, 2) matched keypoint indices
kpts0     = feats0["keypoints"][matches[:, 0]]
kpts1     = feats1["keypoints"][matches[:, 1]]

print(f"Raw matches: {len(matches)}")
```

```python
# DISK + LightGlue (MPS / CPU path)
from lightglue import DISK
extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher   = LightGlue(features="disk").eval().to(device)
# Same matching interface as ALIKED above
```

### Stage 2 — RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kpts0_np, kpts1_np, threshold=4.0):
    """
    kpts0_np, kpts1_np: (M, 2) matched keypoint coordinates as numpy arrays.
    Returns number of RANSAC inliers.
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
```

### Stage 3 — Heading Refinement

```python
# For top-N candidates, test ±45° heading offsets at 15° steps and 3 FOVs
HEADING_OFFSETS = list(range(-45, 46, 15))   # [-45, -30, -15, 0, 15, 30, 45]
FOVS           = [70, 90, 110]

best_inliers = 0
best_heading = base_heading

for offset in HEADING_OFFSETS:
    test_heading = (base_heading + offset) % 360
    for fov in FOVS:
        crop = get_streetview_crop(panoid, test_heading, fov)
        inliers = ransac_verify(*match_keypoints(query_feats, crop))
        if inliers > best_inliers:
            best_inliers = inliers
            best_heading = test_heading
```

### Ultra Mode (LoFTR dense matching)

```python
import kornia.feature as KF
import torch

loftr = KF.LoFTR(pretrained="outdoor").eval().to(device)

def loftr_match(img0_gray, img1_gray):
    """
    img0_gray, img1_gray: (1, 1, H, W) float tensors in [0,1].
    Returns matched keypoint coordinates.
    """
    input_dict = {"image0": img0_gray, "image1": img1_gray}
    with torch.no_grad():
        correspondences = loftr(input_dict)
    kpts0 = correspondences["keypoints0"].cpu().numpy()
    kpts1 = correspondences["keypoints1"].cpu().numpy()
    return kpts0, kpts1
```

---

## Index Management

### Build Index (standalone, large datasets)

```bash
python build_index.py
```

This consolidates all `cosplace_parts/*.npz` chunks into the searchable `index/` directory.

### Index file format

```python
import numpy as np

# Load the compiled index manually
descs = np.load("index/cosplace_descriptors.npy")   # shape: (N, 512), float32
meta  = np.load("index/metadata.npz", allow_pickle=True)

lats     = meta["lats"]      # (N,) float64
lons     = meta["lons"]      # (N,) float64
headings = meta["headings"]  # (N,) float64
panoids  = meta["panoids"]   # (N,) str — Street View panorama IDs
```

### Multi-city indexing

The index is unified — all cities live in one file. Radius filtering at query time handles isolation:

```python
# Index Paris and London into the same index — no conflict
# At search time, just change center coordinates:
paris_results  = search(center=(48.8566, 2.3522),  radius_km=5)
london_results = search(center=(51.5074, -0.1278), radius_km=5)
```

---

## Confidence Scoring

```python
def compute_confidence(top_matches):
    """
    Returns a confidence string based on:
    - Spatial clustering of top matches (50m grid cells)
    - Uniqueness ratio: best inliers vs runner-up at a different location
    """
    from collections import Counter

    # Cluster into 50m cells
    def cell(lat, lon):
        return (round(lat * 1000), round(lon * 1000))  # ~111m per unit → tune as needed

    cells = Counter(cell(m['lat'], m['lon']) for m in top_matches)
    dominant_cell, dominant_count = cells.most_common(1)[0]

    best    = top_matches[0]['inliers']
    # Runner-up at a different location
    runner  = next((m['inliers'] for m in top_matches[1:]
                    if cell(m['lat'], m['lon']) != dominant_cell), 0)

    uniqueness = best / max(runner, 1)

    if uniqueness > 3 and dominant_count >= 3:
        return "HIGH"
    elif uniqueness > 1.5:
        return "MEDIUM"
    else:
        return "LOW"
```

---

## Common Patterns

### Programmatic search (no GUI)

```python
# Pseudocode mirroring test_super.py internals
from cosplace_utils import get_cosplace_model, get_descriptor
import numpy as np

# 1. Load index
descs = np.load("index/cosplace_descriptors.npy")
meta  = np.load("index/metadata.npz", allow_pickle=True)

# 2. Extract query descriptor
device = "cuda"   # or "mps" / "cpu"
model  = get_cosplace_model(device=device)
q_desc = get_descriptor(model, "my_photo.jpg", device=device)
q_desc = q_desc / np.linalg.norm(q_desc)   # L2 normalize

# 3. Radius filter
center_lat, center_lon, radius_km = 48.8566, 2.3522, 2.0
lats, lons = meta["lats"], meta["lons"]
dlat = np.radians(lats - center_lat)
dlon = np.radians(lons - center_lon)
a    = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
dist_km = 6371.0 * 2 * np.arcsin(np.sqrt(a))
mask = dist_km <= radius_km

# 4. Cosine retrieval
sims = descs[mask] @ q_desc
top_idx = np.argsort(sims)[::-1][:500]

# 5. Proceed to Stage 2 with top_idx candidates...
```

### Descriptor hopping (Ultra Mode pattern)

```python
# If best match is weak, re-search using the matched panorama's own descriptor
if best_inliers < 50:
    matched_panoid = top_candidates[0]['panoid']
    matched_crop   = download_panorama_crop(matched_panoid, heading=0, fov=90)
    # matched_crop is a clean, high-quality image — better query than degraded input
    rehop_desc  = get_descriptor(model, matched_crop, device=device)
    rehop_desc  = rehop_desc / np.linalg.norm(rehop_desc)
    # Re-run retrieval with rehop_desc — often finds the exact correct panorama
    new_sims    = descs[mask] @ rehop_desc
    new_top_idx = np.argsort(new_sims)[::-1][:500]
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| GUI renders blank | macOS tkinter bug | `brew install python-tk@3.11` |
| `ImportError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| CUDA OOM | Too many keypoints | Reduce `max_num_keypoints` (ALIKED: 512, DISK: 512) |
| Low inliers everywhere | Wrong search area | Increase radius or recheck center coordinates |
| Indexing stalls | Rate-limited by Street View API | Reduce concurrency or add sleep between requests |
| LoFTR not available | kornia not installed | `pip install kornia` |
| MPS feature extractor wrong | ALIKED unsupported on MPS | Pipeline auto-selects DISK on MPS — expected behavior |
| Confidence always LOW | Small index / sparse coverage | Increase grid resolution or index radius |

### Verify GPU detection

```python
import torch
print("CUDA:", torch.cuda.is_available())
print("MPS: ", torch.backends.mps.is_available())
device = (
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)
print("Using:", device)
```

### Check index integrity

```python
import numpy as np
descs = np.load("index/cosplace_descriptors.npy")
meta  = np.load("index/metadata.npz", allow_pickle=True)
print(f"Descriptors: {descs.shape}")          # (N, 512)
print(f"Metadata entries: {len(meta['lats'])}")
assert descs.shape[0] == len(meta['lats']), "Index mismatch — rebuild with build_index.py"
```

---

## Models Reference

| Model | Role | Backend |
|-------|------|---------|
| CosPlace | Global 512-dim descriptor (retrieval) | CUDA / MPS / CPU |
| ALIKED | Local keypoint extraction | CUDA only |
| DISK | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | CUDA / MPS / CPU |
| LoFTR | Dense detector-free matching (Ultra) | CUDA / MPS |
```
