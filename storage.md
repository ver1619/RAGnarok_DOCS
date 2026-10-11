# RAGnarok — Storage Layer

## Raw Documents

- Users upload supported documents through the application.
- The upload is temporarily held in a staging area while basic file checks are performed.
- Accepted original files are stored unchanged in the local filesystem.
- The raw storage directory is separate from application code and generated processing artifacts.

```
RAGnarok/
└── data/
    ├── staging/       # Temporary uploads awaiting validation
    └── raw/           # Accepted original files
```

Keep `data/` out of Git.

```
User uploads document
        ↓
Application receives file
        ↓
Temporary staging
        ↓
Basic validation
        ↓
Local filesystem
(data/raw/)
```

---

## Parsed Documents

Store the parsed document as a `Markdown (.md)` file in the local filesystem, separate from the original uploaded file.

After upload, the parser extracts the document’s content and writes the parsed output to a dedicated directory. The original raw file remains unchanged and is retained separately.

```
User uploads document
        ↓
Raw document
(data/raw/)
        ↓
Parser
        ↓
Parsed Markdown
(data/parsed/)
```

The parsed Markdown serves as the readable intermediate artifact for downstream processing, including chunk creation and indexing. It can also be regenerated from the retained raw file if parsing needs to be repeated.

```
RAGnarok/
└── data/
    ├── staging/       # Temporary incoming uploads
    ├── raw/           # Original uploaded files
    └── parsed/        # Parser output in Markdown
```

---

## Metadata Store

### Postgres

`PostgreSQL` will be the database for RAGnarok’s document records, parent chunks, child chunks, and chunk text. The filesystem stores the original and parsed files. 

```
Upload file
   ↓
Save original to local filesystem
   ↓
Insert document record in PostgreSQL
   ↓
Parse and save Markdown file locally
   ↓
Update document status
   ↓
Create parent chunks from parsed Markdown
   ↓
Split parent chunks into child chunks
   ↓
Insert parent chunk records in PostgreSQL
   ↓
Insert child chunk records with parent references
```

The filesystem stores the original and parsed files. PostgreSQL stores their paths and the structured records associated with them. Parent chunks preserve broader section context, while child chunks store smaller retrieval units linked to their parent through `parent_chunk_id`.

---

### 1. `documents` table

|Field|Suggested PostgreSQL type|Notes|
|---|---|---|
|`document_id`|	`UUID`|	Primary key|
|`filename`|	`TEXT`|	Original uploaded filename|
|`raw_file_path`|	`TEXT`|	Path to unchanged original file|
|`parsed_file_path`|	`TEXT`|	Path to parsed Markdown|
|`file_type`|	`TEXT`|	Detected source format|
|`file_size_bytes`|	`BIGINT`|	Original file size in bytes|
|`checksum`|	`TEXT`|	Integrity and duplicate detection|
|`status`|	`TEXT`|	Upload/parsing state|
|`created_at`|	`TIMESTAMPTZ`|	Upload record creation time|

---

### 2. `chunks` table

|Field|Suggested PostgreSQL type|Notes|
|---|---|---|
|`chunk_id`|`UUID`|Primary key|
|`document_id`|`UUID`|Foreign key to documents.document_id|
|`content`|`TEXT`|Chunk text, including Markdown syntax where present|
|`chunk_index`|`INTEGER`|Stable order identifier within the document|
|`heading_path`|`TEXT[]`|Stores the hierarchy of Markdown headings associated with the chunk|
|`parent_chunk_id`|`UUID`|Foreign key to parent_chunks.parent_chunk_id|

---

### 3. `parent_chunks` table

|Field|Suggested PostgreSQL type|Notes|
|---|---|---|
|`parent_chunk_id`|`UUID`|Primary key|
|`document_id`|`UUID`|Foreign key to documents.document_id|
|`content`|`TEXT`|Larger parent-context text|
|`parent_index`|`INTEGER`|Stable order within the document|
|`heading_path`|`TEXT[]`|Markdown heading hierarchy, if available|

The `parent_chunks` table stores larger context units, and the chunks table stores child chunks used for retrieval. Each child has a parent reference, while `chunk_index` records the child’s ordering within its document without prescribing how chunks are produced.

---

### 4. `processing_jobs` table

