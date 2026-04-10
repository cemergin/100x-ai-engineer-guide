<!--
  CHAPTER: 41
  TITLE: Image Generation
  PART: 8 — Open Source AI & Inference
  PHASE: 2 — Become an Expert
  PREREQS: Ch 7 (Multimodal AI), Ch 38 (Transformers & Attention), Ch 40 (Hugging Face & Pipelines)
  KEY_TOPICS: stable diffusion, diffusion models, text-to-image, image-to-image, DreamBooth, prompt engineering for images, guidance scale, inference steps, attention slicing, fp16
  DIFFICULTY: Intermediate
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 41: Image Generation

> **Part 8 — Open Source AI & Inference** | Phase 2: Become an Expert | Prerequisites: Ch 7, Ch 38, Ch 40 | Difficulty: Intermediate | Language: Python

In Chapter 7, you generated images through APIs — send a text prompt to DALL-E or Midjourney, get an image back, pay per generation. In Chapter 40, you learned to run NLP models locally with Hugging Face. Now we combine those threads: you will generate images locally using Stable Diffusion, with no API calls, no per-image costs, and complete control over every parameter.

Diffusion models represent one of the most remarkable developments in AI. They start with pure noise and iteratively denoise it into a coherent image, guided by a text prompt. The process is beautiful, the results are stunning, and the math is — once you understand it — elegant. You do not need to understand every equation to use these models effectively, but understanding the core idea will make you much better at getting the images you want.

This chapter covers text-to-image generation, prompt engineering for images, image-to-image transformation, DreamBooth for teaching models new concepts, and the practical engineering of running these models on limited hardware.

### In This Chapter
- How diffusion models work: the noise-to-image pipeline
- Text-to-image with Stable Diffusion
- Prompt engineering for images: what works and what does not
- Negative prompts: telling the model what to avoid
- Image-to-image: transforming existing images with prompts
- Key parameters: inference steps, guidance scale, schedulers
- GPU memory management: fp16, attention slicing, VAE slicing
- DreamBooth: teaching a model new concepts

### Related Chapters
- **Ch 7 (Multimodal AI)** — spirals from API-based image generation to local
- **Ch 38 (Transformers & Attention)** — diffusion models use similar attention mechanisms
- **Ch 40 (Hugging Face & Pipelines)** — same ecosystem, different library (`diffusers`)
- **Ch 45 (Fine-Tuning with LoRA)** — fine-tune diffusion models with DreamBooth + LoRA

---

## 1. How Diffusion Models Work

### 1.1 The Core Idea

Imagine you have a photograph. Now imagine slowly adding random noise to it — a little bit, then a little more, then more, until it is pure static. Diffusion models learn to reverse this process. Given noisy static, they predict and remove the noise, step by step, until a clean image emerges.

```
Forward process (training):
  Clean image → slightly noisy → more noisy → ... → pure noise
  
Reverse process (generation):
  Pure noise → slightly less noisy → clearer → ... → clean image
```

The key insight: the model does not generate the entire image at once. It makes many small denoising steps, each one refining the image slightly. This is why you see images "emerge from noise" when you watch a diffusion model generate.

### 1.2 The Text Conditioning

Text-to-image models add a crucial component: text conditioning. The noise removal process is guided by a text description. At each denoising step, the model considers both the current noisy image and the text prompt to decide what noise to remove.

```
Text: "a cat sitting on a windowsill at sunset"
     ↓
Text Encoder (CLIP) → text embedding
     ↓
Denoising U-Net: takes (noisy image + text embedding) → less noisy image
     ↓
Repeat 20-50 times
     ↓
VAE Decoder: latent image → pixel image
```

The architecture has three main components:

1. **Text Encoder** — converts your text prompt into an embedding (usually CLIP)
2. **U-Net** — the denoising network that does the heavy lifting
3. **VAE** (Variational Autoencoder) — compresses images to/from a lower-dimensional latent space (this is why it is called "latent diffusion")

### 1.3 Why Latent Space?

