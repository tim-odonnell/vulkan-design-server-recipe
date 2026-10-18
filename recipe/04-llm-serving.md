# 4. LLM serving

## Engine

- **llama.cpp** official Vulkan build **b11714** (`llama-server`), one process per loaded model.
- **llama-swap v265** in front, as the single OpenAI-compatible endpoint with API keys. It
  starts models on demand, keeps the `chat` group to one model at a time, and lets one loaded
  model be called under extra aliases with forced parameters (`setParamsByID`) or with client
  sampling stripped (`stripParams`).
- **Never put `#` comment lines inside a multi-line `cmd:` block** in llama-swap's YAML: they
  become command-line arguments.

## Normal-mode chat models (GPU 0, one at a time)

| Model | Quant / file size | Context | Slots | Measured | Sampling (makers' cards) |
|---|---|---|---|---|---|
| **Qwen3.6-35B-A3B** (default) | Unsloth UD-IQ3_S, 13.7 GB | **128K**, 16-bit KV, unified across 2 slots (`-np 2 -kvu`) | 2 | 51 tok/s alone, 35 tok/s each with 2 users; 27 tok/s with a full context | thinking: temp 1.0, top-p 0.95, top-k 20, min-p 0, presence 1.5 |
| Qwen3.8-27B (dense) | UD-IQ3_S, 12.0 GB | 64K, 16-bit KV | 2 | 12.9 tok/s, 8.5 with a full context | temp 1.0, top-p 0.95, top-k 20, min-p 0 |
| Gemma 4 12B QAT | UD-Q4_K_XL, 6.7 GB | 64K | 1 | 18 to 20 tok/s | temp 1.0, top-p 0.95, top-k 64 |

All have vision through a separate `mmproj` file. Common flags: `-dev Vulkan0 -ngl 999 -fa on
--jinja`, answers capped at 32K tokens (`-n 32768`) and thinking at 12K
(`--reasoning-budget 12288` plus a "time to answer" message). The caps were added after a
web-search answer ran away past 28K tokens.

**Background tasks** (search queries, chat titles, tags) use an alias of the same loaded 35B
with thinking switched off (`chat_template_kwargs: {enable_thinking: false}`, Qwen's
non-thinking sampling). Without it, a title took minutes of thinking.

### How 128K was chosen (35B, 16-bit KV)

The 35B's KV cache costs about 20 KB per token (only 10 of its 40 layers use full attention).

| Context | Result |
|---|---|
| 128K | about 1 GB spills to system RAM, 27 tok/s at full context, recall correct |
| 256K | 3.4 GB spills, 2.5 tok/s: unusable |

The 27B's KV costs about 64 KB per token, which is why the smaller-weight model gets the
smaller context.

### Quality comparison (same prompts, 2026-10)

The 35B solved a timing puzzle and gave usable brainstorms fastest; the 27B gave the best
answers (most concrete, most honest) at 3 to 7 times the wall time; Gemma 12B looped on the
puzzle and hit the token limit. The 35B is the default; the 27B is the "think harder" option.

## Coding-mode models (both GPUs, pipeline split)

Only in coding mode, through OpenCode. Both use the owner's rules: **16-bit KV cache only, no
spill into system RAM, MTP (multi-token prediction) on.**

| Model | File | Context | MTP draft depth | Measured |
|---|---|---|---|---|
| **Qwen3.8-27B** (default) | UD-Q4_K_M, 16.5 GB | 128K | 2 | ~12.4 tok/s; first OpenCode answer ~3 min (reads ~11.7K tokens of instructions at 76 tok/s) |
| Qwen3.6-35B-A3B | UD-Q4_K_M, 22.7 GB | 160K | 3 | 44 to 47 tok/s on short answers (24.6 without MTP) |

Flags: `-dev Vulkan0,Vulkan1 -sm layer --tensor-split 1,1 -b 2048 -ub 512 --spec-type
draft-mtp --spec-draft-p-min 0`, env `RADV_PERFTEST=nogttspill`, answers up to 64K tokens and
thinking up to 32K. OpenCode's own sampling values are stripped so the makers' values apply.

- **MTP trade-off:** about twice as fast at writing, about half as fast at reading a long
  prompt (35B: 101 vs 189 tok/s).
- **Why 160K and 128K:** at 224K the 35B's working memory on GPU 1 grew from 15.3 to 16.2 of
  16.4 GB and about 1 GB went through system RAM past ~174K tokens. The 27B had one GPU ring
  timeout at 208K with GPU 1 at 96%.
- [Full-context fill tests at 160K and 128K were not completed; run them to confirm]

## CPU model: Gemma 4 E2B

| File | Use | Settings |
|---|---|---|
| Gemma 4 E2B QAT UD-Q4_K_XL (2.6 GB) + MTP drafter + mmproj | HQ video director (vision + tool calls) | CPU only (`-dev none`, no Vulkan driver visible to the process), 6 threads, 16K context, 16-bit KV, MTP depth 3, Google's own chat template, thinking off per request. About 5 GB RSS, about 11 tok/s while another job shares the CPU [Measure alone] |

## Laptop terminal client

**aichat** on the laptop talks to `https://<server>.<tailnet>:8443` with its own API key.
Context limits set in its config: 35B 120K input, 27B and Gemma 60K. It has **no web search**.
[macOS and Windows setup scripts]
