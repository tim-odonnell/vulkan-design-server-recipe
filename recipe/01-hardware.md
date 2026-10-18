# 1. Hardware

The reference machine is a used office workstation with two used workstation GPUs. The
recipe targets a **fleet** of identical-class units: Intel 9th to 11th generation CPU, 16 GB
DDR4, two WX 9100, no monitor or keyboard.

## Components (reference unit)

| Part | Reference unit | Notes |
|---|---|---|
| Chassis | Dell Precision 3650 Tower | [Chassis model, chipset and BIOS version for each fleet unit] |
| CPU | Intel Core i5-11400T (Rocket Lake), 6 cores / 12 threads, 1.3 GHz base, 3.7 GHz max, 12 MB L3, 35 W TDP | AVX2 and AVX-512 capable. 6 threads measured best for CPU inference (12 is ~15% slower) |
| Integrated GPU | Intel UHD Graphics 730 | **Drives the desktop**, so both WX 9100s stay compute-only with their full 16 GB. A unit without an iGPU loses VRAM to the desktop; re-measure contexts there |
| RAM | 16 GB DDR4 | **Fixed at 16 GB by design** (the whole recipe is budgeted for it). [Speed, number of DIMMs, single or dual channel: `sudo dmidecode -t memory`] |
| GPU 0 | AMD Radeon Pro WX 9100, 16 GB HBM2 | PCI 03:00.0, CPU-attached PCIe 3.0 x16 slot. Holds the chat model |
| GPU 1 | AMD Radeon Pro WX 9100, 16 GB HBM2 | PCI 07:00.0, chipset (PCH) slot, PCIe 3.0 **x4** electrically, shares the DMI link with the NVMe and USB. Render card. [Confirm the electrical width with `sudo lspci -vv`; sysfs reports the card's internal switch link, not the slot] |
| Storage | 512 GB NVMe SSD | About 140 GB of model files in use. Swap file on the same NVMe |
| Archive | External USB drive | Holds retired models and backups ("restorable" archive) |
| Network | Onboard Intel I219-LM gigabit Ethernet (used for the direct-cable setup) + USB 802.11ac Wi-Fi adapter (TP-Link) | Recipients have no home Ethernet: the server lives on Wi-Fi. [Exact Wi-Fi adapter model and measured throughput] |
| Setup cable | Ethernet cable, laptop to server | A USB-to-Ethernet adapter on the laptop side if it has no port. [Adapter model shipped with each unit] |
| PSU | [Wattage from the PSU label; an earlier note says 1000 W, which is unusual for this tower and must be verified] | Two cards at the 100 W cap plus the system have measured well inside the supply's limit [Wall-meter figure needed] |
| Laptop (ships with the server) | [Model] | Dual boot: Ubuntu (default) and Windows 11 |

## The GPU in detail

| Property | Radeon Pro WX 9100 |
|---|---|
| Architecture | GCN 5 "Vega 10 XT", LLVM target **gfx900** |
| Memory | 16 GB HBM2, 2048-bit, about 484 GB/s |
| Compute | 64 CUs, 4096 stream processors; about 12.3 TFLOPS FP32, 24.6 TFLOPS FP16 (packed math) |
| What it lacks | No tensor/matrix cores, no int8 dot-product instructions, no bf16. Formats such as W4A8 or INT8 do not speed it up |
| Vulkan driver | Mesa **RADV** ("VEGA10"); fp16 yes, bf16 no, warp size 64, 64 KB shared memory |
| ROCm | Officially dropped for gfx900; community builds work but wedged both cards twice under dual-GPU video load. **Not used** |
| Board power | Driver default limit 170 W. **Capped to 100 W per card** at every boot by a small systemd service |
| Price | [Typical used price per card at build time] |

## Power

| State | GPU board power (sysfs, per card) | Wall power (whole machine) |
|---|---|---|
| Idle, models loaded | about 5 W | [Measure with a wall meter] |
| Chat generating (GPU 0 only) | [Measure] | [Measure] |
| Image render (GPU 1) | [Measure] | [Measure] |
| Video render, LTX Q4_0 (GPU 1) | 94 to 105 W on the busy card (at the 100 W cap) | [Measure] |
| Coding mode, long prompt (both GPUs) | [Measure] | [Measure] |
| HQ video mode, LTX Q8 (both GPUs) | [Measure] | [Measure] |
| CPU-only director model answering | n/a | [Measure; CPU package power needs root to read RAPL] |
| Yearly energy at a typical duty cycle | | [Estimate from the measurements above] |

The cards spend much of a heavy job throttled at the 100 W cap (852 MHz about half the time
during early video tests). Raising the cap is a possible speed gain that has **not** been
tested for stability.

## Thermals

| Case | Edge | Junction | HBM |
|---|---|---|---|
| Both cards rendering video at once, GPU 0 (x16 slot) | 85 °C | 92 °C | 92 °C |
| Same test, GPU 1 | 80 °C | 87 °C | 84 °C |
| HQ video mode, short probe | [Measure] | [Measure] | [Measure] |

Limits reported by the driver are about 85 °C edge and 95 °C HBM, so two cards both busy for
hours is the worst case. [Room temperature during the tests; fan noise at idle and under
load in dB, for a recipient's home office]

## Fleet notes

- Every unit must have **two WX 9100s and 16 GB**; the memory budgets in this recipe have no
  slack for less.
- **PCI addresses differ between chassis.** The software refers to "the chat card" and "the
  render card" by PCI address in a few places (watchdog) and by Vulkan device order in others;
  both must be checked per unit.
- [List of the exact fleet units, their CPU generation and slot layout, once known]
