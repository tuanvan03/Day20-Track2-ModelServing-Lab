# 01 — Khởi động nhanh với llama.cpp (Quickstart Benchmark)

## Mục đích

Folder này cung cấp một **bài benchmark nhanh (khoảng 30 phút)** để bạn làm quen với các khái niệm cốt lõi trong serving mô hình ngôn ngữ, trực tiếp trên máy tính của mình. Bạn sẽ đo lường các chỉ số hiệu năng quan trọng và so sánh hai phiên bản lượng tử hóa khác nhau của cùng một mô hình.

## Ý nghĩa

Trước khi đi sâu vào các kỹ thuật serving phức tạp, bạn cần hiểu rõ **các chỉ số hiệu năng cơ bản**:
- **TTFT (Time To First Token)** — Thời gian từ khi gửi prompt đến khi nhận token đầu tiên
- **TPOT (Time Per Output Token)** — Thời gian cho mỗi token đầu ra
- **P50 / P95 / P99** — Phân vị thứ 50, 95, 99 của độ trễ đầu cuối (end-to-end)

Hiểu các chỉ số này giúp bạn đánh giá và tối ưu hệ thống serving sau này.

## Cách thức hoạt động

### Script chính: `benchmark.py`

Script này thực hiện các bước sau:

1. **Đọc cấu hình** — Đọc file `models/active.json` để biết đường dẫn mô hình và `hardware.json` để biết thông số phần cứng.

2. **Tải mô hình chính (Q4_K_M)** — Khởi tạo một đối tượng `Llama` từ thư viện `llama-cpp-python` với các tham số:
   - `n_ctx` (mặc định 2048): Kích thước cửa sổ ngữ cảnh
   - `n_threads` (mặc định = số nhân vật lý): Số luồng CPU
   - `n_batch` (mặc định 512): Kích thước batch cho prefill
   - `n_gpu_layers` (mặc định: tự động): Số lớp đẩy xuống GPU

3. **Chạy warm-up** — Một lần gọi ngắn để loại bỏ nhiễu khởi tạo (cold-start), vì lần tải đầu tiên có thể mất 10+ giây.

4. **Benchmark 10 prompt** — Chạy 10 câu hỏi chuyên ngành serving, mỗi câu đo:
   - `ttft_ms`: Thời gian đến token đầu tiên (ms)
   - `tpot_ms`: Thời gian mỗi token đầu ra (ms)
   - `e2e_ms`: Thời gian đầu cuối (ms)
   - `n_tokens`: Số token sinh ra

5. **Tính toán phân vị** — Tính P50, P95, P99 từ 10 kết quả đo.

6. **So sánh lượng tử hóa** — Tải tiếp bản Q2_K và chạy lại cùng 10 prompt, cho kết quả so sánh trực tiếp.

7. **Xuất báo cáo** — Ghi kết quả vào `benchmarks/01-quickstart-results.md`.

### Các chỉ số quan trọng

```
TTFT (Time To First Token):     128 ms     ← Thời gian prefill (xử lý prompt đầu vào)
TPOT (Time Per Output Token):    34 ms     ← Thời gian decode (sinh token)
Decode rate:                     29 tok/s  ← Tốc độ sinh token
P50 / P95 / P99 e2e (32 tok):   1.21 / 1.45 / 1.62 s
```

### Giải thích

- **TTFT** bị chi phối bởi tính toán prefill — prompt càng dài thì TTFT càng lớn.
- **TPOT** bị chi phối bởi băng thông bộ nhớ KV-cache — mô hình nhỏ trên RAM nhanh sẽ cho TPOT tốt.
- **Q4_K_M vs Q2_K**: Q4_K_M dùng nhiều RAM hơn (~2×) nhưng chất lượng cao hơn rõ rệt. Q2_K tiết kiệm RAM nhưng giảm chất lượng và thường tăng TPOT khoảng 30–60%.

### Các biến môi trường để thử nghiệm

| Biến | Mặc định | Tác dụng |
|---|---|---|
| `LAB_N_CTX` | 2048 | Kích thước cửa sổ ngữ cảnh. Lớn hơn = nhiều bộ nhớ KV-cache hơn |
| `LAB_N_THREADS` | Số nhân vật lý | Số luồng CPU. **Nhiều hơn ≠ nhanh hơn** vì tác vụ thường bị giới hạn bởi băng thông |
| `LAB_N_BATCH` | 512 | Batch size xử lý prompt (prefill) |
| `LAB_N_GPU_LAYERS` | 0 hoặc 99 | Số lớp đẩy xuống GPU. `99` = offload toàn bộ |
| `LAB_TEMPERATURE` | 0.7 | Nhiệt độ sampling |

### Cạm bẫy thường gặp

- **Cold-start overhead**: Lần gọi đầu tiên bao gồm thời gian tải mô hình. Bỏ qua lần đầu khi tính P50/P95/P99.
- **OS file cache**: Lần chạy thứ hai nhanh hơn 5–10× vì file đã được cache. Cả hai số đều có ý nghĩa, nhưng cần biết bạn đang báo cáo cái nào.
- **Background processes**: Trình duyệt, IDE chiếm CPU và băng thông. Tắt chúng trước khi benchmark.

## Liên kết với bài giảng (Deck)

| Nội dung deck | Áp dụng ở đây |
|---|---|
| §0 Latency Taxonomy | Các chỉ số TTFT/TPOT/P95 bạn vừa đo |
| §1 Quantization | So sánh Q4_K_M vs Q2_K |
| §2 KV Cache & Attention | `LAB_N_CTX` kiểm soát dung lượng KV-cache |
| §3 Production Tuning | `LAB_N_THREADS`, `LAB_N_BATCH`, `LAB_N_GPU_LAYERS` là các núm vặn |
