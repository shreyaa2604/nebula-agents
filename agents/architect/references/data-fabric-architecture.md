# Data Fabric Architecture Reference

Use this reference when the architect must design metadata-driven data integration, governance automation, data virtualization, semantic access, or a data fabric operating model. Keep examples generic: customers, orders, products.

## Core Principles

Data fabric is a metadata-driven architecture that connects distributed data assets, automates governance, and improves access without forcing every dataset into one physical store.

Core capabilities:
- Active metadata management.
- Knowledge graph for data assets.
- Automated integration.
- Data virtualization where latency allows.
- Augmented data quality.
- Policy automation.
- Self-service access.

## Active Metadata

Active metadata is metadata that can drive recommendations and automation.

Architectural inputs:
- Technical metadata: schemas, fields, types, partitions, jobs.
- Operational metadata: freshness, volume, failures, quality scores, cost.
- Business metadata: glossary terms, owners, purpose, usage terms.
- Social metadata: usage frequency, endorsements, consumer feedback.
- Security metadata: classification, access history, masking policy.

Design decisions:
- Which metadata is collected automatically?
- Which metadata requires steward approval?
- Which metadata can trigger policy or pipeline automation?
- Which metadata is exposed to consumers?

## Data Asset Knowledge Graph

Use a graph model to connect:
- Dataset -> field.
- Dataset -> owner.
- Dataset -> contract.
- Dataset -> quality rule.
- Dataset -> upstream source.
- Dataset -> downstream consumer.
- Dataset -> glossary term.
- Dataset -> access policy.

Architectural uses:
- Impact analysis before schema change.
- Consumer discovery.
- Lineage review.
- Policy propagation.
- Duplicate asset detection.
- Data product health scoring.

## Data Fabric Layers

```text
DATA CONSUMERS       BI, ML, applications, APIs
SEMANTIC LAYER       metrics, dimensions, definitions
GOVERNANCE LAYER     catalog, policy, lineage, contracts
INTEGRATION LAYER    pipelines, streaming, CDC
STORAGE LAYER        lakehouse, warehouse, operational stores
CONNECTIVITY LAYER   connectors, APIs, files, events
DATA SOURCES         databases, SaaS, streams, files
```

Databricks mapping:

| Fabric Layer | Databricks-Oriented Implementation |
|--------------|------------------------------------|
| Active Metadata | Unity Catalog, system tables, generated tags, quality telemetry |
| Knowledge Graph | Unity Catalog lineage plus custom graph overlay |
| Integration | Lakeflow/DLT, Auto Loader, Structured Streaming, Jobs |
| Virtualization | Lakehouse Federation, Delta Sharing, views |
| Quality | DLT expectations and quality score tables |
| Policy Automation | Unity Catalog grants, row filters, column masks, auto-classification |
| Self-Service | SQL warehouses, marketplace views, AI/BI, shared catalogs |

## Data Fabric Vs Data Mesh

Data fabric is infrastructure and automation. Data mesh is an operating model for domain ownership.

| Criterion | Data Fabric | Data Mesh | Combined |
|-----------|-------------|-----------|----------|
| Philosophy | Technology-driven automation | Organization-driven ownership | Fabric enables mesh |
| Best For | Unified access across many sources | Strong domain teams | Enterprise-scale |
| Governance | Centralized or automated | Federated and domain-owned | Federated on shared platform |
| Primary Artifact | Metadata/catalog/policy layer | Domain data product | Data product in governed catalog |

Use data fabric alone when:
- A central platform team owns integration.
- Sources are heterogeneous.
- Compliance and visibility are primary.
- Domain teams are not ready to operate data products.

Use data mesh alone when:
- Domain teams have mature data engineering skills.
- Organizational ownership is the main problem.
- Shared infrastructure already exists.

Combine them when:
- Enterprise scale requires domain ownership and shared governance.
- Domains publish data products.
- A central platform provides catalog, policy, lineage, contracts, and self-service tooling.

## Weighted Decision Matrix

| Criterion | Weight | Fabric | Mesh | Combined |
|-----------|--------|--------|------|----------|
| Source heterogeneity | 20% | 5 | 3 | 5 |
| Domain team maturity | 20% | 3 | 5 | 5 |
| Governance consistency | 20% | 5 | 3 | 5 |
| Time-to-value | 15% | 4 | 3 | 3 |
| Operating complexity | 15% | 4 | 3 | 2 |
| Scalability of ownership | 10% | 3 | 5 | 5 |

Adjust weights per product context and record the final choice in an ADR.

## Implementation Patterns

### Metadata Catalog

Minimum design:
- Auto-discover schemas.
- Attach owner and steward.
- Attach classification and sensitivity.
- Link to contracts.
- Expose quality, freshness, and usage.
- Track upstream and downstream lineage.

### Self-Service Access

Patterns:
- Searchable catalog.
- Data product marketplace.
- Approval workflow for restricted datasets.
- Semantic search over glossary and dataset descriptions.
- Standard access request route.

### Metadata-Driven Pipeline Generation

Use metadata to scaffold pipelines when sources are repetitive.

Flow:
1. Infer schema.
2. Classify fields.
3. Generate Bronze ingestion.
4. Propose Silver typing and quality rules.
5. Require review before activation.

Do not automate business logic into Gold without owner review.

### Governance Policy Automation

Examples:
- pii=true triggers masking.
- sensitivity=restricted requires approval workflow.
- freshness_sla=PT1H creates freshness monitoring.
- retention_policy=short_lived drives lifecycle.
- data_owner missing blocks publication.

## Data Contracts

Use:
- OpenAPI for synchronous request/response APIs.
- ODCS v3.x for datasets, batch feeds, exports, and analytical tables.
- AsyncAPI for events and streams.

ODCS contract essentials:
- Dataset identity and version.
- Owner and contact.
- Schema fields.
- Quality rules.
- Freshness SLA.
- Terms of use.
- Classification.
- Lineage.
- Server/location metadata without secrets.

Lifecycle:
1. Draft.
2. Review.
3. Active.
4. Deprecated.
5. Retired.

Contract testing:
- Validate schema compatibility.
- Validate quality rules.
- Validate freshness.
- Validate classification and masking.
- Validate consumer impact before breaking changes.

## Semantic Layer Architecture

The semantic layer defines business-ready metrics and dimensions.

Design:
- Metric owner.
- Formula.
- Grain.
- Dimensions.
- Freshness.
- Version.
- Downstream consumers.

Rules:
- Avoid duplicate KPI definitions.
- Version metric changes.
- Link metrics to Gold datasets.
- Keep dimensions consistent across products.
- Make stale or unowned metrics fail review.

## Architecture Checklist

- Active metadata sources defined.
- Data asset graph model defined.
- Integration and virtualization boundaries documented.
- Fabric-vs-mesh operating model chosen by ADR.
- ODCS contract lifecycle defined.
- Quality and freshness automation specified.
- Semantic layer ownership documented.
- Policy automation rules testable.
