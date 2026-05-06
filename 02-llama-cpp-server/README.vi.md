# 02 — Llama Server (OpenAI-compat + Prometheus + Locust)

## Mục đích

Folder này nâng cấp từ việc dùng llama.cpp như một **thư viện Python** lên một **hệ thống serving thực tế**. Bạn sẽ:
- Chạy `llama-server` — một HTTP daemon đi kèm với llama.cpp
- Gọi API tương thích với OpenAI (`/v1/chat/completions`)
- Giám sát qua endpoint `/metrics` theo chuẩn Prometheus
- Kiểm thử tải với Locust (10–50 người dùng đồng thời)

## Ý nghĩa

Trong thực tế, mô hình ngôn ngữ không được gọi trực tiếp qua thư viện Python mà thông qua một **HTTP server**. Các hệ thống serving chuyên nghiệp như vLLM, SGLang, TensorRT-LLM đều cung cấp API tương thích OpenAI. Folder này mô phỏng chính xác kiến trúc đó — chỉ khác là mô hình và runtime đủ nhỏ để chạy trên laptop.

## Cách thức hoạt động

### 1. Khởi động llama-server

Có hai cách chạy `llama-server`:

**Cách A — Dùng llama-cpp-python (luôn hoạt động):**
```bash
python -m llama_cpp.server --model "$(jq -r .primary_model models/active.json)" \
    --host 0.0.0.0 --port 8080 \
    --n_threads $(python -c 'import json; print(json.load(open("hardware.json"))["cpu"]["cores_physical"] or 4)') \
    --n_gpu_layers 99
```

**Cách B — Dùng bản build native (nhanh hơn):**
```bash
./BONUS-llama-cpp-optimization/llama.cpp/build/bin/llama-server \
    -m "$(jq -r .primary_model models/active.json)" \
    --host 0.0.0.0 --port 8080 \
    -t $(python -c 'import json; print(json.load(open("hardware.json"))["cpu"]["cores_physical"] or 4)') \
    -ngl 99 \
    --parallel 4 --cont-batching \
    --metrics
```

Các flag quan trọng:
- `--parallel N`: Số slot xử lý đồng thời (continuous batching)
- `--cont-batching`: Bật continuous batching — cho phép thêm request mới vào batch khi đang xử lý
- `--metrics`: Bật endpoint `/metrics` cho Prometheus
- `--ngl 99`: Offload 99 lớp xuống GPU

### 2. Các script trong folder

| File | Chức năng |
|---|---|
| `smoke-test.py` | Kiểm tra nhanh API: gửi một request đến `/v1/chat/completions` và xem phản hồi. Dùng thư viện `httpx` thay vì OpenAI SDK. |
| `load-test.py` | Kịch bản Locust với 80% prompt ngắn (chat-style) và 20% prompt dài (RAG-style). Đo P50/P95/P99 dưới tải. |
| `start-server.sh` / `start-server.ps1` | Script tiện lợi để khởi động server, tự động đọc `models/active.json`. |
| `record-metrics.py` | Quét `/metrics` mỗi 2 giây trong thời gian chạy Locust, ghi kết quả ra CSV. |
| `prometheus.yml` | File cấu hình tối thiểu nếu bạn muốn chạy Prometheus riêng. |

### 3. Luồng kiểm thử

```
Terminal 1: llama-server (chạy nền)
     │
     ├── Terminal 2: smoke-test.py     (kiểm tra 1 request)
     ├── Terminal 2: curl /metrics     (xem metrics thủ công)
     └── Terminal 2: locust load-test  (kiểm tra tải)
             │
             └── Terminal 3: record-metrics.py (ghi metrics trong khi chạy tải)
```

### 4. Locust load test — Chi tiết

Script `load-test.py` mô phỏng:
- **Task short_prompt (80%)**: Các câu hỏi ngắn về serving (8 câu hỏi, random)
- **Task long_prompt_rag (20%)**: Đoạn tài liệu dài + câu hỏi (3 câu, ngẫu nhiên)
- `wait_time = between(0.2, 1.5)` — mỗi user đợi 0.2–1.5 giây giữa các request
- Timeout 120 giây cho mỗi request

Chạy với `-u 10` (10 user) rồi `-u 50` (50 user) để thấy sự khác biệt.

### 5. Prometheus Metrics

Endpoint `/metrics` cung cấp các chỉ số:
| Metric | Ý nghĩa |
|---|---|
| `llamacpp:tokens_predicted_total` | Tổng số token đã dự đoán |
| `llamacpp:prompt_tokens_total` | Tổng số token prompt đã xử lý |
| `llamacpp:n_decode_total` | Số lần decode |
| `llamacpp:kv_cache_usage_ratio` | Tỷ lệ sử dụng KV-cache (0.0–1.0) |
| `llamacpp:requests_processing` | Số request đang xử lý |
| `llamacpp:requests_deferred` | Số request đang xếp hàng |

### 6. Các núm vặn để thử nghiệm

| Flag | Tác dụng | Đo lường |
|---|---|---|
| `--parallel N` | Số slot song song | Throughput với N=1,2,4,8 |
| `--cont-batching` | Continuous batching | P95 có và không có, ở -u 50 |
| `--ctx-size 4096` | Cửa sổ ngữ cảnh lớn hơn | KV-cache RAM (`kv_cache_usage_ratio`) |
| `--cache-type-k q8_0` | Lượng tử hóa KV-cache | RAM tiết kiệm vs chất lượng |
| `--metrics` | Bật endpoint Prometheus | Bắt buộc cho giám sát |

## Liên kết với bài giảng (Deck)

| Nội dung deck | Áp dụng ở đây |
|---|---|
| §0 Latency Taxonomy | Các phân vị độ trễ từ Locust |
| §3 PagedAttention | `--parallel` và `--cont-batching` — tương tự continuous batching của vLLM |
| §3 Production Tuning | `--ctx-size`, `--cache-type-k/v`, `-t`, `-ngl` ánh xạ 1:1 với vLLM/SGLang |
| §3 Observability | `/metrics` — mẫu Prometheus tương tự SGLang `:30000/metrics` |
