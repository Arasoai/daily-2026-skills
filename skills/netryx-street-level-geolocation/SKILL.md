```markdown
---
name: netryx-street-level-geolocation
description: Expertise in using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - reverse image geolocation
  - netryx geolocation
  - index street view panoramas
  - osint geolocation tool
  - locate where a photo was taken
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation tool that identifies the exact GPS coordinates of any street-level photograph. It crawls and indexes Street View panoramas, then matches a query image against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature matching (ALIKED/DISK + LightGlue), and spatial refinement. It achieves sub-50m accuracy without requiring landmarks or internet image presence.

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

# Optional: LoFTR for Ultra Mode (difficult images)
pip install kornia
```

### Optional: Gemini API key for AI Coarse geolocation hint

```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU VRAM | 4 GB | 8 GB+ |
| RAM | 8 GB | 16 GB+ |
| Storage | 10 GB | 50 GB+ |
| Python | 3.9+ | 3.10+ |

**GPU backends:**
- **NVIDIA**: CUDA (uses ALIKED, 1024 keypoints)
- **Apple Silicon**: MPS / Metal (uses DISK, 768 keypoints)
- **CPU**: Supported but slow

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
├── test_super.py          # Main GUI application + full pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # Standalone large-scale index builder
├── requirements.txt
├── cosplace_parts/        # Raw .npz embedding chunks (created during indexing)
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Core Workflow

### Step 1: Create an Index (Index an Area)

In the GUI:
1. Select **Create** mode
2. Enter center latitude/longitude of the area
3. Set radius (km) and grid resolution (default: 300)
4. Click **Create Index**

Indexing crawls Street View panoramas, extracts CosPlace fingerprints, and saves them to `cosplace_parts/`. The index auto-builds into `index/` when complete. **Indexing is resumable** — safe to interrupt and restart.

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

### Step 2: Search

In the GUI:
1. Select **Search** mode
2. Upload a street-level photo
3. Choose:
   - **Manual**: Provide center coordinates + radius if approximate location is known
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score on a map

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace → 512-dim global descriptor
    ├── Flipped CosPlace → catches reversed perspectives
    │
    ▼
Index Search (cosine similarity, haversine radius filter)
    │
    ├── Top 500–1000 visually similar panorama candidates
    │
    ▼
Download Panoramas (8 tiles, stitched) → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) / DISK (MPS/CPU) → local keypoints + descriptors
    ├── LightGlue → deep feature matching
    ├── RANSAC → geometric verification (inlier count)
    │
    ▼
Heading Refinement (±45°, 15° steps, top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    ├── Confidence scoring (cluster strength + uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

**Stage timing (typical hardware):**
- Stage 1 (CosPlace retrieval): <1 second
- Stage 2 (LightGlue verification, 300–500 candidates): 2–5 minutes
- Ultra Mode adds ~50–100% overhead

---

## Ultra Mode

Enable **Ultra Mode** checkbox in GUI for difficult images (night, blur, low texture).

Adds three enhancements:
1. **LoFTR** — detector-free dense matching (handles blur/low-contrast)
2. **Descriptor hopping** — if best match has <50 inliers, re-searches index using the matched panorama's descriptor instead of the degraded query
3. **Neighborhood expansion** — searches all panoramas within 100m of best match

---

## Using the Index Programmatically

### Extract a CosPlace Descriptor

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"

model = load_cosplace_model(device=device)

image = Image.open("query_photo.jpg").convert("RGB")
descriptor = extract_descriptor(model, image, device=device)
# Returns: numpy array, shape (512,)
print(descriptor.shape)  # (512,)
```

### Load the Index and Search

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

