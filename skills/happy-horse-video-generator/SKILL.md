```markdown
---
name: happy-horse-video-generator
description: AI coding agent skill for Happy Horse 1.0 — native joint audio-video generation model with unified Transformer architecture
triggers:
  - generate video with Happy Horse
  - use Happy Horse AI video model
  - happy horse video generation
  - joint audio video generation happy horse
  - run happy horse inference
  - happy horse text to video
  - happy horse image to video
  - happyhorses.io model
---

# Happy Horse 1.0 — AI Video Generator

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## ⚠️ Release Status

Happy Horse 1.0 has **not yet been officially open-sourced** as of this skill's writing. Model weights, inference code, and the official repository have not been published. This skill covers:

- Reported architecture and capabilities (community-compiled, unverified)
- Anticipated usage patterns based on similar open-source video models (Wan 2.2, HunyuanVideo, LTX-2)
- Scaffolding code to wire up Happy Horse once weights/code drop
- Pointers to official resources

**Official resources:**
- Homepage & demo: https://happyhorses.io/
- Leaderboard context: https://artificialanalysis.ai/video/arena

---

## What Happy Horse 1.0 Does

Happy Horse 1.0 is a ~15B-parameter unified self-attention Transformer that generates **video frames and audio together in a single forward pass** — no silent video + separate dubbing pipeline required.

Key reported capabilities:
- **Native joint audio-video generation** (dialogue, Foley, ambient sound)
- **1080p output** in ~38 seconds on NVIDIA H100 (distilled model)
- **8 denoising steps, no classifier-free guidance** (DMD-2 distillation)
- **Text-to-video and image-to-video** via a single set of weights
- **Native lip-sync** in English, Mandarin Chinese, Japanese, Korean, German, French

Architectural highlights:
- 40 layers total: first 4 + last 4 are modality-specific; middle 32 share parameters across text/image/video/audio
- Per-attention-head learned scalar sigmoid gates for multimodal training stability
- No explicit timestep embeddings — noise level inferred from noisy latents
- MagiCompiler runtime for full-graph operator fusion (~1.2× speedup)

---

## Installation (Anticipated — Update When Official Repo Releases)

Watch https://github.com/brooks376/Happy-Horse-1.0 and https://happyhorses.io/ for the official release. The following scaffolding mirrors conventions used by comparable models (HunyuanVideo, Wan 2.2).

```bash
# Clone the official repo once released
git clone https://github.com/<official-org>/happy-horse-1.0
cd happy-horse-1.0

# Create isolated environment
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install dependencies (anticipated)
pip install -r requirements.txt

# Download model weights (anticipated — likely via huggingface_hub)
pip install huggingface_hub
python -c "
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id='<official-org>/happy-horse-1.0',
    local_dir='./weights/happy-horse-1.0'
)
"
```

### Environment variables

```bash
# Set before running inference
export HF_TOKEN=your_huggingface_token          # if weights are gated
export HAPPY_HORSE_MODEL_PATH=./weights/happy-horse-1.0
export CUDA_VISIBLE_DEVICES=0                   # target GPU index
```

---

## Hardware Requirements

| Task | VRAM | Notes |
|---|---|---|
| 1080p generation (distilled) | 80 GB | NVIDIA H100 reference |
| 720p generation | ~40 GB (estimated) | A100 80GB likely sufficient |
| 256p preview | ~16 GB (estimated) | ~2 sec on H100 |

For multi-GPU setups, model parallelism will likely follow the same pattern as HunyuanVideo:

```bash
# Anticipated multi-GPU launch
torchrun --nproc_per_node=4 generate.py \
  --model_path $HAPPY_HORSE_MODEL_PATH \
  --prompt "A galloping horse across open plains at sunset" \
  --resolution 1080p \
  --steps 8
```

---

## Key Commands (Anticipated CLI)

Based on comparable open-source video model CLIs (Wan 2.2, HunyuanVideo):

```bash
# Text-to-video (distilled, 8 steps, no CFG)
python generate.py \
  --task t2v \
  --model_path $HAPPY_HORSE_MODEL_PATH \
  --prompt "A horse galloping through a misty forest, hooves thundering on wet earth" \
  --resolution 1080p \
  --duration 5 \
  --steps 8 \
  --output ./output/horse_forest.mp4

