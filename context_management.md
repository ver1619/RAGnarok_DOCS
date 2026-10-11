# RAGnarok — Context Management

> Context Management is the stage that converts the selected retrieval results into the final context supplied to the generation LLM.

**Purpose**

Context Management is responsible for:
- Receiving final ranked child-chunk results.
- Using parent expansion results to obtain broader context.
- Ordering the retrieved context appropriately.
- Removing duplicate parent contexts.
- Managing the amount of context passed to the LLM.
- Constructing the final context payload for generation.

It does not perform retrieval, reranking, or answer generation.

---

### Pipeline Position

```
Query
  ↓
Self-RAG (if enabled)
  ↓
Query Rewriting (if enabled)
  ↓
Adaptive Retrieval (if enabled) 
  ├── Lexical
  ├── Dense
  └── Hybrid → RRF
  ↓
Reranker (if enabled)
  ↓
Final Top-K Child Chunks
  ↓
Parent Expansion
  ↓
┌───────────────────┐
│ Context Management│           
└─────┬─────────────┘
      ↓
  Generation LLM
      ↓
   Answer
```

---

### Context Assembly Flow

```
Final ranked child results
          ↓
Retrieve associated parents
          ↓
Deduplicate parents
          ↓
Order parent contexts
          ↓
Apply context limits
          ↓
Build context blocks
          ↓
Attach provenance
          ↓
Final LLM context
```

**Example**

Suppose retrieval produces:
```
Child C7  → Parent P2 → rank 1
Child C3  → Parent P1 → rank 2
Child C9  → Parent P2 → rank 3
```

Parent P2 is only included once:
```
P2
 └── matched by C7, C9

P1
 └── matched by C3
```

The parent ordering follows the best-ranked associated child:
```
1. P2
2. P1
```

The individual child matches remain available as provenance.

---

### Context Ordering

The default ordering should follow retrieval relevance.

For each parent:
> Position the parent according to the highest-ranked child that caused it to be selected.

Do not calculate a new parent relevance score.

For example:
```
Child ranking:

C12 → score 0.92 → P4
C05 → score 0.81 → P2
C19 → score 0.77 → P4
C07 → score 0.71 → P1

Final parent order:

P4
P2
P1
```

`P4` remains first because its best child match was ranked first.

---

### Context Deduplication

Multiple retrieved children can belong to the same parent.

The assembly layer must:
1. Identify unique `parent_chunk_id`s.
2. Fetch/use each parent only once.
3. Preserve all matching child references.
4. Avoid sending duplicate parent text to the LLM.

This is particularly important because hybrid retrieval and reranking can produce multiple hits from the same parent.

---

### Context Size Management

Parent expansion can produce substantially more text than the original child results.

Therefore, Context Management needs a configurable context budget.

```
context:
  max_tokens: <configured>
```

The context manager should ensure that the assembled context does not exceed the configured generation context budget.

Conceptually:
```
Retrieved Parents
       ↓
Calculate context size
       ↓
Within budget?
   /          \
 Yes           No
  |             |
  ↓             ↓
Use all     Reduce context
            according to
            ranking/order
```

The important point is that **context truncation happens here**, **not during parent expansion**.

Parent expansion retrieves the complete bounded parent; Context Management decides what can actually be passed to the LLM.

---

### Context Block Structure

Each context block should retain provenance.

Example conceptual structure:
```
[Context 1]
Document: architecture.md
Parent ID: P4
Matched Child IDs: C12, C19

<full parent content>

[Context 2]
Document: retrieval.md
Parent ID: P2
Matched Child IDs: C05

<full parent content>
```

This allows the generation layer to understand where each piece of context originated and supports debugging/evaluation.

---

### Responsibility Boundary

**Context Management does**
- Deduplicate parents.
- Preserve retrieval ordering.
- Apply context-size limits.
- Construct context blocks.
- Preserve provenance.
- Produce the final context payload.

**Context Management does not**
- Perform BM25 retrieval.
- Perform dense retrieval.
- Perform RRF fusion.
- Perform reranking.
- Decide whether retrieval is required.
- Generate the final answer.
- Invent or modify retrieved evidence.

---

### Context Selection Strategy

What happens when expanded parent contexts exceed the LLM's context budget.

```
Final child results
       ↓
Parent expansion
       ↓
Context selection
       ↓
Context ordering
       ↓
Context assembly
       ↓
LLM
```

**Rank-based context selection**

After retrieval, we may have several parent chunks:
```
Parent P1 → Rank 1
Parent P2 → Rank 2
Parent P3 → Rank 3
Parent P4 → Rank 4
```

We give the LLM a maximum context size, for example 4,000 tokens.

The system simply:
1. Takes the highest-ranked parent first.
2. Adds the next highest-ranked parent.
3. Continues until adding another parent would exceed the context limit.
4. Stops there.

```
P1 → P2 → P3 → P4
↓     ↓     ↓
Add   Add   Add
             ↓
       Budget reached
             ↓
        Send P1,P2,P3
```

Key idea:

**`Higher-ranked parents get priority`**

We don't summarize, compress, or cut parents apart. We select the most relevant complete parent chunks that fit within the context budget.

---
