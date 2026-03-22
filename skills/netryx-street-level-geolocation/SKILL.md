```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - netryx geolocation
  - index street view panoramas
  - locate where a photo was taken
  - osint geolocation tool
  - visual place recognition pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that takes any street-level photograph and returns precise GPS coordinates (sub-50m accuracy). It indexes street-view panoramas from providers like Mapillary or KartaView, extracts visual fingerprints using CosPlace, and performs geometric verification with ALIKED/DISK + LightGlue — all on your own hardware, no SaaS required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (deep feature matcher)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (handles blur/night images)
pip install kornia
```

### GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | ALIKED extractor, fastest |
| Apple Silicon (M1+) | MPS | DISK extractor, fast |
| CPU only | — | Works, significantly slower |

### Optional: Gemini API for AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix**: `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### 1. Create an Index (one-time setup per area)

In the GUI:
1. Select **Create** mode
2. Enter center latitude/longitude of the area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Indexing time reference:

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

Indexing is **incremental** — safe to interrupt and resume.

### 2. Search a Photo

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide known center coordinates + radius
   - **AI Coarse**: Let Gemini infer region from visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading & descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim visual fingerprints
    └── metadata.npz               # Coords, headings, panorama IDs
```

---

## Three-Stage Pipeline (How It Works)

```
Query Image
    │
    ├── CosPlace → 512-dim descriptor (+ flipped version)
    ▼
Index Search (cosine similarity, haversine radius filter)
    │
    └── Top 500–1000 visual candidates
    ▼
Download Panoramas → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) or DISK (MPS/CPU) → local keypoints
    ├── LightGlue → feature matching
    └── RANSAC → geometric verification (inlier count)
    ▼
Heading Refinement (±45° @ 15° steps, top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    └── Confidence scoring (uniqueness ratio)
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

# Load model (auto-selects CUDA / MPS / CPU)
model, device = load_cosplace_model()

# Extract descriptor from an image file
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)
# descriptor.shape → (512,)

# Also extract flipped version (catches reversed perspectives)
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = extract_descriptor(model, img_flipped, device)
```

### Search the Index Programmatically

```python
import numpy as np

# Load the compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,)
lons = meta["lons"]       # (N,)
headings = meta["headings"]
panoids = meta["panoids"]

# Cosine similarity search
query_desc = descriptor / np.linalg.norm(descriptor)
db_norms = descriptors / np.linalg.norm(descriptors, axis=1, keepdims=True)
scores = db_norms @ query_desc                             # (N,)

# Haversine radius filter (restrict to search area)
from math import radians, sin, cos, sqrt, atan2

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))

center_lat, center_lon = 48.8566, 2.3522   # Paris
radius_km = 2.0

mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
])

# Get top 500 candidates within radius
filtered_scores = np.where(mask, scores, -1)
top_indices = np.argsort(filtered_scores)[::-1][:500]

for idx in top_indices[:10]:
    print(f"Score: {scores[idx]:.4f} | Lat: {lats[idx]:.6f} | Lon: {lons[idx]:.6f} | Heading: {headings[idx]}")
```

### Build a Large Index with build_index.py

```bash
# For large areas, use the standalone high-performance builder
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --resolution 300
```

### Feature Matching with LightGlue (Stage 2 — verification)

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

# Load images
query = load_image("query_photo.jpg").to(device)
candidate = load_image("candidate_panorama_crop.jpg").to(device)

# Extract keypoints
feats0 = extractor.extract(query)
feats1 = extractor.extract(candidate)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

matched_points0 = feats0["keypoints"][matches01["matches"][..., 0]]
matched_points1 = feats1["keypoints"][matches01["matches"][..., 1]]

print(f"Matched keypoints: {len(matched_points0)}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_inliers(pts0, pts1):
    """Returns number of geometrically consistent matches."""
    if len(pts0) < 4:
        return 0
    pts0_np = pts0.cpu().numpy()
    pts1_np = pts1.cpu().numpy()
    _, mask = cv2.findHomography(pts0_np, pts1_np, cv2.RANSAC, ransacReprojThreshold=8.0)
    if mask is None:
        return 0
    return int(mask.sum())

inliers = ransac_inliers(matched_points0, matched_points1)
print(f"RANSAC inliers: {inliers}")  # Higher = better geometric match
```

### Ultra Mode: LoFTR Dense Matching

```python
import kornia
import torch
import cv2
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
matcher = kornia.feature.LoFTR(pretrained="outdoor").eval().to(device)

def to_gray_tensor(img_path, size=(640, 480)):
    img = cv2.imread(img_path, cv2.IMREAD_GRAYSCALE)
    img = cv2.resize(img, size)
    return torch.tensor(img / 255.0, dtype=torch.float32)[None, None].to(device)

query_gray = to_gray_tensor("query_photo.jpg")
candidate_gray = to_gray_tensor("candidate_crop.jpg")

with torch.no_grad():
    output = matcher({"image0": query_gray, "image1": candidate_gray})

mkpts0 = output["keypoints0"].cpu().numpy()
mkpts1 = output["keypoints1"].cpu().numpy()
conf = output["confidence"].cpu().numpy()

# Filter by confidence
high_conf = conf > 0.5
print(f"High-confidence LoFTR matches: {high_conf.sum()}")
```

