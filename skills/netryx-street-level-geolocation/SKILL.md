```markdown
---
name: netryx-street-level-geolocation
description: Expert guidance for using Netryx, a locally-hosted open-source street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo locally
  - find GPS coordinates from street image
  - run Netryx geolocation
  - index street view panoramas
  - street level geolocation pipeline
  - use CosPlace LightGlue geolocation
  - OSINT geolocation from photo
  - identify location from street photo
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted, open-source geolocation engine that takes any street-level photograph and returns precise GPS coordinates (sub-50m accuracy). It works by crawling and indexing street-view panoramas, then matching query images against that index using a three-stage computer vision pipeline: global retrieval (CosPlace), local feature extraction (ALIKED/DISK), and deep feature matching (LightGlue with RANSAC verification). No landmarks required. Runs entirely on local hardware with CUDA, MPS (Apple Silicon), or CPU.

---

## Installation

```bash
git clone https://github.com/sparkyniner/Netryx-OpenSource-Next-Gen-Street-Level-Geolocation.git
cd Netryx-OpenSource-Next-Gen-Street-Level-Geolocation

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Required: LightGlue (must install from GitHub)
pip install git+https://github.com/cvg/LightGlue.git

# Optional: LoFTR support for Ultra Mode
pip install kornia
```

### macOS tkinter fix (if GUI appears blank)
```bash
brew install python-tk@3.11  # match your Python version
```

### Gemini API key (optional, for AI Coarse mode)
```bash
export GEMINI_API_KEY="your_key_here"
```

---

## Launch the GUI

```bash
python test_super.py
```

The GUI is the primary interface. It handles both index creation and search.

---

## Project Structure

```
netryx/
├── test_super.py          # Main app — GUI, indexing, and search
├── cosplace_utils.py      # CosPlace model loading and descriptor extraction
├── build_index.py         # Standalone high-performance index builder (large datasets)
├── requirements.txt
├── cosplace_parts/        # Raw embedding chunks written during indexing
├── index/
│   ├── cosplace_descriptors.npy   # All 512-dim descriptors (searchable)
│   └── metadata.npz               # Coordinates, headings, panorama IDs
└── README.md
```

---

## Core Workflow

### Step 1 — Create an Index

Index an area by crawling street-view panoramas and extracting CosPlace fingerprints.

**In the GUI:**
1. Select **Create** mode
2. Enter center coordinates (lat, lon)
3. Set radius in km (start with 0.5–1 km for testing)
4. Set grid resolution (default: 300 — do not change)
5. Click **Create Index**

**Indexing time reference:**

| Radius | ~Panoramas | Time (M2 Max) | Index Size |
|--------|-----------|---------------|------------|
| 0.5 km | ~500      | 30 min        | ~60 MB     |
| 1 km   | ~2,000    | 1–2 hours     | ~250 MB    |
| 5 km   | ~30,000   | 8–12 hours    | ~3 GB      |
| 10 km  | ~100,000  | 24–48 hours   | ~7 GB      |

Indexing is **resumable** — if interrupted, it picks up from where it left off.

### Step 2 — Search

1. Select **Search** mode
2. Upload a street-level photo
3. Choose search method:
   - **Manual**: Provide approximate center coordinates + search radius
   - **AI Coarse**: Gemini analyzes visual cues to estimate region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Results appear on a map with confidence score

### Ultra Mode

Enable **Ultra Mode** for difficult images (night shots, blur, low texture). Adds:
- **LoFTR** dense matching (no keypoint detection needed)
- **Descriptor hopping** (re-searches index using matched panorama's clean descriptor)
- **Neighborhood expansion** (searches panoramas within 100m of best match)

Ultra Mode is slower but significantly improves results for degraded images.

---

## Pipeline Details

### Stage 1 — Global Retrieval (CosPlace)
- Extracts a 512-dim fingerprint from the query image and its horizontal flip
- Compares both against the entire index via cosine similarity
- Applies haversine radius filter to restrict to the search area
- Returns top 500–1000 candidates
- Runs in **< 1 second** (single matrix multiplication)

### Stage 2 — Geometric Verification (ALIKED/DISK + LightGlue)
- Downloads panorama tiles from Street View, stitches them
- Crops at the indexed heading angle
- Generates multi-FOV crops (70°, 90°, 110°) to handle zoom mismatches
- Extracts keypoints:
  - **ALIKED** on CUDA (1024 keypoints)
  - **DISK** on MPS/CPU (768 keypoints)
- LightGlue matches query keypoints against candidate keypoints
- RANSAC filters geometrically inconsistent matches
- Runs in **2–5 minutes** for 300–500 candidates

### Stage 3 — Refinement
- **Heading refinement**: Tests ±45° offsets at 15° steps across 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; clusters beat single high-inlier outliers
- **Confidence scoring**: Evaluates geographic clustering and uniqueness ratio of top matches

---

## Index Architecture

All embeddings go into one unified index regardless of location. The radius filter at search time handles geographic isolation — no city selection needed.

```
Create Mode:
  Grid points → Street View API → Panoramas → Crops → CosPlace → cosplace_parts/*.npz

Auto-build:
  cosplace_parts/*.npz → index/cosplace_descriptors.npy + index/metadata.npz

Search Mode:
  Query image → CosPlace → Index (radius-filtered) → Download candidates
             → ALIKED/DISK + LightGlue → Refinement → GPS + Confidence
```

You can index Paris, Tel Aviv, and London into the same index. Search coordinates + radius select the right subset automatically.

---

## Platform-Specific Behavior

| Feature        | CUDA (NVIDIA) | MPS (Apple Silicon) | CPU  |
|----------------|--------------|---------------------|------|
| Feature extractor | ALIKED (1024 kp) | DISK (768 kp) | DISK |
| Ultra Mode (LoFTR) | Full support | Partial | Slow |
| Recommended VRAM | 8GB+ | Unified 8GB+ | N/A |

---

## Real Code Examples

### Extract a CosPlace descriptor from an image

```python
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import torch

# Load model (downloads weights on first run)
model = load_cosplace_model(device="cuda")  # or "mps" or "cpu"

# Extract descriptor from a query image
image = Image.open("query.jpg").convert("RGB")
descriptor = extract_descriptor(model, image, device="cuda")
# descriptor shape: (512,) — float32 numpy array
print(f"Descriptor shape: {descriptor.shape}")
```

### Search the index manually (without GUI)

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

# Load the compiled index
descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
meta = np.load("index/metadata.npz", allow_pickle=True)
lats = meta["lats"]    # (N,)
lons = meta["lons"]    # (N,)
headings = meta["headings"]  # (N,)
panoids = meta["panoids"]    # (N,)

def haversine(lat1, lon1, lat2, lon2):
    """Returns distance in meters between two lat/lon points."""
    R = 6371000
    phi1, phi2 = radians(lat1), radians(lat2)
    dphi = radians(lat2 - lat1)
    dlambda = radians(lon2 - lon1)
    a = sin(dphi/2)**2 + cos(phi1)*cos(phi2)*sin(dlambda/2)**2
    return 2 * R * asin(sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_m, top_k=500):
    """
    Find top_k candidates within radius_m meters of center.
    query_descriptor: (512,) float32 numpy array
    Returns: list of dicts with panoid, lat, lon, heading, score
    """
    # Haversine radius filter
    distances = np.array([
        haversine(center_lat, center_lon, lat, lon)
        for lat, lon in zip(lats, lons)
    ])
    mask = distances <= radius_m
    
    if mask.sum() == 0:
        return []
    
    # Cosine similarity search (descriptors are L2-normalized)
    q = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    local_descs = descriptors[mask]
    scores = local_descs @ q  # cosine similarity
    
    # Also try flipped descriptor
    q_flip = query_descriptor[::-1].copy()
    q_flip /= (np.linalg.norm(q_flip) + 1e-8)
    scores_flip = local_descs @ q_flip
    scores = np.maximum(scores, scores_flip)
    
    # Top-k
    top_indices = np.argsort(scores)[::-1][:top_k]
    masked_indices = np.where(mask)[0]
    
    results = []
    for idx in top_indices:
        global_idx = masked_indices[idx]
        results.append({
            "panoid": str(panoids[global_idx]),
            "lat": float(lats[global_idx]),
            "lon": float(lons[global_idx]),
            "heading": float(headings[global_idx]),
            "score": float(scores[idx]),
        })
    return results

# Example usage
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image

model = load_cosplace_model(device="cuda")
image = Image.open("query.jpg").convert("RGB")
query_desc = extract_descriptor(model, image, device="cuda")

candidates = search_index(
    query_desc,
    center_lat=48.8566,   # Paris
    center_lon=2.3522,
    radius_m=5000,        # 5 km
    top_k=500
)
print(f"Found {len(candidates)} candidates")
print(f"Top match: {candidates[0]}")
```

### Build the searchable index from cosplace_parts chunks

```python
# Run this after a Create Index session to compile parts into the searchable index
# (The GUI does this automatically, but you can trigger it manually)
import subprocess
subprocess.run(["python", "build_index.py"], check=True)

# Or use build_index.py directly for large datasets:
# python build_index.py
```

### Check which device will be used

```python
import torch

if torch.cuda.is_available():
    device = "cuda"
    extractor = "ALIKED"
elif torch.backends.mps.is_available():
    device = "mps"
    extractor = "DISK"
else:
    device = "cpu"
    extractor = "DISK"

print(f"Device: {device}, Feature extractor: {extractor}")
```

### Use LightGlue for matching (after installing from GitHub)

```python
from lightglue import LightGlue, ALIKED, DISK
from lightglue.utils import load_image, rbd
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# On CUDA use ALIKED, on MPS/CPU use DISK
extractor = ALIKED(max_num_keypoints=1024).eval().to(device)
matcher = LightGlue(features="aliked").eval().to(device)

def match_images(image0_path, image1_path):
    """
    Extract features and match two images.
    Returns number of verified inliers.
    """
    image0 = load_image(image0_path).to(device)
    image1 = load_image(image1_path).to(device)

    feats0 = extractor.extract(image0)
    feats1 = extractor.extract(image1)
    matches01 = matcher({"image0": feats0, "image1": feats1})
    
    feats0, feats1, matches01 = [rbd(x) for x in [feats0, feats1, matches01]]
    
    kpts0 = feats0["keypoints"]
    kpts1 = feats1["keypoints"]
    matched_kpts0 = kpts0[matches01["matches"][..., 0]]
    matched_kpts1 = kpts1[matches01["matches"][..., 1]]
    
    num_matches = len(matches01["matches"])
    return num_matches, matched_kpts0, matched_kpts1

inliers, pts0, pts1 = match_images("query.jpg", "candidate_crop.jpg")
print(f"Verified inliers: {inliers}")
```

---

## Common Patterns

### Pattern: Headless batch search (scripted, no GUI)

```python
"""
Run a geolocation search entirely in code.
Useful for batch OSINT pipelines or server deployments.
"""
from cosplace_utils import load_cosplace_model, extract_descriptor
from PIL import Image
import numpy as np

def geolocate(image_path, center_lat, center_lon, radius_km=5.0, device="cuda"):
    model = load_cosplace_model(device=device)
    image = Image.open(image_path).convert("RGB")
    
    # Extract descriptor
    desc = extract_descriptor(model, image, device=device)
    
    # Load index
    descriptors = np.load("index/cosplace_descriptors.npy")
    meta = np.load("index/metadata.npz", allow_pickle=True)
    
    # Search (using search_index from earlier example)
    candidates = search_index(desc, center_lat, center_lon, radius_km * 1000, top_k=500)
    
    # For full verification, launch test_super.py GUI or extend with LightGlue stage
    return candidates

results = geolocate("mystery_photo.jpg", center_lat=48.8566, center_lon=2.3522)
print(results[:5])
```

### Pattern: Progressive radius search (unknown location)

```python
# If you don't know the area, expand radius progressively
# Best used with AI Coarse mode in the GUI to get a starting point

search_configs = [
    {"radius_km": 1},
    {"radius_km": 5},
    {"radius_km": 10},
]

for config in search_configs:
    candidates = search_index(desc, center_lat, center_lon, config["radius_km"] * 1000)
    if candidates and candidates[0]["score"] > 0.75:
        print(f"High-confidence match at {config['radius_km']}km radius")
        print(candidates[0])
        break
```

### Pattern: Multi-city index, single search

```python
# Index Paris, London, Berlin separately — all go into the same index files
# Search is isolated by coordinates + radius — no extra configuration needed

paris_results = search_index(desc, 48.8566, 2.3522, 5000)
london_results = search_index(desc, 51.5074, -0.1278, 5000)
# No overlap between results — haversine filter handles it
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11  # or python-tk@3.12 depending on your Python version
```

### LightGlue import error
```bash
# Must be installed from GitHub, not PyPI
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### LoFTR/Ultra Mode not working
```bash
pip install kornia
# Verify:
python -c "import kornia; print(kornia.__version__)"
```

### CUDA out of memory
- Reduce `max_num_keypoints` in ALIKED (e.g., 512 instead of 1024)
- Process fewer candidates per batch (reduce top_k from 500 to 200)
- Switch to DISK which uses less VRAM

### Index not found / no candidates returned
```bash
# Check that the index was compiled after indexing
python build_index.py

# Verify files exist
ls -lh index/cosplace_descriptors.npy index/metadata.npz
```

### Indexing stalled / slow
- Grid resolution of 300 is optimal — do not increase it
- Use `build_index.py` directly for large datasets (faster than GUI path)
- Indexing is resumable — safe to interrupt and restart

### Low confidence results / wrong location
1. Enable **Ultra Mode** in the GUI
2. Ensure your indexed area actually contains the query location
3. Try a smaller radius to reduce false positive candidates
4. Verify the query image is a genuine street-level photo (not aerial or indoor)

### No Street View data for indexed area
- Some rural areas have sparse or no Street View coverage
- Try Mapillary or KartaView as alternative panorama sources (Netryx is source-agnostic)
- Increase grid resolution slightly to improve coverage in sparse areas (with caution)

---

## Hardware Recommendations

| Setup | Speed | Notes |
|-------|-------|-------|
| NVIDIA GPU 8GB+ CUDA | Fastest | ALIKED 1024 kp, full pipeline |
| Apple M1/M2/M3/M4 (MPS) | Fast | DISK 768 kp, LightGlue works well |
| CPU only | Slow | Functional but 10–20× slower than GPU |
| 4GB VRAM | Minimum | Reduce keypoints if OOM |

---

## Models Reference

| Model | Role | Source |
|-------|------|--------|
| CosPlace | Global descriptor / retrieval | [github.com/gmberton/cosplace](https://github.com/gmberton/cosplace) |
| ALIKED | Local keypoints (CUDA) | [github.com/naver/alike](https://github.com/naver/alike) |
| DISK | Local keypoints (MPS/CPU) | [github.com/cvlab-epfl/disk](https://github.com/cvlab-epfl/disk) |
| LightGlue | Deep feature matching | [github.com/cvg/LightGlue](https://github.com/cvg/LightGlue) |
| LoFTR | Dense matching (Ultra Mode) | via `kornia` |
```
