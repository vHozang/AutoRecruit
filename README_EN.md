# LocCV - AI Resume Screening System

LocCV is an AI-based screening system that ranks candidates by measuring CV-to-JD relevance.

## Project description

### What does this project do?

LocCV provides:

- Single and batch CV upload (`.pdf`, `.docx`)
- Candidate-job fit scoring
- Candidate ranking
- Skill/experience/project-link analysis
- SQLite result history for tracking and audit

### Why these technologies?

- **FastAPI**: lightweight and fast for API-first workflows.
- **Ollama + mxbai-embed-large**: local embedding inference, no cloud dependency.
- **SQLite**: simple and practical for MVP/local/small VPS.
- **Docker Compose**: reproducible setup and deployment.

### Challenges encountered

- Complex PDF layouts can degrade extraction quality.
- Plain text URL parsing causes false positives (fixed by prioritizing PDF annotations).
- Running batch screening on 2 CPU / 4 GB RAM requires resource-aware strategy.

### Planned improvements

- OCR for scanned PDFs.
- Reporting dashboard and CSV/Excel export.
- Background queue (Celery/RQ) for larger batch stability.

---

## Table of contents

1. [System requirements](#system-requirements)
2. [Installation and run](#installation-and-run)
3. [How to use](#how-to-use)
4. [LocCV scoring logic](#loccv-scoring-logic)
5. [Project structure](#project-structure)
6. [Environment variables](#environment-variables)
7. [Credits](#credits)
8. [License](#license)

---

## System requirements

- Docker Desktop (Windows) or Docker Engine (Linux)
- Internet access for initial image/model pull
- Recommended RAM: at least 4 GB

---

## Installation and run

### 1) Open project directory

```powershell
cd F:\hrm
```

### 2) Build and start services

```powershell
docker compose up -d --build
```

### 3) Pull embedding model

```powershell
docker exec -it ollama ollama pull mxbai-embed-large
```

### 4) Health check

```powershell
curl.exe http://localhost:8000/health
```

Expected: `{"status":"ok"}`

### 5) Open UI

- `http://localhost:8000`

---

## How to use

### A. Via UI (recommended)

1. Open `http://localhost:8000`
2. Upload one or more CV files (`.pdf`, `.docx`)
3. Paste JD text
4. Set `Top K` (for batch)
5. Click **Chấm điểm phù hợp**

You will see:

- Final fit score
- Recommendation label (`strong_fit`, `medium_fit`, `weak_fit`)
- Batch ranking
- Project links extracted from CV

### B. Via API

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

### C. Useful endpoints

- `GET /results`: stored screening records
- `GET /jobs`: stored JD list
- `GET /jobs/{job_id}/ranking`: ranking by job

### D. Recommended settings for 2CPU/4GB VPS

- Use `analysis_mode=lite`
- Set `embedding_budget=16..32`
- Keep each batch under ~100 CV files

---

## LocCV scoring logic

### Score components

- `semantic_score`: CV–JD embedding similarity
- `must_have_score`: required-skill match ratio
- `nice_score`: preferred-skill match ratio
- `exp_score`: experience requirement satisfaction
- `project_score`: project/link relevance (if evidence exists)

### Base weights

- semantic: `0.55`
- must: `0.30`
- nice: `0.10`
- exp: `0.05`
- project: `0.12`

LocCV uses **dynamic weighting**:

- Only active components are included in the denominator.
- Generic formula:

`final_score = weighted_sum / total_active_weight`

### Recommendation labels

- `>= 0.8`: `strong_fit`
- `>= 0.6`: `medium_fit`
- `< 0.6`: `weak_fit`

---

## Project structure

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

## Environment variables

- `OLLAMA_URL` (default: `http://ollama:11434`)
- `EMBED_MODEL` (default: `mxbai-embed-large`)
- `DATA_DIR` (default: `/data`)
- `BATCH_EMBED_CHUNK_SIZE` (default: `16`)
- `BATCH_EMBED_BUDGET_DEFAULT` (default: `32`)

---

## Credits

### Owner / Maintainer

- **Vũ Hozang**

### References

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Ollama Documentation](https://ollama.com/)
- [PyMuPDF Documentation](https://pymupdf.readthedocs.io/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## License

This project currently **does not include a final LICENSE file**.

Recommendation:

- Use `MIT` or `Apache-2.0` for permissive open usage
- Use `GPL-3.0` for stronger copyleft

License guide: <https://choosealicense.com/>
