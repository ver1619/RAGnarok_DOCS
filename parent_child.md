# RAGnarok — Parent-Child Retrieval Mechanism

### 1. What is parent-child chunking?

Parent-child chunking is a way of splitting a document into two levels of text units, each with a different job:
- `Parent chunks`: larger sections that preserve broader meaning and surrounding context.
- `Child chunks`: smaller pieces of those parent sections that are easier to match precisely to a user’s question.

In RAGnarok, we will **embed and index only child chunks**. When retrieval finds relevant children, the system uses their parent IDs to fetch the full parent text from PostgreSQL and provide that context to the LLM.

---

### 2. Why use two levels?

If we embed and search only large sections, the search may return a lot of text when only one paragraph is relevant. If we search only small paragraphs, we may find the exact passage but lose the surrounding explanation.

Parent-child retrieval separates these jobs:

|Level	|Main role	|Used for|
|---|---|---|
|Child|Precise retrieval|Dense embeddings and BM25 indexing|
|Parent|Broader context|Fetching full text after a child match, then supplying it to generation|

**NOTE**: The parent is not a second search target in the current design. It is fetched using the relationship stored on the matching child record.

---

### 3. Ingestion and storage flow

```
Original document
       |
       v
Parse document into Markdown
       |
       v
Split Markdown by headings
       |
       v
Build bounded parent chunks
       |
       v
Split each parent into smaller child chunks
       |
       v
Store parent and child records in PostgreSQL
       |
       +---------------------------+
       |                           |
       v                           v
Parent chunks table            Child chunks table
(parent text + metadata)       (child text + parent_chunk_id)
                                   |
                                   v
                         Index child chunks only
                           /               \
                          v                 v
                    BM25 index        TurboVec index
```

Parent chunks are created from heading-based sections, with oversized sections divided into bounded parent units.

---

### 4. How retrieval works?

```
User enters a query
       |
       v
Run retrieval on child chunks
       |
       +------------------------+------------------------+
       |                        |                        | 
       v                        v                        v
  BM25 lexical search      Dense vector search      Hybrid search
       |                        |                        | 
       +------------------------+------------------------+
                                |                                  
                                v
                         rank results
                                |
                                v
                     Select top child matches
                                |
                                v
                  Collect parent_chunk_id values
                                |
                                v
                   Remove duplicate parent IDs
                                |
                                v
              Fetch full parent text from PostgreSQL
                                |
                                v
              Pass expanded parent context to generation
```

Each retrieval result identifies a child chunk and its parent. If several matched children belong to the same parent, RAGnarok fetches that parent once, while preserving the child matches and their ranking information.

---

### 5. Example

Imagine a document with a section called **“Dense Retrieval”**. That section becomes a parent chunk, then is split into smaller child chunks:

```
Parent P1: "Dense Retrieval"
    |
    +-- Child C1: How query embeddings are generated
    |
    +-- Child C2: How vector similarity search works
    |
    +-- Child C3: How matching results are returned
```

A user asks: *“How does the system find similar text?”*
- Dense retrieval searches the embeddings of the child chunks and may return C2 as a match.
- RAGnarok reads C2’s `parent_chunk_id`, finds P1 in PostgreSQL, and retrieves its full text.
- The LLM receives the full parent section as context, not just the small matching child.

That gives the answer generator more surrounding information while keeping the indexed search units small.

---

### 5. How the records are connected

```
documents
   |
   +---- parent_chunks
   |          |
   |          +---- chunks (child records)
   |                    |
   |                    +---- parent_chunk_id
   |
   +---- document_id references on both tables
```

- `parent_chunks` stores the parent text and its document/order metadata.
- `chunks` stores child text, its own order and metadata, and the `parent_chunk_id` linking it to its parent.
- Both tables reference the source document with `document_id`.

> child chunks help RAGnarok find the relevant information; parent chunks provide the broader text context used to generate the answer.