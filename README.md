# csvscope

CSV schema profiling, type inference, statistics, and drift detection.

A Python library and CLI for inspecting CSVs *before* they touch your database. Built to catch silent schema changes, type drift, and data quality issues in data pipelines.

---

## Why csvscope

External CSV sources change without warning. A column gets renamed, a type silently shifts from integer to float, null encoding changes from `""` to `"N/A"` — and your downstream analytics quietly corrupt for weeks until someone notices weird dashboard numbers.

csvscope sits between the download and the ETL to catch problems *early*:

- **Profile** any CSV: schema, types, statistics, sample values
- **Compare** against a baseline profile to detect drift
- **Validate** structure before ingestion, fail loudly when something's off
- **Audit** historical data shape by archiving profiles alongside raw CSVs

It's small (a focused library + CLI), fast (pure Python, streaming-capable), and designed to drop into pipelines as a library import.

---

## Install

```bash
pip install csvscope
```

Requires Python 3.11+.

---

## Quick Start

### CLI

```bash
# Profile a CSV — human-readable output
csvscope data.csv

# JSON output for scripting
csvscope data.csv --json

# Save profile for later comparison
csvscope data.csv --output profiles/data-2026-04-15.json

# Compare against baseline, exit code 1 on drift
csvscope data.csv --baseline profiles/data-2026-04-08.json

# Specify encoding (auto-detected by default)
csvscope data.csv --encoding latin-1

# Strict mode: fail on any malformed row
csvscope data.csv --strict

# Sample mode: profile first N rows of a huge file
csvscope huge.csv --sample 10000
```

### Library

```python
from csvscope import profile_file, profile_bytes, Profile
from csvscope.diff import diff_profiles

# Profile a file
profile = profile_file("data.csv")
print(profile.row_count, profile.column_count)
for col in profile.columns:
    print(col.name, col.inferred_type, col.null_count)

# Profile from bytes (useful in pipelines)
profile = profile_bytes(csv_bytes, source="https://example.com/data.csv")

# Save and load profiles
profile.to_json("baseline.json")
baseline = Profile.from_json("baseline.json")

# Detect drift
report = diff_profiles(baseline, profile)
if report.has_drift:
    for change in report.changes:
        print(change)
```

---

## Features

- **Type inference**: integer, float, boolean, date, datetime, string
- **Statistical profiling**: null counts, distinct counts, min/max/mean/stddev, length stats
- **Encoding support**: UTF-8 (with BOM), UTF-16 LE/BE, Latin-1, auto-detection
- **Drift detection**: schema diffs, type changes, distribution shifts
- **Streaming**: process files larger than memory
- **Structured errors**: actionable exceptions with row numbers and byte positions
- **Two interfaces**: clean library API + opinionated CLI
- **Production-ready**: typed APIs, comprehensive tests, property-based testing

---

## Use Cases

### Pre-flight validation in ETL

```python
from csvscope import profile_bytes, ProfileError, MalformedCsvError

csv_bytes = download_csv(source_url)

try:
    profile = profile_bytes(csv_bytes, source=source_url)
except MalformedCsvError as e:
    alert(f"CSV from {source_url} is malformed at row {e.row_number}")
    raise
except ProfileError as e:
    alert(f"CSV from {source_url} failed profiling: {e}")
    raise

if profile.row_count == 0:
    raise EtlError(f"No data rows from {source_url}")
```

### Drift detection over time

```python
from csvscope.diff import diff_profiles

baseline = Profile.from_json(f"baselines/{source_id}.json")
current = profile_bytes(csv_bytes)

report = diff_profiles(baseline, current)
if report.has_breaking_changes:
    raise EtlError(f"Schema drift: {report.summary}")
if report.has_warnings:
    log_warning(report)

# Save current as new baseline
current.to_json(f"baselines/{source_id}.json")
```

### Schema discovery for new data sources

```bash
# Get a starting schema for a new CSV source
csvscope new_source.csv --json > inferred_schema.json
# Use the output to draft CREATE TABLE statements or Pydantic models
```

