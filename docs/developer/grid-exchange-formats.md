# Grid exchange formats layer

The **grid exchange formats** layer holds all the importers and exporters that convert external power-system file formats to and from the [IIDM grid model](../grid_model/index.md). Each format lives in its own module; together they cover the main interchange formats used in the industry: CGMES, UCTE-DEF, PSS/E, MATPOWER, PowerFactory, the IEEE Common Data Format and the AMPL data files. A few supporting modules complete the layer: `triple-store` (an RDF triple-store abstraction used by CGMES), `entsoe-util` (shared ENTSO-E helpers), `cim-anonymiser` (a tooling utility) and `ampl-executor` (an AMPL model runner).

User-facing documentation of the supported formats lives in the [grid exchange formats](../grid_exchange_formats/index.md) section; this page focuses on the internal design.

## Common design

Every format module follows the same recipe, built on the plugin mechanism described in the [developer guide](index.md#the-plugin-mechanism-service-provider-interface).

- **The `Importer` and `Exporter` SPIs.** Both interfaces are defined in `iidm-api` (`com.powsybl.iidm.network.Importer` and `com.powsybl.iidm.network.Exporter`). An importer implementation provides `getFormat()`, `getComment()`, `exists(ReadOnlyDataSource)` (to detect whether the data source contains its format) and `importData(...)` which builds a `Network`. An exporter provides `getFormat()`, `getComment()` and `exportData(...)` which writes a `Network` out.
- **Discovery through `ServiceLoader`.** Implementations are annotated with `@AutoService(Importer.class)` / `@AutoService(Exporter.class)` (Google `auto-service`), which generates the `META-INF/services` descriptor at compile time. Adding a format to a distribution is therefore just a matter of putting its jar on the classpath; no explicit registration is required.
- **The IIDM model as the pivot.** Conversions never go directly from one external format to another. They always go through the IIDM `Network`: an importer parses the external files into its own intermediate model and then builds an IIDM network through the IIDM *adders*; an exporter walks the IIDM network and writes the external files.
- **`DataSource` for file access.** Importers and exporters do not deal with raw file paths. They read from a `ReadOnlyDataSource` and write to a `DataSource` (`com.powsybl.commons.datasource`), which abstracts the physical storage — a plain directory, a ZIP archive, an in-memory buffer — and resolves the individual entries (suffix + extension) that make up a case.
- **Parameters and reporting.** Configurable behaviour is exposed through `Parameter` objects (read from `PlatformConfig` or passed as `Properties`), and the functional logs produced during a conversion are collected in a `ReportNode`.

Most modules are split into a `*-model` submodule (the intermediate model and its file readers/writers) and a `*-converter` submodule (the `Importer` / `Exporter` and the IIDM mapping), so that the parsing of a format can be reused independently of the IIDM conversion.

## CGMES

CGMES (Common Grid Model Exchange Specification) is the ENTSO-E standard based on the CIM/RDF data model. It is by far the largest format module and is itself split into several submodules.

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `cgmes-model` | `powsybl-cgmes-model` | The CGMES network model, backed by a triple store. |
| `cgmes-conversion` | `powsybl-cgmes-conversion` | Bidirectional conversion between the CGMES model and IIDM. |
| `cgmes-extensions` | `powsybl-cgmes-extensions` | CGMES-specific IIDM extensions (metadata, tap changers, base-voltage mapping...). |
| `cgmes-gl` | `powsybl-cgmes-gl` | Import/export of the GL (Geographical Location) profile. |
| `cgmes-measurements` | `powsybl-cgmes-measurements` | Import of measurement data through a post-processor. |
| `cgmes-shortcircuit` | `powsybl-cgmes-shortcircuit` | Import of short-circuit data through a post-processor. |
| `cgmes-completion` | `powsybl-cgmes-completion` | Completion of missing data in incomplete CGMES instance files. |
| `cgmes-conformity` | `powsybl-cgmes-conformity` | Test support based on ENTSO-E conformity assessment data sets. |
| `cgmes-model-test`, `cgmes-model-alternatives` | — | Test support and experiments around model queries. |

### cgmes-model

`com.powsybl.cgmes.model.CgmesModel` is the central interface: it exposes the CIM content (substations, voltage levels, equipment, topology, state variables...) as the result of queries over an RDF graph. The default implementation, `com.powsybl.cgmes.model.triplestore.CgmesModelTripleStore`, runs SPARQL queries against a `TripleStore` (see [triple-store](#triple-store)). Instances are created from a `DataSource` through the `com.powsybl.cgmes.model.CgmesModelFactory` factory, which loads the CIM/RDF files into a triple store and wraps it as a `CgmesModel`.

### cgmes-conversion

`com.powsybl.cgmes.conversion.CgmesImport` is the `Importer` implementation; it builds a `CgmesModel` and then delegates to `com.powsybl.cgmes.conversion.Conversion`, which translates the CGMES model into an IIDM `Network`. The reverse direction is handled by `com.powsybl.cgmes.conversion.CgmesExport`, the `Exporter` implementation, which serializes an IIDM network back into the CGMES profiles. The import also runs the registered `CgmesImportPostProcessor`s (the mechanism used by `cgmes-gl`, `cgmes-measurements` and `cgmes-shortcircuit`).

### Optional submodules

- **cgmes-extensions** defines the IIDM extensions that carry CGMES information not represented natively in IIDM, such as `CgmesMetadataModels`, `CgmesTapChangers`, `BaseVoltageMapping` and `CimCharacteristics`.
- **cgmes-gl** handles the geographical-location profile through `CgmesGLImporter` and `CgmesGLExporter`, populating/reading IIDM geographical-position extensions.
- **cgmes-measurements** provides `CgmesMeasurementsPostProcessor`, a `CgmesImportPostProcessor` that adds measurement data to the imported network.
- **cgmes-shortcircuit** provides a post-processor importing short-circuit parameters.
- **cgmes-completion** offers helpers to complete instance files that lack mandatory data (for example missing topology or boundary information).

## triple-store

The `triple-store` module is an abstraction over RDF triple stores, used by CGMES to store and query the CIM data.

- `triple-store-api` (`powsybl-triple-store-api`) defines the API. `com.powsybl.triplestore.api.TripleStore` is the main interface (load RDF, run SPARQL queries, update, print/write back the named graphs), and `com.powsybl.triplestore.api.TripleStoreFactory` creates implementations discovered through the plugin mechanism, optionally configured with `TripleStoreOptions`.
- `triple-store-impl-rdf4j` (`powsybl-triple-store-impl-rdf4j`) is the default implementation, built on Eclipse RDF4J.
- `triple-store-test` provides shared tests run against every implementation.

## UCTE

The `ucte` module handles the UCTE-DEF format (`.uct` files). It is split into `ucte-network` (the model, with `com.powsybl.ucte.network.io.UcteReader` / `UcteWriter` and record classes such as `UcteNode`, `UcteLine`, `UcteTransformer`), `ucte-util` (shared helpers) and `ucte-converter` (`powsybl-ucte-converter`). The IIDM mapping is done by `com.powsybl.ucte.converter.UcteImporter` (format `"UCTE"`) and `com.powsybl.ucte.converter.UcteExporter`.

## PSS/E

The `psse` module handles Siemens PSS/E raw files (`.raw` and `.rawx`). `psse-model` defines the power-flow model (`com.powsybl.psse.model.pf.PssePowerFlowModel` and its records `PsseBus`, `PsseGenerator`, `PsseLine`...) and the readers in `com.powsybl.psse.model.io` (such as `LegacyTextReader`). `psse-converter` (`powsybl-psse-converter`) provides `com.powsybl.psse.converter.PsseImporter` (format `"PSS/E"`) and `com.powsybl.psse.converter.PsseExporter`.

## MATPOWER

The `matpower` module handles MATPOWER case files. `matpower-model` defines the model (`com.powsybl.matpower.model.MatpowerModel` with `MBus`, `MGen`, `MBranch`, `MDcLine`...) and the `MatpowerReader` / `MatpowerWriter`. `matpower-converter` (`powsybl-matpower-converter`) provides `com.powsybl.matpower.converter.MatpowerImporter` (format `"MATPOWER"`, from `MatpowerConstants.FORMAT`) and `com.powsybl.matpower.converter.MatpowerExporter`.

## PowerFactory

The `powerfactory` module imports DIgSILENT PowerFactory study cases. It is split into a model (`powerfactory-model`), two file backends — `powerfactory-dgs` (DGS text files) and `powerfactory-db` (binary database export) — and the converter (`powerfactory-converter`). The intermediate model is loaded by `com.powsybl.powerfactory.model.PowerFactoryDataLoader` into a `com.powsybl.powerfactory.model.StudyCase`; the file backends are themselves discovered through `ServiceLoader`. The IIDM mapping is performed by `com.powsybl.powerfactory.converter.PowerFactoryImporter` (format `"POWER-FACTORY"`), assisted by per-equipment converters such as `NodeConverter`, `GeneratorConverter`, `LineConverter` and the HVDC converters. This module is import-only (no exporter).

## IEEE-CDF

The `ieee-cdf` module imports the IEEE Common Data Format. `ieee-cdf-model` defines the model: `com.powsybl.ieeecdf.model.IeeeCdfReader` parses the text file into a `com.powsybl.ieeecdf.model.IeeeCdfModel` made of elements (`IeeeCdfBus`, `IeeeCdfBranch`, `IeeeCdfTitle`...). `ieee-cdf-converter` provides `com.powsybl.ieeecdf.converter.IeeeCdfImporter` (format `"IEEE-CDF"`). This module is import-only.

## AMPL converter and executor

These two modules are concerned with AMPL, a modelling language for optimization. They have distinct responsibilities.

`ampl-converter` (`powsybl-ampl-converter`) converts between IIDM and the AMPL data files. Export is handled by `com.powsybl.ampl.converter.AmplExporter` (the `Exporter` implementation, format `"AMPL"`), which uses `com.powsybl.ampl.converter.AmplNetworkWriter` to serialize the network as a set of CSV-like `.txt` files (one per equipment type). Reading AMPL results back is done by `com.powsybl.ampl.converter.AmplNetworkReader`, which parses the solver output and applies it to a network through an `AmplNetworkUpdater` (default implementation `DefaultAmplNetworkUpdater`). There is no `Importer` here: AMPL files describe a network for a solver rather than a standalone case.

`ampl-executor` (`powsybl-ampl-executor`) is **not** a converter; it is the harness that actually runs an AMPL model on a network, on top of the computation framework. A concrete model implements the `com.powsybl.ampl.executor.AmplModel` interface (which declares the model and run files, the expected outputs and the network-updater factory), and `com.powsybl.ampl.executor.AmplModelRunner` executes it (`run` / `runAsync`) through an `AmplModelExecutionHandler`. Configuration and results are carried by `AmplConfig`, `AmplParameters` and `AmplResults`. Internally the executor relies on `ampl-converter` to write the network and read the results.

## entsoe-util

`entsoe-util` (`powsybl-entsoe-util`) gathers helpers shared by the ENTSO-E-related formats (CGMES and UCTE). Notable classes include `com.powsybl.entsoe.util.EntsoeGeographicalCode` (an enum mapping ENTSO-E geographical codes to ISO countries), the `com.powsybl.entsoe.util.EntsoeArea` IIDM extension on `Substation` (with its adder and `EntsoeAreaSerDe` serializer), `com.powsybl.entsoe.util.EntsoeFileName` (parses date, forecast distance and geographical code out of standard ENTSO-E file names) and `com.powsybl.entsoe.util.BoundaryPoint` / `BoundaryPointXlsParser` for cross-border boundary points.

## cim-anonymiser

`cim-anonymiser` (`powsybl-cim-anonymiser`) is a tooling module that anonymizes CIM/CGMES files, replacing identifiable names, descriptions and RDF references with anonymized values so that cases can be shared without exposing sensitive data. The engine is `com.powsybl.cim.CimAnonymizer`, which streams through the ZIP-archived RDF/XML and substitutes strings while keeping a mapping dictionary (it can preserve internal RDF references while anonymizing external ones). It is exposed as an iTools command by `com.powsybl.cim.CimAnonymizerTool`, an `@AutoService(Tool.class)` implementation providing the `cim-anonymizer` command.
