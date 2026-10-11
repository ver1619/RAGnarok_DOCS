# RAGnarok — Evaluation Architecture

### 1. Purpose

> Evaluation (Evals) ensures that changes to chunking boundaries, retrieval weights, hybrid fusion constants, or system prompts measurably improve system quality rather than degrading it. In RAGnarok, evaluations run as automated regression tests, scoring both retrieval precision and generation faithfulness against a standardized test set.

---

### 2. Locked Technology and Architectural Decisions

| Component | Decision |
|---|---|
| Evaluation Framework | **`DeepEval`** |
| Test Runner | **`pytest`** |
| Evaluation Methodology | **LLM-as-a-Judge** (evaluates semantic metrics) |
| Core Metrics | **The Evaluation Triad** (Contextual Precision, Contextual Recall, Faithfulness, Answer Relevancy) |
| Test Corpus | Version-controlled Golden Dataset (`tests/data/golden_dataset.json`) |
| Execution Modes | Local CLI runs for development and automated GitHub Actions CI on Pull Requests |
| Quality Gate | Strict threshold gating (all scores must meet minimum score $\ge 0.85$) |

---

### 3. The Evaluation Triad

A reliable RAG system must evaluate retrieval and generation as two separate stages:

```
                              User Query
                                  │
                                  ▼
                     ┌──────────────────────────┐
                     │   RAGnarok Pipeline      │
                     └────────────┬─────────────┘
                                  │
        ┌─────────────────────────┴─────────────────────────┐
        ▼                                                   ▼
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│     1. RETRIEVAL EVALUATION     │       │    2. GENERATION EVALUATION     │
│  "Did we fetch the right data?" │       │  "Did the LLM use it properly?" │
├─────────────────────────────────┤       ├─────────────────────────────────┤
│ • Contextual Precision:         │       │ • Faithfulness:                 │
│   Are the most relevant chunks  │       │   Is the generated answer 100%  │
│   ranked at the very top?       │       │   derived from retrieved facts? │
│                                 │       │   (Detects hallucinations)      │
│ • Contextual Recall:            │       │                                 │
│   Did the search find all facts │       │ • Answer Relevancy:             │
│   needed to answer the query?   │       │   Does the response answer the  │
│                                 │       │   question without fluff?       │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

#### Metric Breakdown

| Metric | Focus Stage | What It Measures | Target Threshold |
|---|---|---|---|
| **Contextual Precision** | Retrieval | Ratio of relevant chunks ranked above irrelevant ones | $\ge 0.85$ |
| **Contextual Recall** | Retrieval | Percentage of expected ground-truth facts retrieved | $\ge 0.85$ |
| **Faithfulness** | Generation | Degree to which the answer is grounded in retrieved chunks (zero hallucination) | $\ge 0.90$ |
| **Answer Relevancy** | Generation | How directly and concisely the answer addresses the user query | $\ge 0.85$ |

---

### 4. The Golden Dataset

Evaluations require a repeatable benchmark known as a **Golden Dataset**.

```
                        Raw Knowledge Documents
                                   │
                                   ▼
                       Human Curated Q&A Pairs
                                   │
                                   ▼
                   Synthetic Generation & Expansion
                                   │
                                   ▼
               Golden Dataset (Version Controlled JSON)
                                   │
               ┌───────────────────┴───────────────────┐
               ▼                                       ▼
        Local Test Runner                       GitHub Actions CI
```

#### Dataset Structure

Each evaluation case defines the user question, the expected ideal response, and the required context facts:

```json
{
  "id": "eval-001",
  "input": "How does parent-child retrieval expand a matched child chunk?",
  "expected_output": "It retrieves the child chunk via vector or lexical search, then uses parent_chunk_id to look up the full parent chunk from PostgreSQL.",
  "expected_context": [
    "Child chunks are embedded and indexed in TurboVec and BM25s.",
    "Parent chunks are fetched from PostgreSQL using parent_chunk_id to supply broader context."
  ]
}
```

| Field | Purpose |
|---|---|
| `input` | The test question sent to RAGnarok |
| `expected_output` | The ground-truth answer against which relevancy is compared |
| `expected_context` | The exact factual excerpts that must be retrieved by the search system |

---

### 5. Evaluation Workflow & CI/CD Pipeline

Evaluations act as automated regression guardrails. Any PR modifying retrieval or prompts must pass the evaluation suite before merging:

```
Developer alters retrieval logic (e.g. adjusts RRF 'k' or changes chunk size)
                                   │
                                   ▼
                       Opens Pull Request on GitHub
                                   │
                                   ▼
                  GitHub Actions runs Evaluation Suite
                                   │
                                   ▼
            RAGnarok processes each Golden Dataset query
                                   │
                                   ▼
             LLM-as-a-Judge scores against the Triad:
             ├── Contextual Precision
             ├── Contextual Recall
             ├── Faithfulness
             └── Answer Relevancy
                                   │
                                   ▼
                    Are all scores >= threshold?
                                  /  \
                                 /    \
                               YES     NO
                               /        \
                              v          v
                       PR Approved     PR Blocked (Regression Detected)
```

---

### 6. Configuration

Evaluation settings are configured in `config.toml`:

```toml
[evals]
framework = "deepeval"
judge_model = "gpt-4o"
golden_dataset_path = "tests/data/golden_dataset.json"

[evals.thresholds]
context_precision = 0.85
context_recall = 0.85
faithfulness = 0.90
answer_relevancy = 0.85
```
