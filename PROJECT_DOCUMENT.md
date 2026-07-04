# NL-to-SQL Lakehouse Project Document

**Prepared For:** Sai Veda  
**GitHub Publishing Account:** Nikeshk834  
**Repository Slug:** `06-nl-to-sql-lakehouse`  
**Verified Test Count From Portfolio Index:** 36  

## Background

A DuckDB **lakehouse** over a partitioned-Parquet star schema, with a **semantic
layer** and a deterministic, **offline text-to-SQL** engine. Ask a business
question in English — *"top 5 product categories by revenue"*, *"which region
has the lowest revenue?"*, *"monthly revenue in 2023"* — and the system links
schema entities, compiles validated SQL through the semantic layer, and executes
it out-of-core against 50M fact rows.

No paid LLM, no API keys, fully seeded and reproducible. The NL engine sits
behind an interface so a real LLM can drop in without touching the validation or
execution path.

## Highlights (real measured numbers)

| What | Measured |
|---|---|
| Fact rows built | **50,000,000** (partitioned Parquet) |
| Fact Parquet size | **1.42 GB** (snappy), 28.3 bytes/row |
| Generation throughput | **1.09M rows/s** (45.8 s, memory-bounded, chunked) |
| Full-scan aggregate @ 50M | **196 ms** (cold), 196 ms (warm) |
| Filtered join (year=2023) @ 50M | **196 ms** |
| GROUP BY + join @ 50M | **1.51 s** |
| `COUNT(DISTINCT customer_key)` (2M distinct) @ 50M | **2.31 s** |
| NL→SQL **execution accuracy** | **26/26 = 100%** (per-category all 100%) |
| SQL validated before execution | **26/26** |
| 1B-row extrapolation | **~28 GB** Parquet (measured 28.3 bytes/row) |

> Honest scale: **50M rows actually built and queried**; the generator is
> memory-bounded and parameterized by `--rows` up to 1B (see
> [ARCHITECTURE.md](ARCHITECTURE.md)). 1B numbers are extrapolations from the
> measured per-row footprint, clearly labeled as such.

## Project Purpose

This repository is part of the AI engineering portfolio and focuses on the following problem space:

- DuckDB star schema + semantic layer + text-to-SQL
- Headline result from the portfolio index: **50M-row** lakehouse; **100%** NL→SQL execution accuracy

## What This Project Solves

This project provides a production-style implementation with benchmark evidence and operational checks committed into the repository.

## Technical Approach

```
NL question
   │
   ▼  schema linking  (synonyms → metrics / dimensions / filter values)
LinkResult ─────────────┐
   │  intent/slots       │  (top-N, order direction, limit, year)
   ▼                     │
Query (semantic IR)  ◄───┘   metrics=[...] dims=[...] filters=[...] order/limit
   │
   ▼  semantic layer  (auto-resolves fact→dim joins from the star schema)
SQL string
   │
   ▼  validator  (SELECT-only · no DDL/DML · known columns · live EXPLAIN bind)
validated SQL
   │
   ▼  DuckDB  (out-of-core scan over partitioned Parquet)
answer
```

- **Star-schema lakehouse** — `fact_sales` (50M rows) + `dim_customer`,
  `dim_product`, `dim_store`, `dim_date`. Written to Hive-partitioned Parquet
  (`year=YYYY/`), one file per partition, and queried directly (no load step).
- **Semantic layer** ([`semantic_model.yaml`](src/nlsql/semantic_model.yaml)) —
  business metrics (`revenue`, `gross_margin`, `aov`, `active_customers`, …) and
  dimensions mapped to SQL. Joins are resolved automatically: a metric on the
  fact + a dimension on `dim_store` produces the `store_key` join for you.
- **NL-to-SQL engine** — deterministic schema-linking + intent/slot parser
  (`entity / metric / timegrain / filter` extraction) behind the
  `NLToSQLEngine` interface. `LLMNLToSQL` marks the drop-in seam for a real LLM;
  it would reuse the identical validator + execution path.
- **SQL validation** — every statement is checked for shape (single, read-only),
  blocked verbs (`DROP/DELETE/UPDATE/INSERT/ALTER/CREATE/COPY/PRAGMA/ATTACH/…`,
  including CTE-smuggled mutations), column existence against the model catalog,
  and an optional live DuckDB `EXPLAIN` bind — **before** it touches data.

## Benchmark And Validation Evidence

The portfolio root documents **36 passing tests** for this project, and the repo quickstart uses `make test` as the standard validation path. The benchmark outputs committed in `benchmarks/` and the generated visuals in `assets/` are the evidence package for this delivery.



## Visual Artifacts Reviewed

- `assets/star_schema.png`: Star schema (data model).
- `assets/accuracy_by_category.png`: NL→SQL execution accuracy by category.
- `assets/latency_vs_scale.png`: Query latency vs data scale (log-log).
- `assets/analytics_dashboard.png`: Analytics answered from natural language.

## Engineering Notes

The primary design and scale decisions are documented in [`ARCHITECTURE.md`](./ARCHITECTURE.md). The benchmark markdown in [`benchmarks/`](./benchmarks) and the generated figures in [`assets/`](./assets) should be read together: the markdown gives the measured numbers, and the screenshots make those results easier to inspect quickly during review.

## Files Included In This Repo

- [`README.md`](./README.md) for project overview, quickstart, and headline results
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) for system design and scaling choices
- [`benchmarks/`](./benchmarks) for measured results from the committed runs
- [`assets/`](./assets) for generated screenshots and dashboards
- [`tests/`](./tests) for the automated validation suite

## Delivery Summary

This project document was prepared for **Sai Veda** so the repository reads like a real project handoff: what the system is for, what problem it solves, what evidence supports it, and where the benchmark and test artifacts live inside the repo.
