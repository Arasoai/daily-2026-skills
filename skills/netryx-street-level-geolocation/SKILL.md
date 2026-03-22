```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace + LightGlue + ALIKED/DISK to identify GPS coordinates from any street photo
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - identify location from street view photo
  - run netryx geolocation
  - build a street view index
  - match photo to map coordinates
  - osint geolocation from image
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It indexes Street View panoramas into a searchable vector database, then matches a query image against that index using a three-stage computer vision pipeline: **CosPlace** (global retrieval) → **ALIKED/DISK + LightGlue** (geometric verification) → **refinement + confidence scoring**. No cloud APIs required for search — only indexing requires internet access to fetch panoramas.

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

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Platform notes

| Platform | Feature extractor used | Notes |
|---|---|---|
| NVIDIA GPU (CUDA) | ALIKED (1024 keypoints) | Fastest, best quality |
| Apple Silicon (MPS) | DISK (768 keypoints) | Good performance on M1+ |
| CPU | DISK | Works, significantly slower |

### Optional: Gemini API for AI Coarse mode

```bash
export GEMINI_API_KEY="your_key_here"   # from https://aistudio.google.com
```

Only needed for AI-assisted region guessing — manual coordinate entry works without it.

---

## Launch the GUI

```bash
python test_super.py
```

On macOS, if the GUI renders blank:
```bash
brew install python-tk@3.11   # match your Python version
```

---

## Core Workflow

### Step 1: Index an Area

Indexing crawls Street View panoramas in a geographic area and stores CosPlace 512-dim fingerprints.

**In GUI:**
1. Select **Create** mode
2. Enter center lat/lon (e.g. `48.8566, 2.3522` for Paris)
3. Set radius in km (start with `0.5` for testing)
4. Set grid resolution (`300` recommended — don't change this)
5. Click **Create Index**

**Index size guide:**

| Radius | ~Panoramas | Build time (M2 Max) | Disk size |
|---|---|---|---|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

Indexing is **resumable** — safe to interrupt and restart.

### Step 2: Search

**In GUI:**
1. Select **Search** mode
2. Upload a street-level photo (JPEG/PNG)
3. Choose search method:
   - **Manual**: Enter center coordinates + radius (use when you have a rough idea of location)
   - **AI Coarse**: Let Gemini infer region from visual clues (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result displays GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone high-performance index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (written during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace vectors
    └── metadata.npz               # Lat/lon, headings, panorama IDs
```

---

## How the Pipeline Works

### Stage 1: Global Retrieval (CosPlace)

```
Query image → 512-dim CosPlace descriptor
            + flipped image descriptor (catches reversed perspectives)
→ Cosine similarity against full index
→ Haversine radius filter (your specified search area)
→ Top 500–1000 candidate panoramas
```

Runs in under 1 second — single matrix multiplication.

### Stage 2: Geometric Verification (ALIKED/DISK + LightGlue)

```
For each candidate panorama:
  1. Download 8 Street View tiles → stitch panorama
  2. Crop rectilinear view at indexed heading
  3. Generate crops at 3 FOVs: 70°, 90°, 110°
  4. Extract keypoints: ALIKED (CUDA) or DISK (MPS/CPU)
  5. LightGlue deep feature matching vs. query keypoints
  6. RANSAC geometric verification → count inliers
→ Candidate with most verified inliers = best match
```

Processes 300–500 candidates in 2–5 minutes on GPU.

### Stage 3: Refinement

```
Top 15 candidates:
  - Heading refinement: test ±45° at 15° steps × 3 FOVs
  - Spatial consensus: cluster matches into 50m cells
  - Confidence scoring: clustering density + uniqueness ratio
→ Final GPS coordinates
```

---

## Ultra Mode

Enable for difficult images (night shots, motion blur, low texture):

```
Standard pipeline
  ↓ (if match quality < 50 inliers)
LoFTR: detector-free dense matching (handles blur/low-contrast)
  ↓
Descriptor hopping: re-search index using matched panorama's descriptor
  ↓
Neighborhood expansion: search all panoramas within 100m of best match
```

Enable via **Ultra Mode** checkbox in GUI before running search.

---

## Index Architecture

The index is **unified and coordinate-filtered** — index multiple cities into one store:

```python
# Conceptual: how search uses radius filtering
# All cities coexist in cosplace_descriptors.npy + metadata.npz
# Radius filter (haversine) restricts results to your search area

# Paris search: center=(48.8566, 2.3522), radius=5km → only Paris results
# London search: center=(51.5074, -0.1278), radius=10km → only London results
```

**Build flow:**
```
cosplace_parts/*.npz      ← written incrementally during indexing
       ↓
cosplace_descriptors.npy  ← compiled searchable matrix
metadata.npz              ← lat/lon, headings, panoid IDs
```

---

## Working with the Index Programmatically

### Extract a CosPlace Descriptor

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
model, device = load_cosplace_model()

# Extract 512-dim descriptor from an image
img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)
# descriptor.shape → (512,)
```

### Search the Index Manually

```python
import numpy as np

