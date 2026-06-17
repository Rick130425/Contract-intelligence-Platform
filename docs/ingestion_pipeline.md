# Ingestion Pipeline

The ingestion pipeline turns an uploaded lease contract into searchable structured data and vectorized chunks.

Entry point:

```http
POST /contracts/upload
```

Main orchestrator:

```text
backend/app/services/ingestion_service.py
```

The endpoint processes the full file synchronously. A successful upload response means the contract has already been stored, parsed, indexed, and marked as `indexed`.

## High-Level Flow

```text
UploadFile
  |
  v
Validate file
  |
  v
Calculate SHA-256 hash
  |
  v
Check duplicates
  |
  v
Store file
  |
  v
Create contract row
  |
  v
Extract Markdown
  |
  v
Extract metadata
  |
  v
Persist metadata
  |
  v
Build LangChain documents
  |
  v
Split chunks
  |
  v
Persist chunks
  |
  v
Index vectors in ChromaDB
  |
  v
Mark contract indexed
```

## Pipeline Steps

### 1. Validate File

Service:

```text
FileValidationService
```

Validation rules:

- Filename is required.
- Allowed extensions are `.pdf`, `.docx`, and `.txt`.
- File must not be empty.
- File size must not exceed `MAX_UPLOAD_SIZE_MB`.

The file is read in 1 MB chunks to enforce the size limit without trusting client metadata.

Failure behavior:

- Raises `ValueError`.
- API returns HTTP `400`.
- No contract row is created.

### 2. Calculate File Hash

Utility:

```text
calculate_sha256
```

The backend calculates a SHA-256 hash from the uploaded file bytes. This hash is stored as `contracts.file_hash`.

Purpose:

- Detect duplicate uploads.
- Avoid processing the same exact file more than once.

### 3. Deduplicate Upload

Repository method:

```text
ContractRepository.get_by_hash
```

If a contract with the same `file_hash` already exists, the ingestion service returns the existing contract record immediately.

This means duplicate files are not:

- Stored again.
- Re-extracted.
- Re-embedded.
- Re-indexed.

### 4. Store Source File

Service:

```text
FileStorageService
```

The original filename is sanitized, but the stored filename is generated with a UUID and the original extension.

Stored path:

```text
UPLOAD_DIR/{uuid}.{extension}
```

Local default:

```text
backend/data/uploads/
```

Docker default:

```text
/app/data/uploads/
```

### 5. Create Contract Record

Repository method:

```text
ContractRepository.create_uploaded
```

The initial contract row includes:

- Original filename.
- Stored filename.
- Stored file path.
- File type.
- File hash.
- Status `uploaded`.

### 6. Extract Document Content

Service:

```text
DocumentExtractionService
```

The service delegates by file extension:

| Extension | Extractor | Engine |
| --- | --- | --- |
| `.pdf` | `PdfExtractor` | PyMuPDF4LLM |
| `.docx` | `DocxExtractor` | Docling |
| `.txt` | `TxtExtractor` | Native Python reader |

Output schema:

```text
DocumentConversionResult
```

Fields:

- `full_markdown`
- `units`
- `source_type`
- `extraction_engine`

Each unit contains text and optional source-location metadata.

Failure step:

```text
document_extraction
```

### 7. Normalize Markdown

Service:

```text
MarkdownNormalizationService
```

Each extractor normalizes raw extracted text before returning it to the ingestion pipeline.

The normalized Markdown becomes the canonical text representation used for:

- Metadata extraction.
- SQLite storage.
- LangChain document generation.
- Chunking and vector indexing.

### 8. Persist Extracted Markdown

Repository method:

```text
ContractRepository.save_extracted_markdown
```

Updates:

- `contracts.extracted_markdown`
- `contracts.status = text_extracted`

Failure step:

```text
markdown_persistence
```

### 9. Extract Structured Metadata

Service:

```text
MetadataExtractionService
```

Prompt:

```text
app/prompts/metadata_extraction_prompt.txt
```

Model:

```text
OPENAI_MODEL
```

Default:

```text
gpt-4.1-mini
```

The service sends up to the first 50,000 characters of extracted Markdown to the model.

Output schema:

```text
ContractMetadata
```

Fields:

- `title`
- `contract_type`
- `landlord`
- `tenant`
- `property_address`
- `start_date`
- `end_date`
- `monthly_rent`
- `currency`
- `security_deposit`
- `renewal_notice_days`
- `early_termination_notice_days`
- `risk_level`

Failure step:

