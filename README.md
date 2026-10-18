# Vulkan Design Server: a build recipe

A description of a 24/7 home AI server for a product designer, built from **two used AMD
Radeon Pro WX 9100 cards (Vega, gfx900), an office tower and 16 GB of RAM**, running
everything through **Vulkan** (no ROCm, no CUDA).

It chats and does web research with cited sources, turns sketches into photographic renders,
makes text-to-image pictures, makes short product videos with sound, and runs long coding
sessions, from a phone or laptop anywhere over a private network.

> **Status: ROUGH DRAFT.** Everything not in `[brackets]` was built and measured on the
> reference machine in October 2026. Everything in `[brackets]` is a gap: the data still
> needs to be collected, or the part is not built yet. [MISSING-DATA.md](MISSING-DATA.md)
> collects every gap in one list.

## What this repo is (and is not)

- **It is** a recipe: what the server is, what each part does, how the parts talk to each
  other, which models and settings were chosen, what was measured, and what failed and why.
  It is specific about hardware, versions, ports, memory budgets and timings.
- **It is not** a line-by-line install script or a copy of the source code. Someone rebuilding
  it (or an AI coding agent they direct) writes their own code from this description.
- **It is designed to be used as a multi-stage prompt.** [PROMPT-STAGES.md](PROMPT-STAGES.md)
  splits the build into stages. Each stage has a goal, the recipe pages to read, the
  constraints, and "done when" checks. Hand one stage at a time to a capable coding agent
  working on the target machine.

## Read in this order

| # | Page | What it covers |
|---|---|---|
| 1 | [Hardware](recipe/01-hardware.md) | Components, PCIe layout, power caps, power draw, thermals |
| 2 | [OS and drivers](recipe/02-os-and-drivers.md) | Ubuntu, kernel, Mesa RADV, Vulkan-only rules, Docker, boot without login |
| 3 | [Architecture](recipe/03-architecture.md) | Every service, port and data path; which GPU does what in each mode |
| 4 | [LLM serving](recipe/04-llm-serving.md) | Models, quants, contexts, KV cache, sampling, measured speeds |
| 5 | [Images and video](recipe/05-images-and-video.md) | The render broker, image and video models, timings, the sd.cpp patch |
| 6 | [Modes](recipe/06-modes.md) | Normal, coding and HQ video modes, and how the machine switches between them |
| 7 | [Web research](recipe/07-web-research.md) | Self-hosted search, forced search, citations |
| 8 | [Access and clients](recipe/08-access-and-clients.md) | Private network, phone, laptop terminal, notifications, remote desktop |
| 9 | [Reliability](recipe/09-reliability.md) | GPU watchdog, reboot behaviour, memory limits, known faults |
| 10 | [Gift and fleet](recipe/10-gift-and-fleet.md) | Cloning to many identical units, first-day setup without a monitor |
| 11 | [Benchmarks](recipe/11-benchmarks.md) | Every measured number in one place |
| 12 | [Lessons and pitfalls](recipe/12-lessons-and-pitfalls.md) | What went wrong, and the rule each failure produced |

## The design in one paragraph

GPU 0 holds the main chat model (Qwen3.6-35B-A3B at a 3-bit quant, 128K context) all the time,
so chat is always available. GPU 1 is a render card: a small broker queues one image or video
job at a time and starts the right stable-diffusion.cpp engine for it. Open WebUI is the front
door for phones and laptops, reached only over a private Tailscale network. Two opt-in desktop
modes take over both GPUs: **coding mode** (OpenCode with a bigger model split across both
cards) and **HQ video mode** (LTX-2.3 at 8-bit across both cards, directed by a small model on
the CPU). A reboot always returns to normal mode.

## Privacy

This repo contains no names, addresses, serial numbers, keys, tailnet names or IP addresses.
Use placeholders such as `<server-name>` and `<tailnet>` for your own values.

## Licence

[Owner to choose a licence for this documentation, for example CC BY 4.0. Model licences are
listed per model in the recipe and must be checked before redistributing any model file.]