# Load index
descriptors = np.load("index/cosplace_descriptors.npy")   # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]       # (N,)
lons = meta["lons"]       # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """
    Search the index for the top-k most similar panoramas within radius_km
    of (center_lat, center_lon).
    """
    # Radius mask
    distances = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i])
        for i in range(len(lats))
    ])
    mask = distances <= radius_km
    
    if not mask.any():
        return []

    # Cosine similarity (descriptors should be L2-normalized)
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    d = descriptors[mask]
    d_norm = d / (np.linalg.norm(d, axis=1, keepdims=True) + 1e-8)
    
    similarities = d_norm @ q  # (M,)
    
    masked_indices = np.where(mask)[0]
    top_local = np.argsort(similarities)[::-1][:top_k]
    top_global = masked_indices[top_local]
    
    results = []
    for idx in top_global:
        results.append({
            "panoid": panoids[idx],
            "lat": lats[idx],
            "lon": lons[idx],
            "heading": headings[idx],
            "similarity": float(similarities[top_local[list(top_global).index(idx)]])
        })
    return results

# Example usage
candidates = search_index(descriptor, center_lat=48.8566, center_lon=2.3522, radius_km=2.0)
print(f"Found {len(candidates)} candidates")
print(candidates[0])
```

### Feature Matching with LightGlue

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
from PIL import Image
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Use ALIKED on CUDA, DISK on MPS/CPU (matches Netryx behavior)
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

def extract_features(image_path):
    image = load_image(image_path).to(device)
    with torch.no_grad():
        feats = extractor.extract(image)
    return feats

def match_images(feats0, feats1):
    with torch.no_grad():
        matches = matcher({"image0": feats0, "image1": feats1})
    feats0_, feats1_, matches_ = [rbd(x) for x in [feats0, feats1, matches]]
    matched_kp0 = feats0_["keypoints"][matches_["matches"][..., 0]]
    matched_kp1 = feats1_["keypoints"][matches_["matches"][..., 1])
    scores = matches_["matching_scores"]
    return matched_kp0, matched_kp1, scores

# Usage
feats_query = extract_features("query_photo.jpg")
feats_candidate = extract_features("candidate_panorama_crop.jpg")
kp0, kp1, scores = match_images(feats_query, feats_candidate)
print(f"Matched keypoints: {len(kp0)}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kp0, kp1, threshold=4.0):
    """
    Returns number of geometric inliers using RANSAC homography.
    Higher inlier count = more confident match.
    """
    if len(kp0) < 4:
        return 0
    
    pts0 = kp0.cpu().numpy()
    pts1 = kp1.cpu().numpy()
    
    _, mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, threshold)
    
    if mask is None:
        return 0
    return int(mask.sum())

inliers = ransac_verify(kp0, kp1)
print(f"RANSAC inliers: {inliers}")
# >50 inliers: strong match
# >100 inliers: very confident match
```

---

## Multi-Index Strategy (Multiple Cities)

All embeddings go into one unified index. Use radius filtering to scope searches:

```python
# Index Paris (run once)
# GUI: Create, center=48.8566,2.3522, radius=5km

# Index London (run once)  
# GUI: Create, center=51.5074,-0.1278, radius=5km

# Both are stored in the same index/cosplace_descriptors.npy

# Search only Paris
paris_results = search_index(desc, 48.8566, 2.3522, radius_km=5.0)

# Search only London
london_results = search_index(desc, 51.5074, -0.1278, radius_km=5.0)
```

---

## Build Index from Parts (Standalone)

For large datasets, use `build_index.py` directly instead of the GUI:

```bash
python build_index.py
```

This compiles all `cosplace_parts/*.npz` chunks into `index/cosplace_descriptors.npy` + `index/metadata.npz`.

---

## Common Patterns

### Batch Geolocate Multiple Images

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import numpy as np
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"
model = load_cosplace_model(device=device)

image_paths = ["img1.jpg", "img2.jpg", "img3.jpg"]
center_lat, center_lon = 48.8566, 2.3522
radius_km = 3.0

for path in image_paths:
    img = Image.open(path).convert("RGB")
    desc = extract_descriptor(model, img, device=device)
    candidates = search_index(desc, center_lat, center_lon, radius_km)
    
    if candidates:
        best = candidates[0]
        print(f"{path}: lat={best['lat']:.6f}, lon={best['lon']:.6f}, "
              f"similarity={best['similarity']:.3f}, panoid={best['panoid']}")
    else:
        print(f"{path}: No candidates found in radius")
```

### Check Flip Augmentation (Netryx core trick)

```python
import numpy as np
from PIL import Image, ImageOps

def get_dual_descriptor(model, image_path, device):
    """Extract descriptor + flipped descriptor, return both."""
    img = Image.open(image_path).convert("RGB")
    img_flipped = ImageOps.mirror(img)
    
    desc = extract_descriptor(model, img, device=device)
    desc_flip = extract_descriptor(model, img_flipped, device=device)
    return desc, desc_flip

desc, desc_flip = get_dual_descriptor(model, "query.jpg", device)

# Search with both, merge results, deduplicate by panoid
results_normal = search_index(desc, lat, lon, radius_km, top_k=500)
results_flipped = search_index(desc_flip, lat, lon, radius_km, top_k=500)

# Merge and deduplicate
seen = set()
merged = []
for r in results_normal + results_flipped:
    if r["panoid"] not in seen:
        seen.add(r["panoid"])
        merged.append(r)

merged.sort(key=lambda x: x["similarity"], reverse=True)
top_500 = merged[:500]
```

### Spatial Consensus Clustering

```python
from collections import defaultdict
import numpy as np

def cluster_by_location(candidates, cell_size_m=50):
    """
    Group candidates into ~50m grid cells.
    Returns clusters sorted by size (largest = most likely true location).
    """
    clusters = defaultdict(list)
    
    # ~50m in degrees at equator
    deg_per_meter = 1 / 111_000
    cell_deg = cell_size_m * deg_per_meter
    
    for c in candidates:
        cell = (round(c["lat"] / cell_deg), round(c["lon"] / cell_deg))
        clusters[cell].append(c)
    
    sorted_clusters = sorted(clusters.values(), key=len, reverse=True)
    return sorted_clusters

clusters = cluster_by_location(top_500)
best_cluster = clusters[0]
print(f"Largest cluster: {len(best_cluster)} matches")
centroid_lat = np.mean([c["lat"] for c in best_cluster])
centroid_lon = np.mean([c["lon"] for c in best_cluster])
print(f"Consensus location: {centroid_lat:.6f}, {centroid_lon:.6f}")
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # Match your Python version
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED: use 512 instead of 1024
- Process candidates in smaller batches
- Switch to DISK which uses fewer keypoints (768 default)

### No candidates found in radius
- Increase `radius_km` in the search
- Verify the index covers the expected area (check `metadata.npz` lat/lon range)
- Confirm index was built: `ls index/` should show `cosplace_descriptors.npy` and `metadata.npz`

### Low inlier count (<20) on all candidates
- Try **Ultra Mode** (LoFTR + descriptor hopping)
- Check image quality — motion blur, extreme darkness, or textureless scenes reduce keypoint detection
- Expand the search radius in case the index doesn't cover the exact area

### LightGlue import error
```bash
# Reinstall from source
pip uninstall lightglue
pip install git+https://github.com/cvg/LightGlue.git
```

### Indexing stalls or crashes
- Indexing is resumable — rerun `python test_super.py` and restart the create process
- Parts already saved in `cosplace_parts/` are skipped automatically

### MPS (Apple Silicon) errors
```python
# Force CPU if MPS causes issues
import os
os.environ["PYTORCH_ENABLE_MPS_FALLBACK"] = "1"
```

---

## Key Thresholds Reference

| Metric | Weak | Good | Strong |
|--------|------|------|--------|
| CosPlace similarity | <0.5 | 0.6–0.75 | >0.8 |
| LightGlue matches (raw) | <20 | 50–150 | >200 |
| RANSAC inliers | <10 | 30–80 | >100 |
| Cluster size (candidates) | 1–2 | 5–10 | >15 |

---

## Model References

| Model | Role | Source |
|-------|------|--------|
| CosPlace | Global 512-dim place descriptor | [github.com/gmberton/cosplace](https://github.com/gmberton/cosplace) |
| ALIKED | Local keypoints (CUDA) | [github.com/naver/alike](https://github.com/naver/alike) |
| DISK | Local keypoints (MPS/CPU) | [github.com/cvlab-epfl/disk](https://github.com/cvlab-epfl/disk) |
| LightGlue | Deep feature matching | [github.com/cvg/LightGlue](https://github.com/cvg/LightGlue) |
| LoFTR | Dense matching (Ultra Mode) | via `kornia.feature.LoFTR` |
```
