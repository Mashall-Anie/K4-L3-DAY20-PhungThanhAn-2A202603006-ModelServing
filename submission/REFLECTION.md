# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phung Thanh An
**MSSV:** 2A202603006
**Cohort:** A20 - K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Fedora Linux, kernel 7.2.8-200.fc44.x86_64
- **CPU:** 12th Gen Intel(R) Core(TM) i5-1240P
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 31.0 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-x64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL
- **Chạy ở đâu:** laptop cá nhân

**Chạy ở đâu:** laptop cá nhân
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Tôi chạy lab locally trên laptop Fedora với 31 GB RAM và không có GPU. Setup mặc định chạy thành công, tải prebuilt llama.cpp runtime và hai quantization Gemma 4 E2B. Không cần dùng cloud hoặc workaround đặc biệt. Runtime chạy CPU-only với `ngl=0`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4081 | 720 / 837 | 97.1 / 114.8 | 6845 / 8069 / 8069 | 10.3 |
| UD-Q2_K_XL | 2.24 | 4062 | 758 / 901 | 79.2 / 82.3 | 5772 / 5949 / 5949 | 12.6 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

UD-Q2_K_XL nhỏ hơn 0.73 GB, tương đương giảm khoảng 24.6%, và decode nhanh hơn 1.22 lần, từ 10.3 lên 12.6 tok/s. TPOT của Q2 giảm từ 97.1 xuống 79.2 ms, nhưng TTFT hơi xấu hơn. Trong quality test cùng prompt, Q4 trả lời đúng định nghĩa TTFT và TPOT, còn Q2 nhầm TPOT với thời gian xử lý toàn bộ prompt và response. Vì vậy Q2 có lợi thế rõ về tốc độ và bộ nhớ, nhưng tôi ưu tiên Q4 khi độ chính xác quan trọng; Q2 chỉ phù hợp khi có cơ chế kiểm tra output.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|---:|---:|---:|---:|---:|---:|---:|
| 10 | 0.44 | 19000 | 28000 | 32000 | 7.3 | 0.0% |
| 50 | 0.37 | 18000 | 54000 | 54000 | 8.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.84×, chỉ đạt 17% mức tuyến tính
- **P95 tăng:** 1.93×
- **Effective concurrency ở 50 users:** 8.2 so với `--parallel = 4` slots

- **Peak `n_busy_slots_per_decode`:** 3.91/4 slots, tương đương 98%

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ở mức 50 users hoặc sớm hơn. Khi offered load tăng 5 lần, throughput chỉ đạt 0.84 lần trong khi P95 tăng từ 28 lên 54 giây. Effective concurrency ở 50 users là 8.2, lớn hơn 4 decode slots; đồng thời metrics ghi nhận 3.91/4 busy slots và 46 request deferred. Điều này cho thấy request bổ sung chủ yếu phải xếp hàng. Tôi chọn SLO là P95 không quá 30 giây: run 10 users gần đạt SLO, còn run 50 users vượt SLO. Tôi sẽ giảm `max_tokens` và context RAG trước khi tăng `--parallel`, vì request dài giữ slot lâu và CPU đã có dấu hiệu giới hạn bởi memory bandwidth.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Cloud/IaC không được nối vào pipeline này | stub |
| N17 Data pipeline | Không dùng data-ingestion pipeline thật | stub |
| N18 Lakehouse | Không dùng lakehouse hoặc bảng lưu trữ thật | stub |
| N19 Vector + features | Keyword overlap trên `TOY_DOCS`, không có vector index thật | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2775.6 ms
- total: 2775.6 ms
- **stage chiếm nhiều nhất:** llm, 100% của total

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Pipeline dùng keyword overlap và bộ tài liệu đồ chơi nên embed và retrieve gần như không tốn thời gian. Stage LLM chiếm toàn bộ latency vì phải thực hiện prefill trên prompt đã ghép context và decode câu trả lời. Nếu muốn giảm latency khoảng 2 lần, tôi sẽ tối ưu stage LLM trước bằng cách giảm số context retrieve được và giới hạn output tokens. Q2 có tốc độ decode tốt hơn nhưng quality test cho thấy có lỗi định nghĩa TPOT, nên cần kiểm tra chất lượng kỹ trước khi dùng trong production.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm số thread decode từ `-t 12` xuống `-t 6`

```
before: 12.4 tok/s (-t 12)
after:  12.6 tok/s (-t 6)
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Trên máy của tôi, -t 6 là cấu hình tốt nhất trong các mức đã thử, dù CPU có 12 core vật lý và 16 core logic. Throughput tăng từ 12.4 lên 12.6 tok/s, tương đương speedup 1.02× so với cấu hình mặc định -t 12. Đường cong đạt knee khoảng 6 thread: từ 1 lên 6 thread, throughput tăng từ 9.5 lên 12.6 tok/s, nhưng từ 6 lên 12 thread thì gần như không tăng thêm.

Nguyên nhân là decode chủ yếu bị giới hạn bởi memory bandwidth và cache dùng chung, không chỉ bởi số lượng core. Khi khoảng 6 thread cùng hoạt động, hệ thống đã sử dụng gần hết băng thông bộ nhớ hữu ích cho việc đọc weights. Các thread bổ sung phải tranh chấp memory bandwidth và cache, đồng thời tạo thêm chi phí scheduling và synchronization. Vì vậy throughput giảm xuống 10.9 tok/s ở 16 thread và chỉ còn 6.6 tok/s ở 32 thread. Mức -t 32 còn gây oversubscription vì vượt quá 16 logical cores. Kết quả cho thấy giảm thread từ 12 xuống 6 chỉ đem lại speedup nhỏ, nhưng giúp tránh việc dùng thêm thread mà thực tế làm decode chậm hơn.
---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi đã sử dụng Arena.ai Agent Mode để hiểu các khái niệm TTFT, TPOT, continuous batching và Little's Law; đọc lỗi từ `make verify`; hỗ trợ phân tích các số liệu do chính máy của tôi sinh ra; và cải thiện cách trình bày báo cáo. Tôi tự chạy toàn bộ các lệnh lab, kiểm tra output, tạo screenshots và chịu trách nhiệm về số liệu cũng như kết luận trong submission.