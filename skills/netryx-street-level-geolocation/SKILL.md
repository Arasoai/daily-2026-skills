```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace + LightGlue to identify GPS coordinates from any street photo
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - identify location from street view photo
  - run netryx geolocation
  - index street view panoramas
  - match street photo to coordinates
  - osint geolocation from image
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, builds a searchable index of CosPlace visual fingerprints, then matches query images against that index using ALIKED/DISK keypoints + LightGlue deep feature matching. Sub-50m accuracy, no cloud required.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install git+https://github.com/cvg/LightGlue.git   # Required
pip install kornia                                       # Optional: Ultra Mode (LoFTR)
```

### Optional: Gemini API key for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"   # From https://aistudio.google.com
```

### GPU support matrix

| Hardware | Backend | Notes |
|---|---|---|
| NVIDIA GPU (4GB+ VRAM) | CUDA | Uses ALIKED (1024 keypoints) |
| Apple Silicon M1+ | MPS | Uses DISK (768 keypoints) |
| CPU only | CPU | Works, significantly slower |

---

## Launch the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11`

---

## Core Workflow

### Step 1 — Create an Index

Index a geographic area before searching. The indexer crawls Street View panoramas, extracts CosPlace 512-dim fingerprints, and saves them incrementally (crash-safe).

**Via GUI:**
1. Select **Create** mode
2. Enter center lat/lon
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

**Index size estimates:**

| Radius | ~Panoramas | Build Time (M2 Max) | Disk |
|---|---|---|---|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

For large datasets, use the standalone builder:

```bash
python build_index.py
```

### Step 2 — Search

**Via GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: provide center coords + radius (faster, use when region is known)
   - **AI Coarse**: Gemini analyzes visual clues to estimate region
4. Click **Run Search** → **Start Full Search**

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading and descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptor vectors
    └── metadata.npz               # lat/lon, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├─ CosPlace → 512-dim descriptor
    ├─ Flipped copy → 512-dim descriptor (catches reversed perspectives)
    │
    ▼
Index Search (cosine similarity, haversine radius filter)
    │ top 500–1000 candidates
    ▼
Download panoramas → Rectilinear crops at 3 FOVs (70°/90°/110°)
    │
    ├─ ALIKED (CUDA) / DISK (MPS/CPU) → keypoints + descriptors
    ├─ LightGlue deep feature matching
    ├─ RANSAC geometric verification → inlier count
    │
    ▼
Heading Refinement: top 15 candidates × ±45° offsets × 3 FOVs
    │
    ├─ Spatial consensus clustering (50m cells)
    ├─ Confidence scoring (clustering + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable for difficult images (night, blur, low texture, heavy compression).

Adds three extra passes:

1. **LoFTR** — detector-free dense matcher, works without clear keypoints
2. **Descriptor hopping** — if best match has <50 inliers, extract CosPlace from the *matched panorama* and re-search the index
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

Enable via the **Ultra Mode** checkbox in the GUI before running search.

---

## Using CosPlace Utilities Directly

```python
# cosplace_utils.py exposes model loading and descriptor extraction

from cosplace_utils import load_cosplace_model, get_cosplace_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model()
model.eval()

# Extract descriptor from an image file
img = Image.open("query_photo.jpg").convert("RGB")
descriptor = get_cosplace_descriptor(model, img)   # shape: (512,)

print(f"Descriptor shape: {descriptor.shape}")
print(f"Descriptor norm: {torch.norm(descriptor):.4f}")  # Should be ~1.0
```

---

## Searching the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lon1, lat2, lon2):
    """Return distance in km between two lat/lon points."""
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """
    Search the compiled index for candidates near a location.

    Returns list of dicts: {panoid, lat, lon, heading, score}
    """
    # Load index
    descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512) float32
    meta = np.load("index/metadata.npz", allow_pickle=True)
    lats = meta["lats"]           # (N,)
    lons = meta["lons"]           # (N,)
    headings = meta["headings"]   # (N,)
    panoids = meta["panoids"]     # (N,)

    # Haversine radius filter
    mask = np.array([
        haversine_km(center_lat, center_lon, lat, lon) <= radius_km
        for lat, lon in zip(lats, lons)
    ], dtype=bool)

    if mask.sum() == 0:
        return []

    # Cosine similarity search (descriptors already L2-normalised)
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    scores = descriptors[mask] @ q                    # dot product = cosine similarity
    indices = np.where(mask)[0]

    # Rank and return top_k
    ranked = np.argsort(scores)[::-1][:top_k]
    results = []
    for r in ranked:
        idx = indices[r]
        results.append({
            "panoid": str(panoids[idx]),
            "lat": float(lats[idx]),
            "lon": float(lons[idx]),
            "heading": float(headings[idx]),
            "score": float(scores[r]),
        })
    return results