# Image-to-video
python generate.py \
  --task i2v \
  --model_path $HAPPY_HORSE_MODEL_PATH \
  --image ./reference.png \
  --prompt "The horse rears up and gallops away into the distance" \
  --resolution 1080p \
  --duration 5 \
  --steps 8 \
  --output ./output/horse_i2v.mp4

# Base model (50 steps, CFG — higher quality, slower)
python generate.py \
  --task t2v \
  --model_path $HAPPY_HORSE_MODEL_PATH \
  --variant base \
  --prompt "..." \
  --resolution 1080p \
  --steps 50 \
  --cfg_scale 7.5 \
  --output ./output/horse_base.mp4
```

---

## Python API (Anticipated Usage Patterns)

### Text-to-Video

```python
import os
import torch
from happy_horse import HappyHorsePipeline  # module name TBC at release

def generate_video_from_text(
    prompt: str,
    output_path: str,
    resolution: str = "1080p",
    duration_sec: int = 5,
    steps: int = 8,
    seed: int = 42,
) -> str:
    """
    Generate a video (with native audio) from a text prompt.
    Uses the distilled 8-step model by default.
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",  # or "base" for full 50-step model
    )
    pipe = pipe.to("cuda")

    result = pipe(
        prompt=prompt,
        resolution=resolution,
        duration=duration_sec,
        num_inference_steps=steps,
        guidance_scale=1.0,   # CFG not required for distilled model
        generator=torch.Generator("cuda").manual_seed(seed),
    )

    result.save(output_path)
    print(f"Saved video to {output_path}")
    return output_path


if __name__ == "__main__":
    generate_video_from_text(
        prompt="A joyful horse running along a beach, waves crashing, seagulls calling",
        output_path="./output/beach_horse.mp4",
    )
```

### Image-to-Video

```python
import os
import torch
from PIL import Image
from happy_horse import HappyHorsePipeline

def generate_video_from_image(
    image_path: str,
    prompt: str,
    output_path: str,
    resolution: str = "1080p",
    duration_sec: int = 5,
    steps: int = 8,
    seed: int = 42,
) -> str:
    """
    Animate a reference image into video with native audio.
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    pipe = pipe.to("cuda")

    reference_image = Image.open(image_path).convert("RGB")

    result = pipe(
        image=reference_image,
        prompt=prompt,
        resolution=resolution,
        duration=duration_sec,
        num_inference_steps=steps,
        guidance_scale=1.0,
        generator=torch.Generator("cuda").manual_seed(seed),
    )

    result.save(output_path)
    return output_path


if __name__ == "__main__":
    generate_video_from_image(
        image_path="./horse_portrait.png",
        prompt="The horse turns its head, snorts, and trots away",
        output_path="./output/horse_portrait_anim.mp4",
    )
```

### Extracting Audio and Video Separately

```python
import os
from happy_horse import HappyHorsePipeline
import torch

def generate_and_split_av(
    prompt: str,
    video_output: str = "./output/video_only.mp4",
    audio_output: str = "./output/audio_only.wav",
) -> tuple[str, str]:
    """
    Generate joint audio-video and export video and audio as separate files.
    Useful when you need the audio track independently (e.g. for further processing).
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    pipe = pipe.to("cuda")

    result = pipe(
        prompt=prompt,
        resolution="1080p",
        duration=5,
        num_inference_steps=8,
    )

    # Anticipated API — mirrors common video model output patterns
    result.export_video(video_output, include_audio=False)
    result.export_audio(audio_output)

    print(f"Video: {video_output}")
    print(f"Audio: {audio_output}")
    return video_output, audio_output
```

### Batch Generation

```python
import os
import torch
from happy_horse import HappyHorsePipeline
from pathlib import Path

def batch_generate(
    prompts: list[str],
    output_dir: str = "./output/batch",
    resolution: str = "720p",
    steps: int = 8,
) -> list[str]:
    """
    Generate multiple videos sequentially, reusing the loaded pipeline.
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]
    Path(output_dir).mkdir(parents=True, exist_ok=True)

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    pipe = pipe.to("cuda")

    outputs = []
    for i, prompt in enumerate(prompts):
        out_path = f"{output_dir}/video_{i:04d}.mp4"
        result = pipe(
            prompt=prompt,
            resolution=resolution,
            duration=5,
            num_inference_steps=steps,
            guidance_scale=1.0,
            generator=torch.Generator("cuda").manual_seed(i),
        )
        result.save(out_path)
        outputs.append(out_path)
        print(f"[{i+1}/{len(prompts)}] Saved: {out_path}")

    return outputs


