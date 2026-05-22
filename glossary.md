# Glossary — Lance-specific terms

Use this as a lookup while reading. Terms are grouped roughly by layer; alphabetical within each section.

## Format / on-disk concepts

**Base path** — Lance datasets can reference data files stored *outside* the dataset's own directory (e.g., a shared bucket of pre-computed embeddings). Each external location is a "base path" with a numeric `base_id`. See `BasePath` in `lance-table::format`.

**Detached version** — A dataset version not in the main version chain (used for some maintenance/migration flows). Marked by `DETACHED_VERSION_MASK`. See `is_detached_version` in `lance-table::format`.

**Fragment** — A horizontal partition of a dataset. Each fragment has a stable `id`, one or more `DataFile`s, and an optional deletion file. New fragments are created by writes; existing fragments are not modified in place.

**Fragment bitmap** — A `RoaringBitmap` of fragment ids. Indexes carry one to indicate which fragments they cover. The dataset itself carries one to indicate which fragments are alive in the current version.

**Manifest** — The complete description of the dataset at a specific version. Contains the schema, fragments, indices, timestamps, feature flags, and base paths. Stored as a small Lance file per version.

**Magic** — `"LANC"` (4 ASCII bytes) at the end of every Lance v2 file. Used to validate a file is a Lance file. See `lance-file::format::MAGIC`.

**Row address (`RowAddr`)** — A 64-bit value combining a 32-bit fragment id (upper bits) and a 32-bit row offset (lower bits). *Physical* — changes when fragments are compacted. See `lance-core::utils::address::RowAddress`.

**Row id (`RowId`)** — An optional, *stable* 64-bit identifier that survives compaction. Maintained via a separate row id index. Opt-in via `enable_stable_row_ids` in `WriteParams`.

**Tombstone** — A reserved sentinel address used to mark "this row pointer is invalid." Constants: `RowAddress::TOMBSTONE_FRAG = 0xffffffff`, `TOMBSTONE_ROW = 0xffffffffffffffff`.

**Transaction** — A description of a single change (Append, Delete, Overwrite, Update, MergeInsert, etc.). Stored in `_transactions/` for conflict resolution and audit. Distinct from the *resulting* manifest.

**Version (commit version)** — A monotonically increasing `u64`. Each commit produces a new version. Manifests live at `_versions/N.manifest` (V1 naming) or `_manifests/{u64::MAX-N}.manifest` (V2 naming, for object-store-friendly listing).

## Encoding / file format

**Definition level** — In a nested column, a small integer per value indicating how deep the value is "defined" (non-null) in the type tree. Pairs with the repetition level. See `lance-encoding::repdef`.

**Logical encoding** — An encoding tier that maps Arrow logical types (`List`, `Struct`, `Primitive`) to the encoding tree. One per Arrow field. Knows how to schedule reads of multiple physical columns.

**Miniblock layout** — One of two structural layouts. Splits a page into many small encoded blocks for *fast random access*. See `MiniBlockLayout` in `protos/encodings_v2_1.proto`.

**Page** — A self-contained encoded chunk for one column. Contains values, validity, repdef as needed. Pages have a maximum size (32 MiB at write time in 2.0; effectively unbounded in 2.1 since they can be split on read).

**Physical encoding** — A lower-tier encoding that maps a single buffer of values to bytes (e.g., bit-packing, RLE, FSST, dictionary, value). One column may have a tree of physical encodings under one logical encoding.

**RepDef** — Shorthand for "repetition + definition levels." Lance's compact encoding of nested-array offsets and validity. See `lance-encoding/src/repdef.rs`.

**Repetition level** — In a nested column, a small integer per value indicating where in the structure a new list begins. Pairs with the definition level.

**Structural encoding** — The 2.x model where the encoding tree mirrors the Arrow type tree. Distinguishes logical encodings (structure) from compressive encodings (values).

**Top-level row** — A row at the outermost level of the user schema. With repetition (`List`/`FixedSizeList`), there can be many "items" per top-level row. Used in `EncodedPage::row_number`.

**FSST** — Fast Static Symbol Table; a string compression codec. See `rust/compression/fsst/` and `lance-encoding/src/encodings/physical/fsst.rs`.

**Full-zip layout** — One of two structural layouts. Stores values, validity, and repdef interleaved. Better compression than miniblock but worse random access.

## Storage / I/O

