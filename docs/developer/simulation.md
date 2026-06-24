# Simulation layer

The simulation layer gathers the computation APIs of PowSyBl Core: load flow, security
analysis, sensitivity analysis, short-circuit analysis, dynamic simulation and dynamic
security analysis. These modules define *what* each computation looks like — its inputs,
its parameters and its result model — but, with very few exceptions, they do **not**
implement the numerical algorithms themselves. The actual solvers live in separate
repositories (for example PowSyBl Open Load Flow for load flow, security analysis and
sensitivity analysis, or PowSyBl Dynawo for dynamic simulation and short circuit), and
are plugged in at runtime through the [plugin mechanism](index.md#the-plugin-mechanism-service-provider-interface).

The user-facing documentation of these computations is in the
[simulation](../simulation/index.md) section; this page describes how the APIs are built.

## A shared design pattern

All six modules follow the same recurring structure. Understanding it once is enough to
read any of them:

| Building block | Role |
|----------------|------|
| Provider SPI | A `*Provider` interface (e.g. `LoadFlowProvider`) describing one implementation of the computation. Implementations are discovered through Java's `ServiceLoader` and declare themselves with `@AutoService`. |
| Facade / runner | A final entry-point class (e.g. `LoadFlow`) with static `run`/`runAsync` methods. It selects a provider, wraps it in an inner `Runner` class and delegates. |
| Parameters | A `*Parameters` class holding the configuration of a run. It extends `AbstractExtendable`, can be loaded from `PlatformConfig` and is (de)serialized to JSON. Implementation-specific options are carried as *extensions*. |
| Result model | A `*Result` (interface or class) describing the outcome of the computation, together with its JSON serializers. |
| Tool | An iTools command (a `Tool` implementation, also discovered via `@AutoService`) that runs the computation from the command line. |

### Provider selection

Each `*Provider` interface extends `Versionable` and `PlatformConfigNamedProvider`, so a
provider has a name and a version. The facade's `find(String name)` method resolves which
provider to use: if a name is given it is matched against the providers on the classpath,
otherwise the default one is taken from `PlatformConfig` (for instance the `load-flow`
module's `default-impl-name` property for load flow). When a single provider is present on
the classpath, it is selected automatically. `findAll()` on the provider interface returns
every implementation found by the `ServiceLoader`.

### Parameters

A `*Parameters` object splits its content in two: a set of *common* options understood by
every implementation, and implementation-specific options attached as *extensions* (since
the class extends `AbstractExtendable`). Parameters are typically built in code, loaded
from configuration with a static `load()` / `load(PlatformConfig)` method, or read from a
JSON file through the matching `Json*Parameters` helper in the module's `json` package.
Extension (de)serializers are themselves discovered through `@AutoService`, so an
implementation can contribute its own parameter extension without touching the API.

### Asynchronous results

Runs are exposed both synchronously (`run(...)`) and asynchronously
(`runAsync(...)` returning a `CompletableFuture`). Computations take a `Network`, a working
variant id, a `ComputationManager` and a `ReportNode` for functional logs, most
often grouped into a dedicated `*RunParameters` object.

## Load flow

The `loadflow` module is split into several submodules:

- `loadflow-api` — the API and the provider SPI;
- `loadflow-validation` — tools to validate load-flow results against the network model;
- `loadflow-results-completion` — completion of partial load-flow results;
- `loadflow-scripting` — Groovy scripting support.

### loadflow-api

`LoadFlowProvider` is the SPI; `LoadFlow` is the facade. `LoadFlow.run(...)` returns a
`LoadFlowResult`, while `LoadFlow.runAsync(...)` returns a `CompletableFuture<LoadFlowResult>`.
Internally `LoadFlow.find(...)` resolves a provider against the `load-flow` configuration
module and wraps it in a `LoadFlow.Runner`. Recent runs are configured through a
`LoadFlowRunParameters` object (computation manager, parameters, report node).

`LoadFlowParameters` carries the common options (such as voltage init mode, slack
distribution or transformer voltage control) and extends `AbstractExtendable` so a provider
can add its own option extension; it is loaded with `LoadFlowParameters.load()` and
serialized through `JsonLoadFlowParameters`.

`LoadFlowResult` is an interface. It exposes a global `LoadFlowResult.Status`, a list of
`LoadFlowResult.ComponentResult` (one per connected component, each with its own
`ComponentResult.Status`) and a map of free-form metrics. JSON (de)serialization lives in
`LoadFlowResultSerializer` / `LoadFlowResultDeserializer`.

The `loadflow` iTools command is provided by `RunLoadFlowTool` (command name `loadflow`).

### loadflow-validation

This submodule checks the physical consistency of a network on which a load flow has been
run. It contains per-equipment validation classes (`GeneratorsValidation`,
`ShuntCompensatorsValidation`, `StaticVarCompensatorsValidation`,
`AbstractTransformersValidation` / `Transformers3WValidation`, ...), helper computations
such as `KComputation` and `BalanceTypeGuesser`, the `CandidateComputation` abstraction and
the `ValidationOutputWriter`. The `ValidationTool` exposes it as the `loadflow-validation`
iTools command.

### loadflow-results-completion

`LoadFlowResultsCompletion` fills in missing state on a network after a load flow (for
example the flows on zero-impedance branches, handled in the `z0flows` package via
`Z0FlowsCompletion` and related classes). It can run automatically after an import through
`LoadFlowResultsCompletionPostProcessor`, and is configured by
`LoadFlowResultsCompletionParameters`.

### loadflow-scripting

A thin Groovy integration, `LoadFlowGroovyScriptExtension`, that exposes the load-flow API
to the scripting layer.

## Security analysis

The `security-analysis` module contains a single submodule, `security-analysis-api`.

`SecurityAnalysisProvider` is the SPI and `SecurityAnalysis` the facade. A run takes a
network, a working variant and a contingency list (or a contingencies provider), with the
remaining inputs grouped in a `SecurityAnalysisRunParameters` (subclass of
`AbstractSecurityAnalysisRunParameters`). It returns a `SecurityAnalysisReport`, which wraps
the `SecurityAnalysisResult` and the optional raw log bytes.

`SecurityAnalysisResult` is built from a `PreContingencyResult`, a list of
`PostContingencyResult` and a list of `OperatorStrategyResult`. The detailed result model
lives in the `results` package (`LimitViolationsResult`, `NetworkResult`, `BranchResult`,
`BusResult`, `ThreeWindingsTransformerResult`, `ConnectivityResult`, ...).
`PostContingencyComputationStatus` reports the per-contingency status, and
`SecurityAnalysisResultBuilder` / `SecurityAnalysisResultMerger` help assemble and combine
results (the module supports distributed execution, see the `distributed` and `execution`
packages).

`SecurityAnalysisParameters` extends `AbstractExtendable` and embeds a load-flow parameters
section; `JsonSecurityAnalysisParameters` handles JSON. Other notable packages include
`detectors` and `limitreduction` (limit-violation detection), `interceptors` (hooks into
the computation), `monitor` (state monitoring), `preprocessor` and `comparator`.

The `security-analysis` iTools command is `SecurityAnalysisTool` (with the shared base class
`AbstractSecurityAnalysisTool`); `CompareSecurityAnalysisResultsTool` compares two result
files.

## Sensitivity analysis

The `sensitivity-analysis-api` module defines `SensitivityAnalysisProvider` and the
`SensitivityAnalysis` facade. Because a sensitivity computation can produce a large number
of values, the API is reader/writer oriented: factors are described by `SensitivityFactor`
and fed in through a `SensitivityFactorReader` (with `SensitivityFactorModelReader` and
`SensitivityFactorJsonReader` implementations). The facade offers both a streaming
`run(...)` variant that writes values to a callback and a variant returning a
`SensitivityAnalysisResult`.

`SensitivityAnalysisParameters` extends `AbstractExtendable` and is handled by
`JsonSensitivityAnalysisParameters`; factors are (de)serialized by
`SensitivityFactorJsonSerializer` / `SensitivityFactorJsonDeserializer`. The
`sensitivity-analysis` iTools command is `SensitivityAnalysisTool`.

## Short-circuit analysis

The `shortcircuit-api` module defines `ShortCircuitAnalysisProvider` and the
`ShortCircuitAnalysis` facade. Faults to compute are described by the `Fault` hierarchy
(`AbstractFault`, `BusFault`, `BranchFault`); a run takes the network, a list of faults and
the parameters, and returns a `ShortCircuitAnalysisResult`.

The result aggregates `FaultResult` objects — `MagnitudeFaultResult`,
`FortescueFaultResult` and `FailedFaultResult`, sharing `AbstractFaultResult` — each holding
`FeederResult` and `ShortCircuitBusResults` (with magnitude and Fortescue variants).
`FortescueValue` represents a three-phase quantity.

`ShortCircuitParameters` extends `AbstractExtendable` and is configured globally, while
`FaultParameters` carries per-fault options; JSON support is in the `json` package
(`JsonShortCircuitParameters`, `ShortCircuitAnalysisResultSerializer`, ...). The `converter`
package exports results to several formats (`CsvShortCircuitAnalysisResultExporter`,
`JsonShortCircuitAnalysisResultExporter`, `AsciiShortCircuitAnalysisResultExporter`, all
registered through `ShortCircuitAnalysisResultExporters`). The `shortcircuit` iTools command
is `ShortCircuitAnalysisTool`.

## Dynamic simulation

The `dynamic-simulation` module has three submodules: `dynamic-simulation-api`,
`dynamic-simulation-dsl` (Groovy DSL) and `dynamic-simulation-tool` (iTools commands).

`DynamicSimulationProvider` is the SPI and `DynamicSimulation` the facade. Unlike the other
computations, a dynamic simulation needs more than a network: it takes a
`DynamicModelsSupplier` (the dynamic models attached to network equipments, see
`DynamicModel`), an optional `EventModelsSupplier` (the events to apply, see `EventModel`)
and an `OutputVariablesSupplier` describing which curves to record.

`DynamicSimulationParameters` extends `AbstractExtendable` and defines a `ConfigLoader`
extension point for implementation parameters; `JsonDynamicSimulationParameters` handles
JSON. `DynamicSimulationResult` is an interface exposing a `Status`, the recorded curves as
a map of `DoubleTimeSeries` and a timeline of `TimelineEvent`.

The `dynamic-simulation-dsl` submodule provides the Groovy suppliers
(`GroovyDynamicModelsSupplier`, `GroovyEventModelsSupplier`) and their extension points. The
`dynamic-simulation-tool` submodule provides the `dynamic-simulation` iTools command
(`DynamicSimulationTool`) and `ListDynamicSimulationModelsTool` to list the models a provider
supports.

## Dynamic security analysis

The `dynamic-security-analysis` module combines the dynamic-simulation and security-analysis
APIs: it depends on both `powsybl-dynamic-simulation-api` and `powsybl-security-analysis-api`.

`DynamicSecurityAnalysisProvider` is the SPI and `DynamicSecurityAnalysis` the facade. A run
takes a network, a list of `DynamicModel`, a contingency list and a
`DynamicSecurityAnalysisRunParameters`. It reuses the security-analysis result model,
returning a `SecurityAnalysisReport` (and therefore a `SecurityAnalysisResult`).

`DynamicSecurityAnalysisParameters` extends `AbstractExtendable` and is serialized through
`JsonDynamicSecurityAnalysisParameters`. The `dynamic-security-analysis` iTools command is
provided by `DynamicSecurityAnalysisTool` (with `DynamicSecurityAnalysisCommand`).

## Adding a new simulator

Plugging a new implementation of any of these computations follows the generic extension
recipe described in the [developer guide](index.md): implement the relevant `*Provider`
interface, annotate it with `@AutoService(...)`, optionally contribute a parameters
extension (with its JSON serializer, also annotated with `@AutoService`), and add the jar
to the classpath. No explicit registration is required — the facade discovers and selects
the provider at runtime.
