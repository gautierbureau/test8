# Foundation layer

The foundation layer gathers the cross-cutting modules that every other part of PowSyBl Core builds on: configuration, data sources, reporting, the extension framework, graph and math helpers, time series, the computation abstraction and the iTools command framework. None of these modules know anything about the grid model; they provide generic services that the upper layers reuse.

The modules covered here are `commons`, `math`, `time-series`, `tools`, `computation`, `computation-local` and `config-classic`.

```{toctree}
:hidden:
foundation/commons
foundation/math
foundation/time-series
foundation/tools
foundation/computation
foundation/config-classic
```

## `commons`

`commons` is the lowest-level module and the most widely depended-on. Its code lives under `com.powsybl.commons`, with one sub-package per concern. The root package only holds two very general types: the unchecked `PowsyblException` (the base exception used throughout the project) and the `Versionable` interface.

### Configuration framework

Package `com.powsybl.commons.config`. Configuration is exposed through `PlatformConfig`, the entry point read by every configurable feature. A `PlatformConfig` is backed by a `ModuleConfigRepository`, a simple interface whose single method `getModuleConfig(String name)` returns an `Optional<ModuleConfig>` — one named section of the configuration. `ModuleConfig` is the typed accessor for the properties of a section (`getStringProperty`, `getOptionalStringListProperty`, enum/path/numeric variants, etc.).

The framework is deliberately source-agnostic. Several repository implementations read the same `ModuleConfig` abstraction from different backends:

| Repository | Source |
|------------|--------|
| `YamlModuleConfigRepository`, `XmlModuleConfigRepository`, `PropertiesModuleConfigRepository` | YAML, XML or `.properties` files |
| `EnvironmentModuleConfigRepository` | environment variables |
| `InMemoryModuleConfigRepository` | in-memory map (mostly for tests) |
| `StackedModuleConfigRepository` | several repositories layered on top of each other |

`StackedModuleConfigRepository` (and the matching `StackedModuleConfig`) implements the *stacking* behaviour described in the user-facing configuration documentation: a property is looked up in the first repository that defines it. Abstract bases `AbstractModuleConfig`, `AbstractMapModuleConfig` and `AbstractModuleConfigRepository` factor the common logic, and `PlatformConfig.loadModuleRepository(...)` chooses YAML, then XML, then properties depending on which file exists.

Which `PlatformConfig` is returned by `PlatformConfig.defaultConfig()` is itself pluggable: the static method uses `ServiceLoader` to find a single `PlatformConfigProvider` implementation on the classpath. This is the SPI implemented by the `config-classic` module (see below). `InMemoryPlatformConfig` provides a test-friendly implementation, and `ComponentDefaultConfig` offers a small registry to resolve default implementation classes from configuration.

### Data sources

Package `com.powsybl.commons.datasource`. A `DataSource` abstracts the physical storage that importers and exporters read from and write to, decoupling format code from whether the data lives in a plain directory, an archive or memory. The read side is split out into `ReadOnlyDataSource` (base-name access, `exists(...)`, `newInputStream(...)`, listing) and `DataSource` adds `newOutputStream(...)` for writing. Files are addressed by `<basename><suffix>.<ext>` so a single logical data set can span several physical files.

The implementations form a small hierarchy around `AbstractFileSystemDataSource` and `AbstractArchiveDataSource`:

- `DirectoryDataSource` and its compressed flavours `GzDirectoryDataSource`, `Bzip2DirectoryDataSource`, `XZDirectoryDataSource`, `ZstdDirectoryDataSource` — one directory holding several files;
- `ZipArchiveDataSource`, `TarArchiveDataSource` — all files inside one archive;
- `MemDataSource` / `ReadOnlyMemDataSource` — in-memory storage;
- `ResourceDataSource` with `ResourceSet` — read-only access to classpath resources, used heavily in tests.

`DataSourceUtil` and the package-private `DataSourceBuilder` create the right implementation from a path, guided by `CompressionFormat`, `ArchiveFormat` and `FileInformation` (which guesses the base name and extensions). The *observer* pattern is supported through `DataSourceObserver` (with `DefaultDataSourceObserver`) and the `ObservableInputStream` / `ObservableOutputStream` wrappers, so callers can be notified of opened streams. `MultipleReadOnlyDataSource` and `GenericReadOnlyDataSource` compose several sources.

