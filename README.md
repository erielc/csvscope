# csvscope

A Python library and CLI for CSV schema profiling, type inference, statistics, and drift detection. Designed to be imported by other projects (like beacon) AND used standalone as a CLI.

---

## Goals

- **Library-first**: clean Python API for programmatic use (`from csvscope import profile_file`)
- **CLI-second**: human-friendly CLI and machine-readable JSON output for scripting
- **Edge-case rigorous**: handle malformed CSVs, encoding issues, type ambiguity, scale
- **Well-tested**: comprehensive unit tests, edge cases, property-based testing
- **Production patterns**: custom exceptions, structured errors, typed APIs, dependency injection

---

## Project Structure

```
csvscope/
├── src/
│   └── csvscope/
│       ├── __init__.py         # public API exports
│       ├── __main__.py         # python -m csvscope
│       ├── exceptions.py       # custom exception hierarchy
│       ├── types.py            # Pydantic models for Profile, ColumnStats
│       ├── profile.py          # core profiling logic
│       ├── inference.py        # type inference logic
│       ├── stats.py            # statistical computations
│       ├── diff.py             # baseline comparison / drift detection
│       ├── encoding.py         # encoding detection and decoding
│       ├── reader.py           # CSV reading abstractions
│       └── cli.py              # CLI entry point
├── tests/
│   ├── conftest.py             # shared fixtures
│   ├── test_profile.py
│   ├── test_inference.py
│   ├── test_stats.py
│   ├── test_diff.py
│   ├── test_encoding.py
│   ├── test_cli.py
│   ├── test_property_based.py  # hypothesis tests
│   └── data/                   # sample CSV files
│       ├── clean.csv
│       ├── malformed_unclosed_quote.csv
│       ├── malformed_inconsistent_cols.csv
│       ├── utf8_bom.csv
│       ├── utf16.csv
│       ├── latin1.csv
│       ├── mixed_types.csv
│       ├── all_nulls.csv
│       ├── single_row.csv
│       ├── empty.csv
│       └── large.csv
├── .github/
│   └── workflows/
│       ├── ci.yml              # tests, lint, type-check
│       ├── release.yml         # PyPI publish + GitHub release
│       └── codeql.yml          # security scanning (optional)
├── AGENT.md                    # guide for coding agents
├── README.md
├── LICENSE                     # MIT
├── pyproject.toml
├── .gitignore
└── .python-version
```

---

## Public API Design

### Library usage

```python
from csvscope import profile_file, profile_bytes, profile_stream
from csvscope import Profile, ColumnStats
from csvscope import (
    ProfileError,
    FileAccessError,
    EncodingError,
    MalformedCsvError,
)
from csvscope.diff import diff_profiles, DriftReport

# Profile a file
profile = profile_file("data.csv", encoding="utf-8")
print(profile.row_count, profile.columns)

# Profile bytes (useful in pipelines)
profile = profile_bytes(csv_bytes, encoding="utf-8")

# Profile a file-like object (streaming, doesn't load all into memory)
with open("huge.csv", "rb") as f:
    profile = profile_stream(f)

# Drift detection
baseline = Profile.from_json("baseline.json")
report = diff_profiles(baseline, current_profile)
if report.has_drift:
    for change in report.changes:
        print(change)
```

### CLI usage

```bash
# Basic
csvscope data.csv

# JSON output
csvscope data.csv --json

# Save profile for later comparison
csvscope data.csv --output profiles/data-2026-04-15.json

# Compare against baseline
csvscope data.csv --baseline profiles/data-2026-04-08.json

# Strict mode: fail on any malformed row
csvscope data.csv --strict

# Specify encoding
csvscope data.csv --encoding latin-1

# Sample mode: profile first N rows (for huge files)
csvscope huge.csv --sample 10000

# Quiet (only errors)
csvscope data.csv --quiet

# Verbose (show progress)
csvscope data.csv --verbose
```

Exit codes:
- `0`: success, no drift
- `1`: drift detected (when `--baseline` used)
- `2`: malformed CSV
- `3`: file access error
- `4`: encoding error
- `64`: usage error (invalid args)

---

## Exception Hierarchy

