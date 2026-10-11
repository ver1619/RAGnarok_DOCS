# RAGnarok — System Architecture

> **RAGnarok** is a high-performance, local Python-based Retrieval-Augmented Generation (RAG) system. It combines Docling document parsing, parent-child chunking, configurable retrieval strategies (BM25s lexical search, TurboVec dense vector similarity, or Hybrid fusion), Cross-Encoder reranking, native guardrails, and DeepEval CI/CD evaluation loops.

---

## 1. Ingestion Pipeline

### Description
The Ingestion Pipeline transforms raw, unstructured documents (PDF, DOCX, TXT, Markdown) into structured, search-ready representations. Incoming files are staged and deduplicated using SHA-256 content hashing. **Docling** extracts document layout into normalized Markdown, which is then partitioned into heading-bounded **Parent Chunks** and token-bounded **Child Chunks**. Each stage transitions through the `processing_jobs` state machine in PostgreSQL.

```mermaid
flowchart TD
    RawFile["Incoming Document\n(PDF, DOCX, TXT, MD)"] --> Staging["Staging Directory\n(data/staging/)"]
    Staging --> HashCheck{"SHA-256 Checksum\nDuplicate Check"}
    
    HashCheck -- "Duplicate Found" --> Skip["Reject / Skip Ingestion"]
    HashCheck -- "New File" --> MoveRaw["Move to Permanent Store\n(data/raw/)"]
    
    MoveRaw --> CreateJob["Create Job in PostgreSQL\n(processing_jobs: QUEUED)"]
    CreateJob --> Docling["Docling Parser\n(Extract Headings, Tables, Text)"]
    
    Docling --> SaveMD["Persist Parsed Markdown\n(data/parsed/<doc_id>.md)"]
    SaveMD --> ParentSplit["Parent Chunking\n(Heading Section Boundaries)"]
    
    ParentSplit --> ChildSplit["Child Chunking\n(Token Bounded: 128-256 tokens)"]
    ChildSplit --> ReadyForStorage["Chunks Ready for Database\n& Index Synchronization"]
```

---

## 2. Storage & Indexing Architecture

### Description
RAGnarok uses a decoupled, dual-tier storage strategy:
- **PostgreSQL** serves as the relational metadata store, tracking documents, parent chunk context, child chunk retrieval units, and job lifecycles.
- **Local Filesystem** stores raw source documents, parsed Markdown files, the **BM25s** lexical inverted index, and the **TurboVec** dense vector index.

```mermaid
flowchart TD
    subgraph PostgreSQL ["PostgreSQL Relational Store"]
        Docs["documents\n(metadata, file paths, checksum)"]
        Jobs["processing_jobs\n(stage tracking, errors, timestamps)"]
        Parents["parent_chunks\n(full section text, heading path)"]
        Children["chunks\n(child text, parent_chunk_id, tokens)"]
        
        Docs --> Jobs
        Docs --> Parents
        Parents --> Children
    end

    subgraph DualIndexing ["Dual Indexing Subsystem"]
        Children -->|Child Text| Embedder["Embedding Service\n(sentence-transformers)"]
        Embedder --> TurboVec["TurboVec Vector Index\n(data/indexes/dense/)"]
        
        Children -->|Child Text| LexTokenizer["BM25s Tokenizer\n(Stemming, Stopwords)"]
        LexTokenizer --> BM25Index["BM25s Inverted Index\n(data/indexes/lexical/)"]
    end
```

---

## 3. Query & Retrieval Pipeline

### Description
Queries are optimized prior to retrieval: **Self-RAG** determines whether retrieval is necessary, and **Query Rewriting** clarifies ambiguous wording. **Hybrid mode is not default**; the retrieval strategy is explicitly chosen by the user via configuration (**Lexical**, **Dense**, or **Hybrid**), or dynamically resolved by **Adaptive Retrieval** based on query characteristics. When Hybrid retrieval is selected, results from BM25s and TurboVec are merged via **Reciprocal Rank Fusion (RRF)** and optionally reranked using a **Cross-Encoder**. Finally, **Parent Expansion** fetches the associated parent context from PostgreSQL for the top child matches.