Working directly with pixel images (512x512x3 = 786,432 values) would be computationally expensive. Instead, Stable Diffusion works in a compressed latent space (64x64x4 = 16,384 values). The VAE encoder compresses images into this space for training, and the VAE decoder expands them back to pixels after generation. This 48x compression is why Stable Diffusion can run on consumer GPUs.

---

## 2. Text-to-Image with Stable Diffusion

### 2.1 Setup

```python
# Install diffusers and related libraries
# !pip install diffusers transformers accelerate torch safetensors

import torch
from diffusers import StableDiffusionPipeline

# Check GPU availability
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"Memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")
```

### 2.2 Your First Image

```python
from diffusers import StableDiffusionPipeline
import torch

# Load Stable Diffusion 2.1
# On first run, this downloads ~5GB of model weights
model_id = "stabilityai/stable-diffusion-2-1"

pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    torch_dtype=torch.float16,  # Half precision — uses half the memory
)
pipe = pipe.to("cuda")

# Generate an image
prompt = "a photorealistic golden retriever puppy sitting in a field of wildflowers, soft natural lighting, shallow depth of field"

image = pipe(prompt).images[0]

# Save the image
image.save("first_generation.png")
print(f"Image size: {image.size}")
print("Image saved to first_generation.png")

# In a notebook, display it directly
# image  # Just put the image object as the last line in a cell
```

That is it. One prompt, one image, no API key, no cost per generation.

### 2.3 Using Stable Diffusion XL

For higher quality, use SDXL (Stable Diffusion XL). It generates 1024x1024 images by default and has significantly better prompt understanding.

```python
from diffusers import StableDiffusionXLPipeline
import torch

# SDXL — higher quality but needs more memory (~7GB VRAM minimum)
model_id = "stabilityai/stable-diffusion-xl-base-1.0"

pipe = StableDiffusionXLPipeline.from_pretrained(
    model_id,
    torch_dtype=torch.float16,
    variant="fp16",
    use_safetensors=True,
)
pipe = pipe.to("cuda")

# Enable memory-efficient attention
pipe.enable_xformers_memory_efficient_attention()

prompt = "an astronaut riding a horse on mars, digital art, cinematic lighting, highly detailed"

image = pipe(prompt).images[0]
image.save("sdxl_generation.png")
print(f"Image size: {image.size}")  # 1024x1024
```

> **Memory note:** SDXL requires about 7GB of VRAM in fp16. A Google Colab T4 (15GB VRAM) handles it fine. If you are on a lower-memory GPU, stick with SD 2.1 (needs about 4GB in fp16).

---

## 3. Understanding the Key Parameters

### 3.1 The Parameters That Matter

Every diffusion model has several parameters that dramatically affect the output. Understanding them is the difference between "random image generation" and "controlled image creation."

```python
from diffusers import StableDiffusionPipeline
import torch

pipe = StableDiffusionPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1",
    torch_dtype=torch.float16,
).to("cuda")

prompt = "a cozy coffee shop interior, warm lighting, rain outside the windows"

# Full parameter control
image = pipe(
    prompt=prompt,
    negative_prompt="blurry, low quality, distorted, deformed",
    num_inference_steps=50,     # Number of denoising steps
    guidance_scale=7.5,         # How closely to follow the prompt
    width=768,                  # Image width (multiple of 8)
    height=512,                 # Image height (multiple of 8)
    generator=torch.Generator(device="cuda").manual_seed(42),  # Reproducible
).images[0]

image.save("parameterized_generation.png")
```

### 3.2 Inference Steps: Quality vs Speed

The number of denoising steps controls how refined the image is.

```python
import time

prompt = "a detailed oil painting of a mountain landscape at dawn"
seed = 42

results = {}
for steps in [10, 20, 30, 50, 100]:
    start = time.time()
    generator = torch.Generator(device="cuda").manual_seed(seed)
    
    image = pipe(
        prompt=prompt,
        num_inference_steps=steps,
        generator=generator,
    ).images[0]
    
    elapsed = time.time() - start
    image.save(f"steps_{steps}.png")
    results[steps] = elapsed
    print(f"  {steps:3d} steps: {elapsed:.1f}s")

print("\nDiminishing returns after ~30 steps for most use cases")
```