|Field|Suggested PostgreSQL type|Notes|
|---|---|---|
|`job_id`|`UUID`|Primary key|
|`document_id`|`UUID`|Foreign key to documents.document_id|
|`stage`|`TEXT`|Pipeline stage (`queued`, `parsing`, `chunking`, `indexing`, `completed`, `failed`)|
|`error_message`|`TEXT`|Nullable error details if a stage fails|
|`started_at`|`TIMESTAMPTZ`|Job creation and start timestamp|
|`completed_at`|`TIMESTAMPTZ`|Nullable timestamp when the job reached completion or failure|

---

### 5. Relationships and constraints
- `documents.document_id` is the primary key.
- `parent_chunks.document_id` is a foreign key referencing `documents.document_id`.
- `chunks.document_id` is a foreign key referencing `documents.document_id`.
- `chunks.parent_chunk_id` is a foreign key referencing `parent_chunks.parent_chunk_id`, linking each child chunk to its parent.
- `processing_jobs.document_id` is a foreign key referencing `documents.document_id`.
- Add `UNIQUE (document_id, parent_index)` to prevent duplicate parent ordering values within a document.
- Add `UNIQUE (document_id, chunk_index)` so a document cannot have duplicate child chunk order values.
- Use `ON DELETE CASCADE` so deleting a document record also deletes its associated parent chunks, child chunks, and processing jobs. Deleting a parent chunk also deletes its associated child chunks.

---

### 6. Proposed Schema

