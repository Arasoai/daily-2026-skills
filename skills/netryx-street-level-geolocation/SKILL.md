---
name: netryx-street-level-geolocation
description: Use Netryx to index street-view panoramas and geolocate any street-level photo to precise GPS coordinates using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - index street view panoramas
  - run netryx search
  - identify location from photo
  - osint geolocation tool
  - local geolocation pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that finds the exact GPS coordinates of any street-level photograph. It crawls street-view panoramas, indexes them as visual fingerprints using CosPlace, then matches query images via ALIKED/DISK keypoints + LightGlue deep feature matching. Sub-50m accuracy, no landmarks required, runs entirely on local hardware.

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

### Platform GPU support
| Platform | Backend | Notes |
|---|---|---|
| NVIDIA GPU | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon | MPS | Uses DISK (768 keypoints) |
| CPU only | — | Works, significantly slower |

### Optional: Gemini API key (AI Coarse region detection)
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

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (auto-created)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Core Workflow

### 1. Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center `latitude, longitude`
3. Set radius (km) and grid resolution (default 300)
4. Click **Create Index**

Indexing is incremental — safe to interrupt and resume.

**Time estimates:**
| Radius | ~Panoramas | Time (M2 Max) | Index size |
|---|---|---|---|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hr | ~250 MB |
| 5 km | ~30,000 | 8–12 hr | ~3 GB |
| 10 km | ~100,000 | 24–48 hr | ~7 GB |

### 2. Search (geolocate a photo)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose **Manual** (enter coords + radius) or **AI Coarse** (Gemini auto-detects region)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on map

### 3. Ultra Mode (difficult images)

Enable **Ultra Mode** checkbox for:
- Night photos
- Motion blur
- Low texture scenes

Adds: LoFTR dense matching + descriptor hopping + 100m neighborhood expansion. Slower but catches hard misses.

---

## Pipeline Internals

```
Query Image
    │
    ├─ CosPlace → 512-dim descriptor
    ├─ Flipped image → 512-dim descriptor
    │
    ▼
Cosine similarity vs full index (radius-filtered by haversine)
    │
    └─ Top 500 candidates  (<1 second)
         │
         ▼
    Download panorama tiles → stitch → crop at 3 FOVs (70°/90°/110°)
         │
         ├─ ALIKED (CUDA) or DISK (MPS/CPU) → keypoints + descriptors
         ├─ LightGlue → feature matching
         └─ RANSAC → geometric verification → inlier count
              │
              ▼
         Heading refinement: ±45° at 15° steps, top 15 candidates
              │
              ├─ Spatial consensus clustering (50m cells)
              ├─ Confidence scoring (uniqueness ratio)
              │
              ▼
         📍 GPS + confidence score
```

---

## Using CosPlace Utilities Directly

```python
# cosplace_utils.py exposes descriptor extraction
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

model = load_cosplace_model(device=device)

img = Image.open("query_photo.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)   # shape: (512,)
print(descriptor.shape)  # torch.Size([512])
```

---

## Searching the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = sin(dlat/2)**2 + cos(lat1)*cos(lat2)*sin(dlon/2)**2
    return R * 2 * asin(sqrt(a))

# Load pre-built index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512) float32
meta        = np.load("index/metadata.npz", allow_pickle=True)
lats        = meta["lats"]     # shape (N,)
lons        = meta["lons"]     # shape (N,)
panoids     = meta["panoids"]  # shape (N,)

# Radius filter: Paris center, 1 km
center_lat, center_lon, radius_km = 48.8566, 2.3522, 1.0
mask = np.array([
    haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
    for i in range(len(lats))
], dtype=bool)

# Cosine similarity search
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model  = load_cosplace_model(device=device)
img    = Image.open("query.jpg").convert("RGB")
query_desc = get_descriptor(model, img, device)            # (512,)

filtered_descs = descriptors[mask]                         # (M, 512)
query_norm     = query_desc / (np.linalg.norm(query_desc) + 1e-8)
db_norms       = filtered_descs / (
    np.linalg.norm(filtered_descs, axis=1, keepdims=True) + 1e-8
)
scores  = db_norms @ query_norm                            # (M,)
top_idx = np.argsort(scores)[::-1][:500]                  # top 500

