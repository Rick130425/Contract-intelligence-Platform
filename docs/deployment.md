# Deployment

Contract Intelligence Platform is deployed as two services:

- `backend`: FastAPI API on port `8000`.
- `frontend`: Streamlit application on port `8501`.

The backend requires persistent storage for SQLite, uploaded files, and ChromaDB.

## Deployment Requirements

- Docker and Docker Compose for local full-stack deployment.
- An OpenAI API key.
- Persistent storage mounted to the backend at `/app/data` in containerized environments.
- Public domain or reverse proxy configuration for production-style deployments.

## Environment Variables

### Root Docker Compose

The root `.env` file is read by `docker-compose.yml`.

Required:

```env
OPENAI_API_KEY=your_openai_api_key_here
```

Optional:

```env
OPENAI_MODEL=gpt-4.1-mini
EMBEDDING_MODEL=text-embedding-3-small
```

### Backend

| Variable | Example | Description |
| --- | --- | --- |
| `ENVIRONMENT` | `production` | Runtime environment label. |
| `OPENAI_API_KEY` | `sk-...` | Required for metadata extraction, embeddings, and chat. |
| `OPENAI_MODEL` | `gpt-4.1-mini` | Model for metadata extraction and RAG answers. |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model. |
| `DATABASE_URL` | `sqlite:////app/data/cip.db` | SQLite database URL in Docker. |
| `DATA_DIR` | `/app/data` | Runtime data root. |
| `UPLOAD_DIR` | `/app/data/uploads` | Uploaded file storage. |
| `CHROMA_PERSIST_DIR` | `/app/data/chroma` | ChromaDB persistence directory. |
| `CHROMA_COLLECTION_NAME` | `contracts` | Chroma collection name. |
| `MAX_UPLOAD_SIZE_MB` | `10` | Upload size limit. |
| `BACKEND_CORS_ORIGINS` | `https://your-domain.com` | Comma-separated frontend origins. |

### Frontend

| Variable | Example | Description |
| --- | --- | --- |
| `BACKEND_URL` | `https://api.your-domain.com` | Public or internal backend base URL. |

## Local Docker Compose Deployment

From the root repository:

```bash
cp .env.example .env
docker compose up --build
```

Services:

```text
Backend API:  http://localhost:8000
API docs:     http://localhost:8000/docs
Frontend:     http://localhost:8501
```

Stop services:

```bash
docker compose down
```

Delete containers and persistent data:

```bash
docker compose down -v
```

Only use `-v` when you intentionally want to delete the database, uploaded files, and vector store.

## Docker Compose Storage

The root `docker-compose.yml` defines:

```text
cip_backend_data:/app/data
```

The backend stores:

```text
/app/data/cip.db
/app/data/uploads
/app/data/chroma
```

This volume must persist across rebuilds and redeployments.

## Health Checks

The backend exposes:

```http
GET /health
```

Expected response shape:

```json
{
  "status": "ok",
  "database": "sqlite",
  "environment": "development"
}
```

The Docker Compose backend service uses this endpoint as a health check. The frontend waits for the backend to become healthy before starting.

## Production-Style Deployment

Recommended service split:

```text
Frontend domain: https://your-domain.com
Backend domain:  https://api.your-domain.com
```

Backend settings:

```env
ENVIRONMENT=production
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-4.1-mini
EMBEDDING_MODEL=text-embedding-3-small
DATABASE_URL=sqlite:////app/data/cip.db
DATA_DIR=/app/data
UPLOAD_DIR=/app/data/uploads
CHROMA_PERSIST_DIR=/app/data/chroma
CHROMA_COLLECTION_NAME=contracts
MAX_UPLOAD_SIZE_MB=10
BACKEND_CORS_ORIGINS=https://your-domain.com
```

Frontend settings:

```env
BACKEND_URL=https://api.your-domain.com
```

## Dokploy Deployment Notes

This project can be deployed in Dokploy as two applications.

### Backend Application

Suggested settings:

```text
Name: cip-backend
Source: backend repository
Dockerfile: Dockerfile
Port: 8000
Domain: api.your-domain.com
Persistent volume: /app/data
```

Environment variables:

```env
ENVIRONMENT=production
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-4.1-mini
EMBEDDING_MODEL=text-embedding-3-small
DATABASE_URL=sqlite:////app/data/cip.db
DATA_DIR=/app/data
UPLOAD_DIR=/app/data/uploads
CHROMA_PERSIST_DIR=/app/data/chroma
CHROMA_COLLECTION_NAME=contracts
MAX_UPLOAD_SIZE_MB=10
BACKEND_CORS_ORIGINS=https://your-domain.com
```

### Frontend Application

Suggested settings:

```text
Name: cip-frontend
Source: frontend repository
Dockerfile: Dockerfile
Port: 8501
Domain: your-domain.com
```

Environment variables:

```env
BACKEND_URL=https://api.your-domain.com
```

## Reverse Proxy and CORS

The backend allows origins from `BACKEND_CORS_ORIGINS`.

For local development:

```env
BACKEND_CORS_ORIGINS=http://localhost:8501
```

For production:

```env
BACKEND_CORS_ORIGINS=https://your-domain.com
```

For multiple origins:

```env
BACKEND_CORS_ORIGINS=https://your-domain.com,https://www.your-domain.com
```

The frontend must use the public backend URL in `BACKEND_URL` unless both services are on the same private Docker network.

## Scaling Constraints

Do not run multiple backend replicas with the current storage design.

Reasons:

- SQLite is a local file database.
- ChromaDB is persisted to a local directory.
- Uploaded files are stored on local filesystem paths.
- There is no distributed lock, queue, or shared object storage layer.

Before horizontal scaling, replace or redesign:

- SQLite with PostgreSQL or another server database.
- Local uploads with object storage.
- Local ChromaDB persistence with a vector store deployment suitable for concurrent services.
- Synchronous ingestion with background jobs.

## Backups

Back up the backend data directory:

```text
/app/data
```

This includes:

- SQLite database.
- Uploaded source contracts.
- ChromaDB persistence files.

For consistent backups, stop the backend or use a deployment platform snapshot mechanism that can safely capture file-based databases and persistent volumes.

## Security Checklist

- Keep `OPENAI_API_KEY` in environment variables only.
- Do not commit `.env` files.
- Do not commit `backend/data`.
- Use HTTPS for public deployments.
- Restrict CORS to trusted frontend domains.
- Add authentication before exposing real contract data.
- Use authorized or synthetic contracts for demos.
- Review logs for accidental sensitive data exposure.

## Troubleshooting

### Backend container fails on startup

Check:

- `OPENAI_API_KEY` is set.
- `/app/data` is writable.
- The Docker image built successfully.
- Native dependencies required by document processing installed correctly.

### Frontend cannot reach backend

Check:

- `BACKEND_URL` is correct.
- Backend health check passes.
- CORS allows the frontend origin.
- Reverse proxy routes to port `8000`.

### Upload succeeds locally but fails in deployment

Check:

- Persistent volume exists and is mounted at `/app/data`.
- `UPLOAD_DIR` and `CHROMA_PERSIST_DIR` point inside `/app/data`.
- Upload size is below `MAX_UPLOAD_SIZE_MB`.
- The document has extractable text.

### Search or chat returns empty results

Check:

- Uploaded contract status is `indexed`.
- ChromaDB files exist in `/app/data/chroma`.
- The backend uses the same `CHROMA_COLLECTION_NAME` across restarts.
