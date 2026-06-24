# Grid model layer

The grid model layer is the heart of PowSyBl Core. It is built around the **IIDM** model (the *iTesla Internal Data Model*), the single in-memory representation of a power network that every importer, exporter and simulation works on. Around it sit the **contingency** model (what can fail) and the **action** model (the remedial actions and operator strategies applied in response). This page describes how these modules are organized and the main design patterns they rely on. For the functional description of the model itself, see the [grid model](../grid_model/index.md) user documentation.

```{toctree}
:hidden:
grid-model/iidm
grid-model/contingency
grid-model/action
```

## The `iidm` module

The `iidm` module is split into several submodules so that the API is cleanly separated from its implementations and from optional features.

| Submodule | Role |
|-----------|------|
| `iidm-api` | The model interfaces (`Network`, `VoltageLevel`, `Generator`, ...) and their adders. |
| `iidm-impl` | The default in-memory implementation. |
| `iidm-extensions` | Standard extensions attached to model objects. |
| `iidm-serde` | XML / JSON / binary serialization (XIIDM / JIIDM / BIIDM). |
| `iidm-modification` | Reusable network modifications. |
| `iidm-criteria` | Criteria to filter and select network elements. |
| `iidm-geodata` | Loading of geographical positions as extensions. |
| `iidm-reducer` | Reduction of a network to a subset. |
| `iidm-comparator` | Comparison of two network states. |
| `iidm-tck` | The Technology Compatibility Kit validating alternative implementations. |
| `iidm-test` | Sample networks used across the test suites. |
| `iidm-scripting` | Groovy extensions easing scripting on the model. |

### API vs implementation

`iidm-api` contains only interfaces and a few abstract helpers: it defines *what* a network is, not *how* it is stored. The top-level type is `Network`, a container of `Substation`, `VoltageLevel` and the equipments connected to them (`Generator`, `Load`, `Battery`, `ShuntCompensator`, `StaticVarCompensator`, `Line`, `TwoWindingsTransformer`, `ThreeWindingsTransformer`, `HvdcLine`, ...). All identified objects share the `Identifiable` base interface, and equipments connected to the grid implement `Connectable` and expose one or more `Terminal`s. Utility classes used by every implementation live under `com.powsybl.iidm.network.util` (`SV`, `BranchData`, `TwtData`, `Networks`, ...).

`iidm-impl` provides the default in-memory implementation of those interfaces (`NetworkImpl`, `SubnetworkImpl`, the topology models, ...). It is obtained through a `NetworkFactory`: the implementation is discovered through the plugin mechanism (a `NetworkFactoryService`), so `NetworkFactory.findDefault()` returns the in-memory factory without any explicit reference to `iidm-impl`. This indirection is what allows other implementations (for instance a database-backed one) to be substituted without touching the rest of the codebase.

### The adder pattern

Network objects are never created with constructors. Instead, each object exposes a factory method that returns an *adder* — a fluent builder whose setters return the adder itself and whose terminal `add()` method validates the attributes and creates the object. For example:

```java
Generator g = voltageLevel.newGenerator()
        .setId("GEN")
        .setNode(1)
        .setTargetP(100.0)
        .setVoltageRegulatorOn(true)
        .setTargetV(400.0)
        .setMinP(0.0)
        .setMaxP(150.0)
        .add();
```

Every creatable type has a matching adder interface in `iidm-api` (`SubstationAdder`, `VoltageLevelAdder`, `GeneratorAdder`, `LineAdder`, `TieLineAdder`, ...), all ultimately deriving from `IdentifiableAdder`. The adder enforces required attributes and consistency at `add()` time, so a half-built object can never enter the network. The same pattern is reused for extensions (`object.newExtension(XxxAdder.class)...add()`) and for sub-objects such as reactive limits (`ReactiveCapabilityCurveAdder`) and tap changers (`RatioTapChangerAdder`, `PhaseTapChangerAdder`).

### Variants (state management)

A single `Network` can hold several **variants** (states): different sets of state values (flows, voltages, tap positions, switch positions, ...) sharing the same topology and structural data. Variants are managed through the `VariantManager` returned by `network.getVariantManager()`. The first variant is named `VariantManagerConstants.INITIAL_VARIANT_ID` (`"InitialState"`). The manager exposes:

