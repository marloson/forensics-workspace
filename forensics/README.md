# forensics/

This directory contains sample data for testing the forensics tools in this workspace.

## Contents

- **logarchive/** — Sample `.logarchive` bundles for testing `macos-UnifiedLogs`
- **output/** — Sample parsed output data (JSON, TSV, etc.) for reference and validation

## Test Data Setup

### macos-UnifiedLogs (Rust)

The Rust test suite requires test data to be downloaded separately. Run the following from the workspace root:

```bash
cd macos-UnifiedLogs/tests && \
  wget -O ./test_data.zip https://github.com/mandiant/macos-UnifiedLogs/releases/download/v1.0.0/test_data.zip && \
  unzip test_data.zip
```

Then run the tests:

```bash
cargo test --release
```

### iLEAPP (Python)

Place iOS backup archives (`.zip`, `.tar`, `.gz`) or extracted filesystem images in this directory to use as input for iLEAPP:

```bash
python ileapp.py -t zip -i forensics/<your_input>.zip -o forensics/output/
```

## Notes

- This directory is **gitignored** for large binary files (`.zip`, `.logarchive`). See `.gitignore` at the repo root.
- Do not commit large test archives — store them locally or reference the download URLs above.
