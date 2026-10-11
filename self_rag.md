# RAGnarok - Self-RAG

```
User Query
    |
    v
Self-RAG Decision
    |
    +-----------------------------+
    |                             |                             
    v                             v
Can LLM answer? = YES       Can LLM answer? = NO
    |                             |
    v                             v
Generate Answer             BM25 / Dense / Hybrid
                                  |
                                  v
                              RRF (if Hybrid)
                                  |
                                  v
                            Reranker?
                                  |
                                  v
                         Parent Expansion
                                  |
                                  v
                         Context Assembly
                                  |
                                  v
                              LLM Answer
```

### Important distinction

The initial LLM call is not the final answer generation call.

It acts as a decision gate:

> “Can I answer this query reliably using my internal knowledge, without retrieving information from the document corpus?”

If **yes**, proceed directly to generation.

If **no**, send the query through the RAG pipeline.

---

### Configuration

```
self_rag:
  enabled: true
```

**`enabled: true`**

```
Query
  ↓
Self-RAG decision
  ├── Answerable → LLM
  └── Not answerable → Default RAG Pipeline
```

**`enabled: false`**

```
Query
  ↓
Default RAG Pipeline
  ↓
BM25 / Dense / Hybrid
  ↓
RRF (if Hybrid)
  ↓
Optional Reranker
  ↓
Parent Expansion
  ↓
LLM
```

This gives RAGnarok two operational modes without changing the underlying retrieval components.

---

RAGnarok will use an **LLM-based decision mechanism** for Self-RAG.

```
self_rag:
  enabled: true
  decision_method: "llm"
```

Behavior:

- `enabled: true` → LLM determines whether retrieval is required.
  - **No retrieval required** → answer directly using the LLM.
  - **Retrieval required** → proceed to the Default RAG Pipeline.

- `enabled: false` → bypass Self-RAG and directly enter the Default RAG Pipeline.

```
Query
  ↓
Self-RAG LLM Decision
  ├── No retrieval → LLM → Answer
  │
  └── Retrieval required
          ↓
     BM25 / Dense / Hybrid
          ↓
        RRF (if Hybrid)
          ↓
   Optional Reranker
          ↓
   Parent Expansion
          ↓
   Context Assembly
          ↓
         LLM
          ↓
        Answer
```