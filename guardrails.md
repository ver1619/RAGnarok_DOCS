# RAGnarok — Guardrails Architecture

### 1. Purpose

> Guardrails protect RAGnarok from malicious inputs (prompt injections, jailbreaks, denial-of-service token bloat) and prevent ungrounded hallucinations, sensitive secrets, or PII leaks from reaching the terminal user. RAGnarok adopts a **lightweight, native architecture** combining deterministic pattern matching and structured validation without heavy third-party runtime frameworks.

---

### 2. Locked Technology and Architectural Decisions

| Component | Decision |
|---|---|
| Approach | **Native Python** (No NeMo Guardrails or complex external runtimes) |
| Validation Engine | **`Pydantic v2`** for schema validation |
| Guardrail LLM Client | **`litellm`** using fast, low-latency models (e.g., `llama-3.1-8b-instant` or `gpt-4o-mini`) |
| Input Fast-Path | Deterministic token limits, character sanitization, and regex injection heuristics |
| Semantic Input Guard | Structured LLM classifier evaluating prompt injection and domain topicality |
| Output Protection | Regex-based Secret/PII scanning and Self-RAG grounding verification |
| Failure Strategy | Fail-fast: return polite, standardized fallback responses without invoking retrieval |
| Logging & Auditing | **`structlog`** recording blocked attempts and security events |

---

### 3. End-to-End Guardrail Lifecycle

```
                     User Query (CLI / TUI)
                                │
                                ▼
┌───────────────────────────────────────────────────────────────┐
│                   STAGE 1: INPUT GUARDRAILS                   │
│                                                               │
│   [Tier 1: Deterministic Fast-Path]                           │
│   ├── Token length limits check (max 1,024 tokens)            │
│   ├── Null-byte & ANSI terminal escape sanitization           │
│   └── Known jailbreak/injection heuristics (Regex)            │
│                                                               │
│   [Tier 2: Semantic LLM Guard]                                │
│   ├── Prompt injection & instruction override check           │
│   └── Domain topicality check (aligned to document corpus)    │
└───────────────────────────────┬───────────────────────────────┘
                                │
                        Passed Input Guard?
                               / \
                              /   \
                            YES    NO
                            /       \
                           v         v
┌───────────────────────────────┐   ┌───────────────────────────┐
│     RAGnarok Core Pipeline    │   │  Return Polite Refusal /  │
│                               │   │     Fallback Message      │
│ • Defualt Retrieval           │   └───────────────────────────┘
│ • Parent Context Expansion    │
│ • LLM Answer Synthesis        │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────┐
│                  STAGE 2: OUTPUT GUARDRAILS                   │
│                                                               │
│   [Layer 1: Secret & PII Scanner]                             │
│   ├── Regex scan for API keys, passwords, database URIs       │
│   └── Redact or censor sensitive matches                      │
│                                                               │
│   [Layer 2: Grounding & Faithfulness Verification]            │
│   └── Ensure answer statements are supported by context       │
└───────────────────────────────┬───────────────────────────────┘
                                │
                        Passed Output Guard?
                               / \
                              /   \
                            YES    NO
                            /       \
                           v         v
┌───────────────────────────────┐   ┌───────────────────────────┐
│   Display Final Response in   │   │ Display Grounding Warning │
│         Terminal User UI      │   │    or Fallback Notice     │
└───────────────────────────────┘   └───────────────────────────┘
```

---

### 4. Input Guardrails (Two-Tier Design)

Input guardrails evaluate queries *before* embedding generation or database retrieval, protecting downstream compute resources.

```
User Query
    │
    ▼
[Tier 1: Deterministic Fast-Path] ──(Fails)──→ Reject immediately (< 1ms)
    │ (Passes)
    ▼
[Tier 2: Semantic Model Guard]    ──(Fails)──→ Return domain fallback message
    │ (Passes)
    ▼
Proceed to Retrieval Pipeline
```

#### Tier 1: Deterministic Fast-Path
- **Length Limits:** Enforces a hard ceiling on query length (e.g., maximum 1,024 tokens) to prevent prompt-bloating and denial-of-service.
- **Sanitization:** Strips non-printable characters, null bytes, and ANSI terminal escape codes before queries reach downstream models or terminal renderers.
- **Heuristic Pattern Matcher:** Fast regex scan catching overt injection phrases (e.g., `"ignore all previous instructions"`, `"system prompt override"`, `"you are now in DAN mode"`).

#### Tier 2: Semantic Model Guard
For queries that pass deterministic checks, a lightweight model validates the query's intent against structured criteria:

```json
{
  "is_safe": true,
  "is_topical": true,
  "violation_type": null,
  "fallback_message": null
}
```

- **Prompt Injection Defense:** Ensures the user is not attempting to hijack the system persona or leak internal prompt instructions.
- **Domain Topicality:** Confirms the question relates to the indexed knowledge base. If off-topic, RAGnarok gracefully notifies the user rather than searching irrelevant documents.

---

### 5. Output Guardrails

Output guardrails validate the synthesized text before displaying it in the terminal.

```
Synthesized LLM Response
    │
    ▼
┌─────────────────────────────────────────┐
│ Layer 1: Secret & PII Scanner           │
│ • Detects API keys, DB credentials      │
│ • Replaces matches with [REDACTED]      │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│ Layer 2: Self-RAG Grounding Check       │
│ • Compares answer against parent chunks │
│ • Flags ungrounded factual claims       │
└────────────────────┬────────────────────┘
                     │
                     ▼
          Safe Terminal Display
```

#### Layer 1: Secret & PII Scanner
A fast deterministic scanner inspects the output stream for sensitive information:
- API keys (e.g., OpenAI, Anthropic, HuggingFace tokens)
- Database credentials and connection strings
- Personally Identifiable Information (emails, phone numbers)

Any detected match is automatically masked with `[REDACTED <TYPE>]`.

#### Layer 2: Grounding & Faithfulness Guard
Works alongside the Self-RAG mechanism (see [self_rag.md](self_rag.md)) to verify that claims made in the output are factual deductions from the retrieved parent chunks.

If the model produces ungrounded assertions, RAGnarok replaces or supplements the response with a fallback notice:
> *"I could not find sufficient factual support in the provided documents to answer this question."*

---

### 6. Fallback & Short-Circuit Behavior

When a guardrail triggers, RAGnarok never raises raw application crashes or displays stack traces. Instead, it routes to standardized responses:

| Trigger Scenario | Pipeline Action | User-Facing Fallback Message |
|---|---|---|
| **Query exceeds token limit** | Short-circuit before retrieval | *"Your query exceeds the maximum allowed length (1,024 tokens). Please submit a more concise question."* |
| **Injection pattern detected** | Short-circuit before retrieval | *"Your query could not be processed due to safety policy restrictions."* |
| **Off-topic query (Strict mode)** | Short-circuit before retrieval | *"I am specialized only in answering questions related to the indexed document corpus."* |
| **Ungrounded answer detected** | Intercept generation output | *"I could not find sufficient factual support in the provided documents to answer this question."* |
| **Secret pattern detected** | Redact inline | `[REDACTED CREDENTIAL]` |

---

### 7. Configuration

Guardrail settings are configured in `config.toml`:

```toml
[guardrails]
enabled = true
strict_mode = false                    # If true, strictly reject off-topic questions; if false, best-effort
max_query_tokens = 1024                # Maximum token boundary before fast rejection
enable_pii_filter = true              # Scan and redact secrets/PII in outputs
enable_injection_detection = true     # Enable semantic prompt injection classifier
model = "groq/llama-3.1-8b-instant"   # Fast low-latency LLM for guardrail classification
```
