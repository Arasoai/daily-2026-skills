```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo locally
  - find GPS coordinates from street image
  - run Netryx geolocation
  - index street view panoramas
  - use Netryx to locate a photo
  - street level geolocation without internet
  - open source geoguessr tool
  - visual place recognition pipeline
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source street-level geolocation engine. Given any street-level photograph, it returns precise GPS coordinates (sub-50m accuracy) by matching against an indexed database of street-view panoramas. It uses a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and spatial refinement — all running on your own hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue matcher
pip install git+https://github.com/cvg/LightGlue.git

# Optional: Ultra Mode (LoFTR dense matching)
pip install kornia
```

### Platform GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon (M1+) | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

### Optional: Gemini API for AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"
```

Get a free key at [Google AI Studio](https://aistudio.google.com). Not required — manual coordinate entry works fine and is recommended.

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix**: `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### Step 1: Create an Index (crawl + embed an area)

In the GUI:
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (km) and grid resolution (default 300)
4. Click **Create Index**

Index is saved incrementally — safe to interrupt and resume.

**Indexing time reference:**

| Radius | Panoramas | Est. Time (M2 Max) | Index Size |
|--------|-----------|-------------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

### Step 2: Search (geolocate a photo)

In the GUI:
1. Select **Search** mode
2. Upload your street photo
3. Choose:
   - **Manual**: Enter approximate center lat/lon + search radius
   - **AI Coarse**: Let Gemini estimate region from visual cues
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
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1: Global Retrieval (CosPlace)

- Extracts a 512-dim descriptor from the query image
- Also extracts from horizontally flipped version (catches reversed perspectives)
- Cosine similarity search against the full index
- Haversine radius filter applied
- Returns top 500–1,000 candidates
- Runs in **< 1 second** (single matrix multiply)

### Stage 2: Geometric Verification (ALIKED/DISK + LightGlue)

For each candidate:
1. Downloads panorama (8 tiles, stitched)
2. Crops rectilinear view at indexed heading
3. Generates multi-FOV crops: 70°, 90°, 110°
4. Extracts keypoints: ALIKED (CUDA) or DISK (MPS/CPU)
5. LightGlue deep feature matching
6. RANSAC filters geometrically inconsistent matches
7. Best candidate = most verified inliers

Processes 300–500 candidates in **2–5 minutes** depending on hardware.

### Stage 3: Refinement

- **Heading refinement**: Tests ±45° offsets at 15° steps for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; prefers clusters over single outliers
- **Confidence scoring**: Evaluates geographic clustering + uniqueness ratio

---

## Ultra Mode

Enable the **Ultra Mode** checkbox in the GUI for difficult images (night shots, blur, low texture).

Adds:
- **LoFTR**: Detector-free dense matching — handles blur/low-contrast
- **Descriptor hopping**: Re-searches index using descriptor from the matched (clean) panorama
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

Significantly slower; use when standard pipeline returns low confidence.

---

## Index Architecture

The index is global — all cities share one index. Radius filter at search time isolates the right region:

```
# Index multiple cities into the same index:
# Index Paris (48.8566, 2.3522, radius=5km)
# Index London (51.5074, -0.1278, radius=5km)
# Index Tel Aviv (32.0853, 34.7818, radius=5km)

# Search only Paris results:
# center=(48.8566, 2.3522), radius=5km → only Paris candidates returned

# Search only London results:
# center=(51.5074, -0.1278), radius=5km → only London candidates returned
```

No city selection UI needed — coordinates + radius handle scoping automatically.

---

## Using CosPlace Utilities Directly

```python
# cosplace_utils.py provides model loading and descriptor extraction
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model()
model.eval()

# Extract descriptor from an image file
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, img)  # Returns torch.Tensor, shape [512]

print(f"Descriptor shape: {descriptor.shape}")  # torch.Size([512])
print(f"Descriptor norm: {descriptor.norm().item():.4f}")  # Should be ~1.0 (L2 normalized)
```

---

## Building a Large Index Programmatically

For large areas, use the standalone high-performance builder:

```bash
# build_index.py — faster than the GUI indexer for large datasets
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --resolution 300 \
  --output-dir ./index
```

This writes chunks to `cosplace_parts/` and compiles the final `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Working with the Index Directly (Python)

```python
import numpy as np

# Load compiled index
descriptors = np.load("index/cosplace_descriptors.npy")  # shape: [N, 512]
meta = np.load("index/metadata.npz", allow_pickle=True)

lats = meta["lats"]       # shape: [N] — latitude of each panorama
lons = meta["lons"]       # shape: [N] — longitude
headings = meta["headings"]  # shape: [N] — camera heading in degrees
panoids = meta["panoids"]    # shape: [N] — Google Street View panorama ID