### Reporting framework

Package `com.powsybl.commons.report`. `ReportNode` is the structured, localizable functional-log abstraction used during imports, exports and simulations. A `ReportNode` is a tree node carrying a message *key*, a default message template, a map of `TypedValue` indexed by key, and a collection of child `ReportNode`s. Templates reference values with the `${key}` syntax (substituted with `StringSubstitutor`), and values are inherited by descendants, which makes the tree both hierarchical and translatable.

The framework uses a builder/adder pattern. A root node is created through `ReportNodeBuilder` (`ReportNodeRootBuilderImpl`), and children through `ReportNodeAdder` (`ReportNodeChildAdderImpl`); both share `ReportNodeAdderOrBuilder` and the `AbstractReportNodeAdderOrBuilder` base. The concrete node is `ReportNodeImpl`. A null-object `ReportNodeNoOp` (exposed as the constant `ReportNode.NO_OP`) lets callers report unconditionally without null checks, with `TreeContextNoOp` as its companion no-op `TreeContext` (the `TreeContextImpl` holds the shared dictionary of templates and the tree-wide state).

`TypedValue` wraps a value with a semantic type — `SEVERITY`, `ACTIVE_POWER`, `VOLTAGE`, `FILENAME`, `TIMESTAMP`, etc. — and predefined severity constants such as `TRACE_SEVERITY`. Message templates are resolved through the `MessageTemplateProvider` SPI (`BundleMessageTemplateProvider`, `MultiBundleMessageTemplateProvider`, `EmptyMessageTemplateProvider`, plus the `ReportResourceBundle` abstraction and `PowsyblCoreReportResourceBundle`), which is what enables multilingual reports from resource bundles. JSON serialization is handled by `ReportNodeSerializer` / `ReportNodeDeserializer`, wired through the Jackson module `ReportNodeJsonModule`, with `ReportNodeVersion` tracking the serialization format version.

### Extension framework

Package `com.powsybl.commons.extensions`. This is the generic machinery that lets any object be enriched with typed data without touching its interface; the IIDM model is its primary client. An `Extension<T>` is a piece of data attached to a holder of type `T` (`getExtendable()` / `setExtendable(...)`), and `Extendable<T>` is the holder side. `AbstractExtendable<T>` provides the default implementation, keying extensions both by their interface class (`Map<Class<?>, Extension<T>>`) and by their name, and calling `Extension.cleanup()` when one is replaced or removed.

Extensions are created through the builder pattern: `ExtensionAdder<T, E>` builds an extension and adds it to its extendable, returning the extendable for fluent chaining. `AbstractExtension` and `AbstractExtensionAdder` are the usual bases for implementations.

Discovery is done through the plugin mechanism. `ExtensionProvider` is the common SPI super-type; `ExtensionAdderProvider` supplies adders and `ExtensionSerDe` / `ExtensionJsonSerializer` (with `AbstractExtensionSerDe`) supply the (de)serializers. `ExtensionProviders<T>` is a typed registry that loads all providers of a given SPI through an `ExtensionProvidersLoader` (default `DefaultExtensionProvidersLoader`, backed by `ServiceLoader`) and indexes them by extension name. `ExtensionConfigLoader` allows configuring extensions from `PlatformConfig`.

### JSON, tree-data IO and table formatting

Package `com.powsybl.commons.json` centralizes Jackson usage: `JsonUtil` provides helpers to read and write JSON with a consistent configuration, complemented by the streaming `JsonReader` / `JsonWriter`.

Package `com.powsybl.commons.io` generalizes serialization beyond JSON. `TreeDataWriter` and `TreeDataReader` (with `AbstractTreeDataWriter` / `AbstractTreeDataReader`, `SerializerContext` / `DeserializerContext`, `TreeDataHeader`) define a format-independent tree-document model. The `TreeDataFormat` enum — `XML`, `JSON`, `BIN` — is what lets IIDM serialization target XML, JSON or a binary form through the same writer/reader abstraction. `WorkingDirectory` and `FileUtil` are filesystem helpers.

