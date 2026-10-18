# 12. Lessons and pitfalls

Each line: what happened → the rule it produced.

## GPUs and memory

- ROCm on gfx900 wedged both cards twice with two video jobs → **Vulkan only**, one heavy video
  job at a time.
- RADV spilled a 256K context into RAM and ran at 2.5 tok/s with no error → **size contexts by
  watching GTT**, use `nogttspill` for the coding models.
- An unpinned Vulkan loader ran a model on the Intel iGPU → **pin the AMD ICD everywhere**.
- The x4 chipset slot card had the ring timeouts → **watchdog**, and keep that card below ~96% VRAM.
- Big text encoders in RAM pushed 16 GB into swap → **encoders go on a GPU, read from disk**.
- stable-diffusion.cpp planned a two-card split before freed VRAM was reported → **patch:
  settle free VRAM before a new plan**; restart the HQ engine per job.

## Models and prompts

- A web-search answer thought past 28K tokens → **cap answers and thinking** at the server.
- Native tool calling let the model skip the search and invent sources → **force the search**
  (legacy function calling).
- Chat titles took minutes of thinking → **thinking-off alias** for background tasks.
- Letting the chat model choose an image tool dropped the attached sketch → **deterministic
  pipe** for images and video.
- The small vision model agreed with any prompt → **blind description first, then compare**.
- Examples in a review prompt leaked into the review ("walnut cap") → **no concrete examples
  in judge prompts**.
- A blanket edit raised every model's context, including one that could not fit → **change one
  model at a time and re-test it**.

## Operations

- Comment lines inside a llama-swap `cmd:` block became arguments → **comments outside `cmd`**.
- `pkill -f pattern` matched the shell running it → **select PIDs by exact process name**.
- Test scripts in `/tmp` vanished at reboot → **keep test scripts in the repo**.
- sd-server ignored cancel on a running LTX job → **remove the engine after 30 s**.
- A progress dialog that could not open killed the mode switch (SIGPIPE) → **ignore SIGPIPE in
  progress pipes**.
- Open WebUI kept generating after the client disconnected → restart the UI service to stop a
  runaway generation [find a cleaner way].
- An auto-fixing linter rewrote vendored and benchmark files → **lint only your own files**,
  never "fix" raw results.

## Process

- Measure before choosing: every number in this recipe came from a test on the real machine.
- Keep the owner in charge of trade-offs (context vs speed, access vs safety); write their
  decisions down with the reason.
