# Load flow

The `loadflow` module defines the load-flow API of PowSyBl Core. It describes *what* a
load-flow run is — its parameters, its result model and its provider contract — but does
**not** implement any power-flow algorithm. The numerical solvers live in separate
repositories (for example [PowSyBl Open Load Flow](https://github.com/powsybl/powsybl-open-loadflow))
and are plugged in at runtime through the
[plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface). The
user-facing documentation is in the [load flow](../../simulation/loadflow/index.md) section;
this page describes the code.

The module is split into four submodules:

| Submodule | Role |
|-----------|------|
| `loadflow-api` | The API: provider SPI, facade, parameters and result model. |
| `loadflow-validation` | Tooling that checks the physical consistency of a network on which a load flow has been run. |
| `loadflow-results-completion` | Completion of partial load-flow results (e.g. flows on zero-impedance branches). |
| `loadflow-scripting` | Groovy integration that exposes the load-flow API to the scripting layer. |

## loadflow-api

The package `com.powsybl.loadflow` holds the API, with serialization in
`com.powsybl.loadflow.json` and the iTools command in `com.powsybl.loadflow.tools`.

### Provider SPI

`LoadFlowProvider` is the service provider interface. Like every simulation provider it
extends `Versionable` and `PlatformConfigNamedProvider`, so an implementation has a name and
a version and is discovered through `ServiceLoader` (`LoadFlowProvider.findAll()`). The single
mandatory method is:

```java
CompletableFuture<LoadFlowResult> run(Network network, String workingVariantId, LoadFlowRunParameters runParameters);
```

The run is expected to be **stateless**, so the same provider instance can be called
concurrently with different networks. The contract is that it updates the working variant of
the network in place with the computed physical values, and returns a `LoadFlowResult`.

What distinguishes `LoadFlowProvider` from the other SPIs is its rich support for
*implementation-specific parameters*, carried as an `Extension<LoadFlowParameters>`. A
provider is asked to bridge that extension to the various serialization and configuration
formats through a set of methods:

- `getSpecificParametersClass()` / `getSpecificParametersSerializer()` — the extension class
  and its `ExtensionJsonSerializer` (for JSON);
- `loadSpecificParameters(PlatformConfig)` and `loadSpecificParameters(Map<String, String>)`
  — read the extension from platform configuration or from a string map;
- `createMapFromSpecificParameters(...)`, `updateSpecificParameters(...)` (both `Map` and
  `PlatformConfig` variants) — convert to / update from a string map;
- `getRawSpecificParameters()` / `getSpecificParameters(...)` — expose the provider's
  parameters as a list of `Parameter` descriptors, with config overrides applied through
  `ConfiguredParameter.load(...)`.

`LoadFlowProviderUtil` and `AbstractLoadFlowDefaultParametersLoader` /
`LoadFlowDefaultParametersLoader` help a provider supply default values for its extension;
`AbstractNoSpecificParametersLoadFlowProvider` is a base class for providers that have no
specific parameters. A `checkParameters(LoadFlowRunParameters)` hook lets a provider report
unsupported common options through the run's report node. `LoadFlowProviderPluginInfo`
(annotated `@AutoService(PluginInfo.class)`) registers the SPI with the plugin-info registry.

### Facade and runner

`LoadFlow` is a final utility class exposing the static entry points. `LoadFlow.find(name)`
resolves a provider against the `load-flow` configuration module
(`PlatformConfigNamedProvider.Finder.find(name, "load-flow", LoadFlowProvider.class, ...)`)
and wraps it in an inner `LoadFlow.Runner` (which is itself `Versionable`). `find()` selects
the default. The runner — and the mirror static methods on `LoadFlow` — offer many
overloads of `run(...)` and `runAsync(...)`; the synchronous ones simply call `.join()` on
the asynchronous future. The modern signatures group the run inputs into a
`LoadFlowRunParameters`:

```java
LoadFlowResult run(Network network, String workingStateId, LoadFlowRunParameters runParameters);
CompletableFuture<LoadFlowResult> runAsync(Network network, LoadFlowRunParameters runParameters);
```

The older overloads that pass a `ComputationManager`, a `LoadFlowParameters` and a
`ReportNode` directly are deprecated since `7.0.0` for removal. `LoadFlow.checkParameters(...)`
and `checkDefaultParameters(ReportNode)` delegate to the provider's parameter check.

`LoadFlowRunParameters` is the run-context holder. It carries a `LoadFlowParameters`, a
`ComputationManager` and a `ReportNode`. Its getters lazily fall back to defaults
(`LoadFlowParameters::load`, `LocalComputationManager::getDefault`, `ReportNode.NO_OP`), and
`LoadFlowRunParameters.getDefault()` builds a fully defaulted instance.

### Parameters

`LoadFlowParameters` extends `AbstractExtendable<LoadFlowParameters>` and holds the *common*
options understood by every implementation, each backed by a `DEFAULT_*` constant. Among
them: the `VoltageInitMode` (`UNIFORM_VALUES`, `PREVIOUS_VALUES`, `DC_VALUES`); the slack
handling flags `readSlackBus` / `writeSlackBus` / `distributedSlack`; the `BalanceType`
(`PROPORTIONAL_TO_GENERATION_P_MAX`, `PROPORTIONAL_TO_LOAD`, ...) governing how slack is
distributed; control toggles `transformerVoltageControlOn`, `shuntCompensatorVoltageControlOn`,
`phaseShifterRegulationOn`, `useReactiveLimits`, `twtSplitShuntAdmittance`; the DC options
`dc`, `dcUseTransformerRatio`, `dcPowerFactor`; `hvdcAcEmulation`; the set of
`countriesToBalance`; and the `ComponentMode` (`MAIN_CONNECTED`, `ALL`) selecting which
connected components are solved.

Parameters are built in code, loaded from configuration with `LoadFlowParameters.load()` /
`load(PlatformConfig)` (reading the `load-flow-default-parameters` module, plus the
`load-flow` module's `default-parameters-loader` selection), or read from JSON through
`JsonLoadFlowParameters`. Provider-specific options are attached as extensions and
(de)serialized by an `ExtensionJsonSerializer` discovered via `@AutoService`, so an
implementation contributes its own parameter extension without touching the API. The JSON
plumbing is `JsonLoadFlowParameters`, `LoadFlowParametersSerializer` /
`LoadFlowParametersDeserializer`, registered in `LoadFlowParametersJsonModule`.

### Result model

`LoadFlowResult` is an interface. Its global `LoadFlowResult.Status` is derived from the
per-component statuses, excluding non-calculated components:

| `Status` | Meaning |
|----------|---------|
| `FULLY_CONVERGED` | All calculated components converged (and at least one did). |
| `PARTIALLY_CONVERGED` | At least one component converged, others failed or hit the iteration limit. |
| `FAILED` | No component converged. |

Convenience predicates `isFullyConverged()`, `isPartiallyConverged()`, `isFailed()` wrap the
status; the older boolean `isOk()` and the metrics map round out the global view. The detail
is in `getComponentResults()`, a list of `LoadFlowResult.ComponentResult`. Each component
result carries its connected- and synchronous-component numbers, its own
`ComponentResult.Status` (`CONVERGED`, `MAX_ITERATION_REACHED`, `FAILED`, `NO_CALCULATION`),
a status text, an iteration count, the reference (angle) bus id, the distributed active power
and the list of `SlackBusResult` (each a slack bus id with its active-power mismatch in MW).
`LoadFlowResultImpl` is the default implementation; JSON (de)serialization is handled by
`LoadFlowResultSerializer` / `LoadFlowResultDeserializer`, registered in
`LoadFlowResultJsonModule`.

### Tool

`RunLoadFlowTool` (annotated `@AutoService(Tool.class)`) provides the `loadflow` iTools
command, in the `Computation` category.

## loadflow-validation

This submodule checks the physical consistency of a network on which a load flow has been
run — i.e. whether the equipment equations are actually satisfied. The `ValidationType` enum
enumerates the checks, each tied to an output CSV file: `FLOWS` (`branches_flows.csv`),
`GENERATORS`, `BUSES`, `SVCS`, `SHUNTS`, `TWTS` and `TWTS3W`. Each check has a dedicated
class — `FlowsValidation`, `GeneratorsValidation`, `BusesValidation`,
`StaticVarCompensatorsValidation`, `ShuntCompensatorsValidation`, and the transformer
validations `TransformersValidation` / `Transformers3WValidation` over the shared
`AbstractTransformersValidation`.

Helper computations include `KComputation` and `BalanceTypeGuesser` (which infer the active
slack distribution, with the `BalanceType` enum), and `ValidationUtils`. Validation is
configured by `ValidationConfig` (read from the `loadflow-validation` configuration module).
Results are written through the `io` package: the `ValidationWriter` /
`ValidationWriterFactory` abstraction with CSV and multiline-CSV implementations
(`ValidationFormatterCsvWriter`, `ValidationFormatterCsvMultilineWriter` and their
factories), gathered by `ValidationWriters` and selected by the `ValidationOutputWriter`
enum. The `extension` package (`ExtensionValidation`, `ExtensionsValidation`) lets extensions
contribute their own checks.

Validation is also exposed as a `CandidateComputation` — an SPI (in this submodule) for a
named computation that can be run on a network — through `LoadFlowComputation` (name
`loadflow`), with `CandidateComputations` and `CandidateComputationPluginInfo` providing the
registry. The `ValidationTool` exposes the whole submodule as the `loadflow-validation`
iTools command.

## loadflow-results-completion

`LoadFlowResultsCompletion` fills in state that a load flow may have left incomplete. It is
registered as a `CandidateComputation` (`@AutoService`, name `loadflowResultsCompletion`) and
its main job is computing the flows on zero-impedance (Z0) branches, handled in the
`z0flows` package: `Z0FlowsCompletion` groups buses connected by Z0 lines into `Z0BusGroup`s
(via `Z0Edge` / `BranchTerminal`), checks lines with `Z0LineChecker`, and balances the flows
with `Z0FlowFromBusBalance`. Completion can run automatically after an import through
`LoadFlowResultsCompletionPostProcessor` (an `ImportPostProcessor`), and is configured by
`LoadFlowResultsCompletionParameters`.

## loadflow-scripting

A thin Groovy integration: `LoadFlowGroovyScriptExtension` (annotated
`@AutoService(GroovyScriptExtension.class)`) binds the load-flow API into the scripting
context, defaulting its `LoadFlowParameters` from `LoadFlowParameters.load()` and pulling the
`ComputationManager` from the script's context objects so scripts can run a load flow
directly.

## Adding a load-flow implementation

Following the generic recipe: implement `LoadFlowProvider`, annotate it with
`@AutoService(LoadFlowProvider.class)`, optionally contribute a parameters extension (with its
`ExtensionJsonSerializer`, also `@AutoService`), and put the jar on the classpath. The
`LoadFlow` facade discovers and selects the provider at runtime, using the `load-flow`
configuration module's `default-impl-name` when no name is given.
