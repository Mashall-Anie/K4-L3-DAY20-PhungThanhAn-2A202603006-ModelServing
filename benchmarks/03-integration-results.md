# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3304.1 | 3304.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2594.8 | 2594.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2428.0 | 2428.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2775.6** · total **2775.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

N16 Cloud/IaC: stub. This run did not connect to a real cloud or infrastructure deployment.

N17 Data pipeline: stub. The pipeline used the built-in toy documents rather than a real data-ingestion pipeline.

N18 Lakehouse: stub. No real lakehouse or persistent table was used in this run.

N19 Vector + features: stub. Retrieval used keyword overlap over `TOY_DOCS`; there was no real embedding model or vector index.

The dominant stage was the LLM call, which accounted for 100% of the measured pipeline latency, with a mean of 2775.6 ms. This was expected because embedding and retrieval used lightweight local fallbacks and were rounded to 0.0 ms, while the LLM call included prompt prefill and token decoding. If I had to halve the pipeline latency, I would attack the LLM stage first by reducing the retrieved context and output-token budget. The retrieved prompts contained 113–149 tokens, and the server spent roughly 1.26–1.63 seconds on prefill plus 1.16–1.64 seconds on decoding. Using a faster quantization is another option, but the earlier quality test showed an accuracy regression for Q2, so I would validate quality before adopting it.
