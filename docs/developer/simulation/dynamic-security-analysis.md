# Dynamic security analysis

The `dynamic-security-analysis` module combines the dynamic-simulation and security-analysis
APIs: it runs a time-domain (dynamic) simulation for the base case and for each contingency,
and reports the resulting limit violations using the security-analysis result model. It
depends on both `powsybl-dynamic-simulation-api` (and `-dsl`) and
`powsybl-security-analysis-api`, and is therefore the most composite of the simulation
modules. The algorithm is external ([PowSyBl Dynawo](https://github.com/powsybl/powsybl-dynawo));
this module defines the contract and parameters and *reuses* the security-analysis result
model. The user-facing documentation is in the
[dynamic security analysis](../../simulation/dynamic_security/index.md) section.

The code lives in `com.powsybl.security.dynamic` (note the package: it sits under the security
package, not the dynamic one), with subpackages `json`, `tools`, `execution` and
`distributed`.

## Provider SPI and facade

`DynamicSecurityAnalysisProvider` is the SPI (extending `Versionable` and
`PlatformConfigNamedProvider`, discovered through `ServiceLoader`). Its computation method
mirrors `SecurityAnalysisProvider` but takes a `DynamicModelsSupplier` instead of relying on a
plain load flow:

```java
CompletableFuture<SecurityAnalysisReport> run(Network network, String workingVariantId,
        DynamicModelsSupplier dynamicModelsSupplier, ContingenciesProvider contingenciesProvider,
        DynamicSecurityAnalysisRunParameters runParameters);
```

It exposes the usual specific-parameter hooks plus `getDynamicSimulationProviderName()` (the
underlying dynamic-simulation implementation it relies on).

`DynamicSecurityAnalysis` is the facade. `find(name)` resolves a provider against the
`dynamic-security-analysis` configuration module and wraps it in an inner
`DynamicSecurityAnalysis.Runner`. The static and runner `run` / `runAsync` overloads take a
network, the dynamic models (as a `DynamicModelsSupplier` or a `List<DynamicModel>`), the
contingencies (as a `ContingenciesProvider` or a `List<Contingency>`) and a
`DynamicSecurityAnalysisRunParameters`. Crucially, they return a `SecurityAnalysisReport` —
**the same report (and therefore the same `SecurityAnalysisResult`) as the security-analysis
module** — so all the result tooling (interceptors, monitors, comparators) carries over.

## Run parameters

`DynamicSecurityAnalysisRunParameters` extends the shared
`AbstractSecurityAnalysisRunParameters<DynamicSecurityAnalysisRunParameters>` (the same base
as `SecurityAnalysisRunParameters`), so it inherits the `LimitViolationFilter`,
`ComputationManager`, interceptors, `OperatorStrategy` list, `Action` list, `StateMonitor`
list and `ReportNode`. On top of that it adds the `DynamicSecurityAnalysisParameters` and an
`EventModelsSupplier` (the events to apply during each simulation). `getDefault()` builds a
defaulted instance.

## Parameters

`DynamicSecurityAnalysisParameters` extends
`AbstractExtendable<DynamicSecurityAnalysisParameters>` (`VERSION` `"1.1"`). Rather than
embedding a load-flow parameters section, it embeds a `DynamicSimulationParameters` (the
underlying dynamic simulation configuration), a `debugDir`, and an inner
`ContingenciesParameters` class whose `contingenciesStartTime` (default `5.0`) sets the time
at which contingencies are applied during the simulation. It is loaded with `load()` /
`load(PlatformConfig)` (the `dynamic-security-analysis-default-parameters` configuration
module) and (de)serialized through `JsonDynamicSecurityAnalysisParameters` (with its
serializer/deserializer and `DynamicSecurityAnalysisJsonModule`).

## Result model

There is no dedicated result type: a run returns a `SecurityAnalysisReport` wrapping a
`SecurityAnalysisResult`, both imported from `security-analysis-api`. The pre-contingency
result, per-contingency results, operator-strategy results, limit violations and monitored
network state are exactly those documented on the
[security analysis](security-analysis.md) page.

## Execution and distribution

Like security analysis, this module supports local, distributed and forwarded execution,
parallel to the security-analysis framework: `DynamicSecurityAnalysisExecution` (with
`DynamicSecurityAnalysisExecutionImpl`, `DistributedDynamicSecurityAnalysisExecution` and
`ForwardedDynamicSecurityAnalysisExecution`), built by
`DynamicSecurityAnalysisExecutionBuilder` from a `DynamicSecurityAnalysisExecutionInput`
(which adds the dynamic-models and event-models sources to the inherited security-analysis
input), with the input assembled by a `DynamicSecurityAnalysisInputBuildStrategy`.

## Tool

`DynamicSecurityAnalysisTool` (`@AutoService(Tool.class)`) extends the generic
`AbstractSecurityAnalysisTool` and uses `DynamicSecurityAnalysisCommand` (which extends
`SecurityAnalysisCommand`) to provide the `dynamic-security-analysis` iTools command. Beyond
the inherited security-analysis options it adds `--dynamic-models-file` (required) and
`--event-models-file` (optional), the Groovy files supplying the dynamic and event models.

## Adding a dynamic-security-analysis implementation

Implement `DynamicSecurityAnalysisProvider`, annotate it with
`@AutoService(DynamicSecurityAnalysisProvider.class)`, optionally contribute a parameters
extension, and put the jar on the classpath. The `DynamicSecurityAnalysis` facade selects it
through the `dynamic-security-analysis` configuration module.