Rules of thumb:
- **10-15 steps:** Fast preview, noticeable artifacts
- **20-30 steps:** Good quality for most uses
- **50 steps:** High quality, diminishing returns beyond this
- **100+ steps:** Marginal improvement, significantly slower

### 3.3 Guidance Scale: Prompt Adherence

The guidance scale (also called CFG scale — Classifier-Free Guidance) controls how strictly the model follows your prompt.

```python
prompt = "a red sports car driving through a neon-lit city at night"
seed = 123

for scale in [1.0, 3.0, 7.5, 12.0, 20.0]:
    generator = torch.Generator(device="cuda").manual_seed(seed)
    
    image = pipe(
        prompt=prompt,
        guidance_scale=scale,
        generator=generator,
    ).images[0]
    
    image.save(f"guidance_{scale}.png")
    print(f"  Scale {scale:5.1f}: saved")

print("""
Guidance scale effects:
  1.0  — Model mostly ignores the prompt (random/creative)
  3.0  — Loosely follows the prompt
  7.5  — Good balance (the default)
  12.0 — Very strict adherence, may lose some naturalness
  20.0 — Over-saturated, artifacts likely
""")
```

For most uses, 7-8 is the sweet spot. Go higher when the model is not generating what you described. Go lower for more creative/surprising results.

### 3.4 Negative Prompts

Negative prompts tell the model what to avoid. They are surprisingly effective.

```python
prompt = "portrait of a young woman, professional headshot, studio lighting"

# Without negative prompt
image_no_neg = pipe(
    prompt=prompt,
    generator=torch.Generator(device="cuda").manual_seed(42),
).images[0]

# With negative prompt
image_with_neg = pipe(
    prompt=prompt,
    negative_prompt=(
        "blurry, low quality, distorted, deformed, ugly, "
        "bad anatomy, bad proportions, extra limbs, "
        "watermark, signature, text"
    ),
    generator=torch.Generator(device="cuda").manual_seed(42),
).images[0]

image_no_neg.save("portrait_no_negative.png")
image_with_neg.save("portrait_with_negative.png")
print("Compare the two images — negative prompts make a real difference")
```

Common negative prompt elements:
- Quality: `blurry, low quality, low resolution, jpeg artifacts`
- Anatomy: `deformed, bad anatomy, extra fingers, missing limbs`
- Style: `cartoon, anime, 3d render` (when you want photorealism)
- Artifacts: `watermark, text, signature, frame, border`

### 3.5 Schedulers: The Denoising Algorithm

The scheduler controls how noise is removed at each step. Different schedulers produce different results and have different speed characteristics.

```python
from diffusers import (
    DDIMScheduler,
    PNDMScheduler,
    EulerDiscreteScheduler,
    EulerAncestralDiscreteScheduler,
    DPMSolverMultistepScheduler,
)

prompt = "a majestic castle on a cliff overlooking the ocean, sunset"
seed = 42

schedulers = {
    "DDIM": DDIMScheduler.from_pretrained("stabilityai/stable-diffusion-2-1", subfolder="scheduler"),
    "PNDM": PNDMScheduler.from_pretrained("stabilityai/stable-diffusion-2-1", subfolder="scheduler"),
    "Euler": EulerDiscreteScheduler.from_pretrained("stabilityai/stable-diffusion-2-1", subfolder="scheduler"),
    "Euler A": EulerAncestralDiscreteScheduler.from_pretrained("stabilityai/stable-diffusion-2-1", subfolder="scheduler"),
    "DPM++": DPMSolverMultistepScheduler.from_pretrained("stabilityai/stable-diffusion-2-1", subfolder="scheduler"),
}

for name, scheduler in schedulers.items():
    pipe.scheduler = scheduler
    generator = torch.Generator(device="cuda").manual_seed(seed)
    
    start = time.time()
    image = pipe(
        prompt=prompt,
        num_inference_steps=30,
        generator=generator,
    ).images[0]
    elapsed = time.time() - start
    
    image.save(f"scheduler_{name.replace(' ', '_').replace('+', 'p')}.png")
    print(f"  {name:10s}: {elapsed:.2f}s")

print("\nDPM++ is generally the best balance of quality and speed")
```

