# Exercises

Hands-on tasks per day. **Do them.** Reading without writing rusts fast.

Difficulty:

- **★** = quick (5–15 min) confidence-builder
- **★★** = guided (15–45 min) — most exercises are this
- **★★★** = open-ended (1+ hours) — for if you have extra time or want a deeper dive

If you get stuck on any exercise, ask me. The goal is forward motion, not pride.

---

## Day 1 — Rust + workspace tour

### 1.1 Build and test (★)

Run from the repo root:

```bash
cargo check --workspace --tests --benches
cargo test -p lance-core
cargo run -p lance-examples --example write_read_ds
```

Goal: confirm your environment works. If anything fails, fix the environment before continuing.

### 1.2 Read your first source file (★)

Read [`rust/lance/src/lib.rs`](../rust/lance/src/lib.rs) (~120 lines) end-to-end. After reading, answer (no peeking):

1. Which two Lance modules are publicly exposed at the top level?
2. What's the difference between `lance::open_dataset` and `lance::Dataset::open`? (You may have to peek for this one.)
3. Why is there a `pub mod deps`?

### 1.3 Make a small change and watch it propagate (★★)

Open [`rust/examples/src/write_read_ds.rs`](../rust/examples/src/write_read_ds.rs). The `read_dataset` function uses `println!`. Change it to use `tracing::info!` instead, including a `tracing_subscriber::fmt::init()` setup at the top of `main()`.

Then run:

```bash
RUST_LOG=info cargo run -p lance-examples --example write_read_ds
```

You should see structured log lines instead of plain prints.

**Revert your change before moving on** — `AGENTS.md` says not to merge debug prints, but more importantly, running tests later assumes the example is unchanged.

### 1.4 Trace `Result` through `?` (★★)

Use ripgrep:

```bash
rg "Error::invalid_input" rust/lance-core/src
```

Pick one call site. Open the file, look at the function it's in. Trace upward: what calls this function? What `Error` variant ends up returned? Repeat until you reach a public API.

Goal: feel how a low-level `Error::invalid_input(...)` becomes a top-level `Result::Err` returned from `Dataset::write`.

### 1.5 Self-assess (★)

Re-read [`day1.md`](day1.md) section 1.10 ("What I'm deliberately skipping") and the day-1 self-check questions. Anything you can't answer confidently → re-read the relevant section before bed.

---

## Day 2 — lance-core, lance-arrow, lance-io

### 2.1 Search for a concept (★)

Use `rg` to find every place that constructs an `Error::corrupt_file`:

```bash
rg "Error::corrupt_file" rust/
```

What kinds of conditions produce this error? Pick one and read the surrounding code (10 lines before/after).

### 2.2 Write a unit test (★★)