```python
# src/csvscope/exceptions.py

class ProfileError(Exception):
    """Base for all csvscope errors."""


class FileAccessError(ProfileError):
    """Cannot access the file (missing, permissions, directory, etc)."""
    def __init__(self, path: str, message: str = ""):
        self.path = path
        super().__init__(message or f"cannot access {path}")


class FileEmptyError(ProfileError):
    """File has zero bytes."""


class NoDataError(ProfileError):
    """File has a header but no data rows."""


class EncodingError(ProfileError):
    """File cannot be decoded with the chosen/detected encoding."""
    def __init__(
        self,
        message: str,
        encoding: str,
        byte_position: int | None = None,
    ):
        super().__init__(message)
        self.encoding = encoding
        self.byte_position = byte_position


class MalformedCsvError(ProfileError):
    """CSV structure is invalid."""
    def __init__(
        self,
        message: str,
        row_number: int | None = None,
        column: str | None = None,
    ):
        super().__init__(message)
        self.row_number = row_number
        self.column = column


class InconsistentColumnsError(MalformedCsvError):
    """A row has a different column count than the header."""
    def __init__(self, row_number: int, expected: int, got: int):
        super().__init__(
            f"row {row_number} has {got} columns, expected {expected}",
            row_number=row_number,
        )
        self.expected_columns = expected
        self.got_columns = got


class UnclosedQuoteError(MalformedCsvError):
    """A quoted field was never closed."""


class SchemaInferenceError(ProfileError):
    """Could not infer a coherent schema."""


class ProfileLoadError(ProfileError):
    """Could not load a baseline profile for comparison."""
    def __init__(self, path: str, cause: Exception):
        self.path = path
        self.cause = cause
        super().__init__(f"cannot load profile from {path}: {cause}")
```

**Rationale:**
- `ProfileError` base class allows blanket catching (`except ProfileError`)
- Specific subclasses allow precise handling (`except MalformedCsvError`)
- Errors carry structured fields (row_number, column, byte_position) for actionable messages
- Inheritance reflects real relationships (`InconsistentColumnsError` IS a `MalformedCsvError`)
- Use `raise X from e` everywhere to preserve cause chain

---

## Type Models

```python
# src/csvscope/types.py
from typing import Literal
from pydantic import BaseModel, Field
from datetime import datetime

InferredType = Literal["integer", "float", "boolean", "date", "datetime", "string", "empty"]


class ColumnStats(BaseModel):
    name: str
    inferred_type: InferredType
    null_count: int = 0
    null_percentage: float = 0.0
    distinct_count: int | None = None  # None if not computed
    
    # Numeric stats (None for non-numeric columns)
    min_value: float | None = None
    max_value: float | None = None
    mean: float | None = None
    median: float | None = None
    stddev: float | None = None
    
    # String stats
    min_length: int | None = None
    max_length: int | None = None
    avg_length: float | None = None
    
    # Sample values for human inspection
    sample_values: list[str] = Field(default_factory=list, max_length=5)
    
    # Detected issues
    warnings: list[str] = Field(default_factory=list)


class Profile(BaseModel):
    source: str | None = None  # file path or identifier
    profiled_at: datetime
    file_size_bytes: int | None = None
    encoding: str
    delimiter: str = ","
    has_header: bool = True
    
    row_count: int
    column_count: int
    columns: list[ColumnStats]
    
    # Detected issues at file level
    warnings: list[str] = Field(default_factory=list)
    
    # Hash for idempotency tracking
    content_hash: str | None = None  # SHA-256 of file content
    
    def to_json(self, path: str) -> None:
        with open(path, "w") as f:
            f.write(self.model_dump_json(indent=2))
    
    @classmethod
    def from_json(cls, path: str) -> "Profile":
        try:
            with open(path) as f:
                return cls.model_validate_json(f.read())
        except (OSError, ValueError) as e:
            from .exceptions import ProfileLoadError
            raise ProfileLoadError(path, e) from e
```

---

## Edge Cases to Handle

### Encoding edge cases
- UTF-8 BOM (strip it)
- UTF-16 LE/BE (detect via BOM)
- Latin-1 / Windows-1252 (fall back if UTF-8 fails)
- Invalid byte sequences in declared encoding
- Mixed encodings within one file (rare but happens)

### Structural edge cases
- Empty file (0 bytes) → `FileEmptyError`
- Header-only file (no data rows) → `NoDataError`
- Single column, no delimiter
- Trailing newline vs no trailing newline
- Quoted fields with embedded newlines
- Quoted fields with embedded delimiters
- Quoted fields with embedded quotes (escaped as `""`)
- Unclosed quotes → `UnclosedQuoteError`
- Inconsistent column counts per row → `InconsistentColumnsError`
- Different delimiters: `,` `\t` `|` `;` (auto-detect or specify)

