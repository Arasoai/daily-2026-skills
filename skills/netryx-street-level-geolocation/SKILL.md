```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, a local-first open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue computer vision models.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - netryx geolocation
  - index street view panoramas
  - match street photo to location
  - osint geolocation tool
  - reverse geolocate image
---

# Netryx Street-Level Geolocation Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that identifies precise GPS coordinates (sub-50m accuracy) from any street-level photograph. It crawls and indexes street-view panoramas, then matches query photos against the index using a three-stage computer vision pipeline: global visual retrieval (CosPlace), local feature extraction (ALIKED/DISK), and deep feature matching (LightGlue + RANSAC). No landmarks required. Runs entirely on local hardware.

---

## Installation

### Prerequisites

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Python | 3.9+ | 3.10+ |
| GPU VRAM | 4GB | 8GB+ |
| RAM | 8GB | 16GB+ |
| Storage | 10GB | 50GB+ |

**GPU backend auto-detection:**
- Mac M1/M2/M3/M4 → MPS (Metal)
- NVIDIA GPU → CUDA
- Fallback → CPU (significantly slower)

### Setup

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must install from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR for Ultra Mode (difficult/blurry images)
pip install kornia
```

### Optional: Gemini AI Coarse Mode

```bash
export GEMINI_API_KEY="your_key_from_aistudio_google_com"
```

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix:** `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### Step 1: Create an Index

Index an area before searching. The GUI walks you through this, but understand the parameters:

| Parameter | Description | Recommended |
|-----------|-------------|-------------|
| Center lat/lon | Geographic center of area to index | Target city center |
| Radius (km) | Search radius from center | 0.5–1km for testing, 5–10km production |
| Grid resolution | Panorama density (don't change) | 300 (default) |

**Indexing time estimates:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500 | 30 min | ~60 MB |
| 1 km | ~2,000 | 1–2 hours | ~250 MB |
| 5 km | ~30,000 | 8–12 hours | ~3 GB |
| 10 km | ~100,000 | 24–48 hours | ~7 GB |

Indexing is **resumable** — interrupting and restarting continues from where it left off.

### Step 2: Search

1. Select **Search** mode in GUI
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide center coordinates + radius (fastest, use when region is known)
   - **AI Coarse**: Gemini analyzes visual clues to estimate region (no prior knowledge needed)
4. Click **Run Search** → **Start Full Search**
5. View result: GPS coordinates + confidence score on map

---

## Project Structure

```
netryx/
├── test_super.py              # Main GUI application — indexing + search
├── cosplace_utils.py          # CosPlace model loading + descriptor extraction
├── build_index.py             # Standalone high-performance index builder (large areas)
├── requirements.txt
├── cosplace_parts/            # Raw embedding chunks (.npz), created during indexing
└── index/
    ├── cosplace_descriptors.npy   # All 512-dim CosPlace descriptors
    └── metadata.npz               # Coordinates, headings, panorama IDs
```

---

## Pipeline Deep Dive

### Stage 1 — Global Retrieval (CosPlace)

```
Query photo → 512-dim CosPlace fingerprint
Flipped query → 512-dim CosPlace fingerprint
Both → cosine similarity vs. entire index
Radius filter (haversine) → top 500–1000 candidates
Duration: <1 second (single matrix multiplication)
```

### Stage 2 — Local Geometric Verification

```
For each candidate panorama:
  1. Download from Street View (8 tiles, stitched)
  2. Crop at indexed heading angle
  3. Generate multi-FOV crops: 70°, 90°, 110°
  4. Extract keypoints: ALIKED (CUDA) or DISK (MPS/CPU)
  5. LightGlue deep feature matching vs. query keypoints
  6. RANSAC → filter geometrically consistent inliers
Best candidate = highest inlier count
Duration: 2–5 minutes for 300–500 candidates
```

### Stage 3 — Refinement

```
Heading refinement: top 15 candidates × ±45° at 15° steps × 3 FOVs
Spatial consensus: cluster matches into 50m cells
Confidence scoring: clustering density + uniqueness ratio
Output: GPS coordinates + confidence score
```

### Ultra Mode (difficult images)

Enable for: night photos, motion blur, low texture, fog

```
+ LoFTR: detector-free dense matching (works without keypoints)
+ Descriptor hopping: re-search index using matched panorama's clean descriptor
+ Neighborhood expansion: search all panoramas within 100m of best match
Cost: significantly slower; benefit: catches matches standard pipeline misses
```

---

## Index Architecture

The index is **unified and location-agnostic**. Multiple cities/regions can coexist:

```python
# Conceptual flow — all handled internally by test_super.py

# Indexing writes to:
#   cosplace_parts/*.npz  (raw chunks, written incrementally)
#   index/cosplace_descriptors.npy  (compiled, all 512-dim vectors)
#   index/metadata.npz             (lat, lon, heading, panoid per entry)