---

## 4. Prompt Engineering for Images

### 4.1 What Makes a Good Image Prompt

Image prompt engineering is different from text prompt engineering. You are describing a visual scene, and specific descriptors have outsized impact.

```python
# Structure of an effective image prompt:
# [Subject] + [Style/Medium] + [Lighting] + [Composition] + [Quality modifiers]

prompts = {
    "Basic (vague)": 
        "a cat",
    
    "Better (descriptive)": 
        "a tabby cat sitting on a wooden desk, looking at the camera",
    
    "Good (styled)": 
        "a tabby cat sitting on a wooden desk, looking at the camera, "
        "soft natural window light, shallow depth of field, "
        "professional pet photography",
    
    "Excellent (fully specified)": 
        "a tabby cat sitting on a vintage wooden desk covered with books, "
        "looking directly at the camera with curious green eyes, "
        "soft golden hour window light streaming from the left, "
        "shallow depth of field, bokeh background, "
        "shot on Canon EOS R5, 85mm f/1.4, "
        "professional pet photography, warm color palette",
}

for quality, prompt in prompts.items():
    image = pipe(
        prompt=prompt,
        negative_prompt="blurry, low quality, deformed",
        num_inference_steps=30,
        guidance_scale=7.5,
        generator=torch.Generator(device="cuda").manual_seed(42),
    ).images[0]
    
    image.save(f"prompt_quality_{quality.split()[0].lower()}.png")
    print(f"  {quality}: saved ({len(prompt)} chars)")
```

### 4.2 Effective Prompt Patterns

```python
# Pattern 1: Photography style
photo_prompt = (
    "street photography of a bustling Tokyo intersection at night, "
    "neon reflections on wet pavement, motion blur on pedestrians, "
    "shot on Leica M11, 35mm, Kodak Portra 800, grain"
)

# Pattern 2: Digital art
art_prompt = (
    "concept art of a floating city above the clouds, "
    "massive chains anchoring it to mountains below, "
    "dramatic volumetric lighting, epic scale, "
    "style of Simon Stalenhag, matte painting, 4k"
)

# Pattern 3: Product visualization
product_prompt = (
    "product photography of a minimalist ceramic coffee mug, "
    "matte sage green glaze, on a light marble surface, "
    "soft studio lighting, clean white background, "
    "commercial photography, 8k"
)

# Pattern 4: Illustration
illustration_prompt = (
    "watercolor illustration of a cozy autumn forest path, "
    "warm golden leaves, soft sunlight filtering through trees, "
    "gentle mist, Studio Ghibli inspired, "
    "hand-painted feel, soft edges"
)

for name, prompt in [
    ("photo", photo_prompt), 
    ("art", art_prompt),
    ("product", product_prompt), 
    ("illustration", illustration_prompt)
]:
    image = pipe(
        prompt=prompt,
        negative_prompt="blurry, low quality, deformed, text, watermark",
        num_inference_steps=30,
        guidance_scale=7.5,
        generator=torch.Generator(device="cuda").manual_seed(42),
    ).images[0]
    image.save(f"style_{name}.png")
    print(f"  {name}: saved")
```

### 4.3 What Does NOT Work

