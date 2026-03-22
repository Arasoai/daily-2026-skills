```markdown
---
name: netryx-street-level-geolocation
description: Local-first street-level geolocation engine using CosPlace + LightGlue to match any street photo to precise GPS coordinates
triggers:
  - geolocate a street photo
  - find GPS coordinates from an image
  - street level geolocation
  - match photo to location
  - build a geolocation index
  - run netryx search
  - visual place recognition pipeline
  - identify location from street view image
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, indexes them with CosPlace visual fingerprints, then matches query images using ALIKED/DISK keypoint extraction + LightGlue geometric verification — achieving sub-50m accuracy without internet search or landmark recognition.

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

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

### GPU Backend

- **NVIDIA**: CUDA — uses ALIKED (1024 keypoints)
- **Mac M1+**: MPS — uses DISK (768 keypoints)
- **CPU**: Works, significantly slower

### Optional: Gemini API for AI Coarse Mode

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

### Step 1 — Create an Index

Index a geographic area before searching. The system crawls Street View panoramas and stores CosPlace fingerprints.

**Via GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius (start 0.5–1km for testing)
4. Set grid resolution (default: 300 — do not change)
5. Click **Create Index**

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hrs | ~250 MB |
| 5 km | ~30,000 | 8–12 hrs | ~3 GB |
| 10 km | ~100,000 | 24–48 hrs | ~7 GB |

Index saves incrementally — interruptions resume automatically.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Enter known approximate coordinates + radius
   - **AI Coarse**: Gemini analyzes visual clues to estimate region
4. Click **Run Search** → **Start Full Search**

---

## Project Structure

```
netryx/
├── test_super.py          # Main app: GUI + indexing + search pipeline
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Architecture

```
Query Image
    │
    ├── CosPlace descriptor (512-dim)
    ├── Flipped descriptor (catches reversed perspectives)
    │
    ▼
Index Search (cosine similarity + haversine radius filter)
    │
    └── Top 500–1000 candidates
    │
    ▼
Download Panoramas → Crop at 3 FOVs (70°, 90°, 110°)
    │
    ├── ALIKED (CUDA) or DISK (MPS/CPU) keypoint extraction
    ├── LightGlue deep feature matching
    ├── RANSAC geometric verification
    │
    ▼
Heading Refinement (±45°, 15° steps, top 15 candidates)
    │
    ├── Spatial consensus clustering (50m cells)
    ├── Confidence scoring (uniqueness ratio)
    │
    ▼
📍 GPS Coordinates + Confidence Score
```

---

## Ultra Mode

Enable for difficult images: night shots, blur, low texture.

Adds:
- **LoFTR** — detector-free dense matching (handles blur/low-contrast)
- **Descriptor hopping** — re-searches index using matched panorama's clean descriptor
- **Neighborhood expansion** — searches all panoramas within 100m of best match

Enable via GUI checkbox before running search.

---

## Index Architecture

All areas share a single unified index. Radius filtering handles area scoping at search time:

```python
# Conceptual: index stores everything, search filters by coordinates
search(
    query_image="photo.jpg",
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=5.0       # Only returns Paris results
)
```

You can index Paris, London, Tokyo into the same index — no city selection needed.

**Data flow:**

```
Create Mode:
  Grid points → Street View API → Panoramas → CosPlace → cosplace_parts/*.npz

Auto-build:
  cosplace_parts/*.npz → index/cosplace_descriptors.npy + index/metadata.npz

Search Mode:
  Query → CosPlace → Index (radius-filtered) → Download → ALIKED/LightGlue → Result
```

---

## Models Reference

