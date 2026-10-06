# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=12` `ngl=0` · RAM 15.3 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 124.6 | 2054.1 | 1.00x |
| 1024 | 111.0 | 9228.6 | 1.12x |
| 2048 | 97.0 | 21115.6 | 1.28x |
| 4096 | 86.1 | 47572.6 | 1.45x |
| 8192 | 78.2 | 104783.8 | 1.59x |

At 8192 tokens, prefill costs **104784 ms** --
1.59x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

Prefill bắt đầu "nuốt chửng" toàn bộ thời gian của hệ thống một cách rõ rệt từ mốc 2048 tokens trở đi (tốn ~21 giây, thời gian gấp 1.28x so với dự kiến tuyến tính). Đặc biệt ở mốc 8192 tokens, thời gian prefill đội lên kinh hoàng tới 104.7 giây (1.59x linear).
Đường cong quadratic O(N^2) của Attention hiện ra cực kỳ rõ ràng trên CPU khi kích thước Prompt vượt qua 1024. Điều này cho thấy khi thiết kế RAG pipeline trên CPU, ta phải cực kỳ tằn tiện với lượng context. Chỉ nên truyền 1 đến 2 chunk liên quan nhất (tổng khoảng 500-1000 tokens) để giữ TTFT ở mức < 10 giây. Việc nhồi nhét tối đa context window (8k) sẽ khiến hệ thống sập hoàn toàn vì người dùng phải chờ gần 2 phút mới thấy chữ đầu tiên.
