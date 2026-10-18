# Multi-stage build prompt

Use this file to rebuild the server with a capable AI coding agent working **on the target
machine**. Give it the **standing instructions** once, then one stage at a time. Do not move to
the next stage until every "done when" check passes. Each stage names the recipe pages the
agent must read first.

## Standing instructions (paste first, keep for every stage)

```text
You are building a "Vulkan Design Server": a 24/7 home AI server on two AMD Radeon Pro WX 9100
GPUs (gfx900, 16 GB each), an Intel CPU and 16 GB RAM, running Ubuntu 24.04. The full design is
in this repository's recipe/ folder; treat it as the specification and read the pages each stage
names before acting.

Rules:
- Vulkan only (llama.cpp and stable-diffusion.cpp, GGUF files). No ROCm, no CUDA.
- Pin the AMD Vulkan driver (VK_DRIVER_FILES) in every GPU process.
- RAM is fixed at 16 GB. Before choosing any placement or context size, measure VRAM and GTT
  (system RAM) use in sysfs while the model is fully loaded and its context is filled. Keep at
  least ~6 GB of RAM free during every workload.
- Every service listens on 127.0.0.1 only. Outside access is only through Tailscale serve.
- Every service is a systemd user service. Normal-mode services start at boot without a login;
  coding and HQ video services are never enabled at boot.
- Keep all scripts, configs and test results in the project repository, never in /tmp.
- Ask the owner before anything destructive or hard to undo, and before any trade-off between
  speed, context, quality and safety. Record each decision and its reason in the project plan.
- Report measured numbers, not expectations. If a test fails, say so with the output.
- Never store or print secrets in files that may be shared; keep API keys in one mode-600 file.
```

## Stage 1: Hardware and OS baseline

Read: `01-hardware.md`, `02-os-and-drivers.md`.
Do: confirm both GPUs, their PCI addresses, slot widths and Vulkan order; set the 100 W power cap
at boot; pin the Vulkan ICD; enable linger; record the hardware table for this unit, filling the
bracketed gaps you can measure.
Done when: `llama-server --list-devices` shows exactly the two WX 9100s in a known order, the
cap survives a reboot, and a user service starts at boot with nobody logged in.

## Stage 2: Chat model service

Read: `04-llm-serving.md`, `03-architecture.md`.
Do: install the pinned llama.cpp Vulkan build and llama-swap; serve Qwen3.6-35B-A3B UD-IQ3_S on
GPU 0 with vision, 2 slots, unified 16-bit KV; add the 27B and Gemma 12B as swappable options;
add the thinking-off alias; set makers' sampling and the answer/thinking caps; API keys per device.
Done when: a context-fill test at the chosen size passes recall with less than ~1 GB in GTT, two
clients get more than 10 tok/s each, and an image question is answered correctly.

## Stage 3: Front door and private access

Read: `03-architecture.md`, `08-access-and-clients.md`.
Do: Open WebUI in Docker bound to localhost, pointed at llama-swap; Tailscale serve for the UI and
the API; the laptop aichat setup script.
Done when: a phone on the tailnet chats in Open WebUI and the laptop chats with aichat, and
nothing is reachable from the LAN without Tailscale.

## Stage 4: Web research

Read: `07-web-research.md`.
Do: SearXNG in Docker; Open WebUI web search with forced search, bypassed embedding, 8 results.
Done when: a question about something from the last month returns a summary with working source
links, and the server logs show the search actually ran.

## Stage 5: Render broker and images

Read: `05-images-and-video.md`, `03-architecture.md`.
Do: build the stable-diffusion.cpp Vulkan image; write the broker (queue, engines, model
profiles, saved state, status page, files, job API); install FLUX.2 klein and Qwen-Image Turbo;
write the Open WebUI "Image studio" pipe.
Done when: a sketch uploaded from the phone comes back as a photographic render that keeps the
design and camera angle, text-to-image works, and chat stays fast while GPU 1 renders.

## Stage 6: Video and notifications

Read: `05-images-and-video.md`, `08-access-and-clients.md`.
Do: LTX-2.3 Q4_0 engine on GPU 1 (encoder from disk, VAE weights in RAM), MP4 + AAC
conversion, the "Video" pipe, the cancel workaround; ntfy in Docker, Tailscale serve on 8090,
a notification for every finished or failed job.
Done when: a 2 s video with sound is made from the phone in about 18 minutes while chat keeps
answering, and the phone shows "Video ready".

## Stage 7: Reliability

Read: `09-reliability.md`.
Do: the GPU watchdog with its dry-run mode; the after-reboot check script.
Done when: the watchdog's dry run shows the right action for each card and mode, and after a
reboot the check script prints all PASS.

## Stage 8: Coding mode

Read: `06-modes.md`, `04-llm-serving.md`.
Do: the mode-switch script with a lock and flags; a second llama-swap with the two Q4_K_M
coding models split across both GPUs (16-bit KV, MTP, nogttspill); OpenCode with its own config
and ask-every-time permissions; the two desktop icons; broker refusal while coding.
Done when: Launch → OpenCode reads and fixes a file in a test project; the phone gets a clear
"coding mode" refusal; Stop brings everything back; a reboot comes back in normal mode.

## Stage 9: HQ video mode

Read: `06-modes.md`, `05-images-and-video.md`.
Do: the patched stable-diffusion.cpp image; the Q8 engine (split across both cards, encoder on
GPU 1, fresh per job); the CPU director model; the director page (approval cards, two-step blind
review, page and slide builder, project-folder-only file access, no shell); the two desktop icons.
Done when: a 2 s Q8 video renders with no GPU faults and at least ~6 GB of RAM free, the
director's review names the object actually shown, and Stop returns to normal mode.

## Stage 10: User guides

Read: every page.
Do: plain-language guides for each thing the recipient does (phone notifications, making a
video, coding mode, HQ video mode, sketch to render, research, laptop, Moonlight, "something is
wrong").
Done when: a person who has never used Linux follows each guide without help. [Guides for sketch
to render, research, laptop, Moonlight and troubleshooting still to write]

## Stage 11: Install script and clone

Read: `10-gift-and-fleet.md`.
Do: turn every stage above into an unattended install on a fresh disk, with per-unit values
asked for or detected.
Done when: a fresh unit passes Stages 1 to 9's checks with no manual steps except signing in.
[Not started on the reference build]
