# Data Engineer Code Patterns

Implementation patterns for data pipelines, embedding-based matching, quality checks, and safe loads. Examples use generic `customers`, `orders`, and `products` datasets only as placeholders.

## Embedding Generation With Batching

Prefer batch embedding calls and deterministic text normalization. Store the model, normalized text hash, and vector together so unchanged records never pay for a second embedding call.

```python
from __future__ import annotations

import hashlib
from dataclasses import dataclass
from typing import Iterable, Protocol


class EmbeddingClient(Protocol):
    def embed(self, inputs: list[str], model: str) -> list[list[float]]:
        ...


@dataclass(frozen=True)
class EmbeddingRecord:
    key: str
    text_hash: str
    model: str
    vector: list[float]


def normalize_for_embedding(values: Iterable[object]) -> str:
    parts = [str(value or "").strip().lower() for value in values]
    return " | ".join(part.replace("\n", " ") for part in parts if part)


def content_hash(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def embed_missing(records, cache, client: EmbeddingClient, model="text-embedding-3-small"):
    pending = []
    for record in records:
        text = normalize_for_embedding(record.match_values)
        digest = content_hash(text)
        if cache.get(record.key, digest, model) is None:
            pending.append((record.key, digest, text))

    for chunk_start in range(0, len(pending), 512):
        chunk = pending[chunk_start : chunk_start + 512]
        vectors = client.embed([item[2] for item in chunk], model=model)
        for (key, digest, _), vector in zip(chunk, vectors):
            cache.put(EmbeddingRecord(key=key, text_hash=digest, model=model, vector=vector))
```

## sklearn NearestNeighbors

Use sklearn brute-force cosine matching for small and medium candidate sets. Convert cosine distance to similarity with `1 - distance`.

```python
import numpy as np
from sklearn.neighbors import NearestNeighbors


def nearest_matches(source_vectors, target_vectors, k=3):
    target_matrix = np.asarray(target_vectors, dtype="float32")
    source_matrix = np.asarray(source_vectors, dtype="float32")
    index = NearestNeighbors(n_neighbors=k, metric="cosine", algorithm="brute")
    index.fit(target_matrix)
    distances, indices = index.kneighbors(source_matrix)
    similarities = 1.0 - distances
    return similarities, indices
```

## FAISS Index Creation

Use FAISS for larger data where exact brute force is too slow. Normalize vectors before inner-product search to approximate cosine similarity.

```python
import faiss
import numpy as np


def build_faiss_index(vectors: list[list[float]]):
    matrix = np.asarray(vectors, dtype="float32")
    faiss.normalize_L2(matrix)
    dimension = matrix.shape[1]
    index = faiss.IndexFlatIP(dimension)
    index.add(matrix)
    return index


def search_faiss(index, query_vectors, k=5):
    query = np.asarray(query_vectors, dtype="float32")
    faiss.normalize_L2(query)
    scores, ids = index.search(query, k)
    return scores, ids
```

## Pipeline Idempotency

Every load should answer: what uniquely identifies this output row, and what happens when the same input is processed twice?

Use these controls:
- Natural key or stable synthetic key
- Content hash of normalized source payload
- High-watermark for incremental extraction
- Merge or upsert instead of append-only inserts
- Run ID for audit, not for uniqueness

```sql
MERGE INTO target_customers AS target
USING staged_customers AS source
ON target.customer_key = source.customer_key
WHEN MATCHED AND target.content_hash <> source.content_hash THEN
  UPDATE SET
    full_name = source.full_name,
    email = source.email,
    content_hash = source.content_hash,
    updated_at = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
  INSERT (customer_key, full_name, email, content_hash, created_at)
  VALUES (source.customer_key, source.full_name, source.email, source.content_hash, CURRENT_TIMESTAMP);
```

## ETL Connector Patterns

Database connectors should page by deterministic keys, not by offset when tables are large or changing.

```python
def extract_incremental(conn, watermark):
    sql = """
    select id, updated_at, payload
    from source_records
    where updated_at > :watermark
    order by updated_at, id
    limit :limit
    """
    return conn.fetch_all(sql, {"watermark": watermark, "limit": 1000})
```

API connectors should handle pagination, retries, and checkpointing.

```python
def extract_api_pages(client, first_cursor=None):
    cursor = first_cursor
    while True:
        page = client.get_records(cursor=cursor, limit=500)
        yield from page.items
        if not page.next_cursor:
            break
        cursor = page.next_cursor
```

File connectors should record file path, size, modified time, checksum, parser version, and row count. Treat a parsed file as immutable once accepted.

## Data Quality Checks

Quality rules should be structured data, not scattered assertions. A minimal rule has dataset, field, expectation, severity, and threshold.

```python
def check_not_null(rows, field, max_null_rate=0.0):
    total = len(rows)
    nulls = sum(1 for row in rows if row.get(field) in (None, ""))
    rate = nulls / total if total else 0
    return {
        "expectation": "not_null",
        "field": field,
        "passed": rate <= max_null_rate,
        "observed": rate,
        "threshold": max_null_rate,
    }
```

Run checks at ingress before transformation and at egress before publish. Ingress checks protect transform assumptions; egress checks protect consumers.

## Dead-Letter Queue Pattern

Rows that fail parsing or validation should move to quarantine with enough context for replay.

Capture:
- Source identifier
- Source payload or safe excerpt
- Parser version
- Rule or exception name
- Error message
- Run ID and timestamp
- Retry count

Do not drop bad rows silently. Do not let one malformed row fail a whole batch unless the contract marks that error as critical.

## Mapping Test Pattern

Test mapping rules with small, deterministic samples.

```python
def test_customer_mapping_normalizes_email():
    source = {"cust_id": "C-1", "email_addr": "  USER@example.COM "}
    mapped = map_customer(source)
    assert mapped["customer_key"] == "C-1"
    assert mapped["email"] == "user@example.com"
```

## Dedup Accuracy Test Pattern

Use labeled pairs for threshold validation.

```python
def precision_recall(results):
    true_positive = sum(r.actual_match and r.predicted_match for r in results)
    false_positive = sum((not r.actual_match) and r.predicted_match for r in results)
    false_negative = sum(r.actual_match and (not r.predicted_match) for r in results)
    precision = true_positive / (true_positive + false_positive or 1)
    recall = true_positive / (true_positive + false_negative or 1)
    return precision, recall
```

## Upsert Pattern Selection

Use database-native merge when available. Otherwise use staged tables plus transactionally ordered update and insert statements. Avoid row-by-row writes except for very small maintenance tasks.

| Store Type | Preferred Pattern |
|------------|-------------------|
| PostgreSQL | `insert ... on conflict ... do update` |
| SQL Server | staged table plus guarded `MERGE` |
| Warehouse | staged table plus set-based merge |
| Document store | idempotent replace by stable document key |
| Object storage | immutable file path plus manifest |

## Operational Checklist

- Pipeline has a stable run ID.
- Each step records input count, output count, rejected count, and duration.
- Re-runs with the same input produce identical target state.
- Watermark advances only after successful publish.
- Quality failures are stored with severity and owner.
- Embedding calls are batched and cached.
- Index version is tied to model and source dataset version.
- Tests cover mapping, idempotency, quality rules, and matching thresholds.
