```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, an open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - reverse geolocation from photo
  - netryx geolocation
  - index street view panoramas
  - osint geolocation tool
  - identify location from street image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies exact GPS coordinates from any street-level photograph. It crawls street-view panoramas, indexes them as 512-dimensional CosPlace fingerprints, then matches query images using local feature extraction (ALIKED/DISK) and deep feature matching (LightGlue). Sub-50m accuracy with no landmarks required, running entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue matching library
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"   # Never hardcode — use env var
```

### macOS tkinter fix (blank GUI)

```bash
brew install python-tk@3.11   # Match your Python version
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. It handles both indexing and searching.

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The indexer crawls street-view panoramas in a grid, extracts CosPlace descriptors, and saves them incrementally.

**In the GUI:**
1. Select **Create** mode
2. Enter center latitude/longitude
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Indexing time vs. radius:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is resumable — interrupting and restarting continues from the last saved chunk in `cosplace_parts/`.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Let Gemini estimate the region from visual cues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score plotted on map

---

## Project Structure

```
netryx/
├── test_super.py           # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py       # CosPlace model loading + descriptor extraction
├── build_index.py          # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/         # Raw embedding chunks (auto-created)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panoid IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├─ CosPlace → 512-dim descriptor
    ├─ Flipped image → 512-dim descriptor (handles reversed perspectives)
    │
    ▼
Index Search (cosine similarity + haversine radius filter)
    │
    └─ Top 500 candidates
    │
    ▼
For each candidate:
    Download panorama (8 tiles, stitched)
    Crop at 3 FOVs (70°, 90°, 110°)
    Extract ALIKED (CUDA) or DISK (MPS/CPU) keypoints
    LightGlue matching vs. query keypoints
    RANSAC geometric verification → inlier count
    │
    ▼
Heading Refinement (±45°, 15° steps, top 15 candidates)
Spatial Consensus (50m grid clustering)
Confidence Scoring (cluster density + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable **Ultra Mode** in the GUI for difficult images (night shots, blur, low texture). Adds three extra steps:

1. **LoFTR** — detector-free dense matching (handles blur/low contrast)
2. **Descriptor hopping** — if best match has <50 inliers, re-search using the matched panorama's clean descriptor
3. **Neighborhood expansion** — search all panoramas within 100m of best match

Ultra Mode is ~3–5× slower but recovers matches that the standard pipeline misses.

---

## Using the Standalone Index Builder

For large areas (5km+ radius), use `build_index.py` directly for better performance:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300
```

This writes chunks to `cosplace_parts/` and auto-builds the unified index at completion.

---

## CosPlace Utilities — Programmatic Usage

```python
# cosplace_utils.py exposes descriptor extraction
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-selects CUDA > MPS > CPU)
model, device = load_cosplace_model()

# Extract descriptor from an image file
img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, img, device)
# descriptor: torch.Tensor, shape (512,)

# Flip variant (catches reversed perspectives)
img_flipped = img.transpose(Image.FLIP_LEFT_RIGHT)
descriptor_flipped = extract_descriptor(model, img_flipped, device)
```

---

## Index Search — Programmatic Usage

```python
import numpy as np

# Load the compiled index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512) float32
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,) float64
lons = meta["lons"]       # (N,) float64
headings = meta["headings"]  # (N,) float32
panoids = meta["panoids"]    # (N,) str

# Cosine similarity search
query_desc = descriptor.cpu().numpy().astype(np.float32)
query_desc /= np.linalg.norm(query_desc)

norms = np.linalg.norm(descriptors, axis=1, keepdims=True)
normed = descriptors / (norms + 1e-8)
scores = normed @ query_desc                # (N,) cosine similarities

# Haversine radius filter (example: 5km around Paris)
center_lat, center_lon = 48.8566, 2.3522
radius_km = 5.0

dlat = np.radians(lats - center_lat)
dlon = np.radians(lons - center_lon)
a = np.sin(dlat/2)**2 + np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) * np.sin(dlon/2)**2
dist_km = 6371 * 2 * np.arcsin(np.sqrt(a))

mask = dist_km <= radius_km
scores[~mask] = -1.0                        # Exclude out-of-radius

top_indices = np.argsort(scores)[::-1][:500]   # Top 500 candidates

for idx in top_indices[:5]:
    print(f"panoid={panoids[idx]}, lat={lats[idx]:.6f}, lon={lons[idx]:.6f}, "
          f"heading={headings[idx]:.1f}°, score={scores[idx]:.4f}")
