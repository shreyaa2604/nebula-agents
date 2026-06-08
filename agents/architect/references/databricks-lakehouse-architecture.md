# Databricks Lakehouse Architecture Reference

Use this reference when the architect must design a Databricks lakehouse, medallion model, Unity Catalog hierarchy, Delta Lake storage strategy, compute topology, or Lakeflow/Delta Live Tables pipeline architecture. This is architecture guidance only; implementation code belongs to the data engineering role when that role exists.

## Design Principles

- Start with data products and ownership, then choose catalogs, schemas, tables, and access policies.
- Treat medallion layers as contracts, not folder names.
- Prefer serverless and managed optimization unless a concrete networking, compliance, GPU, or legacy integration constraint requires classic compute.
- Use ADRs for major platform choices: governance model, workspace topology, catalog naming, medallion boundaries, compute strategy, ingestion pattern, SCD strategy, data quality framework, and entity resolution approach.
- Use generic examples such as customers, orders, and products.

## Well-Architected Pillars

| Pillar | Architect Questions |
|--------|---------------------|
| Reliability | Can pipelines retry, replay, recover, and prove data durability? |
| Security | Are identity, catalog grants, row filters, masking, secrets, and audit logs designed? |
| Performance Efficiency | Are storage layout, compute sizing, query patterns, and clustering appropriate? |
| Operational Excellence | Are deployments, monitoring, quality gates, incident workflows, and lineage defined? |
| Cost Optimization | Are serverless defaults, cluster policies, auto-termination, lifecycle, and budgets defined? |

## Medallion Architecture

### Decision Rule

Ask for every transformation: "Does this require domain knowledge?"

- Yes: Gold.
- No: Silver.
- Raw exactly as received: Bronze.

### Bronze Layer

Bronze is raw, immutable, append-only, and optimized for recovery.

Allowed:
- Land records exactly as received.
- Add ingestion metadata such as source file, ingestion timestamp, batch id, and source system.
- Convert files to Delta while preserving source payload.
- Use schema-on-read or rescue columns for unknown fields.

Not allowed:
- Business joins.
- Aggregations.
- Metric definitions.
- Field survivorship decisions.
- Consumer-specific cleanup.

Patterns:
- Auto Loader for cloud file arrival.
- COPY INTO for simple scheduled file loads.
- Structured Streaming for streaming sources.
- CDC ingestion for database changes.
- Partition by ingestion date only when it improves pruning and does not create tiny files.

### Silver Layer

Silver is cleaned, conformed, typed, deduplicated, and still free of domain business logic.

Allowed:
- Type casting and normalization.
- Null handling with documented rules.
- Standardized field names and formats.
- Deduplication using technical keys or generic matching rules.
- SCD Type 1 or Type 2 where the rule is source-change semantics rather than business interpretation.
- Quality expectations such as required identifiers, timestamp parseability, and uniqueness.

Not allowed:
- Cross-domain business joins.
- KPI calculation.
- Business segmentation.
- Consumer-specific aggregates.
- Domain policy decisions.

Health check:
1. Could the same Silver table serve multiple downstream products?
2. Would a non-domain data engineer understand the transformation?
3. Does the transformation avoid business language beyond field descriptions?

### Gold Layer

Gold is business-ready and consumer-oriented.

Allowed:
- Cross-domain joins.
- Star schema or wide analytical models.
- Metrics, KPIs, and business rules.
- Feature tables for machine learning.
- Aggregates and serving tables for BI, APIs, or applications.

Required:
- Named consumer or consumer class.
- Owner for metric definitions.
- Freshness SLA.
- Lineage to Silver sources.
- Contract or semantic definition for shared outputs.

### Anti-Patterns

- Business logic in Silver because it is convenient.
- Raw source payloads exposed as Gold.
- Skipping Silver and building every dashboard from Bronze.
- Layer-first catalog naming that hides ownership.
- Gold tables with no consumer, owner, or freshness SLA.

When medallion degrades, convenience is usually the failure mechanism. Require explicit ADRs for exceptions.

## Unity Catalog Architecture

### Hierarchy

```text
metastore (regional)
  catalog (domain or environment boundary)
    schema (layer, data product, or bounded grouping)
      table | view | volume | function
```

### Metastore

- Use one metastore per region unless legal, residency, or operational isolation requires more.
- Treat the metastore as the top governance container.
- Keep identity, audit, lineage, and grants consistent across workspaces attached to the metastore.

### Catalog Naming

Recommended: domain-first.

```text
sales.bronze.raw_orders
sales.silver.orders
sales.gold.order_metrics
```

Use domain-first when ownership and data products are the main organizing principle.

Environment-first can be acceptable when environment isolation is the primary driver:

```text
prod.sales.orders
staging.sales.orders
```

Avoid layer-first:

```text
bronze.sales.orders
```

Layer-first violates ownership clarity and makes cross-domain governance harder.

### Governance Operating Models

