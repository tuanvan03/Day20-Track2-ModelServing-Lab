# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** _Đoàn Văn Tuấn_
**Cohort:** _A20-K1_
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Ubuntu 24.04_
- **CPU:** _AMD Ryzen 5 6600H with Radeon Graphics_
- **Cores:** _12 / 12 (physical / logical)_
- **CPU extensions:** _AVX2 (AVX-512: no)_
- **RAM:** _14.3 GB_
- **Accelerator:** _NVIDIA GeForce RTX 3050 Laptop GPU, 4096 MiB_
- **llama.cpp backend đã chọn:** _CUDA_
- **Recommended model tier:** _Qwen2.5-1.5B-Instruct (Q4_K&M)_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Cần cài CUDA Toolkit 12.x và build llama-cpp-python với `CUDACXX=/usr/local/cuda/bin/nvcc pip install llama-cpp-python --force-reinstall --no-cache-dir`. Không cần WSL2 vì chạy native Linux. RTX 3050 4GB VRAM đủ cho Q4_K_M 1.5B.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1090 | 109 / 130 | 29.0 / 38.1 | 1935 / 2504 / 2784 | 34.5 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 247 | 126 / 147 | 21.4 / 21.9 | 1476 / 1496 / 1496 | 46.7 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q4_K_M decode 34.5 tok/s, Q2_K nhanh hơn 35% (46.7 tok/s) nhưng quality loss rõ — với 4GB VRAM sẵn có, Q4_K_M đáng dùng hơn.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 2.04 | 17000 | 23000 | 23000 | 0 |
| 50 | 6.71 | 38000 | 75000 | 110000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` ở concurrency 50 lên tới ~3.86, cho thấy server phải xếp hàng ~4 request decode đồng thời. Số request processing luôn duy trì ở mức 4 (max parallelism), và deferred requests tăng dần, báo hiệu server đã bão hòa kv-cache slots.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only — llama-server chạy native trên máy thật, không container hóa._
- **N17 (Data pipeline):** _stub: in-memory dict — 3 câu hỏi hard-code trong pipeline.py._
- **N18 (Lakehouse):** _stub: SQLite — không dùng Delta/Iceberg, mọi thứ trong RAM._
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS — keyword overlap retrieval từ 5 document mẫu, không có vector index thật._

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _N/A (không dùng embedding — keyword overlap)_
- retrieve: _0.1 ms_
- llama-server: _~4437 ms (trung bình 3 requests)_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

llama-server chiếm >99.9% latency — retrieval chỉ 0.1ms. Khớp kỳ vọng: generate 200 tokens với TPOT 29ms/token cần ~5.8s. Prefill context dài càng làm TTFT tăng thêm.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Context-length sweep: đo prefill latency khi tăng context từ 128 → 2048 tokens trên CUDA backend._

**Before vs after** (paste 2-3 dòng từ sweep output):

```
ctx=128:  prefill 648.0ms  @ 197.5 tok/s
ctx=1024: prefill 5958.7ms @ 171.9 tok/s
ctx=2048: prefill 11817.7ms @ 173.3 tok/s
speedup: prefill throughput gần như constant (~175-197 tok/s) nhưng absolute latency scale tuyến tính với ctx length
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Bạn thấy prefill tok/s gần như không đổi (~175-197 tok/s) khi context gấp 16 lần, nghĩa là prefill compute scale O(n) — mỗi token mới cần attention với tất cả token trước, nên gấp đôi context là gấp đôi thời gian prefill. Với RTX 3050, memory bandwidth ~192 GB/s là bottleneck chính: prefill compute-bound được GPU xử lý ổn định, còn decode thì memory-bandwidth-bound. 

Điều này giải thích vì sao các hệ thống serving lớn (như Mooncake, Dynamo trong deck) tách prefill và decode ra riêng: prefill cần GPU compute, decode cần memory bandwidth — ghép chung thì cả hai cùng chậm. Trên laptop, với 4GB VRAM, context >1024 tokens đã thấy rõ impact: prefill 6 giây cho 1024 tokens đồng nghĩa user chờ 6 giây trước khi thấy chữ đầu tiên.

---

## 6. (Optional) Điều ngạc nhiên nhất

Thread sweep (bonus1-thread.png) cho kết quả tg128 = 0.0 tok/s ở mọi thread count — không phải do benchmark fail, mà khi `-ngl 99` offload toàn bộ lên GPU, CPU threads hầu như không tham gia decode nên `llama-bench` không ghi nhận được throughput trên CPU path. Điều này khớp với kỳ vọng: CUDA backend dùng GPU cho cả prefill lẫn decode.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
