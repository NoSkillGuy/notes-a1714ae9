# LanceDB memory growth

Research notes on a long-running process whose resident memory climbs for days and does not come back down. The full write-up is in [research/report.md](research/report.md). Source notes are in [research/notes](research/notes).

## Production setup

- Kubernetes pod: **1.5 GB request**, **8 GB limit**.
- LanceDB table stored on a **network volume or S3**.
- **Cache-aside:** each request loads the table from S3 onto the network drive, then opens it.
- The process stays up. Memory is not released between requests.

## Observed symptom

- RSS starts near **500 MB**.
- It climbs to about **7–7.5 GB** over **10–15 days**.
- It does not release that memory. The chart shows only minor dips.
- No GitHub issue describes this exact window or this pod.

## Findings

Lance keeps two caches on the **connection `Session`**, not on the table object. Current defaults:

| Cache | Default budget |
| --- | --- |
| Index cache (`index_cache_size_bytes`) | **6 GiB** |
| Metadata cache (`metadata_cache_size_bytes`) | **1 GiB** |

Together that is about **7 GiB**, which is **7.5 GB** in decimal units: the same band as the pod. Opening the table per request still fills the connection session. `close()` on the table leaves that session in place. There is no separate data-page cache. Scan buffering is per scan, and the default `io_buffer_size` is **2 GB**.

Those budgets are weighted caps, not a hard RSS limit. They explain the climb only when `session.size_bytes()` moves with RSS.

### Fixed bugs

Two defects let cache memory ignore its cap. Both are merged.

- **Index-cache reference cycle** when many datasets are opened. Dropping the dataset did not free the cache. Weak references fixed it. [lance#4680](https://github.com/lance-format/lance/pull/4680), merged **2025-09-10**.
- **Metadata weigher** under-counted the largest file-metadata fields by about **100×**, so a nominal 1 GiB cap never evicted. A short random-read test gained **7,503 MB** before the fix and plateaued near **500 MB** after it. [commit 8ac0741](https://github.com/lance-format/lance/commit/8ac0741302052ca51e0ba01827d89744f2a62fe2), **2026-04-11**, included in [lance#6480](https://github.com/lance-format/lance/pull/6480).

These commits were not mapped to release tags here. A version number alone does not show that either fix is present.

### Open reports where RSS climbs while the cache stays small

Cache settings do not release this. Maintainers have treated a high-water mark as allocator retention: glibc keeps freed pages because a later query is likely to need them again.

- [lance#6219](https://github.com/lance-format/lance/issues/6219) — open. Concurrent vector search on Azure. RSS about 0.03 GB to 1.04 GB in minutes while the session cache stayed near 100–140 MB inside a 200 MB cap. `malloc_trim` returned about 50–70 MB, then RSS resumed climbing. Not reproduced on AWS.
- [lance#7242](https://github.com/lance-format/lance/issues/7242) — open. Repeated `merge_insert` on pylance 7.0.0, about 5–7 MB RSS per flush. Live heap stayed near 30 MB. `MALLOC_ARENA_MAX=2` held RSS flat in the reporter’s run. A mimalloc change ([lance#7245](https://github.com/lance-format/lance/pull/7245)) was not merged.
- [lancedb#1592](https://github.com/lancedb/lancedb/issues/1592) — open since 2024. One search: about 1.56 GB of system bytes versus 29 MB in use. `malloc_trim` recovered about half. jemalloc via `LD_PRELOAD` ended near 250 MB. Weston Pace called this not a Lance leak.

### Related issues

- [lance#7508](https://github.com/lance-format/lance/issues/7508) — long-lived S3 session, versioned cache keys, reported OOM. Closed without a confirmed cause.
- [lance#7904](https://github.com/lance-format/lance/issues/7904) — cached index readers kept a stale S3 credential for at least 12 hours. Recovery was a process restart.
- [lance#7718](https://github.com/lance-format/lance/pull/7718) — unmerged. Would evict version-scoped cache entries when cleanup deletes their files.
- [lancedb#2468](https://github.com/lancedb/lancedb/issues/2468) — one-shot S3 search peak above 16 GB, fixed by sharing one scan across a query batch in [lancedb#3805](https://github.com/lancedb/lancedb/pull/3805). Different shape from a two-week ramp.
- [lancedb#1336](https://github.com/lancedb/lancedb/issues/1336) — a fresh cache per table instance was called out as a memory problem. The recommended shape is one shared session.
- [lancedb#3106](https://github.com/lancedb/lancedb/issues/3106) — the Python `Session` constructor hardcodes the object-store registry.

## Recommended fixes

Read `session.size_bytes()` against RSS before changing configuration.

**Path 1. Session size tracks RSS.** Keep one long-lived session for the life of the pod and set a byte budget well under 8 GB. A smaller index budget means more index-page reads from the network volume or S3. `LANCE_INDEX_CACHE_SIZE` and `LANCE_METADATA_CACHE_SIZE` are Spark connector settings. They do not change this embedded session.

```python
session = lancedb.Session(
    index_cache_size_bytes=1024 * 1024 * 1024,       # 1 GiB; default is 6 GiB
    metadata_cache_size_bytes=256 * 1024 * 1024,     # 256 MiB; default is 1 GiB
)
db = lancedb.connect(uri, session=session)
```

`0` disables that tier. Confirm `set_index_cache_capacity` and `set_metadata_cache_capacity` on the installed package before shrinking a live session. The performance guide documents them. The docs.rs session source fetched for this research did not include those setter names.

**Path 2. Cache stays small while RSS still climbs.** This matches the open allocator tickets. The setting that held RSS flat in the merge-insert report is glibc, not a Lance API:

```bash
MALLOC_ARENA_MAX=2
```

`malloc_trim(0)` returned only part of the heap in the open tickets, and the floor kept rising. jemalloc via `LD_PRELOAD` is what landed near 250 MB on [lancedb#1592](https://github.com/lancedb/lancedb/issues/1592).

### Scan buffer

Default `io_buffer_size` is 2 GB, already larger than the 1.5 GB request. It is a per-scan peak, not a two-week floor. For a bandwidth-constrained remote store, the Lance guide’s starting point is:

- `LANCE_DEFAULT_IO_BUFFER_SIZE` = 64 MiB (`64 * 1024 * 1024`)
- `LANCE_IO_THREADS=8` (local/network-volume default is already 8; an `s3://` URI defaults to 64)
- `fragment_readahead=1`
- `batch_readahead=2`

Set `LANCE_IO_THREADS` and `LANCE_CPU_THREADS` before Lance is imported. Thread pools are created at startup.

`LANCE_READ_CACHE_REPETITION_INDEX=false` stops repetition-index entries in the metadata cache. That cache is on by default.

## What to check on the running binary

Upgrade to a Lance build that contains the **2025-09-10** weak-reference fix and the **2026-04-11** weigher fix. This research did not map those commits to release tags.

No maintainer has said a healthy cache requires a process restart. Restart is what cleared the credential-stuck index reader in [lance#7904](https://github.com/lance-format/lance/issues/7904).
