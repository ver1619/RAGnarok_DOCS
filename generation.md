# RAGnarok — Answer Generation & Citation Architecture

### 1. Purpose

> Generation is the final synthesis stage of RAGnarok. It takes the structured context payload assembled by Context Management and the user's query, executes synthesis via `litellm`, streams tokens asynchronously to the terminal interface in real time, and binds all statements to verifiable, numbered source citations.

---

### 2. Locked Technology and Architectural Decisions

| Component | Decision |
|---|---|
| LLM Client SDK | **`litellm`** providing unified access to local and cloud LLMs |
| Default Synthesis Model | Configurable (e.g., `gpt-4o`, `claude-3-5-sonnet`, `llama-3.3-70b`) |
| Streaming | **Asynchronous token streaming** enabled by default for zero perceived latency |
| Citation Format | Inline bracketed markers (`[1]`, `[2]`) backed by structured provenance footnotes |
| Grounding Strategy | Strict system prompt contract with low temperature ($\le 0.2$) |
| Refusal Policy | Deterministic fallback when retrieved context lacks factual answers |
| Resilience | Configurable multi-model fallback chain for rate limits and provider timeouts |

---

### 3. End-to-End Generation Lifecycle

```
           Assembled Context Payload (from Context Management)
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 1: PROMPT FORMULATION                           │
│                                                                             │
│  • System Prompt: Persona, strict grounding contract, and citation rules    │
│  • Context Section: Numbered source blocks with document/heading metadata   │
│  • User Section: Original query (or clarified query from rewriter)          │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                 STAGE 2: ASYNC LLM INVOCATION & STREAMING                   │
│                                                                             │
│  • Dispatch async streaming request via litellm                             │
│  • Yield tokens immediately to the terminal interface as they arrive        │
│  • Minimal time-to-first-token (TTFT) for interactive terminal responsiveness│
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 3: CITATION & PROVENANCE RESOLUTION                 │
│                                                                             │
│  • Detect inline citation markers ([1], [2], [3]) in the generated text      │
│  • Map each index back to the underlying parent chunk and source document   │
│  • Build the Footnotes / Sources collateral block                           │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 4: OUTPUT GUARDRAIL VALIDATION                     │
│                                                                             │
│  • Pass synthesized text and context through Output Guardrails              │
│  • Scan for sensitive leaks / PII & verify context faithfulness             │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
                         Final Rendered Terminal Output
                         (Streamed Answer + Cited Sources)
```

---

### 4. Prompt Structure & Grounding Contract

To ensure strict factual accuracy and prevent hallucinations, RAGnarok formats the generation prompt with clear boundaries:

```
┌────────────────────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT                                                          │
│ "You are RAGnarok, a technical assistant specialized in document RAG.  │
│  1. Answer the question using ONLY the provided Context sources.       │
│  2. If the context does not contain the answer, explicitly state:       │
│     'I cannot find sufficient information in the provided documents.'  │
│  3. Every factual claim MUST be followed by its source citation, e.g.  │
│     [1], [2]. Do not invent external facts or citations."              │
├────────────────────────────────────────────────────────────────────────┤
│ CONTEXT BLOCKS                                                         │
│ [Source 1]                                                             │
│ Document: rag_overview.pdf | Heading: Parent-Child Chunking            │
│ Content: "Parent chunks provide broader surrounding context..."        │
│                                                                        │
│ [Source 2]                                                             │
│ Document: storage_spec.md | Heading: PostgreSQL Tables                 │
│ Content: "The parent_chunks table stores section context..."           │
├────────────────────────────────────────────────────────────────────────┤
│ USER QUERY                                                             │
│ "How does parent-child retrieval work and where is it stored?"         │
└────────────────────────────────────────────────────────────────────────┘
```

#### Grounding Directives
- **Zero Extrapolation:** The model is prohibited from filling factual gaps with ungrounded general world knowledge.
- **Explicit Citations:** Every key assertion must carry bracketed indicators matching the corresponding numbered `[Source X]` context block.
- **Graceful Refusal:** When retrieval returns low-relevance chunks that fail to address the query, the model acknowledges the lack of information rather than guessing.

