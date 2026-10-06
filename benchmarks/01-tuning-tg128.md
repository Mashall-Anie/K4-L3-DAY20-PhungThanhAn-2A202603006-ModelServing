# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 9.5 | 76% |
| 6 | 12.6 | 100% |
| 12 | 12.4 | 98% |
| 16 | 10.9 | 86% |
| 32 | 6.6 | 53% |

**Best**: `-t 6` at 12.6 tok/s
**Slowest tested**: `-t 32` at 6.6 tok/s (1.90x spread)
**Against the physical-core default** (`-t 12`, 12.4 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

The knee appears at about 6 threads, not at the 12 physical-core count. Throughput rises from 9.5 tok/s at 1 thread to 12.6 tok/s at 6 threads, then is already flat at 12 threads, where it reaches 12.4 tok/s. This indicates that decode reaches the useful memory-bandwidth and cache limit with roughly six workers. Additional threads compete for shared memory bandwidth and cache, while synchronization and scheduling overhead offset the benefit of more parallelism. Throughput falls to 10.9 tok/s at 16 threads and 6.6 tok/s at 32 threads because logical-thread contention and then oversubscription become increasingly expensive. Therefore, -t 6 is the best tested setting, although its practical speedup over the default -t 12 is modest at 1.02x.
