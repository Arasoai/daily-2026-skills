```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a locally-hosted open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - use netryx to locate
  - index street view panoramas
  - run netryx geolocation pipeline
  - find where a photo was taken
  - osint image geolocation tool
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a visual index using CosPlace embeddings, and matches query images against that index using local feature extraction (ALIKED/DISK) and deep feature matching (LightGlue). It achieves sub-50m accuracy without requiring landmarks or internet image matches.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (install from source)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

**macOS tkinter fix** (if GUI renders blank):
```bash
brew install python-tk@3.11  # match your Python version
```

**Optional Gemini API key** (for AI Coarse blind geolocation):
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

GPU backends: **CUDA** (NVIDIA), **MPS** (Apple Silicon M1+), **CPU** (slow fallback).

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface for all operations: indexing, searching, and viewing results on a map.

---

## Core Workflow

### Step 1: Create an Index

Index a geographic area by crawling street-view panoramas and extracting CosPlace fingerprints.

1. Select **Create** mode in the GUI
2. Enter center coordinates (lat, lon)
3. Set radius (km) — start with `0.5`–`1` for testing
4. Set grid resolution (default `300` — do not change)
5. Click **Create Index**

Indexing is incremental — interruptions are safe and will resume on next run.

**Time and storage estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|------------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

### Step 2: Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + radius
   - **AI Coarse**: Gemini analyzes the image to estimate the region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result appears on the map with a confidence score

---

## Project Structure

```
netryx/
├── test_super.py          # Main application — GUI, indexing, search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (auto-created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Internals

### Stage 1 — Global Retrieval (CosPlace)

Extracts a 512-dim descriptor from the query image (and its horizontal flip), then does a cosine similarity search against the index filtered by haversine radius.

```python
# cosplace_utils.py usage pattern
from cosplace_utils import get_cosplace_model, get_descriptor
import torch
from PIL import Image

model = get_cosplace_model()  # loads CosPlace ResNet backbone
model.eval()

img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img)  # returns torch.Tensor [512]
```

### Stage 2 — Local Feature Matching (ALIKED/DISK + LightGlue)

For each candidate panorama:
- Downloads tiles from Street View, stitches panorama
- Crops at heading angle across 3 FOVs (70°, 90°, 110°)
- Extracts keypoints with ALIKED (CUDA) or DISK (MPS/CPU)
- Matches with LightGlue
- Filters with RANSAC — counts geometric inliers

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Choose extractor based on device
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)

matcher = LightGlue(features="aliked" if device.type == "cuda" else "disk").eval().to(device)

# Load and extract features
image0 = load_image("query.jpg").to(device)
image1 = load_image("candidate_crop.jpg").to(device)

feats0 = extractor.extract(image0)
feats1 = extractor.extract(image1)

matches01 = matcher({"image0": feats0, "image1": feats1})
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

matched_kps0 = feats0["keypoints"][matches01["matches"][..., 0]]
matched_kps1 = feats1["keypoints"][matches01["matches"][..., 1]]
print(f"Matches found: {len(matches01['matches'])}")
```

### Stage 3 — Refinement

- **Heading refinement**: Tests ±45° offsets at 15° steps for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells, prefers clusters over isolated outliers
- **Confidence scoring**: Uniqueness ratio (best vs. runner-up at different location)

### Ultra Mode (Optional)

Enable in GUI checkbox. Adds three techniques for difficult images:

```python
# LoFTR dense matching (no keypoint detection required)
import kornia.feature as KF

matcher = KF.LoFTR(pretrained="outdoor").eval().to(device)

img0 = kornia.io.load_image("query.jpg", kornia.io.ImageLoadType.GRAY32).unsqueeze(0).to(device)
img1 = kornia.io.load_image("candidate.jpg", kornia.io.ImageLoadType.GRAY32).unsqueeze(0).to(device)

input_dict = {"image0": img0, "image1": img1}
with torch.inference_mode():
    correspondences = matcher(input_dict)

mkpts0 = correspondences["keypoints0"]  # matched points in query
mkpts1 = correspondences["keypoints1"]  # matched points in candidate
```

Ultra Mode also enables:
- **Descriptor hopping**: Re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of the best match

---

## Multi-City Index Strategy

All embeddings share one unified index. Radius filtering at search time isolates cities:

```
# Index multiple cities into the same index:
# 1. Create → Paris center (48.8566, 2.3522), radius 5km
# 2. Create → London center (51.5074, -0.1278), radius 5km
# 3. Create → Tel Aviv center (32.0853, 34.7818), radius 5km

# Search only Paris results:
# Search → center (48.8566, 2.3522), radius 5km