### Auditing for compliance/traceability

Profiles are small structured JSON — archive them alongside raw CSVs in blob storage:

```
archive/
├── source_42/
│   ├── 2026-04-15/
│   │   ├── data.csv
│   │   └── profile.json
│   ├── 2026-04-22/
│   │   ├── data.csv
│   │   └── profile.json
```

Now schema history is queryable without re-parsing raw files every time.

---

## What csvscope is NOT

To keep scope clear:

- **Not an ETL tool.** It inspects, it doesn't transform or load.
- **Not a data quality framework** like Great Expectations or Soda. It measures, doesn't enforce.
- **Not a schema registry.** It profiles ad-hoc CSVs; it doesn't store schemas as a service.
- **Not SQL-aware.** Pure CSV today. Parquet support may come later.
- **Not real-time.** Batch-oriented: profile a file, get a result.

If you need heavyweight data quality validation, use Great Expectations. If you need exploratory analysis with HTML reports, use ydata-profiling. csvscope is for the middle: lightweight, pipeline-integrated, programmatic.

---

## Exit Codes (CLI)

| Code | Meaning |
|---|---|
| 0 | Success, no drift |
| 1 | Drift detected (when `--baseline` used) |
| 2 | Malformed CSV |
| 3 | File access error |
| 4 | Encoding error |
| 64 | Usage error (invalid args) |

---

## Public API

### Functions

```python
csvscope.profile_file(path, encoding="utf-8") -> Profile
csvscope.profile_bytes(data, encoding="utf-8", source=None) -> Profile
csvscope.profile_stream(stream, encoding="utf-8") -> Profile
csvscope.diff.diff_profiles(baseline, current) -> DriftReport
```

### Models

```python
csvscope.Profile          # full profile of a CSV
csvscope.ColumnStats      # per-column stats within a profile
csvscope.diff.DriftReport # output of diff_profiles
```

### Exceptions

```python
csvscope.ProfileError              # base for all errors
├── FileAccessError                # missing, permissions, directory
├── FileEmptyError                 # zero bytes
├── NoDataError                    # header only, no rows
├── EncodingError                  # decode failure
├── MalformedCsvError              # structural CSV issues
│   ├── InconsistentColumnsError   # row column count mismatch
│   └── UnclosedQuoteError         # quoted field never closed
├── SchemaInferenceError           # could not infer coherent schema
└── ProfileLoadError               # baseline JSON load failure
```

All errors inherit from `ProfileError` for blanket handling. All errors preserve their cause via `raise X from e` for actionable tracebacks.

---

## Development

```bash
git clone https://github.com/<you>/csvscope
cd csvscope
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

### Common commands

```bash
pytest                          # all tests
pytest -m unit                  # fast tests only
pytest -m "not slow"            # exclude slow tests
pytest --cov=csvscope            # with coverage

ruff check .                    # lint
ruff format .                   # format
mypy csvscope/                  # type check

python -m build                 # build sdist + wheel
```

### Project structure

```
csvscope/
├── csvscope/
│   ├── __init__.py             # public API exports
│   ├── __main__.py             # python -m csvscope
│   ├── exceptions.py           # custom exception hierarchy
│   ├── types.py                # Pydantic models
│   ├── profile.py              # core profiling logic
│   ├── inference.py            # type inference
│   ├── stats.py                # statistical computations
│   ├── diff.py                 # drift detection
│   ├── encoding.py             # encoding detection
│   ├── reader.py               # CSV reading abstractions
│   └── cli.py                  # CLI entry point
├── tests/
│   ├── conftest.py
│   ├── test_*.py
│   ├── test_property_based.py
│   └── data/                   # CSV fixtures
└── .github/
    ├── workflows/              # CI + release pipelines
    └── release-notes/          # per-version release notes
```

See `AGENT.md` for development conventions and rules.

---

## Versioning

[Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

While in `0.x`, the API may change between minor versions. v1.0 will lock the public API.

See `CHANGELOG.md` for release history and `.github/release-notes/` for per-release notes.

---

## License

MIT. See `LICENSE`.
