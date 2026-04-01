# PFC-JSONL v3.3 — Technical Documentation

**Product:** PFC-JSONL | **Version:** 3.3 | **By:** ImpossibleForge
**Contact:** impossibleforge@gmail.com

---

## Table of Contents

1. [What is PFC-Log?](#1-what-is-pfc-log)
2. [Quick Start](#2-quick-start)
3. [Commands Reference](#3-commands-reference)
4. [Compression Presets](#4-compression-presets)
5. [Random Access Features](#5-random-access-features)
6. [Timestamp Index](#6-timestamp-index)
7. [S3 / Cloud Access via Pre-Signed URLs](#7-s3--cloud-access-via-pre-signed-urls)
8. [Performance & Benchmarks](#8-performance--benchmarks)
9. [How it Works](#9-how-it-works)
10. [Requirements & Limits](#10-requirements--limits)
11. [License & Privacy](#11-license--privacy)
12. [FAQ](#12-faq)

---

## 1. What is PFC-Log?

PFC-Log is a **lossless log file compressor** specialized for Apache and Nginx access logs (Common Log Format / Combined Log Format).

**Key facts:**
- Compresses Apache/Nginx logs to **~6.5% of original size** — roughly half the size of gzip
- 100% lossless — byte-for-byte identical decompression (MD5-verified)
- No code changes required in your application
- Ships as a Docker image — 3-minute setup
- Supports **random access**: query time ranges, decompress individual blocks, fetch from any cloud storage without downloading the full file
- v3.3: **no SDK required** — cloud access via pre-signed URLs (AWS S3, Azure Blob, Cloudflare R2, Hetzner, MinIO)

**Compression comparison (1 GB Apache/Nginx access log):**

| Tool | Ratio | Compress Speed |
|------|-------|----------------|
| **PFC-Log default** | **6.73%** | **14.5 MB/s** |
| gzip -6 | 14.29% | ~75 MB/s |
| zstd -3 | 14.32% | ~320 MB/s |
| zstd -19 | 8.43% | 0.45 MB/s |
| bzip2 -9 | 7.39% | ~6.4 MB/s |

> PFC-Log archives are **53% smaller than gzip** at competitive speed. zstd-3 has the same ratio as gzip with no improvement.

---

## 2. Quick Start

### Prerequisites

- Linux x86_64 (Intel/AMD) — ARM64 not currently supported
- Docker Engine 20.10+
- ~200 MB disk for the image + your log files

### Step 1 — Pull the Docker image

```bash
docker pull impossibleforge/pfc-jsonl:v3.3
```

### Step 2 — Place your license key

Copy `license.key` into the same directory as your log files.

### Step 3 — Compress

```bash
docker run --rm \
  -v $(pwd):/data \
  pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc
```

### Step 4 — Decompress

```bash
docker run --rm \
  -v $(pwd):/data \
  pfc-jsonl:v3.3-final \
  decompress /data/access.pfc /data/restored.log
```

The restored file is byte-for-byte identical to the original.

---

## 3. Commands Reference

### compress

Compress a log file:

```bash
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc [OPTIONS]

Options:
  --level fast|default|max   Compression preset (default: default)
  --no-index                 Skip generating the .pfc.idx timestamp index
  --workers N                Number of parallel workers (default: auto)
  --verbose                  Show per-block progress
```

Example with options:

```bash
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc --level max --verbose
```

### decompress

Decompress a .pfc file:

```bash
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  decompress /data/access.pfc /data/restored.log
```

### info

Show information about a compressed file:

```bash
# Basic info (size, ratio, block count, timestamp range):
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  info /data/access.pfc

# Detailed block structure:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  info /data/access.pfc --blocks
```

Example output (`--blocks`):

```
Block 0:  offset=42  size=6,814,308B  2025-01-01T00:00:12 → 2025-04-15T23:58:44
Block 1:  offset=6,814,350  size=6,796,308B  2025-04-15T23:58:45 → 2025-12-31T23:59:58
```

> Block 0 is always the preprocessor dictionary (very small). User log data starts at Block 1.

### seek-block

Decompress a single block without touching the rest of the file:

```bash
# To stdout (pipe-friendly):
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  seek-block 1 /data/access.pfc -

# Filter on the fly:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  seek-block 1 /data/access.pfc - | grep "ERROR"

# To a file:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  seek-block 1 /data/access.pfc /data/block1.log
```

Useful for spot-checking recent logs or sampling a specific time window.

> **Note on block boundaries:** Because log preprocessing tokens can span block edges, seek-block output may differ by a few bytes from the exact corresponding slice of a full decompression. For byte-exact results, use `decompress`.

### query

Search for log lines within a specific time window. Only the relevant blocks are decompressed:

```bash
# Output to stdout:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  query /data/access.pfc \
  --from "2025-03-01T00:00:00" \
  --to "2025-03-01T23:59:59"

# Output to file:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  query /data/access.pfc \
  --from "2025-03-01T00:00:00" \
  --to "2025-03-01T23:59:59" \
  --out /data/march1.log

# Show block info only (no decompression, omit --out):
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  query /data/access.pfc \
  --from "2025-03-01T00:00:00" \
  --to "2025-03-01T23:59:59"

# Decompress to stdout:
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  query /data/access.pfc \
  --from "2025-03-01T00:00:00" \
  --to "2025-03-01T23:59:59" \
  --out -
```

**Timestamp format:** ISO-8601 UTC (`YYYY-MM-DDTHH:MM:SS`).

Requires the `.pfc.idx` index file alongside the `.pfc` file (auto-generated during compression). If missing, falls back to full-file decompression.

---

## 4. Compression Presets

| Preset | Block Size | Compress Speed | Ratio | Use Case |
|--------|------------|----------------|-------|----------|
| `fast` | 8 MiB | ~33 MB/s | ~7.0% | Real-time pipelines, quick archiving |
| `default` | 32 MiB | ~16.8 MB/s | ~6.49% | Best balance — recommended |
| `max` | 128 MiB | ~9 MB/s | ~5.8% | Maximum compression, archiving |

The `default` preset is recommended for most use cases. Use `max` only when storage savings matter more than compression time.

**Setting a preset:**

```bash
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc --level max
```

---

## 5. Random Access Features

PFC-Log uses block-level random access. Each `.pfc` file is divided into independent compressed blocks. This enables:

- **Inspect** block structure and timestamp ranges without decompressing
- **Seek** to a specific block and decompress only that block
- **Query** a time window — only the blocks overlapping that range are loaded
- **Fetch from any cloud storage** via pre-signed URLs without downloading the full file

### Why this is possible with PFC-Log but not gzip

PFC-Log compresses in **independent blocks** (32 MiB by default). Each block is self-contained — it can be decompressed without any other block. The file header records the compressed size of each block, enabling direct seeking.

gzip is a stream compressor — you must decompress from byte 0 to reach any later position. zstd has an optional "seekable" extension, but it is not standard and rarely deployed.

### Block structure

```
[22-byte file header]
[4 bytes × num_blocks: compressed block sizes]
[Block 0: preprocessor dictionary — small]
[Block 1: first chunk of log data]
[Block 2: second chunk of log data]
...
```

### Random access use case example

You have a 10 TB archive on S3 from the past year. An incident happened on March 14 between 14:00 and 15:00.

**Without PFC-Log (gzip):** You must download all 10 TB to find those 60 minutes of logs.

**With PFC-JSONL v3.3:** The `.pfc.idx` index tells you which blocks contain March 14 14:00-15:00. You download only those blocks (~24 MB). S3 egress: ~$0.002 instead of ~$900. That is a **99.9% reduction**.

---

## 6. Timestamp Index

During compression, PFC-Log automatically creates a `.pfc.idx` file alongside the `.pfc` file:

```
access.pfc      ← compressed data
access.pfc.idx  ← timestamp index (auto-generated)
```

The index records the **timestamp range** covered by each block. This enables the `query` command to skip irrelevant blocks entirely.

**Properties:**
- Size: ~1 KB regardless of file size
- Always kept alongside the `.pfc` file
- Auto-detected timestamp format: Apache/Nginx Common Log Format timestamps

**To compress without generating an index:**

```bash
docker run --rm -v $(pwd):/data pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc --no-index
```

If the `.idx` file is missing, `query` falls back to full-file decompression.

---

## 7. S3 / Cloud Access via Pre-Signed URLs

PFC-JSONL v3.3 supports **direct access** to `.pfc` files stored in any cloud storage — without downloading the full archive first, and **without any SDK or cloud credentials inside the container**.

Works with:
- AWS S3 / S3 Glacier Instant Retrieval
- Azure Blob Storage (SAS URLs)
- Cloudflare R2, Hetzner Object Storage, Wasabi, MinIO, Backblaze B2

### How it works

You generate a pre-signed URL using your existing cloud tools. The URL carries its own authentication as query parameters. PFC-Log uses standard HTTP `Range` requests against that URL to fetch only the required blocks — no SDK, no credentials inside Docker.

**Egress savings:**
- 10 TB archive, 1-hour query → PFC downloads ~24 MB
- gzip equivalent → must download entire 10 TB
- At $0.09/GB: gzip = ~$900 per query. PFC = $0.002.

### Step 1 — Upload both files to cloud storage

```bash
# AWS S3:
aws s3 cp access.pfc      s3://my-bucket/logs/
aws s3 cp access.pfc.idx  s3://my-bucket/logs/

# Azure:
az storage blob upload -f access.pfc      -c logs -n access.pfc
az storage blob upload -f access.pfc.idx  -c logs -n access.pfc.idx
```

Always upload **both** the `.pfc` and `.pfc.idx` files.

### Step 2 — Generate pre-signed URLs

```bash
# AWS CLI (valid 1 hour):
aws s3 presign s3://my-bucket/logs/access.pfc     --expires-in 3600
aws s3 presign s3://my-bucket/logs/access.pfc.idx --expires-in 3600

# Azure CLI (SAS URL):
az storage blob generate-sas --full-uri \
  --account-name ACCOUNT --container-name logs \
  --name access.pfc --permissions r --expiry 2025-12-31T00:00:00Z

# rclone (all providers):
rclone link remote:my-bucket/logs/access.pfc
```

### Step 3 — Query directly from cloud

```bash
# Time-range query (most common use case):
docker run --rm \
  -v $(pwd):/data \
  pfc-jsonl:v3.3-final \
  s3-fetch "https://my-bucket.s3.amazonaws.com/logs/access.pfc?X-Amz-..." \
    --idx-url "https://my-bucket.s3.amazonaws.com/logs/access.pfc.idx?X-Amz-..." \
    --from "2025-03-01T00:00:00" --to "2025-03-01T23:59:59" \
    --out /data/march1.log

# Show file metadata without downloading data blocks:
docker run --rm \
  pfc-jsonl:v3.3-final \
  s3-info "https://my-bucket.s3.amazonaws.com/logs/access.pfc?X-Amz-..."

# Fetch specific blocks by index:
docker run --rm \
  -v $(pwd):/data \
  pfc-jsonl:v3.3-final \
  s3-fetch "https://my-bucket.s3.amazonaws.com/logs/access.pfc?X-Amz-..." \
    --blocks 2 3 --out /data/blocks23.log
```

### S3 Glacier

S3 Glacier **Instant Retrieval** supports HTTP Range requests natively — PFC `s3-fetch` works directly.

For Glacier **Flexible / Deep Archive**, restore the object first (AWS-side operation), then use the pre-signed URL of the restored copy.

---

## 8. Performance & Benchmarks

### Benchmark environment

- 1 GB Apache/Nginx access log (real production data, 3-run average)
- 6-core server, 32 MiB blocks (default preset)
- 2 workers (auto-detected based on available RAM)

### Results

| Tool | Ratio | Compress | Decompress | Notes |
|------|-------|----------|------------|-------|
| **PFC-Log default** | **6.73%** | **14.5 MB/s** | **32 MB/s** | Block-parallel |
| gzip -6 | 14.29% | ~75 MB/s | 200+ MB/s | Single-threaded |
| zstd -3 | 14.32% | ~320 MB/s | 900+ MB/s | Same ratio as gzip! |
| zstd -19 | 8.43% | 0.45 MB/s | 900+ MB/s | 26 hours per TB |
| bzip2 -9 | 7.39% | ~6.4 MB/s | ~35 MB/s | Single-threaded |

**PFC-Log archives are 53% smaller than gzip-6.** zstd-3 offers no storage advantage over gzip.

### RAM usage

| Operation | RAM (default preset, 32 MiB blocks) |
|-----------|--------------------------------------|
| Compress | ~555 MiB (2 workers) |
| Decompress | ~620 MiB (2 workers) |

RAM usage is **bounded**: `block_size × 6 × num_workers`. No unbounded memory growth regardless of file size.

### Scaling with more cores

PFC-Log compression is **block-parallel** — each worker processes an independent block. Worker count is auto-detected:

```
workers = min(CPU_cores, num_blocks, available_RAM ÷ (block_size × 6))
```

A 6-core server with a 200 MB file runs 2 workers (auto). A larger file on the same server would use up to 6 workers.

---

## 9. How it Works

PFC-Log uses a pipeline of algorithms designed specifically for log file structure:

```
Raw log  →  [Preprocessor]  →  [BWT]  →  [MTF]  →  [RLE]  →  [rANS O1]  →  .pfc
```

1. **Preprocessor** — Tokenizes repetitive log structure (IP addresses, timestamps, HTTP methods, status codes, path patterns) into compact 2-byte tokens. Reduces data by ~30-40% before the main compression pass.

2. **BWT (Burrows-Wheeler Transform)** — Reorders bytes to cluster similar patterns together. Uses `libsais`, an SA-IS BWT implementation.

3. **MTF (Move-to-Front)** — Transforms the BWT output into a sequence with small values clustered near 0. Optimizes entropy coder input.

4. **RLE (Run-Length Encoding)** — Compresses runs of identical bytes (common after MTF on log data).

5. **rANS O1 (range Asymmetric Numeral Systems, Order-1)** — Arithmetic entropy coder with 1-symbol context. Custom C implementation.

Each stage is applied **independently per block**, enabling parallel compression and random access.

### File format overview

```
Bytes 0-21:    File header (magic, version, num_blocks, flags)
Bytes 22-...:  Block size table (4 bytes per block)
Block 0:       Preprocessor dictionary (token mappings)
Block 1..N:    Compressed data blocks (each independently decompressible)
```

File header is 22 bytes fixed + 4 bytes per block. On a 1 GB file with 32 blocks: 22 + 128 = 150 bytes overhead (0.000015%).

---

## 10. Requirements & Limits

### System requirements

| Requirement | Value |
|-------------|-------|
| OS | Linux x86_64 (Intel/AMD) |
| Docker | Engine 20.10+ |
| RAM | 512 MB minimum (1 GB recommended) |
| Disk | ~200 MB for image + input size + ~10% for output |

> ARM64 is not currently supported (no AWS Graviton, no Apple Silicon).

### Disk space example

A 10 GB log file compressed at 6.49%:
- Output: ~650 MB
- Working space needed: ~10.7 GB total

### No practical file size limit

The Docker image processes files of any size. Block-based processing means even 100 GB+ files are handled without loading the entire file into RAM.

---

## 11. License & Privacy

### License

PFC-Log requires a valid `license.key` file. Place it in the same directory as your log files (the directory mounted as `/data`).

- **Trial licenses:** 30 days from issue date
- **Lifetime licenses:** No expiry
- Contact `impossibleforge@gmail.com` for license issues or purchase

### Privacy — No network connections

PFC-Log makes **zero network connections** for compress/decompress/query operations:

- No telemetry, no usage tracking, no data collection
- No license server — trial validity checked against local system clock only
- Your log files never leave your server
- Fully GDPR-compliant by design
- `s3-fetch` only connects when you explicitly provide a pre-signed URL

Verify with `--network none`:

```bash
docker run --rm --network none \
  -v $(pwd):/data \
  pfc-jsonl:v3.3-final \
  compress /data/access.log /data/access.pfc
```

It works identically with no network access.

### Third-party components

- **libsais** — BWT implementation by Ilya Grebnov. Apache 2.0 License. Bundled in the Docker image.
- **urllib** — Python Standard Library. PSF License. Used for cloud access (pre-signed URLs). No installation required.
- **boto3 / azure-storage-blob** — Optional, for S3/Azure cloud access only. Not bundled. Install separately if needed.

---

## 12. FAQ

**Q: Is compression really lossless?**
Yes. Every release is verified with MD5 round-trip tests:
`MD5(original) == MD5(decompress(compress(original)))` on all test files before release.

**Q: What if ImpossibleForge no longer exists in 3 years?**
The decompressor binary is included in the Docker image you receive. You can freeze that image and use it indefinitely — no server connection required. The .pfc format is documented and readable with ~50 lines of Python.

**Q: What happens if one block is corrupted?**
Only that block is lost — the other blocks remain intact and readable. This is better than gzip, where any corruption can make the entire file unreadable. For critical archives, combine with S3 Versioning + Integrity Checksums.

**Q: Can I compress files from a cron job / Airflow / Lambda?**
Yes. Use the Docker command in any shell script or BashOperator. Lambda may be too CPU-constrained for large files (BWT is compute-intensive) — ECS or EC2 is recommended for large workloads.

**Q: What about files with mixed log formats?**
PFC-Log is optimized for Apache/Nginx Common Log Format (CLF). Mixed formats will still compress correctly and losslessly, but ratio will be worse than on pure CLF logs.

**Q: How do I verify the tool makes no network connections?**
Run with `--network none` as shown in the Privacy section. The tool functions identically.

**Q: Does my compressed file work with future versions?**
Yes. The .pfc format includes a version number. Future versions will always be able to read .pfc files created by v3.2.

**Q: What is the file size overhead of the .pfc format?**
22 bytes fixed header + 4 bytes per block. A 1 GB file with 32 blocks = 150 bytes overhead. Negligible.

---

## Full Command Reference

```bash
docker run pfc-jsonl:v3.3-final --help
docker run pfc-jsonl:v3.3-final compress --help
docker run pfc-jsonl:v3.3-final decompress --help
docker run pfc-jsonl:v3.3-final info --help
docker run pfc-jsonl:v3.3-final seek-block --help
docker run pfc-jsonl:v3.3-final query --help
```

---

*PFC-Log v3.2 | ImpossibleForge | impossibleforge@gmail.com*