Package `com.powsybl.commons.io.table` offers a small table-rendering toolkit: the `TableFormatter` interface (an `AutoCloseable`) with `AsciiTableFormatter` and `CsvTableFormatter` implementations, created via matching `TableFormatterFactory` instances, configured through `TableFormatterConfig` and described with `Column` / `HorizontalAlignment`.

### Parameters

Package `com.powsybl.commons.parameters`. `Parameter` describes a configurable parameter (names/aliases, `ParameterType`, description, default value, possible values, `ParameterScope`, category key). This is the model used by importers, exporters and tools to declare their parameters uniformly; `ParameterDefaultValueConfig` reads defaults from `PlatformConfig` and `ConfiguredParameter` binds a parameter to its resolved value.

### Plugin / PluginInfo mechanism

Package `com.powsybl.commons.plugins`. `PluginInfo<T>` describes a category of plugin (its SPI class and a human-readable plugin name, e.g. importer or exporter) and knows how to derive an id for a given implementation. `Plugins` is a utility that enumerates all `PluginInfo` registered through `ServiceLoader` (cached with `ServiceLoaderCache`), and lists the implementation ids available for each plugin category. This is what the `PluginsInfoTool` in the `tools` module surfaces on the command line.

The `com.powsybl.commons.util` package backs the rest of the framework with `ServiceLoaderCache` (a cached `ServiceLoader` used everywhere the SPI pattern is applied), the `StringToIntMapper` / `StringAnonymizer` helpers and `WeakListenerList`.

## `math`

`math`, under `com.powsybl.math`, collects numerical and graph utilities that have no dependency on the grid model.

### Graph utilities

Package `com.powsybl.math.graph`. `UndirectedGraph<V, E>` is a generic undirected graph carrying a value of type `V` on each vertex and `E` on each edge, with `UndirectedGraphImpl` as the implementation. Indices of vertices and edges are not guaranteed to be contiguous, which is why iteration goes through `getVertices()` / `getEdges()`. Graph mutations notify registered `UndirectedGraphListener`s (with `DefaultUndirectedGraphListener` as a no-op base) — an *observer* pattern reused by the IIDM bus/breaker topology code. Traversal uses the *visitor*/callback pattern: `Traverser` is the callback, returning a `TraverseResult` (`CONTINUE` / `TERMINATE`) and parameterized by `TraversalType` (depth- or breadth-first). `GraphUtil` provides connected-component and decomposition helpers.

### Matrix and solvers

Package `com.powsybl.math.matrix` defines a matrix abstraction generic enough for both dense and sparse storage. `Matrix` is the interface (with the `AbstractMatrix` base), implemented by `DenseMatrix` and `SparseMatrix`. Instances are created through the `MatrixFactory` *abstract factory* (`DenseMatrixFactory`, `SparseMatrixFactory`), so algorithms can stay agnostic of the underlying representation. LU factorization follows the same pattern with the `LUDecomposition` interface and its `DenseLUDecomposition` / `SparseLUDecomposition` implementations. `ComplexMatrix` adds complex-valued support, `MatrixException` is the dedicated error type, and the `serializer` sub-package holds `SparseMatrixMatSerializer` (MATLAB `.mat` export).

Package `com.powsybl.math.solver` wraps the native KINSOL nonlinear solver (SUNDIALS) through `Kinsol`, with `KinsolParameters`, `KinsolContext`, `KinsolResult`, `KinsolStatus` and `KinsolException`. `Kinsol` takes a `FunctionUpdater` and a `JacobianUpdater` (filling a `SparseMatrix`) supplied by the caller. The `casting` package contains the `Double2Float` helper.

## `time-series`

The `time-series` module is split into `time-series-api` (the model) and `time-series-dsl` (a Groovy DSL to build calculated series). The model lives in `com.powsybl.timeseries`.

### Time series model

`TimeSeries<P, T>` is the generic, self-referential interface (`T extends TimeSeries<P, T>`) iterating over points of type `P`. The two concrete value types are `DoubleTimeSeries` (over `DoublePoint`) and `StringTimeSeries` (over `StringPoint`), both backed by `StoredDoubleTimeSeries` / `StringTimeSeries` and the `AbstractTimeSeries` base. Each series carries a `TimeSeriesMetadata` and a `TimeSeriesIndex`.

