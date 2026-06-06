# Data Fabric Patterns

Data fabric is a metadata-driven access layer that makes heterogeneous data sources discoverable, governed, and reusable without forcing every source into one physical store.

## Architecture Goals

- Hide source-specific access details behind cataloged data products.
- Preserve lineage from source to consumer.
- Make freshness, quality, ownership, and access rules visible.
- Support both virtual access and materialized datasets.
- Keep source systems loosely coupled from downstream consumers.

## Logical Layers

```
Source systems
  -> connectors
  -> metadata catalog
  -> canonical datasets
  -> quality and lineage
  -> consumer access layer
```

## Metadata Catalog

A catalog entry should be the first artifact created for a reusable dataset.

```yaml
dataset: customers.v1
owner:
  team: data-platform
  contact: data-owner@example.com
description: Canonical customer records from approved sources.
schema:
  - name: customer_key
    type: string
    required: true
  - name: full_name
    type: string
    required: true
  - name: updated_at
    type: timestamp
    required: true
freshness:
  expected_interval: daily
  max_lag_minutes: 1440
quality:
  - field: customer_key
    rule: unique
    severity: critical
lineage:
  upstream:
    - source_customers
  downstream:
    - customer_summary
access:
  classification: internal
  interface: rest
```

## Virtual View Abstraction

Use virtual views when:
- Consumers need current source data.
- Transform cost is low.
- Data volume is manageable.
- Source availability is high.

Use materialized views when:
- Source systems are slow or unreliable.
- Consumers need stable snapshots.
- Transform cost is high.
- Quality gates must run before publish.

## Materialized Refresh Strategies

| Strategy | Use When | Notes |
|----------|----------|-------|
| Full refresh | Dataset is small or source lacks change tracking | Simple but can be expensive |
| Incremental watermark | Source has updated timestamp or monotonic ID | Most common batch pattern |
| Change data capture | Source emits change stream | Good for low-latency sync |
| Snapshot diff | Source provides snapshots only | Requires stable keys and diff logic |
| Event projection | Events are source of truth | Requires ordering and replay controls |

## Data Mesh Domain Ownership

Use data product ownership when several teams publish and consume datasets.

Each data product needs:
- Clear owner and support path
- Contracted schema and freshness
- Quality rules and issue response
- Versioning policy
- Consumer onboarding path

The data engineer implements the pipeline and catalog mechanics; the architect owns domain boundaries and canonical model decisions.

## Multi-Source Query Federation

Federation is useful for exploration and low-volume lookup. It is risky for high-throughput production workflows unless the query planner, credentials, and source SLAs are mature.

Before federating:
- Confirm each source supports expected query volume.
- Define timeout and retry behavior.
- Document which source owns each field.
- Add quality checks for join completeness.
- Add observability for source latency and failures.

## Access Layer API Patterns

Use catalog APIs for metadata:

```http
GET /data-catalog/datasets
GET /data-catalog/datasets/{datasetName}
GET /data-catalog/datasets/{datasetName}/lineage
```

Use data access APIs for records:

```http
GET /data/customers?updatedAfter=2026-06-01T00:00:00Z
GET /data/customers/{customerKey}
```

Keep catalog APIs distinct from data APIs. Catalog endpoints explain what exists; data endpoints deliver the records.

## GraphQL Catalog Pattern

GraphQL can be useful for catalog exploration because consumers often ask nested metadata questions.

```graphql
type Dataset {
  name: String!
  version: String!
  owner: Owner!
  fields: [Field!]!
  qualityRules: [QualityRule!]!
  upstream: [Dataset!]!
  downstream: [Dataset!]!
}
```

Avoid using GraphQL as a default bulk data extraction interface unless pagination, filtering, and cost limits are explicit.

## Data Dictionary Pattern

Every canonical dataset should have field descriptions.

| Field | Type | Required | Description | Source |
|-------|------|----------|-------------|--------|
| customer_key | string | yes | Stable customer identifier | mapping rule |
| full_name | string | yes | Display name after normalization | source |
| updated_at | timestamp | yes | Last source update time | source |

## Data Fabric Design Checklist

- Catalog entry exists before pipeline publish.
- Dataset has owner, version, description, freshness, and quality rules.
- Source adapter responsibilities are separated from transformation logic.
- Virtual and materialized access choices are documented.
- Dataset contract matches published schema.
- Lineage includes upstream sources and downstream consumers.
- Access pattern is clear: API, table, file, stream, or catalog-only.
- Quality scorecards are linked from the catalog.
- Breaking changes require a new major dataset version.
