```markdown
---
name: netryx-street-level-geolocation
description: Expert skill for using Netryx, the open-source local-first street-level geolocation engine that identifies GPS coordinates from street photos using CosPlace, ALIKED/DISK, and LightGlue.
triggers:
  - geolocate a street photo
  - find GPS coordinates from image
  - street level geolocation
  - build a geolocation index
  - run netryx search
  - identify location from street view photo
  - osint geolocation with computer vision
  - netryx pipeline setup
---

# Netryx Street-Level Geolocation

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Netryx is a locally-hosted geolocation engine that identifies precise GPS coordinates from any street-level photograph. It crawls street-view panoramas, indexes them as 512-dim visual fingerprints (CosPlace), then matches query images via local feature extraction (ALIKED/DISK) and deep feature matching (LightGlue). Sub-50m accuracy. Runs entirely on your hardware — no SaaS, no cloud API required for searching.

---

## Installation

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

### Platform Notes
- **NVIDIA GPU**: Uses ALIKED (1024 keypoints) — fastest, most accurate
- **Mac M1/M2/M3/M4**: Uses DISK via MPS (768 keypoints) — good performance
- **CPU only**: Works but significantly slower; not recommended for large indexes

### Optional: Gemini API for AI Coarse Mode
```bash
export GEMINI_API_KEY="your_key_here"   # From https://aistudio.google.com
```
AI Coarse mode uses Gemini to guess a region from visual cues (signs, architecture) when you have no prior location knowledge. Manual mode with known coordinates is generally more reliable.

---

## Launch the GUI

```bash
python test_super.py
```

> **macOS blank GUI fix**: `brew install python-tk@3.11` (match your Python version)

---

## Core Workflow

### Step 1 — Create an Index

Index an area before searching. This crawls street-view panoramas, extracts CosPlace fingerprints, and saves them locally.

**In the GUI:**
1. Select **Create** mode
2. Enter center lat/lon of the area
3. Set radius (km) and grid resolution (default 300 — don't change)
4. Click **Create Index**

**Index size reference:**

| Radius | ~Panoramas | Time (M2 Max) | Disk |
|--------|-----------|---------------|------|
| 0.5 km | ~500      | 30 min        | ~60 MB |
| 1 km   | ~2,000    | 1–2 hr        | ~250 MB |
| 5 km   | ~30,000   | 8–12 hr       | ~3 GB |
| 10 km  | ~100,000  | 24–48 hr      | ~7 GB |

Indexing is **resumable** — if interrupted, re-run and it picks up where it left off.

For large areas, use the standalone high-performance builder:
```bash
python build_index.py
```

### Step 2 — Search

**In the GUI:**
1. Select **Search** mode
2. Upload a street-level photo
3. Choose method:
   - **Manual**: Provide approximate center lat/lon + radius (preferred)
   - **AI Coarse**: Gemini analyzes the image to guess the region (requires `GEMINI_API_KEY`)
4. Click **Run Search** → **Start Full Search**
5. Result: GPS coordinates + confidence score displayed on map

---

## Index Architecture

All city indexes live in one unified index. Radius filtering at search time scopes results automatically — no per-city selection needed.

```
cosplace_parts/          # Raw embedding chunks (.npz), written during indexing
index/
  cosplace_descriptors.npy    # All 512-dim CosPlace descriptors (matrix)
  metadata.npz                # lat, lon, heading, panorama IDs
```

**You can index multiple cities into the same index:**
```
Index Paris (48.85, 2.35, radius=5km)
Index London (51.50, -0.12, radius=5km)
Index NYC    (40.71, -74.00, radius=5km)

Search: center=Paris coords, radius=5km → only Paris results returned
Search: center=London coords, radius=5km → only London results returned
```

---

## Pipeline Deep-Dive

### Stage 1: Global Retrieval (CosPlace)
- Extracts 512-dim descriptor from query image + horizontally flipped version
- Cosine similarity against all indexed descriptors (single matrix multiply — <1 sec)
- Haversine radius filter narrows to specified area
- Returns top 500–1000 candidates

### Stage 2: Geometric Verification (ALIKED/DISK + LightGlue)
- Downloads each candidate panorama (8 tiles, stitched)
- Crops at indexed heading, 3 FOVs: 70°, 90°, 110° (handles zoom mismatch)
- ALIKED (CUDA) or DISK (MPS/CPU) extracts keypoints + descriptors
- LightGlue matches query keypoints vs. candidate keypoints
- RANSAC filters to geometrically consistent inliers
- Best match = highest verified inlier count
- Runtime: 2–5 minutes for 300–500 candidates

### Stage 3: Refinement
- **Heading refinement**: Tests ±45° at 15° steps × 3 FOVs for top 15 candidates
- **Spatial consensus**: Clusters matches into 50m cells; prefers cluster over isolated outlier
- **Confidence scoring**: Geographic clustering of top matches + uniqueness ratio (best vs. runner-up)

### Ultra Mode (Difficult Images)
Enable for night shots, blur, low-texture scenes:
- **LoFTR**: Detector-free dense matching — works without reliable keypoints
- **Descriptor hopping**: Re-searches index using the matched panorama's clean descriptor
- **Neighborhood expansion**: Searches all panoramas within 100m of best match

---

## Code Examples

### Extract a CosPlace Descriptor Programmatically

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import torch

# Load model (auto-detects CUDA/MPS/CPU)
model, device = load_cosplace_model()

# Extract descriptor from an image file
img = Image.open("query.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)  # shape: (512,)

print(f"Descriptor shape: {descriptor.shape}")
print(f"Device used: {device}")
```

