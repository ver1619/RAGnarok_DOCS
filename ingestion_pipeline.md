# RAGnarok — End-to-End Ingestion Pipeline

### 1. Purpose

> The Ingestion Pipeline is the backbone of RAGnarok. It automates the journey of raw, unstructured files (PDF, DOCX, Markdown, Text) into structured, search-ready representations. It coordinates parsing, parent-child chunking, relational metadata storage in PostgreSQL, and simultaneous index builds across TurboVec (dense vector) and BM25s (lexical).

---

### 2. Locked Technology and Architectural Decisions

| Component | Decision |
|---|---|
| Document Parser | **`Docling`** converts source documents into clean Markdown |
| Intermediate Artifact | Normalized Markdown files saved under `data/parsed/` |
| Chunking Strategy | **Parent-Child Chunking** (Heading-based parents, token-bounded children) |
| Relational Storage | **`PostgreSQL`** for `documents`, `parent_chunks`, `chunks`, and `processing_jobs` |
| Lexical Index | **`bm25s`** indexed directly from child chunk text |
| Vector Index | **`TurboVec`** storing child chunk embeddings with ID mappings |
| Deduplication | **SHA-256 file hashing** to prevent redundant parsing and storage |
| Orchestration State | **`processing_jobs`** table managing stage progression and failure recovery |

---

### 3. End-to-End Ingestion Lifecycle

```
                     Incoming Document (PDF, DOCX, TXT, MD)
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 1: VALIDATION & STAGING                         │
│                                                                             │
│  • Receive file into data/staging/                                          │
│  • Compute SHA-256 hash                                                     │
│  • Check for duplicate hash in PostgreSQL documents table                   │
│    ├── Hash exists? ──→ Skip or notify duplicate                            │
│    └── New file?    ──→ Move to data/raw/ & create processing_jobs record   │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          STAGE 2: DOCLING PARSING                           │
│                                                                             │
│  • Extract text, tables, and hierarchical headings using Docling            │
│  • Normalize layout into Markdown (.md)                                     │
│  • Persist parsed Markdown under data/parsed/<document_id>.md               │
│  • Update processing_jobs status: PARSING → COMPLETED                       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STAGE 3: PARENT-CHILD CHUNKING                         │
│                                                                             │
│  • Split parsed Markdown by structural headings into Parent Chunks          │
│  • Bound oversized parent sections to max token threshold                   │
│  • Subdivide each parent chunk into smaller Child Chunks                    │
│  • Assign chunk indices, token counts, and parent references                │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 4: RELATIONAL METADATA PERSISTENCE                  │
│                                                                             │
│  • Open atomic PostgreSQL transaction                                       │
│  • Insert document record into documents table                              │
│  • Insert all parent sections into parent_chunks table                      │
│  • Insert all child records into chunks table (with parent_chunk_id)        │
│  • Commit transaction & update processing_jobs: CHUNKING → COMPLETED        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STAGE 5: DUAL INDEX CREATION                        │
│                                                                             │
│          ┌───────────────────────────┴───────────────────────────┐          │
│          ▼                                                       ▼          │
│  [Lexical Pipeline]                                      [Dense Pipeline]   │
│  • Read child chunk text                                 • Read child text  │
│  • Tokenize & apply stemming/stopwords                  • Batch embed text │
│  • Build BM25s index                                     • Map UUIDs to IDs │
│  • Persist to data/indexes/lexical/                      • Insert TurboVec  │
│                                                          • Persist Dense    │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STAGE 6: INGESTION COMPLETE                         │
│                                                                             │
│  • Mark processing_jobs status: COMPLETED                                   │
│  • Updated indices are immediately available for query-time retrieval       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 4. Stage Breakdown

#### Stage 1: Validation, Deduplication & Staging
1. Incoming files land in `data/staging/`.
2. The pipeline calculates a **SHA-256 checksum** of the raw file content.
3. If the checksum matches an existing entry in the `documents` table, the pipeline flags the file as a duplicate, preventing redundant parsing and duplicate chunk entries.
4. If unique, the file moves permanently to `data/raw/<document_id>.<ext>`, and a tracking record is created in `processing_jobs`.

#### Stage 2: Document Parsing
1. **Docling** reads the raw document, extracting reading order, heading levels (`#`, `##`, `###`), tables, and paragraphs.
2. The extracted content is converted into structured Markdown and written to `data/parsed/<document_id>.md`.
3. Retaining the parsed Markdown on the filesystem allows developers to inspect parsing quality or re-chunk documents without re-running expensive parser OCR.

