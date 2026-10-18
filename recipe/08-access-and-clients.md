# 8. Access and clients

## Private network

- **Tailscale** on the server, the phone and the laptop, on the owner's own tailnet. The
  server publishes four ports with `tailscale serve` (see [Architecture](03-architecture.md)):
  chat UI (443), LLM API (8443), status and files (10000), notifications (8090, HTTP).
- Works the same at home and away. [Test from outside the home: phone on mobile data, laptop on
  a hotspot]
- No port forwarding, no public exposure, no cloud AI.

## Phone (Android verified; iPhone not yet)

- Browser → Open WebUI (installable as a home-screen app). Model picker shows the chat models
  plus "Image studio" and "Video".
- **ntfy** app subscribed to the server's own ntfy over Tailscale, topic `vds-renders`, with
  instant delivery. Notifications: Image ready (with thumbnail), Video ready, failed jobs
  (high priority), "GPU problem fixed automatically".
- iPhone: ntfy can only wake reliably through Apple push, which needs the server to forward
  to ntfy's public upstream (`upstream-base-url`); left off for privacy. [Decide and test]

## Laptop

- Browser → Open WebUI, same as the phone.
- **aichat** in a terminal (one binary on Linux, macOS and Windows) with the server as an
  OpenAI-compatible provider and its own API key. One setup script writes its config and tests
  the connection. [macOS and Windows scripts]

## Server desktop (remote)

- **Sunshine** on the server, **Moonlight** on the laptop, used for: first-day setup, the
  coding mode, the HQ video mode, and rescue.
- The desktop shows four icons: Launch OpenCode, Stop OpenCode, HQ Video Mode Start, HQ Video
  Mode Stop. [Test double-clicking each over Moonlight]

## Keys

- One API key per device (phone UI, laptop terminal, the UI itself), kept in one secrets file
  (mode 600). Revoking a device = deleting its key.
- [Login for the status/files page before any unit leaves the house]
