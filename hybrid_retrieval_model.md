# Hybrid Retrieval Model

### 1. Purpose

> The Hybrid Retrieval Model combines lexical retrieval (BM25) and dense vector retrieval (TurboVec) to retrieve child chunks using complementary matching signals. BM25 identifies lexical matches, while dense retrieval finds semantically similar content. Their ranked result lists are merged using Reciprocal Rank Fusion (RRF) to produce a single ranked candidate list.

A configurable reranking stage follows fusion. When enabled, a cross-encoder re-evaluates the query–chunk relevance and reorders the fused candidates. When disabled, the system uses the RRF ranking directly.

---

### 2. Retrieval and Fusion Flow

```
                         User Query
                             |
                  Query processing
                             |
              +--------------+--------------+
              |                             |
              v                             v
      BM25 lexical retrieval        Dense vector retrieval
              |                             |
              v                             v
       Ranked BM25 results           Ranked dense results
              |                             |
              +--------------+--------------+
                             |
                             v
                       RRF Fusion
                             |
                             v
                 Fused ranked child chunks
                             |
                             v
                  Select top-N candidates
                             |
                             v
                   Is reranking enabled?
                       /           \
                     Yes            No
                      |              |
                      v              |
             Cross-encoder           |
               reranking             |
                      |              |
                      v              |
             Reranked candidates     |
                      |              |
                      +------+-------+
                             |
                             v
                    Select final top-K
                             |
                             v
                     Parent expansion
                             |
                             v
                     Context assembly
                             |
                             v
                        LLM answer
```

---

### 3. RRF Fusion

RRF combines ranked lists using the rank positions of each child chunk. It does not add or normalize native BM25 and dense scores.

For a child chunk $d$, the RRF score is:

$$
\operatorname{RRF}(d)=
\sum_{r \in \{\text{BM25},\text{Dense}\}}
\frac{1}{k+\operatorname{rank}_r(d)}
$$

Where:
- $d$ is a child chunk.
- $r$ is one of the retrieval result lists.
- $\operatorname{rank}_r(d)$ is the chunk’s one-based rank in that list.
- $k$ is the configurable RRF rank constant.
- If a chunk is absent from a result list, that list contributes zero to its fusion score.

The fused list is ordered by **descending RRF score**. Duplicate chunk IDs across the two input lists are merged into one fused result, retaining their ranks and native scores from each source where available.

---

### 4. Candidate and Result Contract

**4.1 Input**

The fusion stage receives ranked results from both retrievers. Each input result identifies a child chunk and carries its native score and rank.

**4.2 Fused output**

Each fused result should retain the following fields:

|Field	|Description|
|---|---|
|`chunk_id`|Unique child-chunk identifier|
|`parent_chunk_id`|Parent reference used for later expansion|
|`document_id`|Source document identifier|
|`bm25_rank`|Rank in the BM25 result list, if present|
|`dense_rank`|Rank in the dense result list, if present|
|`bm25_score`|Original BM25 score, if present|
|`dense_score`|Original dense score, if present|
|`fusion_score`|RRF-computed fusion score|
|`fusion_rank`|Position in the fused result list|

The native BM25 and dense scores are retained for provenance, debugging, and evaluation. They are not directly compared or combined as raw values. The `fusion_score` is a separate ranking signal, not a probability or a native retriever score.

---

### 5. Configurable Reranking

**5.1 Decision**

Reranking is included in RAGnarok as a configurable stage after RRF fusion and before parent expansion. This allows the system to run with RRF alone or with a cross-encoder reranker, depending on configuration.
RRF and reranking serve different purposes. 

RRF merges the BM25 and dense rankings using rank positions. A reranker evaluates each candidate against the query and can change the ordering based on its query–chunk relevance score.

**5.2 Reranker behavior**

When reranking is enabled:
1. Take the top-N candidates from the RRF-fused list.
2. Score each candidate using the configured cross-encoder model and the query.
3. Sort candidates by descending reranker score, assuming the selected model uses higher scores for greater relevance.
4. Retain the final top-K child chunks for parent expansion.

When reranking is disabled, skip cross-encoder inference and retain the RRF order for final top-K selection.

**5.3 Configuration**

|Setting	|Purpose|
|---|---|
|`reranker.enabled`	|Enables or disables reranking|
|`reranker.model`	|Selects the cross-encoder model|
|`reranker.top_n`	|Number of RRF candidates sent to the reranker|
|`reranker.top_k`	|Number of final child chunks retained after reranking|

`top_n` should be greater than or equal to `top_k`. Exact defaults and candidate counts will be finalized through benchmarking.

**5.4 Score handling**

- Preserve BM25 and dense native scores and ranks unchanged.
- Keep the RRF `fusion_score` and `fusion_rank` as the fusion-stage output.
- If reranking is enabled, store its output separately as `rerank_score` and assign `rerank_rank`.
- Do not compare raw BM25, dense, RRF, and reranker scores as though they share a common scale.
- Parent expansion does not calculate or assign a parent-level score; it preserves the child result’s ranking and score provenance.

---

**6. Parent Expansion Integration**

After the final child-chunk list is selected, the parent expansion stage uses each child’s `parent_chunk_id` to fetch the full parent text from PostgreSQL.

- Preserve the final child order: RRF order when reranking is disabled, or reranked order when enabled.
- Deduplicate parent IDs and fetch each distinct parent once.
- Retain each child’s retrieval provenance, including its source ranks and scores.
- Do not calculate an averaged, summed, or otherwise invented parent score.

The hybrid layer is responsible for fusion and final child ranking. Parent expansion is a downstream operation that retrieves broader context for those ranked child matches.

---

| Configuration | Flow |
|---------------|------|
| Hybrid without reranking | BM25 + dense → RRF → final top-K |
| Hybrid with reranking | BM25 + dense → RRF → top-N → cross-encoder → final top-K |