```mermaid
flowchart TD
    UserQuery["User Query"] --> SelfRAG{"Self-RAG Gate\nRetrieval Needed?"}
    
    SelfRAG -- "NO" --> DirectLLM["Direct LLM Answer\n(Internal Knowledge)"]
    SelfRAG -- "YES" --> Rewriter{"Query Rewriting\nEnabled?"}
    
    Rewriter -- "YES" --> ProcessedQuery["Rewritten Query"]
    Rewriter -- "NO" --> ProcessedQuery["Original Query"]
    
    ProcessedQuery --> AdaptiveCheck{"Adaptive Retrieval\nEnabled?"}
    
    AdaptiveCheck -- "NO (Use Configured Strategy)" --> UserStrategy{"User-Selected Strategy\n(Configured in config.toml)"}
    AdaptiveCheck -- "YES (Dynamic Rules)" --> Router{"Adaptive Router\n(Query Signals)"}
    
    Router -- "Exact match signals" --> BM25["BM25s Lexical Search"]
    Router -- "Semantic signals" --> TurboVec["TurboVec Dense Search"]
    Router -- "No strong signal" --> UserStrategy
    
    UserStrategy -- "Lexical" --> BM25
    UserStrategy -- "Dense" --> TurboVec
    UserStrategy -- "Hybrid" --> HybridBranch["Hybrid Retrieval"]
    
    HybridBranch --> BM25
    HybridBranch --> TurboVec
    
    BM25 --> CheckHybrid{"Is Hybrid Search?"}
    TurboVec --> CheckHybrid
    
    CheckHybrid -- "YES" --> RRF["Reciprocal Rank Fusion (RRF)"]
    CheckHybrid -- "NO" --> RerankerCheck
    RRF --> RerankerCheck{"Cross-Encoder Reranker\nEnabled?"}
    
    RerankerCheck -- "YES" --> TopCandidates["Top Ranked Child Chunks"]
    RerankerCheck -- "NO" --> TopCandidates
    
    TopCandidates --> ParentExpansion["Parent Context Expansion\n(Fetch parent_chunk_id from PostgreSQL)"]
    ParentExpansion --> ContextAssembly["Context Assembly\n(Ordering, Deduplication, Budgeting)"]
```

---

## 4. Generation, Evals & Guardrails Architecture

### Description
Safety and quality are enforced at every boundary:
- **Input Guardrails** filter prompt injections, jailbreaks, and off-topic queries before expensive retrieval runs.
- **Generation Layer** formulates a grounded prompt, executes asynchronous token streaming via `litellm` (with multi-model fallback resilience), and maps factual assertions to numbered inline citations (`[1]`, `[2]`).
- **Output Guardrails** scan the generated text for credential/PII leaks and verify factual grounding against context chunks.
- **Evaluation Loop** executes **DeepEval** against a version-controlled Golden Dataset in GitHub Actions to prevent quality regressions.

```mermaid
flowchart TD
    subgraph GuardrailsAndGeneration ["Runtime Generation & Guardrails"]
        RawQuery["Raw User Query"] --> InGuardFast{"Tier 1: Deterministic Guard\n(Length, Escapes, Injections)"}
        InGuardFast -- "Failed" --> InRefusal["Standardized Safety Refusal"]
        InGuardFast -- "Passed" --> InGuardSemantic{"Tier 2: Semantic Guard\n(Topicality & Overrides)"}
        
        InGuardSemantic -- "Failed" --> InRefusal
        InGuardSemantic -- "Passed" --> CoreRetrieval["Core Retrieval & Context Assembly"]
        
        CoreRetrieval --> PromptBuild["Prompt Formulation\n(System Rules + Context Blocks + Query)"]
        PromptBuild --> LLMEngine["litellm Synthesis Engine\n(Multi-Model Fallback Chain)"]
        
        LLMEngine --> TokenStream["Real-Time Token Streaming\n(Sub-500ms TTFT)"]
        TokenStream --> CitationMapper["Citation & Provenance Resolver\n(Inline [1], [2] to Footnotes)"]
        
        CitationMapper --> OutGuard{"Output Guardrails\n(PII Scanner & Faithfulness)"}
        OutGuard -- "Failed (Hallucination)" --> OutFallback["Grounding Warning / Fallback Notice"]
        OutGuard -- "Passed / Redacted" --> FinalOutput["Final Rendered Terminal Output\n• Streamed Grounded Answer\n• Numbered Citations & Footnotes\n• Provenance / Latency Drawer"]
    end

    subgraph CI_CD_Evals ["CI/CD Evaluation Loop (DeepEval)"]
        PR["Pull Request Created"] --> LoadGolden["Load Golden Dataset\n(tests/data/golden_dataset.json)"]
        LoadGolden --> RunDeepEval["DeepEval Test Runner\n(pytest tests/evals/)"]
        RunDeepEval --> Judge["LLM-as-a-Judge\n(Precision, Recall, Faithfulness, Relevancy)"]
        Judge --> EvalGate{"All Scores >= 0.85?"}
        EvalGate -- "YES" --> PRPass["PR Approved"]
        EvalGate -- "NO" --> PRFail["PR Blocked (Regression Detected)"]
    end
```