- `getVariantIds()` / `getWorkingVariantId()` / `setWorkingVariant(String)` to list and select the active variant;
- `cloneVariant(source, target)` to duplicate a state, and `removeVariant(String)` to drop one;
- `allowVariantMultiThreadAccess(boolean)` to enable per-thread working variants.

In `iidm-impl`, state-dependent attributes are stored in a `VariantArray` indexed by variant, so cloning a variant is cheap and switching the working variant only changes which slot is read. This is what lets a security analysis explore many contingency states of the same network in parallel without copying the whole structure.

### Topology model and views

A `VoltageLevel` carries a **topology model** describing how its equipments are electrically connected. Two model kinds exist, selected by `TopologyKind`:

- **node/breaker** — the most detailed form. Equipments, busbar sections and switches (breakers, disconnectors) are attached to integer *connection nodes*, and the topology is stored as a graph (nodes as vertices, switches as edges). This is the level of detail found in substation diagrams and detailed formats.
- **bus/breaker** — an aggregated form made of buses and breakers, where a bus aggregates busbar sections joined by closed switches. It is convenient when the source data only has bus/branch information.

On top of the model, the same topology can be read through three **views**, ordered from most to least detailed: the node/breaker view (`getNodeBreakerView()`), the bus/breaker view (`getBusBreakerView()`) and the bus view (`getBusView()`). A view more detailed than the underlying model is *not available* (calling it throws); a view at the same level is *modifiable*; a coarser view is *read-only* and computed from the model. The bus/breaker and bus views are variant-dependent because computed buses depend on switch positions. Switches flagged with `Switch.isRetained()` survive the reduction to the bus/breaker view. In `iidm-impl` these kinds are implemented by `NodeBreakerTopologyModel` and `BusBreakerTopologyModel` (both extending `AbstractTopologyModel`), and `VoltageLevel.convertToTopology(TopologyKind)` switches a voltage level from one model to the other.

`Network` also supports sub-networks: a network may be the merge of several sub-networks that keep their identity. See [network and subnetwork](../grid_model/network_subnetwork.md) for details.

## Serialization (`iidm-serde`)

`iidm-serde` reads and writes a `Network` in the native IIDM serialization, in three interchangeable formats backed by a common tree-data abstraction:

| Format | Code | Backing tree format |
|--------|------|---------------------|
| XIIDM | `XML` | XML |
| JIIDM | `JSON` | JSON |
| BIIDM | `BIN` | binary |

The central class is `NetworkSerDe`, which holds the read/write logic shared by all three formats and validates against the IIDM schema. Each format also exposes an `Importer` / `Exporter` pair discovered through the plugin mechanism: `XMLImporter`/`XMLExporter`, `JsonImporter`/`JsonExporter` and `BinaryImporter`/`BinaryExporter`, all built on `AbstractTreeDataImporter` / `AbstractTreeDataExporter`. In practice these are reached transparently through the generic `Network.read(...)` and `Network.write(...)` entry points of `iidm-api`.

The model is versioned by the `IidmVersion` enum (`V_1_0` up to the current `V_1_17`, the latter held by `IidmSerDeConstants.CURRENT_IIDM_VERSION`); each version maps to an XML namespace and an XSD, so older files remain readable. Extensions are serialized by their own providers: an extension contributes an `ExtensionSerDe` (XML / binary) or `ExtensionJsonSerializer` (JSON), discovered through the plugin mechanism, with versioned extensions deriving from `AbstractVersionableNetworkExtensionSerDe`. The `iidm-serde.anonymizer` package (`Anonymizer`, `SimpleAnonymizer`) can mask identifiers on export.

## Network modifications (`iidm-modification`)

`iidm-modification` provides reusable, named transformations of a network. They all implement the `NetworkModification` interface — essentially an `apply(Network, ...)` operation (with overloads taking a `ComputationManager`, a `ReportNode`, a `NamingStrategy` and dry-run / throw-on-error flags) plus `hasImpactOnNetwork(Network)`. Concrete modifications extend `AbstractNetworkModification`, and `NetworkModificationList` composes several into one.

The modifications are grouped by concern:

- **topology** (`topology` package) — structural changes such as `CreateCouplingDevice`, `CreateVoltageLevelTopology`, `ConnectFeedersToBusbarSections`, `RemoveFeederBay`, `RemoveHvdcLine`, `MoveFeederBay`.
- **tripping** (`tripping` package) — disconnecting equipment by simulating the opening of the switches around it. The `Tripping` interface (extending `NetworkModification`) computes the set of switches to open and terminals to disconnect; implementations include `BranchTripping`, `LineTripping`, `GeneratorTripping`, `LoadTripping`, `BusbarSectionTripping`, `HvdcLineTripping`, `SwitchTripping`, ... This is the mechanism used to apply contingencies (see below).
- **scalable** (`scalable` package) — the `Scalable` abstraction, which represents an injection (or a combination of injections) whose active power can be increased or decreased. `GeneratorScalable` and `LoadScalable` scale a single injection; `ProportionalScalable`, `StackScalable` and `UpDownScalable` combine several, and a `ScalingConvention` (generator vs. load) fixes the sign convention. Scalables are used to shift generation/consumption, for example to build study cases.
- **tap changers** (`tapchanger` package) — `RatioTapPositionModification` and `PhaseTapPositionModification`.
- **equipment parameter changes** (root package) — `GeneratorModification`, `LoadModification`, `OpenSwitch`, `CloseSwitch`, `ShuntCompensatorModification`, `StaticVarCompensatorModification`, ...

## The other `iidm` submodules

- **`iidm-extensions`** holds the standard extensions, each defined as an `Extension<T>` with its adder. Examples include `ActivePowerControl`, `ConnectablePosition` and `BusbarSectionPosition` (for diagrams), `GeneratorShortCircuit` and `LineFortescue` (short-circuit data), `BranchObservability` and discrete measurements (state-estimation data), `CoordinatedReactiveControl` and `SecondaryVoltageControl`. They enrich the model without changing the core interfaces; their serializers live in `iidm-serde`.
- **`iidm-criteria`** defines criteria to select network elements by their characteristics. A `NetworkElementCriterion` targets a kind of element (line, transformer, identifiable, ...) and combines `Criterion`s such as `SingleCountryCriterion` / `TwoCountriesCriterion`, the nominal-voltage criteria (`SingleNominalVoltageCriterion`, `TwoNominalVoltageCriterion`, ...), `PropertyCriterion` and `RegexCriterion`. These criteria are reused by criterion-based contingency lists and by limit/duration filtering.
- **`iidm-geodata`** loads geographical data (substation points, line paths) from GeoJSON and applies it to the network as `SubstationPosition` and `LinePosition` extensions, via parsers and adders such as `GeoJsonDataParser`, `GeoJsonDataAdder` and `NetworkGeoDataExtensionsAdder`, using `Coordinate` for latitude/longitude.
- **`iidm-reducer`** shrinks a network to a subset of interest. `NetworkReducer` (built through `DefaultNetworkReducerBuilder`) keeps the substations and voltage levels accepted by a `NetworkPredicate` (`NominalVoltageNetworkPredicate`, `SubNetworkPredicate`, `IdentifierNetworkPredicate`, ...), with `ReductionOptions` controlling boundary handling.
- **`iidm-comparator`** compares two states of a network. `NetworkStateComparator` takes a network and the name of an alternate variant and produces an Excel report of the differences in voltages, angles, flows and tap positions per equipment type.
- **`iidm-tck`** is the Technology Compatibility Kit: a suite of abstract tests (`AbstractNodeBreakerTest`, `AbstractTopologyTraverserTest`, ...) that any IIDM implementation must pass to prove it conforms to the API contract.
- **`iidm-test`** provides ready-made sample networks used throughout the test suites and documentation, exposed as factories such as `EurostagTutorialExample1Factory`, `FourSubstationsNodeBreakerFactory`, `BatteryNetworkFactory` and `SvcTestCaseFactory`.
- **`iidm-scripting`** adds Groovy extension methods to make the model more pleasant to script (concise property access and adder syntax on `Network`, `VoltageLevel`, transformers, ...) and a `GroovyScriptPostProcessor` to run a Groovy script as an import post-processor.

