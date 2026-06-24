# Time series

The `time-series` module provides a model for indexed numeric and textual series — the data structure PowSyBl uses to represent the time-dependent inputs and outputs of simulations (chronicles of load, generation, prices, …). It is split into two Maven submodules: `time-series-api`, which holds the model and lives under `com.powsybl.timeseries`, and `time-series-dsl`, a Groovy DSL that builds *calculated* series from user scripts. The module has no dependency on the grid model.

This page is part of the [foundation layer](../foundation.md).

## Package structure

| Package | Content |
|---------|---------|
| `com.powsybl.timeseries` | the time-series model: `TimeSeries`, the index and data-chunk types, the stores, `CalculatedTimeSeries`, the big-buffer helpers |
| `com.powsybl.timeseries.ast` | the calculated-series expression tree (`NodeCalc`) and its visitors |
| `com.powsybl.timeseries.json` | Jackson (de)serializers and the `TimeSeriesJsonModule` |
| `com.powsybl.timeseries.dsl` (in `time-series-dsl`) | the Groovy DSL loader implementing the `CalculatedTimeSeriesDslLoader` SPI |

## The time series model

`TimeSeries<P, T>` is the root interface. It is generic and self-referential — `T extends TimeSeries<P, T>` — and `Iterable<P>` over points of type `P extends AbstractPoint`. A series exposes its `TimeSeriesMetadata`, can be re-indexed with `synchronize(TimeSeriesIndex)`, and iterated either as a `stream()` (uncompressed: one element per index point) or a compressed stream. Static helpers on the interface centralize JSON and CSV read/write, and the nested `TimeFormat` / `DEFAULT_VERSION_NUMBER_FOR_UNVERSIONED_TIMESERIES` constant cover formatting and versioning concerns.

Two value types specialize it:

- `DoubleTimeSeries` (over `DoublePoint`), whose stored form is `StoredDoubleTimeSeries`;
- `StringTimeSeries` (over `StringPoint`), which is both the interface and its own stored implementation.

Both extend `AbstractTimeSeries<P, C extends DataChunk<P, C>, T>`, which holds the `TimeSeriesMetadata` and the list of data chunks and implements the iteration/streaming logic. Each series therefore carries a `TimeSeriesMetadata` (name, `TimeSeriesDataType`, index) and a `TimeSeriesIndex`.

### The time axis

`TimeSeriesIndex` is an `Iterable<Instant>` abstracting the time axis. Its core operations are `getPointCount()`, `getInstantAt(int point)` (the long-based `getTimeAt` is deprecated since 6.7.0), `getType()`, and JSON export (with a `MILLISECONDS` / `NANOSECONDS` `ExportFormat`). Three implementations, all sharing `AbstractTimeSeriesIndex`, cover the cases:

| Index | Meaning |
|-------|---------|
| `RegularTimeSeriesIndex` | evenly spaced instants, described by start, end and step |
| `IrregularTimeSeriesIndex` | an explicit list of instants |
| `InfiniteTimeSeriesIndex` | a single-instance, unbounded index (used by calculated series before they are bound to a concrete index) |

### Data storage and chunking

Point values are not stored as flat arrays but as a list of `DataChunk`s — `DoubleDataChunk` and `StringDataChunk` — each chunk covering a contiguous offset range. Chunks come in two forms: uncompressed (`UncompressedDoubleDataChunk` / `UncompressedStringDataChunk`) and run-length compressed (`CompressedDoubleDataChunk` / `CompressedStringDataChunk`, on the `AbstractCompressedDataChunk` base). Compression keeps long series with constant stretches compact, and a chunk can convert between forms. For very large series, `BigDoubleBuffer`, `BigStringBuffer` and `CompactStringBuffer` page over several backing buffers to work around the JVM's 2 GB single-array limit.

### Reading and tabulating series

`ReadOnlyTimeSeriesStore` is the lookup interface: `getTimeSeriesNames(TimeSeriesFilter)`, `timeSeriesExists`, metadata accessors, the set of available data versions, and typed getters such as `getDoubleTimeSeries(String name, int version)` and `getStringTimeSeries(...)`. It also accepts `TimeSeriesStoreListener`s for change notification. Implementations compose: `ReadOnlyTimeSeriesStoreCache` wraps a fixed set of series and `ReadOnlyTimeSeriesStoreAggregator` chains several stores. `TimeSeriesTable` offers a columnar, multi-version view suitable for tabular export, while `TimeSeriesFilter` and `TimeSeriesVersions` support filtering and versioning. JSON support is concentrated in `com.powsybl.timeseries.json` and the `TimeSeriesJsonModule`.

## Calculated time series (the `NodeCalc` AST)

