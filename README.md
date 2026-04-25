# PFC-JSONL — High-Ratio JSONL Compressor with Block-Level Random Access

Most log archives are compressed. Most queries touch one hour. Most tools make you decompress everything anyway.

PFC-JSONL stores a block index alongside every compressed file. Query a time window with DuckDB and only the relevant blocks are decompressed — the rest stays on disk, untouched.

> **~9% compression ratio** (25% smaller than gzip, 37% smaller than zstd on typical JSONL logs).
> **30×–700× faster** time-range queries vs. full-file decompression.

[![License: Free for personal use](https://img.shields.io/badge/License-Free%20for%20personal%20use-blue.svg)](https://github.com/ImpossibleForge/pfc-jsonl/blob/main/LICENSE)
[![Version](https://img.shields.io/badge/Version-3.4.4-green.svg)]()
[![DuckDB Extension](https://img.shields.io/badge/DuckDB-Extension-orange.svg)](https://github.com/ImpossibleForge/pfc-duckdb)
[![Fluent Bit](https://img.shields.io/badge/Fluent%20Bit-Ready-green.svg)](https://github.com/ImpossibleForge/pfc-fluentbit)
[![Vector](https://img.shields.io/badge/Vector-Sink-6B40BF.svg)](https://github.com/ImpossibleForge/pfc-vector)
[![Telegraf](https://img.shields.io/badge/Telegraf-Plugin-00acac.svg)](https://github.com/ImpossibleForge/pfc-telegraf)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-Collector-425CC7.svg)](https://github.com/ImpossibleForge/pfc-otel-collector)
[![PyPI](https://img.shields.io/badge/PyPI-pfc--jsonl-blue.svg)](https://pypi.org/project/pfc-jsonl/)
[![Awesome DuckDB](https://awesome.re/mentioned-badge.svg)](https://github.com/davidgasquez/awesome-duckdb)
[![Awesome Observability](https://awesome.re/mentioned-badge.svg)](https://github.com/adriannovegil/awesome-observability)

---

## Why PFC-JSONL?

| Tool | Ratio on JSONL Logs | Block Access | 10 TB archive, 1h query |
|------|---------------------|--------------|-------------------------|
| **PFC-JSONL** | **~9 %** | ✅ Block-level | **~26 MB download** |
| gzip | ~12 % | ❌ Full file | ~1.43 TB |
| zstd | ~14.2 % | ❌ Full file | ~1.43 TB |

**PFC-JSONL default is 25% smaller than gzip and 37% smaller than zstd at typical settings.**
Ratios measured on 200 MB JSONL log data (8 services, mixed log levels, ~961K lines), PFC-JSONL v3.4.

---

## Install

### Linux x86_64 & macOS ARM64 — Direct Binary

**Linux x86_64:**
```bash
curl -L https://github.com/ImpossibleForge/pfc-jsonl/releases/latest/download/pfc_jsonl-linux-x64 \
     -o /usr/local/bin/pfc_jsonl && chmod +x /usr/local/bin/pfc_jsonl

pfc_jsonl --help
```

**macOS (Apple Silicon M1/M2/M3/M4):**
```bash
curl -L https://github.com/ImpossibleForge/pfc-jsonl/releases/latest/download/pfc_jsonl-macos-arm64 \
     -o /usr/local/bin/pfc_jsonl && chmod +x /usr/local/bin/pfc_jsonl

pfc_jsonl --help
```

> **macOS Intel (x64):** Binary coming soon. Contact: info@impossibleforge.com
> **Windows:** No native binary. Use WSL2 or a Linux machine.

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
See [pfc-duckdb on GitHub](https://github.com/ImpossibleForge/pfc-duckdb) for manual install instructions.

---

## Ingest — Send Data to PFC

Plug PFC-JSONL into your existing logging or metrics pipeline. All ingest tools buffer data locally, compress when the buffer is full, and optionally upload to S3.

| Tool | Protocol / Format | Port | Repo |
|------|-------------------|------|------|
| **[pfc-fluentbit](https://github.com/ImpossibleForge/pfc-fluentbit)** | Fluent Bit output plugin | — | Fluent Bit → `.pfc` |
| **[pfc-vector](https://github.com/ImpossibleForge/pfc-vector)** | HTTP sink (JSON / NDJSON) | 8766 | Vector.dev → `.pfc` |
| **[pfc-telegraf](https://github.com/ImpossibleForge/pfc-telegraf)** | HTTP (InfluxDB line protocol + JSON) | 8767 | Telegraf → `.pfc` |
| **[pfc-otel-collector](https://github.com/ImpossibleForge/pfc-otel-collector)** | OTLP/HTTP (logs, traces, metrics) | 4318 | OpenTelemetry → `.pfc` |
| **[pfc-kafka-consumer](https://github.com/ImpossibleForge/pfc-kafka-consumer)** | Kafka / Redpanda consumer | — | Kafka topic → `.pfc` |
| **[pfc-gateway](https://github.com/ImpossibleForge/pfc-gateway)** ↕ | HTTP REST `POST /ingest` | 8765 | Any source → `.pfc` (+ query) |

> **pfc-gateway** is bidirectional — it accepts ingest via `POST /ingest` and serves queries via `POST /query`. No DuckDB required.

---

## Query — Read PFC Archives

### DuckDB Extension
The fastest way to query `.pfc` archives locally — see the [DuckDB Extension](#duckdb-extension) section above.

### pfc-gateway — HTTP REST API

Query `.pfc` archives over HTTP without DuckDB — works with any language, curl, Grafana, or PowerBI:

```bash
# Start the gateway (points at your archive directory)
PFC_ARCHIVE_DIR=/var/lib/pfc PFC_API_KEY=secret \
  python3 pfc_gateway.py --port 8765

# Query a time range
curl -X POST http://localhost:8765/query \
  -H "x-api-key: secret" \
  -H "Content-Type: application/json" \
  -d '{
    "file": "/var/lib/pfc/logs_20260101.pfc",
    "from_ts": "2026-01-01T10:00:00Z",
    "to_ts":   "2026-01-01T11:00:00Z"
  }'

# Query multiple files at once
curl -X POST http://localhost:8765/query/batch \
  -H "x-api-key: secret" \
  -H "Content-Type: application/json" \
  -d '{"files": ["/var/lib/pfc/logs_20260101.pfc", "/var/lib/pfc/logs_20260102.pfc"]}'
```

Also supports **Grafana SimpleJSON** — point the Grafana data source at `http://localhost:8765/grafana`.
See [pfc-gateway on GitHub](https://github.com/ImpossibleForge/pfc-gateway) for full documentation.

---

## Migrate Existing Archives

Already have logs stored as gzip, zstd, bzip2, or lz4 — on disk, on S3, on Azure, or on GCS?

**[pfc-migrate](https://github.com/ImpossibleForge/pfc-migrate)** converts them in one command, directly in your storage (no egress charges):

```bash
pip install pfc-migrate[all]

# Local
pfc-migrate convert --dir /var/log/archive/ --output-dir /var/log/pfc/ -v

# S3
pfc-migrate s3 --bucket my-logs --prefix 2025/ --out-bucket my-logs-pfc --out-prefix pfc/

# Azure Blob
pfc-migrate azure --container my-logs --prefix 2025/ --out-container my-logs-pfc --connection-string "..."

# GCS
pfc-migrate gcs --bucket my-logs --prefix 2025/ --out-bucket my-logs-pfc
```

---

## Python Package

Use the [pfc Python package](https://github.com/ImpossibleForge/pfc-py) (PyPI: `pfc-jsonl`) to compress, decompress, and query `.pfc` files from Python:

```bash
pip install pfc-jsonl
```

```python
import pfc

pfc.compress("logs/app.jsonl", "logs/app.pfc")
pfc.query("logs/app.pfc",
          from_ts="2026-01-15T08:00:00",
          to_ts="2026-01-15T09:00:00",
          output_path="logs/morning.jsonl")
```

---

## Commands

| Command | Description |
|---------|-------------|
| `pfc_jsonl compress <input> <output>` | Compress JSONL → `.pfc` + `.pfc.bidx` |
| `pfc_jsonl decompress <input> <output>` | Full decompression |
| `pfc_jsonl query <input> --from X --to Y --out <output>` | Decompress blocks matching time range |
| `pfc_jsonl seek-block N <input> [output]` | Extract single block by index |
| `pfc_jsonl seek-blocks <input> --blocks N [N...]` | Extract multiple blocks (DuckDB primitive) |
| `pfc_jsonl info <input>` | Show block table + timestamp ranges |

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

## Related Repos

**Ingest**
- [pfc-fluentbit](https://github.com/ImpossibleForge/pfc-fluentbit) — Fluent Bit output plugin → PFC
- [pfc-vector](https://github.com/ImpossibleForge/pfc-vector) — Vector.dev HTTP sink → PFC (Rust)
- [pfc-telegraf](https://github.com/ImpossibleForge/pfc-telegraf) — Telegraf HTTP output plugin → PFC
- [pfc-otel-collector](https://github.com/ImpossibleForge/pfc-otel-collector) — OpenTelemetry OTLP/HTTP → PFC
- [pfc-kafka-consumer](https://github.com/ImpossibleForge/pfc-kafka-consumer) — Kafka / Redpanda consumer → PFC

**Query & Gateway**
- [pfc-gateway](https://github.com/ImpossibleForge/pfc-gateway) — HTTP REST API: ingest + query, no DuckDB required
- [pfc-duckdb](https://github.com/ImpossibleForge/pfc-duckdb) — DuckDB community extension for SQL queries on PFC files

**Archive & Migration**
- [pfc-migrate](https://github.com/ImpossibleForge/pfc-migrate) — convert gzip/zstd/lz4/bz2 archives → PFC (local, S3, Azure, GCS)
- [pfc-archiver-cratedb](https://github.com/ImpossibleForge/pfc-archiver-cratedb) — autonomous archive daemon for CrateDB
- [pfc-archiver-questdb](https://github.com/ImpossibleForge/pfc-archiver-questdb) — autonomous archive daemon for QuestDB

**SDK**
- [pfc-py](https://github.com/ImpossibleForge/pfc-py) — Python client library (PyPI: `pfc-jsonl`)

---

## License

PFC-JSONL is **free for personal and open-source use**.

Commercial use (production pipelines, paid services, or business operations) requires a license.
Contact: **info@impossibleforge.com**

---

*Built by [ImpossibleForge](https://github.com/ImpossibleForge)*
