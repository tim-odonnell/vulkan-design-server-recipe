# 6. Modes

The owner picks the mode; the machine never switches by the clock. Two big-model modes are
started from **desktop icons** on the server (reached over Moonlight), because they turn off
the shared front door.

| | Normal | Coding | HQ video |
|---|---|---|---|
| Started by | Boot (always) | "Launch OpenCode" icon | "HQ Video Mode Start" icon |
| Ended by | n/a | "Stop OpenCode" icon, or closing OpenCode | "HQ Video Mode Stop" icon |
| GPUs | 0: chat, 1: renders | both: one coding model | both: LTX-2.3 Q8 |
| CPU | | OpenCode | Gemma 4 E2B director |
| Open WebUI, phone and laptop chat | On | **Off** | **Off** |
| Renders from the phone | On | Refused (503) | Refused (503) |
| Switch time | | ~72 s until the 27B is ready | ~17 s (warm) [cold-boot figure] |
| Back to normal | | ~1½ min | ~1½ min |

## How a switch works

One script with a lock (`flock`) owns all switching: `mode coding | hq-video | normal | status`.

1. **Ask.** A desktop dialog says what will switch off. If renders are unfinished, a second
   dialog lists them and cancels them only on "Yes".
2. **Flag.** A flag file in `/run/user/<uid>` tells the broker to refuse the wrong kind of
   work, and tells the GPU watchdog what to restart after a fault.
3. **Free the GPUs:** remove the render engine container, stop Open WebUI and the chat service.
4. **Start** the mode's own services (never enabled at boot) and wait for their health checks
   behind a progress window.
5. **Open** the tool: OpenCode full screen in a terminal, or the director page in the browser.

Leaving any mode runs the same "normal" path: stop the mode's services, cancel leftover HQ jobs
(they need GPU 0, which chat is about to take back), remove the flag, start chat and Open WebUI.
The progress pipes ignore SIGPIPE so the switch completes even if the dialog cannot open.
Coding and HQ video mode refuse to start while the other is on.

## Coding mode

- **OpenCode** (pinned version) with its own config folder, separate from any other OpenCode
  setup on the machine: exactly two models (27B default, 35B faster), the server's coding
  endpoint only, auto-update and sharing off.
- **Permissions:** the owner chose "whole computer, ask each time": every edit, shell command,
  web fetch and file outside `~/Projects` needs approval.
- OpenCode reserves its output limit from the context, so the output limit is set to 32K.

## HQ video mode and the HQ Video Director

A small web page (FastAPI + one HTML file) on the server's desktop, backed by Gemma 4 E2B on
the CPU. The director:

- turns an idea into a detailed LTX prompt (one shot, 60 to 150 words: subject, motion,
  setting, light, one camera move, mood, sound) and puts it on a **Render card**;
- renders only when the user presses Render (they can edit the prompt and length first; a
  switch allows direct rendering);
- after each render, copies the MP4 into the project, makes a 2×2 frame sheet and reviews it
  in **two steps**: first describes the frames *without* seeing the prompt, then compares that
  description with the prompt and scores it. A one-step review just agreed with the prompt
  (it called a pitcher a "pour-over kettle", 9/10); the blind step called it a pitcher (7/10);
- builds HTML pages and scroll-snap slide presentations of chosen renders;
- works only inside `~/HQ-Video/<project>/` and writes only text files; it has no shell and
  no internet.

Limits: a 2B model cannot judge camera movement or sound, and its suggested fixes are plain.
The user's eyes are the final judge.
