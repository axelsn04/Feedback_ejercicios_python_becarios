# E01 — Formatos Bajo la Lupa

Build a benchmarking tool that compares storage formats for tabular data across multiple dataset scales. The exercise focuses on understanding the real cost of format decisions: not just disk size, but write time, read time, selective reads, and memory usage — and how those tradeoffs change as data grows.

---

## The Problem

Your team stores transaction logs as CSV because it's always worked. Your job is to challenge that decision with empirical evidence and propose an alternative backed by measurements, not opinion.

---

## Dataset

A synthetic financial transactions dataset that you generate as part of this exercise.

| Field | Type | Description |
|---|---|---|
| transaction_id | UUID4 | Unique per row |
| timestamp | datetime | Uniform distribution over 1 year |
| user_id | int | 1 – 50,000 |
| merchant_id | int | 1 – 10,000 |
| amount | float | 0.01 – 5,000.00 |
| category | string | 10 fixed values |
| country_code | string | 15 LATAM countries |
| status | string | completed (85%) / failed (10%) / pending (5%) |

The schema is fixed. Generate it at three scales: **100K, 500K, and 1M rows**.

---

## Requirements

- Python 3.11+
- `pandas`, `pyarrow`
- Environment managed with `uv` or `poetry`

---

## What to Build

**`generate_data.py`** — CLI script that accepts `--size` (100k, 500k, 1m) and saves the dataset as CSV to `data/`.

**`storage_benchmark/`** — Reusable module with separate functions for reading and writing each format.

**`benchmark_cli.py`** — CLI that accepts `--size` and `--formats` and runs the full benchmark, saving results to `results/`.

Formats to cover: `csv`, `jsonl`, `parquet`, `parquet_snappy`, `parquet_gzip`.

Metrics to measure per format and scale:
- Write time (average of 3 runs)
- Full read time
- Selective read time (columns `amount` and `category` only)
- File size on disk
- Peak memory during read (`tracemalloc`)

> Generation time does not count as write time. Generate the data first, then start measuring.

---

## Deliverables

```
ejercicio-01-formatos/
├── generate_data.py
├── benchmark_cli.py
├── storage_benchmark/
├── results/
│   ├── benchmark_100k.json
│   ├── benchmark_500k.json
│   └── benchmark_1m.json
└── report.md
```

**`report.md`** must include:
- Comparative tables per scale
- Bar charts for read times and disk size (matplotlib)
- A written analysis (400+ words) explaining *why* the differences occur, not just which format won
- A final recommendation with technical justification

---

## Evaluation

| Criterion | Weight |
|---|---|
| Functional CLI and organized module | 20% |
| Benchmark rigor | 25% |
| Analysis across scales | 25% |
| Report and conclusions | 30% |