# Example usage
from cosplace_utils import load_cosplace_model, get_cosplace_descriptor
from PIL import Image

model = load_cosplace_model()
img = Image.open("query.jpg").convert("RGB")
desc = get_cosplace_descriptor(model, img)

candidates = search_index(
    query_descriptor=desc,
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=2.0,
    top_k=500,
)

print(f"Found {len(candidates)} candidates")
print(f"Top match: {candidates[0]}")
```

---

## Index Management

All cities share **one unified index**. The radius filter at search time isolates regions. No per-city files needed.

```python
# The index auto-builds from cosplace_parts/*.npz chunks
# Trigger a rebuild manually:
import subprocess
subprocess.run(["python", "build_index.py"])

# Check index stats
import numpy as np
descriptors = np.load("index/cosplace_descriptors.npy")
meta = np.load("index/metadata.npz", allow_pickle=True)
print(f"Total indexed panoramas: {len(descriptors)}")
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
```

---

## Common Patterns

### Pattern 1: Batch geolocation of multiple images

```python
from cosplace_utils import load_cosplace_model, get_cosplace_descriptor
from PIL import Image
import json, pathlib

model = load_cosplace_model()

images_dir = pathlib.Path("images_to_geolocate")
results = {}

for img_path in images_dir.glob("*.jpg"):
    img = Image.open(img_path).convert("RGB")
    desc = get_cosplace_descriptor(model, img)

    candidates = search_index(
        query_descriptor=desc,
        center_lat=48.8566,   # Paris centre
        center_lon=2.3522,
        radius_km=5.0,
        top_k=300,
    )

    results[img_path.name] = candidates[:5]   # top 5 per image

with open("batch_results.json", "w") as f:
    json.dump(results, f, indent=2)
```

### Pattern 2: Flip augmentation for better recall

```python
import numpy as np
from PIL import Image, ImageOps
from cosplace_utils import load_cosplace_model, get_cosplace_descriptor

model = load_cosplace_model()
img = Image.open("query.jpg").convert("RGB")

desc_normal  = get_cosplace_descriptor(model, img)
desc_flipped = get_cosplace_descriptor(model, ImageOps.mirror(img))

# Average both descriptors before searching
desc_combined = (desc_normal + desc_flipped) / 2
desc_combined /= np.linalg.norm(desc_combined) + 1e-8

candidates = search_index(desc_combined, 48.8566, 2.3522, radius_km=2.0)
```

### Pattern 3: Check CUDA/MPS availability

```python
import torch

if torch.cuda.is_available():
    device = "cuda"
    print(f"CUDA: {torch.cuda.get_device_name(0)}")
elif torch.backends.mps.is_available():
    device = "mps"
    print("Using Apple MPS")
else:
    device = "cpu"
    print("CPU only — expect slower matching")
```

---

## Troubleshooting

### GUI renders blank on macOS
```bash
brew install python-tk@3.11   # Match your Python version
```

### `ModuleNotFoundError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### `ModuleNotFoundError: No module named 'kornia'` (Ultra Mode)
```bash
pip install kornia
```

### Index search returns zero candidates
- Verify index files exist: `ls index/cosplace_descriptors.npy index/metadata.npz`
- If missing, run `python build_index.py` — it compiles `cosplace_parts/*.npz` into the searchable index
- Check that your search radius actually overlaps the indexed area

### Out of VRAM during matching
- Reduce `top_k` candidates (try 200–300 instead of 500)
- Disable Ultra Mode
- Use CPU fallback: set `CUDA_VISIBLE_DEVICES=""` before running

### Indexing interrupted mid-run
No data loss — chunks are written incrementally to `cosplace_parts/`. Re-run and indexing resumes from the last completed chunk.

### Low confidence / wrong location
1. Enable **Ultra Mode** — adds LoFTR and neighborhood expansion
2. Increase search radius if you're uncertain about the region
3. Try the AI Coarse mode to let Gemini estimate the country/city first
4. Ensure the query image is street-level (not aerial, not interior)

### Slow performance on CPU
Expected — ALIKED/DISK + LightGlue are GPU-accelerated. On CPU, reduce candidates:
```python
top_k = 100   # instead of 500
```
And avoid Ultra Mode (LoFTR is especially slow on CPU).
```
