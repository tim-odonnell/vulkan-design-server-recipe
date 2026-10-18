# 5. Images and video

## Engine

- **stable-diffusion.cpp**, built from source with `-DSD_VULKAN=ON` inside an Ubuntu 24.04
  Docker image (reference commit `2988060a`, October 2026). Its `sd-server` keeps a model loaded
  and offers a native async job API; `sd-cli` runs one job and exits.
- A second image tag adds **one small patch** for HQ video mode (see below). The normal image is
  untouched.
- Container flags: `--device /dev/dri`, `VK_DRIVER_FILES` pinned to the AMD ICD, model folder
  mounted read-only, port on 127.0.0.1 only.

## The render broker

A ~600-line FastAPI service that owns the render card. It:

- keeps **one queue, one job at a time** (a long video is never cut off by an image request);
- knows each **engine** (container + model files + flags) and each **model profile** (size,
  steps, sampler, guidance, frames per length) and swaps engines only when the next job needs a
  different one ("first job after a switch takes a few minutes");
- saves the queue to disk and resumes text-only jobs after a restart;
- converts video to H.264 (yuv420p, CRF 18, faststart) + AAC 128k MP4;
- serves a phone-friendly status page (auto-refresh) and the finished files;
- sends a notification for every finished or failed job;
- **cancel workaround:** sd-server ignores a cancel for a running LTX job, so if a cancel is not
  honoured within 30 s the broker removes the engine container (the next job reloads it);
- refuses work that does not fit the current mode (HTTP 503 with a plain-language reason).

Job API: `POST /api/image`, `POST /api/video` (prompt, seconds 1 to 5, model), `GET
/api/jobs[/id]`, `POST /api/jobs/{id}/cancel`, plus OpenAI-style `/v1/images/generations` and
`/v1/images/edits`.

## Image models (GPU 1, normal mode)

| Job | Model | Files | Settings | Measured |
|---|---|---|---|---|
| **Sketch to render** | FLUX.2 klein 4B | Q8_0 GGUF + Qwen3-4B Q8_0 text encoder + FLUX.2 VAE (8.9 GB) | Edit mode, sketch passed as a reference image, 4 steps, cfg 1, sketch aspect ratio at ~1 MP | 166 s at 1168×880, 8.3 GB VRAM; keeps the design, proportions and camera angle |
| **Text to image** | Qwen-Image 2.1 Turbo | Q8_0 GGUF + Qwen3-VL-8B Q8_0 text encoder + VAE (17 GB) | 8 steps, cfg 1, custom sigma list; img2img uses 16 steps | ~32 s at 640×352 warm, ~274 s at 1536×1024 |

The "Image studio" pipe: with an attached picture it runs sketch-to-render (after asking the
chat model to name the object and view in a few words); without one it runs text-to-image from
the user's exact words. [Licence check for each model before any redistribution: FLUX.2 klein is
Apache 2.0; Qwen-Image licence to confirm]

### Tried and retired (archived to the USB drive, with a write-up)

| Model | Why it was retired |
|---|---|
| Qwen-Image-Edit-2511 Q4_0 + Lightning 8-step | 563 s vs 166 s, put its 8 GB text encoder in RAM (3.4 GB left), mirrored the camera angle and turned a concept car into a near-production car |
| Qwen-Image Turbo img2img for sketches | At 8 steps strength is on/off: it either redraws the line art or loses the composition |
| Wan 2.1 1.3B (video) | Worked (2 s ≈ 400 s at 512×288) but clearly below LTX quality; replaced |

## Video model: LTX-2.3 22B (dev), with sound

Common settings: 640×352, 24 fps, Euler, **30 steps**, cfg 6, a fixed negative prompt, frame
counts of 8k+1 (1 s = 25, 2 s = 49, 3 s = 73, 4 s = 97, 5 s = 121). The text encoder is
Gemma 3 12B QAT (Q4_K_XL, 7.4 GB) plus LTX's embedding connectors (2.3 GB); video and audio
VAEs are shared. The output WebM (VP8 + PCM) becomes MP4 with AAC.

| Variant | Where | Placement | 2 s clip | 5 s clip | Memory |
|---|---|---|---|---|---|
| **Q4_0** (12.7 GB; set 24.3 GB) | Normal mode, phone/Open WebUI | Everything on GPU 1; encoder weights read from disk each job, VAE weights in RAM | ~18 min (1098 s) | ~40 min | GPU 1 peak 13.4 GB; RAM ≥ 8 GB free; chat ran at 49 tok/s on GPU 0 meanwhile |
| **Q8_0** (22.8 GB; set 34.3 GB) | HQ video mode only | DiT layer-split across both cards; encoder on GPU 1 only while encoding; engine restarted fresh per job | **20 min measured** (encoder 42 s, model load 64 s) | [Measure; estimate ~40 min] | Peak VRAM 14.6 / 11.8 GB; RAM ≥ 7.1 GB free with the CPU director loaded |

[Side-by-side quality comparison of Q4_0 and Q8_0 on the same prompt and seed]
[Optional x2 latent upscaler (tested earlier: 1280×704 for +59% time) is not wired in]
[Image-to-video from an approved render; shot list recipe for a 30-second film]

### Why HQ video needed a patch to stable-diffusion.cpp

1. With the text encoder on the **CPU**, it took 9 to 11 GB of the 16 GB RAM even with
   `--mmap`; with the director model loaded the machine went 6 GB into swap. Rejected.
2. With the encoder on **GPU 1**, the encoder is released after encoding, but the layer-split
   planner reads live free VRAM a moment later, before the driver reports the memory as free,
   and fails by ~120 MB ("segment 40 needs 5102 MB on Vulkan1, only 4984 MB available").
   `--max-vram`, disk-backed weights and `sd-cli` all failed the same way.
3. **Patch:** when a new split is planned, synchronise each GPU and re-read free VRAM for up to
   2 s, keeping the largest value. Applied as a ~40-line script during the image build; it
   aborts if the upstream code has changed. [Offer the patch upstream]
