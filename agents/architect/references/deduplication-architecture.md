# Deduplication Architecture Reference

Use this reference when the architect must design entity resolution, fuzzy matching, duplicate detection, golden record strategy, or embedding-based matching. This is architecture guidance, not implementation code.

## Approach Selection

### Rule-Based Matching

Methods:
- Exact comparison.
- Regex normalization.
- Levenshtein distance.
- Jaro-Winkler similarity.
- Soundex or phonetic comparison.
- Identifier matching.

Best for:
- Structured fields with known formats.
- Simple matching.
- Cost-sensitive workflows.
- High explainability.

Limitations:
- Weak semantic understanding.
- Brittle with paraphrasing.
- Hard to tune across languages or noisy text.

### Embedding-Based Matching

Approach:
1. Normalize candidate text.
2. Generate embeddings.
3. Build nearest-neighbor index.
4. Retrieve candidate neighbors.
5. Score with cosine similarity.
6. Apply threshold and review policy.

Models:
- text-embedding-3-small: default cost-efficient option.
- text-embedding-3-large: high-accuracy option when false decisions are costly.

Algorithm by scale:
- Under 50K records: sklearn brute-force or exact nearest neighbors.
- 50K to 10M records: FAISS IVF or similar approximate index.
- Over 10M records: distributed Spark plus approximate nearest neighbor index and staged candidate generation.

Default threshold:
- Start at 0.85 cosine similarity.
- Tune using labeled samples.
- Never ship threshold without rationale.

### Hybrid Matching

Recommended default:
- Rule-based blocking first.
- Embedding-based scoring inside blocks.
- Optional final deterministic checks for high-risk merges.

Benefits:
- Reduces O(n^2) pair explosion.
- Controls embedding cost.
- Improves explainability.
- Keeps semantic matching for ambiguous pairs.

## ADRs To Produce

- Matching approach: rule-based vs embedding vs hybrid.
- Embedding model selection.
- Index algorithm selection.
- Threshold strategy.
- Golden record survivorship policy.
- Incremental vs batch matching.
- Audit and split/undo policy.

## Threshold Tuning

Minimum labeled sample:
- 100 true matches.
- 100 true non-matches.

Process:
1. Score pairs from 0.70 to 0.95 thresholds.
2. Plot precision, recall, and F1.
3. Pick F1-optimal threshold as a starting point.
4. Adjust for business risk.

Risk stance:
- High-stakes merge: favor precision.
- Discovery/review queue: favor recall.
- Automated merge: require higher confidence plus deterministic guardrails.

Document:
- Sample size.
- Labeling method.
- Precision target.
- Recall target.
- Chosen threshold.
- Review threshold range.

## Scalability Architecture

Under 50K records:
- In-memory index is acceptable.
- Real-time matching may be feasible.
- Exact nearest neighbor is simple and explainable.

50K to 10M records:
- Batch processing recommended.
- Partition by stable blocking keys.
- Use FAISS IVF or equivalent.
- Persist embeddings and index metadata.

Over 10M records:
- Distributed processing.
- Approximate index.
- Multi-stage candidate generation.
- Periodic full rebuild plus incremental updates.

Incremental pattern:
1. Persist embeddings in a governed table.
2. Hash normalized content to avoid recomputation.
3. Match new records against existing index.
4. Queue low-confidence candidates for review.
5. Rebuild index periodically.

## Cost Architecture

Controls:
- Cache embeddings by content SHA-256.
- Batch API calls.
- Skip unchanged records.
- Use cheaper model for broad candidate generation.
- Use higher-accuracy model for final ambiguous cases only.

Budget formula:

```text
total_cost = (unique_records * avg_tokens * price_per_token) + (queries * price_per_query)
```

Architecture should include:
- Expected record count.
- Average text length.
- Refresh frequency.
- Rebuild frequency.
- Review queue volume.
- Cost alert threshold.

## Composite Matching

### Single-Field Embedding

Best for simple name-only or title-only matching.

Pros:
- Simple.
- Cheap.
- Easy to explain.

Cons:
- Less control.
- Weak when important context spans multiple fields.

### Concatenated Multi-Field Embedding

Combine fields into one string.

Pros:
- Simple architecture.
- Captures context.

Cons:
- Field importance is implicit.
- Hard to explain partial matches.

### Per-Field Weighted Similarity

Recommended when fields differ in reliability.

Example:

```text
name_weight = 0.4
address_weight = 0.3
identifier_weight = 0.2
other_weight = 0.1
```

Pros:
- Tunable.
- Explainable.
- Works with field-specific thresholds.

Cons:
- More design and tuning effort.

### Ensemble Decision

Use separate scores and a decision model.

Best for:
- High-volume systems.
- Regulated decisions.
- Cases needing fine-grained explainability.

## Golden Record Architecture

Survivorship strategies:
- Most complete record.
- Most recent record.
- Source priority.
- Field-level merge.
- Human-reviewed merge.

Physical strategies:
- Merge: one canonical record remains.
- Link: records stay separate with a duplicate group id.
- Hybrid: link first, merge after review.

Cluster handling:
- Use transitive closure for groups larger than two.
- Avoid automatic merge cascades without confidence review.
- Store all source record ids.

Undo/split:
- Required for incorrect merges.
- Preserve source records or reversible change log.
- Track actor, timestamp, reason, confidence, and evidence.

## Audit And Explainability

Every automated or reviewed decision should log:
- Candidate ids.
- Match score.
- Threshold.
- Blocking rule.
- Model/version.
- Field-level scores where available.
- Decision outcome.
- Actor or automation id.
- Timestamp.

Expose explainability to reviewers:
- Why candidates matched.
- Which fields contributed.
- Confidence score.
- Suggested survivorship.
- Risk flags.

## Data Quality Integration

Deduplication contributes to:
- Uniqueness score.
- Completeness score.
- Golden record confidence.
- Source quality scoring.

Quality rules:
- Primary identifiers unique in Silver or Gold depending on domain-knowledge need.
- Duplicate confidence attached as metadata.
- Review queue freshness SLA.
- Merge error rate monitored.

## Architecture Checklist

- Matching approach ADR exists.
- Scale and algorithm selection documented.
- Threshold rationale documented.
- Labeled sample plan defined.
- Golden record policy defined.
- Incremental matching design defined.
- Embedding cache strategy defined.
- Audit and undo/split requirements defined.
