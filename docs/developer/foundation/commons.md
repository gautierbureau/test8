# `commons`

`commons` is the lowest-level module of PowSyBl Core and the most widely depended-on: every other module sits on top of it. It carries no knowledge of the grid model — it provides the generic, cross-cutting services that the upper layers reuse: configuration, data sources, the structured reporting framework, the extension framework, the plugin registry, JSON / tree-data IO, parameter modelling and a handful of low-level utilities.

This page goes deeper than the [foundation layer overview](../foundation.md): it documents the main packages, their key types and the design patterns they embody. All code lives under `com.powsybl.commons`, one sub-package per concern. The root package holds only two very general types: the unchecked `PowsyblException` (the base exception used throughout the project) and the `Versionable` interface (an object that carries a name and a version).

## Configuration framework

Package `com.powsybl.commons.config`.

Configuration is the stacked YAML / XML / properties mechanism described in the user-facing [configuration](../../user/configuration/index.md) section. `commons` defines the *abstractions*; the concrete way the files are located and layered is supplied by a `PlatformConfigProvider` (the standard one lives in [`config-classic`](config-classic.md)).

### `PlatformConfig`, `ModuleConfigRepository`, `ModuleConfig`

Three types form the core:

- `ModuleConfigRepository` — the source-agnostic backend. A single method, `Optional<ModuleConfig> getModuleConfig(String name)`, returns one named section of the configuration.
- `ModuleConfig` — the typed accessor for the properties of a section. It exposes a uniform family of getters for every supported type, each in three flavours: optional (`getOptionalStringProperty`), required (`getStringProperty`, throwing when absent) and defaulted (`getStringProperty(name, defaultValue)`). Supported types include `String`, `List<String>`, `int`/`long`/`float`/`double`, `boolean`, enum and `EnumSet`, `Path` and `List<Path>`, `Class<? extends T>` and `ZonedDateTime`.
- `PlatformConfig` — the entry point read by every configurable feature. It is backed by a `ModuleConfigRepository` (wrapped in a memoizing `Supplier`) and an optional config directory, and offers convenience accessors that fold the "look up module, then property, else default" pattern into a single call (`getStringProperty(moduleName, propertyName, defaultValue)`, and so on for each type).

`PlatformConfig.defaultConfig()` returns the process-wide singleton. It is itself pluggable: the static method uses `ServiceLoader` to find exactly one `PlatformConfigProvider` on the classpath. Finding none yields an empty configuration (with a log hint to add `powsybl-config-classic` or `powsybl-config-test`); finding more than one is a fatal `PowsyblException`.

```java
PlatformConfig config = PlatformConfig.defaultConfig();
String impl = config.getStringProperty("load-flow", "default-impl-name", "OpenLoadFlow");
```

The `PlatformConfigProvider` SPI is tiny — `getName()` plus `getPlatformConfig()` — and its `getPlatformConfig()` is normally called once and cached for the whole application.

### Repository implementations and stacking

The framework is deliberately source-agnostic; several repositories read the same `ModuleConfig` abstraction from different backends:

| Repository | Source |
|------------|--------|
| `YamlModuleConfigRepository`, `XmlModuleConfigRepository`, `PropertiesModuleConfigRepository` | YAML, XML or `.properties` files |
| `EnvironmentModuleConfigRepository` | environment variables |
| `InMemoryModuleConfigRepository` | an in-memory map (mostly for tests) |
| `StackedModuleConfigRepository` | several repositories layered on top of each other |

`AbstractModuleConfigRepository` factors the common logic: it holds a `Map<String, MapModuleConfig>` and resolves `getModuleConfig` by lookup; the YAML / XML / properties repositories simply populate that map at construction. On the `ModuleConfig` side, `AbstractModuleConfig` and `AbstractMapModuleConfig` implement nearly all of the typed getters on top of a single abstract `getValue(String)` "mapping" method, so a new backend only has to expose name-to-value lookup. `MapModuleConfig` and `EnvironmentMapModuleConfig` are the concrete map-based configs.

`StackedModuleConfigRepository` implements the *stacking* behaviour: it holds an ordered list of repositories and, for a given module name, builds a `StackedModuleConfig` chaining every repository that defines that module — a property is resolved by the first repository in the list that defines it. This is what makes per-directory and environment-variable overrides work.

`EnvironmentModuleConfigRepository` deserves a note: it maps a property `property-name` in module `module-name` to the environment variable `MODULE_NAME__PROPERTY_NAME` (double underscore as separator), translating both hyphenated and camel-case names to upper-underscore form.