# Search only London results:
# Search → center (51.5074, -0.1278), radius 5km
```

No city selection UI needed — coordinates + radius handle everything automatically.

---

## Using the Standalone Index Builder (Large Datasets)

For large areas, use `build_index.py` directly instead of the GUI:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 10 \
  --grid 300
```

This is more efficient for multi-hour indexing jobs and supports resume on interruption.

---

## Common Patterns

### Pattern: Programmatic CosPlace descriptor extraction

```python
import numpy as np
from PIL import Image
from cosplace_utils import get_cosplace_model, get_descriptor

model = get_cosplace_model()

def extract_and_save_descriptor(image_path: str, output_path: str):
    img = Image.open(image_path).convert("RGB")
    desc = get_descriptor(model, img)
    np.save(output_path, desc.cpu().numpy())
    return desc

desc = extract_and_save_descriptor("street.jpg", "street_descriptor.npy")
print(f"Descriptor shape: {desc.shape}")  # [512]
```

### Pattern: Cosine similarity search against index

```python
import numpy as np

def search_index(query_descriptor: np.ndarray, 
                 index_path: str = "index/cosplace_descriptors.npy",
                 meta_path: str = "index/metadata.npz",
                 center_lat: float = 48.8566,
                 center_lon: float = 2.3522,
                 radius_km: float = 2.0,
                 top_k: int = 500):
    
    descriptors = np.load(index_path)  # [N, 512]
    meta = np.load(meta_path, allow_pickle=True)
    lats = meta["lats"]
    lons = meta["lons"]
    
    # Haversine radius filter
    R = 6371.0
    dlat = np.radians(lats - center_lat)
    dlon = np.radians(lons - center_lon)
    a = (np.sin(dlat / 2) ** 2 +
         np.cos(np.radians(center_lat)) * np.cos(np.radians(lats)) *
         np.sin(dlon / 2) ** 2)
    distances_km = 2 * R * np.arcsin(np.sqrt(a))
    mask = distances_km <= radius_km
    
    filtered_descs = descriptors[mask]
    
    # Cosine similarity (descriptors assumed L2-normalized)
    query_norm = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    sims = filtered_descs @ query_norm
    
    top_indices = np.argsort(sims)[::-1][:top_k]
    return top_indices, sims[top_indices]
```

### Pattern: Device-aware extractor selection

```python
import torch

def get_device_and_extractor():
    if torch.cuda.is_available():
        device = torch.device("cuda")
        from lightglue import ALIKED
        extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
        extractor_name = "aliked"
    elif torch.backends.mps.is_available():
        device = torch.device("mps")
        from lightglue import DISK
        extractor = DISK(max_num_keypoints=768).eval().to(device)
        extractor_name = "disk"
    else:
        device = torch.device("cpu")
        from lightglue import DISK
        extractor = DISK(max_num_keypoints=512).eval().to(device)
        extractor_name = "disk"
    
    from lightglue import LightGlue
    matcher = LightGlue(features=extractor_name).eval().to(device)
    
    return device, extractor, matcher, extractor_name
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # replace 3.11 with your Python version
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED (try `512` instead of `1024`)
- Close other GPU workloads
- Fall back to MPS/CPU by unsetting CUDA: `CUDA_VISIBLE_DEVICES="" python test_super.py`

### LightGlue import error
```bash
# Reinstall from source — PyPI version may be stale
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### Index build interrupted / incomplete
Safe to re-run — indexing resumes from saved `cosplace_parts/*.npz` chunks. Do not delete the `cosplace_parts/` directory.

### Low confidence scores / wrong location
1. Enable **Ultra Mode** for blurry, dark, or low-texture images
2. Increase the candidate pool by expanding search radius
3. Try **AI Coarse** mode if approximate location is unknown (requires `GEMINI_API_KEY`)
4. Ensure the target area is indexed — check `cosplace_parts/` for relevant chunks

### LoFTR / kornia not found
```bash
pip install kornia
```
Ultra Mode will silently skip LoFTR if kornia is not installed and fall back to standard matching.

### Slow search performance
- CUDA >> MPS >> CPU for speed
- Standard pipeline: 2–5 min for 300–500 candidates on M2 Max
- Ultra Mode adds significant time — use only when standard fails
- For large-scale indexing (>5km radius), use `build_index.py` instead of the GUI

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (ResNet) | Global 512-dim visual fingerprint | All |
| ALIKED | Local keypoint extraction | CUDA only |
| DISK | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching + confidence | All |
| LoFTR | Dense detector-free matching (Ultra) | All (kornia required) |

- CosPlace paper: https://arxiv.org/abs/2204.02287
- ALIKED paper: https://arxiv.org/abs/2304.03608
- DISK paper: https://arxiv.org/abs/2006.13566
- LightGlue paper: https://arxiv.org/abs/2306.13643
- LoFTR paper: https://arxiv.org/abs/2104.00680
```