Add a test for `RowAddress` round-tripping. Create a new test in [`rust/lance-core/src/utils/address.rs`](../rust/lance-core/src/utils/address.rs) (at the bottom, in the existing `#[cfg(test)] mod tests` block, or create one if absent):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_roundtrip_extreme_values() {
        let cases = [(0, 0), (u32::MAX, u32::MAX), (1, 0), (0, 1), (1234, 5678)];
        for (frag, off) in cases {
            let addr = RowAddress::new_from_parts(frag, off);
            assert_eq!(addr.fragment_id(), frag);
            assert_eq!(addr.row_offset(), off);
            let raw: u64 = addr.into();
            let addr2: RowAddress = raw.into();
            assert_eq!(addr, addr2);
        }
    }
}
```

Run it:

```bash
cargo test -p lance-core test_roundtrip_extreme_values
```

Goal: practice the `cargo test -p <crate> <test_name>` workflow.

### 2.3 Explore the I/O scheduler (★★)

Open [`rust/lance-io/src/scheduler.rs`](../rust/lance-io/src/scheduler.rs). Find `bytes_read_counter()` and `iops_counter()` near the top.

Now write a small program that:

1. Reads `iops_counter()` and `bytes_read_counter()` before opening a dataset.
2. Calls `lance::open_dataset(...)`.
3. Reads the counters again and prints the deltas.

You can put this in `rust/examples/src/bin/iops_demo.rs` (one-off binary). Run it against `/tmp/scratch.lance` or anywhere you have a Lance dataset.

Goal: see the actual I/O cost of opening a dataset.

### 2.4 Explore deletion vectors (★★★)

Open [`rust/lance-core/src/utils/deletion.rs`](../rust/lance-core/src/utils/deletion.rs). Notice `BITMAP_THRESDHOLD: usize = 5_000` (typo in the source — `THRESDHOLD` not `THRESHOLD`).

Find the auto-promotion logic (search for "promote" or "Bitmap" in the file). Convince yourself that:

- A `DeletionVector::Set` promotes to `Bitmap` when it grows past 5000 entries.
- It never *demotes* back to `Set`.
- Why might the asymmetry be intentional?

Bonus: this typo is on `pub(crate)` constants — it's safe to fix. If you're feeling ambitious, fix the typo, run `cargo test -p lance-core`, and admire your first patch to Lance.

---

## Day 3 — encoding + file format

### 3.1 Write a tiny dataset (★)

Adapt [`rust/examples/src/write_read_ds.rs`](../rust/examples/src/write_read_ds.rs) to write to `/tmp/scratch.lance` with a single batch of one column. Run it.

Inspect the directory layout:

```bash
find /tmp/scratch.lance -type f | sort
```

Note the `_versions/`, `_transactions/`, `data/` directories.

### 3.2 Look at a real file footer (★★)

```bash
DATA_FILE=$(find /tmp/scratch.lance/data -name '*.lance' | head -1)
xxd "$DATA_FILE" | tail -3
```

Verify the last 4 bytes are `4c 41 4e 43` (`LANC`).

Decode the 8 bytes before the magic as a little-endian `i64` — that's the metadata offset. Verify by computing `file_size - 16 - metadata_offset` and confirming it's positive.

### 3.3 Read the encodings proto (★★)

Read [`protos/encodings_v2_1.proto`](../protos/encodings_v2_1.proto) end-to-end. After reading, answer:

1. What's a "value" in the standardized counting terms? What's a "level"?
2. What is the difference between `MiniBlockLayout` and `FullZipLayout`?
3. Where do compressive encodings (Bitpacked, FSST, Block, etc.) plug into structural encodings?

### 3.4 Examine a physical encoding (★★)

Open [`rust/lance-encoding/src/encodings/physical/value.rs`](../rust/lance-encoding/src/encodings/physical/value.rs) (the simplest physical encoding).

Find the encoder and decoder. Trace how an array of `u32`s would flow through. (Hint: `value.rs` mostly delegates — look for the actual byte-writing code.)

### 3.5 Build the docs (★)

```bash
cargo doc -p lance-encoding --no-deps --open
```

Browse the rendered documentation. This is the same content that ships on docs.rs but with your local overrides.

### 3.6 Implement a one-liner physical encoding (★★★)

This is hard but illuminating. Pick the very simplest possible encoding — say, "store each `u8` value verbatim, no compression." Look at how `value.rs` is structured. Add a similar `MyEncoding` in a scratch module. Then add it to a test that writes and reads a column with it.

This is too involved to walk through here, but if you want to do it, ask me — I'll guide you.

---

## Day 4 — lance-table + Dataset

### 4.1 Inspect a manifest (★★)

Write a small program that opens a dataset and prints its `Manifest` as JSON:

```rust
// rust/examples/src/bin/dump_manifest.rs
use lance::Dataset;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let path = std::env::args().nth(1).expect("usage: dump_manifest <uri>");
    let ds = Dataset::open(&path).await?;
    println!("{}", serde_json::to_string_pretty(&*ds.manifest)?);
    Ok(())
}
```

Add a `[[bin]]` entry in `rust/examples/Cargo.toml` referencing it. Run:

```bash
cargo run -p lance-examples --bin dump_manifest -- /tmp/scratch.lance
```

Read the output. Locate the `version`, `fragments`, and `schema` fields.

### 4.2 Trace a write (★★)

Add `tracing::info!` calls (with appropriate `RUST_LOG=info` env var) at:

1. The top of `Dataset::write` in `rust/lance/src/dataset.rs`.
2. The top of `write_fragments_internal` in `rust/lance/src/dataset/write.rs`.
3. The top of `commit_transaction` in `rust/lance/src/io/commit.rs`.

Run the example again. Watch the log lines. Do they appear in the order you expected? Where does each one trigger?

**Revert** before moving on.

### 4.3 Append to an existing dataset (★)

Modify `write_read_ds.rs` to:

1. Write a first batch.
2. Open the dataset, then `Dataset::write` a second batch with `WriteMode::Append`.
3. Open the resulting dataset and count rows.

You should see twice the rows, and `find /tmp/scratch.lance -type f | sort` should show two manifests in `_versions/`.

### 4.4 Check out an old version (★★)

After (4.3), use `Dataset::checkout_version(1)` to open the dataset at its first version. Verify it has half the rows of the latest version.

```rust
let dataset = Dataset::open(&path).await?;
let v1 = dataset.checkout_version(1).await?;
println!("v1 row count: {}", v1.count_rows(None).await?);
```

Goal: feel time travel work.

### 4.5 Read the optimize module shape (★★★)

Open [`rust/lance/src/dataset/optimize.rs`](../rust/lance/src/dataset/optimize.rs). Use the editor outline. Find:

- `pub struct CompactionOptions`
- `pub async fn compact_files`
- `pub fn plan_compaction`

Read the doc comments on those three. Don't read the bodies — just understand what they do at the API level. This is one of the most complex internal subsystems in Lance; today is just orientation.

---

## Day 5 — indices + bindings

### 5.1 Build an IVF_PQ index in Rust (★★)

Modify or create an example that:

1. Generates a few thousand random 128-dim float32 vectors using `lance-datagen` or by hand.
2. Writes them as a Lance dataset.
3. Calls `dataset.create_index(...)` with `IndexType::IvfPq`.
4. Runs a query with `Scanner::nearest(...)`.

The repo's [`rust/examples/src/ivf_hnsw.rs`](../rust/examples/src/ivf_hnsw.rs) is a great starting point — adapt it to `IVF_PQ` instead of `IVF_HNSW_PQ` and watch the differences.

### 5.2 Inspect the index proto (★)

Read [`protos/index.proto`](../protos/index.proto). What metadata is stored per index? What's stored in the `index_details: prost_types::Any` payload? (Hint: each index type defines its own proto in its own crate.)

### 5.3 Compare scalar index types (★★)

Pick a column type and walk through which scalar indices apply:

| Column                    | BTree | Bitmap | NGram | Inverted | RTree |
|--------------------------|-------|--------|-------|----------|-------|
| `id BIGINT` (range)      |       |        |       |          |       |
| `category VARCHAR(16)`   |       |        |       |          |       |
| `title TEXT`             |       |        |       |          |       |
| `description LONGTEXT`   |       |        |       |          |       |
| `geom GEOMETRY`          |       |        |       |          |       |

Fill in the table. Cross-reference with `skills/lance-user-guide/references/index-selection.md` to check yourself.

### 5.4 Run the Python smoke test (★★)

Per `python/AGENTS.md`:

```bash
cd python
uv sync --extra tests --extra dev
uv run pytest python/tests/test_dataset.py -k test_write_dataset_basic
```

This builds the Rust extension and runs a small test. The first run is slow.

### 5.5 Read a binding (★★)

Open [`python/src/scanner.rs`](../python/src/scanner.rs). Read it fully (~5 KB).

Pair each method with the Rust core method it calls (in `rust/lance/src/dataset/scanner.rs`). You should see a clear "thin wrapper" pattern.

### 5.6 Open-ended: contribute (★★★)

Find a small, real improvement:

- A missing doc comment on a public API.
- A typo in a module-level comment.
- A clippy warning (`cargo clippy --workspace --all-targets`).
- An unimplemented test case in `lance-core`.

Make the change, run the relevant tests, and sit with how it feels. You have made your first edit to a serious open-source codebase. That's a real milestone.

---

## Stretch goals (do any time)

### Run a benchmark

```bash
cargo bench -p lance-encoding --bench buffer
```

(Or any other bench under any crate's `benches/` directory.) Read the criterion output.

### Profile a real query

Use `pprof` (already a dev dependency) to profile a vector search. Integrate it into a binary that opens a dataset and runs 1000 queries. Generate a flamegraph and stare at it.

### Build the full docs site

```bash
cd docs
make serve
```

The Lance website is built from this directory using mkdocs. Browse it locally; the `format/` section is especially good.

### Read a `tests/` file

Pick any file in [`rust/lance/src/dataset/tests/`](../rust/lance/src/dataset/tests). Tests are surprisingly readable because they have to set up everything from scratch — they're great worked examples of the public API.

---

If you're stuck on any exercise, ping me with the exact step you're on and the output you got. I can read the code with you and unblock in minutes.
