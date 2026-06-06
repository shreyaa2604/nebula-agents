# Data Mapping Patterns

Data mapping converts source fields into target fields under a documented contract. It is a specification first and code second.

## Mapping Specification Format

Use YAML for machine-readable mappings and Markdown for review summaries.

```yaml
mapping: customers.v1
version: 1.0.0
source: source_customers
target: canonical_customers
fields:
  - target: customer_key
    source: id
    transform: trim
    required: true
  - target: full_name
    source:
      - first_name
      - last_name
    transform: join_space
    required: true
  - target: email
    source: email_address
    transform: normalize_email
    required: false
  - target: loaded_at
    source: null
    transform: current_timestamp
    required: true
```

Every target field must have one of:
- Direct source field
- Derived expression
- Lookup rule
- Explicit default
- Explicitly documented omission

## Type Coercion

Define coercion rules in one place and test edge cases.

| Source | Target | Pattern |
|--------|--------|---------|
| string | date | Parse with accepted formats and timezone policy |
| integer | decimal | Convert with scale and rounding policy |
| string | boolean | Map accepted true and false values explicitly |
| bytes | string | Decode with declared encoding |
| string | enum | Validate against allowed values |

Do not silently coerce invalid values to null. Route invalid values through quality failure or quarantine based on severity.

## Semantic Alignment

Sources often use local names. Canonical names should stay stable.

Examples:
- `cust_id` maps to `customer_key`
- `email_addr` maps to `email`
- `sku` maps to `product_key`
- `qty` maps to `quantity`

Record synonym choices in the mapping spec. If a source field is ambiguous, ask for approval rather than guessing.

## Null Handling

Choose a policy per field:

| Policy | Use When |
|--------|----------|
| pass-through | Null is valid and meaningful |
| default | Contract defines a safe default |
| derive | Another field can produce the value |
| reject row | Field is required for the target |
| quarantine row | Field needs review or correction |

Null policy belongs in the contract or mapping spec, not only in code.

## Derived Field Patterns

Concatenation:

```python
def join_space(*parts):
    return " ".join(str(part).strip() for part in parts if str(part or "").strip())
```

Conditional:

```python
def derive_status(row):
    if row.get("deleted_at"):
        return "inactive"
    return "active"
```

Lookup:

```python
def map_category(source_value, lookup):
    key = str(source_value or "").strip().lower()
    if key not in lookup:
        raise MappingError(f"Unknown category: {source_value}")
    return lookup[key]
```

## Schema Drift Detection

At pipeline start, compare actual source schema with expected source schema.

Alert on:
- New fields not used by mapping
- Type changes that are compatible but noteworthy

Block on:
- Removed fields with active mappings
- Renamed fields without alias rules
- Type changes that break coercion
- Required fields that become nullable

## Mapping Versioning

Use semantic versioning:
- Major: target field removed or meaning changed
- Minor: target field added with safe default or optional status
- Patch: implementation fix with same contract behavior

Keep a changelog:

```yaml
changes:
  - version: 1.1.0
    date: 2026-06-06
    summary: Added optional product category mapping.
```

## Validation Checklist

- Every target field is mapped, derived, defaulted, or explicitly omitted.
- Every required target field has a non-null strategy.
- Every type coercion has tests for valid and invalid input.
- Every lookup has handling for unknown values.
- Source schema drift is checked before transformation.
- Mapping version is recorded in pipeline output metadata.
- Mapping tests include normal, edge, and invalid rows.
- Contract and implementation use the same field names.
