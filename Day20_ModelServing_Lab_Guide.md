# Day 20 — Model Serving Lab

## Mục tiêu và sản phẩm cần nộp

Dựng llama.cpp server trên laptop; đo TTFT/TPOT và percentile; tìm thread count phù hợp; load test 10/50 user; chứng minh continuous batching qua `/metrics`; phân tích saturation; nối server vào RAG pipeline.

Sản phẩm cuối là một repo GitHub **public** (fork của repo đề bài), có kết quả trong `benchmarks/`, ảnh bằng chứng trong `submission/screenshots/` và `submission/REFLECTION.md`. Bài chấm setup, phép đo, bằng chứng và lập luận; không so tốc độ tuyệt đối giữa các máy. Phần 100 điểm cơ bản không cần GPU, compiler hoặc Docker.

## Chuẩn bị, fork và chạy lệnh

Cần Python ≥3.10, Git, 3–10 GB đĩa trống và mạng tới GitHub/Hugging Face. Chọn model theo RAM:

| RAM | Model / cách chạy |
|---|---|
| ≥8 GB | Gemma 4 E2B (khoảng 5.2 GB tải về) |
| 4–8 GB | Qwen3.5 0.8B (khoảng 0.9 GB); không mất điểm |
| <4 GB | Colab/Kaggle theo `docs/CLOUD.md`; khai báo trong REFLECTION §1 |