if __name__ == "__main__":
    prompts = [
        "A horse grazing in a sunlit meadow, birds singing",
        "Two horses racing on a dirt track, crowd cheering",
        "A foal taking its first steps in a barn, soft whinnying",
    ]
    batch_generate(prompts)
```

---

## Configuration

Anticipated configuration file pattern (based on Wan 2.2 / HunyuanVideo conventions):

```yaml
# configs/happy_horse_1.0.yaml

model:
  name: happy-horse-1.0
  variant: distilled           # "distilled" (8-step) or "base" (50-step)
  dtype: bfloat16
  num_layers: 40
  num_shared_layers: 32        # middle layers with shared parameters
  num_heads: 32
  hidden_dim: 4096

inference:
  steps: 8                     # 8 for distilled, 50 for base
  guidance_scale: 1.0          # 1.0 = no CFG (distilled); 7.5 for base
  resolution: "1080p"
  duration: 5                  # seconds
  aspect_ratio: "16:9"         # or "9:16", "1:1"

audio:
  enabled: true
  sample_rate: 44100
  languages: ["en", "zh", "ja", "ko", "de", "fr"]

runtime:
  compiler: magicompiler        # full-graph compilation
  operator_fusion: true
  devices: [0]                  # GPU indices
```

Load config in code:

```python
import yaml
from happy_horse import HappyHorsePipeline

with open("configs/happy_horse_1.0.yaml") as f:
    config = yaml.safe_load(f)

pipe = HappyHorsePipeline.from_config(config)
pipe = pipe.to("cuda")
```

---

## Common Patterns

### Multilingual Lip-Sync Video

```python
import os
import torch
from happy_horse import HappyHorsePipeline

def generate_lipsync_video(
    speaker_image: str,
    dialogue_text: str,
    language: str = "en",  # "en", "zh", "ja", "ko", "de", "fr"
    output_path: str = "./output/lipsync.mp4",
) -> str:
    """
    Generate a talking-head video with native lip-sync.
    Happy Horse produces audio and lip movement jointly — no separate TTS or sync model needed.
    """
    assert language in ("en", "zh", "ja", "ko", "de", "fr"), (
        f"Unsupported language: {language}. "
        "Happy Horse 1.0 natively supports: en, zh, ja, ko, de, fr"
    )

    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]
    from PIL import Image

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    pipe = pipe.to("cuda")

    image = Image.open(speaker_image).convert("RGB")

    result = pipe(
        image=image,
        prompt=dialogue_text,
        language=language,
        lipsync=True,
        resolution="1080p",
        duration=len(dialogue_text.split()) // 2,  # rough duration estimate
        num_inference_steps=8,
    )

    result.save(output_path)
    return output_path
```

### Low-VRAM: 256p Preview Before Full Render

```python
import os
import torch
from happy_horse import HappyHorsePipeline

