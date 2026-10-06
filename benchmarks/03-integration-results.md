# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 10090.9 | 10091.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5689.3 | 5689.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 6228.4 | 6228.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7336.2** · total **7336.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on the actual request rate that meets specific targets (TTFT and TPOT) rather than ignoring SLOs.

The text explicitly states that Goodput counts only requests per second that met the targets, whereas throughput at saturation ignores SLOs. This distinction means Goodput provides a more accurate and rea

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs (KV cache) in non-contiguous pages.

By using non-contiguous pages, the model avoids wasting most of the available GPU memory that would otherwise be consumed by the internal fragmentation of contiguous memory blocks.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context states that prefill is compute-bound (requires significant processing power) and decode is memory-bound (requires significant bandwidth). By splitting these operations, the system can distribute the compute work across different resources or phases, ensuring that the memory


## Which N16-N19 pieces are real

- N16 Cloud/IaC: **stub** - pipeline chạy local, không provision cloud resource.
- N17 Data pipeline: **stub** - dùng danh sách `TOY_DOCS` có sẵn trong source.
- N18 Lakehouse: **stub** - không có storage/lakehouse thật trong lần chạy này.
- N19 Vector + features: **stub** - không có embedding server; retrieval dùng keyword overlap.
- N20 Serving: **real** - cả ba query gọi endpoint của `llama-server` local.

LLM là stage chiếm ưu thế đúng như kỳ vọng: trung bình 7336.2 ms, gần 100% tổng
latency, trong khi retrieval chỉ 0.1 ms. Muốn giảm tổng latency 2x, tôi sẽ tối ưu LLM
trước bằng cấu hình 5 thread đã tìm được, rút ngắn output/prompt khi phù hợp và thử
GPU offload ổn định; tối ưu retrieval gần như không thay đổi tổng thời gian ở cấu hình
stub hiện tại.
