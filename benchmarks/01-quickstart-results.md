# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4081 | 720 / 837 | 97.1 / 114.8 | 6845 / 8069 / 8069 | 10.3 |
| UD-Q2_K_XL | 2.24 | 4062 | 758 / 901 | 79.2 / 82.3 | 5772 / 5949 / 5949 | 12.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.22x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation 

On the matched quality test, UD-Q4_K_XL produced a concise and technically correct answer: it defined TTFT as the time to the first token and TPOT as the average time per subsequent output token, with a practical example. UD-Q2_K_XL stopped normally but incorrectly described TPOT as the time spent processing the prompt and generating the entire response, despite being explicitly told not to confuse it with end-to-end latency. Therefore, Q2's 24.6% smaller size and 1.22x faster decode come with an observable quality regression on this test. For this workload I prefer Q4 when correctness matters; Q2 is attractive only when speed and memory usage are prioritized and outputs can be validated.
