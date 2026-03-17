# AutoRecruit – Hệ thống lọc CV bằng AI

![Project](https://img.shields.io/badge/Project-AutoRecruit-0A66C2)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

AutoRecruit là hệ thống giúp sàng lọc CV theo Job Description (JD), chấm điểm phù hợp và xếp hạng ứng viên.

## Tính năng chính

- Upload 1 hoặc nhiều CV (`.pdf`, `.docx`)
- Chấm điểm phù hợp CV–JD
- Xếp hạng ứng viên theo điểm giảm dần
- Trích xuất kỹ năng, kinh nghiệm, link dự án
- Lưu lịch sử kết quả bằng SQLite

## Cài đặt nhanh

Yêu cầu:

- Docker Desktop (hoặc Docker Engine)
- Internet để pull model lần đầu

Chạy:

```powershell
cd F:\AutoRecruit
docker compose up -d --build
docker exec -it ollama ollama pull mxbai-embed-large
```

Kiểm tra:

```powershell
curl.exe http://localhost:8000/health
```

Kỳ vọng: `{"status":"ok"}`

Mở giao diện:

- `http://localhost:8000`

## Cách sử dụng

1. Vào `http://localhost:8000`
2. Chọn 1 hoặc nhiều CV (`.pdf`, `.docx`)
3. Dán JD
4. Nhấn **Chấm điểm phù hợp**

Kết quả trả về gồm:

- `final_score`
- nhãn (`strong_fit`, `medium_fit`, `weak_fit`)
- ranking (nếu batch)
- phân tích link/dự án (nếu có)

## API cơ bản

### Chấm 1 CV

```powershell
curl.exe -X POST "http://localhost:8000/screen" ^
  -F "file=@C:\path\cv.pdf" ^
  -F "jd_text=Backend Developer. Must have: JavaScript, SQL."
```

### Chấm nhiều CV

```powershell
curl.exe -X POST "http://localhost:8000/screen/batch" ^
  -F "files=@C:\path\cv1.pdf" ^
  -F "files=@C:\path\cv2.docx" ^
  -F "jd_text=Frontend Developer. Must have: React, JavaScript, SQL." ^
  -F "analysis_mode=lite" ^
  -F "embedding_budget=24" ^
  -F "top_k=10"
```

## Lưu ý quan trọng

- Dùng `analysis_mode=lite` để tối ưu VPS nhỏ (2 CPU / 4 GB RAM)
- Mỗi request batch nên dưới 100 CV
- PDF scan ảnh (không OCR) có thể cho chất lượng parse thấp

## License

Dự án sử dụng **MIT License**. Xem file [LICENSE](./LICENSE).