```python
# Common mistakes in image prompts:

bad_prompts = {
    "Too abstract": "happiness and joy in the morning",
    "Too much text instruction": "create an image that shows a person who is happy and make sure the lighting is good and the background is nice",
    "Contradictory": "a photorealistic cartoon illustration",
    "Relying on negation in prompt": "a dog that is not sitting",
    "Too many subjects": "a cat and a dog and a bird and a horse and a fish in a room with a table and chairs and a window",
}

print("Common prompt mistakes:")
for mistake, prompt in bad_prompts.items():
    print(f"\n  Problem: {mistake}")
    print(f"  Prompt:  '{prompt[:60]}...'")
```

Key lessons:
- Be specific and visual, not abstract
- Describe what you WANT, not what you do not want (use negative prompts for exclusions)
- Limit to 1-2 subjects per image
- Use concrete visual descriptors, not instructions

---

## 5. Image-to-Image Generation

### 5.1 Transforming Existing Images

Instead of starting from pure noise, you can start from an existing image and transform it based on a text prompt. This is incredibly useful for style transfer, editing, and iteration.

```python
from diffusers import StableDiffusionImg2ImgPipeline
from PIL import Image
import torch

# Load the img2img pipeline
pipe_img2img = StableDiffusionImg2ImgPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1",
    torch_dtype=torch.float16,
).to("cuda")

# Create a simple base image (in practice, you'd load a real image)
# Let's first generate a base image, then transform it
base_pipe = StableDiffusionPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1",
    torch_dtype=torch.float16,
).to("cuda")

base_image = base_pipe(
    "a simple sketch of a house with a garden",
    num_inference_steps=30,
    generator=torch.Generator(device="cuda").manual_seed(42),
).images[0]
base_image.save("base_sketch.png")

# Now transform it with different styles
styles = [
    ("watercolor", "a beautiful watercolor painting of a house with a garden, soft colors, artistic"),
    ("photorealistic", "a photorealistic house with a lush garden, golden hour, architectural photography"),
    ("cyberpunk", "a cyberpunk house with a neon garden, futuristic, rain, blade runner style"),
]

for style_name, prompt in styles:
    transformed = pipe_img2img(
        prompt=prompt,
        image=base_image,
        strength=0.75,          # How much to change (0 = keep original, 1 = full generation)
        num_inference_steps=30,
        guidance_scale=7.5,
    ).images[0]
    
    transformed.save(f"style_transfer_{style_name}.png")
    print(f"  {style_name}: saved")
```

### 5.2 The Strength Parameter

The `strength` parameter controls how much the model changes the input image:

```python
prompt = "a beautiful oil painting of a countryside landscape"
seed = 42

for strength in [0.2, 0.4, 0.6, 0.8, 1.0]:
    transformed = pipe_img2img(
        prompt=prompt,
        image=base_image,
        strength=strength,
        num_inference_steps=30,
        generator=torch.Generator(device="cuda").manual_seed(seed),
    ).images[0]
    
    transformed.save(f"strength_{strength}.png")
    print(f"  Strength {strength}: saved")

print("""
Strength effects:
  0.2 — Subtle changes, mostly preserves original
  0.4 — Noticeable style transfer, structure preserved
  0.6 — Significant transformation, some structure remains
  0.8 — Major transformation, original barely recognizable
  1.0 — Equivalent to text-to-image (original ignored)
""")
```

---

## 6. GPU Memory Management

### 6.1 The Memory Challenge

Diffusion models are memory-hungry. Here are approximate VRAM requirements:

```
Model                    | fp32     | fp16     | With optimizations
-------------------------|----------|----------|-------------------
Stable Diffusion 1.5     | ~8 GB   | ~4 GB   | ~3 GB
Stable Diffusion 2.1     | ~10 GB  | ~5 GB   | ~4 GB
Stable Diffusion XL      | ~14 GB  | ~7 GB   | ~5 GB
```

If your GPU does not have enough memory, you will see `OutOfMemoryError`. Here are the solutions, from easiest to most involved.

### 6.2 Memory Optimization Techniques