```{note}
`PlatformConfig.loadModuleRepository(configDir, configName)` is the helper that chooses a single-directory backend: it returns a `YamlModuleConfigRepository` if `<configName>.yml` exists, else an `XmlModuleConfigRepository` if `<configName>.xml` exists, else a `PropertiesModuleConfigRepository` reading `.properties` files from the directory.
```

String values are passed through `PlatformEnv.substitute(...)`, which expands `$HOME`, `${user.home}` / `${user_home}` and `${app.root}` — the keyword substitution used to make config portable across machines.

### Supporting config types

- `InMemoryPlatformConfig` and `InMemoryModuleConfigRepository` — a test-friendly, programmatically populated configuration.
- `ComponentDefaultConfig` — a small registry that resolves a *factory* implementation class from configuration. `findFactoryImplClass(factoryBaseClass)` reads the property named after the base class's simple name from the `componentDefaultConfig` module; `newFactoryImpl(...)` instantiates it (wrapping reflection exceptions as the `Unchecked*` types from `com.powsybl.commons.exceptions`).
- `PlatformConfigNamedProvider` — a richer provider-selection SPI than `PlatformConfigProvider`. Its inner `Finder` looks up, among the `ServiceLoader`-discovered providers of a given class, the one whose name matches a `default-impl-name` property in a module (or the sole provider when only one exists), throwing a helpful error when several are present and none is selected. This is the mechanism behind picking a default load-flow / security-analysis implementation by name.
- `BaseVoltagesConfig` / `BaseVoltageConfig` — a typed configuration object (loaded from a YAML resource) describing nominal-voltage ranges and profiles, used by diagram tooling.
- `ConfigurationException` — the dedicated unchecked exception for configuration errors.

## Data sources

Package `com.powsybl.commons.datasource`.

A `DataSource` abstracts the physical storage that importers and exporters read from and write to, decoupling format code from whether the data lives in a plain directory, a compressed file, an archive, memory or the classpath.

### The two interfaces and the naming scheme

The read side and the write side are split:

- `ReadOnlyDataSource` — `getBaseName()`, `exists(suffix, ext)` / `exists(fileName)`, `newInputStream(suffix, ext)` / `newInputStream(fileName)`, `listNames(regex)`, plus `getDataExtension()` / `isDataExtension(ext)`.
- `DataSource` extends it with the write side: `newOutputStream(fileName, append)` and `newOutputStream(suffix, ext, append)`.

Files are addressed either by a full file name or by the `<baseName><suffix>.<ext>` convention (`DataSourceUtil.getFileName(baseName, suffix, ext)`), so a single logical data set can span several physical files sharing a base name. The static `DataSource.fromPath(path)` builds a source from an existing file.

### Implementation hierarchy

Two package-private abstract bases anchor the hierarchy:

- `AbstractFileSystemDataSource` — a directory holding several files, with an optional `CompressionFormat`. Concrete subclasses: `DirectoryDataSource` (uncompressed) and the per-file-compressed `GzDirectoryDataSource`, `Bzip2DirectoryDataSource`, `XZDirectoryDataSource`, `ZstdDirectoryDataSource`, which wrap streams in the matching compressor.
- `AbstractArchiveDataSource` (itself extending the filesystem base) — all the files live inside one archive, with an abstract `entryExists(...)` hook. Concrete subclasses: `ZipArchiveDataSource` and `TarArchiveDataSource` (the latter supporting an inner compression format). Archive sources do not support `append`.

Other implementations:

- `ReadOnlyMemDataSource` / `MemDataSource` — in-memory storage backed by a `Map<String, byte[]>`; the writable form returns an output stream that stores its bytes on close.
- `ResourceDataSource` with `ResourceSet` — read-only access to classpath resources (a `ResourceSet` is a directory plus a list of resource names, validated to exist at construction), used heavily in tests.
- `MultipleReadOnlyDataSource` — composes several read-only sources and returns the first match.
- `GenericReadOnlyDataSource` — tries a fixed sequence of directory / compressed / archive sources against the same directory, so a caller can read a data set without knowing how it was stored.

### Building and observing

