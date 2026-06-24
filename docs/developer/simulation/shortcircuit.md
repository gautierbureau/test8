# Short-circuit analysis

The `shortcircuit-api` module defines the short-circuit (fault) analysis API: given a network
and a list of faults to apply, it computes the resulting fault currents and voltages, the
feeder contributions and any limit violations. The numerical engine is external (for example
[PowSyBl Dynawo](https://github.com/powsybl/powsybl-dynawo)); this module defines only the
contract and the data model. The user-facing documentation is in the
[short circuit](../../simulation/shortcircuit/index.md) section.

The code lives in `com.powsybl.shortcircuit`, with subpackages `json`, `converter` and
`tools`.

## Provider SPI and facade

`ShortCircuitAnalysisProvider` is the SPI (extending `Versionable` and
`PlatformConfigNamedProvider`, discovered through `ServiceLoader`). Its computation method
takes the network, the faults and both global and per-fault parameters:

```java
CompletableFuture<ShortCircuitAnalysisResult> run(Network network, List<Fault> faults,
        ShortCircuitParameters parameters, ComputationManager computationManager,
        List<FaultParameters> faultParameters, ReportNode reportNode);
```

It also carries the usual specific-parameter hooks (`getSpecificParametersSerializer()`,
`loadSpecificParameters(...)`, `getSpecificParameters(...)`).

`ShortCircuitAnalysis` is the facade. `find(name)` resolves a provider against the
`shortcircuits-analysis` configuration module and wraps it in an inner
`ShortCircuitAnalysis.Runner`. Unlike the load-flow facade, this module did not adopt a
single `*RunParameters` holder: the static and runner `run` / `runAsync` overloads pass the
fault list, `ShortCircuitParameters`, `ComputationManager`, the `List<FaultParameters>` and
the `ReportNode` directly, with shorter overloads filling in defaults. All return a
`ShortCircuitAnalysisResult`.

## Faults

The faults to compute are described by the `Fault` interface and its hierarchy. A `Fault`
carries an `id`, the `elementId` of the faulted equipment, a resistance and reactance to
ground (`getRToGround()` / `getXToGround()`), and three enums:

- `Fault.Type` — `BUS` or `BRANCH`;
- `Fault.ConnectionType` — `SERIES` or `PARALLEL`;
- `Fault.FaultType` — `THREE_PHASE` or `SINGLE_PHASE`.

The two concrete types share the package-private `AbstractFault`: `BusFault` (a fault at a
bus) and `BranchFault`, which adds a `proportionalLocation` (the percentage along the branch,
measured from side ONE, at which the fault occurs). `Fault` also offers static
`read(Path)` / `write(List<Fault>, Path)` JSON helpers.

## Parameters

`ShortCircuitParameters` extends `AbstractExtendable<ShortCircuitParameters>` and holds the
global options, each with a `DEFAULT_*` constant in `ShortCircuitConstants`:

| Option | Notes |
|--------|-------|
| `studyType` | `StudyType` enum — `SUB_TRANSIENT`, `TRANSIENT`, `STEADY_STATE` (default `TRANSIENT`), selecting the stage of the short-circuit being studied. |
| `withFortescueResult` | whether to produce detailed symmetrical-component (Fortescue) results rather than magnitude-only (default `true`). |
| `withFeederResult`, `withVoltageResult`, `withLimitViolations` | which result parts to populate. |
| `withLoads`, `withShuntCompensators`, `withVSCConverterStations`, `withNeutralPosition` | which equipment to include in the model. |
| `minVoltageDropProportionalThreshold`, `subTransientCoefficient` | numerical settings (the sub-transient coefficient scales the sub-transient reactance). |
| `initialVoltageProfileMode` | `InitialVoltageProfileMode` enum — `NOMINAL`, `PREVIOUS_VALUE`, `CONFIGURED` (the last requiring a list of `VoltageRange`). |
| `voltageRanges`, `detailedReport`, `debugDir` | configured voltage profile, log verbosity, debug dump directory. |

It is loaded with `load()` / `load(PlatformConfig)` (the `short-circuit-parameters`
configuration module) and validated with `validate()` (which checks that `CONFIGURED` mode is
accompanied by voltage ranges). `FaultParameters` carries the same set of options but
*per fault* (keyed by the fault `id`), so individual faults can override the global settings;
it too has static `read` / `write` JSON helpers and a `validate()`.

A `VoltageRange` (used by the `CONFIGURED` profile) maps a nominal-voltage range to a
`rangeCoefficient` (bounds 0.8–1.2) and an optional overriding `voltage`;
`VoltageRange.checkVoltageRange(...)` ensures the ranges do not overlap.

## Result model

`ShortCircuitAnalysisResult` (extending `AbstractExtendable`) aggregates a list of
`FaultResult`, looked up by fault id (`getFaultResult(id)`) or by element id
(`getFaultResults(elementId)`).

`FaultResult` is an interface (itself `Extendable`) with a `FaultResult.Status`
(`SUCCESS`, `NO_SHORT_CIRCUIT_DATA`, `SOLVER_FAILURE`, `FAILURE`). It exposes the `Fault`,
the short-circuit power (MVA), the time constant, the list of `FeederResult`, the list of
`ShortCircuitBusResults` (voltage results) and the list of `LimitViolation`. The concrete
results share the package-private `AbstractFaultResult` and split along the magnitude /
Fortescue axis:

- `MagnitudeFaultResult` — scalar `current` (A) and `voltage` (kV);
- `FortescueFaultResult` — `current` and `voltage` as `FortescueValue` (full symmetrical
  components);
- `FailedFaultResult` — a fault whose computation failed, carrying only the failure status.

The same magnitude/Fortescue split applies to the sub-results: `FeederResult`
(`MagnitudeFeederResult` / `FortescueFeederResult`, each tying a `connectableId` and a side to
a contribution current) and `ShortCircuitBusResults`
(`MagnitudeShortCircuitBusResults` / `FortescueShortCircuitBusResults`, each carrying the
voltage-level and bus id, the initial voltage magnitude and the proportional voltage drop).

`FortescueValue` represents a three-phase quantity through its symmetrical components: a
positive-, negative- and zero-sequence magnitude and angle, with a `toThreePhaseValue()`
conversion to the per-phase representation.

## JSON and export

The `json` package centralises (de)serialization in `ShortCircuitAnalysisJsonModule`, with
`JsonShortCircuitParameters` as the parameters helper (`read` / `write` / `update`) and
dedicated serializers for faults, fault parameters, results, feeder/bus results,
`FortescueValue` and `VoltageRange`.

The `converter` package exports a `ShortCircuitAnalysisResult` to external formats through the
`ShortCircuitAnalysisResultExporter` SPI (`getFormat()`, `getComment()`, `export(result,
writer, network)`), discovered and dispatched by `ShortCircuitAnalysisResultExporters`. Three
exporters are provided (each `@AutoService`): `JsonShortCircuitAnalysisResultExporter`
(`JSON`), `CsvShortCircuitAnalysisResultExporter` (`CSV`) and
`AsciiShortCircuitAnalysisResultExporter` (`ASCII`), the latter two sharing
`AbstractTableShortCircuitAnalysisResultExporter`.

## Tool

`ShortCircuitAnalysisTool` (in the `tools` package, `@AutoService(Tool.class)`) provides the
`shortcircuit` iTools command; its options (in `ShortCircuitAnalysisToolConstants`) read the
case, the fault list, the parameters and fault-parameters files, and write the result in a
chosen export format.

## Adding a short-circuit implementation

Implement `ShortCircuitAnalysisProvider`, annotate it with
`@AutoService(ShortCircuitAnalysisProvider.class)`, optionally contribute a parameters
extension, and put the jar on the classpath. The `ShortCircuitAnalysis` facade selects it
through the `shortcircuits-analysis` configuration module.