| Model | Best Fit | Risks |
|-------|----------|-------|
| Centralized | Small org, single platform team, maximum consistency | Platform bottleneck |
| Decentralized | Independent domains with strong data skills | Inconsistent standards |
| Federated | Central standards with domain enforcement | Requires governance discipline |
| Hybrid | Critical shared data centralized; domain data delegated | Boundary ambiguity |

Federated is the default recommendation for enterprise-scale work unless the team structure says otherwise.

### Privileges

Design grants from broad to narrow:
- USE CATALOG
- USE SCHEMA
- SELECT
- MODIFY
- CREATE TABLE / CREATE VIEW / CREATE FUNCTION
- EXECUTE for functions

Use groups, not individual users, for durable access control. Separate producer, steward, consumer, and platform administration groups.

### Tagging

Minimum tags:
- pii
- sensitivity
- data_owner
- data_domain
- freshness_sla
- cost_center
- retention_policy

Use tags to drive policy automation where possible: masking, review workflow, lifecycle, quality thresholds, and access request routing.

### Row-Level Security And Masking

- Use row filters when access depends on row attributes.
- Use dynamic column masking for PII or restricted fields.
- Document whether policies are centrally managed or domain-managed.
- Verify policy behavior in non-production before promotion.

### Delta Sharing

Use Delta Sharing when data products cross organization or tenant boundaries. Define:
- Shared tables/views.
- Recipient identity.
- Contract version.
- Allowed usage.
- Revocation process.

## Delta Lake Storage Architecture

### Storage Layout

Prefer liquid clustering for most modern tables when available. Use partitioning only when predicates are stable, cardinality is low enough, and partition pruning materially improves queries.

Use Z-ordering when liquid clustering is unavailable and query predicates have clear high-value columns.

### Schema Enforcement And Evolution

- Enforcement: use for governed Silver and Gold tables.
- Evolution: allow only where source drift is expected and captured safely, usually Bronze.
- Rescue columns: use in Bronze to preserve unrecognized fields.

### Time Travel

Design time travel for audit, recovery, reproducibility, and debugging. Define retention based on compliance, cost, and recovery expectations.

### MERGE Patterns

- Upsert: match on stable key.
- SCD Type 1: overwrite current values.
- SCD Type 2: expire current row and insert new version.
- Soft delete: mark inactive/deleted while preserving lineage.

### Change Data Feed

Enable Change Data Feed when downstream consumers need incremental processing, audit deltas, or propagation without full reloads.

### Maintenance

Define schedules for:
- OPTIMIZE.
- VACUUM.
- Compaction.
- Statistics refresh.
- Deletion vector review.
- Predictive optimization when supported.

## Compute Strategy

Default recommendations:
- Serverless SQL warehouses for BI and ad hoc analytics.
- Serverless jobs or Lakeflow/DLT for managed pipelines.
- Job clusters for scheduled production jobs.
- All-purpose clusters only for interactive exploration or notebooks with clear cost controls.

Choose classic compute only when required by:
- Custom networking.
- GPU workloads.
- Specific compliance controls.
- Legacy libraries or integrations.
- Unsupported serverless feature gaps.

Use cluster policies to enforce:
- Runtime versions.
- Auto-termination.
- Instance families.
- Maximum size.
- Tagging.
- Photon setting where appropriate.

## Workspace Strategy

Split workspaces when there are real isolation drivers:
- Dev/staging/prod separation.
- Regulatory boundary.
- Geographic residency.
- Strong domain isolation.
- Network boundary.
- Operational blast-radius control.

Do not split workspaces only to simulate data ownership; prefer Unity Catalog grants and catalog design first.

Network architecture questions:
- Does the workspace require private connectivity?
- Are source systems reachable without public exposure?
- Are egress controls required?
- How are secrets managed?

## Lakeflow / Delta Live Tables

Use declarative pipelines when lineage, expectations, incremental processing, and operational simplicity matter more than hand-tuned control.

Use imperative jobs when:
- Logic is highly custom.
- Existing framework reuse is required.
- DLT feature support is insufficient.

Quality expectation patterns:
- expect: record metric, do not block.
- expect_or_drop: drop bad records and record evidence.
- expect_or_fail: fail pipeline for critical invariant breaches.

Testing pattern:
- Keep transformations as plain functions where possible.
- Wrap them with DLT decorators in thin pipeline definitions.
- Unit test transform functions.
- Integration test expectations and lineage in a pipeline run.

## Cost Optimization

- Prefer serverless-first for elastic workloads.
- Use auto-termination for interactive compute.
- Use spot instances only for fault-tolerant batch work.
- Archive cold data to cheaper tiers.
- Monitor DBU spend by job, catalog, owner, and cost center.
- Use budgets and alerts.
- Review small-file issues before scaling compute.
- Enable predictive optimization when governance and platform support are ready.

## Architecture Checklist

- Medallion layer assignment documented.
- Domain-knowledge decision rule applied.
- Unity Catalog hierarchy and governance model documented.
- Catalog naming convention ADR exists.
- Ingestion pattern ADR exists.
- Compute strategy ADR exists.
- Data quality expectations defined.
- Freshness SLAs defined.
- Lineage and ownership visible.
- Cost controls defined.
- Security policies testable.