### Type inference edge cases
- Column with all integers but one is "1.0" → float
- Column with mostly numbers but one "N/A" → string OR float-with-nulls (configurable)
- Column with dates in multiple formats: `2026-01-15`, `01/15/2026`, `15-Jan-2026`
- Column with leading zeros: `007`, `042` → string (not integer, preserve format)
- Column with scientific notation: `1e10`, `1.5E-3`
- Column with currency: `$1,234.56`, `€100`, `£50`
- Column with thousands separators: `1,234,567`
- Column with percentages: `75%`, `99.9%`
- Booleans: `true/false`, `yes/no`, `Y/N`, `1/0`, `True/False`
- Nulls represented as: empty, `NULL`, `null`, `N/A`, `n/a`, `-`, `?`, `\N`

### Numeric edge cases
- Infinity, NaN
- Very large numbers (overflow int64)
- Very small numbers (underflow)
- Negative numbers with parentheses: `(123)` = -123 (accounting notation)
- Numbers with units: `100kg`, `5ms`

### Scale edge cases
- 50GB file (cannot load into memory) → streaming mode required
- 100,000 columns → memory considerations
- Very wide rows (single row, millions of characters)
- File with 1 row and 0 rows

### File system edge cases
- File doesn't exist → `FileAccessError`
- File is a directory → `FileAccessError`
- No read permission → `FileAccessError`
- Symlink (follow vs not)
- File deleted while reading → handle gracefully
- File is a FIFO / device file → handle or reject

---

## Test Plan

### Test categories (use pytest markers)

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: fast unit tests with no I/O",
    "integration: tests with file I/O or external dependencies",
    "slow: tests over 1 second",
    "property: hypothesis property-based tests",
]
```

### Test file structure

```python
# tests/test_profile.py

class TestProfileFile:
    """Tests for profile_file() entry point."""
    
    def test_profiles_clean_csv(self, sample_clean_csv):
        ...
    
    def test_raises_file_access_error_on_missing(self, tmp_path):
        ...
    
    def test_raises_file_access_error_on_directory(self, tmp_path):
        ...
    
    @pytest.mark.skipif(os.name == "nt", reason="POSIX permissions only")
    def test_raises_on_no_permission(self, tmp_path):
        ...

class TestEncoding:
    @pytest.mark.parametrize("encoding,content", [
        ("utf-8", "id,name\n1,café".encode("utf-8")),
        ("latin-1", "id,name\n1,café".encode("latin-1")),
        ("utf-16", "id,name\n1,café".encode("utf-16")),
    ])
    def test_handles_various_encodings(self, tmp_path, encoding, content):
        ...
    
    def test_strips_utf8_bom(self):
        ...
    
    def test_raises_encoding_error_on_invalid_bytes(self):
        ...

class TestMalformedInput:
    def test_raises_on_unclosed_quote(self):
        ...
    
    def test_raises_on_inconsistent_columns(self):
        ...
    
    def test_handles_quoted_newlines(self):
        ...
    
    def test_handles_escaped_quotes(self):
        ...

class TestTypeInference:
    @pytest.mark.parametrize("values,expected_type", [
        (["1", "2", "3"], "integer"),
        (["1.0", "2.5"], "float"),
        (["true", "false"], "boolean"),
        (["2026-01-15", "2026-02-20"], "date"),
        (["", "1", "2"], "integer"),  # nulls don't break inference
        (["007", "042"], "string"),    # preserve leading zeros
    ])
    def test_infers_type(self, values, expected_type):
        ...

class TestEdgeCases:
    def test_empty_file_raises(self, tmp_path):
        ...
    
    def test_header_only_raises_no_data(self, tmp_path):
        ...
    
    def test_handles_single_column(self):
        ...
    
    def test_handles_single_row(self):
        ...
    
    def test_handles_trailing_newline(self):
        ...
    
    def test_handles_no_trailing_newline(self):
        ...

# tests/test_property_based.py
from hypothesis import given, strategies as st

@pytest.mark.property
@given(st.lists(st.integers(), min_size=1, max_size=1000))
def test_integer_column_min_max_correct(numbers):
    csv_content = "value\n" + "\n".join(str(n) for n in numbers)
    profile = profile_bytes(csv_content.encode())
    col = profile.columns[0]
    assert col.min_value == min(numbers)
    assert col.max_value == max(numbers)

