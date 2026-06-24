# AMPL

Two modules deal with [AMPL](../../grid_exchange_formats/ampl/index.md), the modelling language for mathematical optimization. They are complementary but have distinct responsibilities:

| Module | ArtifactId | Role |
|--------|------------|------|
| `ampl-converter` | `powsybl-ampl-converter` | Serializes an IIDM network to the AMPL data files (`Exporter`) and reads AMPL solver output back into a network. |
| `ampl-executor` | `powsybl-ampl-executor` | The run harness that actually executes an AMPL model on a network, on top of the computation framework. |

There is no AMPL `Importer`: AMPL files describe a network *for a solver* (a set of indexed tables), not a standalone interchange case.

## ampl-converter

### Export: `AmplExporter` and `AmplNetworkWriter`

`com.powsybl.ampl.converter.AmplExporter` is the `@AutoService(Exporter.class)` implementation (format `"AMPL"`). It reads its parameters, builds an `AmplExportConfig`, and delegates the actual work to `com.powsybl.ampl.converter.AmplNetworkWriter`. The writer emits one CSV-like `.txt` file per equipment type (buses, branches, loads, generators, tap changers, HVDC lines, ...), using the `TableFormatter` from `commons` and an `AmplSubset`-based id mapper.

The exporter is configured through a set of `iidm.export.ampl.*` parameters:

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `iidm.export.ampl.scope` | string (`ExportScope`) | `ALL` | What to export (full network / only the main connected component, ...). |
| `iidm.export.ampl.with-xnodes` | boolean | `false` | Export tie-line X-nodes. |
| `iidm.export.ampl.action-type` | string (`ExportActionType`) | `CURATIVE` | Preventive vs. curative remedial-action labelling. |
| `iidm.export.ampl.export-ratio-tap-changer-voltage-target` | boolean | `false` | Add the `targetV` column for ratio tap changers. |
| `iidm.export.ampl.twt-split-shunt-admittance` | boolean | `false` | Split transformer shunt admittance between sides. |
| `iidm.export.ampl.export-version` | string | `1.2` | Output format version (see below). |
| `iidm.export.ampl.export-sorted` | boolean | `false` | Sort rows alphabetically by equipment id. |

Several parameters carry legacy aliases via `addAdditionalNames(...)` so that old configuration keys keep working.

### The versioned exporters

The exact set of columns written for each table depends on a format version. This is captured by the enum `com.powsybl.ampl.converter.version.AmplExportVersion`:

| Enum | Exporter id | `AmplColumnsExporter` factory |
|------|-------------|-------------------------------|
| `V1_0` | `"1.0"` | `BasicAmplExporter` |
| `V1_1` | `"1.1"` | `ExtendedAmplExporter` |
| `V1_2` | `"1.2"` | `ExtendedAmplExporterV2` (the default) |

Each enum value carries a `Factory` that builds an `com.powsybl.ampl.converter.version.AmplColumnsExporter` for a given config, network, id mapper and variant. `AmplColumnsExporter` is the interface that defines, per equipment type, the column list (`getBusesColumns()`, `getGeneratorsColumns()`, ...) and the row-writing methods (`writeBusesColumnsToFormatter(...)`, ...). `AmplNetworkWriter` obtains the right `AmplColumnsExporter` from `config.getVersion().getColumnsExporter()`, then walks the network and asks it to emit each table.

The three implementations form an inheritance chain, each adding columns while staying backward compatible with the previous format:

- `com.powsybl.ampl.converter.version.BasicAmplExporter` is the legacy exporter (version 1.0). Its Javadoc explicitly marks it as retro-compatible: only fixes are allowed.
- `com.powsybl.ampl.converter.version.ExtendedAmplExporter` (version 1.1) extends it, adding the synchronous-component number and a slack-bus flag in the bus table, R/G/B characteristics in the tap-changer tables, and the regulated bus for generators and static var compensators.
- `com.powsybl.ampl.converter.version.ExtendedAmplExporterV2` (version 1.2, default) further adds battery `q0`, a generator "is condenser" flag, LCC `targetQ`, and HVDC AC-emulation columns (`pOffset`, `k`).

Adding a new format version is therefore a matter of adding an enum constant and a new `AmplColumnsExporter` subclass, typically extending the previous one and overriding only the changed columns.

### Reading results back: `AmplNetworkReader` and the updater

`com.powsybl.ampl.converter.AmplNetworkReader` parses the solver output files and applies the results to an existing `Network`. It exposes one `readXxx()` method per element type (`readBuses`, `readBranches`, `readGenerators`, `readLoads`, `readRatioTapChangers`, `readHvdcLines`, `readMetrics`, ...), each returning `this` for chaining. Tokenization handles quoted values, and the column format is described by an `OutputFileFormat`.

