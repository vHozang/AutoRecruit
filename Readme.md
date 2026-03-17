# LocCV – Hệ thống lọc CV bằng AI (Docker + FastAPI + Ollama)

LocCV là hệ thống lọc CV tự động theo Job Description (JD), hỗ trợ:

- Upload 1 hoặc nhiều CV (`.pdf`, `.docx`)
- Chấm điểm mức độ phù hợp CV với JD
- Xếp hạng ứng viên
- Lưu lịch sử vào SQLite
- Trích xuất hyperlink dự án trong PDF (link kiểu rê chuột/click được)

Hệ thống hiện tối ưu cho VPS nhỏ (2 CPU / 4 GB RAM) với chế độ batch `lite`.

---

## 1) Kiến trúc hệ thống

- `FastAPI backend`: parse CV, phân tích, chấm điểm, lưu dữ liệu.
- `Frontend static`: giao diện test nhanh, chạy tại `http://localhost:8000`.
- `Ollama`: sinh embedding bằng model `mxbai-embed-large`.
- `SQLite`: lưu kết quả trong `data/screening.db`.

### Luồng xử lý

1. Upload CV + nhập JD.
2. Parse nội dung CV:
   - PDF: `PyMuPDF`
   - DOCX: `python-docx`
3. Trích xuất thông tin:
   - Tên, email, kỹ năng, số năm kinh nghiệm
   - Link dự án (ưu tiên PDF annotation)
   - Đoạn mô tả dự án
4. Parse JD (must-have, nice-to-have, số năm yêu cầu, gợi ý ngôn ngữ).
5. Tính điểm (rule + semantic + project evidence).
6. Trả kết quả và lưu DB.

---

## 2) Cấu trúc thư mục

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

## 3) Cài đặt và chạy

### Yêu cầu

- Docker Desktop (Windows) hoặc Docker Engine (Linux)
- Có internet để pull image/model

### Chạy hệ thống

```powershell
cd F:\hrm
docker compose up -d --build
docker exec -it ollama ollama pull mxbai-embed-large
```

Kiểm tra health:

```powershell
curl.exe http://localhost:8000/health
```

Mở giao diện:

- `http://localhost:8000`

---

## 4) API chính

### `POST /screen`

Chấm 1 CV.

Form data:

- `file`: 1 file `.pdf` hoặc `.docx`
- `jd_text`: nội dung JD

### `POST /screen/batch`

Chấm nhiều CV + xếp hạng.

Form data:

- `files`: nhiều file `.pdf/.docx`
- `jd_text`: nội dung JD
- `top_k`: số ứng viên trả về
- `analysis_mode`: `lite` (mặc định) hoặc `full`
- `embedding_budget`: ngân sách embedding cho `lite` (mặc định 32)

### `GET /results`

Xem kết quả đã lưu.

### `GET /jobs`

Xem danh sách JD đã tạo.

### `GET /jobs/{job_id}/ranking`

Xem ranking theo từng job.

---

## 5) Cách tính điểm của LocCV

### 5.1 Thành phần điểm

- `semantic_score`: độ tương đồng embedding CV và JD
- `must_have_score`: tỷ lệ kỹ năng bắt buộc khớp
- `nice_score`: tỷ lệ kỹ năng ưu tiên khớp
- `exp_score`: mức đáp ứng số năm kinh nghiệm
- `project_score`: mức phù hợp dự án/link dự án với JD (nếu có)

### 5.2 Trọng số cơ sở

- `semantic`: `0.55`
- `must`: `0.30`
- `nice`: `0.10`
- `exp`: `0.05`
- `project`: `0.12`

Hệ thống dùng **trọng số động**:

- Chỉ thành phần nào có dữ liệu/ràng buộc thì mới tính vào mẫu số.
- Công thức:

`final_score = weighted_sum / total_active_weight`

Ví dụ:

- JD không có `nice_to_have` hoặc `min_years` thì 2 phần này bị loại khỏi mẫu số.
- Nếu có `project_score`, điểm dự án được cộng thêm theo trọng số `project`.

