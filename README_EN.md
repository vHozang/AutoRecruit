# AutoRecruit - AI Resume Screening System

![Project](https://img.shields.io/badge/Project-AutoRecruit-0A66C2)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

AutoRecruit is an AI-based CV screening system that matches resumes against a Job Description (JD), scores candidate fit, and ranks applicants.

## Key features
- Upload one or multiple CV files (`.pdf`, `.docx`)
- CV-to-JD fit scoring
- Candidate ranking by score
- Skill/experience/project-link extraction
- SQLite-based result history

## Quick setup
Requirements:
- Docker Desktop (or Docker Engine)
- Internet access for initial model pull

Run:
```powershell
cd F:\AutoRecruit
docker compose up -d --build
docker exec -it ollama ollama pull mxbai-embed-large
```

Health check:
```powershell
curl.exe http://localhost:8000/health
```
Expected: `{"status":"ok"}`

Open UI:
- `http://localhost:8000`

## How to use
1. Open `http://localhost:8000`
2. Upload one or more CV files (`.pdf`, `.docx`)
3. Paste JD text
4. Click **Chấm điểm phù hợp**

Returned output includes:
- `final_score`
- recommendation label (`strong_fit`, `medium_fit`, `weak_fit`)
- ranking (batch mode)
- project/link analysis (when available)

## Basic APIs
### Screen single CV
```powershell
curl.exe -X POST "http://localhost:8000/screen" ^
  -F "file=@C:\path\cv.pdf" ^
  -F "jd_text=Backend Developer. Must have: JavaScript, SQL."
```

### Screen batch CVs
```powershell
curl.exe -X POST "http://localhost:8000/screen/batch" ^
  -F "files=@C:\path\cv1.pdf" ^
  -F "files=@C:\path\cv2.docx" ^
  -F "jd_text=Frontend Developer. Must have: React, JavaScript, SQL." ^
  -F "analysis_mode=lite" ^
  -F "embedding_budget=24" ^
  -F "top_k=10"
```

## Important notes
- Use `analysis_mode=lite` for small VPS (2 CPU / 4 GB RAM)
- Keep each batch request under ~100 CV files
- Image-only scanned PDFs (without OCR) may reduce extraction quality

## License
This project is licensed under the **MIT License**. See [LICENSE](./LICENSE).