| Model | Role | Backend |
|-------|------|---------|
| [CosPlace](https://github.com/gmberton/cosplace) | Global visual descriptor (512-dim) | All |
| [ALIKED](https://github.com/naver/alike) | Local keypoint extraction | CUDA only |
| [DISK](https://github.com/cvlab-epfl/disk) | Local keypoint extraction | MPS/CPU |
| [LightGlue](https://github.com/cvg/LightGlue) | Deep feature matching | All |
| [LoFTR](https://github.com/zju3dv/LoFTR) | Dense matching (Ultra Mode) | All (via kornia) |

---

## Code Examples

### Extract a CosPlace Descriptor

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

model = load_cosplace_model()  # loads pretrained CosPlace weights

img = Image.open("street_photo.jpg")
descriptor = extract_descriptor(model, img)  # returns np.ndarray shape (512,)

# Also extract flipped for reversed-perspective robustness
import PIL.ImageOps
img_flipped = PIL.ImageOps.mirror(img)
descriptor_flipped = extract_descriptor(model, img_flipped)
```

### Cosine Similarity Search Against Index

```python
import numpy as np

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")   # shape (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]    # shape (N,)
lons = meta["lons"]    # shape (N,)
headings = meta["headings"]
panoids = meta["panoids"]

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = np.radians(lat2 - lat1)
    dlon = np.radians(lon2 - lon1)
    a = np.sin(dlat/2)**2 + np.cos(np.radians(lat1)) * np.cos(np.radians(lat2)) * np.sin(dlon/2)**2
    return R * 2 * np.arcsin(np.sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km=5.0, top_k=500):
    # Radius filter
    dists = haversine_km(center_lat, center_lon, lats, lons)
    mask = dists <= radius_km
    
    filtered_descriptors = descriptors[mask]
    filtered_indices = np.where(mask)[0]
    
    if len(filtered_indices) == 0:
        return []
    
    # Cosine similarity (descriptors assumed L2-normalized)
    query_norm = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    sims = filtered_descriptors @ query_norm
    
    top_k_local = min(top_k, len(sims))
    top_local_idx = np.argsort(sims)[::-1][:top_k_local]
    top_global_idx = filtered_indices[top_local_idx]
    
    results = []
    for i, global_idx in enumerate(top_global_idx):
        results.append({
            "rank": i + 1,
            "panoid": panoids[global_idx],
            "lat": lats[global_idx],
            "lon": lons[global_idx],
            "heading": headings[global_idx],
            "similarity": float(sims[top_local_idx[i]])
        })
    
    return results

# Usage
candidates = search_index(descriptor, center_lat=48.8566, center_lon=2.3522, radius_km=5.0)
print(f"Found {len(candidates)} candidates")
print(f"Top match: {candidates[0]}")
```

### LightGlue Feature Matching

```python
import torch
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import numpy as np

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps" if torch.backends.mps.is_available() else "cpu")

# Choose extractor based on hardware
if device.type == "cuda":
    extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
    matcher = LightGlue(features="aliked").eval().to(device)
else:
    extractor = DISK(max_num_keypoints=768).eval().to(device)
    matcher = LightGlue(features="disk").eval().to(device)

def match_images(query_path, candidate_path):
    query_img = load_image(query_path).to(device)
    candidate_img = load_image(candidate_path).to(device)
    
    with torch.no_grad():
        feats0 = extractor.extract(query_img)
        feats1 = extractor.extract(candidate_img)
        matches_data = matcher({"image0": feats0, "image1": feats1})
    
    # Remove batch dimension
    feats0, feats1, matches_data = rbd(feats0), rbd(feats1), rbd(matches_data)
    
    matches = matches_data["matches"]  # shape (M, 2)
    kpts0 = feats0["keypoints"][matches[:, 0]]
    kpts1 = feats1["keypoints"][matches[:, 1]]
    
    return kpts0.cpu().numpy(), kpts1.cpu().numpy(), len(matches)

# Usage
kpts_query, kpts_candidate, num_matches = match_images("query.jpg", "candidate.jpg")
print(f"Raw matches: {num_matches}")
```

### RANSAC Geometric Verification

```python
import cv2
import numpy as np

def ransac_verify(kpts0, kpts1, threshold=4.0):
    """
    Returns number of inlier matches after RANSAC homography estimation.
    Higher inliers = more likely to be the same location.
    """
    if len(kpts0) < 4:
        return 0
    
    pts0 = kpts0.astype(np.float32)
    pts1 = kpts1.astype(np.float32)
    
    H, mask = cv2.findHomography(pts0, pts1, cv2.RANSAC, threshold)
    
    if mask is None:
        return 0
    
    inliers = int(mask.sum())
    return inliers

# Full verification pipeline
kpts0, kpts1, raw_matches = match_images("query.jpg", "candidate.jpg")
inliers = ransac_verify(kpts0, kpts1)
print(f"Inliers after RANSAC: {inliers}")
# > 30 inliers = likely match
# > 80 inliers = high confidence match
```

### Spatial Consensus Clustering

```python
import numpy as np
from collections import defaultdict

def cluster_candidates(candidates, cell_size_m=50):
    """
    Group candidates into ~50m spatial cells.
    Prefer clusters over isolated high-inlier outliers.
    """
    # Convert ~50m to approximate degrees
    deg_per_meter = 1 / 111320
    cell_deg = cell_size_m * deg_per_meter
    
    clusters = defaultdict(list)
    
    for c in candidates:
        cell_lat = round(c["lat"] / cell_deg) * cell_deg
        cell_lon = round(c["lon"] / cell_deg) * cell_deg
        clusters[(cell_lat, cell_lon)].append(c)
    
    # Score each cluster: sum of inliers weighted by cluster size
    best_cluster = None
    best_score = -1
    
    for cell, members in clusters.items():
        total_inliers = sum(m["inliers"] for m in members)
        size_bonus = len(members) ** 1.5  # prefer clusters
        score = total_inliers * size_bonus
        if score > best_score:
            best_score = score
            best_cluster = members
    
    # Best candidate = highest inliers within best cluster
    return max(best_cluster, key=lambda x: x["inliers"])

# Usage after matching all candidates
matched_candidates = [
    {"lat": 48.8566, "lon": 2.3522, "panoid": "abc", "inliers": 120},
    {"lat": 48.8567, "lon": 2.3523, "panoid": "def", "inliers": 95},
    {"lat": 48.9000, "lon": 2.4000, "panoid": "xyz", "inliers": 150},  # outlier
]

best = cluster_candidates(matched_candidates)
print(f"Best location: {best['lat']}, {best['lon']} ({best['inliers']} inliers)")
```

### Multi-FOV Crop Generation

```python
from PIL import Image
import numpy as np

def generate_fov_crops(panorama: Image.Image, heading_deg: float, fovs=(70, 90, 110)):
    """
    Generate rectilinear crops from a panorama at multiple fields of view.
    Handles zoom mismatches between query photo and indexed view.
    """
    crops = []
    pano_w, pano_h = panorama.size
    
    for fov in fovs:
        # Convert heading + FOV to pixel crop bounds
        center_x = int((heading_deg / 360.0) * pano_w) % pano_w
        
        # FOV to pixel width (equirectangular approximation)
        crop_width = int((fov / 360.0) * pano_w)
        crop_height = int(pano_h * 0.5)  # use middle 50% vertically
        
        x_start = (center_x - crop_width // 2) % pano_w
        y_start = pano_h // 4
        
        if x_start + crop_width <= pano_w:
            crop = panorama.crop((x_start, y_start, x_start + crop_width, y_start + crop_height))
        else:
            # Wrap around panorama seam
            right = panorama.crop((x_start, y_start, pano_w, y_start + crop_height))
            left = panorama.crop((0, y_start, (x_start + crop_width) % pano_w, y_start + crop_height))
            crop = Image.new("RGB", (crop_width, crop_height))
            crop.paste(right, (0, 0))
            crop.paste(left, (pano_w - x_start, 0))
        
        crops.append((fov, crop.resize((640, 480), Image.LANCZOS)))
    
    return crops  # list of (fov_degrees, PIL.Image)
```

### Heading Refinement

```python
def refine_heading(top_candidates, query_kpts, extractor, matcher, device, steps=range(-45, 46, 15)):
    """
    For top candidates, test ±45° heading offsets to find optimal viewing direction.
    """
    best_result = None
    best_inliers = 0
    
    for candidate in top_candidates[:15]:
        base_heading = candidate["heading"]
        
        for offset in steps:
            test_heading = (base_heading + offset) % 360
            
            # Download/crop panorama at test_heading
            crops = generate_fov_crops(candidate["panorama"], test_heading)
            
            for fov, crop in crops:
                kpts_c, kpts_q, _ = match_images_from_pil(crop, query_kpts, extractor, matcher, device)
                inliers = ransac_verify(kpts_c, kpts_q)
                
                if inliers > best_inliers:
                    best_inliers = inliers
                    best_result = {
                        **candidate,
                        "refined_heading": test_heading,
                        "refined_fov": fov,
                        "inliers": inliers
                    }
    
    return best_result
```

---

## Configuration Patterns

### Search Radius Guidelines

```python
# Start conservative, expand if no match found
search_configs = {
    "known_city":      {"radius_km": 2.0,  "top_k": 300},
    "known_district":  {"radius_km": 0.5,  "top_k": 200},
    "unknown_country": {"radius_km": 50.0, "top_k": 1000},
    "global_search":   {"radius_km": None, "top_k": 1000},  # no radius filter
}
```

### Index Size vs. Accuracy Trade-off

```python
# Grid resolution: higher = denser panorama coverage = better accuracy
# Default 300 recommended — do not increase beyond 500
indexing_configs = {
    "fast_test":    {"grid_resolution": 300, "radius_km": 0.5},
    "city_block":   {"grid_resolution": 300, "radius_km": 1.0},
    "neighborhood": {"grid_resolution": 300, "radius_km": 3.0},
    "district":     {"grid_resolution": 300, "radius_km": 10.0},
}
```

### When to Use Ultra Mode

```python
# Enable Ultra Mode for:
image_conditions_needing_ultra = [
    "night photography",
    "motion blur",
    "heavy rain or fog",
    "low texture surfaces (walls, empty roads)",
    "heavy compression artifacts",
    "extreme angles (looking straight up/down)",
    "< 30 inliers from standard pipeline",
]
```

---

## Troubleshooting

### No matches found

```
Symptom: Search returns 0 candidates or very low inliers (<10)
```

1. **Verify index covers the area**: Check that your center coordinates + radius overlap with the indexed region
2. **Expand search radius**: Try 2x–5x your initial radius
3. **Enable Ultra Mode**: Especially for degraded images
4. **Check image quality**: Query image should be street-level, not aerial/satellite

### GUI appears blank on macOS

```bash
brew install python-tk@3.11  # or your Python version
# Re-activate venv and relaunch
source venv/bin/activate
python test_super.py
```

### CUDA out of memory

```python
# Reduce keypoint count (modify in test_super.py extractor init)
extractor = ALIKED(max_num_keypoints=512)  # default 1024

# Or reduce top_k candidates processed per search
top_k = 200  # default 500
```

### MPS (Mac) errors with LightGlue

```bash
# Ensure latest PyTorch with MPS support
pip install --upgrade torch torchvision
# LightGlue falls back to DISK on MPS — this is expected
```

### Index build interrupted

No action needed. Re-run Create Index with the same parameters — the builder resumes from `cosplace_parts/*.npz` checkpoints automatically.

### Low confidence scores despite correct location

- Run heading refinement (built into the pipeline, ensure it completes)
- The query photo FOV may be very narrow — try a wider crop
- Enable Ultra Mode for descriptor hopping

### Gemini AI Coarse mode not working

```bash
# Verify key is set
echo $GEMINI_API_KEY

# Re-export if missing
export GEMINI_API_KEY="your_key_here"

# Note: Manual mode works without API key and is generally preferred
```

---

## Key Design Principles

1. **Source agnostic**: Works with Mapillary, KartaView, or any street-view provider — not just Google Street View
2. **Single unified index**: All cities in one index; radius filtering handles scoping at query time
3. **Incremental indexing**: Interrupted builds resume automatically
4. **Hardware adaptive**: ALIKED on CUDA, DISK on MPS/CPU — same pipeline, hardware-appropriate models
5. **Cluster over outliers**: Spatial consensus prevents single high-inlier false positives from overriding geographic clusters