`FileInformation` parses a file name (e.g. `network.iidm.tar.gz`) into its base name, data extension, `ArchiveFormat` and `CompressionFormat`. The package-private `DataSourceBuilder` is the fluent builder that assembles the right implementation from those pieces (`withDirectory`, `withBaseName`, `withCompressionFormat`, `withArchiveFormat`, `withDataExtension`, `withObserver`, …, then `build()`); the `DataSourceUtil` interface exposes the static `createDataSource(...)` entry points that most callers use, plus filename and `OpenOption` helpers. `CompressionFormat` (`BZIP2`, `GZIP`, `XZ`, `ZIP`, `ZSTD`) and `ArchiveFormat` (`ZIP`, `TAR`) are the format enums.

The *observer* pattern lets callers be notified when streams are opened and closed: `DataSourceObserver` (with the no-op `DefaultDataSourceObserver`) defines `opened(streamName)` / `closed(streamName)`, and every implementation wraps its streams in `ObservableInputStream` / `ObservableOutputStream` so the callbacks fire automatically. `ReadOnlyDataSourceFactory` is an SPI for pluggable source creation.

## Reporting framework

Package `com.powsybl.commons.report`.

`ReportNode` is the structured, localizable functional-log abstraction used during imports, exports and simulations (see the user-facing [functional logs](../../user/functional_logs/index.md) page). Instead of free-text log lines, code emits a *tree* of nodes, each carrying a message *key*, a default message template and a set of typed values; the tree can then be rendered in any locale or serialized to JSON.

### The node model

A `ReportNode` carries:

- a message **key** (`getMessageKey()`) identifying the template;
- a **template** (`getMessageTemplate()`) with `${key}` placeholders;
- a map of **values** (`getValues()` → `Map<String, TypedValue>`), looked up with inheritance via `getValue(key)`;
- a list of **children** (`getChildren()`).

`getMessage()` produces the final string by substituting `${key}` placeholders (via Apache Commons `StringSubstitutor`) with the node's values *and any value inherited from an ancestor* — so a value such as a substation id set high in the tree is available to every descendant message. A `ReportFormatter` controls how a `TypedValue` is rendered during substitution.

`TypedValue` wraps a value together with a semantic type name: `SEVERITY`, `ACTIVE_POWER`, `REACTIVE_POWER`, `VOLTAGE`, `ANGLE`, `IMPEDANCE`, `SUBSTATION`, `VOLTAGE_LEVEL`, `FILENAME`, `TIMESTAMP`, `ID` and more, with `UNTYPED` for plain values. It predefines the severity singletons `TRACE_SEVERITY`, `DEBUG_SEVERITY`, `INFO_SEVERITY`, `WARN_SEVERITY`, `ERROR_SEVERITY` and `DETAIL_SEVERITY`, and a `getTimestamp(formatter)` factory.

### Builder / adder pattern

A root is created through a builder, children through an adder; both share the fluent `ReportNodeAdderOrBuilder` contract (`withMessageTemplate`, `withTypedValue`, `withUntypedValue`, `withSeverity`, `withTimestamp`, …) implemented by `AbstractReportNodeAdderOrBuilder`:

- `ReportNode.newRootReportNode()` returns a `ReportNodeBuilder` (`ReportNodeRootBuilderImpl`); `build()` creates the root `ReportNodeImpl` and its `TreeContext`.
- `someNode.newReportNode()` returns a `ReportNodeAdder` (`ReportNodeChildAdderImpl`); `add()` creates and attaches a child `ReportNodeImpl`.

```java
ReportNode root = ReportNode.newRootReportNode()
        .withResourceBundles(PowsyblCoreReportResourceBundle.BASE_NAME)
        .withMessageTemplate("importNetwork")
        .build();
root.newReportNode()
        .withMessageTemplate("substationProcessed")
        .withTypedValue("substation", "S1", TypedValue.SUBSTATION)
        .withSeverity(TypedValue.INFO_SEVERITY)
        .add();
```

`ReportNode.NO_OP` (a `ReportNodeNoOp`) is a null-object: it lets callers report unconditionally without null checks, with `TreeContextNoOp` as its companion no-op `TreeContext`. `addCopy(...)` and `include(...)` graft another tree (or a copy of it) under a node, merging dictionaries.

### Tree context and message templates

`TreeContext` holds the tree-wide shared state: the **dictionary** mapping keys to templates, the `Locale`, and the timestamp formatter. `TreeContextImpl` keeps the dictionary as a synchronized sorted map and merges entries when subtrees are included.

