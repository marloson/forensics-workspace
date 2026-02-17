# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This workspace contains four digital forensics projects focused on Apple log parsing and iOS artifact extraction:

| Project | Language | Status | Description |
|---------|----------|--------|-------------|
| `macos-UnifiedLogs/` | Rust | Active | High-performance macOS Unified Log parser (Mandiant) |
| `iLEAPP/` | Python 3.10-3.12 | Active | iOS Logs, Events, And Plists Parser |
| `UnifiedLogReader/` | Python 3.6+ | **Archived** | Legacy Python tracev3 parser (superseded by macos-UnifiedLogs) |
| `forensics/` | N/A | Data | Sample logarchive and output data for testing |

## macos-UnifiedLogs (Rust)

### Build & Test Commands

```bash
cargo build --release                    # Release build
cargo test --release                     # Run tests (requires test data, see below)
cargo fmt -- --check                     # Format check (CI enforced)
cargo clippy                             # Lint check (CI enforced)
cargo bench                              # Run benchmarks (criterion)
```

**Test data setup:** Tests require downloading test data first:
```bash
cd macos-UnifiedLogs/tests && wget -O ./test_data.zip https://github.com/mandiant/macos-UnifiedLogs/releases/download/v1.0.0/test_data.zip && unzip test_data.zip
```

**Example binary:**
```bash
cd macos-UnifiedLogs/examples/unifiedlog_iterator && cargo build --release
# Parse logarchive: ./target/release/unifiedlog_iterator -i <path/to/logarchive>
# Parse live system: ./target/release/unifiedlog_iterator -l true
```

### Architecture

The library parses Apple's Unified Log binary format (tracev3 files) using `nom` parser combinators.

- **`src/lib.rs`** — Library root. `#![forbid(unsafe_code)]` with strict clippy lints (cast_lossless, cast_possible_wrap, checked_conversions, etc.)
- **`src/parser.rs`** — Main parsing entry points: `collect_timesync()`, `build_log()`
- **`src/unified_log.rs`** — `UnifiedLogData` struct (the primary output type)
- **`src/iterator.rs`** — `UnifiedLogIterator` for streaming log entries
- **`src/traits.rs`** / **`src/filesystem.rs`** — `FileProvider` trait with `LiveSystemProvider` implementation
- **`src/chunks/`** — Binary chunk parsers (firehose logs, signposts, activities, state/simple dumps, oversize entries)
- **`src/decoders/`** — Type-specific decoders (UUID, DNS, network, location, darwin, time, OpenDirectory)
- **`src/catalog.rs`**, **`src/dsc.rs`**, **`src/uuidtext.rs`**, **`src/timesync.rs`** — Format-specific file parsers

### Key Conventions

- Rust edition 2024, unsafe code is forbidden
- Uses `nom` for all binary parsing
- CI runs `cargo fmt`, `cargo clippy`, and `cargo test --release` on macOS (x86_64 + aarch64)

## iLEAPP (Python)

### Build & Run Commands

```bash
pip install -r requirements.txt                                    # Install dependencies
python ileapp.py -t <zip|tar|fs|gz> -i <input> -o <output>       # CLI
python ileappGUI.py                                                # GUI
PYTHONPATH=. pylint <file.py> --disable=C,R                       # Lint (CI enforced)
```

### Architecture

Plugin-based architecture where artifact parsers are dynamically loaded at runtime.

- **`ileapp.py`** / **`ileappGUI.py`** — CLI and GUI entry points
- **`scripts/plugin_loader.py`** — Scans `scripts/artifacts/` for plugins, supports v1 (`__artifacts__`) and v2 (`__artifacts_v2__`) metadata formats
- **`scripts/ilapfuncs.py`** — Core utilities: `@artifact_processor` decorator, output generation (HTML, TSV, KML, LAVA), database helpers
- **`scripts/search_files.py`** — File discovery across input sources (zip, tar, filesystem, gz)
- **`scripts/artifact_report.py`** / **`scripts/report.py`** — HTML report generation with feathericons
- **`scripts/lavafuncs.py`** — LAVA output format processing
- **`scripts/artifacts/`** — ~318 artifact plugin modules (dynamically loaded)

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

The function key must match the decorated function name exactly:

```python
@artifact_processor
def function_name(files_found, report_folder, seeker, wrap_text, timezone_offset):
    # Return (data_headers, data_list, source_path)
```

## UnifiedLogReader (Archived)

Legacy Python parser. Do not invest significant effort here — use `macos-UnifiedLogs` instead.

```bash
pip install -r requirements.txt    # Install deps
python run_tests.py                # Run tests
```