```sql
CREATE TABLE documents (
    document_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename TEXT NOT NULL,
    raw_file_path TEXT NOT NULL,
    parsed_file_path TEXT NOT NULL,
    file_type TEXT NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    checksum TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'stored'
        CHECK (status IN ('stored', 'parsing', 'parsed', 'failed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE parent_chunks (
    parent_chunk_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL
        REFERENCES documents(document_id)
        ON DELETE CASCADE,
    content TEXT NOT NULL,
    parent_index INTEGER NOT NULL,
    heading_path TEXT[],
    UNIQUE (document_id, parent_index)
);

CREATE TABLE chunks (
    chunk_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL
        REFERENCES documents(document_id)
        ON DELETE CASCADE,
    parent_chunk_id UUID NOT NULL
        REFERENCES parent_chunks(parent_chunk_id)
        ON DELETE CASCADE,
    content TEXT NOT NULL,
    chunk_index INTEGER NOT NULL,
    heading_path TEXT[],
    UNIQUE (document_id, chunk_index)
);

CREATE TABLE processing_jobs (
    job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL
        REFERENCES documents(document_id)
        ON DELETE CASCADE,
    stage TEXT NOT NULL DEFAULT 'queued'
        CHECK (stage IN ('queued', 'parsing', 'chunking', 'indexing', 'completed', 'failed')),
    error_message TEXT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

---

## Sample Table Representation

### documents table

|document_id	|filename	|raw_file_path	|parsed_file_path	|file_type	|file_size_bytes	|checksum	|status|	created_at|
|---|---|---|---|---|---|---|---|---|
|a1000000-0000-4000-8000-000000000001|rag_overview.pdf|data/raw/rag_overview.pdf|data/parsed/rag_overview.md|application/pdf|245760|sha256:abc123...|parsed|2026-09-24 10:00:00+05:30|
a1000000-0000-4000-8000-000000000002|database_notes.docx|data/raw/database_notes.docx|data/parsed/database_notes.md|application/vnd.openxmlformats-officedocument.wordprocessingml.document|98304|sha256:def456...|parsing|2026-09-24 10:15:00+05:30|
a1000000-0000-4000-8000-000000000003|retrieval_paper.pdf|data/raw/retrieval_paper.pdf|data/parsed/retrieval_paper.md|application/pdf|512000|sha256:ghi789...|failed|2026-09-24 10:30:00+05:30|

---

### chunks table

Each row represents one child chunk belonging to a document and linked to a parent chunk. `chunk_index` indicates the child chunk’s order within its document. `parent_chunk_id` identifies the parent whose broader context can be retrieved when the child matches.

|chunk_id|document_id|parent_chunk_id|chunk_index|heading_path|content|
|---|---|---|---|---|---|
b2000000-0000-4000-8000-000000000001|a1000000-0000-4000-8000-000000000001|c3000000-0000-4000-8000-000000000001|0|['Retrieval-Augmented Generation']|# Retrieval-Augmented Generation RAG combines information retrieval with language generation.|
b2000000-0000-4000-8000-000000000002|a1000000-0000-4000-8000-000000000001|c3000000-0000-4000-8000-000000000002|1|['Retrieval-Augmented Generation', 'Retrieval']|## Retrieval The retrieval component finds relevant passages from a knowledge source.|
b2000000-0000-4000-8000-000000000003|a1000000-0000-4000-8000-000000000001|c3000000-0000-4000-8000-000000000003|2|['Retrieval-Augmented Generation', 'Generation']|## Generation The language model uses the retrieved passages as context to generate an answer.|
b2000000-0000-4000-8000-000000000004|a1000000-0000-4000-8000-000000000002|c3000000-0000-4000-8000-000000000004|0|['Database Notes']|# Database Notes A relational database organizes data into tables with defined relationships.
b2000000-0000-4000-8000-000000000005|a1000000-0000-4000-8000-000000000002|c3000000-0000-4000-8000-000000000005|1|['Database Notes', 'Primary Keys']|## Primary Keys A primary key uniquely identifies each row in a table.

---

### parent_chunks table

Each row represents one parent chunk containing broader section context. Child chunks reference it through `parent_chunk_id`.

|parent_chunk_id|document_id|parent_index|heading_path|content|
|---|---|---|---|---|
|c3000000-0000-4000-8000-000000000001|a1000000-0000-4000-8000-000000000001|0|['Retrieval-Augmented Generation']|# Retrieval-Augmented Generation RAG combines information retrieval with language generation....
|c3000000-0000-4000-8000-000000000002|a1000000-0000-4000-8000-000000000001|1|['Retrieval-Augmented Generation', 'Retrieval']|## Retrieval The retrieval component finds relevant passages from a knowledge source....
|c3000000-0000-4000-8000-000000000003|a1000000-0000-4000-8000-000000000001|2|['Retrieval-Augmented Generation', 'Generation']|## Generation The language model uses the retrieved passages as context to generate an answer....
|c3000000-0000-4000-8000-000000000004|a1000000-0000-4000-8000-000000000002|0|['Database Notes']|# Database Notes A relational database organizes data into tables with defined relationships....
|c3000000-0000-4000-8000-000000000005|a1000000-0000-4000-8000-000000000002|1|['Database Notes', 'Primary Keys']|## Primary Keys A primary key uniquely identifies each row in a table....

---

### processing_jobs table

Tracks the asynchronous ingestion and indexing lifecycle for each uploaded document.

|job_id|document_id|stage|error_message|started_at|completed_at|
|---|---|---|---|---|---|
|d4000000-0000-4000-8000-000000000001|a1000000-0000-4000-8000-000000000001|completed|NULL|2026-09-24 10:00:00+05:30|2026-09-24 10:00:15+05:30|
|d4000000-0000-4000-8000-000000000002|a1000000-0000-4000-8000-000000000002|parsing|NULL|2026-09-24 10:15:00+05:30|NULL|
|d4000000-0000-4000-8000-000000000003|a1000000-0000-4000-8000-000000000003|failed|Error: Corrupted PDF header|2026-09-24 10:30:00+05:30|2026-09-24 10:30:04+05:30|

---

### How the tables relate

One document can have multiple processing jobs (e.g. initial ingest and subsequent re-index runs), multiple parent chunks, and each parent chunk can have multiple child chunks.

```
documents
a100...0001 — rag_overview.pdf
        │
        ├── processing_jobs
        │   d400...0001 — completed
        │
        ├── parent_chunks
        │   c300...0001 — parent 0 — RAG overview
        │       │
        │       ├── chunks
        │       │   b200...0001 — child index 0
        │       │   b200...0002 — child index 1
        │       │
        │   c300...0002 — parent 1 — Generation
        │       │
        │       └── chunks
        │           b200...0003 — child index 2
        │
        └── Each child also stores document_id
```

The `document_id` in each processing job, parent, and child row points back to the source document. The `parent_chunk_id` in each child row points to its parent chunk. For example, child chunks `b200...0001` and `b200...0002` belong to parent `c300...0001`, while child `b200...0003` belongs to parent `c300...0002`.

**Note**: Parent and child ordering are tracked separately using parent_index and chunk_index. A document that is still parsing has no parent or child chunk rows yet; a failed document likewise has no chunk rows unless the design later allows retaining partial results.