### 5.3 Nhãn đề xuất

- `>= 0.8`: `strong_fit`
- `>= 0.6`: `medium_fit`
- `< 0.6`: `weak_fit`

---

## 6) Chế độ batch tối ưu cho VPS 2CPU/4GB

### `analysis_mode=lite` (khuyên dùng)

Pipeline 2 tầng:

1. **Prefilter toàn bộ CV** (nhanh, nhẹ):
   - lexical overlap + rule-fit
2. **Embedding có ngân sách**:
   - chỉ embed top CV theo `embedding_budget`

Ưu điểm:

- Giảm peak RAM/CPU
- Giữ tốc độ tốt khi chạy dưới 100 CV/lần
- Vẫn có ranking ổn định

### `analysis_mode=full`

- Phân tích sâu từng CV
- Tốn tài nguyên hơn
- Phù hợp khi số CV ít hoặc máy mạnh

### Khuyến nghị cấu hình thực tế

Với VPS 2CPU/4GB:

- `analysis_mode=lite`
- `embedding_budget=16` đến `32`
- Tránh gửi >100 CV trong 1 request

---

## 7) Cơ chế đọc link dự án trong CV

### PDF

- Dùng `PDF annotation link` (link có thể hover/click trong viewer)
- `detection_mode`:
  - `pdf_annotation_strict` (single/full)
  - `pdf_annotation_strict_lite` (batch lite)

### DOCX

- Fallback bằng text URL (`detection_mode=text_url`)

### Dữ liệu trả về cho mỗi link

- `url`, `source`, `page`, `rect`
- `reachable`, `status_code`, `title`, `description`, `relevance_score` (ở mode sâu)

Lưu ý:

- Batch `lite` ưu tiên tốc độ nên có thể không kiểm tra sâu link (`reachable` có thể là `null`).

---

## 8) Biến môi trường quan trọng

- `OLLAMA_URL` (mặc định: `http://ollama:11434`)
- `EMBED_MODEL` (mặc định: `mxbai-embed-large`)
- `DATA_DIR` (mặc định: `/data`)
- `BATCH_EMBED_CHUNK_SIZE` (mặc định: `16`)
- `BATCH_EMBED_BUDGET_DEFAULT` (mặc định: `32`)

---

## 9) Lệnh vận hành thường dùng

### Xem logs

```powershell
docker compose logs -f
docker compose logs backend --tail=200
```

### Xóa sạch database

```powershell
docker compose down
Remove-Item -Force .\data\screening.db
docker compose up -d --build
```

### Xóa cả volume (mất luôn model)

```powershell
docker compose down -v
```

### Kiểm tra model Ollama

```powershell
docker exec -it ollama ollama list
```

---

## 10) Troubleshooting nhanh

### Lỗi `{"detail":"Internal Server Error"}`

Thường liên quan embedding service.

Kiểm tra:

```powershell
docker exec -it ollama ollama list
docker compose logs backend --tail=200
```

Pull lại model nếu cần:

```powershell
docker exec -it ollama ollama pull mxbai-embed-large
```

### FE không cập nhật sau khi sửa code

- Nhấn `Ctrl + F5` (hard refresh)

### Upload DOCX/PDF lỗi

- Chỉ hỗ trợ `.pdf` và `.docx`
- PDF scan ảnh không OCR sẽ làm chất lượng parse thấp

---

## 11) Ví dụ test nhanh bằng curl

### Single CV

```powershell
curl.exe -X POST "http://localhost:8000/screen" ^
  -F "file=@C:\path\to\cv.pdf" ^
  -F "jd_text=Backend Developer. Must have: JavaScript, SQL. Nice to have: Docker."
```

### Batch CV

```powershell
curl.exe -X POST "http://localhost:8000/screen/batch" ^
  -F "files=@C:\path\cv1.pdf" ^
  -F "files=@C:\path\cv2.docx" ^
  -F "jd_text=Backend Developer. Must have: JavaScript, SQL." ^
  -F "analysis_mode=lite" ^
  -F "embedding_budget=24" ^
  -F "top_k=10"
```
