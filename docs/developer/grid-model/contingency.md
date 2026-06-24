# Contingency model

The `contingency` module describes **what can fail** on a network: the equipments that are simultaneously lost in a single event, the lists that group those events, and the providers that supply them to a simulation. It is the input side of a security analysis — the [action model](action.md) describes the remedial actions taken in response. This page covers the model classes, the contingency lists and their providers, the JSON serialization and the Groovy DSL.

The module sits in the [grid model layer](../grid-model.md), just above `iidm`: it depends on `iidm-api`, on `iidm-modification` (to turn a contingency into a network change) and on `iidm-criteria` (for criterion-based lists). It is split into two submodules:

| Submodule | Role |
|-----------|------|
| `contingency-api` | The contingency model, lists, providers, JSON serialization, the operator-strategy / condition / limit-violation classes. |
| `contingency-dsl` | A Groovy DSL describing contingencies as a script, plugged into the framework as a `ContingenciesProvider`. |

## The contingency model

### `Contingency` and `ContingencyElement`

A `Contingency` (package `com.powsybl.contingency`) has an id, an optional name and an ordered list of `ContingencyElement`s — the equipments lost together. It extends `AbstractExtendable<Contingency>`, so it can carry extensions. The class is mutable through `addElement` / `removeElement`, and exposes `getName()` as an `Optional<String>`.

A `ContingencyElement` is the interface implemented by every losable equipment kind:

```java
public interface ContingencyElement {
    String getId();
    ContingencyElementType getType();
    Tripping toModification();
}
```

The `getType()` discriminator is the `ContingencyElementType` enum, and `toModification()` is the bridge to the modification layer: it returns the `Tripping` (from `iidm-modification`) that simulates the loss of that equipment. The concrete element types are:

| Element type | Class | Tripping produced |
|--------------|-------|-------------------|
| `LINE` | `LineContingency` | `LineTripping` |
| `BRANCH` | `BranchContingency` | `BranchTripping` |
| `TIE_LINE` | `TieLineContingency` | `TieLineTripping` |
| `TWO_WINDINGS_TRANSFORMER` | `TwoWindingsTransformerContingency` | `TwoWindingsTransformerTripping` |
| `THREE_WINDINGS_TRANSFORMER` | `ThreeWindingsTransformerContingency` | `ThreeWindingsTransformerTripping` |
| `GENERATOR` | `GeneratorContingency` | `GeneratorTripping` |
| `LOAD` | `LoadContingency` | `LoadTripping` |
| `BATTERY` | `BatteryContingency` | `BatteryTripping` |
| `SHUNT_COMPENSATOR` | `ShuntCompensatorContingency` | `ShuntCompensatorTripping` |
| `STATIC_VAR_COMPENSATOR` | `StaticVarCompensatorContingency` | `StaticVarCompensatorTripping` |
| `HVDC_LINE` | `HvdcLineContingency` | `HvdcLineTripping` |
| `BUSBAR_SECTION` | `BusbarSectionContingency` | `BusbarSectionTripping` |
| `BUS` | `BusContingency` | `BusTripping` |
| `SWITCH` | `SwitchContingency` | `SwitchTripping` |
| `BOUNDARY_LINE` (`DANGLING_LINE`) | `BoundaryLineContingency` | `DanglingLineTripping` |
| `VOLTAGE_SOURCE_CONVERTER` | `VoltageSourceConverterContingency` | |
| `DC_LINE` / `DC_GROUND` / `DC_NODE` | `DcLineContingency` / `DcGroundContingency` / `DcNodeContingency` | |

