# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3825.1 | 3825.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2955.1 | 2955.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 2870.3 | 2870.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3216.8** · total **3216.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 (Embeddings)**: Stub (trả về thời gian 0.0ms).
- **N17 (Vector Store / Retrieval)**: Stub (sử dụng keyword overlap, thời gian 0.0ms).
- **N18 (LLM)**: Real (quá trình suy luận thực chạy qua llama-server, tốn >2800ms).
- **N19 (Serving API)**: Real (sử dụng OpenAI-compatible API trên cổng 8080).

Thời gian bị chiếm dụng 100% (dominant) nằm ở khâu **LLM**, điều này hoàn toàn trùng khớp với thực tế do LLM là tác vụ nặng nhất, đặc biệt là ở bước decode từng token (bị giới hạn băng thông bộ nhớ). Các bước embed và retrieve chỉ được dùng làm stub nên gần như không tốn thời gian.
Nếu buộc phải giảm một nửa latency cho toàn bộ pipeline này, mình sẽ tập trung "tấn công" trực tiếp vào LLM (bởi tối ưu embed hay retrieve lúc này là vô nghĩa khi tổng thời gian của chúng xấp xỉ bằng không). Cách làm cụ thể có thể là đổi xuống dùng model lượng tử hóa thấp hơn (ví dụ 2-bit thay vì 4-bit) để tăng tốc độ decode, tận dụng prompt caching để giảm TTFT, hoặc trang bị thêm accelerator (GPU).
