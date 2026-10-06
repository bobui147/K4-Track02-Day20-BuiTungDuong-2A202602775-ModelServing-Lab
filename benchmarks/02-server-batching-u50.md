# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 12 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2828 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` đạt 4.00/4 slot, đồng thời `requests_processing=4`
và `requests_deferred` lên tới 46. Điều này chứng minh scheduler đã dùng hết bốn slot
và continuous batching thực sự gộp các request trong cùng decode step. Effective
concurrency ở 50 user là 16.0, lớn hơn 4 vì Little's Law tính cả request đang chạy lẫn
request xếp hàng. Vì vậy tôi dùng gauge 4.00 để kết luận slot utilization đạt 100%, và
dùng effective concurrency/deferred để định lượng áp lực hàng đợi.
