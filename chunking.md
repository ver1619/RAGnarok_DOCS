# RAGnarok — Chunking Plan

> The chunking module converts parsed Markdown documents into size-controlled parent and child chunks for downstream retrieval. It preserves heading context and document order, while applying configurable parent and child sizing rules.<br>
Parent chunks provide broader context for generation. Child chunks are the smaller retrieval units used by lexical and dense indexing. Both retrieval pipelines use the same stored child chunks, which link back to their parent chunks and source documents.

---

### 1. Chunking method

Use a two-stage approach:
1. `MarkdownHeaderTextSplitter` — divides parsed Markdown into sections using configured heading levels and captures the heading hierarchy.
2. **Parent chunk construction** — groups heading-based sections into bounded parent units. Oversized sections can be divided into multiple parent units while retaining their heading path.
3. `RecursiveCharacterTextSplitter` — splits each parent unit into smaller child chunks, preferring natural text boundaries and applying the configured child overlap.

A configurable token-counting backend measures chunk length. When the active embedding model has a compatible `Hugging Face tokenizer`, use `transformers.AutoTokenizer` for model-aligned counting. A fallback counter (`tiktoken`) can be configured when the model tokenizer is unavailable.

---

### 2. Chunking configuration

|Setting	|Purpose|
|---|---|
|`strategy`	|Selects the chunking strategy, initially Markdown header splitting followed by recursive splitting.
|`parent_target_size`	|Desired upper bound for each parent chunk, measured by the configured size counter.
|`child_target_size`	|Desired upper bound for each child chunk.|
|`child_overlap`	|Amount of text repeated between adjacent chunks|
|`parent_overlap`	|Amount of text repeated between adjacent parent chunks|
|`heading_levels`	|Markdown heading levels used to identify section boundaries.
|`separators`	|Preferred split boundaries, ordered from more natural to less natural.
|`size_counter`	|Selects the token-counting backend used during chunking.
|`fallback_counter`	|Optional alternative counter for environments without the model tokenizer.
|`max_input_tokens`	|Configured or resolved maximum input length for the selected embedding service, when known.

---

### Counter fallback
Some embedding services do not expose a compatible tokenizer that RAGnarok can load. In that case, the configured fallback counter can be used to estimate chunk length.

A fallback count is not guaranteed to match the embedding model’s actual token count. Therefore:
- Keep the chunk target conservative relative to the configured model input limit.
- Validate each chunk at the embedding-service boundary whenever the service provides a reliable validation mechanism.
- Do not silently truncate oversized text, because truncation can remove relevant context.

---

### 3. Chunk metadata

**Parent chunk metadata**
- `parent_chunk_id` — unique identifier for the parent chunk.
- `document_id` — source document identifier.
- `parent_index` — zero-based order of the parent within its document.
- `heading_path` — associated heading hierarchy, when available.
- `content` — parent chunk text.

**Child chunk metadata**
- `chunk_id` — unique identifier for the child chunk.
- `document_id` — source document identifier.
- `parent_chunk_id` — identifier of the parent containing this child.
- `chunk_index` — zero-based order of the child within its document.
- `heading_path` — associated heading hierarchy, when available.
- `content` — child chunk text.

Store parent chunks in the `parent_chunks` table and child chunks in the `chunks` table in PostgreSQL. Each child references its parent through `parent_chunk_id`; both parent and child records reference the source document through `document_id`.

For more document storage information **[Storage](storage.md)**

**Example Chunk Metadata**

A combined table with a level column to show the relationship

|level|id|document_id|parent_chunk_id|order|content|
|---|---|---|---|---|---|
|parent|c300...0001|a100...0001|—|0|Parent context text|
|child|b200...0001|a100...0001|c300...0001|0|Child retrieval text|
|child|b200...0002|a100...0001|c300...0001|1|Another child from the same parent|

> `parent_index` orders parent chunks within a document; `chunk_index` orders child chunks within a document.

---

**NOTE**:
The chunk-size counter is separate from both pipeline tokenizers. It measures text only during chunking; it does not define BM25 terms or generate embedding vectors.

If the embedding model changes, RAGnarok should re-check the configured input limit. If the existing chunks exceed the new model’s limit or the desired chunking policy changes, the affected chunks and indexes may need to be regenerated.

---

### 4. Chunking pipeline

```
Parsed Markdown
      |
      v
Read chunking and embedding configuration
      |
      v
Resolve size counter and known model input limit
      |
      v
MarkdownHeaderTextSplitter
  - split at configured heading levels
  - retain heading hierarchy
      |
      v
Build bounded parent chunks
  - group heading-based sections
  - split oversized sections into bounded parents
  - preserve heading path
      |
      v
RecursiveCharacterTextSplitter
  - split each parent into child chunks
  - count text using configured length counter
  - split at preferred natural boundaries
  - apply child overlap
      |
      v
Assign parent and child IDs, document IDs,
order, and heading paths
      |
      v
Store parent_chunks and chunks in PostgreSQL
      |
      +-------------------------------+
      |                               |
      v                               v
Lexical indexing                 Dense indexing
on child chunks                 on child chunks
      |                               |
      v                               v
BM25 index                       Vector index
```

---