# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 9.7 | 48% |
| 6 | 20.1 | 100% |
| 12 | 19.7 | 98% |
| 16 | 18.9 | 94% |
| 32 | 13.0 | 65% |

**Best**: `-t 6` at 20.1 tok/s
**Slowest tested**: `-t 1` at 9.7 tok/s (2.06x spread)
**Against the physical-core default** (`-t 12`, 19.7 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

Đỉnh hiệu năng (knee) xuất hiện ở `-t 6` (đạt 20.1 tok/s), dù máy có đến 12 nhân vật lý (Intel i5-12500H gồm 4 nhân P-core và 8 nhân E-core). 
Lý do là quá trình inference (đặc biệt là phase decode) bị giới hạn bởi băng thông bộ nhớ (memory bandwidth) chứ không phải tốc độ tính toán (FLOPs). Với 6 thread, CPU có thể đã chạm mức bão hòa băng thông RAM. Việc tăng thêm số lượng thread (lên 12, 16 hay 32) chỉ làm các thread phải tranh giành băng thông bộ nhớ, gây ra độ trễ chuyển đổi ngữ cảnh (context switching overhead) và thrashing cache. Đặc biệt khi sử dụng quá nhiều E-cores yếu hơn hoặc sử dụng các thread ảo (Hyper-threading), hiệu suất bị tụt giảm rõ rệt (xuống 13.0 tok/s ở `-t 32`). Do đó, 6 thread mang lại sự cân bằng tốt nhất giữa sức mạnh xử lý song song và giới hạn vật lý của băng thông.
