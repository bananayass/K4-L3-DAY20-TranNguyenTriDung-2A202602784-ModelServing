# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 31 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.83 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5191 |

Highest sampled value was **3.83 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét của mình

Gauge ghi nhận trung bình cao nhất 3.83/4 busy slots mỗi bước giải mã, với 4 request đang xử lý và tối đa 46 request chờ. Con số này không cần bằng effective concurrency 22.9: gauge phản ánh mức batching ở bước decode, còn effective concurrency theo định luật Little tính cả request trong hàng đợi. Mình dùng gauge để đánh giá batching và effective concurrency để nhận biết áp lực hàng đợi.
