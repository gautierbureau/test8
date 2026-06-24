# Sensitivity analysis

The `sensitivity-analysis-api` module defines the sensitivity-analysis API: it computes how a
set of monitored quantities (*functions*, e.g. a branch flow) vary with respect to a set of
controlled quantities (*variables*, e.g. an injection or a phase shift), in the base case and
after contingencies. The algorithm is provided by an external implementation (typically
[PowSyBl Open Load Flow](https://github.com/powsybl/powsybl-open-loadflow)); this module
defines only the contract and the data model. The user-facing documentation is in the
[sensitivity analysis](../../simulation/sensitivity/index.md) section.

The code lives in `com.powsybl.sensitivity`, with JSON in `com.powsybl.sensitivity.json`.

## Provider SPI and facade

`SensitivityAnalysisProvider` is the SPI (extending `Versionable` and
`PlatformConfigNamedProvider`, discovered through `ServiceLoader`). Its single computation
method is **reader/writer oriented** rather than returning a value object:

```java
CompletableFuture<Void> run(Network network, String workingVariantId,
        SensitivityFactorReader factorReader, SensitivityResultWriter resultWriter,
        SensitivityAnalysisRunParameters runParameters);
```

This design is what sets sensitivity analysis apart from the other modules: because a run can
produce a very large number of values (factors × contingencies × strategies), the factors are
*streamed in* through a reader and the values are *streamed out* through a writer, avoiding a
large in-memory result unless one is explicitly assembled. The usual specific-parameter hooks
(`getSpecificParametersSerializer()`, `loadSpecificParameters(...)`,
`updateSpecificParameters(...)`) are also present.

`SensitivityAnalysis` is the facade. `find(name)` resolves a provider against the
`sensitivity-analysis` configuration module and wraps it in an inner
`SensitivityAnalysis.Runner`. It exposes two flavours of `run` / `runAsync`:

- the **streaming** variant, mirroring the SPI, taking a `SensitivityFactorReader` and a
  `SensitivityResultWriter` and returning `CompletableFuture<Void>`;
- the **convenience** variant, taking a `List<SensitivityFactor>` and returning a
  `SensitivityAnalysisResult` (internally it wraps the list in a `SensitivityFactorModelReader`
  and collects values into a `SensitivityResultModelWriter`).

## Factors and variables

A `SensitivityFactor` describes one sensitivity to compute. It pairs a *function* (a
`SensitivityFunctionType` such as `BRANCH_ACTIVE_POWER_1`, `BRANCH_CURRENT_1`,
`BRANCH_REACTIVE_POWER_1`, `BUS_VOLTAGE`, `BUS_REACTIVE_POWER`, with a `functionId`) with a
*variable* (a `SensitivityVariableType` such as `INJECTION_ACTIVE_POWER`,
`TRANSFORMER_PHASE`, `BUS_TARGET_VOLTAGE`, `HVDC_LINE_ACTIVE_POWER`, with a `variableId`).
The boolean `variableSet` flag marks whether the `variableId` refers to a single element or to
a `SensitivityVariableSet`, and a `ContingencyContext` (from the contingency module,
`ALL` / `NONE` / `SPECIFIC` / `ONLY_CONTINGENCIES`) scopes the factor to the base case and/or
contingencies.

A `SensitivityVariableSet` (id plus a collection of `WeightedSensitivityVariable`, each an id
and a `weight`) lets several elements act as a single weighted variable — for instance a GLSK
(generation shift key).

## Reader/writer model

The streaming inputs and outputs are abstracted by two interfaces:

- `SensitivityFactorReader` pushes factors to a `SensitivityFactorReader.Handler` via its
  `read(Handler)` method (`onFactor(functionType, functionId, variableType, variableId,
  variableSet, contingencyContext)`). Implementations: `SensitivityFactorModelReader` (from an
  in-memory `List<SensitivityFactor>`) and `SensitivityFactorJsonReader` (from a JSON file).
- `SensitivityResultWriter` receives the computed values: `writeSensitivityValue(factorIndex,
  contingencyIndex, operatorStrategyIndex, value, functionReference)` and
  `writeStateStatus(...)`. Implementations: `SensitivityResultModelWriter` (collects into
  memory), `SensitivityResultJsonWriter` (streams to JSON), `SensitivityResultCsvWriter`
  (streams to a `TableFormatter`).

Indexes (factor / contingency / operator-strategy) are used instead of ids throughout the
streaming API to keep each value compact.

## Result model

When assembled, `SensitivityAnalysisResult` holds the original factors, the list of
contingency ids and operator-strategy ids, the per-state `SensitivityStateStatus` entries,
and the flat list of `SensitivityValue`. Each `SensitivityValue` carries its factor /
contingency / operator-strategy indices, the sensitivity `value` and the
`functionReference` (the reference value of the monitored function). The result indexes
values by `SensitivityState` (a `(contingencyId, operatorStrategyId)` record, with a
`PRE_CONTINGENCY` constant), exposing helpers such as `getValues(SensitivityState)`,
`getPreContingencyValues()` and `getStateStatus(state)`. The per-state
`SensitivityAnalysisResult.Status` is `SUCCESS`, `FAILURE` or `NO_IMPACT`.

## Run parameters

The run context is grouped into `SensitivityAnalysisRunParameters` (fluent builder), which
carries the `SensitivityAnalysisParameters`, the `ComputationManager`, the `ReportNode`, the
list of `Contingency`, the list of `SensitivityVariableSet`, and — supporting remedial-action
sensitivities — the list of `OperatorStrategy` and `Action`. Getters lazily default the
parameters and computation manager; `getDefault()` builds a defaulted instance.

`SensitivityAnalysisParameters` extends `AbstractExtendable<SensitivityAnalysisParameters>`.
It embeds a `LoadFlowParameters` section, a set of value thresholds below which a sensitivity
is treated as negligible (`flowFlowSensitivityValueThreshold`,
`voltageVoltageSensitivityValueThreshold`, `flowVoltageSensitivityValueThreshold`,
`angleFlowSensitivityValueThreshold`) and a `SensitivityOperatorStrategiesCalculationMode`
(`NONE`, `CONTINGENCIES_AND_OPERATOR_STRATEGIES`, `ONLY_OPERATOR_STRATEGIES`). It is loaded
with `load()` / `load(PlatformConfig)` and (de)serialized through
`JsonSensitivityAnalysisParameters`.

## JSON

The `json` package registers all (de)serializers in `SensitivityJsonModule`: factors via
`SensitivityFactorJsonSerializer` / `SensitivityFactorJsonDeserializer`, values via
`SensitivityValueJsonSerializer` / `Deserializer`, variable sets, state statuses, the
parameters and the full result, plus the streaming `SensitivityFactorJsonReader` /
`SensitivityResultJsonWriter` helpers described above.

## Tool

`SensitivityAnalysisTool` (`@AutoService(Tool.class)`) provides the `sensitivity-analysis`
iTools command.

## Adding a sensitivity-analysis implementation

Implement `SensitivityAnalysisProvider` (its reader/writer `run`), annotate with
`@AutoService(SensitivityAnalysisProvider.class)`, optionally contribute a parameters
extension, and put the jar on the classpath. The `SensitivityAnalysis` facade selects it
through the `sensitivity-analysis` configuration module.
