# LocCV - AI Resume Screening System (Docker + FastAPI + Ollama)

LocCV is an AI-powered CV screening system that supports:

- Uploading one or multiple CV files (`.pdf`, `.docx`)
- Scoring candidate-job fit against a JD
- Ranking candidates
- Storing screening history in SQLite
- Extracting clickable project hyperlinks from PDF annotations

The current setup is optimized for small VPS instances (2 CPU / 4 GB RAM) using batch `lite` mode.

---

## 1) Architecture

- `FastAPI backend`: parsing, analysis, scoring, persistence.
- `Static frontend`: quick testing UI at `http://localhost:8000`.
- `Ollama`: embeddings via `mxbai-embed-large`.
- `SQLite`: local storage at `data/screening.db`.

### Processing flow

1. Upload CV(s) + JD text.
2. Parse CV content:
   - PDF via `PyMuPDF`
   - DOCX via `python-docx`
3. Extract candidate signals:
   - name, email, skills, years of experience
   - project hyperlinks (PDF annotation first)
   - project snippets
4. Parse JD (must-have, nice-to-have, min years, language hint).
5. Compute scores (semantic + rules + project evidence).
6. Return response and persist result.

---

## 2) Project structure

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

## 3) Setup and run

### Requirements

- Docker Desktop (Windows) or Docker Engine (Linux)
- Internet access to pull images/models

### Run

```powershell
cd F:\hrm
docker compose up -d --build
docker exec -it ollama ollama pull mxbai-embed-large
```

Health check:

```powershell
curl.exe http://localhost:8000/health
```

Open UI:

- `http://localhost:8000`

---

## 4) Main APIs

### `POST /screen`

Screen one CV.

Form fields:

- `file`: one `.pdf` or `.docx`
- `jd_text`: job description text

### `POST /screen/batch`

Screen multiple CVs and rank them.

Form fields:

- `files`: multiple `.pdf/.docx`
- `jd_text`: job description text
- `top_k`: number of candidates returned
- `analysis_mode`: `lite` (default) or `full`
- `embedding_budget`: embedding budget for `lite` mode (default 32)

### `GET /results`

List stored screening results.

### `GET /jobs`

List stored jobs/JDs.

### `GET /jobs/{job_id}/ranking`

Get ranking for a specific job.

---

## 5) LocCV scoring logic

### 5.1 Score components

- `semantic_score`: embedding similarity between CV and JD
- `must_have_score`: required skill match ratio
- `nice_score`: preferred skill match ratio
- `exp_score`: experience requirement satisfaction
- `project_score`: project/link relevance score (if evidence exists)

### 5.2 Base weights

- `semantic`: `0.55`
- `must`: `0.30`
- `nice`: `0.10`
- `exp`: `0.05`
- `project`: `0.12`

LocCV uses **dynamic weighting**:

- Only active components (with real constraints/evidence) are included.
- Formula:

`final_score = weighted_sum / total_active_weight`

Examples:

- If JD has no `nice_to_have` and no `min_years`, those components are excluded.
- If `project_score` exists, project weight is included.

### 5.3 Recommendation label

- `>= 0.8`: `strong_fit`
- `>= 0.6`: `medium_fit`
- `< 0.6`: `weak_fit`

---

## 6) Batch optimization for 2CPU / 4GB RAM

### `analysis_mode=lite` (recommended)

Two-stage pipeline:

1. **Prefilter all CVs** (lightweight):
   - lexical overlap + rule-fit
2. **Budgeted embedding**:
   - embed only top candidates based on prefilter score

Benefits:

- Lower memory/CPU spikes
- Better throughput for <100 CV/request
- Stable ranking quality

### `analysis_mode=full`

- Deep analysis for each CV
- More expensive in compute/memory
- Better for small batches or stronger machines

### Practical settings for small VPS

- `analysis_mode=lite`
- `embedding_budget=16` to `32`
- Keep each request under ~100 CV files

---

## 7) Project hyperlink extraction behavior

### PDF

- Uses strict PDF annotation links (hover/click links in viewer)
- Modes:
  - `pdf_annotation_strict` (single/full)
  - `pdf_annotation_strict_lite` (batch lite)

### DOCX

- Falls back to text URL extraction (`detection_mode=text_url`)

### Link fields in response

- `url`, `source`, `page`, `rect`
- `reachable`, `status_code`, `title`, `description`, `relevance_score` (deep mode)

Note:

- In `lite` mode, deep URL checks may be skipped (`reachable` can be `null`).

---

## 8) Important environment variables

- `OLLAMA_URL` (default: `http://ollama:11434`)
- `EMBED_MODEL` (default: `mxbai-embed-large`)
- `DATA_DIR` (default: `/data`)
- `BATCH_EMBED_CHUNK_SIZE` (default: `16`)
- `BATCH_EMBED_BUDGET_DEFAULT` (default: `32`)

---

## 9) Operational commands

### Logs

```powershell
docker compose logs -f
docker compose logs backend --tail=200
```

### Reset database only

```powershell
docker compose down
Remove-Item -Force .\data\screening.db
docker compose up -d --build
```

### Remove all volumes (including pulled models)

```powershell
docker compose down -v
```

### Verify Ollama models

```powershell
docker exec -it ollama ollama list
```

---

## 10) Quick troubleshooting

### `{"detail":"Internal Server Error"}`

Usually related to embedding service/runtime.

Check:

```powershell
docker exec -it ollama ollama list
docker compose logs backend --tail=200
```

Re-pull model if needed:

```powershell
docker exec -it ollama ollama pull mxbai-embed-large
```

### Frontend does not refresh after changes

- Hard refresh browser: `Ctrl + F5`

### PDF/DOCX upload issues

- Only `.pdf` and `.docx` are supported
- Image-only scanned PDFs (no OCR) reduce parsing quality

---

## 11) Quick curl examples

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
