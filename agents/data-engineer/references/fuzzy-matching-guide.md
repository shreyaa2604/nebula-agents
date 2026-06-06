# Fuzzy Matching Guide

Fuzzy matching resolves records that refer to the same real-world entity but differ in spelling, formatting, abbreviations, missing fields, or source conventions.

## Why Embeddings Beat Character Matching

Traditional fuzzy matching compares strings. It works well for typos but poorly for semantic equivalence. GPT-based embeddings compare meaning in vector space, so related phrases can match even when the characters are different.

Use embeddings for:
- Name and description similarity
- Multi-field entity matching
- Abbreviation expansion
- Records with inconsistent word order
- Records where semantic meaning matters more than exact spelling

Use traditional matching for:
- Cheap pre-filtering
- Exact identifiers
- Short codes
- Phone numbers and normalized emails
- Explaining simple edit-distance differences

## Model Selection

| Model | Use When | Tradeoff |
|-------|----------|----------|
| `text-embedding-3-small` | Default cost-efficient matching | Good quality for most records |
| `text-embedding-3-large` | Accuracy must improve and budget allows it | Higher cost and larger vectors |
| `text-embedding-ada-002` | Existing legacy index compatibility | Prefer newer models for new work |

Always tie an index to model name, normalization version, and dataset version. Changing any of those requires a new index version or controlled rebuild.

## Algorithm Selection

| Data Size | Algorithm | Notes |
|-----------|-----------|-------|
| Less than 50K records | sklearn `NearestNeighbors`, brute-force cosine | Simple, exact, easy to test |
| 50K to 10M records | FAISS IVF-Flat or IVF-PQ | Fast large-scale search |
| More than 10M records | Annoy or HNSW approximate search | Tune recall vs latency |

Start simple. Move to approximate search only when exact search is too slow or memory heavy.

## Threshold Tuning

Do not choose a threshold by intuition alone. Use labeled pairs.

1. Create a labeled sample with match and non-match pairs.
2. Generate candidate pairs and scores.
3. Sweep thresholds such as 0.70 to 0.95.
4. Compute precision and recall at each threshold.
5. Choose match, review, and no-match bands.
6. Document the threshold rationale.

Example bands:
- `>= 0.90`: auto-match
- `0.80` to `0.90`: human review
- `< 0.80`: no match

## Multi-Field Composite Matching

Concatenation is simple:

```python
text = f"name: {name} | address: {address} | email: {email}"
```

Weighted per-field scoring is more explainable:

```python
score = (
    0.50 * name_similarity
    + 0.30 * address_similarity
    + 0.20 * email_similarity
)
```

Use concatenation for fast prototypes. Use weighted fields when reviewers need to understand why a record matched.

## Blocking and Partitioning

Blocking reduces candidate space before expensive vector comparison.

Useful blocking keys:
- Country or region
- First letter of normalized name
- Postal code prefix
- Product category
- Source system
- Date bucket

Blocking must not remove true matches. Prefer loose blocks and review recall on labeled samples.

## Incremental Matching

For new records:
1. Normalize and hash match text.
2. Reuse cached embedding when hash is unchanged.
3. Query against the persisted target index.
4. Store match decisions and candidate scores.
5. Add confirmed new records to the index.
6. Rebuild the full index on a scheduled cadence or after model changes.

## Golden Record Resolution

After duplicates form a cluster, choose a canonical record with documented rules.

Common rules:
- Prefer verified values over unverified values.
- Prefer most recently updated values for mutable fields.
- Prefer source priority order for conflicting fields.
- Preserve all source identifiers in a cross-reference table.
- Store merge rationale and reviewer identity when human review occurred.

## Audit and Explainability

Every match decision should include:
- Source record key
- Target record key or cluster key
- Similarity score
- Threshold
- Decision: match, no-match, or review
- Model name
- Normalization version
- Input hash
- Timestamp and run ID
- Reviewer state when applicable

## Testing Patterns

Use known-pair fixtures:

```yaml
pairs:
  - left: {name: "Acme Incorporated", city: "Boston"}
    right: {name: "ACME Inc", city: "Boston"}
    match: true
  - left: {name: "Northwind", city: "Denver"}
    right: {name: "Southwind", city: "Denver"}
    match: false
```

Regression tests should fail when precision or recall drops below agreed thresholds. Store scores from the prior run so a threshold change is visible.

## Cost Optimization

- Cache embeddings by SHA-256 of normalized text.
- Batch calls up to the provider limit.
- Embed only fields that contribute to matching.
- Use cheaper models for broad candidate generation.
- Use a stronger model only for ambiguous review-band records.
- Track cost per run and per dataset.

## Failure Modes

Low recall means true matches are missed. Lower thresholds carefully, improve normalization, add fields, or reduce overly strict blocking.

Low precision means false matches are accepted. Raise thresholds, add blocking, require multiple signals, or move ambiguous records to review.

Poor explainability means reviewers cannot trust the result. Add field-level scores, show normalized text, and store decision rationale.
