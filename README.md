
# Data Contract Enforcer

A system for generating, validating, and enforcing machine-checkable data contracts across inter-system boundaries in the TRP1 (Training Round Progression 1) program.

## Overview

Multiple AI/ML systems built across TRP1 weeks produce structured data outputs without explicit contracts governing their schemas, ranges, and cross-system dependencies. This tool makes those implicit contracts **explicit and machine-checkable** using [Bitol v3.0.0](https://bitol-io.github.io/open-data-contract-standard/) data contracts, enabling early detection of schema drift and data integrity violations across system boundaries.

**Covered boundaries:**

| Source System | Target System | Contract |
|---|---|---|
| automaton-auditor (Week 3) | Downstream consumers | `generated_contracts/week3_verdicts.yaml` |
| The-Ledger (Week 5) | Downstream consumers | `generated_contracts/week5_events.yaml` |

## Architecture

```
outputs/week{3,5}/*.jsonl          # Raw JSONL data from TRP1 systems
         │
         ▼
contracts/generator.py             # 4-stage contract generation pipeline
    Stage 1: Load & Flatten        # Normalize nested JSONL payloads
    Stage 2: Column Profiling      # Types, nullability, stats, cardinality
    Stage 3: Bitol Clause Gen      # Domain rules → Bitol contract clauses
    Stage 4: Lineage + LLM + dbt   # Inject lineage, annotate, emit dbt schema
         │
         ▼
generated_contracts/*.yaml         # Bitol v3.0.0 contracts
generated_contracts/*_dbt.yml      # dbt-compatible schema files
schema_snapshots/                  # Timestamped immutable contract history
         │
         ▼
contracts/runner.py                # Contract validation runner
         │
         ▼
validation_reports/*.json          # Validation results (CI/CD gate-compatible)
```

## Quickstart

### Prerequisites

- Python 3.11+
- An `ANTHROPIC_API_KEY` for LLM annotation (optional — falls back to placeholder annotations)

### Install

```bash
# Clone and install
pip install -e .

# Or with uv
uv sync

# Set up environment variables
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Generate Contracts

```bash
# Week 3 — automaton-auditor verdict records
python contracts/generator.py \
  --source outputs/week3/verdicts.jsonl \
  --contract-id week3-automaton-auditor-verdicts \
  --lineage outputs/week4/lineage_snapshots.jsonl \
  --output generated_contracts/

# Week 5 — The-Ledger event records
python contracts/generator.py \
  --source outputs/week5/events.jsonl \
  --contract-id week5-ledger-events \
  --lineage outputs/week4/lineage_snapshots.jsonl \
  --output generated_contracts/
```

### Validate Data Against Contracts

```bash
# Week 3
python contracts/runner.py \
  --source outputs/week3/verdicts.jsonl \
  --contract generated_contracts/week3_verdicts.yaml \
  --output validation_reports/

# Week 5
python contracts/runner.py \
  --source outputs/week5/events.jsonl \
  --contract generated_contracts/week5_events.yaml \
  --output validation_reports/
```

The runner exits with code `1` if any check fails — suitable for use as a CI/CD gate.

## Project Structure

```
data-contract-enforcer/
├── contracts/
│   ├── models.py              # Shared dataclasses (ColumnProfile, ValidationResult, etc.)
│   ├── generator.py           # Contract generation pipeline (594 lines)
│   └── runner.py              # Contract validation runner (790 lines)
├── generated_contracts/       # Output Bitol YAML contracts and dbt schemas
├── schema_snapshots/          # Timestamped contract history per contract-id
├── outputs/
│   ├── week3/verdicts.jsonl   # automaton-auditor verdict records
│   ├── week4/lineage_snapshots.jsonl  # Legacy-Code-Cartographer lineage graph
│   └── week5/events.jsonl     # The-Ledger event records
├── validation_reports/        # JSON validation run results
├── pyproject.toml
└── .env.example
```

## Contract Generation Pipeline

The generator runs four stages:

1. **Load & Flatten** — Reads JSONL, flattens nested dicts (`payload.*`, `metadata.*`) with prefixed keys. List columns are dropped (too variable to profile).

2. **Column Profiling** — Computes null fraction, cardinality, sample values, and numeric statistics (min, max, mean, percentiles at 25/50/75/95/99, stddev) per column.

3. **Bitol Clause Generation** — Maps profiles to contract clauses using domain rules:
   - `confidence`/`score` fields → `0.0–1.0` range with `BREAKING` severity
   - `judicial_score` fields → `1–5` integer bounds
   - `position` fields → `>= 1` (ordering invariants)
   - UUID fields → `format: uuid` with pattern validation
   - Timestamps → ISO 8601 `date-time` format
   - Low-cardinality strings → `enum` enforcement

4. **Lineage + LLM + dbt** — Three parallel enrichments:
   - **Lineage injection**: Reads the Week 4 Cartographer snapshot and injects downstream consumers + breaking fields
   - **LLM annotation**: Calls Claude to describe ambiguous columns (e.g., `reasoning`, `dissent_summary`, `hash`). Gated on ambiguity — deterministic columns are not sent to the LLM
   - **dbt schema**: Emits `schema.yml` with `not_null`, `accepted_values`, and `expression_is_true` tests

## Validation Checks

The runner implements the following checks:

| Check | Type | Description |
|---|---|---|
| `row_count` | dataset | At least 1 record present |
| `required` | column | No null values in required fields |
| `type` | column | Values match declared type (string, integer, number, boolean) |
| `format_uuid` | column | UUID v4 format validation |
| `format_datetime` | column | ISO 8601 datetime format (4 variant support) |
| `enum` | column | Values within declared allowed set |
| `minimum` / `maximum` | column | Numeric bounds |
| `uniqueness` | column | ID fields have no duplicates |
| `monotonic` | cross-field | `global_position` increases globally; `stream_position` increases per `stream_id` |
| `score_consistency` | cross-field | `final_score` within range of component scores; flags missing `dissent_summary` on high variance |
| `temporal_ordering` | cross-field | `occurred_at <= recorded_at` (no clock skew) |

## Validation Results

Latest validation runs against TRP1 data:

| Contract | Checks | Pass | Fail | Warn |
|---|---|---|---|---|
| `week3_verdicts` | 39 | 39 | 0 | 0 |
| `week5_events` | 32 | 32 | 0 | 0 |

## Key Design Decisions

- **LLM is gated on ambiguity** — only columns with non-obvious names are sent to Claude; deterministic rules handle the rest. This keeps contracts reproducible and cheap to regenerate.
- **Run-all checks, exit on FAIL** — all checks run in a single pass; exit code 1 if any FAIL exists. WARNs are informational and require human review.
- **Dual snapshot strategy** — a stable named file (`week3_verdicts.yaml`) tracks the current contract; timestamped snapshots in `schema_snapshots/` track data-driven schema evolution over time. Git tracks intent changes; snapshots track observed data changes.
- **Trust tier design** — built for Tier 1 (same repo, full lineage). Tier 2 (registry integration) and Tier 3 (cross-org webhooks) are explicitly deferred.

## Dependencies

| Package | Purpose |
|---|---|
| `pandas` | Column profiling and data analysis |
| `numpy` | Numeric statistics |
| `pyyaml` | Bitol YAML contract serialization |
| `anthropic` | Claude API for LLM column annotation |
| `python-dotenv` | `.env` file loading |
| `jsonschema` | JSON schema validation utilities |

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | No | Claude API key for LLM annotation. Falls back to domain-aware placeholder annotations if absent. |
