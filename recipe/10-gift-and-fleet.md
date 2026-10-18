# 10. Gift and fleet

The build is meant to be **cloned to many units** and given away, each with a laptop, to people
who may be new to Linux. The builder has **no remote access after delivery**.

## Constraints

- No monitor, keyboard or mouse ships with a unit. Recipients have **no Ethernet at home**.
- Each recipient sets up their own Wi-Fi, Tailscale account, passwords and keys.
- Nothing personal from the builder may remain on a unit.
- Plain-language guides must cover everything the recipient does (written as part of the build,
  in a `docs/user-guide/` folder).

## Build from a script, not by scrubbing

Scrubbing a used drive always misses something (a token, shell history, a cache). Instead:
Ubuntu autoinstall + an install script/playbook on a fresh disk, testing done on a dedicated
non-personal account. Per-unit steps the script must cover (collected while building):

- llama.cpp build, llama-swap, model downloads with checksums, both chat and coding models
- Docker images: stable-diffusion.cpp (normal and patched HQ tag), SearXNG, ntfy, Open WebUI
- systemd user units; enable the normal ones, install but **never enable** the coding and HQ ones
- linger, power-cap service, Vulkan driver pin, Sunshine, Tailscale serve mapping
- desktop icons (marked trusted), `~/Projects`, `~/HQ-Video`
- per-unit values: tailnet name, API keys, ntfy address, **PCI addresses and Vulkan order**
- [The install script itself: not written yet]

## First day for the recipient (direct-cable plan)

1. Plug the laptop into the server with an Ethernet cable (USB adapter if needed); both get a
   link-local or fixed address.
2. Open Moonlight on the laptop, pair with the PIN, see the server's desktop.
3. On the server's desktop: join home Wi-Fi, sign in to Tailscale with the recipient's account.
4. Sign in to Tailscale on the laptop and phone; unplug the cable. From now on everything goes
   over Wi-Fi + Tailscale.
5. Run the per-unit "rename" step: the new tailnet name goes into Open WebUI, the broker, ntfy
   and the laptop config. Create the recipient's Open WebUI admin account and new API keys.
6. Phone: Open WebUI in the browser, ntfy subscription.

The same cable is the **rescue path** whenever Wi-Fi or Tailscale breaks.
[Test the whole sequence on a unit with no monitor; decide dummy display plug]
[Fixed vs link-local addressing on the cable; what to do if the laptop's OS changes it]

## Laptop

Dual boot, Ubuntu by default and Windows 11 as a personal backup. Ships with Moonlight,
Tailscale, aichat and the guides. [Laptop model and image]

## Learning layer

A second goal is teaching Linux to beginners: guides are GUI-first and introduce at most one
terminal command at a time. [Learning path content]

## Open questions

- [Licences: can each model be redistributed on a gifted drive? Can renders be used
  commercially?]
- [Success criteria: how the builder learns whether the gift worked]
- [Owner's manual: what is in the box, first day, what to do if it breaks]
