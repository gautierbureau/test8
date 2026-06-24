# Developer guide

This guide describes the internal architecture of PowSyBl Core: how the code is organized, the main design patterns, and the extension points used to plug new behaviour into the framework. It is intended for developers who want to understand the codebase or extend it, as a complement to the user-facing documentation of the other sections.

## Module organization

PowSyBl Core is a multi-module Maven project. The modules are organized in layers, each layer depending only on the ones below it:

| Layer | Modules | Role |
|-------|---------|------|
| **Foundation** | `commons`, `math`, `time-series`, `tools`, `computation`, `computation-local`, `config-classic` | Cross-cutting utilities: configuration, data sources, reporting, the extension framework, graph and math helpers, time series, the computation abstraction and the iTools command framework. |
| **Grid model** | `iidm`, `contingency`, `action-api`, `action-ial` | The IIDM grid model (the core data model), together with the contingency and remedial-action models. |
| **Grid exchange formats** | `cgmes`, `ucte`, `psse`, `matpower`, `powerfactory`, `ieee-cdf`, `ampl-converter`, `ampl-executor`, `entsoe-util`, `cim-anonymiser`, `triple-store` | Importers and exporters converting external formats to and from IIDM. |
| **Simulation** | `loadflow`, `security-analysis`, `sensitivity-analysis-api`, `shortcircuit-api`, `dynamic-simulation`, `dynamic-security-analysis` | The simulation APIs. Each defines a provider interface; the actual algorithms live in separate repositories (e.g. PowSyBl Open Load Flow). |
| **Scripting** | `dsl`, `scripting`, `iidm-scripting` | The Groovy DSLs and scripting support. |
| **Distribution** | `itools-packager`, `distribution-core` | Packaging of an iTools distribution. |

Several modules are themselves split into submodules to separate the API from its implementations and optional features. For example `iidm` contains `iidm-api` (the interfaces), `iidm-impl` (the in-memory implementation), `iidm-extensions`, `iidm-serde` (XML/JSON/binary serialization), `iidm-modification`, `iidm-criteria`, `iidm-geodata`, `iidm-reducer` and `iidm-tck` (the technology compatibility kit used to validate alternative implementations).

## The IIDM grid model

The [IIDM grid model](../grid_model/index.md) is the heart of PowSyBl. All the importers, exporters and simulations work on this single in-memory model. It is defined as a set of interfaces in `iidm-api` (`Network`, `VoltageLevel`, `Generator`, ...) and the default in-memory implementation is provided by `iidm-impl`. Network objects are created through *adders* (a fluent builder pattern, e.g. `voltageLevel.newGenerator()...add()`), and the model supports multiple variants (states) of the same network.

(developer-guide-spi)=
## The plugin mechanism (Service Provider Interface)

The most important design pattern in PowSyBl is its **plugin mechanism**: most behaviours are defined by an interface (a Service Provider Interface, or SPI) and discovered at runtime through Java's `ServiceLoader`. Implementations declare themselves by being annotated with `@AutoService(...)` (from Google `auto-service`), which generates the `META-INF/services` descriptor at compile time. At runtime, the framework loads all the implementations found on the classpath — in an iTools distribution, this means all the jars present in the `share/java` folder.

This is why adding a new format, a new simulator or a new extension is simply a matter of putting the right jar on the classpath: nothing has to be registered explicitly. The main extension points are:

| Service Provider Interface | Purpose |
|----------------------------|---------|
| `Importer` / `Exporter` | Convert a given format to / from IIDM |
| `LoadFlowProvider`, `SecurityAnalysisProvider`, `SensitivityAnalysisProvider`, `ShortCircuitAnalysisProvider`, `DynamicSimulationProvider` | Provide a simulation implementation |
| `Tool` | Add an iTools command |
| `ExtensionAdderProvider`, `ExtensionSerDe` / `ExtensionJsonSerializer` | Add and (de)serialize an IIDM extension |
| `ImportPostProcessor`, `CgmesImportPostProcessor` | Run automatic processing after an import |
| `ContingenciesProviderFactory`, `NamingStrategyProvider`, `NetworkFactoryService` | Pluggable contingency providers, naming strategies and network factories |

## The extension mechanism

The IIDM model can be enriched without modifying its core interfaces, through *extensions*. An extension is a piece of typed data attached to a network object (an `Extension<T>` where `T` is the extended type, e.g. `Extension<Generator>`). Extensions are added through an adder (`object.newExtension(XxxAdder.class)...add()`) and serialized through a dedicated `ExtensionSerDe` (XML/binary) or `ExtensionJsonSerializer` (JSON). The available extensions are documented in the [extensions](../grid_model/extensions.md) page; the adders and serializers are themselves discovered through the plugin mechanism.

## Cross-cutting services

A few foundation services are used throughout the codebase:

- **Configuration** — `PlatformConfig` gives access to the stacked YAML/XML configuration described in the [configuration](../user/configuration/index.md) section. Each configurable feature reads its own module from it.
- **Computation** — `ComputationManager` abstracts where and how computations run (locally or on a cluster). The `computation-local` module provides the local implementation.
- **Reporting** — `ReportNode` collects the structured, localizable functional logs produced during imports, exports and simulations (see [functional logs](../user/functional_logs/index.md)).
- **Data sources** — `DataSource` abstracts the physical storage (plain files, archives, in-memory) that importers and exporters read from and write to.

## Extending PowSyBl

Thanks to the plugin mechanism, extending PowSyBl follows a recurring recipe: implement the relevant SPI, annotate the implementation with `@AutoService`, and add the jar to the classpath. Typically:

- to support a **new format**, implement `Importer` and/or `Exporter`;
- to provide a **new simulation implementation**, implement the corresponding `*Provider`;
- to attach **new data** to the model, define an `Extension` with its adder and serializer;
- to add a **new command line tool**, implement `Tool`.