# Map back to global indices
global_indices = np.where(mask)[0][top_idx]
for i, gi in enumerate(global_indices[:5]):
    print(f"Rank {i+1}: lat={lats[gi]:.6f} lon={lons[gi]:.6f} "
          f"panoid={panoids[gi]} score={scores[top_idx[i]]:.4f}")
```

---

## Building a Large Index with build_index.py

For large areas (5+ km radius), use the standalone builder which is faster than the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --grid 300
```

This writes chunks to `cosplace_parts/` and auto-compiles `index/cosplace_descriptors.npy` + `index/metadata.npz` when complete.

---

## Multi-Area Indexing

All areas share one unified index. The radius filter at search time scopes results automatically — no per-city selection needed.

```python
# You can index multiple cities into the same index:
# Paris:  center=(48.8566, 2.3522), radius=5km
# London: center=(51.5074, -0.1278), radius=5km
# Then search Paris-only by passing center=Paris + radius=5km
# London results are filtered out by haversine distance automatically.
```

---

## Common Patterns

### Flip augmentation for reversed perspectives

```python
import torchvision.transforms.functional as TF

img_flipped  = TF.hflip(img)
desc_normal  = get_descriptor(model, img, device)
desc_flipped = get_descriptor(model, img_flipped, device)

# Use max score across both descriptors per candidate
score_normal  = db_norms @ (desc_normal  / np.linalg.norm(desc_normal))
score_flipped = db_norms @ (desc_flipped / np.linalg.norm(desc_flipped))
combined      = np.maximum(score_normal, score_flipped)
```

### Confidence scoring heuristic

```python
# Uniqueness ratio: best match vs runner-up at different location (>50m away)
sorted_scores = np.sort(combined)[::-1]
best_score    = sorted_scores[0]

# Find first candidate that is >50m from best match
best_lat = lats[global_indices[0]]
best_lon = lons[global_indices[0]]
for i, gi in enumerate(global_indices[1:], 1):
    if haversine_km(best_lat, best_lon, lats[gi], lons[gi]) * 1000 > 50:
        runner_up = sorted_scores[i]
        break

uniqueness_ratio = best_score / (runner_up + 1e-8)
confidence = "HIGH" if uniqueness_ratio > 1.3 else "MEDIUM" if uniqueness_ratio > 1.1 else "LOW"
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| GUI appears blank on macOS | Bundled tkinter bug | `brew install python-tk@3.11` |
| `ImportError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| Ultra Mode / LoFTR not available | kornia missing | `pip install kornia` |
| CUDA OOM on verification stage | VRAM too low | Reduce top-K candidates from 500 to 200; or use CPU fallback |
| Indexing stalls / slow | Network throttling | Indexing is incremental — restart resumes from last checkpoint |
| Poor match accuracy | Index too sparse | Lower grid resolution value (e.g. 200) for denser coverage |
| AI Coarse mode fails | Missing API key | `export GEMINI_API_KEY="..."` or use Manual mode instead |
| MPS device errors on Mac | PyTorch version mismatch | `pip install --upgrade torch torchvision` |
| Low inlier count (<30) | Query image blurry/dark | Enable Ultra Mode; LoFTR handles low-texture scenes |

---

## Key Constants & Defaults

| Parameter | Default | Notes |
|---|---|---|
| Grid resolution | 300 | Points per axis; don't modify without understanding density impact |
| Top-K candidates (Stage 1) | 500–1000 | Balanced retrieval recall vs Stage 2 speed |
| Keypoints — ALIKED (CUDA) | 1024 | Higher = better matching, more VRAM |
| Keypoints — DISK (MPS/CPU) | 768 | |
| Heading refinement range | ±45° at 15° steps | Applied to top 15 candidates |
| Spatial clustering cell | 50m | Consensus over isolated high-inlier outliers |
| Neighborhood expansion (Ultra) | 100m | Searches all panoramas within 100m of best match |
| CosPlace descriptor dim | 512 | Fixed by model architecture |
