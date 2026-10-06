# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7332 | 2153 / 2608 | 30.1 / 44.6 | 3987 / 4774 / 4774 | 33.2 |
| UD-Q2_K_XL | 0.39 | 4572 | 2629 / 5336 | 43.4 / 69.9 | 5351 / 8262 / 8262 | 23.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.44x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên phép đo CPU (`ngl=0`), `UD-Q2_K_XL` nhỏ hơn `Q4_K_M` 0.11 GB, tương đương
22%, và nạp model nhanh hơn 37.6%. Tuy nhiên, Q2 có TTFT P50 cao hơn 22.1%, TPOT
P50 cao hơn 44.2%, và decode chỉ đạt 23.1 tok/s so với 33.2 tok/s của Q4. Như vậy,
Q2 chậm hơn 1.44 lần khi decode dù dung lượng nhỏ hơn. Kết quả này cho thấy máy đang
bị giới hạn bởi chi phí tính toán/dequantization của quantization nặng hơn, nên lợi ích
giảm lượng dữ liệu đọc từ bộ nhớ không bù được chi phí đó.

Tôi hỏi cả hai model cùng câu: "Giải thích ngắn gọn vì sao bầu trời có màu xanh, bằng
tiếng Việt." Q4 trả lời mạch lạc nhưng vẫn nhầm rằng ánh sáng đỏ bị tán xạ nhiều hơn
ánh sáng xanh. Q2 lặp lại cùng một câu nhiều lần, giải thích mơ hồ và bị cắt dở. Vì Q2
vừa chậm hơn vừa giảm chất lượng rõ rệt, phần tiết kiệm 0.11 GB không đáng để đánh đổi
trên máy này; tôi chọn Q4 cho các checkpoint tiếp theo.
