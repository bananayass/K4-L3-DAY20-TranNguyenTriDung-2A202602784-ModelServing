# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Nguyễn Trí Dũng
**MSSV:** 2A202602784
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux trên WSL2 (kernel 5.15.167.4-microsoft-standard-WSL2)
- **CPU:** AMD Ryzen 7 5800H with Radeon Graphics
- **Cores:** 8 lõi vật lý / 16 lõi logic
- **CPU extensions:** AVX2
- **RAM:** 15.3 GB
- **Accelerator:** GPU NVIDIA GeForce RTX 3050 Ti Laptop, 4 GB; đã được nhận diện nhưng runtime Linux Vulkan của lab không hỗ trợ GPU offload (`ngl=0`)
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** Laptop cá nhân chạy WSL2
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Mình chọn Qwen3.5 0.8B và chạy mô hình trên máy cá nhân. Probe nhận diện GPU RTX 3050 Ti; runtime đang dùng là gói Vulkan được ghim trong lab. Các benchmark ghi `ngl=0`, nên những lượt đo này chạy bằng CPU, chưa offload lên GPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3270 | 279 / 315 | 27.0 / 29.6 | 1839 / 2147 / 2147 | 37.1 |
| UD-Q2_K_XL | 0.39 | 3225 | 414 / 459 | 30.1 / 37.9 | 2302 / 2777 / 2777 | 33.2 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 0.11 GB (22%) nhưng chậm hơn Q4: 33.2 so với 37.1 tok/s; TTFT P50 là 414 so với 279 ms. Với prompt “tôi laf ai”, Q4 mất ~2.8 giây, hỏi làm rõ; Q2 mất ~4.2 giây nhưng lạc đề. Dù Q2 có vẻ cảm xúc hơn, theo mình nghĩ nên chọn Q4 vì nhanh và hữu ích hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.79 | 11000 | 20000 | 20000 | 9.0 | 0.0% |
| 50 | 0.73 | 32000 | 54000 | 58000 | 22.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.92×
- **P95 tăng:** 2.70×
- **Effective concurrency ở 50 users:** 22.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.83 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

50 người dùng làm server bão hòa: tải gấp 5× nhưng RPS còn 0.92×; P95 tăng 2.70×. Busy slots đạt 3.83/4 và có 46 request chờ, xác nhận queueing nhưng chưa tách riêng queue time. Mình thử SLO P95 ≤ 25 giây: 10 người dùng đạt 0.79 RPS ở 20 giây; 50 người dùng không đạt (54 giây). Mình sẽ giới hạn hàng đợi trước rồi đo lại `--parallel`; thêm slot khi `ngl=0` có thể tăng tranh chấp CPU.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost only | stub |
| N17 Data pipeline | in-memory list | stub |
| N18 Lakehouse | `TOY_DOCS` (toy dict) | stub |
| N19 Vector + features | `TOY_DOCS` + keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 5283.6 ms
- **stage chiếm nhiều nhất:** LLM (100% of total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Pipeline RAG dùng tài liệu mẫu và tìm theo từ khóa; chỉ llama-server là thật. LLM chiếm 5,283.6 ms trung bình, còn truy xuất 0.1 ms. Để giảm tổng độ trễ 2×, mình sẽ giảm token cần sinh hoặc tăng tốc giải mã; tối ưu truy xuất gần như không giúp vì thời gian của nó rất nhỏ.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số luồng CPU từ `-t 16` xuống `-t 8`

```
before:  10.1 tok/s (`-t 16`, tg128)
after:   37.2 tok/s (`-t 8`, tg128)
speedup: 3.68×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Tám luồng tương ứng với 8 lõi vật lý của máy và cho tốc độ giải mã cao nhất trong các lần đo. Ở 16 luồng, thông lượng giảm còn 10.1 tok/s; ở 32 luồng, còn 0.8 tok/s. Vì không có GPU offload, thêm worker CPU khiến số luồng vượt quá số lõi vật lý và làm các worker tranh chấp tài nguyên xử lý, bộ nhớ nhiều hơn thời gian giải mã. Giảm từ 16 xuống 8 luồng giúp tốc độ đo được tăng 3.68×.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 sweep-batch; B3 before/after từ sweep này; B5/C9 embedding serving bằng make serve-embed và make embed-demo.

**Numbers:**

```
before:  139.4 tok/s (`-b 512 -ub 256`, pp512)
after:   286.8 tok/s (`-b 512 -ub 512`, pp512)
speedup: 2.06×
```

**Điều này nói lên gì mà deck chưa nói:**

Trên Qwen3.5 0.8B Q4_K_M, threads=8, ngl=0, giữ -b=512 và tăng -ub từ 256 lên 512 làm prefill 512 token tăng từ 139.4 lên 286.8 tok/s (2.06×). Với micro-batch 512, prompt này được xử lý trong một chunk thay vì hai chunk 256 token, nên giảm overhead giữa các lượt prefill. Đây là throughput của llama-bench; sweep chưa đo P95 khi có cạnh tranh, nên mình chưa kết luận cấu hình này tốt hơn cho server đông người dùng.

Với C9, make embed-demo dùng Qwen chat GGUF ở pooling mode: throughput là 8.5 texts/s ở batch 1, đạt cao nhất 9.7 ở batch 2, rồi giảm còn 9.5, 8.8 và 7.9 ở batch 4, 8 và 16. Embedding chỉ cần một forward pass cho mỗi văn bản, nên có thể gom batch tĩnh; chat còn có vòng decode và cần continuous batching để dùng các slot hiệu quả. Hai kết quả có đơn vị và workload khác nhau nên không so tốc độ trực tiếp. Đây cũng chỉ là decoder được pooling, không phải embedding model chuyên dụng; top match có cosine 0.896 nhưng các kết quả kế tiếp là 0.850 và 0.821, nên demo chưa chứng minh chất lượng retrieval production.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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

Mình đã dùng ChatGPT/OpenAI Codex để đọc hướng dẫn lab, xử lý lỗi setup, diễn giải số liệu. Giải thích các khái niệm và quy trình chưa hiểu
