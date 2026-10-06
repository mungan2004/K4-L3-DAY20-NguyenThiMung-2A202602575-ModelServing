# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 27 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.90 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3112 |

Highest sampled value was **3.90 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width đạt 3.90/4 slot, minh chứng rõ ràng cho việc continuous batching đang hoạt động (gần 100% tài nguyên decode đang được dùng). Con số này thấp hơn Effective concurrency ở file `02-server-results.md` (đạt 9.8). Sự chênh lệch này là bình thường vì Effective concurrency tính tổng số request đang "bay" (in-flight), tức là bao gồm cả 4 request đang chạy trên 4 slot decode cộng thêm lượng lớn các request đang nằm chờ (requests_deferred = 46). Do đó, để đánh giá mức độ sử dụng phần cứng, Peak batch width là đáng tin cậy hơn, còn để đánh giá độ "kẹt xe" tổng thể của server, ta dựa vào Effective concurrency.
