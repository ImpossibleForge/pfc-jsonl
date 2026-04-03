# PFC-JSONL — High-Ratio JSONL Compressor with Block-Level Random Access

> Compress structured log files (JSONL, JSON Lines) to **~10% of original size** — with timestamp-based queries that decompress only the blocks you need.

[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://hub.docker.com/r/impossibleforge/pfc-jsonl)
[![Version](https://img.shields.io/badge/Version-3.4-green.svg)]()
[![DuckDB Extension](https://img.shields.io/badge/DuckDB-Extension-orange.svg)](https://github.com/ImpossibleForge/pfc-duckdb)

---

## Why PFC-JSONL?

| Tool | Ratio on JSONL Logs | Block Access | 10 TB archive, 1h query |
|------|---------------------|--------------|-------------------------|
| **PFC-JSONL** | **~10 %** | ✅ Block-level | **~29 MB download** |
| gzip | ~14.5 % | ❌ Full file | ~1.43 TB |
| zstd | ~16 % | ❌ Full file | ~1.43 TB |

**PFC-JSONL is 26–34 % smaller than gzip/zstd on typical structured log data.**
Ratios measured on real JSONL log data (8 services, mixed log levels, 115 MB / 3 M lines).

---

## What's New in v3.4

- **DuckDB Extension** — query `.pfc` files directly from SQL via `read_pfc_jsonl()` ([pfc-duckdb](https://github.com/ImpossibleForge/pfc-duckdb))
- **Binary index format** `.pfc.bidx` — fixed-size 32-byte-per-block index, directly readable by the DuckDB C++ extension
- **`seek-blocks`** primitive — decompress multiple specific blocks in one call (used by DuckDB extension)
- **Community Mode** — no license key required for ≤ 5 GB decompressed output per UTC day

---

## Install

### Option A: Direct Binary (Linux x86_64)

```bash
curl -L https://github.com/ImpossibleForge/pfc-jsonl/releases/latest/download/pfc_jsonl-linux-x64 \
     -o /usr/local/bin/pfc_jsonl && chmod +x /usr/local/bin/pfc_jsonl

pfc_jsonl --help
```

### Option B: Docker

```bash
docker pull impossibleforge/pfc-jsonl:v3.4

# Compress
docker run --rm -v $(pwd):/data impossibleforge/pfc-jsonl:v3.4 \
    pfc_jsonl compress /data/events.jsonl /data/events.pfc

# Query by time range
docker run --rm -v $(pwd):/data impossibleforge/pfc-jsonl:v3.4 \
    pfc_jsonl query --from 2025-01-15T06:00 --to 2025-01-15T08:00 \
    /data/events.pfc /data/result.jsonl
```

---

## DuckDB Extension

Query `.pfc` files directly from DuckDB SQL — no intermediate decompression step:

```sql
INSTALL pfc FROM community;
LOAD pfc;
LOAD json;

-- Read all lines
SELECT line->>'$.level' AS level, line->>'$.message' AS msg
FROM read_pfc_jsonl('/path/to/events.pfc')
LIMIT 10;

-- Block-level timestamp filter: only decompress relevant blocks
SELECT count(*)
FROM read_pfc_jsonl(
    '/path/to/events.pfc',
    ts_from = epoch(TIMESTAMPTZ '2026-01-01 00:00:00+00'),
    ts_to   = epoch(TIMESTAMPTZ '2026-01-02 00:00:00+00')
);
```

The DuckDB extension calls `pfc_jsonl` as a subprocess. Install the binary first (see above).
→ [pfc-duckdb extension on GitHub](https://github.com/ImpossibleForge/pfc-duckdb)

---

## Commands

| Command | Description |
|---------|-------------|
| `pfc_jsonl compress <input> <output>` | Compress JSONL → `.pfc` + `.pfc.bidx` |
| `pfc_jsonl decompress <input> <output>` | Full decompression |
| `pfc_jsonl query --from X --to Y <input>` | Decompress blocks matching time range |
| `pfc_jsonl seek-block N <input> [output]` | Extract single block by index |
| `pfc_jsonl seek-blocks <input> --blocks N [N...]` | Extract multiple blocks (DuckDB primitive) |
| `pfc_jsonl info <input>` | Show block table + timestamp ranges |

---

## Community Mode

Starting with v3.4, PFC-JSONL ships with a built-in free tier:

- **5 GB of decompressed output per UTC day** — tracked locally in `~/.pfc/usage.json`
- **No account, no signup, no phone-home** — nothing leaves your machine
- Works for `decompress`, `query`, `seek-block`, `seek-blocks`
- `compress` requires a license key

For production use (> 5 GB/day): contact **impossibleforge@gmail.com**

---

## Input Format

One JSON object per line with a timestamp field:

```json
{"timestamp": "2025-01-15T06:32:11Z", "level": "ERROR", "service": "api", "msg": "timeout"}
{"timestamp": "2025-01-15T06:32:12Z", "level": "INFO",  "service": "db",  "msg": "query_ok"}
```

Supported timestamp fields: `timestamp`, `ts`, `time`, `@timestamp` (ISO 8601 or Unix epoch seconds).

---

## How It Works

PFC divides JSONL logs into independent blocks (configurable, default 32 MiB).
Each block is compressed with: **BWT → MTF → RLE → rANS O1**.
Block timestamp ranges are stored in `.pfc.bidx` (32 bytes/block, binary, C++-readable).

To query a time range, only the relevant blocks are decompressed — the rest is never read.

---

## License

PFC-JSONL is **proprietary software**.

- **Community Mode** (v3.4+): free for ≤ 5 GB decompressed/day, no key required
- **Production license**: unlimited throughput — contact **impossibleforge@gmail.com**
- **White-label / OEM**: integrations and partnership available on request

---

*Built by [ImpossibleForge](https://github.com/ImpossibleForge)*