---

### 5. Real-Time Token Streaming

In terminal applications (CLI and TUI), latency is felt immediately. Rather than waiting 5–10 seconds for full completion synthesis, RAGnarok streams output tokens incrementally:

```
LLM Provider API
       │ (Chunk 1: "Parent-")
       ▼
   litellm client
       │
       ▼
Terminal UI Renderer ──→ Displays "Parent-" (instant)
       │
LLM Provider API
       │ (Chunk 2: "child retrieval ")
       ▼
   litellm client
       │
       ▼
Terminal UI Renderer ──→ Displays "Parent-child retrieval "
```

- **Time-to-First-Token (TTFT):** Typically 200–500 ms, giving the user immediate visual engagement.
- **Buffer Management:** As tokens stream onto the screen, the system aggregates the full text in memory for post-generation citation resolution and output guardrail validation.

---

### 6. Citation & Provenance Resolution

Once synthesis concludes, the system parses all bracketed citation markers and attaches complete provenance metadata:

```
Generated Answer:
"Parent-child retrieval indexes small child chunks for precise search [1], 
while fetching the larger parent chunk from PostgreSQL for context [2]."

                                   │
                                   ▼
Footnote Resolution:
┌────────────────────────────────────────────────────────────────────────┐
│ [1] rag_overview.pdf                                                   │
│     Section: Retrieval Mechanics > Child Indexing                      │
│     Parent Chunk ID: c300...0001                                       │
│                                                                        │
│ [2] storage_spec.md                                                    │
│     Section: PostgreSQL Metadata > Parent Chunks                       │
│     Parent Chunk ID: c300...0002                                       │
└────────────────────────────────────────────────────────────────────────┘
```

#### Structured Citation Schema

The resulting answer payload follows the contract:

```json
{
  "answer": "Parent-child retrieval indexes small child chunks...",
  "citations": [
    {
      "citation_index": 1,
      "document_id": "a1000000-0000-4000-8000-000000000001",
      "filename": "rag_overview.pdf",
      "parent_chunk_id": "c3000000-0000-4000-8000-000000000001",
      "heading_path": ["Retrieval Mechanics", "Child Indexing"]
    },
    {
      "citation_index": 2,
      "document_id": "a1000000-0000-4000-8000-000000000002",
      "filename": "storage_spec.md",
      "parent_chunk_id": "c3000000-0000-4000-8000-000000000002",
      "heading_path": ["PostgreSQL Metadata", "Parent Chunks"]
    }
  ]
}
```

---

### 7. Resilience & Model Fallback Chain

External LLM APIs can encounter rate limits (HTTP 429), server timeouts, or service interruptions. RAGnarok incorporates an automated fallback chain:

```
                    Attempt Primary Model
                  (e.g., gpt-4o / claude-3-5)
                               │
                       Request Succeeded?
                              / \
                             /   \
                           YES    NO (Rate limit / Timeout / Outage)
                           /       \
                          v         v
                  Stream Response  Attempt Secondary Fallback Model
                                   (e.g., groq/llama-3.3-70b)
                                            │
                                    Request Succeeded?
                                           / \
                                          /   \
                                        YES    NO
                                        /       \
                                       v         v
                               Stream Response  Attempt Local Fallback (Ollama)
                                                 or Return Graceful Error
```

- **Timeout Boundaries:** Strict socket timeout (default: 15 seconds) prevents queries from hanging indefinitely.
- **Token Budget Protection:** If prompt token count exceeds the model's allowed context window, Context Management dynamically trims lower-ranked parent blocks before dispatching the request.

---

### 8. Configuration

Generation parameters are configured in `config.toml` under `[generation]`:

```toml
[generation]
model = "gpt-4o"
temperature = 0.1
max_tokens = 1500
stream = true
timeout_seconds = 15.0

[generation.fallbacks]
enabled = true
models = [
  "anthropic/claude-3-5-sonnet-20241022",
  "groq/llama-3.3-70b-versatile"
]

[generation.citations]
enabled = true
format = "bracketed"     # Numbered inline citations: [1], [2]
show_provenance = true   # Append footnote summary with document & heading path
```
