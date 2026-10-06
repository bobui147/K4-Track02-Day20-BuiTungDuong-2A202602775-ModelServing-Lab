# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Bùi Tùng Dương
**MSSV:** 2A202602775
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-1235U
- **Cores:** 10 physical / 12 logical
- **CPU extensions:** không được probe báo cáo
- **RAM:** 7.7 GB
- **Accelerator:** Vulkan được phát hiện; benchmark dùng CPU (`ngl=0`)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Windows PowerShell 5.1 đọc sai UTF-8 của `lab.ps1`, nên tôi đổi ba dấu gạch dài sang
ASCII và ép Python dùng UTF-8. Probe RAM ban đầu cũng sai khi WMI bị từ chối; tôi dùng
Windows `GlobalMemoryStatusEx` để lấy đúng 7.7 GB. Vulkan bị treo khi đổi sang Q2, nên
mọi phép đo chính thức chạy nhất quán trên CPU với `ngl=0`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7332 | 2153 / 2608 | 30.1 / 44.6 | 3987 / 4774 / 4774 | 33.2 |
| UD-Q2_K_XL | 0.39 | 4572 | 2629 / 5336 | 43.4 / 69.9 | 5351 / 8262 / 8262 | 23.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 22% nhưng decode chậm hơn 1.44x (23.1 so với 33.2 tok/s), nên không đáng
đổi trên CPU này. Với cùng câu hỏi về màu xanh của bầu trời, Q4 trả lời mạch lạc dù
còn một lỗi khoa học; Q2 lặp câu, mơ hồ và bị cắt dở. Tôi chọn Q4.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.37 | 20000 | 39000 | 39000 | 8.0 | 0.0% |
| 50 | 0.46 | 38000 | 57000 | 58000 | 16.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.27×
- **P95 tăng:** 1.46×
- **Effective concurrency ở 50 users:** 16.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 4.00 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server đã có queue ở 10 user và bão hòa rõ ở 50 user: load tăng 5x nhưng RPS chỉ
tăng 1.27x, P95 tăng lên 57 giây, 4/4 slot đều bận và `deferred` đạt 46. Effective
concurrency 16 gồm cả request xếp hàng, nên latency tăng chủ yếu là queue time. Tôi
sẽ thử `--parallel 8` trước, rồi đo lại P95 vì thêm slot có thể tăng tranh chấp memory.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Không provision cloud resource | stub |
| N17 Data pipeline | `TOY_DOCS` trong source | stub |
| N18 Lakehouse | Không có lakehouse/storage thật | stub |
| N19 Vector + features | Keyword overlap, không có embedding server | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7336.2 ms
- **stage chiếm nhiều nhất:** llm (gần 100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck đúng như kỳ vọng; retrieval stub gần như miễn phí. Muốn giảm latency
pipeline 2x, tôi sẽ tối ưu LLM trước bằng cấu hình 5 thread đã đo, giảm prompt/output
khi phù hợp và thử GPU offload ổn định. Tối ưu retrieval 0.1 ms không tạo khác biệt
đáng kể cho tổng 7336.3 ms.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm thread count từ 10 physical core xuống 5 thread

```
before:  22.2 tok/s ở 10 thread
after:   27.1 tok/s ở 5 thread
speedup: 1.22×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả bất ngờ là peak nằm ở 5 thread, thấp hơn 10 physical core. Decode sinh từng
token và phải đọc lại trọng số model, nên nhanh chóng bị giới hạn bởi memory bandwidth
và cache thay vì thiếu FLOPs. Năm thread đã tạo đủ request bộ nhớ để gần chạm trần
băng thông của máy.

Khi tăng lên 10–12 thread, các thread tranh cùng memory channel/cache và phát sinh thêm
chi phí scheduling, nên throughput chỉ còn khoảng 22.2 tok/s. Oversubscribe 24 thread
làm tình hình rõ hơn: throughput giảm xuống 12.9 tok/s. Vì vậy giảm về 5 thread vừa
giảm contention vừa tăng throughput 1.22x; đây là thay đổi có cơ chế và số đo rõ nhất.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Không làm bonus; ưu tiên hoàn thiện base track.

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

Q2 nhỏ hơn nhưng vừa chậm hơn vừa giảm chất lượng; ít bit không tự động đồng nghĩa với
nhanh hơn. Peak thread ở 5 thay vì 10 core cũng cho thấy bottleneck nằm ở memory system.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi dùng OpenAI Codex để đọc tài liệu, sửa lỗi tương thích PowerShell/UTF-8, chạy các
lệnh benchmark, hỗ trợ phân tích số liệu và soạn bản nháp nhận xét. Tất cả số liệu và
output đều được sinh trực tiếp trên máy đã khai báo; tôi đã đọc và hiểu các kết luận.
