# 03 — Tích hợp Milestone 1 (RAG Pipeline)

## Mục đích

Folder này kết nối **llama-server** từ track 02 với các thành phần bạn đã xây dựng từ các ngày trước đó (N16–N19) để tạo thành một **hệ thống RAG (Retrieval-Augmented Generation) hoàn chỉnh**. Mục tiêu là chứng minh endpoint serving của bạn nói được OpenAI-compat đủ tốt để thay thế vào một stack RAG/agent thực tế.

## Ý nghĩa

Trong thực tế, một mô hình ngôn ngữ đơn thuần không đủ để tạo ra câu trả lời chính xác. Nó cần được kết hợp với:
- **Truy vấn vector** — tìm kiếm thông tin liên quan trong cơ sở dữ liệu vector
- **Feature store** — lấy thêm đặc trưng có cấu trúc
- **Lakehouse** — dữ liệu đã qua xử lý

Đây là bài toán **tích hợp** (integration) — kiểm tra xem toàn bộ hệ thống có hoạt động ăn khớp với nhau không.

## Cách thức hoạt động

### Script chính: `pipeline.py`

Script này cài đặt một pipeline RAG hoàn chỉnh với các bước:

```
user query
   │
   ▼
[ 1. retrieve() ]  → tìm top-K tài liệu từ vector index (N19)
   │
   ▼
[ 2. build_prompt() ]  → ghép system prompt + tài liệu + câu hỏi
   │
   ▼
[ 3. call_llm() ]  → POST /v1/chat/completions → http://localhost:8080/v1
   │
   ▼
   answer
```

### Các hàm chính

1. **`retrieve(query, k=3)`** — Truy vấn vector index để tìm top-K tài liệu liên quan nhất
   - *Hiện tại*: Dùng toy data trong memory với keyword overlap (để chạy thử)
   - *Thực tế*: Sẽ gọi vào vector store từ N19 (ví dụ: Qdrant, Pinecone, Chroma)
   - Trả về list `Doc(id, text, score)`

2. **`build_prompt(query, contexts)`** — Xây dựng prompt theo định dạng OpenAI messages
   - System prompt: Hướng dẫn mô hình chỉ trả lời dựa trên tài liệu
   - User message: Ghép các tài liệu + câu hỏi
   - Trả về list các dict `{role, content}`

3. **`call_llm(messages)`** — Gọi llama-server qua HTTP
   - Gửi `POST /v1/chat/completions` với `httpx`
   - Dùng `model: "local"`, `temperature: 0.3`, `max_tokens: 200`
   - Trả về (nội dung trả lời, thời gian xử lý)

4. **`answer(query)`** — Hàm tổng hợp
   - Gọi `retrieve()` → `build_prompt()` → `call_llm()`
   - Đo thời gian từng bước
   - Trả về dict gồm: câu trả lời, các context đã dùng, thời gian từng bước

### Dữ liệu toy mẫu

Để chạy thử mà không cần vector store thật, script cung cấp 5 tài liệu về các chủ đề serving:
- PagedAttention và KV cache fragmentation
- RadixAttention và prefix caching
- Disaggregated serving
- Goodput@SLO
- GGUF quantization

### Các thành phần từ các ngày trước

| Ngày | Thành phần | Vai trò trong pipeline |
|---|---|---|
| **N16** Cloud/IaC | K8s cluster hoặc Compose stack | (không dùng trực tiếp, như là cơ sở hạ tầng) |
| **N17** Data Pipelines | Airflow DAG / batch job | Sản xuất dữ liệu có cấu trúc |
| **N18** Lakehouse | Delta Lake / Iceberg table | Lưu dữ liệu đã xử lý |
| **N19** Vector + Feature Store | Embedding index + Feast | Truy vấn vector & feature lookup |
| **N20** Serving (ngày này) | llama-server | Sinh câu trả lời |

### Ba câu hỏi mẫu

Script sẽ chạy 3 câu hỏi và in ra:
- Các context đã truy xuất (id + score)
- Thời gian từng bước (retrieve, llm, total)
- Câu trả lời (300 ký tự đầu)

```
=== Why is goodput more useful than throughput? ===
  contexts: ['n20-goodput']
  timings : retrieve=0.5ms, llm=1520.3ms, total=1521.0ms
  answer  : Goodput@SLO = req/s satisfying TTFT and TPOT SLOs...
```

### Đo thời gian (Timing)

Mỗi lần gọi `answer()` được bọc trong `time.perf_counter()` để đo:
- `retrieve_ms`: Thời gian truy vấn vector index (thường < 1ms nếu dùng toy data, có thể vài trăm ms nếu dùng vector store thật)
- `llm_ms`: Thời gian llama-server sinh câu trả lời (thường 1–5 giây tuỳ độ dài)
- `total_ms`: Tổng thời gian pipeline

## Kiểm tra live demo

Khi demo, bạn cần chứng minh:

1. **`curl localhost:8080/v1/models`** — Server đang chạy
2. **`python pipeline.py`** — Chạy end-to-end với query mới, hiển thị context và câu trả lời
3. **`/metrics`** — Cho thấy `llamacpp:requests_processing` tăng và `tokens_predicted_total` tăng sau mỗi lần gọi

## Cạm bẫy thường gặp

- **OpenAI SDK version**: Dùng `openai>=1.0.0` hoặc dùng `httpx` trực tiếp (script đã dùng httpx)
- **Token counting**: Tokenizer của llama.cpp không khớp hoàn hảo với OpenAI, nên cắt context theo token-budget cần dùng `Llama.tokenize()` chứ không phải `tiktoken`
- **System prompt caching**: Giữ system prompt giống nhau qua các lần gọi để tận dụng prefix caching. Bạn có thể kiểm tra bằng cách xem `llamacpp:prompt_tokens_total` tăng chậm hơn `tokens_predicted_total` sau lần gọi đầu tiên