(The exact `Tripping` class produced is given by each element's `toModification()`.) The `DANGLING_LINE` enum constant is kept only for deserialization backward-compatibility; it maps to the same `BoundaryLineContingency` as `BOUNDARY_LINE`.

Most elements extend `AbstractContingency`, which holds the id and implements `equals`/`hashCode` on `(id, type)`. Branch-like elements that can be tripped on a specific side instead extend `AbstractSidedContingency` and implement `SidedContingencyElement`, which adds an optional `getVoltageLevelId()`: when set, only the terminal on that voltage level is opened. The static helper `SidedContingencyElement.getContingencySide(Network, element)` resolves the `TwoSides` corresponding to that voltage level for lines, branches, tie lines, two-windings transformers and HVDC lines.

### Building contingencies

Contingencies are built fluently with `Contingency.builder(id)`, which returns a `ContingencyBuilder` exposing one `addXxx` method per element type (`addLine`, `addLine(id, voltageLevelId)`, `addGenerator`, `addTwoWindingsTransformer`, `addBus`, ...), plus `addName(name)` and `addIdentifiable(...)`:

```java
Contingency contingency = Contingency.builder("contingency1")
        .addLine("NHV1_NHV2_1")
        .addGenerator("GEN")
        .build();
```

For the common single-element case, `Contingency` also offers static factory shortcuts — `Contingency.line(id)`, `Contingency.generator(id)`, `Contingency.branch(id, voltageLevelId)`, `Contingency.busbarSection(id)`, and so on for every type.

`ContingencyElementFactory.create(Identifiable)` maps an arbitrary IIDM object to the right `ContingencyElement` by its concrete class (a `Map<Class<?>, Function<Identifiable<?>, ContingencyElement>>`), which is how `addIdentifiable` and the criterion/identifier lists turn network elements into contingencies. It throws a `PowsyblException` for an unsupported type (and special-cases `HvdcConverterStation`, mapping it to its `HvdcLineContingency`).

### Applying and validating a contingency

`Contingency.toModification()` returns a `NetworkModificationList` aggregating the `toModification()` of every element, so simulating a contingency simply reuses the tripping machinery of `iidm-modification`: applying that modification opens the relevant switches and disconnects the relevant terminals.

`Contingency.isValid(Network)` checks that every referenced equipment exists in the network (and, for sided elements, that the optional voltage-level id matches one of the equipment's sides), logging a warning and returning `false` otherwise. `ContingencyList.getValidContingencies(list, network)` filters a list down to the valid ones.

## Contingency lists

A `ContingencyList` (package `com.powsybl.contingency.list`) is a named, typed source of contingencies for a given network:

```java
public interface ContingencyList {
    String getName();
    String getType();
    List<Contingency> getContingencies(Network network);
}
```

The implementations differ in how `getContingencies(Network)` produces its result:

| Type string | Class | How contingencies are produced |
|-------------|-------|--------------------------------|
| `default` | `DefaultContingencyList` | An explicit, fixed list (filtered to the valid ones). |
| `list` | `ListOfContingencyLists` | The concatenation of several nested lists. |
| `identifier` | `IdentifierContingencyList` | Generated from `NetworkElementIdentifier`s (id-based, wildcard, voltage-levels-and-order, ...). |
| `injectionCriterion`, `lineCriterion`, `hvdcLineCriterion`, `tieLineCriterion`, `twoWindingsTransformerCriterion`, `threeWindingsTransformerCriterion` | `InjectionCriterionContingencyList`, `LineCriterionContingencyList`, ... | Generated by filtering the network with `iidm-criteria` criteria. |

The criterion-based lists derive from `AbstractEquipmentCriterionContingencyList` (and `AbstractLineCriterionContingencyList`): they stream the identifiables of a given `IdentifiableType` and keep those matching a country criterion (`SingleCountryCriterion` / `TwoCountriesCriterion`), a nominal-voltage criterion (`SingleNominalVoltageCriterion`, ...), `PropertyCriterion`s and an optional `RegexCriterion`, producing one single-element contingency per match. The `IdentifierContingencyList` resolves each `NetworkElementIdentifier` (from `iidm-api`'s `identifiers` package — `ID_BASED`, `ID_WITH_WILDCARDS`, `VOLTAGE_LEVELS_AND_ORDER`, `LIST`, `SUBSTATION_OR_VOLTAGE_LEVEL_EQUIPMENTS`) against the network, optionally grouping several elements under one `contingencyId`. Both reuse `ContingencyElementFactory` to turn matched identifiables into elements.

A `ContingencyList` is loaded from a file through the static `ContingencyList.load(Path)` / `load(filename, InputStream)`, which dispatch on the file extension to a `ContingencyListLoader` discovered through the plugin mechanism. `ContingencyListLoaderProvider` looks up the loader whose `getFormat()` matches the extension (using a `ServiceLoaderCache`), so adding a new on-disk contingency format is a matter of contributing a new `@AutoService(ContingencyListLoader.class)`. The JSON format is provided by `JsonContingencyListLoader` (format `"json"`).

## Providers (the SPI)

At the simulation boundary, contingencies are supplied through a `ContingenciesProvider`:

```java
public interface ContingenciesProvider {
    List<Contingency> getContingencies(Network network);
    default List<Contingency> getContingencies(Network network, Map<Class<?>, Object> contextObjects) { ... }
    default String asScript() { ... }
}
```

Providers are created by a `ContingenciesProviderFactory`, which is the actual plugin point listed in the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface): `create()`, plus `create(Path)` and `create(InputStream)` overloads to build a provider from a contingencies file. `ContingenciesProviders` is the entry-point helper: `newDefaultFactory()` resolves the configured factory through `ComponentDefaultConfig`, `emptyProvider()` returns the `EmptyContingencyListProvider`, and `newSubProvider(provider, Partition)` wraps a provider with a `SubContingenciesProvider` to split a contingency set across parallel computations.

Two factories ship with the module, both `@AutoService`-registered:

- `JsonContingenciesProviderFactory` — builds a provider from a JSON contingency file, delegating to `ContingencyList.load(...)`.
- `EmptyContingencyListProviderFactory` — supplies no contingency.

The Groovy DSL provider (below) and any external one (a database-backed list, for example) plug in the same way.

## `ContingencyContext`

`ContingencyContext` is a small value object used by security and sensitivity analysis to say *for which contingency* a piece of information (network results, limit reductions, sensitivity factors) is requested. It pairs an optional `contingencyId` with a `ContingencyContextType`:

| Context type | Meaning |
|--------------|---------|
| `ALL` | All contingencies plus the pre-contingency state. |
| `NONE` | The pre-contingency (base-case) state only. |
| `ONLY_CONTINGENCIES` | All contingencies, without the pre-contingency state. |
| `SPECIFIC` | A single contingency, whose id must be provided. |

It is created through static factories (`ContingencyContext.all()`, `none()`, `onlyContingencies()`, `specificContingency(id)`, or the generic `create(id, type)`); the `ALL`, `NONE` and `ONLY_CONTINGENCIES` instances are shared singletons. A `SPECIFIC` context with a null id is rejected. The class is also referenced by `OperatorStrategy`, which uses it to bind a strategy to its triggering contingency (see the [action model](action.md)).

## Operator strategies, conditions and limit violations

Although they are conceptually part of the remedial-action workflow, three groups of classes live in `contingency-api` so they can be reused without depending on the local action implementation:

- **`com.powsybl.contingency.strategy`** — `OperatorStrategy`, `ConditionalActions` and `OperatorStrategyList`, which describe *when* to apply remedial actions after a contingency.
- **`com.powsybl.contingency.strategy.condition`** — the `Condition` hierarchy (`TrueCondition`, the violation conditions and the threshold conditions) evaluated to trigger those actions.
- **`com.powsybl.contingency.violations`** — `LimitViolation`, `LimitViolationType`, `LimitViolationFilter`, `ViolationLocation` (with `NodeBreakerViolationLocation` / `BusBreakerViolationLocation`) and helpers, the representation of a limit overrun.

These are documented together with the action model on the [action page](action.md), since they are produced and consumed there.

## JSON serialization

JSON (de)serialization is centralized in `ContingencyJsonModule`, a Jackson `SimpleModule` that registers the serializers and deserializers for the whole module: `Contingency` and `ContingencyElement`, every `ContingencyList` flavour (default, list-of-lists, identifier and the criterion lists), `Criterion` and `NetworkElementIdentifier`, the `LimitViolation` / `ViolationLocation` classes, and the `OperatorStrategy` / `OperatorStrategyList` / `ConditionalActions` / `Condition` classes. A `ContingencyList` is polymorphic on its `type` field. A default list looks like:

```json
{
  "type" : "default",
  "version" : "1.1",
  "name" : "list",
  "contingencies" : [ {
    "id" : "contingency",
    "elements" : [
      { "id" : "NHV1_NHV2_1", "type" : "BRANCH" },
      { "id" : "NHV1_NHV2_2", "type" : "BRANCH" }
    ]
  }, {
    "id" : "contingency2",
    "elements" : [ { "id" : "GEN", "type" : "GENERATOR" } ]
  } ]
}
```

The format is versioned: `ContingencyList.VERSION` is `"1.1"` (version 1.1 renamed *dangling line* to *boundary line*), and `IdentifierContingencyList` carries its own `"1.3"` version. `JsonContingencyListLoader` is the `@AutoService`-registered loader that reads these files for the `"json"` format.

## The Groovy DSL (`contingency-dsl`)

`contingency-dsl` lets contingencies be expressed as a Groovy script rather than JSON, with each `contingency` block validated against the network. The user-facing syntax is described in the [contingency DSL](../../simulation/security/contingency-dsl.md) page; this section covers the implementation.

The core class is `ContingencyDslLoader` (Groovy, package `com.powsybl.contingency.dsl`), a subclass of `com.powsybl.dsl.DslLoader`. It binds a `contingency(String id, Closure)` keyword whose closure sets an `equipments` field; on evaluation, each listed equipment is resolved through `network.getIdentifiable(...)` and dispatched to the matching `ContingencyBuilder.addXxx` by its concrete type. Unknown ids or unsupported types make the contingency invalid (a warning is logged and it is skipped). `load(Network)` returns the resulting `List<Contingency>`, and `loadNotFoundElements(Network)` reports the unresolved ids per contingency. The DSL is extensible: `ContingencyDslExtension` (an `ExtendableDslExtension<Contingency>`) lets other modules contribute extra keywords to the contingency closure, discovered through `ServiceLoader`.

```groovy
contingency('contingency1') {
    equipments 'NHV1_NHV2_1'
}
```

This DSL is plugged into the framework as a provider: `GroovyDslContingenciesProvider` (extending `AbstractDslContingenciesProvider`) wraps a script read from a `Path` or `InputStream` and delegates `getContingencies` to a `ContingencyDslLoader`. `GroovyDslContingenciesProviderFactory` is the `@AutoService(ContingenciesProviderFactory.class)` that creates it — from the `dsl-file` property of the `groovy-dsl-contingencies` configuration module when no file is given explicitly. `GroovyContingencyList` and its `ContingencyListLoader` (`GroovyContingencyListLoader`) expose the same DSL through the `ContingencyList` abstraction.
