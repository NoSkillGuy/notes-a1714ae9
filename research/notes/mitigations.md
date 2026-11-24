# Releasing and bounding LanceDB and Lance memory

Scope: documented or maintainer-stated knobs and operational workarounds that release or cap memory from LanceDB caches, table handles, and object-store clients in a long-running process. User context (not a finding): one LanceDB table on a network volume or S3, opened per request in a pod with a 1.5 GB request and an 8 GB limit; RSS starts near 500 MB and reaches 7–7.5 GB over 10–15 days.

No user-facing type named `LanceCache` turned up in the Lance or LanceDB docs searched. The named caches are the session **index cache** and **metadata cache**. There is no separate page-cache byte budget. Encoded pages are limited by the scan I/O buffer, and index pages live inside the index cache.

## Which settings bound or evict the index cache, metadata cache, page cache, or LanceCache?

### Takeaway

The supported cap is a per-session byte budget: index cache default **6 GiB**, metadata cache default **1 GiB**. Set it on a `Session` or on dataset open, or shrink it later so entries are evicted. Zero disables a tier. That budget is explicitly not a process RSS limit. Spark-only env vars exist; they were not found as pylance settings.

### Cited Findings

Index and metadata cache (bytes, current API):

- Rust `ReadParams.index_cache_size_bytes` is “Size of the index cache in bytes… The default is 6 GiB.” `ReadParams.metadata_cache_size_bytes` is “Size of the metadata cache in bytes… The default is 1 GiB.” — [ReadParams](https://docs.rs/lance/latest/lance/dataset/struct.ReadParams.html)
- Source constants: `DEFAULT_INDEX_CACHE_SIZE = 6 * 1024 * 1024 * 1024` and `DEFAULT_METADATA_CACHE_SIZE = 1024 * 1024 * 1024`, with the comment that column metadata can be about 40 MB so 1 GiB covers a few hundred columns. — [dataset.rs](https://docs.rs/lance/latest/src/lance/dataset.rs.html)
- `DatasetBuilder::with_index_cache_size_bytes` and `with_metadata_cache_size_bytes` set those byte sizes. Field comments: “If it is zero, index cache is disabled” and the metadata field is disabled at zero. `with_metadata_cache_size_bytes` docs: default 1 GiB. Deprecated `with_metadata_cache_size` says “Set to zero to disable this cache.” — [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- If a session is passed, `DatasetBuilder::with_session` ignores `with_index_cache_size` and `with_metadata_cache_size`. `load` uses the provided session instead of constructing one from the builder sizes. A custom `with_index_cache_backend` overrides `with_index_cache_size_bytes`; the backend owns capacity. — [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Performance guide: both caches are LRU and sized in bytes (metadata default 1 GiB, index default 6 GiB via `index_cache_size_bytes` on `LanceDataset`). `session.set_index_cache_capacity(...)` and `session.set_metadata_cache_capacity(...)` change capacity after construction. “Shrinking a capacity evicts entries as needed” without dropping the session. A zero-capacity cache reports `enabled=False`. — [Performance Guide](https://lance.org/guide/performance/)
- The same guide says `size_bytes`, `capacity_bytes`, write bytes, and removal bytes are cache-policy weights. They “are different from the deep-size estimate returned by `Session.size_bytes()` and from process RSS.” — [Performance Guide](https://lance.org/guide/performance/)
- docs.rs `Session::new`: `index_cache_size` is “the weighted-entry budget of the index cache, backed by a single-shard `QuickCacheBackend`. The budget is shared by all indices in the session and is not a process RSS limit.” `metadata_cache_size` is the metadata cache size on `QuickCacheBackend`. — [Session in lancedb](https://docs.rs/lancedb/latest/lancedb/struct.Session.html)
- Python public session: `lancedb.Session(index_cache_size_bytes: Optional[int] = None, metadata_cache_size_bytes: Optional[int] = None)`. `Session.default()` “is equivalent to creating a session with 6GB index cache and 1GB metadata cache.” Pass it as `lancedb.connect(..., session=...)`. The connect docstring says sessions “can significantly impact memory use and performance” and can be reused across connections to share cache state. Embedded only: `db://` connections reject `session` with `NotImplementedError` and tell you to omit it “to use server-managed storage and caching.” — [Python API](https://lancedb.github.io/lancedb/python/python/); [\_lancedb.pyi](https://github.com/lancedb/lancedb/blob/8ae4f42f/python/python/lancedb/_lancedb.pyi)
- pylance `LanceDataset` / `lance.dataset` kwargs include `index_cache_size_bytes` and `metadata_cache_size_bytes`, plus `session`. — [dataset.py](https://github.com/lance-format/lance/blob/main/python/python/lance/dataset.py)
- Node: `new Session(indexCacheSizeBytes?, metadataCacheSizeBytes?)`. Docs name the parameters `index_cache_size_bytes` and `metadata_cache_size_bytes`, both bytes, defaults 6GB and 1GB. — [JS Session](https://lancedb.github.io/lancedb/js/classes/Session/)
- Maintainer PR (wjones127, merged 2025-07-24): Python and TypeScript `Session` exposes `index_cache_size_bytes` and `metadata_cache_size_bytes`. “The `Session` is attached to the `Connection`, and thus shared across all tables in that connection.” Table-level cache configuration is deprecated. — [lancedb#2530](https://github.com/lancedb/lancedb/pull/2530)

Deprecated entry-count API (still seen in docs; converts to bytes):

- `index_cache_size` (entries) deprecated since 0.30.0. Use `index_cache_size_bytes`. — [Performance Guide](https://lance.org/guide/performance/)
- Rust `ReadParams::index_cache_size` and `DatasetBuilder::with_index_cache_size`: “Set to zero, to disable the cache.” Conversion is `cache_size * 20 * 1024 * 1024` (20 MiB per entry). — [dataset.rs](https://docs.rs/lance/latest/src/lance/dataset.rs.html); [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Deprecated `with_metadata_cache_size` converts with `10 * 1024 * 1024` (10 MB per entry). — [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Python `open_table(..., index_cache_size=)` is deprecated. Docstring still says “default 256” entries. IVF: one entry per partition. BTREE: one entry for the whole index. The cache “applies to the entire opened table, across all indices.” pylance warns that the old parameter “will be converted to bytes using 20 MiB per entry.” `LanceDataset.create_index(..., index_cache_size=)` is ignored: “index building does not use the index cache, and the cache is sized on the dataset or session.” — [db.py](https://github.com/lancedb/lancedb/blob/8ae4f42f/python/python/lancedb/db.py); [dataset.py](https://github.com/lance-format/lance/blob/main/python/python/lance/dataset.py)
- Older pylance docs described `index_cache_size` as a count of index pages (IVF partitions), default 256. — [pylance dataset docs](https://lance-format.github.io/lance-python-doc/dataset.html)

What the two caches hold (so there is no third “page cache” budget):

- Metadata cache keys include manifests, transactions, deletion files, row-id mask/index/sequence, index metadata, old index details, file global metadata, and file column metadata (“the search cache for a column”). Keys are scoped to the dataset URI. — [Performance Guide](https://lance.org/guide/performance/)
- Index cache holds opened vector and scalar indices. BTree search “is loaded into the index cache in pages. Each page contains 4096 values.” Those pages are index-cache entries, not a separate cache. — [Performance Guide](https://lance.org/guide/performance/)
- JS `prewarmIndex`: loads an index into memory; “If the index does not fit in the cache then this call may be wasteful.” — [JS Table](https://lancedb.github.io/lancedb/js/classes/Table/)

Page / scan buffer (the closest thing to a page cache):

- `io_buffer_size` is “the amount of RAM that will be reserved for holding I/O received from storage before it is processed.” If unset, “v2 scans choose a default based on the object store and `LANCE_DEFAULT_IO_BUFFER_SIZE` can override that default.” “This value is not a hard cap.” — [scanner.rs](https://docs.rs/lance/latest/src/lance/dataset/scanner.rs.html)
- Performance guide: default `io_buffer_size` is 2GB, “enough to buffer 64 pages.” Pages are “typically between 8 and 32 MB.” Rule of thumb: about 32 MB per I/O thread. Scanner kwargs shown: `fragment_readahead`, `batch_readahead`, `io_buffer_size` (example `64 * 1024 * 1024`). Formula: scans “could use up to `(2 * io_buffer_size) + (batch_size * num_compute_threads)` bytes.” `io_buffer_size` is a soft limit because Lance “cannot read less than one page at a time.” Default `batch_size` is 8192 rows. — [Performance Guide](https://lance.org/guide/performance/)
- Index-build input scan “uses a 2 GiB I/O readahead buffer by default (configurable via `LANCE_DEFAULT_IO_BUFFER_SIZE`).” — [Performance Guide](https://lance.org/guide/performance/)
- Observability: each scheduler bounds its share by `io_buffer_size` for a scan, and “32 MiB per I/O thread for most other reads, such as takes and index loads.” Priority reads can pass that bound. `datafusion_pool` is a separate reservation in DataFusion memory pools Lance builds for scans and index builds. No user-facing pool-size setter was on that page. — [Observability](https://lance.org/guide/observability/)

Repetition-index cache (metadata cache contents, not its own byte cap):

- `read_params={"cache_repetition_index": ...}` and env `LANCE_READ_CACHE_REPETITION_INDEX=false`. pylance docs: caching repetition indices for large string/binary columns “is enabled by default.” — [pylance dataset docs](https://lance-format.github.io/lance-python-doc/dataset.html)
- Maintainer Xuanwo (PR, merged 2026-02-27): repetition-index cache “grows linearly; for 200k rows it occupies about 1.6 MiB” and “will be managed by our global metadata cache.” Weston Pace asked to keep a way to disable it for billion-row string columns; Xuanwo agreed to leave the option. — [lance#5981](https://github.com/lance-format/lance/pull/5981)

Spark-only cache env vars (do not treat as pylance/LanceDB embedded settings):

- `LANCE_INDEX_CACHE_SIZE` default “6GB”, `LANCE_METADATA_CACHE_SIZE` default “1GB”, both “in bytes.” Spark docs say the 6GB/1GB defaults are “from Lance native.” Per-catalog `index_cache_backend` / `metadata_cache_backend` (example `moka://?capacity=6442450944`) override those size env vars. Env fallbacks: `LANCE_INDEX_CACHE_BACKEND`, `LANCE_METADATA_CACHE_BACKEND`. — [Spark config](https://lance.org/integrations/spark/config/); [Spark performance](https://lance.org/integrations/spark/performance/)
- `LANCE_ALLOCATOR_SIZE` default `Long.MAX_VALUE`, “Arrow allocator size in bytes (global),” Spark connector only. Example in the doc is `4294967296` (4GB). — [Spark config](https://lance.org/integrations/spark/config/)
- A search for `LANCE_INDEX_CACHE_SIZE` on `github.com/lance-format/lance` did not return a core reader of that variable. The spark connector commit describes the per-catalog Rust `Session` as “configured via `LANCE_INDEX_CACHE_SIZE` / `LANCE_METADATA_CACHE_SIZE`.” — [lance-spark commit be64654](https://github.com/lance-format/lance-spark/commit/be64654d0349f3ca247ac1c0a34f454b1496539f)

Anecdotal (not a documented setting):

- Issue author capped caches at `int(1e8)` bytes (100 MB) each and set `LANCE_IO_THREADS`, `LANCE_PROCESS_IO_THREADS_LIMIT`, and `LANCE_CPU_THREADS` to `"2"` before import. Logged `session.size_bytes` stayed near 100–147 MB (`status=OK` against a 200 MB budget) while RSS still rose from about 0.29 GB to about 1.11 GB over ~7 minutes. — [lance#6219](https://github.com/lance-format/lance/issues/6219)
- Same issue: the author called libc `malloc_trim(0)` every 30 seconds. Each call freed roughly 50–85 MB in the pasted log, and RSS kept ratcheting up between trims. This is not a Lance API. — [lance#6219](https://github.com/lance-format/lance/issues/6219)

### Inferences

- One long-lived session left at the defaults can account for about 7 GiB of cache budget (6 + 1). That lines up with an 8 GB pod limit and a 7–7.5 GB RSS plateau, but the docs also say accounted cache size is not RSS, so a plateau at that level is consistent with the defaults and not proof that the caches are what filled the pod.
- `index_cache_size=256` on the deprecated entry API is not 256 bytes. At the documented 20 MiB-per-entry conversion it is 5 GiB, close to the 6 GiB byte default, only if the caller actually passes the entry count. Omitting it uses the 6 GiB byte default.
- Setting a small `index_cache_size_bytes` bounds the cache’s weighted budget. Issue 6219 is the maintainer-visible evidence that RSS can still climb while that budget stays inside its cap.

### Gaps

- The performance guide names `set_index_cache_capacity` and `set_metadata_cache_capacity` but does not publish a parameter name, unit, or Python signature in the text that was fetched. Units are only implied by the surrounding byte-capacity section.
- No documented byte size for a standalone page cache or a type called `LanceCache`.
- Not confirmed from core source that pylance or LanceDB reads `LANCE_INDEX_CACHE_SIZE` / `LANCE_METADATA_CACHE_SIZE`. Those names are documented on the Spark connector.
- No documented DataFusion or PyArrow memory-pool size (`ARROW_*`, `PYARROW_*`) that Lance tells users to set. The DataFusion pool is described as an internal reservation.

## Does closing a table, connection, or database release those caches?

### Takeaway

Docs say `close()` on a table frees that table’s cached index data, and that garbage collection does the same if you skip `close()`. They do not say you must exit the process. A shared `Session` outlives any one table. A fixed bug (circular `Arc` in the index cache) used to keep caches alive after the dataset was dropped. Embedded `close()` is documented as releasing connection resources, with an explicit note that embedded connections do not currently hold open resources.

### Cited Findings

- JS `Table`: “Table objects will cache a certain amount of index data in memory. This cache will be freed when the Table is garbage collected. To eagerly free the cache you can call the `close` method.” “Closing a table is optional.” `close()` “releas[es] any underlying resources” and is safe to call more than once. — [JS Table](https://lancedb.github.io/lancedb/js/classes/Table/); [table.ts](https://github.com/lancedb/lancedb/blob/main/nodejs/lancedb/table.ts)
- Python `AsyncTable` uses the same words: cache freed on GC, or eagerly via `close()`. Context manager calls `close()`. `close()` “free[s] any resources associated with it.” — [table.py](https://github.com/lancedb/lancedb/blob/8ae4f42f/python/python/lancedb/table.py); [Python API](https://lancedb.github.io/lancedb/python/python/)
- Python sync `DBConnection.close`: “Close the connection, releasing any underlying resources.” Safe to call twice. A closed connection raises `RuntimeError: Connection is closed`. Context manager closes on exit. — [Python API](https://lancedb.github.io/lancedb/python/python/)
- Python `AsyncConnection`: “Embedded connections do not currently hold any open resources but they may do so in the future (for example, for shared cache or connections to catalog services).” Enterprise connections are the ones described as an open server connection. “The close method can be used to release any underlying resources eagerly.” “Closing a connection is optional” and it is closed when the object is deleted. “in many cases a single connection can be used for the lifetime of the application.” — [Python API](https://lancedb.github.io/lancedb/python/python/)
- `in-memory` URI only: “Tables are dropped when the last connection or table handle referencing them is closed.” Maintainer wjones127: “as soon as you close the table handle, the table is deleted from memory.” That is the `memory://` database, not the index cache of a persisted table. — [Python API](https://lancedb.github.io/lancedb/python/python/); [lancedb#2247](https://github.com/lancedb/lancedb/issues/2247)
- Session sharing: the session is on the connection and shared by its tables (wjones127). A dataset opened with no session builds a new `Session` from the cache sizes. Dropping that dataset is what drops that session, unless another handle holds it. — [lancedb#2530](https://github.com/lancedb/lancedb/pull/2530); [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Maintainer fix, wkalt, merged 2025-09-10: loading an index stored a strong reference from the index back to the index cache, “a circular reference… This prevented the cache from getting deallocated when the dataset was dropped, resulting in a leak if many datasets were opened over time.” Fixed by weak references. — [lance#4680](https://github.com/lancedb/lance/pull/4680)
- Maintainer wjones127 on writes (closed 2024-12-05): `Table.add` reset `index_cache_size` to the default and “creates a new session instance.” He called that “serious memory usage footguns.” Fix was to reuse the table instance during write. — [lancedb#1655](https://github.com/lancedb/lancedb/issues/1655)
- Shrinking capacity evicts without dropping the session. Disabling is the zero-size setting above, not a separate cleanup call. — [Performance Guide](https://lance.org/guide/performance/); [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Object-store clients: storage option `object_store_cache_prune_threshold` is the “smallest number of cached object stores at which a session's object store cache sweeps out entries for stores that are no longer in use.” Default 64. `0` disables the sweep. Shared by every dataset on the same session; “the last value seen applies to all of them.” It is a count of stores, not a byte cap. Relevant when one session opens many tables with distinct storage options. — [Object store configuration](https://lance.org/guide/object_store/)

No maintainer comment in the issues read says the process must be restarted to release caches. The documented release paths are drop/GC, `close()`, a smaller capacity (evict), or a zero capacity (disable).

Anecdotal:

- Blog (not a maintainer): for `lancedb==0.21.2` the author closed the async table and connection per operation. Update dated 2025-09-09: “`lancedb==0.25.0` fixed the issues with `connect_async`, and the old context manager pattern now causes memory leaks. Use a singleton for connections and tables instead.” — [sprytnyk.dev](https://sprytnyk.dev/posts/running-lancedb-in-production/)
- lancedb#2512: user measured `Table.add` in a loop peaking at 11.6 GiB RSS, and 2.9 GiB if they `del` the batch and call `gc.collect()` each iteration. wjones127 did not recommend `gc.collect()`. He wrote he suspects “the preprocessing generators we create when calling `Table.add()`,” and asked whether it reproduces in Lance. Issue left open as a bug. — [lancedb#2512](https://github.com/lancedb/lancedb/issues/2512)
- lance#6219 remains open (assignee Xuanwo, label performance). On 2026-03-30 Xuanwo wrote: “I failed to reproduce this on AWS. The RSS is coming back after calling `trim`. Is this related to azure?” He did not prescribe a process restart or a close API. — [lance#6219](https://github.com/lance-format/lance/issues/6219)

### Inferences

- Closing one table does not clear a session that the connection, or another table, still holds. Per-request `open_table` without `close()` or without dropping the last reference keeps a session per open. Before the weak-reference fix, dropping the dataset was not enough.
- `AsyncConnection`’s “embedded connections do not currently hold any open resources” means `connection.close()` is not documented as the thing that frees the index and metadata caches. The table/dataset/session handles are.
- The sync Python `Table` wrapper in the fetched `table.py` snapshot does not define `close()`; `AsyncTable` and the Rust inner table do. Callers on the sync API may be relying on GC only. That split was not re-checked against a newer commit than `8ae4f42f` plus the current Python API HTML, where the documented table `close()` sits on `AsyncTable`.

### Gaps

- Docs do not say whether `DBConnection.close()` drops a `Session` that the caller still holds, or only resources the connection itself owns.
- No statement that closing a database object (as distinct from a connection or table) flushes the index cache.
- No maintainer answer on lance#6219 that names a cleanup call for the slow RSS climb once cache accounting is already inside its budget.

## What do they recommend for S3 or network-volume deployments and for per-request table opens?

### Takeaway

Maintainers tell you to open one table and one session and reuse them. Per-table sessions are described as a memory problem. For remote or bandwidth-limited stores they tell you to lower I/O concurrency and the scan buffer. A filesystem network volume is treated as a local store (fewer I/O threads, no cloud rate limiter). There is no Kubernetes-specific memory guide.

### Cited Findings

Reuse, do not open a fresh cache per call:

- “The metadata cache is not shared between tables by default. For best performance you should create a single table and share it across your application. Alternatively, you can create a single session and specify it when you open tables.” The index-cache section repeats: not shared between tables; share one table. — [Performance Guide](https://lance.org/guide/performance/)
- Maintainer westonpace (lancedb#1336): “Every `Table` has a cache for index & metadata. Right now that cache is newly created for each table instance. This can lead to memory issues.” He wanted multiple table instances to share one cache for concurrent queries. wjones127 replied that Rust `DatasetBuilder` already accepts a `Session` and that it needed to be exposed in Python. That exposure later shipped as `lancedb.Session`. — [lancedb#1336](https://github.com/lancedb/lancedb/issues/1336); [lancedb#2530](https://github.com/lancedb/lancedb/pull/2530)
- JS and Python async docs: a table “is expected to be long lived and reused for multiple operations.” — [JS Table](https://lancedb.github.io/lancedb/js/classes/Table/); [table.py](https://github.com/lancedb/lancedb/blob/8ae4f42f/python/python/lancedb/table.py)
- Async connections: “a single connection can be used for the lifetime of the application.” — [Python API](https://lancedb.github.io/lancedb/python/python/)

S3 / cloud vs a path on a network volume:

- I/O pool: “Local object stores will use 8 threads by default. Cloud object stores will use 64 threads by default.” Override with `LANCE_IO_THREADS`. Raising it means you may also need a larger `io_buffer_size`. Compute pool defaults to the machine’s core count, override `LANCE_CPU_THREADS`. — [Performance Guide](https://lance.org/guide/performance/)
- “Cloud stores default to 64, which is intended for high-bandwidth, in-region access and can be too aggressive across regions or over the public internet.” Suggested starting point for a bandwidth-constrained remote connection: `LANCE_IO_THREADS=8`, `fragment_readahead=1`, `batch_readahead=2`, `io_buffer_size=64 * 1024 * 1024`. `fragment_readahead` limits how many fragments have reads scheduled at once. `batch_readahead` limits concurrent batch decoding, not the storage range size. — [Performance Guide](https://lance.org/guide/performance/)
- Cloud stores (S3, GCS, Azure) get an AIMD rate limiter. “Local and in-memory stores are not throttled.” AIMD “replaces the old `LANCE_PROCESS_IO_THREADS_LIMIT` process-wide cap.” Same-bucket clients share the budget in-process. Defaults include `LANCE_AIMD_MAX_RATE` 10000 requests/second (11000 for native S3 reads), `LANCE_AIMD_INITIAL_RATE` 2000, `LANCE_AIMD_BURST_CAPACITY` 100. Setting max rate to `0` removes the client-side ceiling and leaves AIMD feedback on. These are request rates, not byte caps. — [Performance Guide](https://lance.org/guide/performance/)
- `block_size` / `DatasetBuilder::with_block_size`: “Defaults to 4KB for local files and 64KB for others.” Larger `block_size` merges nearby reads “at the cost of reading the bytes in between.” — [Object store configuration](https://lance.org/guide/object_store/); [builder.rs](https://docs.rs/lance/latest/src/lance/dataset/builder.rs.html)
- Historical scheduler source (not the current performance guide): `LANCE_PROCESS_IO_THREADS_LIMIT` defaulted to 128 process-wide scan IOPS; `<= 0` disabled it. Comment: the cap “does not necessarily limit the number of open TCP connections,” because “the underlying object store may open more connections anyways.” — [scheduler.rs at 119f87b3](https://github.com/lancedb/lance/blob/119f87b3/rust/lance-io/src/scheduler.rs)
- Object-store registry prune threshold (default 64 stores) is the documented control when one session opens many tables with different storage options or per-table credentials. — [Object store configuration](https://lance.org/guide/object_store/)
- Maintainer BubbleCal, on an S3 IVF_HNSW_SQ workload that was pulling multi-megabyte `auxiliary.idx` ranges: “lance uses a fixed size cache for indices, so yes, increasing the cache would help.” That is a performance recommendation, the opposite of a memory cap. — [lance#3722](https://github.com/lance-format/lance/issues/3722)

Anecdotal:

- The production blog’s later advice is a process-wide singleton connection and table, after the author decided per-request open/close leaked on `lancedb==0.25.0`. Not a maintainer source, and it disagrees with the older “close on the way out” pattern the same post first published. — [sprytnyk.dev](https://sprytnyk.dev/posts/running-lancedb-in-production/)

### Inferences

- A pod that `open_table`s on every request and does not share a `Session` creates a new 6 GiB + 1 GiB budget each time. Reuse is the documented way to keep that to one budget. If requests must open a handle, the documented shape is one long-lived session with an explicit byte cap, then drop or `close()` the per-request handle (and be on a build that includes the weak-reference cache fix).
- A network volume mounted as a normal filesystem path follows the local-store rules (8 I/O threads, 4 KB block size, no AIMD). An `s3://` URI follows the cloud rules (64 I/O threads, 64 KB block size, AIMD). Nothing in the docs adds an NFS- or EFS-specific cache.
- The remote-scan paragraph is the closest maintainer recipe for a small pod in front of S3: cut `LANCE_IO_THREADS` and `io_buffer_size` before raising them. The default 2 GB `io_buffer_size` is already larger than this pod’s 1.5 GB request, and the documented scan formula counts it twice.

### Gaps

- No maintainer doc or issue comment was found that mentions Kubernetes, a 1.5 GB request, an 8 GB limit, or a 10–15 day RSS climb on a network volume.
- No documented recommendation that is specific to NFS, EBS multi-attach, or “network volume” as a storage class. Local-vs-cloud above is the stand-in.
- Not verified whether current `main` still reads `LANCE_PROCESS_IO_THREADS_LIMIT` after the performance guide said AIMD replaced it.

## Are there env vars such as LANCE_*, object-store cache sizes, or arrow/memory pool settings that people successfully used?

### Takeaway

Env vars that are documented for the embedded library bound threads and scan buffers, not the index cache. The index-cache env vars are Spark’s. The one public long-running-search report that set cache byte sizes and `LANCE_*` thread caps kept cache accounting inside the cap and still saw RSS rise. No Arrow pool env var is documented as a successful cap.

### Cited Findings

Documented embedded env vars that affect memory or how much memory I/O can pin:

| Name | What it does | Default | Source |
| --- | --- | --- | --- |
| `LANCE_IO_THREADS` | I/O thread-pool size. Local stores otherwise use 8; cloud stores 64. | unset means those store defaults | [Performance Guide](https://lance.org/guide/performance/) |
| `LANCE_CPU_THREADS` | Compute pool size. | core count; index-build text also says `num_cpus - 2` batches in flight | [Performance Guide](https://lance.org/guide/performance/) |
| `LANCE_DEFAULT_IO_BUFFER_SIZE` | Overrides the v2 scan / index-build I/O buffer default. | guide: 2 GiB when unset | [scanner.rs](https://docs.rs/lance/latest/src/lance/dataset/scanner.rs.html); [Performance Guide](https://lance.org/guide/performance/) |
| `LANCE_READ_CACHE_REPETITION_INDEX` | `false` disables repetition-index caching. | enabled | [pylance dataset docs](https://lance-format.github.io/lance-python-doc/dataset.html) |
| `LANCE_PROCESS_IO_THREADS_LIMIT` | Old process-wide scan IOPS cap. Guide says AIMD replaced it. | 128 in the older scheduler; `<= 0` disabled | [Performance Guide](https://lance.org/guide/performance/); [scheduler.rs at 119f87b3](https://github.com/lancedb/lance/blob/119f87b3/rust/lance-io/src/scheduler.rs) |
| `LANCE_AIMD_MAX_RATE`, `LANCE_AIMD_INITIAL_RATE`, `LANCE_AIMD_MIN_RATE`, `LANCE_AIMD_BURST_CAPACITY`, `LANCE_AIMD_DECREASE_FACTOR`, `LANCE_AIMD_ADDITIVE_INCREMENT`, `LANCE_AIMD_MAX_RETRIES`, `LANCE_AIMD_MIN_BACKOFF_MS`, `LANCE_AIMD_MAX_BACKOFF_MS`, plus `LANCE_AIMD_{READ,WRITE,DELETE,LIST}_*` | Cloud request-rate limiter. Same names work as lowercase storage options (`lance_aimd_read_max_rate` and so on). | max 10000 rps (11000 native S3 reads); initial 2000; min 1; burst 100; decrease 0.5; increment 300; retries 3; backoff 100–300 ms | [Performance Guide](https://lance.org/guide/performance/) |
| `GLIBC_TUNABLES=glibc.malloc.hugetlb=1` | Opt-in huge pages for large decode allocations. Docs say it can raise RSS (2 MiB pages) and can stall on compaction. “check the resident set against your memory limits.” | off | [Performance Guide](https://lance.org/guide/performance/) |

Documented storage options that are not byte caches:

- `object_store_cache_prune_threshold`: store-count sweep, default 64, `0` disables. — [Object store configuration](https://lance.org/guide/object_store/)
- `timeout` (default `30s`), `connect_timeout` (default `5s`), `client_max_retries` (default 3), `client_retry_timeout` (default 180 seconds), `download_retry_count` (default 3). Timeouts and retries, not cache sizes. Pass as storage options or, for some keys, environment variables. The only env example on that page is `TIMEOUT=60s`. — [Object store configuration](https://lance.org/guide/object_store/)
- `hf_enable_resolve_cache`: `"true"` / `"false"`, default `"false"`. Caches resolved Hugging Face URLs and Xet metadata, “does not cache file contents.” — [Object store configuration](https://lance.org/guide/object_store/)

Spark-only, listed so they are not applied blindly to a Python pod:

- `LANCE_INDEX_CACHE_SIZE`, `LANCE_METADATA_CACHE_SIZE`, `LANCE_INDEX_CACHE_BACKEND`, `LANCE_METADATA_CACHE_BACKEND`, `LANCE_ALLOCATOR_SIZE`. — [Spark config](https://lance.org/integrations/spark/config/)

What was actually tried against a long-running search process:

- lance#6219 (open): cache budgets of 1e8 bytes and the three thread env vars at `"2"` kept the session size report inside ~110% of the budget. RSS was not held flat. Xuanwo’s only follow-up is that on AWS, RSS came back down after `malloc_trim`, and he asked if the remaining growth is Azure-specific. That is a maintainer observation that trim returned memory on AWS, not a documented Lance setting and not a claim that trim stops a multi-day climb. — [lance#6219](https://github.com/lance-format/lance/issues/6219)
- lancedb#2512: `gc.collect()` after each `add` cut that user’s peak RSS from 11.6 GiB to 2.9 GiB. Maintainer did not adopt it as the fix. — [lancedb#2512](https://github.com/lancedb/lancedb/issues/2512)

Diagnostic, not a cap: `dataset.session().size_bytes()`, `dataset.session().cache_diagnostics()` (`index` / `metadata`, including `backend.size_bytes` and `backend.capacity_bytes`), and `LANCE_MEMORY_OBSERVE` for Linux RSS/cgroup events during index builds. — [Performance Guide](https://lance.org/guide/performance/)

### Inferences

- For this pod, the documented embedded knobs that can actually move the ceiling are `Session(index_cache_size_bytes=..., metadata_cache_size_bytes=...)` (or the same kwargs on `lance.dataset`), optionally shrunk later, plus `io_buffer_size` and `LANCE_IO_THREADS` if the table is on S3 or another high-latency store. Thread env vars must be set before Lance’s pools start; the issue author set them before `import lancedb` / `import lance`.
- `GLIBC_TUNABLES=glibc.malloc.hugetlb=1` is a speed knob that the docs warn can increase resident memory. It is not a way to stay under 8 GB.
- A network-volume mount will not pick up AIMD or the cloud 64-thread default. `LANCE_IO_THREADS` still applies if set.

### Gaps

- No source was found where a maintainer says a particular `LANCE_*` value or Arrow pool setting held RSS flat for days.
- No documented object-store client byte cache (HTTP body cache, connection-pool byte cap) beyond `io_buffer_size`, the 32 MiB-per-I/O-thread note, and the store-count prune threshold.
- `LANCE_PROCESS_IO_THREADS_LIMIT`’s current status is inconsistent across a current guide (“replaced”) and an older source file (still read). Do not treat it as a live cap without checking the version in the pod.
