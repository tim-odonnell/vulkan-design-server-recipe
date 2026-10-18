# 11. Benchmarks

All on the reference unit (i5-11400T, 16 GB DDR4, 2× WX 9100 at 100 W, Mesa RADV, October 2026).
"tok/s" is generation speed unless marked "read".

## Chat (GPU 0)

| Model | Setting | Result |
|---|---|---|
| Qwen3.6-35B-A3B UD-IQ3_S | 1 user, short answer | 51 tok/s |
| | 2 users at once | 35 tok/s each |
| | 128K context full | 27 tok/s, recall correct, ~1 GB in system RAM |
| | 256K context | 2.5 tok/s, 3.4 GB spill (rejected) |
| | during an LTX Q4_0 render on GPU 1 | 49 tok/s |
| Qwen3.8-27B UD-IQ3_S | short / full 64K | 12.9 / 8.5 tok/s |
| | 2 users | 8.3 tok/s each |
| Gemma 4 12B QAT | short | 18 to 20 tok/s (15 each for 2 users) |
| Swap 35B ↔ 27B | | ~35 s |

## Coding (both GPUs, 16-bit KV, MTP)

| Model | Result |
|---|---|
| Qwen3.8-27B UD-Q4_K_M, 128K, MTP 2 | 12.4 tok/s; reads 76 tok/s; mode switch + load 72 s |
| Qwen3.6-35B-A3B UD-Q4_K_M, 160K, MTP 3 | 44 to 47 tok/s short (24.6 without MTP); reads 101 tok/s with MTP, 189 without |
| MTP depth sweeps | 35B: depth 3 = 46.4 vs depth 2 = 42.3 tok/s. 27B: depth 2 best of 1 to 5 |

## CPU

| Model | Result |
|---|---|
| Gemma 4 E2B QAT + MTP 3, 6 threads | ~27 tok/s alone (earlier dispatcher test), ~11 tok/s sharing the CPU, ~4 to 5 GB RAM |

## Images (GPU 1)

| Job | Result |
|---|---|
| FLUX.2 klein sketch → render, 1168×880, 4 steps | 166 s, 8.3 GB VRAM |
| Qwen-Image Turbo text → image, 640×352 warm | ~32 s |
| Qwen-Image Turbo, 1536×1024 | ~274 s |
| Qwen-Image-Edit-2511 (retired) | 563 s, RAM down to 3.4 GB free |

## Video

| Job | Result |
|---|---|
| LTX-2.3 Q4_0, GPU 1, 2 s (49 frames), 30 steps | 1098 s; GPU 1 peak 13.4 GB; RAM ≥ 8 GB free |
| LTX-2.3 Q8_0, both GPUs (HQ mode), 2 s, 30 steps | 20 min; encoder 42 s on GPU 1, Q8 load 64 s; VRAM 14.6 / 11.8 GB; RAM ≥ 7.1 GB free; 0 GPU faults |
| LTX Q8 step time | ~30 s/step at 25 frames, ~33 at 49, 69 at 121 (earlier test) |
| Earlier LTX Q8 with the encoder on the CPU (rejected) | 33 frames, 20 steps: 14.8 min; RAM down to 447 MB free |
| Wan 2.1 1.3B (retired) | 2 s ≈ 400 s at 512×288 |

## Still to measure

- [Wall power per mode, idle power, yearly energy]
- [LTX Q8 at 3, 4 and 5 seconds]
- [Q4_0 vs Q8_0 quality on the same prompt and seed]
- [Coding contexts filled to 128K and 160K]
- [3 to 8 concurrent chat users]
- [E2B director speed alone]
- [Noise levels]
