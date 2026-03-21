```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace + ALIKED/DISK + LightGlue to identify GPS coordinates from any street photo
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - identify location from street view photo
  - netryx geolocation
  - reverse image geolocation
  - match street photo to coordinates
  - osint geolocation from photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, indexes them as 512-dim CosPlace embeddings, then matches a query image through a three-stage pipeline: global retrieval → local geometric verification → spatial refinement. Sub-50m accuracy. No landmarks required. Runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue feature matcher
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Platform GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | Uses ALIKED (1024 keypoints) — fastest |
| Apple Silicon (M1+) | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

### Optional: Gemini API for AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"
```

AI Coarse mode lets Gemini analyze visual clues (signs, architecture, vegetation) to estimate a region when you have zero prior knowledge of where the image is from. Manual coordinate entry works better in practice.

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11` (match your Python version). The system Python tkinter has rendering bugs on recent macOS.

---

## Core Workflow

### 1. Create an Index

Index an area before searching. This crawls street-view panoramas and extracts CosPlace fingerprints.

**In the GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set search radius (km)
4. Set grid resolution (default: 300 — don't change this)
5. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

**Index size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Disk |
|--------|-----------|---------------|------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

**Index storage layout:**
```
cosplace_parts/          # Raw embedding chunks (.npz), written during indexing
index/
├── cosplace_descriptors.npy   # All 512-dim descriptors (matrix)
└── metadata.npz               # lat/lon, headings, panorama IDs
```

Multiple cities can share one index — radius filtering at search time restricts results geographically. No city selection needed.

### 2. Search

**In the GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Enter approximate center coordinates + radius
   - **AI Coarse**: Gemini estimates region from visual clues
4. Click **Run Search** → **Start Full Search**
5. Result appears on the map with GPS coordinates + confidence score

### 3. Ultra Mode

Enable the **Ultra Mode** checkbox for:
- Night photos
- Blurry or low-resolution images
- Low-texture scenes (fog, rain, featureless walls)

Ultra Mode adds:
- **LoFTR** dense matching (detector-free, handles blur)
- **Descriptor hopping** (re-searches index from the matched panorama's clean descriptor)
- **Neighborhood expansion** (searches all panoramas within 100m of best match)

Ultra Mode is 2–3× slower but catches matches the standard pipeline misses.

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace → 512-dim descriptor
    ├── Flipped image → 512-dim descriptor (catches reversed perspectives)
    │
    ▼
Index Search: cosine similarity × radius filter (haversine)
    │
    └── Top 500–1000 candidates (< 1 second, single matrix multiply)
    │
    ▼
For each candidate:
    ├── Download panorama (8 tiles, stitched)
    ├── Crop at indexed heading angle
    ├── Multi-FOV crops: 70°, 90°, 110° (handles zoom mismatch)
    ├── ALIKED/DISK → keypoints + descriptors
    ├── LightGlue → feature matches
    └── RANSAC → geometric inliers
    │
    ▼
Heading Refinement (top 15 candidates)
    ├── Test ±45° offsets at 15° steps × 3 FOVs
    │
    ▼
Spatial Consensus: cluster matches into 50m cells
    │
    ▼
Confidence Scoring: clustering strength + uniqueness ratio
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

**Stage timings (M2 Max, 500 candidates):**
- Stage 1 (retrieval): < 1 second
- Stage 2 (verification): 2–5 minutes
- Stage 3 (refinement): included in Stage 2

---

## Models Reference

| Model | Role | Source |
|-------|------|--------|
| CosPlace | Global visual place recognition (512-dim descriptor) | [github.com/gmberton/cosplace](https://github.com/gmberton/cosplace) |
| ALIKED | Local keypoint extraction — CUDA | [github.com/naver/alike](https://github.com/naver/alike) |
| DISK | Local keypoint extraction — MPS/CPU | [github.com/cvlab-epfl/disk](https://github.com/cvlab-epfl/disk) |
| LightGlue | Deep feature matching | [github.com/cvg/LightGlue](https://github.com/cvg/LightGlue) |
| LoFTR | Detector-free dense matching (Ultra Mode) | via `kornia` |

CosPlace and ALIKED/DISK weights download automatically on first use.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks written during indexing
└── index/
    ├── cosplace_descriptors.npy
    └── metadata.npz
```