```

---

## Feature Matching — Programmatic Usage

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU (matches Netryx's own logic)
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

# Load and extract features
image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

# Match
matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matched keypoints: {len(kpts0)}")
```

---

## RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kpts0: np.ndarray, kpts1: np.ndarray, threshold: float = 3.0):
    """
    Returns inlier count after RANSAC homography estimation.
    Higher inliers = stronger geometric match.
    """
    if len(kpts0) < 4:
        return 0
    _, mask = cv2.findHomography(
        kpts0.astype(np.float32),
        kpts1.astype(np.float32),
        cv2.RANSAC,
        threshold
    )
    if mask is None:
        return 0
    return int(mask.sum())

# Usage
kp0_np = kpts0.cpu().numpy()
kp1_np = kpts1.cpu().numpy()
inliers = ransac_verify(kp0_np, kp1_np)
print(f"Inliers: {inliers}  ({'strong match' if inliers > 50 else 'weak match'})")
```

---

## Multi-Index Strategy

All cities share a single index. Use coordinate + radius to scope searches:

```python
# Index Paris: center=(48.8566, 2.3522), radius=5km
# Index London: center=(51.5074, -0.1278), radius=5km
# Index Tel Aviv: center=(32.0853, 34.7818), radius=5km
# All stored in the same cosplace_descriptors.npy

# Search only Paris results:
search(center_lat=48.8566, center_lon=2.3522, radius_km=5.0)

# Search only London results:
search(center_lat=51.5074, center_lon=-0.1278, radius_km=5.0)
```

No city selection dropdown needed — radius filtering handles isolation automatically.

---

## Hardware & Device Selection

```python
import torch

def get_device():
    if torch.cuda.is_available():
        return torch.device("cuda")          # NVIDIA — ALIKED, fastest
    elif torch.backends.mps.is_available():
        return torch.device("mps")           # Apple Silicon — DISK
    else:
        return torch.device("cpu")           # Fallback — DISK, slowest
```

| Platform       | Feature Extractor | Typical Search Time |
|---------------|------------------|---------------------|
| NVIDIA CUDA    | ALIKED (1024 kp) | 2–3 min             |
| Apple MPS (M1+)| DISK (768 kp)    | 3–5 min             |
| CPU only       | DISK (768 kp)    | 15–30 min           |

---

## Common Patterns

### Pattern 1: Batch index multiple cities

```bash
# Run indexing sequentially for multiple areas
python build_index.py --lat 48.8566 --lon 2.3522 --radius 3.0  # Paris
python build_index.py --lat 51.5074 --lon -0.1278 --radius 3.0 # London
# Index auto-merges into the same cosplace_parts/ directory
```

### Pattern 2: Headless search (no GUI)

The search logic lives in `test_super.py`. Extract and call the search function directly:

```python
# Approximate pattern — adapt to actual function signatures in test_super.py
import sys
sys.argv = ["test_super.py"]   # Prevent GUI arg parsing if needed

from test_super import run_search_pipeline

result = run_search_pipeline(
    query_image_path="photo.jpg",
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=2.0,
    ultra_mode=False
)
print(f"Result: lat={result['lat']}, lon={result['lon']}, confidence={result['confidence']}")
```

### Pattern 3: Verify your index is healthy

```python
import numpy as np

desc = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

print(f"Total indexed panoramas: {len(desc)}")
print(f"Descriptor shape: {desc.shape}")         # Should be (N, 512)
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
print(f"NaN descriptors: {np.isnan(desc).any()}")  # Should be False
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # Use your actual Python version
```

### CUDA out of memory
Reduce `max_num_keypoints` in ALIKED:
```python
extractor = ALIKED(max_num_keypoints=512).eval().to(device)  # Down from 1024
```

### LightGlue import error
```bash
pip install git+https://github.com/cvg/LightGlue.git --force-reinstall
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
# kornia ships LoFTR — Ultra Mode checkbox will activate once installed
```

### Indexing stalls / no panoramas found
- Verify the area has street-view coverage (rural areas may have sparse data)
- Lower grid resolution (try 200 instead of 300) for denser sampling
- Check internet connectivity — panorama downloads require broadband

### Weak matches (low inlier count)
- Enable **Ultra Mode** for blurry/night/low-texture images
- Increase search radius to include more candidates
- Ensure query image is street-level (not aerial, not interior)

### Resuming interrupted indexing
Just re-run the same Create command with the same parameters. The system detects existing chunks in `cosplace_parts/` and skips already-processed grid cells.

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim descriptor | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoints (preferred) | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints (fallback) | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra Mode) | All (via kornia) |
```
