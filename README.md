# PFC-JSONL — High-Ratio JSONL Compressor with Block-Level Random Access

> Compress structured log files (JSONL, JSON Lines) to **9.6% of original size** — with timestamp-based queries that download only the blocks you need.

[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://hub.docker.com/r/impossibleforge/pfc-jsonl)
[![Version](https://img.shields.io/badge/Version-3.3-green.svg)]()

---

## Why PFC-JSONL?

| Tool | Ratio on JSONL Logs | Random Access | Egress (10TB, 1h query) |
|------|---------------------|---------------|------------------------|
| **PFC-JSONL** | **9.6%** | ✅ Block-level | **~29 MB** |
| gzip-6 | 21.7% | ❌ Full download | ~1.43 TB |
| zstd-3 | 18.8% | ❌ Full download | ~1.43 TB |
| zstd-19 | 10.0% | ❌ Full download | ~1 TB |
| bzip2-9 | 13.1% | ❌ Full download | ~1.3 TB |

**67% smaller archives than gzip-6. 10 TB archive, 1-hour query: download ~29 MB instead of 1.43 TB.**

---

## How It Works

PFC divides JSONL logs into independent 32 MiB blocks (BWT → MTF → RLE → rANS). Each block's timestamp range is stored in a `.pfc.idx` file. To query a time range, only the relevant blocks are fetched via HTTP Range requests — no full archive download needed.

Works with **any S3-compatible storage** via pre-signed URLs: AWS S3, S3 Glacier Instant, Azure Blob (SAS), Cloudflare R2, Hetzner, MinIO, Wasabi.

---

## Quick Start

```bash
# Pull the image
docker pull impossibleforge/pfc-jsonl:v3.3

# Compress a JSONL file (requires license.key — see below)
docker run --rm -v $(pwd):/data impossibleforge/pfc-jsonl:v3.3 \
    pfc compress /data/events.jsonl /data/events.pfc

# Query a time range (from pre-signed URL)
docker run --rm -v $(pwd):/data impossibleforge/pfc-jsonl:v3.3 \
    pfc query --from 2024-01-15T06:00 --to 2024-01-15T08:00 \
    --idx-url "https://your-presigned-idx-url" \
    "https://your-presigned-pfc-url"

# Decompress
docker run --rm -v $(pwd):/data impossibleforge/pfc-jsonl:v3.3 \
    pfc decompress /data/events.pfc /data/restored.jsonl
```

---

## Commands

| Command | Description |
|---------|-------------|
| `pfc compress <input> <output>` | Compress JSONL file → `.pfc` + `.pfc.idx` |
| `pfc decompress <input> <output>` | Full decompression |
| `pfc query --from X --to Y --idx-url <url> <url>` | Fetch only matching blocks |
| `pfc seek-block N <input> [output]` | Extract single block by number |
| `pfc info <input>` | Show block table + timestamp ranges |

---

## Input Format

PFC-JSONL expects one JSON object per line with a timestamp field:

```json
{"timestamp": "2024-01-15T06:32:11Z", "level": "ERROR", "service": "api", "event": "timeout"}
{"timestamp": "2024-01-15T06:32:12Z", "level": "INFO", "service": "db", "event": "query_ok"}
```

Supported timestamp fields: `timestamp`, `ts`, `time`, `@timestamp` (ISO 8601 or Unix epoch).

---

## License & Trial

PFC-JSONL is **proprietary software**. A free 30-day trial is available.

📧 **Request trial:** impossibleforge@gmail.com
🛒 **Buy license:** [impossibleforge.lemonsqueezy.com/buy/pfc-jsonl](https://impossibleforge.lemonsqueezy.com/buy/pfc-jsonl)

Without a valid `license.key` file, the tool displays:
```
[PFC-JSONL] No license.key found. Contact impossibleforge@gmail.com
```

---

## Documentation

- [Full Documentation](docs/PFC_JSONL_DOCS.md)
- [Setup Guide](docs/SETUP_GUIDE.txt)
- [Third Party Licenses](docs/THIRD_PARTY_LICENSES.txt)

---

## White-Label & Partnership

Interested in integrating PFC-JSONL into your product?
We offer white-label licensing and OEM partnerships.

📧 **Contact:** impossibleforge@gmail.com

---

*Built by [ImpossibleForge](https://github.com/ImpossibleForge) — proprietary compression research.*