**`EncodingsIo`** — Trait that abstracts I/O behind the encoder/decoder. Defined in `lance-encoding`. Implemented by Lance's scheduler.

**IOPS counter / bytes-read counter** — Global atomics in `lance-io::scheduler` that track how many I/O operations and bytes Lance has issued. Useful for benchmarking.

**Object store** — The unified storage abstraction (`object_store` crate, wrapped by `lance-io::object_store::ObjectStore`). Backends include local, memory, S3, Azure, GCS, OSS, HuggingFace.

**Priority** (in I/O requests) — A `u64` where **lower = more urgent** (typically derived from row number). The scheduler dispatches in priority order so that low-row-number reads are returned first.

**ScanScheduler** — The I/O coordinator. Coalesces nearby ranges, applies backpressure, dispatches in priority order. See `lance-io/src/scheduler.rs`.

## Dataset / table layer

**Branch** — A parallel commit history, like a Git branch. Each branch has its own version chain. See `lance/src/dataset/refs.rs`.

**Commit handler** — A pluggable strategy for atomically writing the next manifest. Implementations include `RenameCommitHandler` (default), `UnsafeCommitHandler` (legacy S3), and `ExternalManifestCommitHandler` (with an external coordinator like DynamoDB). See `lance-table/src/io/commit.rs`.

**DataFile** — One Lance v2 file inside a fragment. Holds a subset of fields. Multiple `DataFile`s per fragment enable cheap column addition.

**DatasetBuilder** — The configurable constructor for a `Dataset`. Pattern: `from_uri(...).with_*(...).load()`. Builders are everywhere in Lance APIs.

**FileFragment** — Runtime wrapper around a `lance-table::format::Fragment`, holding open `FileReader`s. Lives in `lance/src/dataset/fragment.rs`. Don't confuse with the on-disk `Fragment` proto/struct.

**Manifest cache / metadata cache** — Per-session bytes-bounded cache for parsed manifests, schemas, transactions. Defaults to ~1 GiB. Backed by `moka`.

**Index cache** — Per-session bytes-bounded cache for loaded index data. Defaults to ~6 GiB.

**Reader / Writer** (in `lance-io`) — Async traits for byte-range reads and sequential writes. Implemented by `CloudObjectReader`, `LocalObjectReader`, `ObjectWriter`, `LocalWriter`.

**Refs** — Branch and tag namespace, analogous to Git refs. Stored under `_refs/`.

**Session** — Process-wide shared state across datasets: caches, registries. See `lance/src/session.rs`.

**Tag** — A stable name for a specific dataset version (like a Git tag).

**Take** — Gather rows by row id or row offset. Distinct from a Scan (which reads contiguous rows). Used heavily in vector and full-text search.

**Updater** — Internal incremental writer used by schema evolution and other "rewrite each fragment" operations. See `lance/src/dataset/updater.rs`.

## Indices

**ANN** — Approximate Nearest Neighbor (vector search that trades some recall for huge speedups).

**BQ (Binary Quantization)** — 1 bit per dimension. Hamming distance. Used for cheap first-stage filtering.

**BTree (scalar)** — Lance's default general-purpose scalar index. Range and equality.

**Bitmap (scalar)** — One `RoaringBitmap` per distinct value. Best for low-cardinality categorical columns.

**BloomFilter** — Probabilistic membership test. May say "maybe contains" but never "doesn't contain" wrong. Used per-page.

**Centroid** — A k-means cluster center. IVF training produces K centroids; each row is assigned to its closest one.

**Distance type / Metric type** — `L2` (Euclidean), `Cosine`, `Dot` (inner product). Configurable per index. See `lance-linalg::distance::DistanceType`.

**FTS (Full-Text Search)** — Token-based text search. In Lance, the inverted index. Supports BM25 ranking, phrase, fuzzy, and boolean queries.

**Flat (vector)** — No quantization, store full vectors. Used standalone for tiny datasets, or inside an IVF partition to balance recall/space.

**HNSW (Hierarchical Navigable Small World)** — Multi-layer graph index. Excellent recall/latency curve. Always inside an IVF partition in Lance.

**Inverted index** — The full-text search index. One posting list per token. Lives in `lance-index/src/scalar/inverted/`.

**IVF (Inverted File)** — Partition-based vector indexing. Train K centroids, assign each row to its nearest, search by selecting the top `nprobes` partitions. Always the *outer* layer in Lance vector indices.

