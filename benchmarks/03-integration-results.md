# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 5716.1 | 5716.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 3512.9 | 3513.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 6621.7 | 6621.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **5283.6** · total **5283.7**
Dominant stage: **llm** (100% of total)

## Câu trả lời do mô hình tạo

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on the specific metrics that define a system's operational state and reliability.

According to the text:
*   **Goodput** counts only requests per second that met the **TTFT** (Throughput Target for Functionality) and **TPOT** (Throughput Target for Performance) targets.
*   **Raw throughput** ignores 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the Key-Value (KV) cache in non-contiguous pages.

By utilizing non-contiguous pages, the model avoids the wasted space that would otherwise exist if all KV cache entries were stored contiguously in a single contiguous block of memory. This optimization allows the engine to utilize more of the available

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the model's **prefill (inference) and decode (inference) phases are computationally and bandwidth-boundly incompatible**, or when the **prefill phase is the bottleneck** itself.

Here is the breakdown based on the provided context:

1.  **Prefill is Compute-bound**: Prefill involves heavy computation (often involving the entire model or a large subset of it)


## Thành phần N16–N19: thật hay mô phỏng?

N16–N19 được mô phỏng: pipeline dùng `TOY_DOCS` và truy xuất theo từ khóa, không có embedding server. N20 `llama-server` là thật. LLM chiếm 5,283.6 ms trung bình, còn truy xuất 0.1 ms. Để giảm độ trễ 2×, mình sẽ giảm token cần sinh hoặc tăng tốc giải mã; tối ưu truy xuất gần như không giúp vì thời gian của nó rất nhỏ.