```python
from diffusers import StableDiffusionPipeline
import torch

model_id = "stabilityai/stable-diffusion-2-1"

# Optimization 1: Use fp16 (half precision) — the most impactful single change
pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    torch_dtype=torch.float16,  # 50% memory reduction
).to("cuda")

# Optimization 2: Enable attention slicing — processes attention in chunks
pipe.enable_attention_slicing()
# Slight speed reduction (~10%), significant memory savings

# Optimization 3: Enable VAE slicing — processes VAE in chunks
pipe.enable_vae_slicing()
# Useful for generating multiple images or large images

# Optimization 4: Enable model CPU offloading — move unused components to CPU
# pipe.enable_model_cpu_offload()  
# Much slower but dramatically reduces VRAM usage

# Optimization 5: Enable sequential CPU offloading (most aggressive)
# pipe.enable_sequential_cpu_offload()
# Slowest but can run on GPUs with as little as 2GB VRAM

# Check actual memory usage
if torch.cuda.is_available():
    print(f"GPU memory allocated: {torch.cuda.memory_allocated() / 1e9:.2f} GB")
    print(f"GPU memory reserved: {torch.cuda.memory_reserved() / 1e9:.2f} GB")
```

### 6.3 xformers for Memory-Efficient Attention

```python
# xformers provides a more memory-efficient attention implementation
# !pip install xformers

try:
    pipe.enable_xformers_memory_efficient_attention()
    print("xformers enabled — memory-efficient attention active")
except Exception as e:
    print(f"xformers not available: {e}")
    print("Falling back to standard attention")
    pipe.enable_attention_slicing()
```

### 6.4 Generating Multiple Images Efficiently

```python
prompt = "a serene Japanese zen garden, morning mist, raked sand, stones"

# Generate a batch of 4 images
# Note: this needs 4x the memory of a single image
# Use VAE slicing to help
pipe.enable_vae_slicing()

images = pipe(
    prompt=[prompt] * 4,  # Same prompt 4 times
    num_inference_steps=30,
    guidance_scale=7.5,
    generator=torch.Generator(device="cuda").manual_seed(42),
).images

for i, img in enumerate(images):
    img.save(f"batch_{i}.png")
    print(f"  Image {i}: saved ({img.size})")

# If memory is tight, generate one at a time with different seeds
for i in range(4):
    image = pipe(
        prompt=prompt,
        num_inference_steps=30,
        generator=torch.Generator(device="cuda").manual_seed(42 + i),
    ).images[0]
    image.save(f"sequential_{i}.png")
```

---

## 7. DreamBooth: Teaching a Model New Concepts

### 7.1 What Is DreamBooth?