**LabelList (scalar)** — Index for `List<Scalar>` columns. Answers "which rows have value V in their list?"

**NGram (scalar)** — Substring search via n-grams (default trigrams). Different from FTS — catches non-word-aligned substrings.

**`nprobes`** — Number of IVF partitions to probe at query time. Higher = better recall, worse latency.

**Posting list** — In FTS, the list of documents (row ids) containing a token, with positions and frequencies for scoring.

**PQ (Product Quantization)** — Compress vectors by splitting into sub-vectors and quantizing each independently. Default `IVF_PQ` index.

**Prefilter** — Apply a non-vector filter *before* the vector search, ensuring top-k results all match the filter. Vs. *postfilter* which can return fewer than k.

**Refine factor** — Multiplier applied to `k` for the second-stage exact re-ranking. Vector search returns `k * refine_factor` candidates from the index, then re-ranks them with exact distances.

**RQ (Residual Quantization)** — Multi-stage quantization where each stage encodes the residual of the previous. Currently `IvfRq` at index version 2.

**RTree** — Spatial index for geometry columns. Gated behind the `geo` feature.

**SQ (Scalar Quantization)** — Quantize each dimension into 4 or 8 bits. Less aggressive than PQ; preserves dimension semantics.

**SARGable** — "Search ARGument-able." A predicate an index can directly answer (e.g., `x = 5`, `x BETWEEN 1 AND 10`). Non-SARGable predicates must be evaluated as filters on materialized rows.

**Subindex** — The per-IVF-partition inner index. Can be Flat, PQ, SQ, BQ, RQ, or HNSW.

**Target partition size** — Per index type, the heuristic row count per IVF partition. Used to derive K (number of partitions) from total row count. See `IndexType::target_partition_size`.

**WAND (Weak AND)** — A top-k retrieval algorithm for inverted indices. Skips documents that can't possibly enter the top-k. Implemented in `lance-index/src/scalar/inverted/wand.rs`.

**ZoneMap (scalar)** — Per-page (zone) min/max statistics. Lightweight; doesn't precisely answer queries but skips irrelevant pages.

## Operations

**Cleanup** — Garbage-collect old versions and unreferenced data files. See `Dataset::cleanup_old_versions`.

**Compaction** — Coalesce small fragments and rewrite to fewer, larger files. See `Dataset::optimize_compaction`.

**Mem-WAL** — Write-Ahead Log for atomic batched writes. See `lance/src/dataset/mem_wal/`.

**Merge insert (upsert)** — SQL `MERGE INTO` semantics: for each input row, update or insert based on a join key. See `MergeInsertBuilder`.

**Optimize** — Umbrella term for maintenance: compaction, index rebuilds, cleanup. See `lance/src/dataset/optimize.rs`.

**Schema evolution** — Add, alter, or drop columns without rewriting unaffected data. See `lance/src/dataset/schema_evolution.rs`.

**System columns** — Virtual columns produced at read time, not stored: `_rowid`, `_rowaddr`, `_rowoffset`, `_row_last_updated_at_version`, `_row_created_at_version`. See `lance-core::is_system_column`.

## Build / project conventions

**`AGENTS.md`** — Per-directory coding standards. Always check before contributing.

**`prost`** — Protocol Buffers Rust codegen. Build scripts (`build.rs`) generate Rust types from the `.proto` files in `protos/`.

**`Snafu`** — Error enum derive macro used by `lance-core::Error`. Adds location capture and context selectors.

**`Workspace`** — The Cargo workspace at the repo root. All `rust/lance-*` crates are members; `python/`, `java/lance-jni/`, and `memtest/` are excluded.

## Rust idioms (cross-reference)

**`Arc<T>`** — Atomically reference-counted shared pointer. Used everywhere a value is shared across threads/futures.

**`async-trait`** — Macro that allows `async fn` in trait definitions. Pre-Rust-1.75 workaround still used everywhere in Lance.

**`BoxFuture<'static, T>`** — A boxed, type-erased future that doesn't borrow anything. Common return type for async trait methods.

**`?`** — Error propagation operator. Returns the error from the current function if a `Result` is `Err`.

**`Result<T>`** — Lance-defined alias for `std::result::Result<T, lance_core::Error>`.

**`#[instrument]`** — Tracing macro that creates a span around a function. Used on hot paths for observability.