# Load pre-built index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]      # (N,)
lons = meta["lons"]      # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

def haversine_km(lat1, lon1, lat2, lon2):
    """Vectorized haversine distance in km."""
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """Return top-k candidates within radius sorted by cosine similarity."""
    # Radius filter
    dists = haversine_km(center_lat, center_lon, lats, lons)
    in_radius = np.where(dists <= radius_km)[0]
    
    if len(in_radius) == 0:
        return []
    
    # Cosine similarity (descriptors assumed L2-normalized)
    local_descs = descriptors[in_radius]  # (M, 512)
    sims = local_descs @ query_descriptor  # (M,)
    
    top_idx = np.argsort(sims)[::-1][:top_k]
    candidates = in_radius[top_idx]
    
    return [
        {
            "panoid": panoids[i],
            "lat": lats[i],
            "lon": lons[i],
            "heading": headings[i],
            "similarity": float(sims[top_idx[j]])
        }
        for j, i in enumerate(candidates)
    ]

# Usage
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image

model, device = load_cosplace_model()
img = Image.open("query.jpg").convert("RGB")
desc = get_descriptor(model, img, device)

# Also check flipped version
desc_flip = get_descriptor(model, img.transpose(Image.FLIP_LEFT_RIGHT), device)

# Combine scores (take max similarity)
results = search_index(desc, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
results_flip = search_index(desc_flip, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
```

### Build Index Programmatically (Large Areas)

For large-scale indexing, use the standalone builder:

```bash
# build_index.py handles high-performance parallel indexing
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 5.0 \
    --resolution 300 \
    --output ./index
```

---

## Common Patterns

### Pattern 1: OSINT — Geolocating a Photo with Known Rough Region

```
1. Index the suspected area (city/district level, 2–5 km radius)
2. Upload photo in Search mode
3. Use Manual mode with the city center + same radius
4. If result confidence is low, expand radius or try Ultra Mode
```

### Pattern 2: Geolocating with No Prior Knowledge

```
1. Use AI Coarse mode (requires GEMINI_API_KEY)
   → Gemini reads visible text, architecture, vegetation, vehicles
   → Returns a probable region with lat/lon estimate
2. Index that region if not already indexed
3. Run standard search within suggested radius
```

### Pattern 3: Conflict/Event Monitoring (Recurring)

```
1. Pre-index all relevant areas once (store in unified index)
2. For each new photo: run search with appropriate region + radius
3. Compare result coordinates against known POIs
4. Use Ultra Mode for footage captured in poor conditions
```

### Pattern 4: Improving Match Quality

```python
# If standard pipeline returns low inlier count (<50), try:

# 1. Enable Ultra Mode in GUI (LoFTR + descriptor hopping)
# 2. Try expanding search radius
# 3. Try flipped image as query (handles mirrored sources)
# 4. Ensure the area is indexed at sufficient density (radius 0.5km+)
```

---

## Troubleshooting

### GUI appears blank on macOS

```bash
brew install python-tk@3.11    # match your Python version exactly
# Then re-activate venv and relaunch
```

### `ImportError: No module named 'lightglue'`

```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### `kornia` not found (Ultra Mode fails)

```bash
pip install kornia
```

### CUDA out of memory

- Reduce `top_k` candidates (500 → 200) in search settings
- Use DISK instead of ALIKED (lower keypoint count: 768 vs 1024)
- Close other GPU processes

### Index search returns 0 candidates

```python
# Verify your index covers the search area:
import numpy as np
meta = np.load("index/metadata.npz", allow_pickle=True)
print(f"Index contains {len(meta['lats'])} panoramas")
print(f"Lat range: {meta['lats'].min():.4f} – {meta['lats'].max():.4f}")
print(f"Lon range: {meta['lons'].min():.4f} – {meta['lons'].max():.4f}")
# If your search center is outside these bounds, increase radius or re-index
```

### Indexing interrupted — how to resume

Just re-run the same Create Index command with the same parameters. The system checks `cosplace_parts/` for existing chunks and skips already-processed panoramas.

### Low confidence on valid matches

- Verify the indexed area actually contains the photo location
- Enable Ultra Mode
- Try AI Coarse mode to reconfirm the region
- Check that grid resolution wasn't changed from default (300)

---

## Models Reference

| Model | Role | Activated when |
|---|---|---|
| CosPlace | Global 512-dim place fingerprint | Always (Stage 1) |
| ALIKED | Local keypoints + descriptors | CUDA available |
| DISK | Local keypoints + descriptors | MPS or CPU |
| LightGlue | Deep feature matching | Always (Stage 2) |
| LoFTR | Dense detector-free matching | Ultra Mode enabled |

---

## Performance Expectations

| Hardware | Stage 2 speed (300 candidates) | Accuracy |
|---|---|---|
| NVIDIA RTX 3090 | ~1–2 min | Best (ALIKED 1024kp) |
| Apple M2 Max | ~3–5 min | Good (DISK 768kp) |
| CPU only | ~30–60 min | Good (DISK 768kp) |

Accuracy at sufficient index density: **sub-50m** for clear street photos in indexed areas.
```
