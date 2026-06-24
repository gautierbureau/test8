# Dynamic simulation

The `dynamic-simulation` module defines the time-domain simulation API: it integrates the
dynamic behaviour of the network over a time interval, recording the evolution of chosen
variables as curves. The actual solver and the catalogue of dynamic models live in an
external repository ([PowSyBl Dynawo](https://github.com/powsybl/powsybl-dynawo)); this
module defines only the contract, the suppliers and the result model. The user-facing
documentation is in the [dynamic simulation](../../simulation/dynamic/index.md) section.

The module has three submodules:

| Submodule | Role |
|-----------|------|
| `dynamic-simulation-api` | The API: provider SPI, facade, suppliers, parameters and result model (`com.powsybl.dynamicsimulation`). |
| `dynamic-simulation-dsl` | Groovy suppliers and their extension SPIs (`com.powsybl.dynamicsimulation.groovy`). |
| `dynamic-simulation-tool` | The iTools commands (`com.powsybl.dynamicsimulation.tool`). |

## Provider SPI and facade

`DynamicSimulationProvider` is the SPI (extending `Versionable` and
`PlatformConfigNamedProvider`, discovered through `ServiceLoader`). Its computation method is
the widest of all the simulation modules, because a dynamic simulation needs more than a
network:

```java
CompletableFuture<DynamicSimulationResult> run(Network network,
        DynamicModelsSupplier dynamicModelsSupplier, EventModelsSupplier eventModelsSupplier,
        OutputVariablesSupplier outputVariablesSupplier, String workingVariantId,
        ComputationManager computationManager, DynamicSimulationParameters parameters,
        ReportNode reportNode);
```

It exposes the usual specific-parameter hooks plus `getSpecificParametersClass()` for the
parameters extension type.

`DynamicSimulation` is the facade. `find(name)` resolves a provider against the
`dynamic-simulation` configuration module and wraps it in an inner `DynamicSimulation.Runner`.
The many `run` / `runAsync` overloads fill in defaults for the optional inputs — empty
suppliers (`EventModelsSupplier.empty()`, `OutputVariablesSupplier.empty()`), the network's
working variant, `LocalComputationManager.getDefault()` and `ReportNode.NO_OP` — and return a
`DynamicSimulationResult`.

## Suppliers and models

The simulation inputs that a network alone cannot provide are passed as *suppliers*, sharing
the base `SimulatorInputSupplier<T>` interface (a `get(Network, ReportNode)` returning a
`List<T>`, plus an optional `getName()` tying the supplier to a particular provider):

- `DynamicModelsSupplier` → `List<DynamicModel>`: the dynamic models attached to network
  equipments. `DynamicModel` is a marker interface (its concrete models come from the
  provider).
- `EventModelsSupplier` → `List<EventModel>`: the events to apply during the simulation (e.g.
  a fault or a disconnection). `EventModel` exposes a `getStartTime()`.
- `OutputVariablesSupplier` → `List<OutputVariable>`: the variables to record. `OutputVariable`
  carries a `modelId`, a `variableName` and an `OutputVariable.OutputType` (`CURVE` to record
  the full time series, or `FINAL_STATE` to keep only the final scalar value).

## Parameters

`DynamicSimulationParameters` extends `AbstractExtendable<DynamicSimulationParameters>`
(`VERSION` `"1.1"`). Its common options are the simulation interval — `startTime` (default
`0`) and `stopTime` (default `10`) — and a `debugDir`. Implementation-specific options are
attached as extensions through the `DynamicSimulationParameters.ConfigLoader` extension point
(an `ExtensionConfigLoader`), discovered under the provider name
`dynamic-simulation-parameters`. The class is loaded with `load()` / `load(PlatformConfig)`
(the `dynamic-simulation-default-parameters` configuration module) and (de)serialized through
`JsonDynamicSimulationParameters` (with its `ExtensionSerializer` SPI and the
`DynamicSimulationParametersJsonModule`).

## Result model

`DynamicSimulationResult` is an interface. It exposes a `Status` (`SUCCESS` or `FAILURE`) and
a `statusText`, the recorded curves as a `Map<String, DoubleTimeSeries>` (with a
`getCurve(String)` shorthand), the `FINAL_STATE` variables as a `Map<String, Double>`
(`getFinalStateValues()`), and a timeline of `TimelineEvent` (`getTimeLine()`). `TimelineEvent`
is a record `(double time, String modelName, String message)`. `DynamicSimulationResultImpl`
is the default implementation, with a `createSuccessResult(...)` factory.

## dynamic-simulation-dsl

This submodule provides Groovy-based suppliers that build the model lists from a script.
`GroovyDynamicModelsSupplier`, `GroovyEventModelsSupplier` and `GroovyOutputVariablesSupplier`
each extend `AbstractGroovySupplier<T, R>` and implement the corresponding supplier interface,
evaluating a Groovy file against the network. The DSL itself is open: each supplier delegates
to a `GroovyExtension<T>` SPI — `DynamicModelGroovyExtension`, `EventModelGroovyExtension`,
`OutputVariableGroovyExtension` — whose `load(Binding, Consumer<T>, ReportNode)` method
registers the keywords a provider contributes to the script (`getModelNames()` lists the
available models). Extensions are discovered with `GroovyExtension.find(clazz, providerName)`.
`DynamicSimulationSupplierFactory` builds the right supplier from a path (only the `.groovy`
extension is supported), and `DynamicSimulationReports` provides the report nodes.

## dynamic-simulation-tool

`DynamicSimulationTool` (`@AutoService(Tool.class)`) provides the `dynamic-simulation` iTools
command (category `Computation`); its options read the case, the dynamic-models, event-models
and output-variables Groovy files and the JSON parameters, and write the result and log
files. `ListDynamicSimulationModelsTool` provides the `list-dynamic-simulation-models` command
to list the dynamic and event models a provider supports.

## Adding a dynamic-simulation implementation

Implement `DynamicSimulationProvider`, annotate it with
`@AutoService(DynamicSimulationProvider.class)`, and ship the dynamic/event-model catalogue as
Groovy extensions (the `*GroovyExtension` SPIs) plus, optionally, a parameters
`ConfigLoader`. Put the jar on the classpath; the `DynamicSimulation` facade selects the
provider through the `dynamic-simulation` configuration module.
