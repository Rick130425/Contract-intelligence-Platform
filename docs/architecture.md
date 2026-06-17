# Architecture

Contract Intelligence Platform is a two-service application for processing and querying real estate lease contracts. The frontend provides a Streamlit user interface, while the backend owns document ingestion, persistence, vector indexing, semantic retrieval, and RAG responses.

## System Context

```text
User
  |
  v
Streamlit Frontend
  |
  | HTTP
  v
FastAPI Backend
  |
  +--> SQLite
  +--> Uploaded file storage
  +--> ChromaDB
  +--> OpenAI models
```

The frontend never reads the database, file storage, or vector store directly. It communicates only with the backend API.

## Services

### Frontend

Location:

```text
frontend/frontend/streamlit_app.py
```

Responsibilities:

- Upload contract files to the backend.
- Display processed contract metadata.
- Send semantic search requests.
- Send global or contract-specific chat questions.
- Request expiring-contract reports.

Runtime:

- Streamlit on port `8501`.
- Configured with `BACKEND_URL`.

### Backend

Location:

```text
backend/app/
```

Responsibilities:

- Expose HTTP APIs with FastAPI.
- Validate uploads.
- Store source files.
- Extract contract text into Markdown.
- Extract structured metadata using an LLM.
- Persist contracts and chunks in SQLite.
- Create embeddings and store vectors in ChromaDB.
- Retrieve relevant chunks for search and RAG chat.

Runtime:

- FastAPI on port `8000`.
- Configured through environment variables in `app/config.py`.

## Backend Module Layout

```text
app/
+-- api/routes/
|   +-- contracts.py
|   +-- search.py
|   +-- chat.py
+-- database/
|   +-- models.py
|   +-- session.py
|   +-- repositories/
+-- prompts/
|   +-- metadata_extraction_prompt.txt
|   +-- rag_answer_prompt.txt
+-- schemas/
+-- services/
|   +-- document_processing/
|   +-- ingestion_service.py
|   +-- metadata_extraction_service.py
|   +-- chunking_service.py
|   +-- vector_store_service.py
|   +-- rag_service.py
+-- config.py
+-- dependencies.py
+-- main.py
```

## Request Flow

### Upload and Ingestion

```text
POST /contracts/upload
  |
  v
IngestionService
  |
  +--> FileValidationService
  +--> FileStorageService
  +--> DocumentExtractionService
  +--> MetadataExtractionService
  +--> ContractRepository
  +--> LangChainDocumentService
  +--> ChunkingService
  +--> ChunkRepository
  +--> VectorStoreService
```

The upload endpoint completes ingestion synchronously. A successful response means the contract has been processed, chunked, embedded, indexed, and marked as `indexed`.

### Semantic Search

```text
POST /search/semantic
  |
  v
VectorStoreService.semantic_search
  |
  v
ChromaDB similarity_search_with_score
```

Search can run globally or be filtered by `contract_id`.

### Chat

```text
POST /chat/global
POST /chat/contract/{contract_id}
  |
  v
RagService.answer
  |
  +--> VectorStoreService.semantic_search
  +--> Prompt assembly
  +--> OpenAI response
  +--> Answer with sources
```

The RAG service asks the model to answer using only retrieved context and returns source metadata for each retrieved chunk.

### Reports

```text
GET /contracts/reports/expiring?days=90
  |
  v
ContractRepository.get_expiring_contracts
  |
  v
SQLite query on contracts.end_date
```

The report returns contracts whose `end_date` is between the current UTC date and the selected number of days.

## Persistence Architecture

Runtime data is stored in one data directory.

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

### SQLite

SQLite stores:

- Contract records.
- Extracted Markdown.
- Extracted structured metadata.
- Ingestion status and error state.
- Chunk records and Chroma IDs.

Tables are created automatically on backend startup.

### Uploaded Files

Uploaded files are saved with generated UUID-based filenames. The original filename is preserved in SQLite.

### ChromaDB

ChromaDB stores:

- Embedded chunk content.
- Chunk metadata.
- Vector IDs in the format `contract-{contract_id}-chunk-{chunk_index}`.

The SQLite `contract_chunks.chroma_id` column stores the corresponding vector ID.

## LLM Architecture

The backend uses the OpenAI API in three places:

- Metadata extraction in `MetadataExtractionService`.
- Embedding generation in `VectorStoreService`.
- RAG answer generation in `RagService`.

Configured models:

| Purpose | Environment variable | Default |
| --- | --- | --- |
| Metadata extraction | `OPENAI_MODEL` | `gpt-4.1-mini` |
| RAG answers | `OPENAI_MODEL` | `gpt-4.1-mini` |
| Embeddings | `EMBEDDING_MODEL` | `text-embedding-3-small` |

Metadata extraction uses structured parsing into the `ContractMetadata` Pydantic schema.

## API Surface

```text
GET  /
GET  /health

POST /contracts/upload
GET  /contracts
GET  /contracts/{contract_id}
GET  /contracts/reports/expiring

POST /search/semantic

POST /chat/global
POST /chat/contract/{contract_id}
```

## Contract Lifecycle

```text
uploaded
  |
  v
text_extracted
  |
  v
metadata_extracted
  |
  v
indexed
```

Any ingestion failure moves the contract to:

```text
failed
```

The backend also stores:

- `failed_step`
- `error_message`

## Design Decisions

### Markdown as the Intermediate Format

All supported document types are normalized into Markdown before metadata extraction, chunking, and retrieval. This gives the pipeline one consistent text representation.

### SQLite for the MVP

SQLite keeps setup simple and works well for a local or single-instance portfolio deployment. It should be replaced with a server database before multi-user, multi-replica production use.

### ChromaDB for Local Vector Persistence

ChromaDB provides a local persistent vector store that fits the MVP deployment model. The backend should not be horizontally scaled while using a single local ChromaDB directory.

### Synchronous Ingestion

The upload endpoint processes the full document before responding. This keeps the architecture simple, but large documents can make upload requests slow. A production system should move ingestion to a background job queue.

## Current Limitations

- No OCR for scanned PDFs.
- No authentication or authorization.
- No database migrations.
- No background jobs.
- No multi-replica backend support with the current SQLite and local ChromaDB setup.
- No legal advice or legal review guarantees.