---

## Configuration Reference

### Index Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `radius` | — | Search area radius in km |
| `resolution` | 300 | Grid density — higher = more panoramas, don't change unless you know why |
| Center lat/lon | — | Center of area to index |

### Search Parameters

| Parameter | Notes |
|-----------|-------|
| Center coords | Required for manual mode |
| Radius | Restricts index search by haversine distance |
| Ultra Mode | Enable for blur, night, low-texture images |
| AI Coarse | Requires `GEMINI_API_KEY` env var |

### Multi-Index Strategy

The index is **location-agnostic** — all cities share one index file. The radius filter at search time handles scoping:

```python
# Index Paris + London into same index — no conflict
# Search Paris:  center=(48.8566, 2.3522), radius=5km → only Paris results
# Search London: center=(51.5074, -0.1278), radius=5km → only London results
```

---

## Common Patterns

### Pattern 1: Programmatic Full Search Pipeline

```python
# Pseudocode reflecting the actual pipeline in test_super.py
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import numpy as np

model, device = load_cosplace_model()

# Stage 1: Global retrieval
query_img = Image.open("target.jpg").convert("RGB")
desc = extract_descriptor(model, query_img, device)
desc_flip = extract_descriptor(model, query_img.transpose(Image.FLIP_LEFT_RIGHT), device)

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

# Combined similarity (original + flipped)
scores = descriptors @ desc + descriptors @ desc_flip
top_candidates = np.argsort(scores)[::-1][:500]

# Stage 2: LightGlue verification on top candidates
# (download panorama tiles, crop at multiple FOVs, run ALIKED+LightGlue+RANSAC)

# Stage 3: Spatial consensus — cluster top matches into 50m cells
# Winner = cell with highest aggregate inlier count
```

### Pattern 2: Batch Indexing Multiple Cities

```python
cities = [
    {"name": "Paris",  "lat": 48.8566, "lon": 2.3522,   "radius": 5.0},
    {"name": "London", "lat": 51.5074, "lon": -0.1278,  "radius": 5.0},
    {"name": "Berlin", "lat": 52.5200, "lon": 13.4050,  "radius": 5.0},
]

# Run build_index.py for each; all write to the same cosplace_parts/ directory
import subprocess
for city in cities:
    subprocess.run([
        "python", "build_index.py",
        "--lat", str(city["lat"]),
        "--lon", str(city["lon"]),
        "--radius", str(city["radius"]),
        "--resolution", "300"
    ])
```

### Pattern 3: Confidence Evaluation

```python
def evaluate_confidence(top_matches, cluster_radius_m=50):
    """
    Spatial consensus: group matches into 50m cells.
    High confidence = dominant cluster + strong uniqueness ratio.
    """
    from collections import defaultdict

    cell_scores = defaultdict(int)
    for match in top_matches:
        # Quantize to ~50m grid cells
        cell_lat = round(match["lat"] / 0.0005) * 0.0005
        cell_lon = round(match["lon"] / 0.0005) * 0.0005
        cell_scores[(cell_lat, cell_lon)] += match["inliers"]

    sorted_cells = sorted(cell_scores.items(), key=lambda x: x[1], reverse=True)

    if len(sorted_cells) < 2:
        return sorted_cells[0] if sorted_cells else None, 0.0

    best_score = sorted_cells[0][1]
    second_score = sorted_cells[1][1]
    uniqueness_ratio = best_score / (second_score + 1e-6)

    confidence = min(1.0, uniqueness_ratio / 3.0)   # ratio of 3+ → high confidence
    return sorted_cells[0][0], confidence
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11   # Replace 3.11 with your Python version
```

### LightGlue import error

```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory

- Reduce `max_num_keypoints` in ALIKED (try 512 instead of 1024)
- Process fewer candidates per batch (reduce top-N from 500 to 200)
- Use `torch.cuda.empty_cache()` between candidate batches

### MPS (Apple Silicon) errors with ALIKED

MPS uses DISK automatically — this is expected behavior. ALIKED requires CUDA. No action needed.

### Index search returns no results

- Verify the search radius actually covers the indexed area (check lat/lon)
- Confirm the index was built: `index/cosplace_descriptors.npy` and `index/metadata.npz` must exist
- Re-run index compilation: in GUI, trigger **Auto-build** after indexing completes

### Poor match quality / wrong location

1. Enable **Ultra Mode** (adds LoFTR + descriptor hopping + neighborhood expansion)
2. Increase search radius slightly — correct panorama may be at edge of indexed area
3. Try **AI Coarse** mode to get a better center coordinate estimate
4. Check image quality — heavily compressed, watermarked, or very narrow FOV images reduce accuracy

### Indexing hangs or stops

Indexing is incremental — `cosplace_parts/` saves progress chunk by chunk. Simply re-run the same command/GUI action and it will resume from the last saved chunk.

---

## Models Reference

| Model | Role | Paper |
|-------|------|-------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global visual fingerprint (512-dim) | CVPR 2022 |
| [ALIKED](https://github.com/naver/alike) | Local keypoints — CUDA | IEEE TIP 2023 |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints — MPS/CPU | NeurIPS 2020 |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | ICCV 2023 |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching, Ultra Mode | CVPR 2021 |
```
