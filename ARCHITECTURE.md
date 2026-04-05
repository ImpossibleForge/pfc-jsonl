# PFC-JSONL — Architecture Overview

<!-- updated: 2026-04-05 -->

This document explains how PFC-JSONL works at a high level — the file format, index structure, and how integrations (DuckDB, Fluent Bit, Python) connect to the core binary.

---

## Compression Pipeline

PFC-JSONL uses a purpose-built multi-stage pipeline optimized for structured log data (JSONL):

```
Input JSONL
    │
    ▼
[Transform Stage]
    │  Block-based processing (8 / 32 / 128 MiB blocks)
    │  Each block is compressed independently → enables parallelism
    │
    ▼
[Entropy Coding]
    │  High-efficiency entropy coder
    │  Achieves ~9% ratio on typical JSONL log data
    │
    ▼
Compressed .pfc file  +  .pfc.bidx block index
```

The algorithm is specifically tuned for structured log patterns: repeated JSON keys, ISO timestamps, log levels, and service names. Generic compressors (gzip, zstd) are not optimized for this structure.

---

## File Format

A `.pfc` file is a sequence of independently compressed blocks:

```
[File Header]
[Block 0: compressed data]
[Block 1: compressed data]
...
[Block N-1: compressed data]
```

**Block independence** is the key design choice: any block can be decompressed without reading others. This enables:
- Parallel compression (workers process blocks simultaneously)
- Random access (decompress only the blocks you need)

---

## Block Index (.pfc.bidx)

Every `.pfc` file is accompanied by a `.pfc.bidx` index file, created automatically during compression:

```
[Index Header]
[Block 0: byte_offset, block_size, ts_start, ts_end]
[Block 1: byte_offset, block_size, ts_start, ts_end]
...
```

Each entry stores:
- **byte_offset** — where the block starts in the `.pfc` file
- **ts_start / ts_end** — the min/max Unix timestamps of all records in the block

The index is binary, fixed-size per entry, and directly readable from C++ — no binary invocation required for index lookups.

**Supported timestamp fields:** `timestamp`, `ts`, `time`, `@timestamp`
**Supported formats:** ISO 8601 (requires timezone suffix, e.g. `Z` or `+00:00`) and Unix epoch

---

## Time-Range Queries

```
Query: --from 2026-01-15T08:00 --to 2026-01-15T09:00

1. Read .pfc.bidx (fast, C-level)
2. Find all blocks where [ts_start, ts_end] overlaps [from, to]
3. Decompress only those blocks
4. Output matching JSON lines
```

For a 30-day archive with hourly blocks (720 blocks total):
a 1-hour query reads **1 block** — a ~720× speedup over full decompression.

---

## Parallelization

Workers are auto-detected at runtime:

```
workers = min(cpu_cores, num_blocks, available_ram ÷ (block_size × overhead_factor))
```

Workers are capped to prevent OOM. The formula is RAM-aware and safe on shared infrastructure.

---

## Community Mode

The binary tracks daily usage locally in `~/.pfc/usage.json`:
- **5 GB/day free** — no account, no network calls
- `compress` counts input bytes; `decompress`, `query`, `seek-blocks` count decompressed output bytes
- Resets at midnight UTC

---

## DuckDB Extension Architecture

```
DuckDB SQL
    │
    ▼
pfc extension (C++, open source, MIT)
    │  Reads .pfc.bidx directly in C++
    │  Selects relevant blocks
    │
    ▼
subprocess call → pfc_jsonl binary
    │  Decompresses selected blocks
    │  Streams JSON lines back via stdout
    │
    ▼
JSON lines → DuckDB table (one row = one log line)
```

The extension is a thin open-source wrapper. The compression engine stays in the binary.
Source: [github.com/ImpossibleForge/pfc-duckdb](https://github.com/ImpossibleForge/pfc-duckdb)

---

## Fluent Bit Integration Architecture

```
Fluent Bit → [OUTPUT tcp, json_lines] → pfc_forwarder.py → pfc_jsonl compress → .pfc archives
```

`pfc_forwarder.py` is a lightweight Python daemon (standard library only) that:
1. Listens on TCP port 5170
2. Buffers incoming JSON log records
3. Rotates and compresses when a size or time threshold is reached

Source: [github.com/ImpossibleForge/pfc-fluentbit](https://github.com/ImpossibleForge/pfc-fluentbit)

---

## Python Package Architecture

The `pfc` Python package (`pip install pfc-jsonl`) is a thin wrapper that calls the `pfc_jsonl` binary as a subprocess:

```python
pfc.compress("app.jsonl", "app.pfc")    # → pfc_jsonl compress ...
pfc.query("app.pfc", from_ts, to_ts)   # → pfc_jsonl query ...
```

Source: [github.com/ImpossibleForge/pfc-py](https://github.com/ImpossibleForge/pfc-py)

---

## Version History

| Version | Key Change |
|---------|-----------|
| v3.0 | Current compression pipeline introduced |
| v3.2 | Block-level random access + `.pfc.bidx` index |
| v3.3 | Community Mode, Docker CLI |
| v3.4 | `seek-blocks` primitive, C++-readable index, DuckDB Extension |