Templates are resolved through the `MessageTemplateProvider` SPI (base `AbstractMessageTemplateProvider`), which turns a `(key, locale)` pair into a template string. Implementations: `BundleMessageTemplateProvider` (one `ResourceBundle` base name), `MultiBundleMessageTemplateProvider` (several, searched in order) and `EmptyMessageTemplateProvider` (used during deserialization). Bundles are discovered through the `ReportResourceBundle` SPI — an implementation simply returns its base name and registers with `@AutoService(ReportResourceBundle.class)`; `PowsyblCoreReportResourceBundle` is the core module's own bundle. A provider can run in *strict* mode (a missing key is an error) or loose mode.

### JSON serialization

`ReportNodeSerializer` / `ReportNodeDeserializer` (with inner `TypedValue` (de)serializers) handle JSON, wired through the Jackson module `ReportNodeJsonModule`. The serialized form carries a `version`, the `dictionaries` and the `reportRoot`. `ReportNodeVersion` enumerates the format versions (current `V_3_0`); `ReportConstants` holds the well-known keys (`reportSeverity`, `reportTimestamp`) and the default timestamp pattern.

## Extension framework

Package `com.powsybl.commons.extensions`.

This is the generic machinery that lets any object be enriched with typed data without touching its interface; the IIDM model is its primary client (see the [extension mechanism overview](../index.md#the-extension-mechanism)).

### Extension and extendable

- `Extension<T>` — a piece of data attached to a holder of type `T`: `getName()`, `getExtendable()` / `setExtendable(...)`, and a `cleanup()` hook called before removal. `AbstractExtension<T>` is the usual base.
- `Extendable<O>` — the holder side: `addExtension(type, extension)`, `getExtension(type)`, `getExtensionByName(name)`, `removeExtension(type)`, `getExtensions()`, plus the fluent `newExtension(adderClass)` and a `getImplementationName()` used to select the right adder provider.
- `AbstractExtendable<T>` — the default holder implementation. It keys extensions both by their interface class (`Map<Class<?>, Extension<T>>`) and by their name (`Map<String, Extension<T>>`), and calls `Extension.cleanup()` whenever an extension is replaced or removed.

### Adder (builder) pattern

Extensions are created through a builder that adds itself to its extendable: `ExtensionAdder<T, E>` exposes `getExtensionClass()` and `add()`. `AbstractExtensionAdder<T, E>` holds the target extendable and implements `add()` as "create the extension, register it under `getExtensionClass()`, return it"; subclasses only implement `createExtension(extendable)`.

```java
generator.newExtension(ActivePowerControlAdder.class)
        .withDroop(4.0)
        .add();
```

### Discovery and serialization SPIs

Discovery uses the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface):

- `ExtensionProvider<T, E>` is the common SPI super-type (`getExtensionName`, `getCategoryName`, `getExtensionClass`).
- `ExtensionAdderProvider<T, E, B>` supplies adders, matched against an extendable's `getImplementationName()`. `ExtensionAdderProviders` discovers and caches them by implementation name + adder class / name.
- `ExtensionSerDe<T, E>` (XML / tree-data, base `AbstractExtensionSerDe`) and `ExtensionJsonSerializer<T, E>` (JSON) supply the (de)serializers; both extend `ExtensionProvider`. `ExtensionSerDe` also carries XSD, namespace and version information and is `Versionable`.
- `ExtensionConfigLoader<T, E>` builds an extension from `PlatformConfig`.

`ExtensionProviders<T>` is the typed registry: its static `createProvider(...)` factories load every provider of a given SPI through an `ExtensionProvidersLoader` (default `DefaultExtensionProvidersLoader`, backed by a cached `ServiceLoader`) and index them by extension name, exposing `findProvider(name)` / `findProviderOrThrowException(name)`.

## Plugin / `PluginInfo` mechanism

Package `com.powsybl.commons.plugins`.

This package surfaces, for tooling, the catalogue of plugin categories present on the classpath. `PluginInfo<T>` describes one category of plugin: its SPI class (`getPluginClass()`) and a human-readable name (`getPluginName()`), plus `getId(impl)` deriving an id for a given implementation (the implementation's class name by default). `Plugins` is the static utility that enumerates all `PluginInfo` instances registered through `ServiceLoader` (`getPluginInfos()`, `getPluginInfoByName(name)`) and, for a given category, lists the ids of the available implementations (`getPluginImplementationsIds(pluginInfo)`). This is what the `PluginsInfoTool` in the `tools` module prints on the command line.

Underpinning all of the SPI usage above is `ServiceLoaderCache<S>` from `com.powsybl.commons.util`: a thread-safe, lazily-populated wrapper around `ServiceLoader.load(...)` that caches the discovered services. Because `ServiceLoader` is comparatively expensive, this cache is used nearly everywhere the framework loads providers.

## JSON, tree-data IO and table formatting

### JSON

Package `com.powsybl.commons.json`. `JsonUtil` centralizes Jackson usage behind a consistent configuration. Beyond factory creation (`createObjectMapper`, `createJsonFactory`), it offers: `writeJson` / `parseJson` overloads taking a `Consumer<JsonGenerator>` or `Function<JsonParser, T>` (so callers stream without managing the parser/generator lifecycle); object read/write/update helpers; optional-field writers (`writeOptionalStringField`, `writeOptionalEnumField`, …); typed array parsers; extension read/write/update helpers; a `skip(...)` for subtrees; and a family of version-assertion helpers (`assertSupportedVersion`, `compareVersions`, …) used by versioned serializers. `JsonReader` / `JsonWriter` are the streaming tree-data implementations (see below).

### Tree-data IO

Package `com.powsybl.commons.io` generalizes serialization beyond JSON. `TreeDataWriter` and `TreeDataReader` (with `AbstractTreeDataWriter` / `AbstractTreeDataReader`, the `SerializerContext` / `DeserializerContext` holders and the `TreeDataHeader` record carrying root and extension versions) define a format-independent tree-document model: start/end nodes, typed attributes, arrays, namespaces and content. The `TreeDataFormat` enum — `XML`, `JSON`, `BIN` — is what lets IIDM serialization target XML, JSON or a binary form through the same abstraction; the concrete writers/readers live in the `xml`, `json` and `binary` packages. `WorkingDirectory` (an `AutoCloseable` temporary directory, kept when debugging) and `FileUtil` (`removeDir`, `copyDir`, …) are filesystem helpers.

### Table formatting

Package `com.powsybl.commons.io.table` is a small table-rendering toolkit: the `TableFormatter` interface (an `AutoCloseable` with `writeCell(...)` / `writeComment(...)` / `writeEmptyLine(...)`), the `AbstractTableFormatter` base and the `AsciiTableFormatter` / `CsvTableFormatter` implementations, created via matching `TableFormatterFactory` instances, configured through `TableFormatterConfig` (locale, CSV separator, header/title flags, loadable from `PlatformConfig`) and described with `Column` and `HorizontalAlignment`.

## Parameters

Package `com.powsybl.commons.parameters`. `Parameter` is the uniform model used by importers, exporters and tools to declare a configurable parameter: one or more names/aliases, a `ParameterType` (`BOOLEAN`, `STRING`, `STRING_LIST`, `DOUBLE`, `INTEGER`), a description, a default value, optional possible values, a `ParameterScope` (`FUNCTIONAL` or `TECHNICAL`) and a category key. Static `read*` helpers read a value from a `Properties` instance, falling back to configuration. `ParameterDefaultValueConfig` reads default-value overrides from `PlatformConfig` (module `import-export-parameters-default-value`), and `ConfiguredParameter` binds a parameter to its resolved value while retaining the original (`baseDefaultValue`) so UIs can show what was overridden.

## Other notable packages

The remaining packages back the rest of the framework:

| Package | Notable types | Role |
|---------|---------------|------|
| `binary` | `BinReader`, `BinWriter`, `BinUtil` | Binary `TreeDataReader` / `TreeDataWriter` implementations, with a string dictionary for interning. |
| `xml` | `XmlReader`, `XmlWriter`, `XmlUtil` | StAX-based tree-data IO and XML helpers. |
| `compress` | `ZipPackager`, `SafeZipInputStream` | Build ZIP archives; guard against Zip-Slip path traversal. |
| `concurrent` | `CleanableExecutors`, `CleanableThreadPoolExecutor` | Thread pools with pluggable per-thread cleanup. |
| `exceptions` | the `Unchecked*` family (e.g. `UncheckedIllegalAccessException`, `UncheckedInterruptedException`, `UncheckedXmlStreamException`) | Unchecked wrappers around common checked exceptions. |
| `net` | `UserProfile` | A small DTO (first/last name). |
| `ref` | `Ref`, `RefChain`, `RefObj` | Indirection holders used by IIDM to keep references stable across network merges. |
| `util` | `ServiceLoaderCache`, `StringToIntMapper`, `StringAnonymizer`, `WeakListenerList`, `Colors` | The cached `ServiceLoader`, bidirectional string↔int and string-anonymization mappers, a weak-reference listener list, ANSI colours. |
