# Retrieval Results & Parent Expansion Contract

## 1. Child Result Schema

Both dense and lexical retrievers return ranked child-chunk matches. Each result should carry the same core identifiers and provenance fields, while retaining the retriever’s original score.

```json
{
    "chunk_id": UUID,
    "parent_chunk_id": UUID,
    "document_id": UUID,
    "rank": int,
    "score": float | None
}
```

|Field|	Type	|Description|
|---|---|---|
|`chunk_id`|	UUID	|ID of the matched child chunk|
|`parent_chunk_id`|	UUID	|ID of the parent associated with the child|
|`document_id`|	UUID	|ID of the source document|
|`rank`|	Integer	|Position in that retriever’s ranked results|
|`score`|	Number	|Native score returned by the retriever|
|`score_type`|	String	|Identifies what the native score represents, e.g. bm25_score or similarity_score|

`score_type` prevents downstream components from treating different scoring systems as if they were directly comparable. Keep the original score unchanged at this stage.

---

## 2. How native scores are handled

A **native score** is the score produced directly by the retrieval engine. The two retrievers may use different scoring scales and meanings:
- BM25: typically higher scores indicate stronger lexical relevance.
- TurboVec: returns a similarity score or distance according to the configured index metric. Whether higher or lower is better depends on that metric.

Therefore, RAGnarok should:

1. Preserve each retriever’s native score unchanged in its results.
2. Record its score type and direction (`higher_is_better` or `lower_is_better`).
3. Avoid directly comparing raw BM25 and vector scores as if they were on the same scale.
4. Let the hybrid fusion stage combine the ranked lists using the chosen fusion method. It can use ranks rather than raw scores, or apply a defined score-normalization method if that is the selected design.
5. Retain the original score and score type for debugging, evaluation, and traceability, even after fusion.

**Important**: The common result schema does not require the raw scores to be comparable. It requires each score to be interpretable and traceable to its retrieval source.

---

## 3. Parent expansion contract

**Input**:

Parent expansion receives the selected, ranked child results after retrieval or hybrid fusion. It needs each child’s `chunk_id`, `parent_chunk_id`, `document_id`, and `rank`, while preserving available native score information.

**Processing**

```
Ranked child results
        |
        v
Collect parent_chunk_id values
        |
        v
Deduplicate parent IDs
        |
        v
Fetch full parent records from PostgreSQL
        |
        v
Associate each parent with its matched children
        |
        v
Return parents in best-child-rank order
```

If multiple child matches point to the same parent, fetch that parent once and attach all corresponding child matches.

**Output**

Return one record per unique parent, containing its full text and the child matches that led to its retrieval.


Sample illustration:
```json
[
    {
        "parent_chunk_id": "parent_A",
        "document_id": "document_1",
        "parent_index": 0,
        "heading_path": ["Retrieval", "Dense Search"],
        "content": "Full parent chunk text...",
        "matched_children": [
            {
                "chunk_id": "child_01",
                "rank": 1,
                "score": 0.91,
                "score_type": "similarity_score"
            },
            {
                "chunk_id": "child_02",
                "rank": 2,
                "score": 0.86,
                "score_type": "similarity_score"
            }
        ]
    }
]
```

**Output behavior**

- Order parents by the best-ranked child associated with each parent.
- Preserve each child’s rank and native score in matched_children.
- Do not invent a parent score by averaging or summing child scores.
- If there are no child results, return an empty list.
- If a parent record is missing, omit it from the expanded context and log the integrity issue.

The output provides full parent text for context management, while keeping child-level ranking and native scoring available for traceability.

---

For RAGnarok, lock the metric direction per retriever, not one global direction:

|Retriever|	Native score direction|
|---|---|
|BM25	|Higher is better|
|Dense (TurboVec)	|Depends on configured metric|

**NOTE**: For dense retrieval, choose cosine similarity, where higher is better. This gives both retrievers a higher-is-better convention, but their raw scores still should not be compared directly because they have different scales and meanings.

---

**Contract to lock**:

- `score`: preserve the retriever’s raw score.
- `score_type`: `bm25_score` || `similarity_score`.
- `score_direction`: `higher_is_better`.
- `Hybrid fusion`: combine ranked results through a separately defined fusion method; do not add or average the raw scores by default.
- `Parent expansion`: preserve each child’s score and rank; do not calculate a parent score.