DreamBooth lets you teach a diffusion model a new concept (a specific person's face, your pet, a product, a style) using just 3-10 images. After training, you can generate that concept in any context.

The idea: you fine-tune the model on your images, associating them with a unique identifier token (like `sks`). Then you can prompt "a photo of sks dog on the moon" and get your specific dog on the moon.

### 7.2 DreamBooth with Diffusers

Here is a simplified training script. In practice, you would use the `diffusers` training scripts or the `autotrain` tool.

```python
# DreamBooth training requires more setup than other examples
# This shows the conceptual workflow — for full training,
# see: https://huggingface.co/docs/diffusers/training/dreambooth

# !pip install diffusers accelerate transformers bitsandbytes

# Step 1: Prepare your training images
# You need 3-10 images of your subject
# All images should show the subject clearly
# Variety in pose/angle/lighting helps

import os
from pathlib import Path

# Create a directory for training images
train_dir = Path("dreambooth_training_data")
train_dir.mkdir(exist_ok=True)

print(f"""
DreamBooth Training Setup:
  1. Place 3-10 images of your subject in: {train_dir}/
  2. Choose a unique identifier (e.g., 'sks')
  3. Choose a class name (e.g., 'dog', 'person', 'style')

Your prompt template will be: 'a photo of sks dog'
  - 'sks' = your unique identifier (not a real word)
  - 'dog' = the class, so the model knows what kind of thing it is

Training will take 10-30 minutes on a T4 GPU.
""")

# Step 2: Training configuration (conceptual)
training_config = {
    "pretrained_model": "stabilityai/stable-diffusion-2-1",
    "instance_data_dir": str(train_dir),
    "instance_prompt": "a photo of sks dog",
    "class_prompt": "a photo of a dog",           # For prior preservation
    "num_class_images": 200,                       # Auto-generated class images
    "learning_rate": 5e-6,
    "max_train_steps": 800,
    "train_batch_size": 1,
    "gradient_accumulation_steps": 1,
    "lr_scheduler": "constant",
    "mixed_precision": "fp16",
}

for key, value in training_config.items():
    print(f"  {key}: {value}")
```

### 7.3 Using a DreamBooth Model

```python
# After training, load and use the model like any other SD model
from diffusers import StableDiffusionPipeline
import torch

# In practice, you'd point to your trained model directory
# pipe = StableDiffusionPipeline.from_pretrained(
#     "path/to/dreambooth/model",
#     torch_dtype=torch.float16,
# ).to("cuda")

# Generate images of your subject in new contexts
dreambooth_prompts = [
    "a photo of sks dog wearing a tiny astronaut helmet, space background",
    "a photo of sks dog painted by Claude Monet, impressionist style",
    "a photo of sks dog sitting in a cozy cafe, warm lighting, cozy atmosphere",
    "a watercolor illustration of sks dog, children's book style",
]

# for prompt in dreambooth_prompts:
#     image = pipe(
#         prompt=prompt,
#         negative_prompt="blurry, low quality, deformed",
#         num_inference_steps=50,
#         guidance_scale=7.5,
#     ).images[0]
#     filename = prompt.replace(" ", "_")[:50] + ".png"
#     image.save(filename)
#     print(f"  Generated: {filename}")

print("DreamBooth lets you place YOUR subject in ANY context")
print("Common uses: product photography, personalized avatars, style transfer")
```

### 7.4 DreamBooth + LoRA (Memory-Efficient)

Full DreamBooth fine-tuning modifies all model weights and requires significant memory. DreamBooth with LoRA only trains small adapter layers, reducing memory requirements dramatically.

```python
# DreamBooth + LoRA training (conceptual configuration)
lora_config = {
    "pretrained_model": "stabilityai/stable-diffusion-2-1",
    "instance_prompt": "a photo of sks dog",
    "use_lora": True,
    "lora_rank": 4,                    # Rank of the LoRA matrices
    "learning_rate": 1e-4,             # Higher LR for LoRA
    "max_train_steps": 500,            # Fewer steps needed
    "train_batch_size": 1,
    "mixed_precision": "fp16",
}

print("DreamBooth + LoRA advantages:")
print(f"  Full DreamBooth model size: ~5 GB")
print(f"  LoRA adapter size:          ~3 MB")
print(f"  Training memory:            ~10 GB vs ~24 GB")
print(f"  Training time:              ~10 min vs ~30 min")
print(f"  Quality:                    Slightly lower but very good")
print()
print("LoRA adapters are tiny — you can train dozens of concepts")
print("and swap between them without re-downloading the base model")
```

> **Spiral forward:** We will cover LoRA in detail in Ch 45 (Fine-Tuning with LoRA). The same technique applies to both text and image models.

---

## 8. Putting It All Together: An Image Generation Pipeline

```python
from diffusers import StableDiffusionPipeline, DPMSolverMultistepScheduler
import torch
from dataclasses import dataclass
from typing import Optional
from pathlib import Path


@dataclass
class ImageConfig:
    """Configuration for image generation."""
    prompt: str
    negative_prompt: str = "blurry, low quality, deformed, distorted, watermark, text"
    width: int = 512
    height: int = 512
    steps: int = 30
    guidance_scale: float = 7.5
    seed: Optional[int] = None
    num_images: int = 1


class ImageGenerator:
    """A reusable image generation pipeline with sensible defaults."""
    
    def __init__(self, model_id="stabilityai/stable-diffusion-2-1"):
        print(f"Loading model: {model_id}")
        
        self.pipe = StableDiffusionPipeline.from_pretrained(
            model_id,
            torch_dtype=torch.float16,
        )
        
        # Use DPM++ solver for best quality/speed
        self.pipe.scheduler = DPMSolverMultistepScheduler.from_config(
            self.pipe.scheduler.config
        )
        
        self.pipe = self.pipe.to("cuda")
        self.pipe.enable_attention_slicing()
        
        print("Model loaded and optimized!")
    
    def generate(self, config: ImageConfig) -> list:
        """Generate images from a configuration."""
        generator = None
        if config.seed is not None:
            generator = torch.Generator(device="cuda").manual_seed(config.seed)
        
        result = self.pipe(
            prompt=[config.prompt] * config.num_images,
            negative_prompt=[config.negative_prompt] * config.num_images,
            width=config.width,
            height=config.height,
            num_inference_steps=config.steps,
            guidance_scale=config.guidance_scale,
            generator=generator,
        )
        
        return result.images
    
    def generate_variations(self, prompt: str, num_variations: int = 4, 
                           base_seed: int = 42) -> list:
        """Generate multiple variations of the same prompt."""
        images = []
        for i in range(num_variations):
            config = ImageConfig(
                prompt=prompt,
                seed=base_seed + i,
            )
            images.extend(self.generate(config))
        return images
    
    def save_images(self, images: list, output_dir: str = "output", 
                    prefix: str = "gen"):
        """Save generated images to disk."""
        path = Path(output_dir)
        path.mkdir(exist_ok=True)
        
        saved = []
        for i, img in enumerate(images):
            filename = path / f"{prefix}_{i:03d}.png"
            img.save(filename)
            saved.append(str(filename))
        
        return saved


# Usage
if torch.cuda.is_available():
    gen = ImageGenerator()
    
    # Single image with full control
    config = ImageConfig(
        prompt="a futuristic library with holographic books, sci-fi interior design, volumetric lighting",
        width=768,
        height=512,
        steps=30,
        seed=42,
    )
    
    images = gen.generate(config)
    paths = gen.save_images(images, prefix="library")
    print(f"Saved: {paths}")
    
    # Multiple variations
    images = gen.generate_variations(
        "a cozy treehouse in an enchanted forest, fairy lights, magical atmosphere",
        num_variations=4,
    )
    paths = gen.save_images(images, prefix="treehouse")
    print(f"Saved {len(paths)} variations")
else:
    print("GPU required for image generation — run this on Google Colab!")
```

---

## 9. Key Takeaways

1. **Diffusion models denoise.** They start from random noise and iteratively refine it into an image, guided by your text prompt. Understanding this helps you tune parameters.

2. **Parameters matter enormously.** `num_inference_steps` controls quality/speed. `guidance_scale` controls prompt adherence. `strength` (img2img) controls how much to change the original.

3. **Negative prompts are essential.** Always include quality-related negative prompts: `blurry, low quality, deformed, distorted`.

4. **Prompt like a photographer.** Describe subject, medium/style, lighting, composition, and quality modifiers. Be specific and visual.

5. **fp16 is mandatory in practice.** Always use `torch_dtype=torch.float16`. It halves memory usage with negligible quality loss.

6. **DreamBooth personalizes.** 3-10 images can teach a model your specific subject. DreamBooth + LoRA makes this memory-efficient and produces tiny adapter files.

7. **Memory management is engineering.** Attention slicing, VAE slicing, xformers, and CPU offloading let you run models on hardware that would otherwise be too small.

---

## What's Next

In **Chapter 42: Embeddings & Sentence Transformers**, we go from generating content (text and images) to understanding it. You will generate embeddings locally — the same kind of vectors you used for semantic search in Part 3, but now you control the model, the quality, and the cost.

Then in **Chapter 43**, we synthesize everything from Part 8 into a comprehensive model selection framework: when to use BERT, when to use GPT, when to use T5, and how to choose between the hundreds of thousands of models on the Hub.
