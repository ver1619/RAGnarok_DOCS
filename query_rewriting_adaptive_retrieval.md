# RAGnarok — Query Rewriting & Adaptive Retrieval

> RAGnarok introduces Query Rewriting and Adaptive Retrieval as configurable stages before retrieval.

**Query Rewriting** uses an LLM to transform the user's query into a clearer retrieval-oriented query while preserving its original intent.<br>

**Adaptive Retrieval** uses deterministic rules to select the most appropriate retrieval strategy:
- Lexical → BM25
- Dense → TurboVec
- Hybrid → BM25 + Dense → RRF

Both mechanisms are independently configurable.

---

### Query Rewriting

**Flow**:

```
User Query
    ↓
Query Rewriter
    ↓
Rewritten Query
    ↓
Adaptive Retrieval
```

The query rewriter receives the original user query and produces **one rewritten query** for retrieval.

It does not generate multiple queries. Multi-query expansion is not part of the current design.

---

**LLM-based rewriting**

The rewriter uses a configurable LLM to:
- Clarify ambiguous wording.
- Preserve important technical terminology.
- Make implicit retrieval intent explicit.
- Produce a query better suited for the retrieval corpus.

Example:
```
Original:
"How does it get the bigger context?"

Rewritten:
"How does parent-child retrieval expand a matched child chunk into its parent context?"
```

The rewritten query is then passed to the Adaptive Retrieval router.

---

**Configuration**

```
query_rewriting:
  enabled: true
  model: "<configured-llm>"
```

When:
```
query_rewriting:
  enabled: true
```

the rewritten query is used for retrieval.

When:

```
query_rewriting:
  enabled: false
```

the original user query is passed directly to Adaptive Retrieval.

---

### Adaptive Retrieval

**Purpose**:

Adaptive Retrieval prevents RAGnarok from forcing every retrieval-required query through the Default pipeline.

After query rewriting, a rule-based router selects one retrieval strategy based on characteristics of the query.

```
Rewritten Query
      ↓
Rule-Based Router
      ├── Lexical → BM25
      ├── Dense   → TurboVec
      └── Hybrid  → BM25 + Dense → RRF
```

**Routing Rules**

|Query characteristics|	Strategy|
|---|---|
|Strong exact-match signals + semantic signals|	Hybrid|
|Strong exact-match signals|	Lexical|
|Strong semantic signals|	Dense|
|No strong signal|	Hybrid|


**Exact-match signals**

Examples include:
- Identifiers
- Function/class/API names
- Configuration keys
- Filenames
- Error codes
- Quoted phrases
- Technical terminology
- Version-like tokens

**Semantic signals**

Examples include queries primarily asking for:
- Explanations
- Reasons
- Relationships
- Conceptual understanding
- Paraphrased information

The router evaluates the rewritten query, when rewriting is enabled. If rewriting is disabled, it evaluates the original query.

---

### Configuration

```
adaptive_retrieval:
  enabled: true
```

When enabled, the rule-based router selects between Lexical, Dense, and Hybrid retrieval.

When disabled, Adaptive Retrieval is bypassed and a configured default retrieval strategy is used.

```
adaptive_retrieval:
  enabled: false
  default_strategy: "hybrid"
```

The default strategy should be explicitly configured rather than inferred.

---

### Combined Flow

```
                         User Query
                             |
                             v
                         Self-RAG
                             |
              +--------------+--------------+
              |                             |
        No retrieval                  Retrieval required
              |                             |
              v                             v
             LLM                   Query Rewriting?
              |                       /          \
           Answer                   Yes           No
                                      |            |
                                      v            |
                               Rewritten Query     |
                                      |            |
                                      +------+-----+
                                             |
                                             v
                                   Adaptive Retrieval?
                                      /          \
                                    Yes           No
                                     |             |
                                     v             v
                              Rule-Based Router  Default
                               /     |      \     Strategy
                              /      |       \
                         Lexical   Dense   Hybrid
                           |         |       |
                          BM25    TurboVec  BM25 + Dense
                                             |
                                            RRF
                              \          |          /
                               +---------+----------+
                                         |
                                         v
                              Configurable Reranker
                                         |
                                         v
                                  Final Top-K
                                         |
                                         v
                                  Parent Expansion
                                         |
                                         v
                                  Context Assembly
                                         |
                                         v
                                        LLM
                                         |
                                         v
                                       Answer
```

Query rewriting determines **what query should be searched**; Adaptive Retrieval determines **how that query should be searched**.