`CalculatedTimeSeries` is a `DoubleTimeSeries` whose values come from an *expression* rather than stored data. The expression is an abstract syntax tree of `NodeCalc` nodes (package `com.powsybl.timeseries.ast`). A `CalculatedTimeSeries` holds its name, the root `NodeCalc`, and a `TimeSeriesNameResolver` used to bind referenced series; its index stays `InfiniteTimeSeriesIndex` until resolved against real data.

The node types fall into three groups:

- **literals** — `IntegerNodeCalc`, `FloatNodeCalc`, `DoubleNodeCalc`, `BigDecimalNodeCalc` (sharing `LiteralNodeCalc`);
- **references** — `TimeSeriesNameNodeCalc` (a named series), `TimeSeriesNumNodeCalc` (a series by number) and `TimeNodeCalc` (the current instant);
- **operations** — `BinaryOperation` (an `Operator` enum: `PLUS`, `MINUS`, `MULTIPLY`, `DIVIDE`, the comparisons `LESS_THAN`/`GREATER_THAN` and their `…_OR_EQUALS_TO` forms), `UnaryOperation` (`ABS`, `NEGATIVE`, `POSITIVE`), the min/max operators in both their unary-with-constant form (`MinNodeCalc` / `MaxNodeCalc`) and their two-operand form (`BinaryMinCalc` / `BinaryMaxCalc`), and `CachedNodeCalc` which memoizes a subtree. Shared bases include `AbstractBinaryNodeCalc`, `AbstractSingleChildNodeCalc`, `AbstractMinMaxNodeCalc` and `AbstractBinaryMinMax`.

Each node knows how to serialize itself to JSON (`writeJson`); the static `NodeCalc.parseJson` dispatches on a per-type field name (`IntegerNodeCalc.NAME`, `BinaryOperation.NAME`, …) to rebuild the tree.

### The visitor and its hybrid traversal

`NodeCalc` is processed with the **visitor** pattern through `NodeCalcVisitor<R, A>` — a visitor producing a result of type `R` and threading an argument of type `A`. The interface has a `visit(...)` method per node type (receiving the already-computed children results, e.g. `visit(BinaryOperation, A, R left, R right)`) and `iterate(...)` methods that describe which children to descend into and in what order.

The traversal itself is deliberately *hybrid*, for performance. As the `NodeCalc` javadoc explains, plain recursion is roughly 5× faster than an explicit-stack walk, but deep trees overflow the JVM stack. So each node implements three methods — `accept(visitor, arg, depth)`, `acceptIterate(...)` and `acceptHandle(...)`: `accept` recurses directly while the depth stays below a threshold, then hands off to `NodeCalcVisitors.visit(root, arg, visitor)`, which performs an iterative post-order traversal using two `ArrayDeque` stacks (a pre-order stack feeding a post-order stack, with a sentinel `NULL` object because `ArrayDeque` forbids `null`). The recursion threshold defaults to 1000 and is configurable via the `timeseries` module's `recursion-threshold` property.

`DefaultNodeCalcVisitor` is the do-nothing base; the concrete visitors implement the operations on the tree:

| Visitor | Role |
|---------|------|
| `NodeCalcEvaluator` | evaluate the tree to a numeric value |
| `NodeCalcResolver` | bind `TimeSeriesNameNodeCalc` references to numbers/series |
| `NodeCalcSimplifier` | constant-fold and simplify |
| `NodeCalcCloner` | deep-copy a tree |
| `NodeCalcPrinter` | render to a readable string |
| `NodeCalcDuplicateDetector` | find shared subtrees (used together with `NodeCalcCacheCreator` to insert `CachedNodeCalc` nodes) |

The names referenced by a tree are collected with `TimeSeriesNames`, and unresolved series are looked up through the `TimeSeriesNameResolver` SPI — its methods `getTimeSeriesMetadata`, `getTimeSeriesDataVersions` and `getDoubleTimeSeries` are typically backed by `FromStoreTimeSeriesNameResolver`, which adapts a `ReadOnlyTimeSeriesStore`.

## Building calculated series from scripts

`CalculatedTimeSeriesDslLoader` is the SPI that turns a script into a map of name → `NodeCalc`:

```java
Map<String, NodeCalc> load(String script, ReadOnlyTimeSeriesStore store);

static CalculatedTimeSeriesDslLoader find();
```

`find()` resolves the single implementation on the classpath through `ServiceLoader` (failing if none or several are present). The `time-series-dsl` submodule provides that implementation: a Groovy DSL (`CalculatedTimeSeriesGroovyDslLoader`) whose `NodeCalcGroovyExtensionModule` and AST transformation let scripts write arithmetic over series names directly, producing `NodeCalc` trees that feed `CalculatedTimeSeries`.
