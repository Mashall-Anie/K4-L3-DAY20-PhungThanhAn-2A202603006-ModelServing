# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 24 | 0.44 | 19000 | 28000 | 32000 | 7.3 | 0.0% |
| 50 | 20 | 0.37 | 18000 | 54000 | 54000 | 8.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.84x** (17% of linear) |
| P95 latency | **1.93x** |
| Effective concurrency at 50 users | 8.2 vs `--parallel 4` slots (occupancy/slot ratio 2.05) |

**Saturated.** Throughput delivered only 0.84x for 5x the offered load, and effective concurrency (8.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.84x while P95 moved 1.93x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is saturated at the 50-user load, or somewhere below it. The strongest evidence is that the offered load increased 5x, but delivered throughput changed only 0.84x, reaching just 17% of linear scaling. At the same time, P95 latency increased from 28 seconds to 54 seconds, or 1.93x. The server metrics support this conclusion: `n_busy_slots_per_decode` reached 3.91 of 4 slots and `requests_deferred` reached 46. Therefore, the extra load was mainly converted into queue time rather than additional throughput.

I choose a P95 SLO of 30 seconds. The 10-user run was close to this SLO, with P95 latency of 28 seconds and throughput of 0.44 requests/s. The 50-user run exceeded the SLO with P95 latency of 54 seconds, so its 0.37 requests/s should not all be counted as goodput@SLO. To improve goodput, I would first reduce `max_tokens` and the retrieved RAG context instead of immediately increasing `--parallel`, because long requests hold decode slots for longer. The thread sweep also suggests that decode is already close to the CPU's useful memory-bandwidth limit, so adding more slots could increase contention and KV-cache pressure. I would test another `--parallel` value only after reducing request cost.
