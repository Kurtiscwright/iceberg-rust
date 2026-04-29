<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one
  ~ or more contributor license agreements.  See the NOTICE file
  ~ distributed with this work for additional information
  ~ regarding copyright ownership.  The ASF licenses this file
  ~ to you under the Apache License, Version 2.0 (the
  ~ "License"); you may not use this file except in compliance
  ~ with the License.  You may obtain a copy of the License at
  ~
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing,
  ~ software distributed under the License is distributed on an
  ~ "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  ~ KIND, either express or implied.  See the License for the
  ~ specific language governing permissions and limitations
  ~ under the License.
-->

# RFC: File Format API for Apache Iceberg Rust

## Background

### Current state

The `iceberg` crate (version 0.9.0, Rust 1.92 as of this writing) is the core library of the Apache Iceberg Rust project. Its module layout in `crates/iceberg/src/lib.rs` exposes public modules for `spec`, `arrow`, `writer`, `scan`, `io`, `expr`, `transaction`, and others. The crate depends directly on the `parquet` crate (with the `async` feature) and on the `arrow-*` crates. It has no feature flags today.

For data file writing, the crate provides a three-layer architecture described in `crates/iceberg/src/writer/mod.rs`:

1. **`FileWriter` / `FileWriterBuilder`** in `writer/file_writer/mod.rs`: traits for physical file writers, generic over an output type (defaulting to `Vec<DataFileBuilder>`). `FileWriterBuilder::build` takes an `OutputFile` and returns a `FileWriter`. `FileWriter::write` takes a `&RecordBatch`.

2. **`IcebergWriter` / `IcebergWriterBuilder`** in `writer/mod.rs`: traits for logical Iceberg writers (data files, equality deletes, position deletes, partitioning), defaulting to `RecordBatch` input and `Vec<DataFile>` output.

3. **Concrete implementations**: `ParquetWriterBuilder` and `ParquetWriter` in `writer/file_writer/parquet_writer.rs` are the only `FileWriter` implementation. Higher-level writers such as `DataFileWriterBuilder` (`writer/base_writer/data_file_writer.rs`) and `EqualityDeleteFileWriterBuilder` (`writer/base_writer/equality_delete_writer.rs`) are generic over any `FileWriterBuilder`, but every example, test, and integration instantiates them with `ParquetWriterBuilder`.

The trait layer is format-agnostic. Every concrete instantiation uses Parquet.

For data file reading, the crate provides `ArrowReaderBuilder` and `ArrowReader` in `arrow/reader.rs`. Both are Parquet-specific despite the generic name. `ArrowReader::process_file_scan_task` calls `open_parquet_file` directly, constructs a `ParquetRecordBatchReaderBuilder`, and uses Parquet-specific row group filtering and page index logic. `TableScan::to_arrow` in `scan/mod.rs` wires `ArrowReaderBuilder` in as the only reader path. `FileScanTask` carries a `data_file_format: DataFileFormat` field, but `process_file_scan_task` ignores it. `DataFileFormat` appears throughout `src/`. Almost every non-test reference is `DataFileFormat::Parquet`. The only non-Parquet references are two uses of `DataFileFormat::Avro` in `transaction/snapshot.rs` for manifest files.

The `DataFileFormat` enum in `spec/manifest/data_file.rs` has four variants: `Avro`, `Orc`, `Parquet`, and `Puffin`. Puffin files hold statistics and deletion vectors rather than row data. They are handled in `crates/iceberg/src/puffin/` and are out of scope for this RFC (see Non-Goals). Data file support today:

| Format  | Data file read | Data file write | Manifests |
|---------|----------------|-----------------|-----------|
| Parquet | Yes            | Yes             | No        |
| Avro    | No             | No              | Yes       |
| ORC     | No             | No              | No        |

A table containing ORC or Avro data files cannot be read from Rust today, even though both are valid per the Iceberg spec.

Arrow `RecordBatch` is the only in-memory data representation in the crate. It is the input type for every writer trait and the output type for every reader. The `arrow/` module provides schema conversion (`schema_to_arrow_schema`, `arrow_schema_to_schema`), `RecordBatchTransformer` for schema evolution and constant column injection, and the Parquet-to-Arrow read path. Every integration crate, including `iceberg-datafusion`, consumes Arrow. The crate does not define a generic `Record` type and does not integrate with engine-specific row types such as Java's `InternalRow` or `RowData`.

### Pain points

1. **No extension point for new formats.** Adding ORC means editing `ArrowReader` to branch on `DataFileFormat` and threading ORC-specific logic through every layer that touches it. The write path has the same shape.

2. **Parquet assumptions leak into generic code.** `ArrowReaderBuilder` exposes Parquet-specific options (`with_metadata_size_hint`, `with_row_group_filtering_enabled`, `with_row_selection_enabled`) that are meaningless for ORC or Avro. The name "ArrowReader" conflates the in-memory representation (Arrow) with the on-disk format (Parquet).

3. **No format-agnostic statistics.** `ParquetWriter` computes column sizes, value counts, and min/max bounds through `MinMaxColAggregator` and `NanValueCountVisitor`, both tightly coupled to Parquet's `Statistics` type. Another format cannot produce comparable manifest metadata without a shared statistics interface.

