# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **10 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 9.6 | 35% |
| 5 | 27.1 | 100% |
| 10 | 22.2 | 82% |
| 12 | 22.3 | 82% |
| 24 | 12.9 | 48% |

**Best**: `-t 5` at 27.1 tok/s
**Slowest tested**: `-t 1` at 9.6 tok/s (2.84x spread)
**Against the physical-core default** (`-t 10`, 22.2 tok/s): 1.22x

Use this in your run:

```bash
LAB_N_THREADS=5 make bench
```

## Your explanation

Knee của đường cong nằm ở 5 thread với 27.1 tok/s. So với cấu hình mặc định dùng
10 physical core (22.2 tok/s), giảm từ 10 xuống 5 thread tạo speedup 1.22x. Tăng lên
12 thread gần như không cải thiện so với 10 thread, còn oversubscribe 24 thread làm
throughput giảm xuống 12.9 tok/s.

Decode phải đọc trọng số model lặp lại cho từng token nên nhanh chóng chạm giới hạn
memory bandwidth và cache thay vì tận dụng hết số core. Sau 5 thread, các thread bổ
sung tranh cùng memory channel/cache và tạo thêm chi phí scheduling; vì vậy nhiều
thread hơn không đồng nghĩa với nhiều tok/s hơn trên CPU này.
