# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This workspace contains four digital forensics projects focused on Apple log parsing and iOS artifact extraction:

| Project | Language | Status | Description |
|---------|----------|--------|-------------|
| `macos-UnifiedLogs/` | Rust | Active | High-performance macOS Unified Log parser (Mandiant) |
| `iLEAPP/` | Python 3.10-3.12 | Active | iOS Logs, Events, And Plists Parser |
| `UnifiedLogReader/` | Python 3.6+ | **Archived** | Legacy Python tracev3 parser (superseded by macos-UnifiedLogs) |
| `forensics/` | N/A | Data | Sample logarchive and parser output (CSV, JSONL, SQLite) for validation |

These are **independent tools** with no cross-dependencies. macos-UnifiedLogs parses macOS `.logarchive` files; iLEAPP parses iOS backups/extractions. They share a workspace but not code.

## macos-UnifiedLogs (Rust)

### Build & Test Commands

```bash
cargo build --release                    # Release build
cargo test --release                     # Run tests (requires test data, see below)
cargo test --release <test_name>         # Run a single test
cargo fmt -- --check                     # Format check (CI enforced)
cargo clippy                             # Lint check (CI enforced)
cargo bench                              # Run benchmarks (criterion)
cargo deny check                         # License/advisory/ban checks
```

**Test data setup:** Tests require downloading test data first:
```bash
cd macos-UnifiedLogs/tests && wget -O ./test_data.zip https://github.com/mandiant/macos-UnifiedLogs/releases/download/v1.0.0/test_data.zip && unzip test_data.zip
```

**Example binary** (`examples/` is a separate Cargo workspace):
```bash
cd macos-UnifiedLogs/examples && cargo build --release
# Parse logarchive:  ./target/release/unifiedlog_iterator -m log-archive -i <path.logarchive>
# Parse live system: ./target/release/unifiedlog_iterator -m live
# Parse single file: ./target/release/unifiedlog_iterator -m single-file -i <path.tracev3>
# Output format:     --format csv  or  --format jsonl (default)
```

### Architecture

The library parses Apple's Unified Log binary format (tracev3 files) using `nom` parser combinators.

- **`src/lib.rs`** — Library root. `#![forbid(unsafe_code)]` with strict clippy lints (cast_lossless, cast_possible_wrap, checked_conversions, etc.)
- **`src/parser.rs`** — Main parsing entry points: `collect_timesync()`, `build_log()`
- **`src/unified_log.rs`** — `UnifiedLogData` struct (the primary output type); each log entry includes an `evidence` field with the source tracev3 file path (added v0.5.1)
- **`src/iterator.rs`** — `UnifiedLogIterator` for streaming log entries
- **`src/traits.rs`** / **`src/filesystem.rs`** — `FileProvider` trait with `LiveSystemProvider` implementation
- **`src/chunks/`** — Binary chunk parsers (firehose logs, signposts, activities, state/simple dumps, oversize entries)
- **`src/decoders/`** — Type-specific decoders (UUID, DNS, network, location, darwin, time, OpenDirectory)
- **`src/catalog.rs`**, **`src/dsc.rs`**, **`src/uuidtext.rs`**, **`src/timesync.rs`** — Format-specific file parsers

### Key Conventions

- Rust edition 2024, unsafe code is forbidden (`#![forbid(unsafe_code)]`)
- Uses `nom` for all binary parsing
- rustfmt: Unix LF, 4-space tabs, max_width=100, chain_width=60
- Allowed licenses (cargo-deny): MIT, Apache-2.0, Unlicense, Unicode-3.0

### CI (6 workflows)

- **pullrequest.yml** — fmt check + clippy + test on macOS x86_64 and aarch64
- **deny.yml** — cargo-deny checks on PRs (licenses, advisories, bans)
- **audit.yml** — cargo-audit on PRs touching Cargo.toml/lock + daily cron
- **publish.yml** — publishes to crates.io on version tags (`v[0-9]+.*`)
- **release.yml / cross-release.yml** — builds binaries for macOS/Windows/Linux on version tags

## iLEAPP (Python)

### Build & Run Commands

