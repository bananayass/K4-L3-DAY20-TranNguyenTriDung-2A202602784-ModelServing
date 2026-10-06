# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 44 | 0.79 | 11000 | 20000 | 20000 | 9.0 | 0.0% |
| 50 | 42 | 0.73 | 32000 | 54000 | 58000 | 22.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.92x** (18% of linear) |
| P95 latency | **2.70x** |
| Effective concurrency at 50 users | 22.9 vs `--parallel 4` slots (occupancy/slot ratio 5.72) |

**Saturated.** Throughput delivered only 0.92x for 5x the offered load, and effective concurrency (22.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.92x while P95 moved 2.70x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Nhận định của mình

50 người dùng làm server bão hòa: tải gấp 5× nhưng throughput còn 0.92×; P95 tăng 2.70×. Busy slots đạt 3.83/4 và có 46 request chờ, xác nhận queueing nhưng chưa tách riêng queue time. Mình thử SLO P95 ≤ 25 giây: 10 người dùng đạt 0.79 RPS/P95 20 giây; 50 người dùng không đạt (0.73 RPS/P95 54 giây). Mình sẽ giới hạn hàng đợi rồi sweep --parallel; thêm slot khi ngl=0 có thể tăng tranh chấp CPU.
