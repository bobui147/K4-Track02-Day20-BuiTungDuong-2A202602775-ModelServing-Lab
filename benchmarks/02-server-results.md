# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=5` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 20 | 0.37 | 20000 | 39000 | 39000 | 8.0 | 0.0% |
| 50 | 27 | 0.46 | 38000 | 57000 | 58000 | 16.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.27x** (25% of linear) |
| P95 latency | **1.46x** |
| Effective concurrency at 50 users | 16.0 vs `--parallel 4` slots (occupancy/slot ratio 4.00) |

**Saturated.** Throughput delivered only 1.27x for 5x the offered load, and effective concurrency (16.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.27x while P95 moved 1.46x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server đã có queue ngay ở 10 user (effective concurrency 8.0 so với 4 slot) và bão
hòa rõ ở 50 user: offered load tăng 5x nhưng RPS chỉ tăng 1.27x, trong khi P95 tăng
1.46x lên 57 giây. Metrics xác nhận cả 4/4 slot đều bận và có lúc 46 request bị
deferred, nên phần latency tăng thêm chủ yếu là queue time chứ không phải mỗi request
đột nhiên cần nhiều compute hơn.

Knob tôi sẽ thử đầu tiên là tăng `--parallel` từ 4 lên 8 vì hàng đợi đang lớn trong
khi toàn bộ slot hiện tại đã kín; thread count đã được tune ở CP2. Sau đó cần đo lại
P95 và goodput@SLO, vì thêm slot cũng có thể làm các request tranh memory bandwidth
và tăng TPOT. Nếu P95 xấu đi, admission control ở mức gần 4 request đồng thời sẽ phù
hợp hơn cho một SLO latency chặt.