### Search the Index Programmatically

```python
import numpy as np
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    """Distance in km between two lat/lon points."""
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    return 2 * R * asin(sqrt(a))

def search_index(query_descriptor, center_lat, center_lon, radius_km, top_k=500):
    """
    Search the Netryx index for candidates near a given location.
    
    Args:
        query_descriptor: np.ndarray shape (512,) — CosPlace descriptor of query image
        center_lat, center_lon: float — center of search area
        radius_km: float — search radius in kilometers
        top_k: int — number of candidates to return
    
    Returns:
        list of dicts with keys: lat, lon, heading, panoid, similarity
    """
    # Load index
    descriptors = np.load("index/cosplace_descriptors.npy")  # (N, 512)
    meta = np.load("index/metadata.npz", allow_pickle=True)
    lats = meta["lats"]
    lons = meta["lons"]
    headings = meta["headings"]
    panoids = meta["panoids"]

    # Radius filter
    in_radius = np.array([
        haversine(center_lat, center_lon, lat, lon) <= radius_km
        for lat, lon in zip(lats, lons)
    ])
    
    filtered_desc = descriptors[in_radius]
    filtered_idx = np.where(in_radius)[0]

    if len(filtered_desc) == 0:
        return []

    # Cosine similarity (descriptors assumed L2-normalized)
    query_norm = query_descriptor / (np.linalg.norm(query_descriptor) + 1e-8)
    sims = filtered_desc @ query_norm  # shape: (N_filtered,)

    # Top-k
    top_k = min(top_k, len(sims))
    top_local_idx = np.argpartition(sims, -top_k)[-top_k:]
    top_local_idx = top_local_idx[np.argsort(sims[top_local_idx])[::-1]]

    results = []
    for local_i in top_local_idx:
        global_i = filtered_idx[local_i]
        results.append({
            "lat": float(lats[global_i]),
            "lon": float(lons[global_i]),
            "heading": float(headings[global_i]),
            "panoid": str(panoids[global_i]),
            "similarity": float(sims[local_i]),
        })

    return results


# Usage
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image

model, device = load_cosplace_model()
img = Image.open("mystery_street.jpg").convert("RGB")
descriptor = get_descriptor(model, img, device)

candidates = search_index(
    query_descriptor=descriptor,
    center_lat=48.8566,
    center_lon=2.3522,
    radius_km=5.0,
    top_k=500
)

print(f"Top candidate: {candidates[0]}")
```

### Build Index from Custom Panorama List

```python
import numpy as np
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import os

def build_custom_index(panorama_records, output_dir="index"):
    """
    Build a Netryx index from your own panorama images.
    
    Args:
        panorama_records: list of dicts with keys:
            - image_path: str
            - lat: float
            - lon: float
            - heading: float
            - panoid: str
    """
    os.makedirs(output_dir, exist_ok=True)
    model, device = load_cosplace_model()

    descriptors = []
    lats, lons, headings, panoids = [], [], [], []

    for i, record in enumerate(panorama_records):
        try:
            img = Image.open(record["image_path"]).convert("RGB")
            desc = get_descriptor(model, img, device)
            # L2 normalize for cosine similarity via dot product
            desc = desc / (np.linalg.norm(desc) + 1e-8)
            descriptors.append(desc)
            lats.append(record["lat"])
            lons.append(record["lon"])
            headings.append(record["heading"])
            panoids.append(record["panoid"])
            
            if (i + 1) % 100 == 0:
                print(f"Indexed {i+1}/{len(panorama_records)}")
        except Exception as e:
            print(f"Skipping {record['image_path']}: {e}")

    # Save
    np.save(f"{output_dir}/cosplace_descriptors.npy", np.array(descriptors))
    np.savez(
        f"{output_dir}/metadata.npz",
        lats=np.array(lats),
        lons=np.array(lons),
        headings=np.array(headings),
        panoids=np.array(panoids, dtype=object),
    )
    print(f"Index built: {len(descriptors)} panoramas → {output_dir}/")
```

### Batch Geolocate Multiple Images