`TimeSeriesIndex` (an `Iterable<Instant>`) abstracts the time axis: `RegularTimeSeriesIndex` (evenly spaced), `IrregularTimeSeriesIndex` (explicit instants) and `InfiniteTimeSeriesIndex`, all sharing `AbstractTimeSeriesIndex`. Data is stored in `DataChunk`s — `DoubleDataChunk` and `StringDataChunk` — which come in uncompressed (`UncompressedDoubleDataChunk`) and run-length-compressed (`CompressedDoubleDataChunk`) forms; this chunking keeps long series with constant stretches compact. Large buffers are handled by `BigDoubleBuffer`, `BigStringBuffer` and `CompactStringBuffer` to work around the 2 GB single-array limit.

Series are read through the `ReadOnlyTimeSeriesStore` interface (lookup by name and version, metadata access), with `ReadOnlyTimeSeriesStoreCache` and `ReadOnlyTimeSeriesStoreAggregator` as composable implementations and `TimeSeriesStoreListener` for change notifications. `TimeSeriesTable` provides a columnar, multi-version view. `TimeSeriesFilter` and `TimeSeriesVersions` support filtering and versioning, and `TimeSeries` itself exposes static JSON read/write helpers (with the `json` sub-package).

### Calculated time series (AST)

`CalculatedTimeSeries` (a `DoubleTimeSeries`) is a series defined by an expression rather than stored data. The expression is an abstract syntax tree of `NodeCalc` nodes in package `com.powsybl.timeseries.ast`: literals (`DoubleNodeCalc`, `IntegerNodeCalc`, `FloatNodeCalc`, `BigDecimalNodeCalc`), references (`TimeSeriesNameNodeCalc`, `TimeSeriesNumNodeCalc`, `TimeNodeCalc`) and operations (`BinaryOperation`, `UnaryOperation`, `MinNodeCalc` / `MaxNodeCalc` and their binary forms, `CachedNodeCalc`).

The tree is processed with the *visitor* pattern through `NodeCalcVisitor` (default `DefaultNodeCalcVisitor`), driven by `NodeCalcVisitors`. As its javadoc explains, the visit is a hybrid recursive/iterative algorithm: it recurses up to a stack-depth limit (recursion being markedly faster) and then switches to an explicit-stack iteration to avoid `StackOverflowError` on deep trees. Concrete visitors include `NodeCalcEvaluator` (evaluate), `NodeCalcResolver` (bind names), `NodeCalcSimplifier`, `NodeCalcCloner`, `NodeCalcPrinter` and `NodeCalcDuplicateDetector`. The names referenced by a tree are collected via `TimeSeriesNames`, and unresolved series are looked up through the `TimeSeriesNameResolver` SPI (e.g. `FromStoreTimeSeriesNameResolver`). `CalculatedTimeSeriesDslLoader` is the SPI used by `time-series-dsl` to build these trees from Groovy scripts.

## `tools`

`tools`, under `com.powsybl.tools`, implements the iTools command-line framework. A command is an implementation of the `Tool` SPI: it returns a `Command` (its metadata) and implements `run(CommandLine line, ToolRunningContext context)`. `Command` describes the command's `name`, `theme` (used to group commands in the help), `description`, Apache Commons CLI `Options` and usage footer.

`Tool` implementations are discovered through `ServiceLoader` (so a new command is added simply by putting an `@AutoService(Tool.class)`-annotated jar on the classpath). `CommandLineTools` is the dispatcher: it loads all `Tool`s, parses the arguments, selects the matching tool and runs it, returning status codes (`COMMAND_OK_STATUS`, `COMMAND_NOT_FOUND_STATUS`, `INVALID_COMMAND_STATUS`, `EXECUTION_ERROR_STATUS`). `Main` is the `itools` entry point; it builds a `ToolInitializationContext` from `System.out`/`System.err` and a `DefaultComputationManagerConfig`.

Execution context is split into two objects. `ToolInitializationContext` carries what is needed to set up a run, while `ToolRunningContext` is what each tool receives: output and error `PrintStream`s, a `FileSystem`, and two `ComputationManager`s — one for short-time and one for long-time executions (the long one falls back to the short one when unset). `ToolOptions` wraps the parsed `CommandLine` with typed accessors, and `CommandLineUtil` collects parsing helpers.

