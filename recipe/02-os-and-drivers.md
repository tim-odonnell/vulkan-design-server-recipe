# 2. Operating system and drivers

## Base system

| Item | Reference unit |
|---|---|
| OS | Ubuntu 24.04.3 LTS (desktop, because the setup and two modes happen on the desktop) |
| Kernel | 7.0 (generic) [Record why this kernel; whether the stock 24.04 HWE kernel works equally] |
| GPU driver | amdgpu (in-kernel) + Mesa RADV 25.2.8 for Vulkan |
| Vulkan loader | Pin the AMD driver explicitly: `VK_DRIVER_FILES=/usr/share/vulkan/icd.d/radeon_icd.json` in every GPU process, so nothing silently runs on the Intel iGPU or a software rasteriser |
| Container runtime | Docker 29 (used for the image/video engine, search and notifications) |
| Remote desktop | Sunshine 2026.9 (host) and Moonlight (client) |
| Private network | Tailscale 1.104 |

## Why Vulkan only

- **llama.cpp** and **stable-diffusion.cpp** both have mature Vulkan back ends that run on
  gfx900 with no vendor stack.
- ROCm for gfx900 needs unofficial builds. In testing, two heavy video jobs on ROCm (one per
  card) **wedged both GPUs twice**, needing a reboot; the same workload on Vulkan only produced
  recoverable ring timeouts.
- One stack (GGUF files, Vulkan) for chat, images and video keeps the fleet simple.

## Device order

Inside processes and containers, the Vulkan device list here is: **Vulkan0 = PCI 03:00.0 (chat
card), Vulkan1 = PCI 07:00.0 (render card)**, once the Intel iGPU is hidden by pinning the AMD
ICD. Without the pin, index 0 can be the iGPU and a model silently runs at a fraction of the
speed. [Verify the order on each fleet unit: `llama-server --list-devices`]

## Memory behaviour you must know about (RADV)

- **RADV silently spills VRAM overflow into system RAM (GTT).** Nothing fails; the model just
  becomes slow and eats the 16 GB. Every context size in this recipe was chosen by watching
  `mem_info_vram_used` and `mem_info_gtt_used` in sysfs while filling the context.
- `RADV_PERFTEST=nogttspill` stops VRAM allocations from spilling. Used for the coding models.
- When llama.cpp splits one model across both cards (pipeline mode), it keeps a by-design
  host staging buffer of about 0.8 to 1 GB in RAM. That is expected, not a spill.

## Users, groups and services

- The service user is **not** in the `render` group, so host processes can use Vulkan but not
  `/dev/kfd`; GPU containers get `--device /dev/dri`.
- Every server part is a **systemd user service**. `loginctl enable-linger <user>` (one-time,
  needs sudo) makes them start at boot with nobody logged in. Measured: all services up 13 to
  16 s after boot, before the desktop auto-login.
- The two big-model modes (coding, HQ video) are user services that are **never enabled** at
  boot; their mode flags live in `/run/user/<uid>`, which is emptied at boot. A reboot is
  therefore always a return to normal mode.
- A small boot service sets the 100 W power cap on both cards.

## Desktop

- Auto-login to the desktop, so Sunshine always has a session to stream. [Decide and record the
  screen-lock policy, given auto-login]
- A dummy HDMI/DP plug may be needed for a headless unit's desktop resolution. [Test on a unit
  with no monitor attached]