Repo đề bài: [K4-Track02-Day20-ModelServing-Lab](https://github.com/VinUni-AI20k/K4-Track02-Day20-ModelServing-Lab). Fork về tài khoản của bạn, đặt tên `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (không dấu, không khoảng trắng), rồi clone **fork của bạn**:

```bash
git clone https://github.com/<tai-khoan>/K4-L3-DAY20-<HoVaTen>-<MSSV>-ModelServing.git
cd K4-L3-DAY20-<HoVaTen>-<MSSV>-ModelServing
git remote -v
```

Kiểm tra `origin` trỏ vào fork cá nhân. Nếu đã fork với tên mặc định, đổi ở **Settings → General → Repository name**.

- macOS/Linux: `make <target>`
- Windows PowerShell: `\.\lab.ps1 <target>`

Ví dụ `make bench` tương ứng `\.\lab.ps1 bench`. Lab dùng virtualenv `.venv`: Python trực tiếp là `.venv/bin/python` trên macOS/Linux hoặc `.venv\Scripts\python` trên Windows. Trước khi đo, đóng bớt tab, IDE nặng và Slack.

## 1. Probe và setup

Probe chạy trước khi cài package, ghi phần cứng vào `hardware.json`:

```bash
make probe
```

Chụp `submission/screenshots/01-hardware-probe.png`. Sau đó setup:

```bash
make setup
# RAM 4–8 GB:
LAB_MODEL=qwen35-0.8b make setup
```

Windows dùng `\.\lab.ps1 setup`; nếu chọn Qwen, đặt biến môi trường `LAB_MODEL` trước khi chạy. Setup tạo venv, cài package, tải llama.cpp binary đã pin và hai quantization. Khi xong có `models/active.json`. Không commit `models/*.gguf`, `runtime/` hoặc `.env`.

Nếu Hugging Face bị chặn: `docs/MANUAL-DOWNLOAD.md`. Lỗi `unknown model architecture: 'gemma4'` thường do runtime cũ; chạy `make runtime` hoặc Windows `\.\lab.ps1 runtime`.

**Checkpoint:** có `hardware.json`, `models/active.json` và thông báo Setup complete.

## 2. Baseline latency và quantization

- **TTFT** là thời gian từ khi gửi request tới token đầu tiên; gồm prefill và thời gian chờ.
- **TPOT** là thời gian trung bình mỗi token decode sau token đầu.
- **P50/P95/P99** mô tả phân phối từ điển hình tới tail latency.

Prefill chủ yếu tốn compute và tăng theo độ dài prompt. Decode thường bị giới hạn memory bandwidth vì mỗi token phải đọc weights. Quantization thấp hơn có thể giảm bytes di chuyển, nhưng dequantization có thể làm chậm trên máy bị giới hạn compute.

Chạy cả hai quantization:

```bash
make bench
```

Script warm-up một lần, stream 10 prompt qua API rồi lặp lại với quantization khác. Kết quả ở `benchmarks/01-quickstart-results.md`. Chụp `submission/screenshots/02-bench.png`.

So chất lượng bằng cùng một câu hỏi. Server 2-bit có thể chạy riêng:

```bash
.venv/bin/python labs/02-serve/serve.py --compare --port 8090
```

Điền observation trong report: 2-bit nhanh/chậm hơn bao nhiêu, nhỏ hơn bao nhiêu và chất lượng có đáng đánh đổi không. Ghi rõ nếu số đo lấy từ lần chạy đầu (weights chưa ở page cache).

## 3. Tune thread count

Decode thường tăng tốc khi thêm thread tới một ngưỡng rồi đi ngang/giảm. Thread thừa tranh memory bandwidth hoặc tăng scheduling overhead; hyperthread chia sẻ tài nguyên với core vật lý. Điểm đường cong ngừng tăng gọi là **knee**.

```bash
make tune
```

Kết quả sweep `tg128` nằm trong `benchmarks/01-tuning-tg128.md`. Điền Best, Slowest tested, mức tốt nhất so với physical-core default, vị trí knee và cơ chế khả dĩ. Nếu curve phẳng hoặc vẫn tăng ở logical cores, mô tả đúng dữ liệu.

Tuỳ chọn chạy baseline với thread tốt nhất:

```bash
LAB_N_THREADS=<N> make bench
```

Windows PowerShell: `$env:LAB_N_THREADS = '<N>'` rồi `\.\lab.ps1 bench`. **Checkpoint:** bảng thread → tok/s, xác định knee và giải thích một đến hai câu.

## 4. Server và smoke test

Mở terminal 1, để server chạy:

```bash
make serve
```

Mặc định port 8080, `--parallel 4`, continuous batching, metrics, context 2048, reasoning off. Endpoint chính:

| Endpoint | Mục đích |
|---|---|
| `POST /v1/chat/completions` | OpenAI-compatible completion |
| `GET /metrics` | Prometheus metrics |
| `GET /slots` | Trạng thái slot |
| `GET /health` | Health check |

Terminal 2:

```bash
make smoke
```

Smoke gửi request và đọc metrics trước/sau. Các chỉ số cần để ý: `llamacpp:tokens_predicted_total`, `prompt_tokens_total`, `n_decode_total`, `requests_processing`. Thành công khi có completion và tokens predicted khác 0. Chụp `submission/screenshots/03-serve-and-smoke.png` (có thể tách thành 03a/03b).

Nếu port 8080 bị chiếm, đặt `LAB_SERVER_PORT=8090` và dùng nhất quán cho serve, smoke, load, metrics, pipeline. Có thể truyền tham số llama-server sau `--`, ví dụ đổi `--ctx-size 4096`; context lớn tăng KV cache/RSS.

## 5. Load test và continuous batching

Locust mô phỏng 80% chat ngắn (48 output tokens), 20% prompt dài kiểu RAG (96 tokens), nghỉ 0.2–1.5 giây giữa request.

Chạy tải 10 user trong 60 giây:

```bash
make load-10
```

Chụp `submission/screenshots/04-locust-10.png`, thấy bảng có Median, 95%ile, 99%ile. Tiếp theo chạy 50 user ở terminal 2:

```bash
make load-50
```

**Trong khi load-50 chạy**, terminal 3 chạy:

```bash
make metrics
```

Gauge `llamacpp:n_busy_slots_per_decode` là số slot trung bình cùng decode mỗi bước. Giá trị >1 và tiến gần `--parallel` là bằng chứng nhiều request cùng được xử lý (continuous batching). `requests_deferred > 0` nghĩa là có request phải chờ slot. Metrics phải được lấy đồng thời với tải; khi server rảnh, busy slots ≈1 không chứng minh batching.

Khi load-50 kết thúc, chụp `submission/screenshots/05-locust-50.png`. Điền observation trong `benchmarks/02-server-batching-u50.md`. CSV là `benchmarks/locust-10_stats.csv` và `locust-50_stats.csv`. Nếu `kv_cache_usage_ratio` là `n/a`, không ghi thay bằng 0. Máy yếu có thể cần chạy lâu hơn (`-t 3m)) hoặc giảm `LAB_LOAD_SHORT_TOKENS` để tăng mẫu.

## 6. Saturation, Little’s Law và goodput

Sau khi có cả hai CSV:

```bash
make load-report
```

Đọc `benchmarks/02-server-results.md`: offered load, throughput/RPS thực tế, mức tăng so với tuyến tính, hệ số tăng P95, effective concurrency và kết luận.

**Little’s Law:** `L = λ × W`. Script ước lượng effective concurrency = RPS × latency trung bình. Nó tính cả request đang chạy lẫn đang đợi:

- concurrency ≤ số slot: request thường tìm được slot; latency chủ yếu là thời gian phục vụ.
- concurrency > số slot: có queue; một phần tăng latency đến từ chờ slot.

**Goodput@SLO** là số request/giây vừa hoàn tất vừa đạt ngưỡng latency do bạn chọn; throughput tính mọi request. Quá saturation, throughput có thể tăng ít trong khi P95 tăng mạnh, làm goodput giảm.

Script kết luận **Saturated**, **At capacity, still scaling** hoặc **Not saturated**, dựa trên throughput có phẳng và slot có gần kín không. Nếu cảnh báo Small sample (dưới 20 request), diễn giải thận trọng. Điền report: tải bão hòa ở đâu, RPS 10→50 tăng bao nhiêu lần, P95 tăng bao nhiêu, concurrency so với slot và knob đầu tiên sẽ đổi để nâng goodput. Có thể so 1 slot với 4 slot bằng `LAB_PARALLEL=1` nhất quán khi chạy serve và load-report.

## 7. RAG pipeline

Giữ server bật rồi chạy:

```bash
make pipeline
```

Pipeline chạy ba câu hỏi, in contexts, answer và latency từng stage: embed → retrieve → LLM. Mặc định `TOY_DOCS` và `retrieve()` là stub; retrieval dùng keyword overlap, embed là 0 ms nếu không có embedding server. Dùng stub không mất điểm nếu khai báo đúng.

Điền `benchmarks/03-integration-results.md`: N16–N19 (cluster, data pipeline, lakehouse, vector index) là real hay stub; stage nào chậm nhất; sẽ tối ưu stage nào nếu cần giảm latency một nửa. Nếu có embed server:

```bash
make serve-embed
.venv/bin/python labs/03-integrate/pipeline.py --embed-url http://localhost:8081
```

Giữ system prompt giống hệt từng byte để có thể reuse prefix cache. Context dài làm prefill/TTFT tăng. Context mặc định 2048 được chia giữa slots; dùng endpoint `/tokenize` của llama.cpp để đếm token.

## 8. REFLECTION, verify và nộp

Điền `submission/REFLECTION.md`:

| Mục | Nội dung | Rubric |
|---|---|---|
| §1 | Hardware/runtime; khai báo Colab/Kaggle nếu dùng | 1, 2 |
| §2 | Baseline hai quantization và nhận xét 2-bit | 3, 4, 5 |
| §3 | Load results, busy slots và saturation reading | 8, 9, 10 |
| §4 | Real/stub N16–N19, latency theo stage | 12, 13 |
| §5 | Before/after và cơ chế của thay đổi hiệu quả nhất | 11 |
| §9 | Khai báo dùng AI theo `docs/RULES.md` | — |

Số trong REFLECTION phải khớp `benchmarks/*.md`. Ảnh cần có trong `submission/screenshots/`:

1. `01-hardware-probe.png`
2. `02-bench.png`
3. `03-serve-and-smoke.png` (hoặc 03a/03b)
4. `04-locust-10.png`
5. `05-locust-50.png`

Commit file rồi verify và push:

```bash
make verify
git push
```

Verify phải exit 0. Sửa placeholder, file thiếu hoặc thay đổi chưa commit theo checklist. Trên GitHub kiểm tra fork đúng tên, public, có đủ benchmarks/submission; mở cửa sổ ẩn danh để kiểm tra quyền xem. Paste URL repo lên VinUni LMS trước 23:59 giờ Việt Nam ngày làm lab, trừ khi key coach thông báo deadline khác. Repo private bị 0 điểm; không commit weights, runtime hoặc secrets.

Checklist nộp: verify exit 0; không còn `required -- replace this line`; REFLECTION khớp report; tên repo đúng; URL public xem được.

## 9. Bonus (tùy chọn, tối đa +10)

Chỉ làm sau khi base `make verify` thành công. Mỗi tiêu chí 2 điểm; chọn 1–2 mục có phân tích sâu:

| Bonus | Yêu cầu |
|---|---|
| B1 | Compile llama.cpp và so với prebuilt: `make build-llama && make compare-builds` (cần cmake) |
| B2 | Sweep quant/ctx/batch/GPU: `make sweep-quant`, `sweep-ctx`, `sweep-batch`, `sweep-gpu` |
| B3 | Before/after từ bonus trong REFLECTION §6 |
| B4 | Làm challenge C1–C7 hoặc C10 trong `docs/bonus/CHALLENGES.md` |
| B5 | So sánh runtime/regime: MLX (Mac), C8, C9 hoặc C6; ví dụ `make semantic-cache`, `make embed-demo` |

Mỗi script tạo `benchmarks/bonus-*.md`; thay placeholder và commit. §6 cần before/after của bonus, không dùng lại `make tune`.

## Lỗi thường gặp

| Lỗi | Cách xử lý |
|---|---|
| `unknown model architecture: 'gemma4'` | Runtime cũ; chạy `make runtime` / Windows `\.\lab.ps1 runtime`. |
| Port 8080 bị chiếm | Đặt `LAB_SERVER_PORT=8090`, dùng cho mọi bước. |
| Serve không thấy venv | Chạy setup trước. |
| busy_slots ≈ 1 | Lấy metrics đồng thời với `make load-50`. |
| Scrape failed | Bật server trước. |
| Hugging Face bị chặn | Xem `docs/MANUAL-DOWNLOAD.md`. |
| Verify báo lỗi | Đọc checklist; thường còn placeholder/file thiếu/chưa commit. |

## Thang điểm base

| Hạng mục | Điểm |
|---|---:|
| Setup | 10 |
| Đo lường | 20 |
| Serving | 25 |
| Phân tích | 20 |
| Integration | 15 |
| Submission | 10 |
| **Tổng** | **100** |