```bash
pip install -r requirements.txt                                          # Install dependencies
pip install -r requirements-dev.txt                                      # Install test dependencies
python ileapp.py -t <fs|tar|zip|gz|itunes|file> -i <input> -o <output>  # CLI
python ileappGUI.py                                                      # GUI (Linux: sudo apt-get install python3-tk)
PYTHONPATH=. pylint <file.py> --disable=C,R                              # Lint (CI enforced, W+E only)
PYTHONPATH=. pytest tests/unit/ -v -m unit                               # Fast unit tests (no external data)
PYTHONPATH=. pytest tests/artifacts/ -v -m artifact                      # Integration tests (needs iOS extraction data)
```

Python 3.10–3.12 required. Two pytest markers: `unit` (fast, no I/O) and `artifact` (needs real iOS data).

### Architecture

Plugin-based architecture where artifact parsers are dynamically loaded at runtime.

- **`ileapp.py`** / **`ileappGUI.py`** — CLI and GUI entry points
- **`scripts/plugin_loader.py`** — Scans `scripts/artifacts/` for plugins, supports v1 (`__artifacts__`) and v2 (`__artifacts_v2__`) metadata formats
- **`scripts/ilapfuncs.py`** — Core utilities: `@artifact_processor` decorator, output generation (HTML, TSV, KML, LAVA), database helpers
- **`scripts/search_files.py`** — File discovery across input sources (zip, tar, filesystem, gz)
- **`scripts/artifact_report.py`** / **`scripts/report.py`** — HTML report generation with feathericons
- **`scripts/lavafuncs.py`** — LAVA output format processing
- **`scripts/artifacts/`** — ~318 artifact plugin modules (dynamically loaded)
- **`admin/scripts/`** — Auto-run on main branch: generates module catalog, device info docs, filepath summaries (via CI workflow `update_module_info.yml`)

### CI (3 workflows)

- **python_lint.yml** — pylint on changed .py files only (`--disable=C,R`)
- **update_module_info.yml** — auto-generates admin/ docs when plugins change (commits directly to main)
- **filepath_search_summary.yml** — manual dispatch for filepath search docs

### Writing Artifact Plugins

Each plugin in `scripts/artifacts/` must define `__artifacts_v2__` at module top:

```python
__artifacts_v2__ = {
    "function_name": {
        "name": "Display Name",
        "description": "...",
        "author": "@username",
        "version": "0.1",
        "date": "YYYY-MM-DD",
        "requirements": "none",
        "category": "Category Name",
        "notes": "",
        "paths": ('*/path/to/files*',),
        "output_types": "all",
        "artifact_icon": "feather-icon-name"
    }
}
```

The function key must match the decorated function name exactly. Two signatures are supported by the decorator (auto-detected via `inspect.signature`):

```python
# Legacy 5-arg signature
@artifact_processor
def function_name(files_found, report_folder, seeker, wrap_text, timezone_offset):
    # Return (data_headers, data_list, source_path)

# Newer context-based signature (preferred for new plugins)
@artifact_processor
def function_name(context):
    files_found = context.get_files_found()
    # Return (data_headers, data_list, source_path)
```

**Header tuple conventions:**
- Plain string `'Column Name'` — regular column
- `('Column Name', 'datetime')` — marks column for timeline output
- `('Column Name', 'media')` — marks column for media handling

Each sub-project also has its own detailed `CLAUDE.md` with deeper coverage of architecture, patterns, and gotchas.

## UnifiedLogReader (Archived)

Legacy Python parser. Do not invest significant effort here — use `macos-UnifiedLogs` instead.

```bash
pip install -r requirements.txt    # Install deps
python run_tests.py                # Run tests
```

## Session Continuity

- On vague prompts ("continue", "next step", "keep going"): **read `~/.claude/projects/-home/memory/MEMORY.md` first**, then either ask for clarification or use `/phase-next`
- Max **2 tool calls** exploring before starting actual work on focused tasks
- Read the **target project's CLAUDE.md**, not the whole codebase — the per-repo files have everything you need

## Git Workflow

- Run format/lint/test **before every commit** — use `/qa` or the per-repo checklist
- Never commit directly to `main` — use feature branches (`feat/`, `fix/`, `refactor/`, `test/`)
- Conventional commit messages: `fix:`, `feat:`, `refactor:`, `test:`, `docs:`, `chore:`
- One logical change per commit — don't bundle unrelated fixes
- NEVER add Co-authored-by trailers to git commits

## Exploration Budget

- Max **3 Read/Grep calls** before making your first edit on focused tasks
- If you've read **>10 files** without editing, stop and ask the user whether you're on the right track
- Prefer targeted searches (Grep for a specific symbol) over broad exploration (reading whole directories)
