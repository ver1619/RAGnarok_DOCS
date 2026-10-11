# *RAGnarok Docs* 

<div align="center">

![RAGnarok](https://img.shields.io/badge/RAGnarok-v1.0-blueviolet?style=for-the-badge&logo=shield)
![documentation](https://img.shields.io/badge/documentation-Complete-brightgreen?style=for-the-badge&logo=gitbook)

<br/>

**A high-performance, modular, local Python-based Retrieval-Augmented Generation (RAG) system with parent-child chunking, multi-strategy retrieval, native guardrails, and continuous CI/CD evaluation loops.**

</div>

---

## 📖 System Overview

**RAGnarok** is engineered from the ground up for low-latency, deterministic, and verifiable document question answering. Rather than relying on monolithic, opaque RAG frameworks, RAGnarok decomposes the retrieval-augmented lifecycle into discrete, independently testable stages:

1. **Document Ingestion & Normalization**: Layout-aware parsing powered by **Docling** with SHA-256 content deduplication and relational job state machines.
2. **Parent-Child Chunking**: Structural hierarchy preservation where heading-bounded **Parent Chunks** provide broad context for generation, while compact token-bounded **Child Chunks** (128–256 tokens) serve as granular retrieval units.
3. **Decoupled Dual-Tier Storage**: PostgreSQL for relational metadata, job workflows, and parent chunk context; local filesystem for **TurboVec** dense vector indexes and **BM25s** lexical inverted indexes.
4. **Adaptive & Multi-Strategy Retrieval**: Flexible retrieval modes (**Lexical**, **Dense**, or **Hybrid**) driven by user configuration or deterministic query classification. Hybrid mode fuses candidate ranks via **Reciprocal Rank Fusion (RRF)** and refines matches with an optional **Cross-Encoder Reranker**.
5. **Parent Context Expansion**: Top-ranked child matches resolve back to their parent context blocks in PostgreSQL, eliminating fragmented context windows.
6. **Token-Budgeted Context Management & Synthesis**: Strict context window deduplication and packing, followed by asynchronous token streaming via **`litellm`** with verifiable, numbered inline citations (`[1]`, `[2]`).
7. **Native Guardrails & DeepEval CI/CD**: Two-tier guardrails (deterministic fast-path and semantic validation) on input and output, combined with automated **DeepEval** regression testing against golden datasets.

---

## 📑 Documentation Index

The technical specifications are organized below in logical pipeline order, tracing the full lifecycle of data and query execution across the RAGnarok system. Click any document link to navigate directly to its documentation.

| Order | Document | Pipeline Stage | Scope & Technical Focus |
|:---:|:---|:---|:---|
| **01** | [`architecture.md`](./architecture.md) | **System Architecture** | Master system blueprint, end-to-end dataflow diagrams, component boundaries, and overall architectural design decisions. |
| **02** | [`configuration.md`](./configuration.md) | **Configuration** | Central configuration specifications, environment variables, hyperparameter tuning, model endpoints, and indexing thresholds. |
| **03** | [`storage.md`](./storage.md) | **Storage Layer** | Decoupled persistence architecture: PostgreSQL relational schemas (`documents`, `parent_chunks`, `chunks`, `processing_jobs`) and filesystem indexes. |
| **04** | [`ingestion_pipeline.md`](./ingestion_pipeline.md) | **Ingestion Pipeline** | End-to-end document intake workflow, SHA-256 deduplication, Docling layout extraction, and asynchronous multi-stage index building. |
| **05** | [`chunking.md`](./chunking.md) | **Chunking Strategy** | Two-stage chunking implementation: Markdown header hierarchy preservation, section aggregation, and parent/child sizing boundaries. |
| **06** | [`parent_child.md`](./parent_child.md) | **Chunking Strategy** | Theoretical and mechanical rationale for parent-child partitioning, child-only indexing, and relational parent expansion. |
| **07** | [`guardrails.md`](./guardrails.md) | **Safety & Guardrails** | Native two-tier safety framework: Tier 1 deterministic checks (regex, length, injections) and Tier 2 semantic topic and PII validators. |
| **08** | [`self_rag.md`](./self_rag.md) | **Query Processing** | Self-reflective gating mechanism evaluating whether queries require document retrieval or can be answered directly via parametric memory. |
| **09** | [`query_rewriting_adaptive_retrieval.md`](./query_rewriting_adaptive_retrieval.md) | **Query Processing** | Intent disambiguation via LLM query rewriting and deterministic rule-based adaptive routing between Lexical, Dense, and Hybrid modes. |
| **10** | [`lexical.md`](./lexical.md) | **Retrieval Engine** | Lexical retrieval engine powered by `bm25s`, custom tokenization, stopword filtering, inverted index management, and child-level scoring. |
| **11** | [`dense.md`](./dense.md) | **Retrieval Engine** | Dense vector retrieval engine using `TurboVec`, sentence-transformer embeddings, vector indexing, and cosine similarity matching. |
| **12** | [`hybrid_retrieval_model.md`](./hybrid_retrieval_model.md) | **Retrieval Engine** | Hybrid retrieval fusion using Reciprocal Rank Fusion (RRF), score normalization, and optional Cross-Encoder reranking. |
| **13** | [`result_contract.md`](./result_contract.md) | **Retrieval Contracts** | Standardized JSON data contracts for child search outputs, rank schemas, provenance fields, and parent lookup interfaces. |
| **14** | [`context_management.md`](./context_management.md) | **Context Management** | Deduplication of overlapping parent contexts, token budget allocation, lost-in-the-middle reordering, and prompt context payload construction. |
| **15** | [`generation.md`](./generation.md) | **Generation & Citations** | Synthesis engine using `litellm`, async streaming response handling, fallback provider chains, and verifiable inline citation mapping. |
| **16** | [`evals.md`](./evals.md) | **Evaluation & CI/CD** | Automated regression testing suite using `DeepEval`, LLM-as-a-judge scoring (Precision, Recall, Faithfulness), and CI/CD quality gates. |

---

## 🚀 Quick Navigation

| Focus Area | Documentation | Description |
|:---|:---|:---|
| 🏛️ **Architecture & Setup** | [`architecture.md`](./architecture.md)<br>[`configuration.md`](./configuration.md)<br>[`storage.md`](./storage.md) | End-to-end design blueprint, runtime configurations, and PostgreSQL/filesystem storage layer. |
| 📥 **Ingestion & Chunking** | [`ingestion_pipeline.md`](./ingestion_pipeline.md)<br>[`chunking.md`](./chunking.md)<br>[`parent_child.md`](./parent_child.md) | Document intake workflow, markdown header splitting, and parent-child indexing relationships. |
| 🛡️ **Safety & Intent Gating** | [`guardrails.md`](./guardrails.md)<br>[`self_rag.md`](./self_rag.md) | Two-tier input/output guardrails (deterministic & semantic) and Self-RAG retrieval necessity checks. |
| 🔍 **Routing & Retrieval** | [`query_rewriting_adaptive_retrieval.md`](./query_rewriting_adaptive_retrieval.md)<br>[`lexical.md`](./lexical.md)<br>[`dense.md`](./dense.md)<br>[`hybrid_retrieval_model.md`](./hybrid_retrieval_model.md)<br>[`result_contract.md`](./result_contract.md) | Query rewriting, BM25s lexical search, TurboVec vector similarity, RRF hybrid fusion, and result schemas. |
| ✍️ **Context & Synthesis** | [`context_management.md`](./context_management.md)<br>[`generation.md`](./generation.md) | Context window deduplication, token budgeting, LiteLLM async streaming, and inline citation binding. |
| 📊 **Evaluation & CI/CD** | [`evals.md`](./evals.md) | DeepEval regression testing suite, LLM-as-a-judge scoring, and CI/CD quality gates. |