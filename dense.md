# RAGnarok — Dense Vector Index Pipeline

### 1. Purpose

> The dense retrieval layer finds relevant chunks by comparing semantic embeddings of the user’s query with embeddings of indexed chunk text. RAGnarok will use **`TurboVec`** for vector indexing and similarity search, with a configurable embedding service to generate vectors.<br>
After retrieving matching child chunks, RAGnarok uses their parent_chunk_id values to fetch the associated parent context from PostgreSQL. Parent chunks are not embedded or indexed in the dense vector index.

---

### 2. Locked technology and storage decisions

|Component|Decision|
|---|---|
|Retrieval method|**`Dense vector similarity search`**|
|Vector index|**TurboVec**|
|Source text|`chunks.content` in PostgreSQL (child chunks only)|
|Parent context source |`parent_chunks.content` in PostgreSQL, fetched after retrieval
|Chunk creation|Shared Chunking Module, producing parent and child chunks|
|Chunk-size counter|Configured in the Chunking Module; `transformers.AutoTokenizer` when compatible with the selected embedding model|
|Embedding generation|Configurable embedding service|
|Source of chunk records|**PostgreSQL**|
|Vector index persistence|TurboVec built-in persistence|
|Index storage location|Local filesystem under `data/indexes/dense/`|
|Result-to-record mapping|Stable mapping between PostgreSQL chunk UUIDs and TurboVec numeric IDs|
|Index database|No separate vector database|

---

### 3. Index creation

After chunks are available in PostgreSQL:

1. Read each chunk’s `chunk_id` and `content` in batches.
2. Send the child chunk text to the configured embedding service.
3. Validate that the returned vectors match the expected count and dimension.
4. Map each PostgreSQL child `chunk_id` to a stable numeric ID supported by TurboVec.
5. Insert the vectors and numeric IDs into the TurboVec index.
6. Persist the vector index and its corresponding child-ID mapping.

*Parent chunks are stored for context expansion and are not part of the dense index corpus.*

The mapping is required because PostgreSQL uses UUID chunk IDs, while the selected TurboVec ID-mapping index uses unsigned 64-bit numeric IDs. The application uses this mapping to associate search results with the correct PostgreSQL chunk records.

The indexed corpus should use the finalized chunk text from PostgreSQL, not the original or parsed source files.

---

### 4. Index persistence and reload

```
RAGnarok/
└── data/
    ├── raw/
    ├── parsed/
    └── indexes/
        └── dense/
            ├── TurboVec index files
            └── chunk-ID mapping
```

Use TurboVec’s persistence capabilities rather than designing a custom vector-index format. The application remains responsible for choosing the directory, saving the ID mapping, and ensuring that the saved index and mapping correspond to the same corpus snapshot.

---

### 5. Dense Index Ingestion

```
Child chunks available in PostgreSQL
    ↓
Read child chunk IDs and content in batches
    ↓
Send child chunk text to the configured embedding service
    ↓
Receive embeddings for each child chunk
    ↓
Validate vector count, dimensions, and values
    ↓
Map child chunk IDs to stable TurboVec numeric IDs
    ↓
Insert vectors and IDs into the TurboVec index
    ↓
Persist the TurboVec index and child-ID mapping
    ↓
Dense index ready for query-time retrieval
```

---

### 6. Query-time retrieval

```
User query
    ↓
Generate query embedding using the configured embedding service
    ↓
Search the persisted TurboVec index
    ↓
Retrieve top-k matching numeric vector IDs and scores
    ↓
Map numeric IDs to PostgreSQL child chunk IDs
    ↓
Fetch matching child chunk text and metadata from PostgreSQL
    ↓
Collect the matching child chunks' parent_chunk_id values
    ↓
Fetch each associated parent chunk from PostgreSQL
    ↓
Return ranked child matches with parent context
```

The child chunk determines the retrieval match and ranking. Its parent chunk supplies broader context for the downstream answer-generation stage. If multiple matched children share the same parent, retrieve that parent once and retain the individual child matches and scores.

The query embedding must be compatible with the embeddings used to build the index, including the selected model and any required input formatting. The dense layer returns ranked chunks and similarity scores; it does not generate the final answer.

---

### 7. Index maintenance

When indexed child chunk records are added, changed, or deleted, the dense index may no longer reflect the current PostgreSQL data. Rebuild and persist a matching vector index and child-ID mapping so retrieval results stay aligned with the source records.

Changes to parent-only text that do not alter indexed child chunks do not require rebuilding the dense index. Changes to child content, child IDs, chunk boundaries, or embedding configuration may require affected embeddings to be regenerated and the index updated or rebuilt.

---

### 8. Embedding Configuration

The embedding service converts chunk text and user queries into vectors for semantic retrieval. The same compatible embedding configuration must be used for indexing and query processing.

Configurable settings:
- Provider: selected embedding service
- Model: configured embedding model
- Vector dimension: must match the vectors produced by the model and the TurboVec index
- Batch size: number of chunk texts sent per embedding request
- Input formatting: any model-specific document and query prefixes or instructions
- Similarity metric: configured consistently with the embedding model and index

**NOTE**: The embedding service’s tokenizer is separate from the Chunking Module’s size counter. The Chunking Module determines chunk boundaries and metadata; the embedding service generates vectors from the finalized chunk text.

---

### 9. Metadata Filtering

Metadata filtering narrows dense vector search to chunks that match constraints expressed in the user’s terminal query. The user enters a natural-language query; they do not select filters through a UI.

RAGnarok will use PostgreSQL to resolve the supported metadata constraints, then pass the eligible chunk IDs to TurboVec as an allowlist.

---

### Supported metadata fields

| Field | Source | Purpose |
|---|---|---|
| `document_id` | PostgreSQL `documents` | Restrict search to a specific document |
| `filename` | PostgreSQL `documents` | Resolve a named file to its document ID |
| `chunk_index` | PostgreSQL `chunks` | Restrict search to a specific chunk or explicitly stated chunk range |

---

### Filtering FLow

```
User enters query in terminal
        ↓
Identify explicit metadata constraints in query
        ↓
Resolve filename / document_id / chunk_index in PostgreSQL
        ↓
Get eligible chunk records
        ↓
Map eligible chunk IDs to TurboVec numeric IDs
        ↓
Pass numeric IDs as TurboVec search allowlist
        ↓
Run dense top-k similarity search within eligible IDs
        ↓
Map results back to chunk IDs
        ↓
Fetch ranked chunks and metadata from PostgreSQL
```

TurboVec’s `IdMapIndex` supports an ID allowlist during search, so the dense search can be restricted to eligible vector IDs. It does not directly query PostgreSQL metadata; RAGnarok resolves metadata first and supplies the resulting IDs.

---

**Behavior**
- If the query contains no explicit metadata constraints, search the full vector index.
- If constraints resolve to eligible chunks, search only those vectors.
- If no chunks match the constraints, return an empty result without calling TurboVec with an empty allowlist.
- If fewer eligible chunks exist than top_k, return up to the number available.

**NOTE**: Metadata filtering changes which vectors are eligible for search; it does not change how embeddings are generated or how similarity is calculated.