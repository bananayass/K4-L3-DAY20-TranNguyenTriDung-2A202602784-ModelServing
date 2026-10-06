# Bonus - Batch-size sweep (chunked prefill)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=8` `ngl=0` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 212.1 | 74% |
| 256 | 256 | 191.2 | 67% |
| 512 | 256 | 139.4 | 49% |
| 512 | 512 | 286.8 | 100% |
| 1024 | 512 | 196.3 | 68% |
| 2048 | 512 | 199.6 | 70% |

Best: `-b 512 -ub 512` at 286.8 tok/s
(2.06x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Nhận xét của mình

Với prefill 512 token trên Qwen3.5 0.8B, -b 512 -ub 512 đạt 286.8 tok/s, nhanh hơn 2.06× so với cấu hình chậm nhất -b 512 -ub 256 (139.4 tok/s). Vì -b được giữ nguyên, kết quả này cho thấy micro-batch 512 xử lý prompt trong một chunk thay vì chia thành hai chunk, giảm overhead giữa các lượt prefill. Đây mới là throughput đo bằng llama-bench; trước khi chọn cấu hình này cho server chịu tải, cần chạy load test cùng điều kiện ở hai mức -ub và so sánh P95, RPS cùng tỷ lệ lỗi.