---

## 5. Complete End-to-End Pipeline

### Description
The master architecture connects document ingestion, relational and vector indexing, secure query handling, user-configured retrieval (Lexical, Dense, or Hybrid based on user selection or dynamic routing), streaming answer synthesis with citation mapping, output safety validation, and automated continuous evaluation into a unified, modular system.

```mermaid
flowchart TD
    subgraph INGESTION ["1. Ingestion & Storage"]
        Doc["Raw Document\n(PDF, DOCX, TXT, MD)"] --> Parse["Docling Parser"]
        Parse --> MD["Parsed Markdown\n(data/parsed/)"]
        MD --> Chunk["Parent-Child Chunker\n(Heading & Token Boundaries)"]
        Chunk --> PG[("PostgreSQL\n(documents, parent_chunks,\nchunks, processing_jobs)")]
        Chunk --> DenseIdx[("TurboVec Vector Index\n(data/indexes/dense/)")]
        Chunk --> LexIdx[("BM25s Lexical Index\n(data/indexes/lexical/)")]
    end

    subgraph QUERY ["2. Input Guardrails & Query Routing"]
        User["User Terminal\n(CLI / TUI)"] --> InGuard{"Input Guardrails\n(Fast-path + Semantic)"}
        InGuard -- "Passed" --> SelfRAG{"Self-RAG Gate\nRetrieval Needed?"}
        InGuard -- "Blocked" --> SafetyRefusal["Standardized Safety Refusal"]
        SafetyRefusal --> User
        
        SelfRAG -- "YES" --> Rewrite["Query Rewriter\n(Clarifies Intent)"]
        Rewrite --> StrategySelect["Strategy Selector\n(User Choice or Adaptive)"]
    end

    subgraph RETRIEVAL ["3. Retrieval & Expansion"]
        StrategySelect --> LexSearch["BM25s Lexical Search"]
        StrategySelect --> DenseSearch["TurboVec Dense Search"]
        LexSearch --> Fusion{"RRF Fusion\n(If Hybrid)"}
        DenseSearch --> Fusion
        Fusion --> RerankStep["Cross-Encoder Reranker\n(Optional)"]
        LexSearch --> RerankStep
        DenseSearch --> RerankStep
        RerankStep --> TopChildren["Top-K Child Chunks"]
        TopChildren --> ParentFetch["Parent Lookup in PostgreSQL\n(parent_chunk_id Expansion)"]
        ParentFetch --> AssembledCtx["Assembled Context Payload\n(Ordering, Deduplication, Budget)"]
    end

    subgraph OUTPUT ["4. Synthesis & Output Generation"]
        AssembledCtx --> PromptForm["Prompt Assembly\n(System Rules + Context Blocks + Query)"]
        SelfRAG -- "NO (Direct Answer)" --> LLMEngine["litellm Synthesis Engine\n(Multi-Model Fallback Chain)"]
        PromptForm --> LLMEngine
        
        LLMEngine --> TokenStream["Async Token Stream\n(Incremental Terminal Output)"]
        TokenStream --> CitationMapper["Citation & Footnote Resolver\n(Inline [1], [2] to Documents)"]
        CitationMapper --> OutGuard{"Output Guardrails\n(PII Scanner & Faithfulness)"}
        
        OutGuard -- "Failed (Hallucination)" --> WarningNotice["Grounding Warning / Fallback Notice"]
        OutGuard -- "Passed / Clean" --> RenderedOutput["Rendered Terminal Presentation\n• Streamed Answer Text\n• Numbered Source Citations\n• Heading & Document Footnotes"]
        
        RenderedOutput --> User
        WarningNotice --> User
    end

    subgraph EVALS ["5. Continuous Evaluation Loop"]
        Golden[("Golden Dataset\n(tests/data/golden_dataset.json)")] --> DeepEval["DeepEval Test Runner\n(pytest tests/evals/)"]
        DeepEval -.->|Evaluates Quality| INGESTION
        DeepEval -.->|Evaluates Quality| RETRIEVAL
        DeepEval -.->|Evaluates Quality| OUTPUT
    end

    PG -.->|Supplies Parent Context| ParentFetch
    DenseIdx -.->|Vector Search| DenseSearch
    LexIdx -.->|Lexical Search| LexSearch
```