```python
from cosplace_utils import load_cosplace_model, get_descriptor
from PIL import Image
import numpy as np

def batch_geolocate(image_paths, center_lat, center_lon, radius_km):
    """
    Geolocate multiple images using Stage 1 retrieval only (fast approximation).
    For full pipeline accuracy, use the GUI or integrate Stage 2 matching.
    """
    model, device = load_cosplace_model()
    
    results = []
    for path in image_paths:
        img = Image.open(path).convert("RGB")
        desc = get_descriptor(model, img, device)
        candidates = search_index(desc, center_lat, center_lon, radius_km, top_k=1)
        
        if candidates:
            best = candidates[0]
            results.append({
                "image": path,
                "lat": best["lat"],
                "lon": best["lon"],
                "confidence_proxy": best["similarity"],
            })
        else:
            results.append({"image": path, "lat": None, "lon": None})
    
    return results

# Usage
images = ["photo1.jpg", "photo2.jpg", "photo3.jpg"]
hits = batch_geolocate(images, center_lat=48.8566, center_lon=2.3522, radius_km=3.0)
for h in hits:
    print(f"{h['image']}: ({h['lat']:.5f}, {h['lon']:.5f}) sim={h.get('confidence_proxy', 0):.3f}")
```

---

## Common Patterns

### Pattern: OSINT — Unknown City, Use AI Coarse Mode
1. Set `GEMINI_API_KEY` env var
2. In GUI, choose **AI Coarse** — upload photo, Gemini infers country/region from visual cues
3. System auto-sets center coordinates; proceed with full search
4. If result confidence is low, manually refine center coordinates and re-run

### Pattern: Conflict/Event Monitoring
1. Index the full city/region in advance (larger radius, done once)
2. For each new image, use **Manual** mode with known general area, radius ~2–5km
3. Enable **Ultra Mode** for night/smoke/degraded footage
4. Result confidence score: >0.8 = high confidence, 0.5–0.8 = probable, <0.5 = verify manually

### Pattern: Multi-City Index
```python
# Index once, search anywhere — radius handles scoping
# Paris
python test_super.py  # Create, center=48.8566,2.3522, radius=5

# London  
python test_super.py  # Create, center=51.5074,-0.1278, radius=5

# All go into the same index/cosplace_descriptors.npy
# Search Paris: center=48.8566,2.3522, radius=5 → only Paris results
# Search London: center=51.5074,-0.1278, radius=5 → only London results
```

### Pattern: Large-Scale Indexing
Use `build_index.py` instead of the GUI for areas >5km radius:
```bash
python build_index.py
# Optimized for parallel panorama fetching and bulk descriptor extraction
# Saves incrementally to cosplace_parts/ — safe to interrupt and resume
```

---

## Troubleshooting

### GUI appears blank on macOS
```bash
brew install python-tk@3.11   # Match your exact Python version
```

### CUDA out of memory
- Reduce the number of candidates (lower `top_k` in Stage 1)
- Use DISK instead of ALIKED (fewer keypoints): DISK is auto-selected on MPS; on CUDA you can modify `test_super.py` to force DISK
- Close other GPU processes

### Low match confidence / wrong location
1. Ensure the indexed area actually covers the query location
2. Try **Ultra Mode** (enables LoFTR + descriptor hopping + neighborhood expansion)
3. Increase the search radius
4. For AI Coarse mode failures, switch to Manual with a known approximate location
5. Check that the query image is street-level (not aerial, not indoor)

### IndexError / empty index
The index hasn't been compiled yet from parts:
```bash
# The GUI auto-builds the index after Create Mode finishes.
# If parts exist but index/ is missing, re-run Create Mode 
# or run build_index.py to compile cosplace_parts/ → index/
python build_index.py
```

### LightGlue ImportError
```bash
# Must install from GitHub, not PyPI
pip uninstall lightglue -y
pip install git+https://github.com/cvg/LightGlue.git
```

### Slow indexing on CPU
- Indexing on CPU can take 3–5x longer than on GPU
- Use `build_index.py` for better CPU utilization
- Consider indexing smaller radius areas in batches

### Panorama download failures during search
- Network timeouts are normal; the pipeline retries automatically
- Check internet connection; Street View tile downloads require broadband
- Behind a proxy: set `HTTP_PROXY` / `HTTPS_PROXY` env vars before running

---

## Project Structure

```
netryx/
├── test_super.py          # Main GUI app — indexing + search + visualization
├── cosplace_utils.py      # CosPlace model loading + descriptor extraction
├── build_index.py         # High-performance standalone index builder
├── requirements.txt       # Python dependencies
├── cosplace_parts/        # Raw .npz chunks written during indexing
└── index/
    ├── cosplace_descriptors.npy   # Compiled descriptor matrix (N × 512)
    └── metadata.npz               # lats, lons, headings, panoids
```

## Models Reference

| Model | Role | Hardware |
|-------|------|----------|
| CosPlace (512-dim) | Global visual fingerprint | All |
| ALIKED (1024 kp) | Local keypoint extraction | CUDA only |
| DISK (768 kp) | Local keypoint extraction | MPS / CPU |
| LightGlue | Deep feature matching | All |
| LoFTR | Dense matching (Ultra Mode) | All (needs `kornia`) |
```
