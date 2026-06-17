# Contract Intelligence Platform

Contract Intelligence Platform is an AI-powered application for uploading, processing, searching, and asking questions about real estate lease contracts.

The platform converts PDF, DOCX, and TXT contracts into normalized Markdown, extracts structured lease metadata with an LLM, stores operational data in SQLite, indexes contract chunks in ChromaDB, and exposes the workflow through a FastAPI backend and a Streamlit frontend.

> This project is for educational and portfolio purposes only. It does not provide legal advice.

## What It Does

- Upload lease contracts in PDF, DOCX, or TXT format.
- Extract readable contract text into Markdown.
- Extract structured metadata such as parties, dates, rent, deposits, notice periods, and risk level.
- Store contract records and chunk references in SQLite.
- Split contracts into retrievable chunks.
- Generate embeddings and persist them in ChromaDB.
- Search contracts semantically.
- Ask contract questions through a RAG workflow.
- Generate an expiring-contracts report.

## Repository Layout

```text
contract-intelligence-platform/
+-- backend/
|   +-- app/
|   +-- data/
|   +-- Dockerfile
|   +-- pyproject.toml
|   +-- uv.lock
|   +-- README.md
+-- frontend/
|   +-- frontend/
|   +-- Dockerfile
|   +-- pyproject.toml
|   +-- uv.lock
|   +-- README.md
+-- docs/
|   +-- architecture.md
|   +-- dataset.md
|   +-- deployment.md
|   +-- ingestion_pipeline.md
+-- docker-compose.yml
+-- .env.example
+-- .gitignore
+-- .dockerignore
+-- README.md
```

This root repository orchestrates the complete system. The backend and frontend can also be run independently from their own directories.

## Tech Stack

### Backend

- Python 3.13
- FastAPI
- SQLAlchemy
- SQLite
- Pydantic
- OpenAI Responses API
- LangChain
- ChromaDB
- PyMuPDF4LLM
- Docling
- uv

### Frontend

- Python 3.13
- Streamlit
- Requests
- Pandas
- uv

### Infrastructure

- Docker
- Docker Compose
- Persistent Docker volume for runtime data
- Dokploy-compatible deployment model

## Architecture Summary

```text
User
  |
  v
Streamlit Frontend
  |
  v
FastAPI Backend
  |
  +--> File validation and storage
  +--> Document extraction to Markdown
  +--> LLM metadata extraction
  +--> SQLite persistence
  +--> Chunking
  +--> Embedding generation
  +--> ChromaDB vector indexing
  |
  +--> Semantic search
  +--> RAG chat
  +--> Expiring contract reports
```

See [docs/architecture.md](docs/architecture.md) for the full design.

## Data Storage

The backend stores runtime data under one data directory.

Local development:

```text
backend/data/
+-- cip.db
+-- uploads/
+-- chroma/
```

Docker:

```text
/app/data/
+-- cip.db
+-- uploads/
+-- chroma/
```

- `cip.db` stores structured contract records and chunk references.
- `uploads/` stores the uploaded source documents.
- `chroma/` stores the persistent ChromaDB collection.

See [docs/dataset.md](docs/dataset.md) for details about persisted fields and generated data.

## Environment Variables

Create a root `.env` file for Docker Compose:

```bash
cp .env.example .env
```

Required:

```env
OPENAI_API_KEY=your_openai_api_key_here
```

Optional defaults:

```env
OPENAI_MODEL=gpt-4.1-mini
EMBEDDING_MODEL=text-embedding-3-small
```

The Docker Compose file passes these variables to the backend container.

## Run with Docker Compose

From the root repository:

```bash
docker compose up --build
```

Services:

```text
Backend API:  http://localhost:8000
API docs:     http://localhost:8000/docs
Frontend:     http://localhost:8501
```

Stop containers:

```bash
docker compose down
```

Remove containers and persistent volume:

```bash
docker compose down -v
```

Use `-v` only when you intentionally want to delete the SQLite database, uploaded files, and ChromaDB data.

## Run Without Docker

Backend:

```bash
cd backend
cp .env.example .env
uv sync
uv run uvicorn app.main:app --reload
```

Frontend:

```bash
cd frontend
uv sync
uv run streamlit run frontend/streamlit_app.py
```

The frontend defaults to `http://localhost:8000`. To point it to another backend:

```bash
BACKEND_URL=http://localhost:8000 uv run streamlit run frontend/streamlit_app.py
```

Default URLs:

```text
Backend: http://localhost:8000
Frontend: http://localhost:8501
```

## API Overview

Health:

```http
GET /
GET /health
```

Contracts:

```http
POST /contracts/upload
GET  /contracts
GET  /contracts/{contract_id}
GET  /contracts/reports/expiring?days=90
```

Semantic search:

```http
POST /search/semantic
```

```json
{
  "query": "contracts with early termination penalties",
  "contract_id": null,
  "k": 5
}
```

Chat:

```http
POST /chat/global
POST /chat/contract/{contract_id}
```

```json
{
  "question": "What are the tenant's maintenance obligations?",
  "k": 5
}
```

## Ingestion Pipeline

When a contract is uploaded, the backend runs the full ingestion workflow before returning a successful response:

```text
1. Validate file extension and size.
2. Read the upload safely in chunks.
3. Calculate a SHA-256 file hash.
4. Return an existing contract if the file was already uploaded.
5. Store the uploaded file.
6. Create the contract record in SQLite.
7. Extract document content into Markdown.
8. Persist extracted Markdown.
9. Extract structured metadata with an LLM.
10. Validate metadata with Pydantic.
11. Persist metadata in SQLite.
12. Build LangChain documents.
13. Split content into chunks.
14. Persist chunks in SQLite.
15. Generate embeddings.
16. Index vectors in ChromaDB.
17. Mark the contract as indexed.
```

See [docs/ingestion_pipeline.md](docs/ingestion_pipeline.md) for the detailed pipeline.

## Contract Statuses

```text
uploaded
text_extracted
metadata_extracted
indexed
failed
```

If processing fails, the backend stores `failed_step` and `error_message` on the contract record.

## Deployment

The project can be deployed as two services:

- Backend: FastAPI API on port `8000`.
- Frontend: Streamlit app on port `8501`.

The backend must have persistent storage mounted at `/app/data`.

See [docs/deployment.md](docs/deployment.md) for Docker Compose and Dokploy-oriented deployment notes.

## Limitations

- Scanned PDFs are not supported in the MVP because OCR is not implemented.
- Upload processing runs synchronously in the request lifecycle.
- SQLite is appropriate for this MVP, but not for multi-replica backend deployments.
- The application does not implement authentication or authorization.
- The application does not provide legal advice.

## Roadmap

- OCR for scanned PDFs.
- Authentication and role-based access.
- Background jobs for long-running ingestion.
- PostgreSQL support.
- Cloud object storage for uploaded files.
- Contract comparison workflows.
- Exportable reports.
- Evaluation scripts for extraction and RAG quality.

## Documentation

- [Backend README](backend/README.md)
- [Frontend README](frontend/README.md)
- [Architecture](docs/architecture.md)
- [Dataset](docs/dataset.md)
- [Deployment](docs/deployment.md)
- [Ingestion Pipeline](docs/ingestion_pipeline.md)

## Disclaimer

This project is for educational and workflow automation purposes only. It does not provide legal advice, legal recommendations, or legal review. Any contract interpretation should be reviewed by a qualified legal professional.
