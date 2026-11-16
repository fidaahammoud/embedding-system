# Installation & Usage

This project runs a local **n8n + Docling + PGVector** stack that powers RAG (Retrieval-Augmented Generation) workflows backed by **AWS Bedrock**.

```
                 ┌────────────────────── docker-compose ──────────────────────┐
  Browser ──────►│  n8n  :5678  ──HTTP──►  docling  :5001  (convert files)    │
  (form / chat)  │    │                                                       │
                 │    └──SQL──►  pgvector  :5432  (vector database)           │
                 └────┼───────────────────────────────────────────────────────┘
                      └──HTTPS──►  AWS Bedrock (embeddings + Claude chat model)
```

---

## 1. Components

### Containers (`docker-compose.yml`)

| Service | Image / build | Port | What it does |
|---|---|---|---|
| **n8n** | `n8nio/n8n` (pinned by digest) | `5678` | Workflow engine. Hosts the upload forms, the chat UI, and runs the RAG workflows. Data (workflows, credentials, executions) persists in the `n8n_data_1` volume. |
| **docling** | built from `Dockerfile.docling-asr` | `5001` | Document converter (**docling-serve**). Turns PDF / DOCX / XLSX / CSV / TXT / MD into clean Markdown (tables, headings, reading order). This build also adds **ffmpeg + Whisper** so it can transcribe audio. Web UI enabled at `http://localhost:5001/ui`. |
| **pgvector** | `pgvector/pgvector:pg16` | `5432` | PostgreSQL 16 with the `vector` extension. Stores text chunks + their embeddings and does similarity search. Data persists in the `pgvector_data` volume. |

n8n waits for pgvector to be healthy (`pg_isready -U n8n -d vectordb`) before starting.

### Project files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Defines and wires the three services above. |
| `.env` | Environment variables shared by n8n and pgvector (timezone, Postgres user/password/db). **Never commit it.** |
| `Dockerfile.docling-asr` | **Used by compose.** Extends the official `docling-serve-cpu` image with a static ffmpeg binary and `openai-whisper` (base model pre-downloaded) for audio transcription. |
| `resume-rag.json` | n8n workflow: **Resume RAG** (resumes → `resumes` table → recruiter chat). See [README.md](README.md). |

### External service: AWS Bedrock

| Model | Used for |
|---|---|
| `amazon.titan-embed-text-v2:0` | Turning text chunks and questions into vectors (embeddings). |
| `us.anthropic.claude-sonnet-4-6` | Chat model behind the AI Agent (cross-region inference profile, US regions). |

---

## 2. Prerequisites

- **Docker** and **docker-compose** v2 (`docker-compose version`)
- ~10 GB free disk (the docling image with ML models + Whisper is large)
- An **AWS account** with:
  - Bedrock **model access** granted (Bedrock console → *Model access*) for *Titan Text Embeddings V2* and *Claude Sonnet 4.6*
  - An IAM user/access key with at least `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream`
  - A **US region** (e.g. `us-east-1`), required by the `us.` inference profile

---

## 3. Installation

### 3.1 Configure `.env`

Create `.env` in the project root:

```env
# General
GENERIC_TIMEZONE=Europe/Paris

# PostgreSQL (for pgvector service only)
POSTGRES_USER=n8n
POSTGRES_PASSWORD=change-me
POSTGRES_DB=vectordb
```

> The healthcheck in `docker-compose.yml` expects user **`n8n`** and database **`vectordb`**. If you change them, update the healthcheck too.

### 3.2 Build and start

```bash
docker-compose up -d --build
```

The first build of `docling` takes several minutes (it downloads Docling models, ffmpeg and the Whisper model).

Check everything is up:

```bash
docker-compose ps
docker-compose logs -f docling      # wait for "Uvicorn running on http://0.0.0.0:5001"
```

| Check | URL |
|---|---|
| n8n editor | http://localhost:5678 |
| Docling UI | http://localhost:5001/ui |
| Docling API docs | http://localhost:5001/docs |