---

## Code Examples

### Extract a CosPlace Descriptor from an Image

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first call)
model = load_cosplace_model()
device = torch.device("cuda" if torch.cuda.is_available() else
                       "mps" if torch.backends.mps.is_available() else "cpu")
model = model.to(device)

# Extract 512-dim descriptor
img = Image.open("street_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)
# descriptor.shape == (512,)
```

### Search the Index Manually

```python
import numpy as np

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]      # (N,)
lons = meta["lons"]      # (N,)
panoids = meta["panoids"]

# Cosine similarity search
query = descriptor / np.linalg.norm(descriptor)
index_norm = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = index_norm @ query                               # (N,)

# Haversine radius filter (example: 5km around Paris)
CENTER_LAT, CENTER_LON = 48.8566, 2.3522
RADIUS_KM = 5.0

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

distances = haversine_km(CENTER_LAT, CENTER_LON, lats, lons)
in_radius = distances < RADIUS_KM

# Top 500 candidates within radius
scores[~in_radius] = -1
top_idx = np.argsort(scores)[::-1][:500]

for i in top_idx[:5]:
    print(f"  panoid={panoids[i]}  lat={lats[i]:.6f}  lon={lons[i]:.6f}  score={scores[i]:.4f}")
```

### LightGlue Feature Matching (CUDA)

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda")

extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

img0 = load_image("query.jpg").to(device)
img1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(img0)
feats1 = extractor.extract(img1)

matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(kpts0)}")
```

### LightGlue Feature Matching (MPS / Apple Silicon)

```python
import torch
from lightglue import LightGlue, DISK
from lightglue.utils import load_image, rbd

device = torch.device("mps")

# DISK instead of ALIKED on MPS
extractor = DISK(max_num_keypoints=768).eval().to(device)
matcher = LightGlue(features="disk").eval().to(device)

img0 = load_image("query.jpg").to(device)
img1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(img0)
feats1 = extractor.extract(img1)

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

def count_ransac_inliers(kpts0: np.ndarray, kpts1: np.ndarray, threshold: float = 4.0) -> int:
    """Returns number of geometrically consistent matches via RANSAC homography."""
    if len(kpts0) < 4:
        return 0
    _, mask = cv2.findHomography(
        kpts0, kpts1,
        cv2.RANSAC,
        ransacReprojThreshold=threshold
    )
    if mask is None:
        return 0
    return int(mask.sum())

# Usage after LightGlue matching:
inliers = count_ransac_inliers(
    kpts0.cpu().numpy(),
    kpts1.cpu().numpy()
)
print(f"RANSAC inliers: {inliers}")
# > 30 inliers = strong match; > 80 = near-certain match
```

### Ultra Mode: LoFTR Dense Matching

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Load LoFTR (outdoor weights)
matcher = kornia.feature.LoFTR(pretrained="outdoor").eval().to(device)

def to_gray_tensor(img_path: str, size=(640, 480)) -> torch.Tensor:
    img = cv2.imread(img_path, cv2.IMREAD_GRAYSCALE)
    img = cv2.resize(img, size)
    return torch.from_numpy(img).float()[None, None] / 255.0

img0 = to_gray_tensor("query.jpg").to(device)
img1 = to_gray_tensor("candidate.jpg").to(device)

with torch.no_grad():
    result = matcher({"image0": img0, "image1": img1})

kpts0 = result["keypoints0"].cpu().numpy()
kpts1 = result["keypoints1"].cpu().numpy()
conf  = result["confidence"].cpu().numpy()