# Searching filters by radius at query time:
#   center=(48.8566, 2.3522), radius=5km  → only Paris results
#   center=(51.5074, -0.1278), radius=10km → only London results
# No city selection needed — radius handles isolation
```

---

## Code Examples

### Extract a CosPlace Descriptor Manually

```python
import torch
from PIL import Image
from torchvision import transforms
from cosplace_utils import load_cosplace_model, extract_descriptor

# Load model (auto-detects CUDA/MPS/CPU)
model = load_cosplace_model()

# Prepare image
transform = transforms.Compose([
    transforms.Resize((512, 512)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])

img = Image.open("street_photo.jpg").convert("RGB")
tensor = transform(img).unsqueeze(0)  # (1, 3, 512, 512)

# Determine device
device = (
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)
tensor = tensor.to(device)
model = model.to(device)

# Extract 512-dim descriptor
with torch.no_grad():
    descriptor = extract_descriptor(model, tensor)  # shape: (512,)

print(f"Descriptor shape: {descriptor.shape}")
print(f"Descriptor norm: {descriptor.norm().item():.4f}")
```

### Load and Query the Index Directly

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity
from math import radians, cos, sin, asin, sqrt

def haversine_km(lat1, lon1, lat2, lon2):
    """Return distance in km between two lat/lon points."""
    R = 6371.0
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

# Load prebuilt index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
metadata = np.load("index/metadata.npz", allow_pickle=True)
lats = metadata["lats"]      # (N,)
lons = metadata["lons"]      # (N,)
headings = metadata["headings"]  # (N,)
panoids = metadata["panoids"]    # (N,)

def search_index(query_descriptor, center_lat, center_lon, radius_km=2.0, top_k=500):
    """
    Find top-K candidates within radius_km of center point.
    query_descriptor: np.ndarray shape (512,), L2-normalized
    """
    # Cosine similarity (descriptors should be L2-normalized)
    query = query_descriptor.reshape(1, -1)
    scores = cosine_similarity(query, descriptors)[0]  # (N,)

    # Haversine radius filter
    in_radius = np.array([
        haversine_km(center_lat, center_lon, lats[i], lons[i]) <= radius_km
        for i in range(len(lats))
    ])

    # Apply filter and rank
    filtered_scores = np.where(in_radius, scores, -1.0)
    top_indices = np.argsort(filtered_scores)[::-1][:top_k]

    results = []
    for idx in top_indices:
        if filtered_scores[idx] < 0:
            break
        results.append({
            "panoid": panoids[idx],
            "lat": lats[idx],
            "lon": lons[idx],
            "heading": headings[idx],
            "score": float(filtered_scores[idx]),
        })
    return results

# Example usage
results = search_index(
    query_descriptor=my_descriptor,   # your 512-dim np array
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=3.0,
    top_k=500,
)
print(f"Top match: {results[0]}")
```

### Build Index for Large Areas (CLI)

For areas too large for the GUI, use the standalone builder:

```bash
# High-performance standalone index builder
python build_index.py \
    --lat 48.8566 \
    --lon 2.3522 \
    --radius 10 \
    --resolution 300 \
    --output ./index
```

### Check Device and Model Setup

```python
import torch

def get_device():
    if torch.cuda.is_available():
        device = "cuda"
        print(f"Using CUDA: {torch.cuda.get_device_name(0)}")
        print(f"VRAM: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB")
    elif torch.backends.mps.is_available():
        device = "mps"
        print("Using Apple MPS (Metal)")
    else:
        device = "cpu"
        print("Using CPU — expect slow performance")
    return device

device = get_device()

# Verify LightGlue installed correctly
try:
    from lightglue import LightGlue, ALIKED, DISK
    print("LightGlue: OK")
except ImportError:
    print("LightGlue missing — run: pip install git+https://github.com/cvg/LightGlue.git")

# Verify Ultra Mode dependency
try:
    import kornia
    print(f"kornia (Ultra Mode): OK — v{kornia.__version__}")
except ImportError:
    print("kornia not installed — Ultra Mode unavailable. Run: pip install kornia")
```

### Feature Matching Example (ALIKED + LightGlue)

```python
import torch
from lightglue import LightGlue, ALIKED
from lightglue.utils import load_image, rbd

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Initialize extractor and matcher
extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

# Load images (query and candidate panorama crop)
query = load_image("query_photo.jpg").to(device)
candidate = load_image("candidate_crop.jpg").to(device)

# Extract keypoints
with torch.no_grad():
    feats0 = extractor.extract(query)
    feats1 = extractor.extract(candidate)

    # Match
    matches01 = matcher({"image0": feats0, "image1": feats1})

# Remove batch dimension
feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]

# Get matched keypoint coordinates
kpts0 = feats0["keypoints"][matches01["matches"][..., 0]]
kpts1 = feats1["keypoints"][matches01["matches"][..., 1]]
num_matches = len(matches01["matches"])
print(f"Matched keypoints: {num_matches}")

# Apply RANSAC to filter geometric outliers
import cv2
import numpy as np

if num_matches >= 8:
    pts0 = kpts0.cpu().numpy()
    pts1 = kpts1.cpu().numpy()
    _, inlier_mask = cv2.findFundamentalMat(
        pts0, pts1,
        method=cv2.FM_RANSAC,
        ransacReprojThreshold=3.0,
        confidence=0.999,
    )
    num_inliers = int(inlier_mask.sum()) if inlier_mask is not None else 0
    print(f"RANSAC inliers: {num_inliers}")
```

---

## Common Patterns

### Pattern 1: Multi-City Index, City-Scoped Search

```python
# Index multiple cities into the same index (done via GUI, once per city)
# Paris: center=(48.8566, 2.3522), radius=5km
# London: center=(51.5074, -0.1278), radius=5km
# Tokyo: center=(35.6762, 139.6503), radius=5km

# Search is automatically scoped by radius at query time
paris_results = search_index(descriptor, 48.8566, 2.3522, radius_km=5.0)
london_results = search_index(descriptor, 51.5074, -0.1278, radius_km=5.0)
# No overlap — haversine filter handles it
```

### Pattern 2: Flipped Descriptor for Reversed Perspectives

```python
import torchvision.transforms.functional as TF

img_tensor = load_my_image("photo.jpg")  # (1, 3, H, W)
img_flipped = TF.hflip(img_tensor)

with torch.no_grad():
    desc_normal = extract_descriptor(model, img_tensor)
    desc_flipped = extract_descriptor(model, img_flipped)

# Search with both, take best result
results_normal = search_index(desc_normal, lat, lon, radius_km=2.0)
results_flipped = search_index(desc_flipped, lat, lon, radius_km=2.0)

best = results_normal[0] if results_normal[0]["score"] >= results_flipped[0]["score"] \
       else results_flipped[0]
```

### Pattern 3: Multi-FOV Candidate Crops

```python
# Netryx tests 3 FOVs to handle zoom mismatches between query and indexed panorama
FOVS = [70, 90, 110]

def get_rectilinear_crop(panorama_pil, heading_deg, fov_deg, output_size=(512, 512)):
    """
    Extract a perspective crop from an equirectangular panorama.
    heading_deg: 0=North, 90=East, 180=South, 270=West
    fov_deg: horizontal field of view
    """
    # Implementation uses equirectangular-to-rectilinear projection
    # Available in test_super.py — reference that for full implementation
    pass

crops = {fov: get_rectilinear_crop(pano, heading=90, fov_deg=fov) for fov in FOVS}
# Match query against each FOV crop, keep highest inlier count
```

---

## Troubleshooting

### GUI Appears Blank (macOS)

```bash
brew install python-tk@3.11   # Replace 3.11 with your Python version
```

### LightGlue Import Error

```bash
# Must install from GitHub, not PyPI
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### CUDA Out of Memory

```python
# Reduce keypoints to lower VRAM usage
extractor = ALIKED(max_num_keypoints=512)  # default 1024
# Or switch to DISK which uses less memory
extractor = DISK(max_num_keypoints=768)
```

### MPS Errors on Mac (ALIKED not supported)

Netryx auto-falls back to DISK on MPS — this is expected behavior. ALIKED requires CUDA. DISK produces slightly fewer keypoints (768 vs 1024) but works correctly on Apple Silicon.

### Slow Indexing

```python
# Use standalone builder for large areas — significantly faster than GUI indexer
python build_index.py --lat LAT --lon LON --radius KM --resolution 300
```

### Index Partially Built / Corrupted

The index builds incrementally into `cosplace_parts/*.npz`. Restart indexing — it resumes from the last saved chunk automatically. To force rebuild:

```bash
rm -rf cosplace_parts/ index/
python test_super.py  # restart indexing from scratch
```

### Low Confidence Results

1. Ensure the area is indexed at sufficient density (radius and resolution)
2. Enable **Ultra Mode** for degraded images
3. Verify the query photo is street-level (not aerial/satellite)
4. Try **AI Coarse** mode if region is unknown — Gemini may identify a more precise starting area
5. Check that the search radius actually covers the photo's true location

### Gemini AI Coarse Not Working

```bash
# Verify API key is set
echo $GEMINI_API_KEY

# Set it if missing
export GEMINI_API_KEY="your_key_here"
```

---

## Configuration Reference

| Setting | Location | Effect |
|---------|----------|--------|
| Grid resolution | GUI / `build_index.py --resolution` | Panorama density; don't change from 300 |
| Search radius | GUI search step | Haversine filter at query time |
| Top-K candidates | Internal (Stage 1) | 500–1000; tradeoff: accuracy vs. speed |
| Ultra Mode | GUI checkbox | Adds LoFTR + hopping + expansion |
| AI Coarse | GUI dropdown | Uses Gemini for blind region estimation |
| `GEMINI_API_KEY` | Environment variable | Required for AI Coarse mode |

---

## Models Reference

| Model | Task | Hardware |
|-------|------|----------|
| CosPlace (512-dim) | Global descriptor / retrieval | All (CUDA/MPS/CPU) |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU fallback |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | All (requires `kornia`) |
```
