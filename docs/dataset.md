# Dataset

Contract Intelligence Platform does not ship with a fixed public dataset. The dataset is created at runtime from the lease contracts uploaded by the user.

The application stores three categories of data:

- Original uploaded files.
- Structured records in SQLite.
- Vectorized contract chunks in ChromaDB.

> Use synthetic, sample, public-domain, or properly authorized contracts for demos. Lease contracts can contain sensitive personal, financial, and legal information.

## Runtime Data Layout

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

## Source Documents

Supported file types:

| Format | Extension | Extraction strategy |
| --- | --- | --- |
| PDF | `.pdf` | PyMuPDF4LLM |
| DOCX | `.docx` | Docling |
| TXT | `.txt` | Native UTF-8 text reader with ignored decode errors |

Upload constraints:

- The filename must be present.
- The file extension must be one of `.pdf`, `.docx`, or `.txt`.
- The file cannot be empty.
- The file cannot exceed `MAX_UPLOAD_SIZE_MB`.

The backend calculates a SHA-256 hash of the uploaded file bytes. If the same file is uploaded again, the existing contract record is returned instead of creating a duplicate.

## SQLite Dataset

SQLite stores operational and structured contract data in `cip.db`.

### `contracts`

The `contracts` table stores one row per unique uploaded contract.

| Field | Type | Description |
| --- | --- | --- |
| `id` | Integer | Primary key. |
| `original_filename` | String | Filename provided by the user. |
| `stored_filename` | String | UUID-based filename used in storage. |
| `stored_file_path` | String | Local path to the uploaded source file. |
| `file_type` | String | File type without leading dot. |
| `file_hash` | String | SHA-256 hash used for deduplication. |
| `status` | String | Current ingestion status. |
| `failed_step` | String, nullable | Pipeline step that failed, if any. |
| `error_message` | Text, nullable | Error message captured on failure. |
| `extracted_markdown` | Text, nullable | Full normalized Markdown extracted from the contract. |
| `title` | String, nullable | Extracted contract title. |
| `contract_type` | String, nullable | Extracted contract type. |
| `landlord` | String, nullable | Extracted landlord, lessor, or owner. |
| `tenant` | String, nullable | Extracted tenant, lessee, or occupant. |
| `property_address` | Text, nullable | Extracted leased property address. |
| `start_date` | Date, nullable | Contract start date. |
| `end_date` | Date, nullable | Contract end date. |
| `monthly_rent` | Float, nullable | Explicit monthly rent amount. |
| `currency` | String, nullable | One of `MXN`, `USD`, or `EUR` when found. |
| `security_deposit` | Float, nullable | Extracted security deposit. |
| `renewal_notice_days` | Integer, nullable | Renewal notice period in days. |
| `early_termination_notice_days` | Integer, nullable | Early termination notice period in days. |
| `risk_level` | String, nullable | `low`, `medium`, `high`, or `unknown`. |
| `created_at` | DateTime | Creation timestamp. |
| `updated_at` | DateTime, nullable | Update timestamp. |

Contract statuses:

```text
uploaded
text_extracted
metadata_extracted
indexed
failed
```

### `contract_chunks`

The `contract_chunks` table stores the text chunks generated for retrieval.

| Field | Type | Description |
| --- | --- | --- |
| `id` | Integer | Primary key. |
| `contract_id` | Integer | Foreign key to `contracts.id`. |
| `chunk_index` | Integer | Sequential chunk index for the contract. |
| `content` | Text | Chunk text. |
| `source_file` | String | Original source filename. |
| `page_number` | Integer, nullable | PDF page number when available. |
| `paragraph_index` | Integer, nullable | DOCX paragraph/block index when available. |
| `line_start` | Integer, nullable | TXT starting line when available. |
| `line_end` | Integer, nullable | TXT ending line when available. |
| `section_title` | String, nullable | Reserved for section metadata when available. |
| `chroma_id` | String | ID of the corresponding ChromaDB vector. |
| `created_at` | DateTime | Creation timestamp. |

Deleting a contract row cascades to related chunk rows through the SQLAlchemy relationship configuration.

## Extracted Metadata Schema

The LLM output is parsed into `ContractMetadata`:

```json
{
  "title": "Commercial Lease Agreement",
  "contract_type": "lease",
  "landlord": "Example Properties LLC",
  "tenant": "Example Tenant Inc.",
  "property_address": "123 Market Street, Example City",
  "start_date": "2026-01-01",
  "end_date": "2026-12-31",
  "monthly_rent": 2500.0,
  "currency": "USD",
  "security_deposit": 2500.0,
  "renewal_notice_days": 60,
  "early_termination_notice_days": 30,
  "risk_level": "medium"
}
```

Rules enforced by the extraction prompt and schema:

- Use only information explicitly present in the contract.
- Do not infer missing values.
- Return `null` for missing values.
- Dates must use `YYYY-MM-DD`.
- Currency must be `MXN`, `USD`, `EUR`, or `null`.
- If rent is annual and monthly rent is not explicitly stated, `monthly_rent` should be `null`.

## ChromaDB Dataset

ChromaDB stores embedded chunks in the configured collection, defaulting to:

```text
contracts
```

Vector IDs use this format:

```text
contract-{contract_id}-chunk-{chunk_index}
```

Each vector stores chunk text and metadata such as:

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
- `page_number`
- `paragraph_index`
- `line_start`
- `line_end`
- `section_title`
- `chunk_index`
- `chroma_id`

Search and chat use this vector dataset for retrieval.

## Generated Text Dataset

The backend stores `extracted_markdown` on the contract row. This value is generated from the source document and becomes the base text for:

- Metadata extraction.
- LangChain document construction.
- Chunking.
- Vector indexing.

Extraction engines:

| Source type | Engine | Unit strategy |
| --- | --- | --- |
| PDF | `pymupdf4llm` | One unit per page with page metadata. |
| DOCX | `docling` | Markdown split into paragraph-like blocks. |
| TXT | `python-native` | Blocks of up to 30 lines. |

## Data Quality Considerations

- Scanned PDFs are rejected when no extractable text is found.
- Very short extracted documents are rejected.
- LLM metadata may be incomplete when a contract omits a field or uses unusual wording.
- Source text quality directly affects metadata extraction, retrieval quality, and chat answers.
- The system should be tested with realistic lease samples that include different layouts, clause wording, and currency/date formats.

## Privacy and Security

Uploaded contracts may contain sensitive data. For demos and portfolio use:

- Prefer synthetic contracts.
- Do not commit files from `backend/data/uploads`.
- Do not commit `cip.db`.
- Do not commit ChromaDB persistence files.
- Do not expose the API publicly without authentication.
- Do not use confidential client contracts unless you have explicit authorization.

The root `.gitignore` excludes local database files, uploaded files, ChromaDB data, virtual environments, and environment files.
