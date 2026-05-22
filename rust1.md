# Rust patterns you will see repeatedly in Lance

A field guide to the idioms that show up so often you should recognize them on sight. Keep this open while reading source.

Each pattern shows the shape, why it exists, and a real example from the codebase. When you encounter unfamiliar Rust elsewhere, search this file first.

---

## 1. `Arc<T>` shared ownership

**Shape:**
```rust
let shared: Arc<MyType> = Arc::new(MyType::new());
let same = Arc::clone(&shared);   // cheap; just bumps a refcount
```

**Why:** Rust's "one owner" rule is too strict for async code where many futures share the same data. `Arc` lets you have many owners; the value drops when the last `Arc` does.

**You'll see:**
- `Arc<dyn Trait>` for runtime polymorphism + sharing
- `Arc<RecordBatch>` for cheaply-shared Arrow batches
- `Arc<Schema>` for schemas (which can be megabytes for wide tables)
- `Arc<Manifest>` because the manifest is read by many concurrent scanners

**Example from `rust/lance/src/dataset.rs`:**
```rust
pub struct Dataset {
    pub(crate) object_store: Arc<ObjectStore>,
    pub(crate) commit_handler: Arc<dyn CommitHandler>,
    pub manifest: Arc<Manifest>,
    pub(crate) session: Arc<Session>,
    pub(crate) fragment_bitmap: Arc<RoaringBitmap>,
    // ... seven Arcs in total
}
```

**Notes:** Cloning a `Dataset` is cheap *because of* the `Arc`s; it just bumps refcounts. The fields themselves are not deep-cloned.

---

## 2. `Arc<dyn Trait>` for polymorphism

**Shape:**
```rust
trait Index: Send + Sync { fn name(&self) -> &str; }
let indices: Vec<Arc<dyn Index>> = ...;
for index in &indices {
    println!("{}", index.name());
}
```

**Why:** When you have many implementations of a trait and want to hold a heterogeneous collection of them, you need a *trait object*. `Arc<dyn Trait>` is the shareable form.

**Example:** Lance keeps `Vec<Arc<dyn ScalarIndex>>` per column to support multiple index types over the same data.

**Trait bound trick:** `dyn Trait` requires the trait to be *object-safe*. The most common gotcha is `async fn` in traits — you need `#[async_trait]` to make those object-safe.

---

## 3. The builder pattern with `with_*` methods

**Shape:**
```rust
let dataset = DatasetBuilder::from_uri(uri)
    .with_index_cache_size_bytes(1 << 30)
    .with_version(42)
    .with_object_store_params(params)
    .load()
    .await?;
```

**Why:** Lance config structs have many optional fields. Constructors with 10 parameters are unreadable, and Rust doesn't have keyword arguments. Builders fix this.

**Convention** (per `rust/AGENTS.md`):
- `MyType::new(required_args)` returns a builder.
- `with_X(self, value) -> Self` (consumes self, returns Self) sets optional fields.
- A terminal call (`build()`, `load()`, `execute()`) consumes the builder and returns the result.

**Example from `lance/src/dataset/scanner.rs`:**
```rust
let mut scanner = dataset.scan();
scanner
    .project(&["id", "name"])?
    .filter("id > 100")?
    .limit(Some(10), None)?
    .with_row_id();
let batches = scanner.try_into_stream().await?;
```

(Note: `Scanner` uses `&mut self` rather than consuming, but the spirit is the same.)

---

## 4. `Result<T>` and `?` for error propagation

**Shape:**
```rust
fn outer() -> Result<X> {
    let a = inner1()?;       // returns early if Err
    let b = inner2(a)?;
    Ok(transform(b))
}
```

**Why:** Forces you to handle errors explicitly without try/catch ceremony.

**Lance specifics:**
- `Result<T>` (one type parameter) is `lance_core::Result<T>` = `std::result::Result<T, lance_core::Error>`.
- The `?` operator calls `From::from` on the error if the types don't match. Lance has `impl From<ArrowError> for Error`, `impl From<object_store::Error> for Error`, etc., so `?` works across all foreign error types.
- When a function needs to return errors from a third-party type that doesn't have a `From` impl, wrap it: `.map_err(|e| Error::IO { source: Box::new(e), location: location!() })`.

