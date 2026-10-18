# 9. Reliability

The owner's priority order: **usability, durability and reliability over speed.**

## Boot

- All normal-mode services are systemd user services with linger: up 13 to 16 s after boot,
  with nobody logged in. A reboot always lands in normal mode.
- A check script verifies every service, endpoint, the chat model's answer and context size,
  GPU power caps and the four tailnet addresses, and prints PASS/FAIL lines.
- [BIOS "power on after AC loss" setting, so a power cut does not leave the server off]

## GPU watchdog

The amdgpu driver occasionally hits a **compute ring timeout** on a WX 9100 (seen on the x4-slot
card with Wan video and once with the 27B coding model). The driver resets and recovers the card,
but the program that was using it has lost its GPU and fails until restarted.

A small service follows the kernel log (`journalctl -k -f`) for ring timeouts, "device wedged"
and "GPU reset" lines, waits out the burst (one action per 60 s), and restarts what used that card:

| Mode | Card | Action |
|---|---|---|
| Coding | either | restart the coding model service |
| HQ video | either | remove the render engine (the next render reloads it) |
| Normal | chat card | restart the chat service |
| Normal | render card | remove the render engine |

It notifies the desktop and the phone. It has a dry-run mode for testing with a sample log line.
[It has not yet seen a real fault in service]

## Rules learned the hard way

- **One heavy video job at a time.** Two heavy jobs at once wedged both cards on ROCm.
- **16 GB RAM is the hard ceiling.** Anything that puts a big text encoder in RAM (Qwen-Image-Edit,
  LTX on the CPU) pushes the machine into swap. Every placement in this recipe was chosen to
  keep at least ~6 GB free.
- **Watch GTT, not just VRAM.** RADV spills silently; a "working" model can be 10x slower.
- **Pin the Vulkan driver** in every process.
- **Cancelling a running LTX job** needs the engine removed (sd-server ignores the cancel).

## Gaps

- [Alerts when chat or search stop answering (not only GPU faults)]
- [Backups of configs, units and the job queue]
- [Disk-space check: keep 100 GB free]
- [Burn-in test procedure for each fleet unit: hours, workloads, pass criteria]
