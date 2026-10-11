# RAGnarok — Lexical Retrieval Layer

### 1. Purpose

> The lexical retrieval layer finds relevant child chunks by matching query terms against indexed child chunk text. RAGnarok will use BM25 via the **`bm25s`** Python library for lexical search.<br>
After retrieving matching child chunks, RAGnarok uses their `parent_chunk_id` values to fetch broader parent context from PostgreSQL. Parent chunks are not included in the BM25 index.

---

### 2. Locked technology and storage decisions

|Component	|Decision|
|---|---|
|Retrieval method*|**BM25**|
|Python library|**bm25s**|
|Source text|`chunks.content` in PostgreSQL (child chunks only)|
|Parent context source |`parent_chunks.content` in PostgreSQL, fetched after retrieval|
|Chunk creation	|Shared Chunking Module, producing parent and child chunks|
|Chunk-size counter	|Configured in the Chunking Module; `transformers.AutoTokenizer` when compatible with the selected embedding model|
|Lexical tokenizer	|bm25s tokenizer|
|Source of chunk records|**PostgreSQL**|
|Index persistence|bm25s built-in save/load|
|Index storage location|Local filesystem under data/indexes/bm25/|
|Result-to-record mapping|Persist ordered child chunk IDs alongside the index|
|Index database|No separate SQLite database or custom index format|

---

### 3. Index creation

After parent and child chunks are available in PostgreSQL:

1. Read each child chunk’s `chunk_id` and content.
2. Tokenize and normalize the child text using `bm25s`.
3. Build the BM25 index from the processed child chunk text.
4. Save the index files to the configured local directory.
5. Save the ordered child-ID mapping used during indexing.

The mapping translates returned indexed positions into the corresponding PostgreSQL child `chunk_id`. Parent chunks are stored separately and are not indexed by BM25.

The indexed corpus should use the finalized child chunk text from PostgreSQL, not the original or parsed source files.

---

### 4. Index persistence and reload

```
RAGnarok/
└── data/
    ├── raw/
    ├── parsed/
    └── indexes/
        └── bm25/
            ├── bm25s index files
            └── ordered chunk-ID mapping
```

Use `bm25s`’s built-in persistence rather than designing a custom BM25 index storage format. The application remains responsible for choosing the directory, saving the ID mapping, and ensuring the saved index and mapping correspond to the same corpus snapshot.

---

### 5. Query-time retrieval

```
User query
    ↓
Tokenize and normalize query using the same BM25 configuration
    ↓
Search persisted BM25 index
    ↓
Rank matching indexed positions
    ↓
Map positions to child chunk IDs
    ↓
Fetch matching child chunk text and metadata from PostgreSQL
    ↓
Collect the matching child chunks' parent_chunk_id values
    ↓
Fetch each associated parent chunk from PostgreSQL
    ↓
Return ranked child matches with parent context
```

BM25 ranks child chunks based on lexical relevance. Parent chunks provide broader context for downstream answer generation; they do not change the BM25 scores. If multiple matched children share a parent, retrieve that parent once while preserving the individual child matches and ranks.

The lexical index is loaded from disk when needed; query-time search does not reread and reindex the original files.

---

### 6. Index maintenance

When indexed child chunk records are added, changed, or deleted, the lexical index may no longer reflect the current PostgreSQL data. Rebuild and persist a matching index and ordered child-ID mapping so retrieval results stay aligned with the source records.

Parent-only content changes that do not alter indexed child chunks do not require rebuilding the BM25 index. Changes to child content, child IDs, or chunk boundaries require the affected child chunks to be regenerated and the lexical index rebuilt. If the dense index uses the same child records, it must also be kept in sync.

---

### 7. BM25 Tokenizer Configuration

The tokenizer converts child chunk text and user queries into tokens that BM25 can index and match. The same configuration must be applied during indexing and query processing to keep retrieval consistent.

Proposed settings:

- Library: bm25s
- Case: lowercase text
- Token splitting: use the library’s default splitter initially
- Stop words: removed
- Stemming: enabled
- Consistency: identical tokenizer settings for chunks and queries

**NOTE**: These settings apply only to lexical retrieval. They do not affect the Chunking Module’s size counter or the tokenizer used internally by the embedding service.

The lexical layer consumes the shared chunks produced by the Chunking Module. Chunking and lexical tokenization are separate processes: the chunk module determines chunk boundaries and metadata, while the lexical layer tokenizes the finalized chunk text for BM25 indexing and query matching.