4. **V3 types will need per-format serialization.** The V3 spec adds Variant and Geometry types. Each format encodes them differently: Parquet uses variant shredding, ORC uses binary, Avro uses union types. Implementing either type without a format abstraction means new `match` arms in every reader and writer code path.

### Prior work

The Java project shipped `FormatModel<D, S>` in February 2026 (PR [#12774](https://github.com/apache/iceberg/pull/12774)) after a 10-month review. Implementations for Parquet, Avro, ORC, and Arrow followed in PRs [#15253](https://github.com/apache/iceberg/pull/15253) through [#15258](https://github.com/apache/iceberg/pull/15258). Engine migrations for Spark ([#15328](https://github.com/apache/iceberg/pull/15328)) and Flink ([#15329](https://github.com/apache/iceberg/pull/15329)) landed shortly after.

Java's `FormatModel` carries two generic parameters: a data type `D` (Java uses `Record`, Spark `InternalRow`, Flink `RowData`, and Arrow `ColumnarBatch`) and an engine schema type `S` (Spark `StructType`, Flink `RowType`, and others). A static `FormatModelRegistry` stores implementations keyed by `Pair<FileFormat, Class<?>>` and populates itself through Java reflection at startup. The old `Parquet.WriteBuilder`, `Avro.WriteBuilder`, and `ORC.WriteBuilder` are not deprecated. The new `FormatModel` implementations wrap them rather than replace them.

PyIceberg has an open proposal for the equivalent capability in issue [apache/iceberg-python#3100](https://github.com/apache/iceberg-python/issues/3100), with an in-progress PR at [apache/iceberg-python#3119](https://github.com/apache/iceberg-python/pull/3119). Because PyIceberg uses PyArrow as its only in-memory representation, the proposal drops Java's generic type parameters and keys the registry on file format alone. Two prior PyIceberg ORC PRs ([#790](https://github.com/apache/iceberg-python/pull/790), [#2236](https://github.com/apache/iceberg-python/pull/2236)) were closed as stale without merging, which reinforces the case for landing an abstraction layer before adding new formats.

Design decisions in this RFC that differ from Java (no generics, single-dimension registry key, hard cutover from the old Parquet types) are justified inline where they appear. Specific Java design points that shaped those decisions are called out in Alternatives Considered.

## Goals

The user-facing outcome of this proposal is that every Iceberg data file and delete file flows through the same stable and extensible API. Parquet is the first format to land. ORC, Avro, and others follow on the same interface. Every goal below serves that outcome.

1. Define a `FormatModel` trait that encapsulates format-specific read and write behavior independent of the on-disk format.

2. Remove hard-coded Parquet assumptions from scan and write orchestration. After this work, `TableScan::to_arrow` and `DataFileWriterBuilder` dispatch through the format abstraction instead of constructing Parquet types directly.

3. Provide a registry that maps `DataFileFormat` values to `FormatModel` implementations, so callers obtain readers and writers without naming the concrete format type.

4. Define a conformance test suite (TCK) that any `FormatModel` implementation must pass before it merges.

5. Match the Java and PyIceberg designs where they align, and diverge where Rust's single-data-type ecosystem and pre-1.0 status justify it. Divergences are called out inline.

## Non-Goals

The items below are deliberately out of scope to keep this proposal focused on the abstraction and its Parquet implementation. Most are follow-up work that the API enables but does not itself deliver.

1. **Ship new format implementations.** This RFC lands the abstraction and a Parquet implementation. ORC, Avro data-file, Vortex, and Lance are follow-up RFCs.

2. **Introduce a plugin protocol or runtime library loading.** Rust does not offer a clean mechanism for loading compiled plugins at runtime. A runtime-linking approach using `libloading` or similar would expand scope beyond what the formats currently under discussion (Parquet, ORC, Avro, Vortex, Lance) require.

3. **Add Puffin support to the FormatModel.** Puffin files hold statistics and deletion vectors rather than row data. They have a different lifecycle from data files and are already handled separately in `crates/iceberg/src/puffin/`.

4. **Redesign the writer trait hierarchy.** The existing `IcebergWriter` and `FileWriter` layering is sound. This RFC adds a format abstraction beneath `FileWriter`, not a replacement for it.

5. **Implement variant shredding or encryption.** Java exposes `engineProjection` and `engineSchema` as extension points for variant shredding and similar format-specific type mapping, and `withFileEncryptionKey` and `withAADPrefix` for Parquet encryption. Equivalent hooks are noted as future extensions in the Rust design. Implementing either requires a dedicated RFC.

6. **Change the Iceberg table spec.** This proposal is a Rust-only API change. It does not modify the Iceberg spec, the manifest format, the manifest list format, or the on-disk layout of any file.

7. **Modify manifest read or write paths.** Manifests and manifest lists remain in Avro and are handled by the existing `ManifestReader` and `ManifestWriter` paths. The File Format API is about data files and delete files only.


## Design

The Rust API is three traits and a registry. `FormatModel` is the trait that each format implementation provides. `FormatReadBuilder` and `FormatWriteBuilder` are the per-operation configurators that `FormatModel` returns. `FormatRegistry` maps `DataFileFormat` values to `FormatModel` instances. None of the traits carry generic parameters. The subsections below introduce each type, and a final "Design rationale" subsection explains the choices.

### The FormatModel trait

```rust
pub trait FormatModel: Send + Sync + 'static {
    fn format(&self) -> DataFileFormat;
    fn read_builder(&self, input: InputFile) -> Box<dyn FormatReadBuilder>;
    fn write_builder(&self, output: OutputFile) -> Box<dyn FormatWriteBuilder>;
}
```

Each implementation registers one instance per `DataFileFormat` variant it supports. The `format` method returns that variant. `read_builder` and `write_builder` are the entry points for reading and writing a file. Both return trait objects so that the registry can hand them back from a `DataFileFormat`-keyed lookup.

The data type is fixed to Arrow `RecordBatch`, the Iceberg schema type to `iceberg::spec::Schema`, and the physical schema type to `arrow::datatypes::SchemaRef`. These are the only types in iceberg-rust today that would fill the roles Java uses generic parameters for. Arguments for keeping them fixed, including comparisons with Java's `FormatModel<D, S>`, are in "Design rationale" below.

### The read and write builders

`FormatReadBuilder` and `FormatWriteBuilder` configure a single read or write. `FormatModel` produces them, and the caller consumes them with `build`.

```rust
pub trait FormatReadBuilder: Send {
    fn project(&mut self, schema: Schema) -> &mut Self;
    fn filter(&mut self, predicate: BoundPredicate) -> &mut Self;
    fn split(&mut self, start: u64, length: u64) -> &mut Self;
    fn case_sensitive(&mut self, case_sensitive: bool) -> &mut Self;
    fn batch_size(&mut self, batch_size: usize) -> &mut Self;
    fn build(self: Box<Self>) -> BoxFuture<'static, Result<ArrowRecordBatchStream>>;
}

pub trait FormatWriteBuilder: Send {
    fn schema(&mut self, schema: Schema) -> &mut Self;
    fn set(&mut self, key: String, value: String) -> &mut Self;
    fn build(self: Box<Self>) -> BoxFuture<'static, Result<Box<dyn FormatFileWriter>>>;
}
```

Both builders take Iceberg `Schema` values. Format implementations convert to physical schemas internally using `schema_to_arrow_schema` from `arrow/schema.rs`. This deliberately departs from Java's `ReadBuilder<D, S>`, which has a separate `engineProjection(S)` method for variant shredding and similar per-engine type mapping. Rust's builders expose one projection surface, and the hook for a variant-shredding "engine projection" is a future-extension point described in "Design rationale."

`FormatWriteBuilder::build` produces a `FormatFileWriter`:

```rust
pub trait FormatFileWriter: Send {
    fn write(&mut self, batch: &RecordBatch) -> BoxFuture<'_, Result<()>>;
    fn close(self: Box<Self>) -> BoxFuture<'static, Result<Vec<DataFileBuilder>>>;
}
```

The async methods return `BoxFuture` rather than using `async fn` in traits. The `self: Box<Self>` signature on `build` and `close` lets those methods consume the value while keeping the traits object-safe. Both patterns are forced by the trait-object boundary at the registry. The "BoxFuture instead of async fn in traits" and "Dynamic dispatch at the registry, static inside the format" subsections under "Design rationale" explain why.

### The FormatRegistry

`FormatRegistry` maps `DataFileFormat` values to `FormatModel` instances.

```rust
pub struct FormatRegistry {
    models: HashMap<DataFileFormat, Box<dyn FormatModel>>,
}

impl FormatRegistry {
    pub fn new() -> Self { ... }
    pub fn register(&mut self, model: Box<dyn FormatModel>) { ... }
    pub fn read_builder(
        &self,
        format: DataFileFormat,
        input: InputFile,
    ) -> Result<Box<dyn FormatReadBuilder>> { ... }
    pub fn write_builder(
        &self,
        format: DataFileFormat,
        output: OutputFile,
    ) -> Result<Box<dyn FormatWriteBuilder>> { ... }
}
```

Using `DataFileFormat` as a `HashMap` key requires adding `#[derive(Hash)]` to the enum. That is a non-breaking addition.

The registry is an owned value, not a global static. Tests construct their own. Applications construct one at startup and pass it to scan planners and write orchestrators. For the common case of a single registry for the lifetime of a process, `default_format_registry()` returns a `&'static FormatRegistry` initialized through `OnceLock` on first call.

`read_builder` and `write_builder` return `Err(Error { kind: ErrorKind::FeatureUnsupported, .. })` for unregistered formats. The error message distinguishes two cases: the format is implemented but its feature flag is disabled in this build, or the format has no implementation in this crate.

### Feature flags

Format implementations live in `iceberg::formats::{format}` and are gated behind a feature flag per format: `format-parquet` on by default, `format-orc` when an ORC implementation lands, and so on. The default feature set includes every format the crate implements. Users who build from source and want a smaller binary disable what they do not need.

`FormatRegistry::default()` registers every format enabled by the current feature set at compile time:

```rust
impl Default for FormatRegistry {
    fn default() -> Self {
        let mut registry = Self::new();
        #[cfg(feature = "format-parquet")]
        registry.register(Box::new(ParquetFormatModel::new()));
        #[cfg(feature = "format-orc")]
        registry.register(Box::new(OrcFormatModel::new()));
        registry
    }
}
```

Format implementations live inside the `iceberg` crate rather than as separate dependencies. This matches how the Java project keeps its `Parquet`, `Avro`, and `ORC` modules inside the `apache/iceberg` repository. In-tree implementations let the community control each format's quality and release cadence and keep the build graph narrow for downstream consumers. Nothing in the trait design prevents a downstream crate from defining its own `FormatModel` and registering it with a custom `FormatRegistry`, but in-tree is the recommended path for formats intended to ship in iceberg-rust itself.

The `parquet` crate remains an unconditional dependency of `iceberg` for now, because non-format code (page index evaluators, row group metric evaluators, delete file loaders) still uses `parquet` types directly. A later pass gates the `parquet` dependency on the `format-parquet` feature once those callers move to format-agnostic abstractions.

### Module layout

```
crates/iceberg/src/formats/
├── mod.rs        # FormatModel, FormatReadBuilder, FormatWriteBuilder, FormatFileWriter
├── registry.rs   # FormatRegistry, default_format_registry
└── parquet.rs    # ParquetFormatModel, wrapping existing ParquetWriter and ArrowReader
```

Additional formats land as `formats/orc/`, `formats/avro_data/`, and so on, with directory-per-format for anything larger than a few hundred lines.

In the initial implementation, `ParquetWriter`, `ParquetWriterBuilder`, `ArrowReader`, and `ArrowReaderBuilder` stay at their current module paths. `ParquetFormatModel` wraps them. Phase 3 of the Migration Plan moves them into `formats/parquet/` and retires the old paths.

### Design rationale

#### No generic over the data type

Java's `FormatModel<D, S>` uses `D` for Spark `InternalRow`, Flink `RowData`, Arrow `ColumnarBatch`, and other engine-native row types. iceberg-rust has one row type: Arrow `RecordBatch`. Every writer accepts it, and every reader returns it. No format on the near-term queue (ORC, Avro data-file, Vortex, Lance) produces anything other than `RecordBatch` at the Iceberg-facing boundary. No engine integration in iceberg-rust today brings an engine-native row type the way Spark and Flink do in Java.

Adding a `D` parameter today means writing `<RecordBatch>` everywhere the trait is used, for no present caller benefit. If a future format cannot bridge to Arrow, the trait can gain an associated type with a default, which is a semver-compatible addition.

#### No generic over the engine schema

Java's `S` parameter serves Spark's `StructType`, Flink's `RowType`, and the other engine-native schema types. iceberg-rust has `iceberg::spec::Schema` for the logical schema and `arrow::datatypes::SchemaRef` for the physical schema, with conversion in `arrow/schema.rs`. There is no third schema type a generic parameter would serve.

Variant shredding is the concrete use case that Java's `engineProjection(S)` method addresses. `FormatReadBuilder` can later gain an `engine_projection(&mut self, schema: ArrowSchemaRef) -> &mut Self` method with a default no-op implementation. Format implementations that support shredding override it. The parameter is `ArrowSchemaRef`, not a generic `S`, because Arrow is the only engine schema in iceberg-rust.

#### Dynamic dispatch at the registry, static inside the format

Registry lookup is inherently dynamic: the caller has a `DataFileFormat` value at runtime. The read or write hot loop should use static dispatch and inline normally. Splitting the two puts `Box<dyn FormatModel>` at the registry boundary and concrete types everywhere inside the format implementation. Dispatch through the trait object happens once per `read_builder` or `write_builder` call, which is once per file. The hot loop sees concrete types.

`object_store` uses the same split (`dyn ObjectStore` at the boundary, concrete `AsyncRead` and `AsyncWrite` streams returned). `datafusion` uses the same split (`dyn TableProvider` at the boundary, concrete `RecordBatchStream` returned).

#### BoxFuture instead of async fn in traits

`async fn` in traits (stable since Rust 1.75) works for static dispatch. It does not work through `dyn Trait`, because the returned future has an opaque type that trait-object dispatch cannot carry. The `async-trait` crate desugars `async fn` to the same `BoxFuture` a manual implementation produces. Using `BoxFuture` explicitly keeps the future's lifetime visible in the signature and avoids one proc-macro expansion per trait.

#### Feature flags instead of inventory or libloading

Two other registration mechanisms were considered.

The `inventory` crate collects static items at link time through platform-specific linker sections. Each format would register itself with a `submit!` macro. `inventory` drops items silently across static-library boundaries, interacts poorly with analysis tools that do not run a real linker, and registers formats that the caller did not explicitly depend on.

Runtime library loading through `libloading` or similar would allow formats to ship as separate shared libraries. Rust has no stable ABI, so implementations would need to match the host's compiler version and feature flags exactly. Every use would need `unsafe`. PyIceberg has flagged runtime loading as an open question without landing a mechanism.

Feature flags work across `cargo check`, `cargo miri`, cross-compilation, and static linking. `datafusion`, `opendal`, and `reqwest` use the same pattern.

#### RecordBatch as the canonical data type

Arrow `RecordBatch` is the interchange format for every Rust data ecosystem iceberg-rust plausibly integrates with. DataFusion consumes it directly. Polars bridges to Arrow with zero-copy conversion for primitive columns. Lance uses its own columnar layout internally and exposes Arrow at read and write boundaries. Every integration and near-term format proposal either produces Arrow or bridges to it.

#### Name: FormatModel

The Java community settled on `FormatModel` after considering `ObjectModel` and `FileAccessFactory` during the 10-month review of PR [#12774](https://github.com/apache/iceberg/pull/12774). This RFC reuses the settled Java name to keep the concept addressable in cross-project discussion.

## Conformance tests (TCK)

A new `FormatModel` implementation should pass a shared conformance test suite before it merges. Java has one (`BaseFormatModelTests<T>`) but it is intentionally minimal: a single struct-of-primitives data generator, no V3 types, no schema evolution, no metadata columns. Issue [#15415](https://github.com/apache/iceberg/issues/15415) tracks the gaps.

The Rust TCK can cover more scenarios because the testing ecosystem (`#[test]`, `proptest`, `rstest`) parameterizes cleanly over any `FormatModel` implementation.

### Phase 1 TCK (lands with the initial FormatModel PR)

1. Round-trip every `PrimitiveType`: boolean, int, long, float, double, decimal, date, time, timestamp, timestamptz, timestamp_ns, timestamptz_ns, string, uuid, fixed, binary. Null and non-null values for each.
2. Projection: write a wide schema, read back a subset of columns.
3. Null handling: all-null columns, mixed null and non-null, required versus optional fields.

### Follow-up TCK (separate PRs)

4. Nested types (struct, list, map, and combinations).
5. Filtering with predicate pushdown. Formats that cannot pushdown (Avro) opt out. Opt-outs are documented.
6. Schema evolution: added columns, renamed columns, widened types.
7. Delete files (equality and position).
8. Metadata columns (`_file`, `_pos`, `_partition`).
9. Split reading: write a large file, read a byte-range split, verify results.
10. Statistics validation on the `DataFileBuilder` returned by `close`: column sizes, value counts, null counts, min/max bounds.

TCK scenarios for V3 types (Variant, Geometry) land with those types in follow-up RFCs.

### Parameterization

The TCK tests use `rstest`:

```rust
#[rstest]
#[case::parquet(ParquetFormatModel::new())]
// #[case::orc(OrcFormatModel::new())]  // added when ORC lands
fn test_round_trip_primitives(#[case] model: impl FormatModel) {
    // ...
}
```

Each new format adds one `#[case]` line to enter the entire TCK matrix.

### Merge gate

A `FormatModel` implementation must pass every TCK scenario that applies to its capabilities before it merges into the `iceberg` crate. Format-specific exceptions (Avro does not support predicate pushdown) are documented on the implementation.


## Migration / Rollout Plan

iceberg-rust is pre-1.0 (currently 0.9.0), and the project has made breaking changes in past 0.x releases. This RFC commits to breaking changes across a sequence of 0.x releases starting with the first release that includes Phase 2. Every break is enumerated in the Breaking Changes table with per-item migration guidance.

The migration has four phases. Phase 1 is additive. Phases 2, 3, and 4 are breaking. The phases are sequential, but new format implementations can begin as soon as Phase 1 merges.

### Phase 1: Land the abstraction (additive only)

**Scope.** Add the `formats/` module with `FormatModel`, `FormatReadBuilder`, `FormatWriteBuilder`, `FormatFileWriter`, `FormatRegistry`, and `ParquetFormatModel`. Add the `format-parquet` feature flag (on by default). Add initial TCK tests for round-trip primitives, projection, and null handling.

**Breaking changes.** None. No existing code is modified.

**PR structure.** One PR for the traits and registry. One PR for `ParquetFormatModel`. One PR for initial TCK tests.

**Validation.** `ParquetFormatModel` must produce identical results to the existing `ParquetWriter` and `ArrowReader` for the same inputs. The TCK tests verify this.

### Phase 2: Migrate internal callers (breaking)

**Scope.** Modify `TableScan::to_arrow` to dispatch through `FormatRegistry` instead of wiring `ArrowReaderBuilder` in directly. Add a `with_format_registry` method to `TableScan` or `TableScanBuilder`. (Open Question 4 tracks which type is the right place.) The default behavior uses `default_format_registry()`. Callers get back an `ArrowRecordBatchStream` from `to_arrow` as before. The signature is unchanged.

Modify `DataFileWriterBuilder` and `EqualityDeleteFileWriterBuilder` to dispatch through the registry. Today both are generic over `B: FileWriterBuilder`, `L: LocationGenerator`, and `F: FileNameGenerator`. After this phase, they accept a `FormatRegistry` (or use the default) and resolve the writer internally. Open Question 5 tracks the exact signature.

The DataFusion integration (`crates/integrations/datafusion/src/physical_plan/write.rs`) constructs `ParquetWriterBuilder` directly and passes it to `DataFileWriterBuilder`. It migrates to the registry-based constructor in the same PR.

**Breaking changes.**

- `DataFileWriterBuilder<B, L, F>` signature changes. Generic parameters are removed or replaced.
- `EqualityDeleteFileWriterBuilder<B, L, F>` signature changes. Same treatment.
- `TableScan::to_arrow` behavior changes to dispatch through the registry. Unsupported formats return `ErrorKind::FeatureUnsupported` instead of a Parquet deserialization error.

**PR structure.** One PR for the scan path. One PR for the write path. Each modifies existing code and includes migration of internal callers.

**Validation.** Existing tests pass. New tests verify that the registry-based path produces identical results to the direct path.

### Phase 3: Relocate format-specific types (breaking)

**Scope.** Move format-specific types to their canonical locations under `formats/`.

- `ParquetWriterBuilder` and `ParquetWriter` move from `iceberg::writer::file_writer` to `iceberg::formats::parquet`. A `#[deprecated]` re-export from the old path may be kept for one release or removed outright.
- `ArrowReaderBuilder` and `ArrowReader` move from `iceberg::arrow` to `iceberg::formats::parquet`, since they are Parquet-specific despite the generic name. A `#[deprecated]` re-export may be kept for one release.

With Phase 3 complete, new format implementations can land as their own PRs. Each new format is a separate PR that adds a feature flag, implements `FormatModel` inside the `formats/{format}/` module, registers with `FormatRegistry::default()`, and passes the TCK.

**Breaking changes.**

- `iceberg::writer::file_writer::ParquetWriterBuilder` path changes to `iceberg::formats::parquet::ParquetWriterBuilder`.
- `iceberg::arrow::ArrowReaderBuilder` path changes to `iceberg::formats::parquet::ArrowReaderBuilder`, possibly renamed to `ParquetReaderBuilder`.
- Direct construction of `ParquetWriterBuilder` for use with `DataFileWriterBuilder` is no longer the intended pattern. Callers use the registry instead.

**Validation.** The TCK is the gatekeeper. A new format must pass every applicable scenario or document its opt-outs.

### Phase 4: Clean up and harden

**Scope.** Audit remaining `DataFileFormat::Parquet` match arms in non-format code. Dispatch on the format enum outside the `formats/` module is eliminated or replaced with registry lookups. Add `#[non_exhaustive]` to `DataFileFormat` so future format variants can be added without a semver break each time. The attribute is added in Phase 4 rather than Phase 1 so that downstream consumers see all the API changes from Phases 2 and 3 before picking up the `#[non_exhaustive]` requirement. Remove any remaining `#[deprecated]` re-exports from Phases 2 and 3.

**Breaking changes.**

- `DataFileFormat` gains `#[non_exhaustive]`. Downstream `match` statements need a wildcard arm.
- Deprecated re-exports are removed.

**Validation.** CI enforces that no non-format code uses deprecated paths or dispatches on `DataFileFormat` directly.

### Phase summary

| Phase | Breaking? | Summary |
|-------|-----------|---------|
| Phase 1: Land the abstraction | No | Additive only |
| Phase 2: Migrate internal callers | Yes | `DataFileWriterBuilder` and `EqualityDeleteFileWriterBuilder` signatures change. Scan path dispatches through the registry. |
| Phase 3: Relocate format-specific types | Yes | Parquet types move to `iceberg::formats::parquet` |
| Phase 4: Clean up and harden | Yes (minor) | `DataFileFormat` gains `#[non_exhaustive]`. Deprecated re-exports removed. |

See the [Breaking Changes](#breaking-changes) section for per-item migration guidance.

## Breaking Changes

This section lists every breaking change introduced by this RFC, keyed to the phase that introduces it.

| # | What breaks | Phase | Why | Migration |
|---|-------------|-------|-----|-----------|
| 1 | `DataFileWriterBuilder<B: FileWriterBuilder, L, F>`. Generic parameters are removed or replaced. The `new(RollingFileWriterBuilder<B, L, F>)` constructor signature changes. | Phase 2 | The builder must resolve the file writer through the `FormatRegistry` rather than accepting a concrete `FileWriterBuilder` generic. This enables format-agnostic write orchestration. | Replace `DataFileWriterBuilder::new(rolling_writer_builder)` with the new registry-based constructor, for example `DataFileWriterBuilder::new_with_registry(registry, file_io, location_gen, file_name_gen)`. |
| 2 | `EqualityDeleteFileWriterBuilder<B: FileWriterBuilder, L, F>`. Same treatment as `DataFileWriterBuilder`. | Phase 2 | Same rationale. Format-agnostic delete file writing. | Same migration pattern as `DataFileWriterBuilder`. |
| 3 | `ParquetWriterBuilder` and `ParquetWriter`. Module path changes from `iceberg::writer::file_writer::ParquetWriterBuilder` to `iceberg::formats::parquet::ParquetWriterBuilder`. | Phase 3 | Co-locating all Parquet-specific types under `formats/parquet/` matches the module layout to the abstraction. | Update `use` statements. A `#[deprecated]` re-export from the old path may exist for one release. |
| 4 | `ArrowReaderBuilder` and `ArrowReader`. Module path changes from `iceberg::arrow::ArrowReaderBuilder` to `iceberg::formats::parquet::ArrowReaderBuilder`, possibly renamed to `ParquetReaderBuilder`. | Phase 3 | These types are Parquet-specific despite the generic name. Moving them clarifies the scope and leaves room for format-agnostic reader types at `iceberg::arrow` later. | Update `use` statements. A `#[deprecated]` re-export from the old path may exist for one release. |
| 5 | `DataFileFormat` enum gains `#[non_exhaustive]`. | Phase 4 | Allows adding new format variants (for example `Vortex`, `Lance`) in minor releases without a semver break each time. The enum has four variants (`Avro`, `Orc`, `Parquet`, `Puffin`) and is not `#[non_exhaustive]` today. | Add a wildcard arm (`_ => { ... }`) to any `match DataFileFormat` blocks. |
| 6 | `TableScan::to_arrow` behavior changes to dispatch through `FormatRegistry`. Signature unchanged. | Phase 2 | Enables reading non-Parquet data files. Unsupported formats return `ErrorKind::FeatureUnsupported` instead of a Parquet deserialization error. | No code change needed for callers using the default registry. Code that branches on the specific error type for non-Parquet files should update its error handling. |

Notes:

- Breaks 1 and 2 have the widest impact. They touch the DataFusion integration (`physical_plan/write.rs`, `task_writer.rs`) and any downstream code that constructs these builders directly.
- Breaks 3 and 4 are module path changes only. The types themselves are unchanged.
- Break 6 changes behavior, not signatures. Code using the default registry sees no compile-time impact.
- `IcebergWriter` and `IcebergWriterBuilder` in `writer/mod.rs` are format-agnostic today and do not change.

Open Question 7 discusses whether `FileWriterBuilder` and `FileWriter` gain an associated `Format` type or a `format()` method returning `DataFileFormat`. If that question resolves toward a trait-level change, a new row will be added to this table in a follow-up revision.

## Open Questions

These questions are intentionally unresolved and will be decided by the `apache/iceberg-rust` community during RFC review. Where the proposal has a preferred answer, that preference is noted as "Current lean" so reviewers can see what the RFC commits to if the question is not debated further.

### 1. Async trait methods: `BoxFuture` versus native `async fn`

This proposal uses `BoxFuture` for async methods on `dyn`-compatible traits. An alternative is to use native `async fn` in traits (stable since Rust 1.75) with the `async-trait` crate's `#[async_trait]` macro for `dyn` compatibility. The `async-trait` crate is already a dependency of the `iceberg` crate, used by `IcebergWriter` and `IcebergWriterBuilder`.

The tradeoff. `BoxFuture` is explicit and avoids a proc-macro expansion step for new code. It also makes the future's lifetime visible in the trait signature, which matters when the trait is used behind a `dyn` pointer where lifetime reasoning is less obvious. `#[async_trait]` is more ergonomic for implementors but hides the same information behind the macro. Both produce the same runtime behavior: heap-allocated futures.

**Current lean:** `BoxFuture` for the new traits.

**Question for the community:** Is the explicit choice worth the slight inconsistency with existing `async-trait`-using traits?

### 2. Generic versus fixed data type on `FormatModel`

This proposal fixes the data type to `RecordBatch` (see "No generic over the data type" under Design rationale for the full justification). An alternative is to make `FormatModel` generic over a data type `D: Send + 'static`. If a future format or engine integration brought a row type that cannot round-trip through Arrow, the generic approach would avoid a later breaking change.

**Current lean:** Fix the data type to `RecordBatch`.

**Question for the community:** Is `RecordBatch` the right call, or should the trait be generic from the start?

### 3. Registry bootstrapping: global default versus explicit construction

This proposal provides both a `OnceLock`-based global default (`default_format_registry()`) and an explicit constructor (`FormatRegistry::new()` plus `register()`). The question is which should be the primary API for scan and write orchestration to consume.

The tradeoff. `OnceLock` initializes exactly once and cannot be mutated afterward, which avoids the mutable-global issues of `lazy_static`. Explicit construction is more testable because each test builds its own registry, and it supports multiple registries in the same process.

**Current lean:** Offer both, with the global default as the default path for scan and write orchestration. Tests and embedded use cases construct their own registries.

**Question for the community:** Should `TableScan` use the global default, or should it require an explicit registry parameter?

### 4. Public API surface for the registry

The registry can be exposed in several shapes:

- **Free function:** `iceberg::formats::default_format_registry()` returns a `&'static FormatRegistry`. Simple and discoverable.
- **Method on a context type:** A new `IcebergContext` struct holds the registry, `FileIO`, and similar handles. More structured, but it adds a new concept to the public surface.
- **Builder pattern:** `FormatRegistryBuilder::new().with_parquet().build()`. Explicit, but verbose for the common case where defaults suffice.

**Current lean:** Free function, with explicit construction available for custom cases.

**Question for the community:** Which API shape is preferred?

### 5. Shape of the `DataFileWriterBuilder` break

The migration plan commits to changing `DataFileWriterBuilder`'s public signature in Phase 2. The open question is how to change it. Three candidate shapes:

- **Remove generic parameters entirely.** The builder uses the registry internally and is no longer generic over `FileWriterBuilder`. Cleanest API, but it removes the ability to use a custom `FileWriterBuilder` without going through the registry.
- **Replace the `B: FileWriterBuilder` generic with a `FormatModel` bound.** The builder is still generic, but over the format abstraction rather than the low-level writer. Preserves static dispatch for callers who know their format at compile time.
- **Keep the existing generic constructor as `pub(crate)` and add a new public constructor.** Internal code, tests, and the DataFusion integration can still use the generic path. External callers use the registry path.

**Current lean:** Remove generic parameters entirely. The registry covers every current use case for constructing a data file writer. If a future use case needs a custom `FileWriterBuilder` that bypasses the registry, a narrower advanced-users API can be added without reintroducing generics on the primary builder.

**Question for the community:** Which approach best balances API cleanliness with flexibility?

### 6. Acceptable divergence from the Java API

Java's `FormatModel<D, S>` uses two generic parameters. This proposal uses zero. Java's registry key is `(FileFormat, Class<?>)`. This proposal's key is `DataFileFormat` alone. These are deliberate simplifications based on iceberg-rust's single-data-type ecosystem (see "RecordBatch as the canonical data type" under Design rationale). The Rust API is not a one-to-one port of the Java API.

**Current lean:** Accept the divergence. Java's generics exist to serve engine-native row types that do not exist in the Rust ecosystem.

**Question for the community:** Is the divergence acceptable, or should the Rust API mirror Java more closely for cross-project consistency?

### 7. Associated `Format` type on `FileWriterBuilder` and `FileWriter`

The write orchestration needs to know which format a `FileWriter` produces, so that `DataFileBuilder` metadata is set correctly. Two approaches:

- **Option A: Add an associated type.** `type Format: Into<DataFileFormat>` on `FileWriterBuilder`, making each writer explicitly format-aware.
- **Option B: Keep the traits format-agnostic.** Let the `FormatRegistry` glue handle format identification externally, so that `FileWriter` implementations remain decoupled from the format enum.

**Current lean:** Option B. The registry already knows which format it dispatched to, so it can set the metadata field without the writer needing to declare its format. This preserves compatibility with existing implementors of `FileWriterBuilder` and `FileWriter`, and avoids a trait-level breaking change.

**Question for the community:** Is Option B correct, or does a future use case (for example per-writer configuration that varies by format) require Option A?

## Alternatives Considered

### Runtime library loading

Format implementations as separate shared libraries loaded via `libloading` or similar. Rejected because Rust has no stable ABI. Implementations must be compiled with the exact same compiler version, target, and feature flags as the host. Every use needs `unsafe`. No other crate in the iceberg-rust ecosystem uses this pattern, and PyIceberg has flagged runtime loading as an open question without landing a mechanism.

### Auto-registration via `inventory`

The `inventory` crate collects items at link time through platform-specific linker sections. Each format implementation would register itself with an `inventory::submit!` macro invocation. Rejected because items can be stripped silently at library boundaries, the pattern is hostile to analysis tools that do not run a real linker, and it registers formats that the caller did not explicitly depend on. Feature flags are explicit and work across `cargo check`, `cargo miri`, and cross-compilation. `datafusion`, `opendal`, and `reqwest` use the same feature-flag pattern.

### Non-breaking additive-only rollout

Add the `FormatModel` traits and registry alongside the existing API. Do not change `DataFileWriterBuilder` or `ArrowReaderBuilder`. Leave both paths available indefinitely. Rejected because this leaves two paths to do the same thing, retains the `ArrowReader` name on a Parquet-specific type, and keeps the `B: FileWriterBuilder` generic parameter on `DataFileWriterBuilder` that the rest of this RFC treats as format-agnostic orchestration. iceberg-rust is pre-1.0. The cost of a clean break is lowest now.

## References

### Apache Iceberg (Java)

- [PR #12774](https://github.com/apache/iceberg/pull/12774): Core File Format API interfaces
- [PR #15253](https://github.com/apache/iceberg/pull/15253): ParquetFormatModel
- [PR #15254](https://github.com/apache/iceberg/pull/15254): AvroFormatModel
- [PR #15255](https://github.com/apache/iceberg/pull/15255): ORCFormatModel
- [PR #15258](https://github.com/apache/iceberg/pull/15258): ArrowFormatModel
- [PR #15328](https://github.com/apache/iceberg/pull/15328): Spark migration
- [PR #15329](https://github.com/apache/iceberg/pull/15329): Flink migration
- [PR #15441](https://github.com/apache/iceberg/pull/15441): TCK
- [Issue #15415](https://github.com/apache/iceberg/issues/15415): TCK tracking (open)
- [Issue #15416](https://github.com/apache/iceberg/issues/15416): Vortex format (open)
- [PR #15751](https://github.com/apache/iceberg/pull/15751): Lance format (draft)
- [File Format API announcement, Feb 2026](https://iceberg.apache.org/blog/apache-iceberg-file-format-api/)

### Apache Iceberg (Python)

- [Issue #3100](https://github.com/apache/iceberg-python/issues/3100): File Format API proposal
- [PR #3119](https://github.com/apache/iceberg-python/pull/3119): File Format API implementation
- [PR #790](https://github.com/apache/iceberg-python/pull/790): ORC read/write (closed, stale)
- [PR #2236](https://github.com/apache/iceberg-python/pull/2236): ORC read/write (closed, stale)

## Conclusion

This RFC adds a `FormatModel` trait, read and write builders, and a `FormatRegistry` to the `iceberg` crate, with a Parquet implementation landing on the same API. The migration is phased across four PR series: Phase 1 is additive, Phases 2 through 4 are breaking, and every break is enumerated in the Breaking Changes table with per-item migration guidance. The abstraction removes hard-coded Parquet from the scan and write orchestration and makes ORC, Avro data-file, and other formats follow-up RFCs on the same API.