Quick Docling test:

```bash
curl -F "files=@some.pdf" http://localhost:5001/v1/convert/file | head -c 500
```

### 3.3 First n8n setup

1. Open http://localhost:5678 and create the owner account (first run only).
2. Go to **Credentials → Add credential** and create:

   **AWS (IAM) account** — type *AWS*
   | Field | Value |
   |---|---|
   | Region | `us-east-1` (or another US region) |
   | Access Key ID / Secret | your IAM key |

   **Postgres account** — type *Postgres*
   | Field | Value |
   |---|---|
   | Host | `pgvector` (the container name, **not** `localhost`) |
   | Port | `5432` |
   | Database | value of `POSTGRES_DB` (`vectordb`) |
   | User / Password | values of `POSTGRES_USER` / `POSTGRES_PASSWORD` |

   > Inside Docker, n8n reaches Docling at `http://docling:5001`. That URL is already set in the workflows.

### 3.4 Import the workflows

1. **Workflows → ⋮ → Import from File** and pick `resume-rag.json` (and/or `aws-example-1.json`).
2. Open every node with a red warning (*Embeddings AWS Bedrock*, *AWS Bedrock Chat Model*, *Insert Data to Store*, *Query Data Tool*) and select the credentials you created.
3. **Save**.

The vector tables (`resumes`, `documents`) are created automatically on the first insert.

---

## 4. How to use

### Resume RAG (`resume-rag.json`)

1. **Load resumes:** open the **Upload resumes** node → *Test step* (or activate the workflow and use the production form URL). Upload one or more PDF / DOCX / TXT / MD files.
   Name files after the candidate, e.g. `jane_doe.pdf`, because the filename becomes the candidate name.
2. **Ask questions:** click **Open chat** (bottom of the canvas) and ask, for example:
   - *Who has more than 5 years of Python experience?*
   - *Compare the candidates for a Senior DevOps role.*
   - *Summarize Jane Doe's education and certifications.*

### Generic RAG (`aws-example-1.json`)

Same idea: upload any document (PDF, DOCX, XLSX, CSV, TXT, MD, audio) through **Upload your file here**, then ask questions in the chat. Data goes to the `documents` table.

### Activating (production URLs)

Toggle the workflow **Active** in n8n. The form and chat then get stable production URLs (shown in the trigger nodes) that work without the editor open.

---

## 5. Maintenance

```bash
docker-compose logs -f n8n                     # follow logs
docker-compose restart docling                 # restart one service
docker-compose down                            # stop (data kept in volumes)
docker-compose down -v                         # stop AND delete all data (irreversible)
```

Inspect the vector store:

```bash
docker exec -it pgvector psql -U n8n -d vectordb
```
```sql
\dt                                                     -- list tables
SELECT count(*) FROM resumes;                           -- number of chunks
SELECT DISTINCT metadata->>'candidate' FROM resumes;    -- loaded candidates
DELETE FROM resumes WHERE metadata->>'source' = 'jane_doe.pdf';  -- remove one resume
TRUNCATE resumes;                                       -- wipe all resumes
```

---

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `Docling returned empty content` | The PDF is a scanned image. Set `do_ocr` to `true` in the **Docling API** node (slower). |
| Docling request times out | Large file / first run. The node timeout is 5 min, and `DOCLING_SERVE_MAX_SYNC_WAIT=600`. Check `docker-compose logs docling`. |
| `ECONNREFUSED` to Postgres | Credential host must be `pgvector`, not `localhost`. |
| Bedrock `AccessDeniedException` | Model access not granted in the Bedrock console, or IAM lacks `bedrock:InvokeModel`. |
| Bedrock `ValidationException` on Claude | Region isn't a US region (needed for `us.` inference profile). |
| Answers mix in unrelated docs | Make sure the Resume workflow uses table `resumes` and the generic one uses `documents`. |
| Poor / no search results | Insert and Query must use the **same** embedding model. If you change it, empty the table and re-ingest. |