**Anti-patterns to recognize and reject:**
- `.unwrap()` and `.expect()` in library code (allowed in tests only).
- `option.is_some()` followed by `option.unwrap()` — use `if let Some(x) = option`.
- `panic!()` for fallible operations — use `Error::Internal` if truly unreachable.

---

## 5. `?` with `Option`

```rust
fn first_char(s: Option<&str>) -> Option<char> {
    let s = s?;          // returns None if input is None
    s.chars().next()
}
```

`?` works on `Option` too. Inside a function returning `Option`, `?` short-circuits on `None`.

For mixing `Option` and `Result`, use `.ok_or(...)` or `.ok_or_else(...)`:

```rust
fn lookup(name: &str) -> Result<&Field> {
    schema.field_by_name(name)
        .ok_or_else(|| Error::not_found(format!("field {}", name)))
}
```

---

## 6. `async fn` and `.await`

**Shape:**
```rust
async fn open(path: &str) -> Result<Dataset> {
    let manifest = read_manifest(path).await?;
    let schema = parse_schema(&manifest)?;
    Ok(Dataset { manifest, schema })
}
```

**Why:** Lance's hot path is I/O. Async lets thousands of in-flight reads coexist on a few OS threads.

**Mental model:**
- `async fn foo() -> T` is shorthand for `fn foo() -> impl Future<Output = T>`.
- `.await` suspends until the future is ready.
- A *runtime* (`tokio`) drives the futures.

**Top-level entry:**
```rust
#[tokio::main]
async fn main() -> Result<()> { ... }
```

or, in tests:

```rust
#[tokio::test]
async fn test_thing() -> Result<()> { ... }
```

---

## 7. `#[async_trait]` macro

**Shape:**
```rust
#[async_trait]
pub trait CommitHandler: Send + Sync {
    async fn commit(&self, manifest: Manifest) -> Result<()>;
}
```

**Why:** Until Rust 1.75 you couldn't have `async fn` in trait definitions. The `async-trait` crate's macro rewrites your trait to use `Pin<Box<dyn Future<Output = T> + Send>>` returns, which works on stable.

**Even now**, Lance keeps using `async_trait` consistently because:
- Object-safety: `async fn in trait` (the modern stable form) doesn't allow `dyn Trait` directly.
- Consistency across the codebase.

When you see `#[async_trait]`, you can read the trait as if `async fn` worked natively.

---

## 8. `Stream<Item = T>` — async iterators

**Shape:**
```rust
use futures::StreamExt;

let mut stream = dataset.scan().try_into_stream().await?;
while let Some(batch) = stream.next().await {
    let batch = batch?;
    // ...
}
```

**Combinators (from `futures::StreamExt` and `TryStreamExt`):**
- `.map`, `.filter`, `.take`, `.skip`, `.fold` — synchronous transformations
- `.then`, `.and_then` — async transformations
- `.try_collect::<Vec<_>>()` — collect a `Stream<Item = Result<T>>` into `Result<Vec<T>>`
- `.boxed()` — produce a `BoxStream<'static, T>` (heap-allocated, type-erased)

**Why:** Lance reads are streaming — you don't want all batches in memory at once. Streams are the foundation for that.

---

## 9. `BoxFuture<'static, T>` — type-erased future

**Shape:**
```rust
fn submit_request(...) -> BoxFuture<'static, Result<Vec<Bytes>>>;
```

**Why:** A function returning `impl Future` only works if the concrete future type is fixed. Trait methods can't return `impl Future` (well, they can post-1.75, but it's restricted). `BoxFuture` (a `Pin<Box<dyn Future + Send>>`) is the type-erased equivalent.

**`'static` lifetime:** Means the future doesn't borrow anything from the caller. This is necessary because the future may outlive the function call.

**Construction:**
```rust
async fn inner() -> Result<()> { ... }

fn outer() -> BoxFuture<'static, Result<()>> {
    inner().boxed()  // From FutureExt
}
```

---

## 10. `LazyLock<T>` for global constants

**Shape:**
```rust
pub static ROW_ID_FIELD: LazyLock<ArrowField> =
    LazyLock::new(|| ArrowField::new(ROW_ID, DataType::UInt64, true));

// Use with .deref() (often automatic):
let f: &ArrowField = &ROW_ID_FIELD;
```

**Why:** Some "constants" can't be initialized at compile time (e.g., they involve allocation or function calls). `LazyLock` runs the closure on first access and caches the result.

