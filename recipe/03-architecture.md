# 3. Architecture: services, ports and data paths

Every service listens on **127.0.0.1 only**. The only way in from another device is Tailscale
("tailscale serve"), which publishes a few chosen ports on the server's private tailnet name.
Nothing is exposed to the LAN or the internet.

## Normal mode (the default, always after a reboot)

```text
 phone / laptop browser                      laptop terminal (aichat)
        │ https://<server>.<tailnet>                │ https://<server>.<tailnet>:8443 (API key)
        ▼                                           ▼
 ┌──────────────┐  OpenAI API   ┌──────────────────────────────┐
 │ Open WebUI   │──────────────▶│ llama-swap :8100 (API keys)  │──▶ llama-server ── GPU 0
 │ :3000        │               │ group "chat": one model      │    Qwen3.6-35B (default)
 │ (Docker)     │               │ loaded at a time             │    or Qwen3.8-27B or Gemma 12B
 └──┬───────┬───┘               └──────────────────────────────┘
    │       │ web search                        ┌─────────────────────┐
    │       └──────────────────────────────────▶│ SearXNG :8890       │──▶ public search engines
    │ "Image studio" and "Video" pipes          └─────────────────────┘
    ▼
 ┌──────────────────────────────┐   native async API   ┌──────────────────────────────┐
 │ Render broker :8200 (FastAPI)│─────────────────────▶│ sd-server :8210 (Docker)     │── GPU 1
 │ one job at a time, saved     │  starts/stops the    │ one engine at a time:        │
 │ queue, MP4 conversion,       │  right engine        │ FLUX.2 klein / Qwen-Image /  │
 │ status page + files          │                      │ LTX-2.3 Q4_0                 │
 └──┬───────────────────────────┘                      └──────────────────────────────┘
    │ "Image ready" / "Video ready"
    ▼
 ┌──────────────┐  http://<server>.<tailnet>:8090   ┌───────────────┐
 │ ntfy :8300   │──────────────────────────────────▶│ ntfy phone app│
 └──────────────┘                                    └───────────────┘
```

## Ports

| Port (localhost) | Service | Published on the tailnet as | Auth |
|---|---|---|---|
| 3000 | Open WebUI | `https://<server>.<tailnet>` (443) | Open WebUI accounts |
| 8100 | llama-swap, chat models | `https://<server>.<tailnet>:8443` | One API key per device |
| 8200 | Render broker (status page, files, job API) | `https://<server>.<tailnet>:10000` | API key for the job API; **status page and files have no login yet** [must be fixed before giving a unit away] |
| 8300 | ntfy (notifications) | `http://<server>.<tailnet>:8090` (plain HTTP, still encrypted by Tailscale; all HTTPS ports were taken) | Topic name |
| 8890 | SearXNG | not published | n/a |
| 8210 | sd-server (inside Docker) | not published | n/a |
| 8101 | llama-swap, coding models (coding mode only) | not published | n/a |
| 8099 | Director model, CPU (HQ video mode only) | not published | n/a |
| 8230 | HQ Video Director page (HQ video mode only) | not published (used on the server's desktop) | n/a |

## How the parts talk

- **Open WebUI → LLM:** OpenAI-compatible chat completions to llama-swap with an API key.
  llama-swap starts the requested model's llama-server process on demand and keeps one "chat"
  model loaded on GPU 0.
- **Open WebUI → images and video:** two small Open WebUI **pipe functions** (they appear as
  models in the model picker: "Image studio" and "Video"). A pipe reads the user's message and
  any attached sketch, calls the broker's job API, streams progress lines back to the chat, and
  shows the finished picture or an MP4 link. A deterministic pipe was needed because letting the
  chat model pick an image tool dropped the attached sketch.
- **Broker → engines:** for each job it makes sure the right `sd-server` container is running
  (starting it with that engine's model files and flags if not), submits the job to sd-server's
  native async API, polls it, saves the result, converts video to H.264 + AAC MP4 (plays on
  Android and iPhone), and sends a notification. The queue is saved to disk; text-only jobs
  resume after a restart.
- **Notifications:** the broker posts to its own ntfy server; the phone app subscribes to the
  topic over Tailscale. Tapping opens the file. The GPU watchdog posts there too.
- **Search:** Open WebUI queries SearXNG's JSON API and puts the fetched pages straight into the
  model's context (no embedding step; the 128K context is large enough).

## Which GPU does what, per mode

| Mode | GPU 0 (03:00.0) | GPU 1 (07:00.0) | CPU | Open WebUI |
|---|---|---|---|---|
| **Normal** | Chat model (35B IQ3_S, 128K) | Image or video engine, one job at a time | Idle helpers | On |
| **Coding** | Half of a Q4_K_M coding model | Other half (pipeline split) | OpenCode in a terminal | **Off** |
| **HQ video** | Part of LTX-2.3 Q8 | Rest of LTX Q8 + its text encoder (only while encoding) | Gemma 4 E2B director | **Off** |

See [Modes](06-modes.md) for how the switch works.

## Data on disk

| Path (generic) | What |
|---|---|
| `~/models/...` | GGUF and safetensors model files, about 140 GB |
| `~/VDS-Output/images/<date>/`, `videos/<date>/` | Every render, plus the saved job queue |
| `~/HQ-Video/<project>/{media,pages,notes}` | HQ video projects |
| `~/Projects/` | Where OpenCode starts |
| `~/.config/vds/` | Secrets (API keys, mode 600), broker settings |
| `<repo>/server/...` | Every service's config, unit file and script [Publish a sanitised copy of the service code or not: owner's decision] |
