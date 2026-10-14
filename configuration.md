# RAGnarok — Component Configuration Guide

This guide outlines all the configuration settings required for each component in the RAGnarok architecture. These configurations allow developers to fine-tune the system's performance, chunking logic, retrieval strategies, and more.

## 1. Chunking Module

The Chunking Module controls how original markdown files are broken down into parent and child chunks.

*   `strategy`: Selects the chunking strategy (e.g., Markdown header splitting followed by recursive splitting).
*   `parent_target_size`: The desired upper bound for each parent chunk, measured by the size counter.
*   `child_target_size`: The desired upper bound for each child chunk.
*   `child_overlap`: The amount of text repeated between adjacent child chunks to preserve context.
*   `parent_overlap`: The amount of text repeated between adjacent parent chunks.
*   `heading_levels`: The Markdown heading levels used to identify section boundaries.
*   `separators`: Preferred split boundaries, ordered from more natural (like paragraphs) to less natural (like characters).
*   `size_counter`: Selects the token-counting backend used during chunking (typically a Hugging Face tokenizer).
*   `fallback_counter`: An optional alternative counter for environments without the model tokenizer.
*   `max_input_tokens`: The configured or resolved maximum input length for the selected embedding service.

## 2. Context Management

Context management controls how much context is sent to the Language Model (LLM) for the final generation phase.

*   `max_tokens`: The maximum token budget allocated for context to be passed to the generation LLM. Context selection will prioritize higher-ranked parent chunks until this budget is reached.

## 3. Dense Retrieval Layer (TurboVec)

The embedding service and dense index settings must be aligned to ensure accurate semantic similarity search.

*   `provider`: The selected embedding service provider.
*   `model`: The configured embedding model.
*   `vector_dimension`: Must match the dimensions produced by the chosen model and expected by the TurboVec index.
*   `batch_size`: The number of chunk texts sent per embedding request.
*   `input_formatting`: Any model-specific document and query prefixes or instructions.
*   `similarity_metric`: The metric used for comparison (e.g., cosine similarity).

## 4. Lexical Retrieval Layer (BM25s)

The lexical tokenizer determines how text is broken into searchable terms. It must be consistent for indexing and querying.

*   `library`: The BM25 library used (e.g., `bm25s`).
*   `case`: Setting for case sensitivity (e.g., lowercase text).
*   `token_splitting`: The method used to split text into tokens (e.g., library's default splitter).
*   `stop_words`: Determines whether stop words are removed.
*   `stemming`: Determines whether stemming is enabled.
*   `consistency`: Ensuring identical tokenizer settings for both indexed chunks and user queries.

## 5. Hybrid Retrieval & Reranking

Combines dense and lexical retrieval, optionally followed by a cross-encoder reranker.

*   `k`: The Reciprocal Rank Fusion (RRF) rank constant used when merging dense and BM25 results.
*   `reranker.enabled`: A boolean flag to enable or disable cross-encoder reranking.
*   `reranker.model`: The specific cross-encoder model to use when reranking is enabled.
*   `reranker.top_n`: The number of RRF candidates sent to the reranker for scoring.
*   `reranker.top_k`: The number of final child chunks retained after the reranking step is complete.

## 6. Query Rewriting & Adaptive Retrieval

Controls pre-retrieval optimization and routing logic.

*   **Query Rewriting:**
    *   `query_rewriting.enabled`: A boolean flag to enable or disable LLM-based query rewriting.
    *   `query_rewriting.model`: The LLM used to rewrite and clarify user queries.
*   **Adaptive Retrieval:**
    *   `adaptive_retrieval.enabled`: A boolean flag to enable or disable dynamic routing to Lexical, Dense, or Hybrid retrieval based on query intent.
    *   `adaptive_retrieval.default_strategy`: The fallback retrieval strategy (e.g., "hybrid") used if adaptive retrieval is disabled or a clear routing rule isn't matched.

## 7. Self-RAG

Self-RAG determines if the system can answer a question directly without needing to query the document corpus.

*   `self_rag.enabled`: A boolean flag indicating whether the initial LLM decision gate is active.
*   `self_rag.decision_method`: The mechanism used to make the retrieval decision (e.g., "llm").

## Example Configuration (`config.toml`)

Here is an example `.toml` file demonstrating how these configurations can be structured:

```toml
[ingestion]
staging_directory = "data/staging"
raw_directory = "data/raw"
parsed_directory = "data/parsed"
supported_formats = [".pdf", ".docx", ".md", ".txt"]
batch_size = 50

[chunking]
strategy = "markdown_header_recursive"
parent_target_size = 2000
child_target_size = 500
child_overlap = 50
parent_overlap = 0
heading_levels = ["#", "##", "###"]
separators = ["\n\n", "\n", " ", ""]
size_counter = "transformers.AutoTokenizer"
fallback_counter = "tiktoken"
max_input_tokens = 8192

[context_management]
max_tokens = 4000

[dense_retrieval]
provider = "sentence-transformers"
model = "BAAI/bge-small-en-v1.5"
vector_dimension = 384
batch_size = 100
similarity_metric = "cosine"

[lexical_retrieval]
library = "bm25s"
case = "lowercase"
stop_words_removed = true
stemming_enabled = true

[hybrid_retrieval]
k = 60

[hybrid_retrieval.reranker]
enabled = true
model = "cross-encoder/ms-marco-MiniLM-L-6-v2"
top_n = 50
top_k = 10

[query_rewriting]
enabled = true
model = "groq/openai/gpt-oss-20b"

[adaptive_retrieval]
enabled = true
default_strategy = "hybrid"

[self_rag]
enabled = true
decision_method = "llm"
model = "groq/openai/gpt-oss-20b"

[guardrails]
enabled = true
strict_mode = false
max_query_tokens = 1024
enable_pii_filter = true
enable_injection_detection = true
model = "groq/openai/gpt-oss-20b"

[evals]
framework = "deepeval"
judge_model = "gpt-4o"
golden_dataset_path = "tests/data/golden_dataset.json"

[evals.thresholds]
context_precision = 0.85
context_recall = 0.85
faithfulness = 0.90
answer_relevancy = 0.85

[generation]
model = "groq/openai/gpt-oss-20b"
temperature = 0.1
max_tokens = 1500
stream = true
timeout_seconds = 15.0

[generation.fallbacks]
enabled = true
models = [
  "groq/qwen/qwen3.8-27b",
  "groq/openai/gpt-oss-120b"
]

[generation.citations]
enabled = true
format = "bracketed"
show_provenance = true

[cli]
theme = "dark"
show_waterfall = true
show_provenance = true
```
