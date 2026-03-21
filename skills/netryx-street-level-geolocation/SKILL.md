---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision pipelines.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - netryx geolocation
  - identify location from street view photo
  - visual place recognition pipeline
  - index street view panoramas
  - osint geolocation from photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable visual index using CosPlace embeddings, then verifies matches with local feature matching (ALIKED/DISK + LightGlue). Sub-50m accuracy, no landmarks required, runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Requirements
- Python 3.9+ (3.10+ recommended)
- GPU: NVIDIA (CUDA, 4GB+ VRAM) or Apple Silicon (MPS) — CPU works but is slow
- RAM: 8GB minimum, 16GB+ recommended
- Storage: 10GB minimum; 50GB+ for large city indexes

### Optional: Gemini API Key (AI Coarse mode)
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

## Core Workflow

### 1. Create an Index (crawl + embed an area)

In GUI: select **Create** mode, enter center lat/lon, radius (km), grid resolution (default 300), click **Create Index**.

The index saves incrementally — safe to interrupt and resume.

**Indexing time estimates:**

| Radius | Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

For large areas, use the standalone high-performance builder:
```bash
python build_index.py
```

### 2. Search

In GUI: select **Search** mode, upload a street photo, choose search method:
- **Manual**: provide approximate center coordinates + radius
- **AI Coarse**: let Gemini infer the region from visual cues (requires `GEMINI_API_KEY`)

Click **Run Search** → **Start Full Search**.