Two built-in tools ship in the module: `VersionTool` (built on `AbstractVersion` / `Version`) and `PluginsInfoTool` (which prints the plugins discovered via `commons`' `Plugins`). The `com.powsybl.tools.autocompletion` sub-package generates Bash completion scripts: `BashCompletionTool` (itself a `Tool`) drives a `BashCompletionGenerator` (default `StringTemplateBashCompletionGenerator`) that turns the registered commands and their options (`BashCommand`, `BashOption`, `OptionType`, `OptionTypeMapper`) into a completion script.

## `computation`

`computation`, under `com.powsybl.computation`, provides the abstraction for running external computations independently of where they run. `ComputationManager` is the central interface (an `AutoCloseable`): its `execute(...)` methods submit work described by an `ExecutionHandler<R>` within an `ExecutionEnvironment` and return a `CompletableFuture<R>`, so execution is asynchronous and cancellable. Implementations provide a temporary per-execution working directory (kept when `ExecutionEnvironment.isDebug()` is true).

`ExecutionHandler<R>` is the *template* for a job: `before(Path workingDir)` performs preprocessing and returns the list of `CommandExecution`s to run, and `after(Path, ExecutionReport)` performs postprocessing and produces the business result of type `R`. `AbstractExecutionHandler` is the usual base. Progress can be observed through `ExecutionListener` (default `DefaultExecutionListener`), and failures are reported with `ExecutionError`, `ExecutionReport` (`DefaultExecutionReport`) and `ComputationException` (built via `ComputationExceptionBuilder`).

Commands are modelled separately from their execution. The `Command` interface (`getId`, `getType`, input/output files) has two kinds given by the `CommandType` enum — `SIMPLE` and `GROUP` — implemented by `SimpleCommandImpl` and `GroupCommandImpl` and built with the *builder* pattern (`SimpleCommandBuilder`, `GroupCommandBuilder`, on the `AbstractCommandBuilder` / `AbstractCommand` bases). Input and output files (`InputFile`, `OutputFile`) carry a `FilePreProcessor` / `FilePostProcessor` (e.g. archive extraction) and use `FileName` strategies (`StringFileName`, `FunctionFileName`) so file names can depend on the execution number. `Partition` splits a workload across parallel executions.

Which `ComputationManager` is used by default is itself configurable: `ComputationManagerFactory` is the SPI, and `DefaultComputationManagerConfig` reads from `PlatformConfig` which factory to use for short- and long-time managers. `LazyCreatedComputationManager` defers creation until first use, and `ComputationParameters` (with its builder) carries optional technical parameters such as time-outs.

## `computation-local`

`computation-local`, under `com.powsybl.computation.local`, is the default `ComputationManager` implementation: it runs commands as local OS processes. `LocalComputationManager` implements `ComputationManager`, and `LocalComputationManagerFactory` is its `@AutoService`-registered `ComputationManagerFactory` — putting this module on the classpath is what makes computations run locally. Configuration (local working directory, available processor count) comes from `LocalComputationConfig`, and `LocalComputationResourcesStatus` reports the `ComputationResourcesStatus`.

Process execution is abstracted by the `LocalCommandExecutor` interface (`execute(...)`, `stop`, `stopForcibly`), with `AbstractLocalCommandExecutor` and platform-specific `UnixLocalCommandExecutor` / `WindowsLocalCommandExecutor` implementations and a `ProcessHelper` utility — a *strategy* pattern selecting the right executor per operating system.

## `config-classic`

`config-classic`, under `com.powsybl.config.classic`, contains a single class: `ClassicPlatformConfigProvider`, the standard `@AutoService(PlatformConfigProvider.class)` implementation. It is the default `PlatformConfig` provider used by an iTools distribution. It locates configuration directories from the system properties `powsybl.config.dirs` or `itools.config.dir` (defaulting to `${HOME}/.itools`), resolving keywords from `commons`' `PlatformEnv` (such as `app.root`, `user.home`), reads YAML/XML/properties files from those directories, and layers an `EnvironmentModuleConfigRepository` on top so configuration can also be supplied through environment variables. Separating this provider into its own module lets an application substitute a different configuration strategy simply by replacing the jar on the classpath.