```text
metadata_extraction
```

### 10. Persist Metadata

Repository method:

```text
ContractRepository.save_metadata
```

The repository maps the parsed `ContractMetadata` object into the `contracts` table.

Date strings are parsed with:

```text
date.fromisoformat
```

Updates:

- Structured metadata fields.
- `contracts.status = metadata_extracted`.

Failure step:

```text
metadata_persistence
```

### 11. Build LangChain Documents

Service:

```text
LangChainDocumentService
```

The service converts each `DocumentUnit` into a LangChain `Document`.

Base metadata:

- `contract_id`
- `source_file`
- `stored_file_path`
- `file_type`
- `source_type`
- `extraction_engine`
- `unit_index`
- `contract_type`
- `tenant`
- `landlord`
- `end_date`

Optional metadata:

- `page_number`
- `paragraph_index`
- `line_start`
- `line_end`
- `section_title`

### 12. Split Chunks

Service:

```text
ChunkingService
```

Splitter:

```text
RecursiveCharacterTextSplitter
```

Settings:

```text
chunk_size = 1200
chunk_overlap = 150
```

Separators:

```text
"\n\n", "\n", ". ", "; ", ", ", " ", ""
```

The service strips empty chunks and assigns `chunk_index`.

Failure step:

```text
chunking
```

### 13. Persist Chunks

Repository method:

```text
ChunkRepository.save_chunks
```

Chunks are saved in `contract_chunks`.

Each chunk receives a Chroma-compatible ID:

```text
contract-{contract_id}-chunk-{chunk_index}
```

Failure step:

```text
chunk_persistence
```

### 14. Index Vectors

Service:

```text
VectorStoreService
```

Embedding model:

```text
EMBEDDING_MODEL
```

Default:

```text
text-embedding-3-small
```

Vector store:

```text
Chroma
```

Default collection:

```text
contracts
```

The service adds all chunks to ChromaDB using the same IDs stored in SQLite.

Failure step:

```text
vector_indexing
```

### 15. Mark Contract Indexed

Repository method:

```text
ContractRepository.mark_indexed
```

Updates:

- `contracts.status = indexed`
- `contracts.failed_step = null`
- `contracts.error_message = null`

At this point the contract can be used by:

- `GET /contracts`
- `GET /contracts/{contract_id}`
- `POST /search/semantic`
- `POST /chat/global`
- `POST /chat/contract/{contract_id}`
- `GET /contracts/reports/expiring`

## Source Location Strategy

The pipeline preserves source location metadata differently by file type.

| File type | Unit strategy | Retrieval source metadata |
| --- | --- | --- |
| PDF | One unit per extracted page | `page_number` |
| DOCX | Markdown split into blocks | `paragraph_index` |
| TXT | Blocks of up to 30 lines | `line_start`, `line_end` |

This metadata is returned in RAG sources and semantic search results.

## Failure Handling

After the initial contract row is created, pipeline errors are recorded on the contract.

Stored fields:

```text
status = failed
failed_step = <pipeline_step>
error_message = <exception message>
```

Known `failed_step` values:

```text
document_extraction
markdown_persistence
metadata_extraction
metadata_persistence
chunking
chunk_persistence
vector_indexing
```

Upload validation errors happen before contract creation and return HTTP `400`.

Unexpected errors return HTTP `500`.

## Search After Ingestion

Semantic search uses:

```text
VectorStoreService.semantic_search
```

The method calls ChromaDB similarity search and returns:

- Chunk content.
- Chunk metadata.
- Similarity score.

Optional contract filter:

```json
{
  "contract_id": 1
}
```

When `contract_id` is provided, ChromaDB receives a metadata filter:

```json
{
  "contract_id": 1
}
```

## RAG After Ingestion

RAG chat uses:

```text
RagService.answer
```

Flow:

```text
Question
  -> semantic search
  -> context construction
  -> RAG prompt
  -> OpenAI response
  -> answer plus sources
```

Prompt:

```text
app/prompts/rag_answer_prompt.txt
```

The prompt instructs the model to:

- Use only provided context.
- Avoid inventing information.
- Say when information was not found.
- Include sources.
- Avoid legal advice.

## Operational Notes

- The pipeline is synchronous, so large files can make upload requests slow.
- Scanned PDFs are rejected because OCR is not implemented.
- Extraction quality depends on the source document quality.
- Metadata extraction depends on the first 50,000 characters of Markdown.
- A contract is ready for retrieval only when its status is `indexed`.
- Runtime data must persist across backend restarts.