## The `contingency` module

The `contingency` module describes *what can fail* on a network. A `Contingency` has an id, an optional name and a list of `ContingencyElement`s — the equipments that are simultaneously lost. It is built fluently through `Contingency.builder(id)` (`addLine`, `addGenerator`, ...) or with static factory shortcuts like `Contingency.line(id)`. Each contingency element type corresponds to a piece of equipment: `LineContingency`, `BranchContingency`, `GeneratorContingency`, `LoadContingency`, `BusbarSectionContingency`, `HvdcLineContingency`, `TwoWindingsTransformerContingency`, `ThreeWindingsTransformerContingency`, `TieLineContingency`, `SwitchContingency`, `ShuntCompensatorContingency`, `StaticVarCompensatorContingency`, `BusContingency`, `BatteryContingency`, and the DC-side variants. Applying a contingency to a network turns each element into the corresponding `Tripping` modification from `iidm-modification`, so simulating a contingency reuses the tripping machinery described above.

Contingencies are grouped into a `ContingencyList`. The simplest implementation is `DefaultContingencyList` (an explicit list); criterion-based lists (built on `iidm-criteria`) generate contingencies from filters on the network, and `ListOfContingencyLists` aggregates several lists. At the SPI level, a `ContingenciesProvider` supplies the contingencies for a given network, and is created by a `ContingenciesProviderFactory` (a plugin point listed in the developer guide). The `contingency-dsl` submodule adds a Groovy DSL: `ContingencyDslLoader` evaluates a script and returns the list of `Contingency` objects, validating that the referenced equipments exist in the network.

## Remedial actions: `action-api` and `action-ial`

`action-api` models the **remedial actions** an operator can take and the **operator strategies** deciding when to take them.

An `Action` is a single elementary remedial action; every action carries an id and a `toModification()` method that turns it into an `iidm-modification`, so applying an action ultimately reuses the modification layer. The available action types include `SwitchAction`, `TerminalsConnectionAction` (connect/disconnect an equipment), `GeneratorAction`, `LoadAction` and `PercentChangeLoadAction`, `ShuntCompensatorPositionAction`, `StaticVarCompensatorAction`, `HvdcAction`, the tap-changer actions (`PhaseTapChangerTapPositionAction`, `RatioTapChangerTapPositionAction`, `PhaseTapChangerRegulationAction`, `RatioTapChangerRegulationAction`), `AreaInterchangeTargetAction`, and `MultipleActionsAction` to compose several. Each type has a matching fluent builder (`SwitchActionBuilder`, `GeneratorActionBuilder`, ...). Actions are collected in an `ActionList`, with JSON serialization provided through `ActionJsonModule`.

When to apply actions is described by an `OperatorStrategy`: it references a contingency and one or more stages of `ConditionalActions`, each pairing a `Condition` with the ordered ids of the actions to run. Conditions evaluate the post-contingency / post-action state: `TrueCondition` (always), `AnyViolationCondition`, `AllViolationCondition`, `AtLeastOneViolationCondition` (on specific equipments) and the threshold conditions (`BranchThresholdCondition`, `InjectionThresholdCondition`, ...). `OperatorStrategyList` groups several strategies. Operator strategies are consumed by security analysis to model curative actions.

`action-ial` (the *IIDM action local* module) is the local implementation that ties these models to the IIDM network and runs them. It is itself split into submodules:

- `action-ial-dsl` — a Groovy DSL (`ActionDslLoader`) populating an `ActionDb` of contingencies, rules and actions, where a `Rule` associates an `ExpressionCondition` with a list of actions; it also exposes a `GroovyDslContingenciesProvider` plugging this DSL into the contingency framework.
- `action-ial-dsl-spi` — the `DslModificationExtension` SPI, allowing new DSL action keywords to be contributed.
- `action-ial-util` — DSL modification extensions (generator, switch and phase-shifter modifications) registered through `@AutoService`.
- `action-ial-simulator` — the `ActionSimulator` engine, with `LocalLoadFlowActionSimulator` and `ParallelLoadFlowActionSimulator` running a load flow between actions, network snapshot strategies (`CopyStrategy` and its variants) and the `ActionSimulatorTool` iTools command.