This replaces the older `lazy_static!` macro.

---

## 11. The `derive` zoo

You'll see many `#[derive(...)]` attributes. Quick reference:

| Derive | What it does |
|--------|------|
| `Debug` | Auto-implements `{:?}` formatting |
| `Clone` | Adds `.clone()` |
| `Default` | Adds `MyType::default()` |
| `PartialEq, Eq` | `==` and `!=` |
| `PartialOrd, Ord` | `<`, `>`, etc. (Ord requires Eq) |
| `Hash` | Usable as `HashMap` key |
| `Copy` | Auto-copies on assignment (only for tiny `Copy`-eligible types) |
| `serde::Serialize, Deserialize` | JSON/MessagePack/etc. via `serde` |
| `Snafu` | Error enum with snafu (Lance's choice) |
| `DeepSizeOf` | For cache size accounting |
| `prost::Message` | Protobuf serialization (auto-applied by prost-build) |

Lance-specific: `DeepSizeOf` is on almost every cacheable type.

---

## 12. `match` and `if let`

**Match:**
```rust
match self.mode {
    WriteMode::Create => self.create_dataset(),
    WriteMode::Append => self.append_to_existing(),
    WriteMode::Overwrite => self.overwrite_existing(),
}
```

`match` is exhaustive — if you add a new `WriteMode` variant, the compiler tells you everywhere you forgot to handle it. Lance uses this everywhere there's an enum.

**`if let`:**
```rust
if let Some(handler) = &self.commit_handler {
    handler.commit(manifest).await?;
}
```

For when you only care about one variant.

**`let ... else`:**
```rust
let Some(reader) = self.reader.as_ref() else {
    return Err(Error::not_found("reader not initialized"));
};
// reader is in scope below
```

Unwraps an `Option`/`Result` or runs a divergent branch (return/break/panic). Common for early-return validation.

---

## 13. Newtypes for type safety

**Shape:**
```rust
pub struct RowAddress(u64);  // not just a u64

impl RowAddress {
    pub fn new_from_parts(frag: u32, off: u32) -> Self {
        Self(((frag as u64) << 32) | off as u64)
    }
    pub fn fragment_id(&self) -> u32 { (self.0 >> 32) as u32 }
}
```

**Why:** A bare `u64` could be a row address, a row id, a byte offset, or a count. Wrapping it in a newtype catches type confusion at compile time.

Lance enforces this hard. `rust/AGENTS.md` says: never use raw `u64` for both `RowId` and `RowAddress`.

**Cost:** zero. The compiler optimizes the wrapper away.

---

## 14. Extension traits (`*Ext`)

**Shape:**
```rust
pub trait DataTypeExt {
    fn is_binary_like(&self) -> bool;
    fn is_struct(&self) -> bool;
}

impl DataTypeExt for arrow_schema::DataType { ... }

// Now anyone with `use lance_arrow::DataTypeExt;` can call .is_binary_like()
let dt: DataType = ...;
if dt.is_binary_like() { ... }
```

**Why:** You want to add methods to a foreign type (like `arrow_schema::DataType`) but the orphan rule says you can't `impl` directly. Extension traits work around this: define a trait you own, implement it on the foreign type.

**Lance examples:** `DataTypeExt`, `RecordBatchExt`, `SchemaExt` in `lance-arrow`. `DatasetIndexExt` in `lance/src/index/api.rs`. `ObjectStoreExt` in `lance-io`.

**Convention:** suffix the trait name with `Ext`.

---

## 15. `From` and `Into` for conversions

**Shape:**
```rust
impl From<arrow_schema::Field> for lance_core::datatypes::Field {
    fn from(f: arrow_schema::Field) -> Self { ... }
}

// Then both work:
let lance_field: lance_core::datatypes::Field = arrow_field.into();   // via Into
let lance_field = lance_core::datatypes::Field::from(arrow_field);   // via From
```

`Into<T>` is auto-derived from `From<T>`. Always implement `From`, never `Into` directly.

For fallible conversions, use `TryFrom`/`TryInto`:

```rust
impl TryFrom<i32> for IndexType {
    type Error = lance_core::Error;
    fn try_from(value: i32) -> Result<Self> { ... }
}
```

---

## 16. `AsRef<T>` for flexible inputs

**Shape:**
```rust
pub async fn open_dataset<T: AsRef<str>>(uri: T) -> Result<Dataset> {
    let uri: &str = uri.as_ref();
    // ...
}

// All work:
open_dataset("s3://bucket/data.lance").await;
open_dataset(String::from("s3://...")).await;
open_dataset(&some_string).await;
```

**Why:** Lets the caller pass any `&str`-like type without forcing them to convert. Generic but cheap.

Common bounds: `AsRef<str>`, `AsRef<Path>`, `AsRef<[u8]>`, `Into<String>`.

---

## 17. `RoaringBitmap` for compact `Set<u32>`

**Shape:**
```rust
let mut bitmap = RoaringBitmap::new();
bitmap.insert(42);
bitmap.insert(100);
if bitmap.contains(42) { /* yes */ }
let intersection = &a & &b;  // bitwise ops are bitmap operations
```

**Why:** A `HashSet<u32>` uses ~24 bytes per element. A `RoaringBitmap` uses 2 bits per element on dense ranges and degrades gracefully on sparse data.

Lance uses `RoaringBitmap` for fragment ids, deletion vectors, prefilter masks, index coverage. `rust/AGENTS.md` mandates using it instead of `HashSet<u32>`.

---

## 18. `tracing` instead of `println!`

**Shape:**
```rust
use tracing::{info, debug, warn, instrument};

#[instrument(skip(self))]
async fn read_manifest(&self, path: &Path) -> Result<Manifest> {
    debug!("reading manifest from {}", path);
    let bytes = self.object_store.get(path).await?;
    info!(size = bytes.len(), "manifest loaded");
    Ok(parse(bytes)?)
}
```

**Why:**
- `println!` is forbidden in library code (clippy lint `print_stdout` is `deny`).
- `tracing` produces structured logs that can be filtered (`RUST_LOG=info`), formatted as JSON, and routed through async-aware subscribers.
- `#[instrument]` creates a span around the function with its arguments, so you get nested timings for free.

**Levels:** `error!`, `warn!`, `info!`, `debug!`, `trace!`. Lance convention (per `rust/AGENTS.md`): `debug!` for routine ops, `info!` for state changes, `warn!` for unexpected.

---

## 19. `cfg` and feature flags

**Shape:**
```rust
#[cfg(feature = "aws")]
pub mod dynamic_credentials;

#[cfg(target_os = "linux")]
pub mod uring;

#[cfg(test)]
mod tests {
    // only compiled in test builds
}
```

**Why:** Lance has many optional features (cloud backends, geospatial, io-uring). They're gated so you only pay for what you use.

Common flags you'll see:
- `feature = "aws"` / `"azure"` / `"gcp"` — cloud storage
- `feature = "dynamodb"` — DynamoDB external manifest store
- `feature = "geo"` — geospatial types and indices
- `target_os = "linux"` — io-uring
- `cfg(test)` — test-only code

---

## 20. `pub(crate)` and re-exports

**Shape:**
```rust
mod private_module;

pub(crate) use private_module::InternalType;  // visible within the crate

pub use private_module::PublicType;          // visible outside
```

**Why:** Lance has lots of internals it doesn't want to expose. `pub(crate)` is "public to my crate but private to the world." `rust/AGENTS.md` recommends using it liberally.

The pattern in many crates: re-export only the curated public surface from `lib.rs`:

```rust
pub use dataset::Dataset;
pub use dataset::scanner::{Scanner, DatasetRecordBatchStream};
pub use error::{Error, Result};
```

---

## 21. `Default` for config structs

**Shape:**
```rust
#[derive(Debug, Default)]
pub struct WriteParams {
    pub max_rows_per_file: usize,
    pub mode: WriteMode,
    // ...
}

// Then:
let params = WriteParams::default();
let params = WriteParams { mode: WriteMode::Append, ..Default::default() };
```

**Why:** Most config structs have many fields with sensible defaults. `Default` means callers only override what they care about.

The `..Default::default()` syntax is "spread the rest from the default." Used pervasively for `WriteParams`, `ReadParams`, `ScanConfig`, etc.

---

## 22. Compile-time string constants

**Shape:**
```rust
pub const ROW_ID: &str = "_rowid";
pub const INDEX_FILE_NAME: &str = "index.idx";
pub const MAGIC: &[u8; 4] = b"LANC";
```

**Why:** Centralizing magic strings prevents typos and makes refactoring safe. Lance does this everywhere — column names, file names, schema metadata keys, etc.

When you see a string literal in code that *isn't* a constant, that's a code smell.

---

## 23. `cfg_attr` for conditional derives

**Shape:**
```rust
#[derive(Debug, Clone)]
#[cfg_attr(test, derive(PartialEq))]
struct Foo { ... }
```

`PartialEq` is only derived in test builds. Saves compile time and runtime size in production.

---

## 24. The `?Sized` and `Send`/`Sync` bounds

**Shape:**
```rust
fn process<T: Send + Sync + 'static>(item: T) { ... }
```

- `Send` — safe to transfer to another thread.
- `Sync` — safe to share via reference between threads (`&T: Send`).
- `'static` — doesn't borrow non-static data.

You'll see these on every async type. They're propagated through generic bounds and trait objects:

```rust
let task: BoxFuture<'static, Result<()>> = ...;
//                  ^^^^^^^                       no borrow
//                                              the Future inside is Send (BoxFuture is Send by default)
```

If you ever get an "X is not Send" error, you're trying to share something across an `await` that can't be (e.g., an `Rc` or a `RefCell`). The fix is usually to use `Arc` and `parking_lot::Mutex` instead.

---

## 25. Smart logging at hot paths

**Shape:**
```rust
#[instrument(skip(self, batch))]
async fn write_batch(&mut self, batch: RecordBatch) -> Result<()> {
    debug!(rows = batch.num_rows(), "writing batch");
    // ...
}
```

The `#[instrument(skip(...))]` says "trace this function, but don't include `self` and `batch` in the span fields" (because they'd be enormous). Without `skip`, every span carries the full debug-printed argument set.

This is everywhere on hot paths. It costs almost nothing when tracing is disabled.

---

## 26. `Box<dyn std::error::Error + Send + Sync + 'static>`

You'll see this monstrosity everywhere it's hidden behind a type alias:

```rust
type BoxedError = Box<dyn std::error::Error + Send + Sync + 'static>;
```

**Why:** A type-erased error that meets all the requirements for being `?`-propagated across thread boundaries. Lance uses this as the inner `source` of every error variant.

When you see `BoxedError` in a function signature, just read it as "any error type, dynamically."

---

## 27. The orphan rule and how to dodge it

You can implement a trait for a type only if you own *either* the trait *or* the type. This is the orphan rule. So you can't:

```rust
// FORBIDDEN: neither MyTrait nor String is yours
impl MyTrait for String { ... }
```

Workarounds you'll see in Lance:

- **Extension traits** (pattern 14 above) — define your own trait, implement it on foreign types.
- **Newtype wrappers** — `struct Wrap(ForeignType);` then implement traits on `Wrap`.
- **From/Into impls** between owned types — these aren't "implementing a trait on a foreign type"; they're implementing your conversion.

---

## 28. `unsafe` and SIMD

There are pockets of `unsafe` in Lance, mostly in:

- SIMD intrinsics (`lance-linalg` for vector distance, `lance-encoding` for bit-packing).
- FFI boundaries (PyO3, JNI, io-uring).
- Buffer reuse in hot encoding paths (with `// SAFETY:` comments explaining the invariant).

You won't write `unsafe` for the curriculum, but recognize it. Every `unsafe` block must have a `// SAFETY:` comment per Lance's standards.

---

## 29. Reading large files with the editor outline

A practical pattern, not a Rust feature: when a file is over 30 KB, **never read it linearly.** Use the editor's outline view (most editors: `Ctrl-Shift-O` or "Outline" pane) and:

1. Find the public `struct` and read its fields.
2. Find the public `impl` block and skim method names.
3. Pick 3-5 methods to read in detail.
4. Skip the test module unless you're debugging.

This is how you absorb 100 KB files without going insane.

---

## When in doubt

- If a Rust feature surprises you, search for it in this file first.
- If it's not here, ask Kiro: "what does this `Pin<Box<...>>` mean in `lance-encoding/src/lib.rs:42`?" with the file and line.
- The Rust Reference is exhaustive but slow to navigate: <https://doc.rust-lang.org/reference/>.
- The async chapter of the Tokio book is the best brief intro to async Rust: <https://tokio.rs/tokio/tutorial>.