Parsing and mutation are separated: the reader extracts values, then calls a `com.powsybl.ampl.converter.AmplNetworkUpdater` to actually modify the network. `AmplNetworkUpdater` is an interface with one method per element type (`updateNetworkGenerators`, `updateNetworkBus`, `updateNetworkBranch`, ...); its default implementation is `DefaultAmplNetworkUpdater` (with the shared base `AbstractAmplNetworkUpdater`), and a `DefaultAmplNetworkUpdaterFactory` builds it. The updater is supplied through the functional `AmplNetworkUpdaterFactory`, so callers can substitute their own application logic (e.g. apply only a subset of results) without touching the parser. The `AmplReadableElement` enum maps each element type to its `AmplNetworkReader::readXxx` method, which is how the executor declares what to read back.

### The `AmplExtension` SPI

Beyond the standard equipment tables, custom data attached to IIDM through extensions can be written to dedicated AMPL files. `com.powsybl.ampl.converter.AmplExtensionWriter` is the SPI:

```java
public interface AmplExtensionWriter {
    String getName();
    void write(List<AmplExtension> extensions, Network network, int variantIndex,
               StringToIntMapper<AmplSubset> mapper, DataSource dataSource, boolean append,
               AmplExportConfig config) throws IOException;
}
```

`com.powsybl.ampl.converter.AmplExtension<A extends Extension<B>, B>` is a small wrapper binding an IIDM extension to the AMPL index (`extendedNum`) of the object it extends. `com.powsybl.ampl.converter.AmplExtensionWriters.getWriter(name)` discovers the registered writers through `ServiceLoader` and returns the one matching a given name. This is the standard [plugin recipe](../index.md#the-plugin-mechanism-service-provider-interface) applied to AMPL extension serialization.

`AmplSubset` is the enum of indexed object families (BUS, BRANCH, GENERATOR, ...) used by the `StringToIntMapper` to turn IIDM string ids into the integer ids AMPL expects, consistently across export and read-back.

## ampl-executor

`ampl-executor` is **not** a converter; it is the harness that runs an AMPL model on a network, reusing `ampl-converter` to write the network and read the results, and the `computation` framework to actually run the solver.

### Describing a model: `AmplModel`

`com.powsybl.ampl.executor.AmplModel` is the interface a concrete model implements. It declares everything the harness needs:

- `getModelAsStream()` — the model files (`.mod`, `.dat`, `.run`) as name/`InputStream` pairs;
- `getAmplRunFiles()` — the `.run` files to execute;
- `getNetworkUpdaterFactory()` — which `AmplNetworkUpdaterFactory` to use when applying results;
- `getAmplReadableElement()` — the `AmplReadableElement`s to read back after the solve;
- `getVariant()`, `getOutputFormat()`, `getOutputFilePrefix()`, `getNetworkDataPrefix()` — variant and file-naming/format details;
- `checkModelConvergence(metrics)` — interpret the output indicators to decide whether the solve converged.

`com.powsybl.ampl.executor.AbstractAmplModel` provides a base implementation; `AbstractMandatoryOutputFile` helps declare required output files.

### Running a model: `AmplModelRunner` and the execution handler

`com.powsybl.ampl.executor.AmplModelRunner` is the entry point (static `run` / `runAsync`). It builds an `ExecutionEnvironment`, creates an `AmplModelExecutionHandler`, and submits it to a `ComputationManager`; `run` simply joins the `CompletableFuture` returned by `runAsync`.

`com.powsybl.ampl.executor.AmplModelExecutionHandler` (an `AbstractExecutionHandler<AmplResults>`) does the actual orchestration in the computation working directory: it copies the model files, exports the network with `AmplExporter`, writes any extra input files, invokes the `ampl` binary on the run files, then — if the model converged — reads the standard outputs through the `AmplReadableElement`s and the extra outputs, applying everything to the network. The id mapper (`AmplUtil.createMapper`) is shared between export and read-back.

### Configuration, parameters and results

- `com.powsybl.ampl.executor.AmplConfig` reads the `ampl` module from `PlatformConfig` (notably `homeDir`, the AMPL installation directory).
- `com.powsybl.ampl.executor.AmplParameters` lets a caller extend a run with extra input files (`AmplInputFile`) and extra output files (`AmplOutputFile`), control debug mode (keep temporary files, choose a debug directory) and provide the `AmplExportConfig`. `AmplInputFile`/`AmplOutputFile` write/read through a `BufferedWriter`/`BufferedReader` and receive the same `StringToIntMapper<AmplSubset>`. `EmptyAmplParameters` is a no-extra-files default.
- `com.powsybl.ampl.executor.AmplResults` is the result object: a `success` flag plus a map of indicators read from the solve.

## Design notes

The split between the two modules is the key design point: `ampl-converter` owns the data format (export, read-back, versioned column layouts and the extension SPI) and knows nothing about how AMPL is run, while `ampl-executor` owns the run lifecycle on top of the computation framework and delegates all I/O back to `ampl-converter`. The two main extension points are the versioned `AmplColumnsExporter` family (new output format versions) and the `AmplExtensionWriter` SPI (serializing custom IIDM extensions); on the executor side, the `AmplModel` and `AmplParameters` interfaces let callers plug in their own model files and extra I/O.