print(f"Index contains {len(lats):,} panoramas")

# Manual cosine similarity search
import torch

query_desc = torch.tensor(descriptors[0])  # Example: use first entry as query
index_tensor = torch.tensor(descriptors)   # [N, 512]

similarities = torch.nn.functional.cosine_similarity(
    query_desc.unsqueeze(0), index_tensor
)  # shape: [N]

top_k = similarities.topk(10)
print("Top 10 candidate indices:", top_k.indices.tolist())
print("Top 10 similarities:", top_k.values.tolist())
```

---

## Haversine Radius Filter

```python
import numpy as np

def haversine_km(lat1, lon1, lat2, lon2):
    """Returns distance in kilometers between two lat/lon points."""
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = (np.sin(dlat / 2) ** 2 +
         np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon / 2) ** 2)
    return R * 2 * np.arcatan2(np.sqrt(a), np.sqrt(1 - a))

# Filter index to a search area before similarity search
meta = np.load("index/metadata.npz", allow_pickle=True)
lats, lons = meta["lats"], meta["lons"]

CENTER_LAT, CENTER_LON = 48.8566, 2.3522
RADIUS_KM = 2.0

distances = haversine_km(CENTER_LAT, CENTER_LON, lats, lons)
mask = distances <= RADIUS_KM

print(f"Candidates in radius: {mask.sum():,} / {len(lats):,}")

# Apply mask to descriptors before search
descriptors = np.load("index/cosplace_descriptors.npy")
filtered_descriptors = descriptors[mask]
filtered_lats = lats[mask]
filtered_lons = lons[mask]
```

---

## Common Patterns

### Pattern: Index a city, search with a photo

```bash
# 1. Launch GUI
python test_super.py

# 2. Create mode → lat=48.8566, lon=2.3522, radius=1, resolution=300 → Create Index
# 3. Search mode → upload photo → Manual → same lat/lon, radius=1 → Run Search
```

### Pattern: Multi-city index, targeted search

```python
# Index Paris and Berlin in the same index (GUI: Create mode, run twice with different coords)
# Search only Berlin by providing Berlin center coords at search time
# The radius filter handles isolation — no manual city tracking needed
```

### Pattern: Resume interrupted indexing

The indexer saves `cosplace_parts/*.npz` incrementally. If interrupted:
- Re-run with the same parameters
- Already-indexed panoramas are skipped
- Progress resumes from last checkpoint

### Pattern: Enable Ultra Mode for night/blurry photos

```
GUI → Search mode → Enable "Ultra Mode" checkbox → Run Search
```

Adds LoFTR dense matching + descriptor hopping + 100m neighborhood expansion. Use when standard pipeline returns confidence < 50%.

---

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global 512-dim descriptor (retrieval) | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoints + descriptors | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoints + descriptors | MPS/CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Detector-free dense matching (Ultra) | All (via kornia) |

---

## Troubleshooting

### GUI is blank on macOS
```bash
brew install python-tk@3.11  # Match your Python version (3.10, 3.11, etc.)
```

### LightGlue import error
```bash
# Must install from GitHub, not PyPI
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR / Ultra Mode not available
```bash
pip install kornia
```

### CUDA out of memory
- Reduce candidate count in search settings
- Use DISK instead of ALIKED (lower keypoint count)
- Reduce batch size in verification stage

### Low confidence results (< 30%)
- Ensure the photo's area is indexed (correct center coords + sufficient radius)
- Try Ultra Mode
- Increase search radius if location is approximate
- Check that the query photo is street-level (not aerial, not interior)

### Indexing is very slow
- Use `build_index.py` instead of GUI for large areas
- Ensure GPU is detected: check for `cuda` or `mps` in startup logs
- Start with small radius (0.5km) to validate pipeline before large runs

### Index seems to have wrong panoramas
- Verify center coordinates (lat/lon order: latitude first, longitude second)
- Confirm radius is in **kilometers**
- Grid resolution 300 is recommended — don't lower it significantly

### Search returns a location far from expected
- Enable spatial consensus (default on) — it prefers clustered matches over outliers
- Check heading refinement is running (top 15 candidates tested at ±45°)
- Use Ultra Mode for descriptor hopping — may find better initial CosPlace match

---

## Data Sources

Netryx is source-agnostic. The pipeline works with panoramas from:
- **Mapillary**
- **KartaView**
- **Google Street View** (used for candidate download during verification)
- Any provider with geo-tagged street-level imagery

The index stores provider-agnostic embeddings; only the verification stage downloads panoramas on-demand.
```
