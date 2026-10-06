# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyen Thi Mung
**MSSV:** 2A202602575
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 6.8.0-136-generic (x86_64)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-12500H
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.3 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** prebuilt release b10488
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** gemma-4-E2B-it-UD-Q4_K_XL.gguf + gemma-4-E2B-it-UD-Q2_K_XL.gguf

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Quá trình setup diễn ra suôn sẻ nhờ tài nguyên RAM trên máy đáp ứng đủ (15.3GB). Để tránh xung đột với các package toàn cục, tôi đã sử dụng virtual environment `.venv` đã có sẵn của thư mục gốc.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3157 | 274 / 315 | 50.4 / 53.8 | 3199 / 3675 / 3675 | 19.9 |
| UD-Q2_K_XL | 2.24 | 2101 | 374 / 441 | 42.1 / 43.6 | 2815 / 3147 / 3147 | 23.7 |

**Quan sát** (≤ 60 chữ): Bản 2-bit nhanh hơn 1.19x và nhẹ hơn 0.73GB. Tuy nhiên, nó đánh đổi lại chất lượng câu trả lời. Vì cấu hình máy (RAM > 15GB) hoàn toàn thừa sức chịu tải bản 4-bit với tốc độ mượt mà (~20 tok/s), việc dùng bản 4-bit sẽ tốt hơn và đáng giá hơn hẳn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 31 | 0.53 | 15000 | 26000 | 31000 | 8.0 | 0.0% |
| 50 | 23 | 0.41 | 18000 | 55000 | 56000 | 9.8 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.78×
- **P95 tăng:** 2.12×
- **Effective concurrency ở 50 users:** 9.8 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.90 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ngay mức 10 users, bằng chứng là Effective concurrency đạt 8.0 (vượt giới hạn 4 slots). Khi load tăng lên 50 users, P95 tăng 2.12 lần trong khi throughput tụt xuống 0.78x (bởi phần lớn là queue time chứ không phải compute time). Để nâng goodput@SLO, ưu tiên vặn knob `--parallel` lên cao hơn để xử lý được nhiều slots đồng thời (miễn là RAM chịu nổi context), giảm tối đa việc xếp hàng.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 | Embeddings | stub |
| N17 | Vector Store / Retrieval | stub |
| N18 | Lakehouse | stub |
| N19 | Vector + features | stub |
| N20 | Serving (`llama-server`) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 3216.8 ms
- **stage chiếm nhiều nhất:** llm (100% of total)

**Reflection** (≤ 60 chữ): Bottleneck tuyệt đối nằm ở khâu LLM (100% thời gian do các khâu khác đã bị stub). Khớp hoàn toàn với kỳ vọng vì inference LLM tiêu thụ rất nhiều tài nguyên. Để giảm latency pipeline 2×, phải đánh thẳng vào tác vụ LLM (dùng model lượng tử hóa thấp hơn, nâng cấp phần cứng, hoặc dùng prompt caching).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Điều chỉnh số lượng luồng `-t` (thread count) từ 12 xuống 6

```
before:  19.7 tok/s
after:   20.1 tok/s
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đỉnh hiệu năng (knee) xuất hiện ở `-t 6`, dù máy có đến 12 nhân vật lý (Intel i5-12500H). Lý do là quá trình inference, đặc biệt là giai đoạn sinh token (decode phase), hoàn toàn bị giới hạn bởi tốc độ truyền tải dữ liệu của bộ nhớ (memory bandwidth bound) chứ không phải tốc độ xử lý toán học (FLOPs/compute bound) của chip.

Khi dùng tới 6 nhân, CPU đã hút cạn sạch băng thông RAM. Việc nhồi nhét thêm thread (như 12 hay cao hơn) không giúp đẩy nhanh tốc độ tính toán vì không còn băng thông để kéo dữ liệu từ RAM lên bộ đệm, trái lại còn làm tăng overhead của hệ điều hành do phải liên tục chuyển đổi ngữ cảnh (context switching) và chia sẻ băng thông. Càng nhiều thread thì hiệu suất càng giảm đi trông thấy do hiện tượng Cache Thrashing.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Chạy `make sweep-ctx` để kiểm chứng chi phí Context-length prefill khi kích thước Prompt lớn dần.

**Numbers:**

```
before:  2054 ms (prefill ở 256 tokens)
after:   104783 ms (prefill ở 8192 tokens)
speedup: 0.02x (chậm đi 51 lần khi context tăng 32 lần)
```

**Điều này nói lên gì mà deck chưa nói:**

Thực nghiệm này phơi bày rõ ràng sức tàn phá của độ phức tạp O(N^2) trong Attention đối với CPU. Ở dải context ngắn (dưới 1024), prefill tăng khá tuyến tính (1.0 - 1.12x) vì lúc này các phép tính Linear (O(N)) chiếm ưu thế. Nhưng khi bứt qua 2048 tokens, độ cong quadratic bắt đầu trừng phạt hệ thống: tại 8192 tokens, chi phí prefill thực tế vọt lên gấp 1.59 lần so với mức tăng tuyến tính lý tưởng, ngốn đến 104 giây chỉ để load prompt.
Điều này khẳng định rằng trong thiết kế RAG, chất lượng của bộ Retrieve quan trọng hơn dung lượng Context Window. Việc nhồi nhét càng nhiều chunk vào prompt là "tự sát" về mặt TTFT (Time To First Token).

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Điều thú vị nhất là nhận ra đôi khi số lượng luồng (threads) càng lớn không đồng nghĩa với tốc độ càng nhanh, đặc biệt là khi bài toán bị bottleneck bởi memory bandwidth.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Có sử dụng Gemini để thực hiện tự động các lệnh lab 