# Filter by confidence
high_conf = conf > 0.5
print(f"High-confidence LoFTR matches: {high_conf.sum()}")
```

### Build Index from CLI (Large Datasets)

For large areas, use the standalone high-performance builder instead of the GUI:

```bash
python build_index.py
```

The builder reads `cosplace_parts/*.npz` and compiles them into `index/cosplace_descriptors.npy` + `index/metadata.npz`. Run after indexing completes or after resuming an interrupted session.

---

## Configuration Patterns

### Choosing Search Radius

```python
# Conservative (known city): 1–3 km
center = (48.8566, 2.3522)   # Paris city center
radius_km = 2.0

# Regional (known country, unknown city): 50–200 km
center = (46.2276, 2.2137)   # France centroid
radius_km = 100.0

# Full index search (truly unknown): set radius very large
radius_km = 20000.0          # effectively global
```

### Confidence Score Interpretation

| Inlier Count | Confidence | Interpretation |
|-------------|------------|----------------|
| > 80 | High | Near-certain match |
| 30–80 | Medium | Likely correct, verify on map |
| 10–30 | Low | Plausible but uncertain |
| < 10 | Very Low | Likely false positive |

---

## Common Patterns

### Pattern: Geolocating OSINT Images with Unknown Location

1. Use **AI Coarse** mode to let Gemini estimate country/city from visual clues
2. Manually verify Gemini's estimate (check signs, driving side, vegetation, architecture)
3. Run a large-radius search (5–20 km) centered on the estimated location
4. If no match: expand radius or try a different city in the same region
5. Enable Ultra Mode for low-quality or compressed images (social media screenshots)

### Pattern: Conflict/Event Monitoring

1. Index the area of interest in advance (index persists, reusable)
2. As images surface, run Search mode with Manual coordinates + tight radius (1–2 km)
3. Cross-reference multiple images from the same event — spatial consensus across results increases confidence

### Pattern: Dense City Index vs. Sparse Regional Index

```
Dense (precise, slow to build):
  radius=1km, grid=300 → ~2000 panoramas, high match accuracy

Sparse (fast to build, lower accuracy):
  radius=5km, grid=150 → faster crawl, misses side streets
  → Use for initial triage, then rebuild dense in confirmed area
```

### Pattern: Multi-City Index

```
# Index Paris
Create mode → center=(48.8566, 2.3522), radius=5km → cosplace_parts/paris_*.npz

# Index London (same index, different parts)
Create mode → center=(51.5074, -0.1278), radius=5km → cosplace_parts/london_*.npz

# Search only Paris:
Search mode → center=(48.8566, 2.3522), radius=5km

# Search only London:
Search mode → center=(51.5074, -0.1278), radius=5km

# No conflict — radius filter separates them automatically.
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11    # replace 3.11 with your Python version
```

The system Python's bundled tkinter has rendering bugs on recent macOS. Homebrew's version fixes this.

### `ModuleNotFoundError: No module named 'lightglue'`

LightGlue must be installed from GitHub, not PyPI:

```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory during matching

Reduce keypoints per image:

```python
# In your extractor initialization, lower max_num_keypoints
extractor = ALIKED(max_num_keypoints=512).eval().to(device)  # was 1024
```

Or process fewer candidates per search (edit Stage 1 top-k from 500 to 200).

### Index search returns no results / empty candidates

- Confirm the search center + radius actually overlaps indexed panoramas
- Run `build_index.py` if `index/` is empty but `cosplace_parts/` has `.npz` files
- Check that indexing completed at least one chunk (cosplace_parts/ should be non-empty)

### LoFTR import error (`No module named 'kornia'`)

```bash
pip install kornia
```

LoFTR is optional — only required for Ultra Mode.

### MPS errors on Apple Silicon

Some LightGlue operations fall back silently. If you see MPS errors:

```python
# Force CPU fallback for debugging
device = torch.device("cpu")
```

MPS support is functional but occasionally slower than expected on M1. M2/M3/M4 are more reliable.

### Matching is slow on CPU

Expected — ALIKED/DISK + LightGlue on CPU takes ~30–60s per candidate. For 500 candidates: 4–8 hours. Options:
- Reduce top-k candidates (Stage 1) from 500 to 50–100
- Use a GPU instance (Colab with T4 works)
- Pre-filter candidates with a tighter radius

---

## Key Files Quick Reference

| File | Purpose |
|------|---------|
| `test_super.py` | Launch GUI + full pipeline |
| `cosplace_utils.py` | `load_cosplace_model()`, `extract_descriptor()` |
| `build_index.py` | Compile `cosplace_parts/` → searchable `index/` |
| `index/cosplace_descriptors.npy` | All descriptors matrix `(N, 512)` float32 |
| `index/metadata.npz` | `lats`, `lons`, `panoids`, `headings` arrays |
| `cosplace_parts/*.npz` | Incremental chunks from indexing runs |
```
