# LocCV – Hệ thống lọc CV bằng AI

LocCV là hệ thống giúp sàng lọc và xếp hạng ứng viên tự động dựa trên mức độ phù hợp giữa CV và Job Description (JD).

## Mô tả dự án

### Dự án làm gì?

LocCV cung cấp:

- Upload 1 hoặc nhiều CV (`.pdf`, `.docx`)
- Chấm điểm phù hợp giữa CV và JD
- Xếp hạng ứng viên theo điểm giảm dần
- Phân tích kỹ năng, kinh nghiệm, dự án, và link sản phẩm
- Lưu lịch sử kết quả vào SQLite để tra cứu lại

### Vì sao dùng các công nghệ hiện tại?

- **FastAPI**: nhẹ, nhanh, phù hợp API xử lý file và scoring.
- **Ollama + mxbai-embed-large**: chạy embedding cục bộ, không phụ thuộc cloud.
- **SQLite**: đơn giản, đủ dùng cho MVP và môi trường local/VPS nhỏ.
- **Docker Compose**: dựng toàn bộ stack nhanh, dễ triển khai.

### Thách thức đã gặp

- CV PDF bố cục phức tạp gây khó trích xuất tên/thông tin chuẩn.
- Link trong CV dễ bị nhiễu nếu chỉ parse text (đã ưu tiên PDF annotation).
- VPS 2 CPU / 4 GB RAM cần chiến lược batch tiết kiệm tài nguyên.

### Kế hoạch mở rộng

- OCR cho PDF scan ảnh.
- Dashboard thống kê và export báo cáo (CSV/Excel).
- Hàng đợi nền (Celery/RQ) để xử lý batch lớn ổn định hơn.

---

## Mục lục

1. [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
2. [Cài đặt và chạy](#cài-đặt-và-chạy)
3. [Cách sử dụng dự án](#cách-sử-dụng-dự-án)
4. [Cách tính điểm LocCV](#cách-tính-điểm-loccv)
5. [Cấu trúc dự án](#cấu-trúc-dự-án)
6. [Biến môi trường](#biến-môi-trường)
7. [Credits](#credits)
8. [Giấy phép](#giấy-phép)

---

## Yêu cầu hệ thống

- Docker Desktop (Windows) hoặc Docker Engine (Linux)
- Có internet để pull image/model lần đầu
- RAM khuyến nghị tối thiểu: 4 GB

---

## Cài đặt và chạy

### 1) Clone/đặt source code

```powershell
cd F:\hrm
```

### 2) Build và chạy container

```powershell
docker compose up -d --build
```

### 3) Pull model embedding

```powershell
docker exec -it ollama ollama pull mxbai-embed-large
```

### 4) Kiểm tra health

```powershell
curl.exe http://localhost:8000/health
```

Kỳ vọng: `{"status":"ok"}`

### 5) Truy cập giao diện

- `http://localhost:8000`

---

## Cách sử dụng dự án

### A. Dùng qua giao diện (khuyến nghị)

1. Mở `http://localhost:8000`
2. Chọn 1 hoặc nhiều file CV (`.pdf`, `.docx`)
3. Dán JD vào ô nhập
4. Chọn `Top K` (khi upload nhiều CV)
5. Bấm **Chấm điểm phù hợp**

Kết quả sẽ hiển thị:

- Điểm phù hợp tổng
- Nhãn đề xuất (`strong_fit`, `medium_fit`, `weak_fit`)
- Ranking (nếu batch)
- Link dự án trích xuất từ CV

### B. Dùng qua API

#### Single CV

```powershell
curl.exe -X POST "http://localhost:8000/screen" ^
  -F "file=@C:\path\cv.pdf" ^
  -F "jd_text=Backend Developer. Must have: JavaScript, SQL."
```

#### Batch CV

```powershell
curl.exe -X POST "http://localhost:8000/screen/batch" ^
  -F "files=@C:\path\cv1.pdf" ^
  -F "files=@C:\path\cv2.docx" ^
  -F "jd_text=Frontend Developer. Must have: React, JavaScript, SQL." ^
  -F "analysis_mode=lite" ^
  -F "embedding_budget=24" ^
  -F "top_k=10"
```

### C. Một số endpoint hữu ích

- `GET /results`: danh sách kết quả đã lưu
- `GET /jobs`: danh sách JD đã tạo
- `GET /jobs/{job_id}/ranking`: ranking theo job

### D. Gợi ý tài nguyên cho VPS 2CPU/4GB

- Dùng `analysis_mode=lite`
- `embedding_budget=16..32`
- Mỗi request nên dưới 100 CV

---

## Cách tính điểm LocCV

### Thành phần điểm

- `semantic_score`: tương đồng embedding CV–JD
- `must_have_score`: tỷ lệ match kỹ năng bắt buộc
- `nice_score`: tỷ lệ match kỹ năng ưu tiên
- `exp_score`: mức đáp ứng số năm kinh nghiệm
- `project_score`: mức liên quan dự án/link dự án (nếu có dữ liệu)

### Trọng số cơ sở

- semantic: `0.55`
- must: `0.30`
- nice: `0.10`
- exp: `0.05`
- project: `0.12`

LocCV dùng **trọng số động**:

- Chỉ thành phần có dữ liệu/ràng buộc mới được đưa vào mẫu số.
- Công thức tổng quát:

`final_score = weighted_sum / total_active_weight`

### Nhãn đề xuất

- `>= 0.8`: `strong_fit`
- `>= 0.6`: `medium_fit`
- `< 0.6`: `weak_fit`

---

## Cấu trúc dự án

```text
F:\hrm
|-- app
|   |-- main.py
|   |-- skills.json
|   |-- requirements.txt
|   `-- static
|       |-- index.html
|       `-- main.js
|-- data
|   `-- screening.db
|-- Dockerfile
|-- compose.yaml
|-- Readme.md
`-- README_EN.md
```

---

## Biến môi trường

- `OLLAMA_URL` (mặc định: `http://ollama:11434`)
- `EMBED_MODEL` (mặc định: `mxbai-embed-large`)
- `DATA_DIR` (mặc định: `/data`)
- `BATCH_EMBED_CHUNK_SIZE` (mặc định: `16`)
- `BATCH_EMBED_BUDGET_DEFAULT` (mặc định: `32`)

---

## Credits

### Tác giả/chủ dự án

- **Vũ Hozang**

### Công cụ và tài liệu tham khảo

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Ollama Documentation](https://ollama.com/)
- [PyMuPDF Documentation](https://pymupdf.readthedocs.io/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## Giấy phép

Hiện tại dự án **chưa đính kèm file LICENSE chính thức**.

Khuyến nghị:

- Nếu muốn mở cho cộng đồng dùng rộng rãi: dùng `MIT` hoặc `Apache-2.0`
- Nếu muốn ràng buộc copyleft mạnh: dùng `GPL-3.0`

Tham khảo chọn license: <https://choosealicense.com/>