@pytest.mark.property
@given(st.text(min_size=0, max_size=1000))
def test_parser_never_crashes_unexpectedly(text):
    """Property: parser should either succeed or raise ProfileError, never something else."""
    try:
        profile_bytes(text.encode("utf-8", errors="replace"))
    except ProfileError:
        pass  # expected
    # Any other exception is a bug
```

### Coverage targets

- Aim for 85%+ on `src/csvscope/`
- Exclude `cli.py` from strict coverage (mostly glue)
- Use `pytest-cov` with `--cov-fail-under=85` in CI

---

## CI/CD Setup

### `pyproject.toml` essentials

```toml
[project]
name = "csvscope"
version = "0.1.0"
description = "CSV schema profiling, type inference, and drift detection"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [{name = "Eriel"}]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Topic :: Software Development :: Libraries",
]
dependencies = [
    "pydantic>=2.0",
    "click>=8.0",      # or typer
    "chardet>=5.0",    # encoding detection
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "hypothesis>=6.0",
    "ruff>=0.5",
    "mypy>=1.0",
]

[project.scripts]
csvscope = "csvscope.cli:main"

[project.urls]
Homepage = "https://github.com/<you>/csvscope"
Repository = "https://github.com/<you>/csvscope"
Issues = "https://github.com/<you>/csvscope/issues"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM"]

[tool.mypy]
strict = true
python_version = "3.11"

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "unit: fast unit tests with no I/O",
    "integration: tests with file I/O or external dependencies",
    "slow: tests over 1 second",
    "property: hypothesis property-based tests",
]
```

---

## README Outline

```markdown
# csvscope

CSV schema profiling, type inference, and drift detection.

## Install
pip install csvscope

## Quick Start
[CLI examples]
[Library examples]

## Why
[explain the niche]

## Features
- Type inference
- Statistical profiling
- Drift detection
- Multiple encoding support
- Streaming for large files

## Examples
[3-5 concrete examples]

## API Reference
[link to docs or inline]

## Development
[how to contribute]

## License
MIT
```

---

## Versioning + Release Strategy

- Follow [Semantic Versioning](https://semver.org/): MAJOR.MINOR.PATCH
- v0.x while API is evolving (breaking changes allowed)
- v1.0 when API is stable and beacon depends on it
- Tag releases: `git tag v0.1.0`
- GitHub release on tag push triggers PyPI publish
- Use `hatch version` or manual bump in `pyproject.toml`

### Release flow

1. Bump version in `pyproject.toml`
2. Update `CHANGELOG.md`
3. Commit: `git commit -m "release: v0.1.0"`
4. Tag: `git tag v0.1.0`
5. Push: `git push && git push --tags`
6. GitHub Actions builds and publishes to PyPI
7. GitHub release created automatically with source tarball

---

## Development Workflow

```bash
# Setup
git clone https://github.com/<you>/csvscope
cd csvscope
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# Run tests
pytest                          # all
pytest -m unit                  # fast only
pytest -m "not slow"            # exclude slow
pytest --cov=csvscope            # with coverage

# Lint + format
ruff check .
ruff format .

# Type check
mypy src/

# Build for testing
pip install build
python -m build
```

---

## What to build first (suggested order)

1. **Day 1**: Project skeleton, `pyproject.toml`, exception hierarchy, type models
2. **Day 2**: Core profile_bytes with happy-path tests
3. **Day 3**: Type inference engine + tests (integer, float, string, bool, null)
4. **Day 4**: Encoding detection + handling
5. **Day 5**: Malformed input handling (unclosed quotes, inconsistent cols)
6. **Day 6**: Statistics computation (min/max/mean/stddev/distinct)
7. **Day 7**: CLI with click/typer
8. **Day 8**: Drift detection (diff between profiles)
9. **Day 9**: Property-based tests with hypothesis
10. **Day 10**: README, CI workflows, polish
11. **Day 11**: First release to PyPI
12. **Day 12**: Use it in beacon, iterate

---

## Future enhancements (post-v1.0)

- Polars/pyarrow integration for huge files
- Parquet output (profile + samples)
- Multi-file profiling (directory mode)
- Schema YAML export (for downstream type-strict ETL)
- Streaming profile from stdin
- Auto-fix mode: emit a cleaned CSV based on detected issues
- Web UI for visual exploration