Enable **Ultra Mode** checkbox for difficult images (night, blur, low texture).

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (generated during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors (searchable)
    └── metadata.npz               # Coordinates, headings, panoid IDs
```

---

## Key Code Patterns

### Extract a CosPlace Descriptor Manually

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
model = load_cosplace_model()

# Extract 512-dim descriptor from an image
img = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, img)  # shape: (512,)
print(descriptor.shape)  # (512,)
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]   # (N,)
lons = meta["lons"]   # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1))*cos(radians(lat2))*sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km=2.0, top_k=500):
    """
    Returns top_k candidate indices sorted by cosine similarity,
    filtered to within radius_km of center.
    """
    # Radius filter
    mask = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
        for i in range(len(lats))
    ])
    filtered_idx = np.where(mask)[0]

    if len(filtered_idx) == 0:
        return []

    # Cosine similarity (descriptors assumed L2-normalized)
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    d = descriptors[filtered_idx]
    d = d / (np.linalg.norm(d, axis=1, keepdims=True) + 1e-8)
    sims = d @ q  # (M,)

    # Top-k
    top_local = np.argsort(-sims)[:top_k]
    return filtered_idx[top_local].tolist()

# Usage
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

model = load_cosplace_model()
query = Image.open("query.jpg").convert("RGB")
desc = extract_descriptor(model, query)

candidates = search_index(desc, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
print(f"Found {len(candidates)} candidates")
# candidates[0] -> index into lats/lons/panoids arrays
print(f"Top match: lat={lats[candidates[0]]:.6f}, lon={lons[candidates[0]]:.6f}")
```

### Flipped Descriptor Search (catches reversed perspectives)

```python
import PIL.ImageOps

query_img = Image.open("query.jpg").convert("RGB")
query_flipped = PIL.ImageOps.mirror(query_img)

desc_orig = extract_descriptor(model, query_img)
desc_flip = extract_descriptor(model, query_flipped)

# Search with both, merge and deduplicate
cands_orig = search_index(desc_orig, 48.8566, 2.3522, radius_km=2.0, top_k=500)
cands_flip = search_index(desc_flip, 48.8566, 2.3522, radius_km=2.0, top_k=500)
all_candidates = list(dict.fromkeys(cands_orig + cands_flip))[:500]
```

---

## Pipeline Stages Summary

| Stage | Models | Speed | Purpose |
|-------|--------|-------|---------|
| Global retrieval | CosPlace (512-dim) | <1 sec | Narrow millions → top 500 via cosine similarity |
| Geometric verification | ALIKED (CUDA) / DISK (MPS/CPU) + LightGlue + RANSAC | 2–5 min | Prove same location via keypoint matching |
| Heading refinement | Same as stage 2 | +1–2 min | Test ±45° offsets, 3 FOVs for top 15 candidates |
| Spatial consensus | Haversine clustering (50m cells) | Instant | Prefer clustered matches, reduce false positives |
| Ultra Mode (optional) | + LoFTR + descriptor hopping | +3–10 min | Handle blur/night/low-texture images |

---

## Platform-Specific Behavior

| Feature | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU |
|---------|--------------|---------------------|-----|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| Speed | Fastest | Fast | Slow |
| Ultra Mode (LoFTR) | Supported | Supported | Supported |

---

## Multi-City Indexing

All embeddings go into a **single unified index**. The radius filter at search time scopes results automatically — no per-city files needed.

```
# Index Paris
Create Index: center=(48.8566, 2.3522), radius=5km

# Index London  
Create Index: center=(51.5074, -0.1278), radius=5km

# Search only Paris
Search: center=(48.8566, 2.3522), radius=5km  → only Paris results

# Search only London
Search: center=(51.5074, -0.1278), radius=5km  → only London results
```

---

## Common Patterns

### Batch Index Multiple Cities

```python
import subprocess
import os

cities = [
    ("paris",   48.8566,  2.3522,  5.0),
    ("london",  51.5074, -0.1278,  5.0),
    ("berlin",  52.5200, 13.4050,  5.0),
]

for name, lat, lon, radius in cities:
    print(f"Indexing {name}...")
    # Modify build_index.py args or call programmatically
    # The index auto-merges into cosplace_parts/ and index/
```

### Check Index Size

```python
import numpy as np

descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)

print(f"Total indexed panoramas: {len(descriptors):,}")
print(f"Descriptor matrix shape: {descriptors.shape}")
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
```

### Verify GPU Detection

```python
import torch

if torch.cuda.is_available():
    print(f"CUDA: {torch.cuda.get_device_name(0)}")
elif torch.backends.mps.is_available():
    print("MPS (Apple Silicon) available")
else:
    print("CPU only — expect slow performance")
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `ModuleNotFoundError: lightglue` | Not installed from GitHub | `pip install git+https://github.com/cvg/LightGlue.git` |
| macOS GUI blank/unresponsive | Broken system tkinter | `brew install python-tk@3.11` |
| LoFTR import error in Ultra Mode | kornia not installed | `pip install kornia` |
| No candidates returned | Index empty or wrong radius | Check index size; increase radius; verify coordinates |
| Low confidence / wrong result | Query outside indexed area | Re-index the correct city/region |
| Slow search (>30 min) | CPU-only, no GPU | Use CUDA/MPS machine; reduce `top_k` candidates |
| Index build interrupted | Process killed mid-crawl | Safe to restart — resumes from saved `.npz` chunks |
| Gemini AI Coarse not working | Missing API key | `export GEMINI_API_KEY="..."` before launching |
| ALIKED unavailable on MPS | Expected behavior | MPS uses DISK automatically — normal fallback |

### Verify Installation End-to-End

```python
# Quick smoke test
import torch
from PIL import Image
import numpy as np

# 1. Check LightGlue
from lightglue import LightGlue, ALIKED, DISK
print("LightGlue OK")

# 2. Check CosPlace utils
from cosplace_utils import load_cosplace_model, extract_descriptor
model = load_cosplace_model()
print("CosPlace model loaded")

# 3. Extract descriptor from a test image
img = Image.new("RGB", (640, 480), color=(128, 128, 128))
desc = extract_descriptor(model, img)
assert desc.shape == (512,), f"Expected (512,), got {desc.shape}"
print(f"Descriptor shape: {desc.shape} ✓")

# 4. Check index exists
try:
    d = np.load("index/cosplace_descriptors.npy")
    m = np.load("index/metadata.npz")
    print(f"Index loaded: {len(d):,} panoramas")
except FileNotFoundError:
    print("No index found — run Create Index first")
```

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global visual place recognition | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoint extraction | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoint extraction | MPS / CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense detector-free matching (Ultra Mode) | All (via kornia) |
