# Medallion Architecture Design

## Overview
[Brief description of the data platform and consumers.]

## Layer Assignments

### Bronze Layer (Raw)

| Source | Ingestion Pattern | Format | Frequency | Volume |
|--------|-------------------|--------|-----------|--------|
| [source_1] | [Auto Loader / COPY INTO / CDC] | [JSON/CSV/Parquet] | [real-time/hourly/daily] | [rows/day] |

**Bronze Conventions**
- Append-only and immutable.
- No transformations beyond format conversion to Delta.
- Preserve source payload and ingestion metadata.
- Use rescue column for unknown fields when schema drift is expected.

### Silver Layer (Conformed)

| Table | Source (Bronze) | Transformations | Quality Rules |
|-------|-----------------|-----------------|---------------|
| [silver_table_1] | [bronze_source] | [type cast, dedup, null handling, SCD] | [expect/expect_or_drop] |

**Silver Conventions**
- No domain knowledge: no business joins, aggregations, KPIs, or consumer-specific rules.
- Every transform must answer "Does this require domain knowledge?" with "No."
- Deduplication and SCD may live here when rules are technical/source-change rules.

### Gold Layer (Business-Ready)

| Table | Sources (Silver) | Business Logic | Consumers |
|-------|------------------|----------------|-----------|
| [gold_table_1] | [silver tables] | [aggregation/business rules] | [BI/ML/API] |

**Gold Conventions**
- Domain joins, aggregations, metrics, and business KPIs live here.
- Star schema or feature tables are allowed when consumer-driven.
- Each Gold output needs owner, freshness SLA, and lineage.

## Unity Catalog Hierarchy

```text
metastore: [name] ([region])
  catalog: [domain_1]
    schema: bronze
    schema: silver
    schema: gold
  catalog: [domain_2]
    schema: bronze
    schema: silver
    schema: gold
  catalog: shared
    schema: reference_data
```

## Data Quality SLAs

| Dataset | Freshness | Completeness | Uniqueness |
|---------|-----------|--------------|------------|
| [dataset_1] | [<= 1 hour] | [>= 99.5%] | [100% on primary key] |

## Related ADRs
- ADR-XXXX: [Governance model]
- ADR-XXXX: [Compute strategy]
- ADR-XXXX: [Ingestion patterns]