#### Stage 3: Parent-Child Chunking
1. The chunking module parses the Markdown document along heading boundaries to create **parent chunks** (preserving contextual coherence).
2. Each parent chunk is subdivided into smaller **child chunks** using token boundaries (typically 128–256 tokens).
3. Every child chunk retains a strict relational pointer (`parent_chunk_id`) back to its source parent.

#### Stage 4: PostgreSQL Persistence
1. Parent chunks are written to `parent_chunks` (`parent_chunk_id`, `document_id`, `content`, `heading_path`, `token_count`).
2. Child chunks are written to `chunks` (`chunk_id`, `parent_chunk_id`, `document_id`, `content`, `token_count`, `chunk_index`).
3. Storage occurs within a database transaction: if a write fails, no orphaned chunk records remain.

#### Stage 5: Dual Indexing (Dense & Lexical)
Once child chunk records exist in PostgreSQL:
* **Lexical Indexing (BM25s):** Child chunk texts are read, tokenized, and indexed into `bm25s`. The index is saved to `data/indexes/lexical/`.
* **Dense Indexing (TurboVec):** Child chunk texts are embedded in batches via the configured embedding model. Each chunk UUID is mapped to a numeric 64-bit ID and inserted into the `TurboVec` index, saved to `data/indexes/dense/`.

---

### 5. Job State Machine & Failure Boundaries

Ingestion status is tracked deterministically in PostgreSQL via the `processing_jobs` table:

```
                      ┌───────────────┐
                      │    QUEUED     │
                      └───────┬───────┘
                              │
                              ▼
                      ┌───────────────┐
                      │    PARSING    │
                      └───────┬───────┘
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
       ┌───────────────┐             ┌───────────────┐
       │   CHUNKING    │             │    FAILED     │
       └───────┬───────┘             │ (Parse error) │
               │                     └───────────────┘
        ┌──────┴──────┐
        ▼             ▼
┌───────────────┐ ┌───────────────┐
│   INDEXING    │ │    FAILED     │
└───────┬───────┘ │(Chunk/DB err) │
        │         └───────────────┘
   ┌────┴────┐
   ▼         ▼
┌──────────┐ ┌───────────────┐
│COMPLETED │ │    FAILED     │
└──────────┘ │ (Index error) │
             └───────────────┘
```

#### State Definitions & Recovery

| Job Status | Description | Action on Failure |
|---|---|---|
| `QUEUED` | Job registered, awaiting worker execution | Can be safely cancelled or retried |
| `PARSING` | Docling actively extracting content | File retained in `data/raw/`; mark `FAILED` with error log |
| `CHUNKING` | Markdown splitting into parent and child chunks | Retryable from existing `data/parsed/` file |
| `INDEXING` | Generating embeddings and updating BM25s/TurboVec | Database records intact; re-run indexing stage only |
| `COMPLETED` | Document parsed, stored, and indexed | Ready for retrieval |
| `FAILED` | Exception occurred during processing | Detailed stack trace logged in `processing_jobs.error_message` |

---

### 6. Idempotency & Re-Indexing

To support evolving models and configurations, RAGnarok supports two types of updates:

1. **Document-Level Idempotency:**
   - If the exact file is re-submitted, the SHA-256 hash check prevents duplicate ingestion.
   - If a document is updated, the previous records in `documents`, `parent_chunks`, and `chunks` cascade-delete, and the document is re-ingested.

2. **Full Re-Indexing (Model Upgrade):**
   - If the embedding model or BM25 tokenizer changes, the raw text and parsed Markdown already exist in PostgreSQL and `data/parsed/`.
   - The system can rebuild both TurboVec and BM25s indexes directly from PostgreSQL child chunk records without re-parsing raw files.

---

### 7. Directory Layout During Ingestion

```
RAGnarok/
└── data/
    ├── staging/                 # Incoming uploads awaiting hash verification
    ├── raw/                     # Verified original source files
    ├── parsed/                  # Docling extracted Markdown (.md)
    └── indexes/
        ├── dense/               # Persisted TurboVec vector index & ID mapping
        └── lexical/             # Persisted BM25s index artifacts
```

---

### 8. Configuration

Ingestion behavior is defined in `config.toml`:

```toml
[ingestion]
staging_directory = "data/staging"
raw_directory = "data/raw"
parsed_directory = "data/parsed"
supported_formats = [".pdf", ".docx", ".md", ".txt"]
batch_size = 50

[ingestion.deduplication]
hash_algorithm = "sha256"
reject_duplicates = true

[ingestion.parser]
engine = "docling"
export_format = "markdown"
```
