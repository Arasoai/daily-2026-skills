```markdown
---
name: netryx-street-level-geolocation
description: Expertise in using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - netryx geolocation
  - identify location from street view photo
  - osint geolocation tool
  - reverse geolocate image locally
  - build street view index
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It works by crawling street-view panoramas into a searchable index, then using a three-stage computer vision pipeline (global retrieval → geometric verification → refinement) to match your query image against that index. No cloud APIs required — runs entirely on local hardware.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue matching library
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Platform GPU Support

| Platform | Backend | Notes |
|----------|---------|-------|
| NVIDIA GPU | CUDA | ALIKED extractor, 1024 keypoints, fastest |
| Apple Silicon (M1+) | MPS | DISK extractor, 768 keypoints |
| CPU only | — | Works, significantly slower |

### Optional: Gemini API for AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launching the GUI

```bash
python test_super.py
```

> macOS blank GUI fix: `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### Step 1: Build an Index for a Target Area

In the GUI:
1. Select **Create** mode
2. Enter center latitude/longitude of target area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Index is saved incrementally to `cosplace_parts/` — safe to interrupt and resume.

**Time/size estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

### Step 2: Search (Geolocate an Image)

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose method:
   - **Manual**: Provide approximate center coords + radius
   - **AI Coarse**: Gemini analyzes visual cues to guess region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**

Enable **Ultra Mode** for night shots, blurry images, or low-texture scenes.

---

## Project Structure

```
netryx/
├── test_super.py          # Main entry point — GUI + indexing + search
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (auto-created)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace vectors
    └── metadata.npz               # Coords, headings, panorama IDs
```

---

## Using the Index Programmatically

### Extract a CosPlace Descriptor

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-selects CUDA / MPS / CPU)
model, device = get_cosplace_model()

# Load and process your query image
img = Image.open("query_street.jpg").convert("RGB")

# Get 512-dim fingerprint
descriptor = get_descriptor(model, img, device)
# descriptor.shape → (512,)
```

### Search the Index (Cosine Similarity + Radius Filter)

```python
import numpy as np

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km=2.0, top_k=500):
    """
    Search the Netryx index for visually similar street-view panoramas
    within a geographic radius.
    """
    # Load compiled index
    descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
    meta = np.load("index/metadata.npz", allow_pickle=True)
    lats = meta["lats"]      # (N,)
    lons = meta["lons"]      # (N,)
    headings = meta["headings"]  # (N,)
    panoids = meta["panoids"]    # (N,)

    # --- Stage 1: Radius filter ---
    distances = haversine_km(center_lat, center_lon, lats, lons)
    in_radius = distances <= radius_km
    
    filtered_descriptors = descriptors[in_radius]
    filtered_idx = np.where(in_radius)[0]

    if len(filtered_descriptors) == 0:
        return []

    # --- Stage 2: Cosine similarity ---
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    db = filtered_descriptors / (np.linalg.norm(filtered_descriptors, axis=1, keepdims=True) + 1e-8)
    scores = db @ q  # (M,)

    # Also search with horizontally flipped descriptor
    q_flip = query_descriptor.copy()
    # CosPlace flip: negate the second half of the descriptor
    q_flip[256:] = -q_flip[256:]
    q_flip = q_flip / (np.linalg.norm(q_flip) + 1e-8)
    scores_flip = db @ q_flip

    combined_scores = np.maximum(scores, scores_flip)

    # --- Top-K candidates ---
    top_indices = np.argsort(combined_scores)[::-1][:top_k]
    
    results = []
    for local_idx in top_indices:
        global_idx = filtered_idx[local_idx]
        results.append({
            "panoid": panoids[global_idx],
            "lat": lats[global_idx],
            "lon": lons[global_idx],
            "heading": headings[global_idx],
            "score": float(combined_scores[local_idx]),
        })
    
    return results
```

### Full Search Pipeline (Retrieval + Verification)

```python
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image
import numpy as np

def geolocate_image(image_path, center_lat, center_lon, radius_km=2.0):
    """
    Full Netryx geolocation pipeline.
    Returns best match dict with lat, lon, confidence.
    """
    # Load model
    model, device = get_cosplace_model()
    
    # Extract query descriptor
    img = Image.open(image_path).convert("RGB")
    query_desc = get_descriptor(model, img, device)
    
    # Stage 1: Global retrieval
    candidates = search_index(
        query_desc, 
        center_lat, center_lon, 
        radius_km=radius_km, 
        top_k=500
    )
    
    if not candidates:
        return {"error": "No candidates found in index for this area"}
    
    print(f"Stage 1 complete: {len(candidates)} candidates")
    print(f"Top candidate: {candidates[0]['panoid']} @ "
          f"({candidates[0]['lat']:.6f}, {candidates[0]['lon']:.6f}) "
          f"score={candidates[0]['score']:.3f}")
    
    # Stage 2 & 3: LightGlue verification happens inside test_super.py GUI
    # For programmatic use, refer to the verify_candidates() function in test_super.py
    
    return candidates[0]  # Return top retrieval result

# Example usage
result = geolocate_image(
    image_path="mystery_street.jpg",
    center_lat=48.8566,   # Paris center
    center_lon=2.3522,
    radius_km=3.0
)
print(f"Best match: {result['lat']}, {result['lon']}")
```

---

## Build a Large Index Without the GUI

For indexing large areas (5km+), use the standalone builder:

```bash
python build_index.py \
  --lat 48.8566 \
  --lon 2.3522 \
  --radius 5.0 \
  --grid-resolution 300
