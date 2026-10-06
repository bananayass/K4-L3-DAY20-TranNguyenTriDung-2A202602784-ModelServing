# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 11.5 | 31% |
| 4 | 34.1 | 92% |
| 8 | 37.2 | 100% |
| 16 | 10.1 | 27% |
| 32 | 0.8 | 2% |

**Best**: `-t 8` at 37.2 tok/s
**Slowest tested**: `-t 32` at 0.8 tok/s (48.96x spread)
**Against the physical-core default** (`-t 8`, 37.2 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Giải thích của mình

Mình thấy điểm tối ưu nằm ở 8 luồng, tương ứng với 8 lõi vật lý: thông lượng đạt đỉnh ở
37.2 tok/s. Tốc độ giảm còn 10.1 tok/s khi dùng 16 luồng và 0.8 tok/s khi dùng 32.
Với `ngl=0`, mô hình chạy suy luận trên CPU; số worker bổ sung vượt quá số lõi vật
lý và làm tăng chi phí lập lịch cùng tranh chấp bộ nhớ thay vì giúp giải mã hiệu quả.
