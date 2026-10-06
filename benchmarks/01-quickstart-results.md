# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2110 | 276 / 310 | 47.8 / 50.7 | 3282 / 3470 / 3470 | 20.9 |
| UD-Q2_K_XL | 2.24 | 2092 | 378 / 445 | 38.6 / 39.9 | 2812 / 2957 / 2957 | 25.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.24x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

Phiên bản 2-bit (UD-Q2_K_XL) nhỏ hơn khoảng 0.73 GB và có tốc độ giải mã (decode) nhanh hơn 1.24x (25.9 tok/s so với 20.9 tok/s của bản 4-bit). 
Mặc dù tốc độ sinh token của Q2 có sự cải thiện rõ rệt và giúp tiết kiệm RAM, nhưng điều này đồng nghĩa với việc đánh đổi bằng chất lượng câu trả lời do mất mát thông tin khi lượng tử hóa xuống mức 2-bit. 
Với máy tính hiện tại có 15.3GB RAM (rất thoải mái để chạy bản 4-bit), và tốc độ ~21 tok/s của bản 4-bit đã đủ mượt mà để đọc, việc dùng bản 4-bit (UD-Q4_K_XL) sẽ mang lại trải nghiệm tốt hơn nhiều nhờ chất lượng ngôn ngữ vượt trội, phù hợp cho các tác vụ cần độ chính xác cao. Bản 2-bit chỉ thực sự đáng giá nếu bị giới hạn RAM rất nghiêm ngặt.
