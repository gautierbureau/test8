# Security analysis

The `security-analysis` module defines the security-analysis API: given a network and a list
of contingencies, it computes the limit violations in the base case (N) and after each
contingency (N-1), optionally applying remedial actions (operator strategies). As with the
other simulation APIs, the numerical work is delegated to an external provider (typically
[PowSyBl Open Load Flow](https://github.com/powsybl/powsybl-open-loadflow)); this module only
defines the contract, the inputs and the result model. The user-facing documentation is in
the [security analysis](../../simulation/security/index.md) section.

The module contains a single submodule, `security-analysis-api`, with the API in
`com.powsybl.security` and a rich set of subpackages (`results`, `monitor`, `interceptors`,
`detectors`, `limitreduction`, `distributed`, `execution`, `preprocessor`, `comparator`,
`converter`, `extensions`, `json`, `tools`).

## Provider SPI and facade

`SecurityAnalysisProvider` is the SPI; like the others it extends `Versionable` and
`PlatformConfigNamedProvider` and is discovered through `ServiceLoader`. Its core method is:

```java
CompletableFuture<SecurityAnalysisReport> run(Network network, String workingVariantId,
        ContingenciesProvider contingenciesProvider, SecurityAnalysisRunParameters runParameters);
```

Besides the usual specific-parameter hooks (`getSpecificParametersSerializer()`,
`loadSpecificParameters(...)`, `updateSpecificParameters(...)`,
`getSpecificParametersNames()`), it exposes `getLoadFlowProviderName()` — because a security
analysis is built on top of a load flow, a provider can declare which load-flow
implementation it relies on.

`SecurityAnalysis` is the facade. `find(name)` resolves a provider against the
`security-analysis` configuration module and wraps it in an inner `SecurityAnalysis.Runner`.
The static and runner `run(...)` / `runAsync(...)` overloads take a network, a working
variant and either a `ContingenciesProvider` or a `List<Contingency>`, plus a
`SecurityAnalysisRunParameters`, and return a `SecurityAnalysisReport` (the async variant a
`CompletableFuture` of it).

### Run parameters

The run context is grouped into `SecurityAnalysisRunParameters`, which extends the generic
`AbstractSecurityAnalysisRunParameters<T>`. The abstract base carries the inputs shared by
plain and dynamic security analysis: a `LimitViolationFilter` (defaulting to
`LimitViolationFilter.load()`), a `ComputationManager`, a list of
`SecurityAnalysisInterceptor`, a list of `OperatorStrategy`, a list of `Action`, a list of
`StateMonitor` and a `ReportNode`, each with `set*` and `add*` accessors. The concrete
subclass adds the `SecurityAnalysisParameters` and a list of `LimitReduction`.
`SecurityAnalysisRunParameters.getDefault()` builds a fully defaulted instance.

### Parameters

`SecurityAnalysisParameters` extends `AbstractExtendable<SecurityAnalysisParameters>`
(`VERSION` `"1.2"`). Beyond provider-specific extensions, it embeds a `LoadFlowParameters`
section (the load flow that underlies the analysis), an `IncreasedViolationsParameters` inner
class — the thresholds (`flowProportionalThreshold`, low/high-voltage proportional and
absolute thresholds) above which a post-contingency violation is reported as *increased* with
respect to the base case — and the `intermediateResultsInOperatorStrategy` flag. It is loaded
with `SecurityAnalysisParameters.load()` / `load(PlatformConfig)` (the
`security-analysis-default-parameters` configuration module) and (de)serialized through
`JsonSecurityAnalysisParameters`.

## Result model

A run returns a `SecurityAnalysisReport`, a thin wrapper (extending `AbstractExtendable`)
holding the `SecurityAnalysisResult` and, optionally, the raw log bytes
(`getLogBytes()` returns an `Optional<byte[]>`).

`SecurityAnalysisResult` (also `AbstractExtendable`) is composed of:

- a `PreContingencyResult` — the base-case (N) result, carrying the load-flow component
  status (`LoadFlowResult.ComponentResult.Status`);
- a list of `PostContingencyResult` — one per contingency, each holding its `Contingency`, a
  `PostContingencyComputationStatus`, and a `ConnectivityResult` describing the islanding
  caused by the contingency;
- a list of `OperatorStrategyResult` — the outcome of each remedial-action strategy, broken
  into `ConditionalActionsResult` stages;
- a `NetworkMetadata`.

The detailed model lives in the `results` package. The `*ContingencyResult` classes share an
`AbstractContingencyResult` carrying a `LimitViolationsResult` (the list of `LimitViolation`
plus the `actionsTaken`) and a `NetworkResult`. `NetworkResult` aggregates the monitored
state as maps of `BranchResult` (p/q/i on both sides plus a `flowTransfer`), `BusResult`
(voltage magnitude and angle) and `ThreeWindingsTransformerResult` (p/q/i on all three legs).
`ConnectivityResult` reports created connected/synchronous component counts and the
disconnected load/generation active power and elements.

`PostContingencyComputationStatus` is the per-contingency status enum: `CONVERGED`,
`MAX_ITERATION_REACHED`, `SOLVER_FAILED`, `FAILED`, `NO_IMPACT`.

`SecurityAnalysisResultBuilder` assembles a result incrementally during a run (working with
the interceptors), and `SecurityAnalysisResultMerger` combines partial results — both
essential for the distributed execution described below.

## Monitors

A `StateMonitor` declares which equipment state to record in the result, scoped by a
`ContingencyContext` (so monitoring can be applied to the base case, to all contingencies, or
to a specific one). It lists `branchIds`, `voltageLevelIds` (for buses) and
`threeWindingsTransformerIds`, and can be `merge`d with another. `StateMonitorIndex` indexes
the configured monitors by contingency-context type (an `allStateMonitor`, a
`noneStateMonitor` and a map of `specificStateMonitors`) so the provider can look up what to
record for a given situation. The recorded state ends up in the `NetworkResult` of each
contingency result.

## Interceptors

The `interceptors` package lets code hook into the computation as results are produced.
`SecurityAnalysisInterceptor` defines callbacks `onPreContingencyResult`,
`onPostContingencyResult`, `onSecurityAnalysisResult`, and two `onLimitViolation` overloads
(base-case and post-contingency), each receiving a `SecurityAnalysisResultContext`.
`DefaultSecurityAnalysisInterceptor` provides no-op implementations; `CurrentLimitViolationInterceptor`
extends it to attach the `ActivePowerExtension` / `CurrentExtension` to current-limit
violations. Interceptors are passed in through the run parameters.

## Operator strategies, actions and limit reductions

Remedial actions reuse the grid-model layer: `OperatorStrategy` comes from
`com.powsybl.contingency.strategy`, `Action` from the `com.powsybl.action` module, and
conditions from the same packages. They are supplied through the run parameters
(`setOperatorStrategies`, `setActions`) and their outcome is reported in the
`OperatorStrategyResult` / `ConditionalActionsResult` chain.

The `detectors` package owns limit-violation detection: `LimitViolationDetector` (with
`checkCurrent` / `checkVoltage` / `checkPower` callbacks) and its `DefaultLimitViolationDetector`.
The `limitreduction` package lets users relax (or tighten) limits before detection: a
`LimitReduction` couples a `LimitType` (`CURRENT`, `ACTIVE_POWER`, `APPARENT_POWER`), a
reduction `value`, a `ContingencyContext` and network-element / limit-duration criteria;
`LimitReductionList` (`VERSION` `"1.2"`, supporting factors above 1.0) groups them, and
`DefaultLimitReductionsApplier` applies them. See the
[limit reductions](../../simulation/security/limit-reductions.md) user page.

## Distributed and external execution

A distinctive feature of this module is its support for running a security analysis split
across several tasks or forwarded to an external `itools` process. The `execution` and
`distributed` packages model this:

- `SecurityAnalysisExecution` is the execution SPI:
  `CompletableFuture<SecurityAnalysisReport> execute(ComputationManager, SecurityAnalysisExecutionInput)`.
- `SecurityAnalysisExecutionImpl` runs the analysis locally (in-process), driven by a
  `SecurityAnalysisInputBuildStrategy`.
- `DistributedSecurityAnalysisExecution` splits the contingency list into a `Partition` and
  spawns several subtasks through the `ComputationManager`, each handed a
  `SubContingenciesProvider` (a `ContingenciesProvider` returning only its slice), and merges
  the partial reports with `SecurityAnalysisResultMerger`.
- `ForwardedSecurityAnalysisExecution` forwards the whole computation to an external process.

`SecurityAnalysisExecutionInput` and `SecurityAnalysisInput` carry the resolved inputs
(network variant, filter, contingencies source, result extensions, violation types and the
`SecurityAnalysisParameters`). The `preprocessor` package
(`SecurityAnalysisPreprocessor` / `SecurityAnalysisPreprocessorFactory`) lets a configurable
step adjust the input (for example to read contingencies from a file) before execution.

## Comparator and tools

The `comparator` package compares two results: `SecurityAnalysisResultEquivalence` performs a
tolerance-based comparison, exposed as the `compare-security-analysis-results` iTools command
by `CompareSecurityAnalysisResultsTool`.

The main command is `SecurityAnalysisTool` (`@AutoService(Tool.class)`), which extends the
generic `AbstractSecurityAnalysisTool` and uses `SecurityAnalysisCommand` (command name
`security-analysis`). Its options cover the case file, parameters, contingencies, limit
types, output file/format, the `--task` / `--task-count` distributed options and the
`--external` forwarded option, mirroring the execution framework above.

## Adding a security-analysis implementation

Implement `SecurityAnalysisProvider`, annotate it with `@AutoService(SecurityAnalysisProvider.class)`,
optionally contribute a parameters extension, and put the jar on the classpath. The
`SecurityAnalysis` facade selects it through the `security-analysis` configuration module.
