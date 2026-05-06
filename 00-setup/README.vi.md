# 00 — Thiết lập Môi trường (Setup)

## Mục đích

Folder này chịu trách nhiệm **thiết lập toàn bộ môi trường** để chạy các bài thực hành trong lab. Nó tự động:
- Phát hiện cấu hình phần cứng máy tính của bạn (CPU, GPU, RAM)
- Cài đặt các thư viện Python cần thiết
- Tải xuống mô hình ngôn ngữ (GGUF) phù hợp với dung lượng RAM
- Biên dịch `llama-cpp-python` với backend tối ưu cho phần cứng

## Ý nghĩa

Trước khi có thể chạy inference (suy luận) mô hình ngôn ngữ trên máy local, cần có một môi trường đồng nhất và tối ưu. Folder này giải quyết vấn đề **tái lập** (reproducibility): dù bạn dùng Linux, macOS hay Windows, dùng GPU NVIDIA, AMD hay chỉ có CPU, kết quả sau khi chạy setup sẽ là một môi trống thống nhất có thể dùng cho các bước sau.

## Cách thức hoạt động

### Luồng xử lý chính

1. **Phát hiện phần cứng** — Script `detect-hardware.py` quét máy tính của bạn:
   - Thông tin CPU: kiểu, số nhân vật lý (physical cores), số nhân logic (logical cores), các tập lệnh AVX2/AVX512/NEON
   - Dung lượng RAM tổng
   - GPU khả dụng: NVIDIA CUDA, AMD ROCm, Apple Metal, Vulkan, hoặc CPU-only
   - Đề xuất mô hình phù hợp dựa trên RAM (ví dụ: Qwen2.5-1.5B nếu RAM < 8GB, Llama-3.2-3B nếu RAM ~16GB, Qwen2.5-7B nếu RAM >= 32GB)
   - Kết quả được ghi vào file `hardware.json` ở thư mục gốc

2. **Cài đặt môi trường Python** — Script setup (`.sh` / `.ps1`) thực hiện:
   - Tạo virtualenv `.venv/` ở thư mục gốc
   - Cài `requirements.txt` (huggingface_hub, numpy, ...)
   - Biên dịch `llama-cpp-python` với backend phù hợp:
     - **NVIDIA GPU** → `GGML_CUDA=on` (tăng tốc CUDA)
     - **Apple Silicon** → `GGML_METAL=on` (tăng tốc Metal)
     - **Intel/AMD GPU** → `GGML_VULKAN=on` (tăng tốc Vulkan)
     - **CPU thuần** → wheel mặc định
   
3. **Tải mô hình** — Script `download-model.py`:
   - Đọc `hardware.json` để biết mô hình được đề xuất
   - Tải về **hai phiên bản lượng tử hóa** của cùng một mô hình:
     - `Q4_K_M` — bản chính, cân bằng giữa chất lượng và hiệu năng
     - `Q2_K` — bản so sánh, nhẹ hơn nhưng chất lượng thấp hơn
   - file `.gguf` được lưu trong thư mục `models/`
   - Ghi đường dẫn vào `models/active.json`

### Đầu ra quan trọng

| File | Vai trò |
|---|---|
| `hardware.json` | Thông tin phần cứng — được các track khác đọc để điều chỉnh hành vi |
| `models/<tên_file>.gguf` | File mô hình đã lượng tử hóa (Q4_K_M và Q2_K) |
| `models/active.json` | Đường dẫn đến các file .gguf đã tải |

### Tuỳ chỉnh

Bạn có thể ghi đè phát hiện tự động bằng biến môi trường:
- `LLAMA_CUDA=1` — Buộc dùng CUDA (Linux/Windows + NVIDIA)
- `LLAMA_VULKAN=1` — Buộc dùng Vulkan
- `PYTHON=python3.11` — Chọn interpreter Python cụ thể

Nếu không truy cập được Hugging Face, xem `MANUAL-DOWNLOAD.md` để tải thủ công.

## Liên kết với các folder khác

Sau khi hoàn tất setup, bạn có thể chạy:
```bash
cd 01-llama-cpp-quickstart  # Chạy benchmark cơ bản
cd 02-llama-cpp-server      # Chạy server OpenAI-compat
cd 03-milestone-integration  # Tích hợp RAG pipeline
```