def preview_then_render(
    prompt: str,
    final_output: str = "./output/final.mp4",
) -> str:
    """
    Generate a fast 256p preview (~2 sec on H100), confirm it looks right,
    then render full 1080p.
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    pipe = pipe.to("cuda")

    # Fast preview pass
    preview = pipe(
        prompt=prompt,
        resolution="256p",
        duration=5,
        num_inference_steps=8,
    )
    preview.save("./output/preview.mp4")
    print("Preview saved to ./output/preview.mp4 — check before full render.")

    confirm = input("Looks good? Render full 1080p? [y/N]: ")
    if confirm.strip().lower() != "y":
        print("Aborted.")
        return ""

    result = pipe(
        prompt=prompt,
        resolution="1080p",
        duration=5,
        num_inference_steps=8,
    )
    result.save(final_output)
    return final_output
```

### Super-Resolution Post-Processing

```python
import os
import torch
from happy_horse import HappyHorsePipeline, HappyHorseSuperResolution

def generate_with_sr(
    prompt: str,
    output_path: str = "./output/sr_output.mp4",
) -> str:
    """
    Generate at 720p, then upscale to 1080p with the SR module.
    Useful when base generation VRAM is constrained.
    """
    model_path = os.environ["HAPPY_HORSE_MODEL_PATH"]

    pipe = HappyHorsePipeline.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
        variant="distilled",
    )
    sr = HappyHorseSuperResolution.from_pretrained(
        model_path,
        torch_dtype=torch.bfloat16,
    )
    pipe = pipe.to("cuda")
    sr = sr.to("cuda")

    base_result = pipe(
        prompt=prompt,
        resolution="720p",
        duration=5,
        num_inference_steps=8,
    )

    hires_result = sr(base_result, target_resolution="1080p")
    hires_result.save(output_path)
    return output_path
```

---

## Troubleshooting

### CUDA out of memory

```python
# Option 1: Use CPU offloading (slower but lower peak VRAM)
pipe.enable_model_cpu_offload()

# Option 2: Enable attention slicing
pipe.enable_attention_slicing(slice_size="auto")

# Option 3: Lower resolution or duration
result = pipe(prompt=prompt, resolution="720p", duration=3, num_inference_steps=8)

# Option 4: Use bfloat16 (should already be default)
pipe = HappyHorsePipeline.from_pretrained(model_path, torch_dtype=torch.bfloat16)
```

### Audio not included in output

```python
# Ensure audio is enabled (may be off by default in some configs)
result = pipe(
    prompt=prompt,
    resolution="1080p",
    generate_audio=True,   # explicit flag if needed
    num_inference_steps=8,
)
```

### Slow inference beyond 38 seconds

```python
# Ensure MagiCompiler is active
pipe = HappyHorsePipeline.from_pretrained(
    model_path,
    torch_dtype=torch.bfloat16,
    compile=True,            # enables MagiCompiler full-graph compilation
)

# Warm up the compiled graph before timed runs
_ = pipe(prompt="warmup", resolution="256p", duration=1, num_inference_steps=8)
```

### Using the base model (50 steps) for higher quality

```python
pipe = HappyHorsePipeline.from_pretrained(
    model_path,
    torch_dtype=torch.bfloat16,
    variant="base",          # NOT "distilled"
)
result = pipe(
    prompt=prompt,
    resolution="1080p",
    num_inference_steps=50,
    guidance_scale=7.5,      # CFG applies to base model
)
```

### Reproducible outputs

```python
import torch

generator = torch.Generator("cuda").manual_seed(12345)
result = pipe(
    prompt=prompt,
    num_inference_steps=8,
    generator=generator,
)
```

---

## Comparison to Current Open-Source Leaders

| Feature | Happy Horse 1.0 | Wan 2.2 A14B | LTX-2 Pro | HunyuanVideo-1.5 |
|---|---|---|---|---|
| Native audio | ✅ joint | ❌ | ❌ | ❌ |
| Parameters | ~15B | 14B | ~13B | ~13B |
| Steps (distilled) | 8, no CFG | ~50 | ~25 | ~50 |
| 1080p time (H100) | ~38s | minutes | minutes | minutes |
| Weights available | ❌ not yet | ✅ | ✅ | ✅ |

**Until Happy Horse weights are released**, use Wan 2.2 or LTX-2 for production work. Both are mature and have working inference stacks.

---

## Staying Updated

```bash
# Watch this info-collection repo for updates
# https://github.com/brooks376/Happy-Horse-1.0

# Official release channel
open https://happyhorses.io/

# Leaderboard placement (check when weights release)
open https://artificialanalysis.ai/video/arena
```

The anticipated release package includes:
1. Base Happy Horse 1.0 model weights
2. Distilled 8-step model weights
3. Super-resolution module weights
4. Inference code

License terms have not yet been published. Confirm licensing at https://happyhorses.io/ before production use.
```
