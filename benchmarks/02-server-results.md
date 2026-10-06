# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 31 | 0.53 | 15000 | 26000 | 31000 | 8.0 | 0.0% |
| 50 | 23 | 0.41 | 18000 | 55000 | 56000 | 9.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.78x** (16% of linear) |
| P95 latency | **2.12x** |
| Effective concurrency at 50 users | 9.8 vs `--parallel 4` slots (occupancy/slot ratio 2.46) |

**Saturated.** Throughput delivered only 0.78x for 5x the offered load, and effective concurrency (9.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.78x while P95 moved 2.12x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Máy chủ đã bị bão hòa ở mức dưới hoặc ngay quanh mốc 10 người dùng. Bằng chứng rõ nhất là ngay tại tải 10 user, giá trị **Effective concurrency** đã đạt 8.0 (cao gấp đôi so với 4 slot song song mặc định của `--parallel 4`). Khi đẩy tải lên 50 user (gấp 5 lần), thông lượng (throughput) chẳng những không tăng mà còn tụt giảm (còn 0.78x), trong khi đó độ trễ P95 bị đội lên tới 2.12 lần. Con số này chứng tỏ mọi yêu cầu thêm vào sau khi bão hoà chỉ khiến hàng đợi (queue time) dài ra chứ không được tính toán ngay.
Để cải thiện goodput tại một ngưỡng SLO nhất định (ví dụ P95 < 20s), nút điều chỉnh nên được ưu tiên vặn đầu tiên là tăng số lượng slot xử lý đồng thời (`--parallel`), miễn là hệ thống vẫn còn đủ RAM để chứa context của các slot mới. Nếu phần cứng hạn chế, việc hạ lượng tử hóa (xuống 2-bit) cũng là một giải pháp nhằm giảm thời gian decode, đẩy token ra nhanh hơn để giải phóng slot sớm cho các request khác.