```

This is more efficient than the GUI for large-scale indexing — outputs to `cosplace_parts/` automatically.

---

## Understanding the Three-Stage Pipeline

### Stage 1 — Global Retrieval (CosPlace, <1 second)
- Extracts a 512-dim visual fingerprint from the query
- Also tests a horizontally-flipped version (catches reversed perspectives)
- Matrix multiply against all index descriptors → top 500–1000 candidates
- Filtered by haversine distance to search radius

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue, 2–5 min)
- Downloads Street View panorama (8 tiles, stitched)
- Crops at 3 FOVs: 70°, 90°, 110° (handles zoom mismatches)
- Extracts local keypoints with ALIKED (CUDA) or DISK (MPS/CPU)
- LightGlue matches query keypoints to candidate keypoints
- RANSAC filters to geometrically consistent inliers
- Candidate with most inliers = best match

### Stage 3 — Refinement
- Tests ±45° heading offsets at 15° steps for top 15 candidates
- Spatial consensus: clusters matches into 50m cells, prefers clusters over outliers
- Confidence score: cluster quality + uniqueness ratio vs runner-up

### Ultra Mode (difficult images)
```
Enable in GUI → Ultra Mode checkbox
```
Adds:
- **LoFTR**: Detector-free dense matching for blurry/low-contrast images
- **Descriptor hopping**: Re-searches index using the matched panorama's descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Multi-City Index Strategy

The index is unified — all cities go into the same files. Search radius filters by geography automatically:

```python
# Paris results only
candidates = search_index(desc, 48.8566, 2.3522, radius_km=5.0)

# London results only  
candidates = search_index(desc, 51.5074, -0.1278, radius_km=5.0)

# No city selection needed — coordinates + radius handle everything
```

---

## Common Patterns

### Batch Geolocate Multiple Images

```python
import os
from cosplace_utils import get_cosplace_model, get_descriptor
from PIL import Image

model, device = get_cosplace_model()

image_dir = "./images_to_geolocate"
results = {}

for filename in os.listdir(image_dir):
    if not filename.lower().endswith((".jpg", ".jpeg", ".png")):
        continue
    
    path = os.path.join(image_dir, filename)
    img = Image.open(path).convert("RGB")
    descriptor = get_descriptor(model, img, device)
    
    candidates = search_index(
        descriptor,
        center_lat=48.8566,
        center_lon=2.3522,
        radius_km=5.0,
        top_k=10  # Just top-10 for batch screening
    )
    
    results[filename] = candidates[0] if candidates else None
    print(f"{filename}: {results[filename]}")
```

### Check Index Coverage

```python
import numpy as np

meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]
lons = meta["lons"]

print(f"Total indexed panoramas: {len(lats):,}")
print(f"Lat range: {lats.min():.4f} → {lats.max():.4f}")
print(f"Lon range: {lons.min():.4f} → {lons.max():.4f}")
print(f"Index files: cosplace_descriptors.npy, metadata.npz")
```

### Verify Index is Built

```python
import os

parts_dir = "cosplace_parts"
index_dir = "index"

parts_count = len([f for f in os.listdir(parts_dir) if f.endswith(".npz")]) if os.path.exists(parts_dir) else 0
index_ready = os.path.exists(os.path.join(index_dir, "cosplace_descriptors.npy"))

print(f"Raw parts: {parts_count} chunks in cosplace_parts/")
print(f"Compiled index ready: {index_ready}")

if not index_ready and parts_count > 0:
    print("→ Run build_index.py or use GUI 'Auto-build' to compile parts into searchable index")
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # match your Python version
```

### `ModuleNotFoundError: No module named 'lightglue'`
```bash
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA out of memory
- Reduce `top_k` candidates (try 200–300 instead of 500)
- Use `DISK` instead of `ALIKED` (lower memory, fewer keypoints)
- Close other GPU-using processes

### MPS errors on Apple Silicon
```bash
# Force CPU if MPS is unstable
export PYTORCH_ENABLE_MPS_FALLBACK=1
```

### Index search returns no results
- Verify index covers your search area: check lat/lon ranges (see "Check Index Coverage" above)
- Ensure `index/cosplace_descriptors.npy` and `index/metadata.npz` both exist
- Increase `radius_km` — if the area wasn't indexed, no results will appear

### Indexing interrupted / incomplete
Safe to re-run — indexing resumes from the last saved chunk in `cosplace_parts/`. Do not delete `cosplace_parts/` between runs.

### Low confidence / wrong location
- Enable **Ultra Mode** in the GUI
- Ensure the index is dense enough for the area (try smaller grid resolution)
- Verify the query image is actual street-level (not aerial or interior)
- Increase search radius if approximate location is uncertain

---

## Model Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (512-dim) | Global visual fingerprint | All |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | All (needs `kornia`) |

---

## Key File Paths

```
cosplace_parts/*.npz         ← Raw embedding chunks from indexing
index/cosplace_descriptors.npy  ← Compiled descriptor matrix (N×512)
index/metadata.npz           ← lat, lon, heading, panoid arrays
test_super.py                ← Main app (GUI + full pipeline)
cosplace_utils.py            ← Model loading + descriptor extraction
build_index.py               ← CLI index builder for large areas
```
```
