# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3270 | 279 / 315 | 27.0 / 29.6 | 1839 / 2147 / 2147 | 37.1 |
| UD-Q2_K_XL | 0.39 | 3225 | 414 / 459 | 30.1 / 37.9 | 2302 / 2777 / 2777 | 33.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Nhận xét của mình

Q2 nhỏ hơn 0.11 GB (22%) nhưng chậm hơn Q4: 33.2 so với 37.1 tok/s; TTFT P50 là 414 so với 279 ms. Với prompt “tôi laf ai”, Q4 mất ~2.8 giây, hỏi làm rõ; Q2 mất ~4.2 giây nhưng lạc đề. Dù Q2 có vẻ cảm xúc hơn, theo mình nghĩ nên chọn Q4 vì nhanh và hữu ích hơn.
