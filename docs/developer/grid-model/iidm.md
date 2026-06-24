# IIDM

IIDM (the *iTesla Internal Data Model*) is the central grid model of PowSyBl Core: the single in-memory representation of a power network that every importer, exporter and simulation reads from and writes to. This page details the design and code structure of the `iidm` module, submodule by submodule. It assumes familiarity with the [grid model layer](../grid-model.md) overview and complements the functional [grid model](../../grid_model/index.md) user documentation.

The `iidm` module is a Maven aggregator split into submodules so that the API is cleanly separated from its implementations and from optional features:

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

## `iidm-api` — the interface model

`iidm-api` contains only interfaces and a few abstract helpers in the `com.powsybl.iidm.network` package: it defines *what* a network is, not *how* it is stored. Utility classes used by every implementation live under `com.powsybl.iidm.network.util` (`SV`, `BranchData`, `TwtData`, `Networks`, `TieLineUtil`, `SwitchPredicates`, ...).

### The `Identifiable` hierarchy

Every object that is part of the model and uniquely identified by a `String` id implements `Identifiable<I>`. It extends `com.powsybl.commons.extensions.Extendable<I>` (so any identifiable can carry extensions) and `PropertiesHolder` (free-form string properties), and exposes id, optional name, aliases (with an optional alias *type*), a fictitious flag, and `getType()` returning an `IdentifiableType` (`NETWORK`, `SUBSTATION`, `VOLTAGE_LEVEL`, `GENERATOR`, `LOAD`, `LINE`, `TWO_WINDINGS_TRANSFORMER`, ...). It also gives access to `getNetwork()` and `getParentNetwork()` (the smallest sub-network containing the object).

On top of `Identifiable`, two refinements describe equipment that is electrically connected to the grid:

- `Connectable<I>` — AC equipment that is part of a substation topology. It owns one or more `Terminal`s (`getTerminals()`), can be removed (`remove()`), and exposes `connect(...)` / `disconnect(...)` methods that operate the surrounding breakers (driven by a `Predicate<Switch>`, with defaults from `SwitchPredicates`).
- `Injection<I> extends Connectable<I>` — a connectable with exactly one terminal (`getTerminal()`). `Generator`, `Load`, `Battery`, `ShuntCompensator`, `StaticVarCompensator`, `DanglingLine`/`BoundaryLine`, `Ground` and the HVDC converter stations are injections.

Two-terminal equipment derives from `Branch<I>` (`getTerminal1()`/`getTerminal2()`, `getSide(Terminal)`, and the operational-limits machinery), implemented by `Line` and `TwoWindingsTransformer`; `ThreeWindingsTransformer` has three legs rather than extending `Branch`. The structural containers — `Network`, `Substation`, `VoltageLevel`, `Area` — extend `Container<Identifiable>`. `Switch`, `Bus`, `BusbarSection`, `HvdcLine`, `TieLine` and the DC objects (`DcNode`, `DcLine`, `DcSwitch`, ...) complete the type set.

`Network` is the top-level container: it holds substations, voltage levels, areas and all the equipment, exposes typed accessors and streams (`getGenerators()`, `getGeneratorStream()`, `getGeneratorCount()`, `getIdentifiable(id)`, ...), and the generic read/write entry points (`Network.read(Path)`, `Network.write(...)`) that dispatch to the format importers and exporters. A `Network` may also be the merge of several **sub-networks** that keep their identity — see [network and subnetwork](../../grid_model/network_subnetwork.md).

### The adder (fluent builder) pattern

Network objects are never created with constructors. Each container exposes a factory method that returns an *adder* — a fluent builder whose setters return the adder itself and whose terminal `add()` method validates the attributes and creates the object:

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

All adders ultimately derive from `IdentifiableAdder<T, A>`, which declares the common `setId`, `setName`, `setEnsureIdUnicity`, `setFictitious` and the terminal `add()`. Every creatable type has its own adder interface (`SubstationAdder`, `VoltageLevelAdder`, `GeneratorAdder`, `LineAdder`, `TieLineAdder`, ...). The adder enforces required attributes and consistency at `add()` time, so a half-built object can never enter the network. The same pattern is reused for extensions (`object.newExtension(XxxAdder.class)...add()`) and for sub-objects such as reactive limits (`ReactiveCapabilityCurveAdder`, `MinMaxReactiveLimitsAdder`), tap changers (`RatioTapChangerAdder`, `PhaseTapChangerAdder`) and operational limits (`CurrentLimitsAdder` and the other `LoadingLimitsAdder`s, created from an `OperationalLimitsGroup`).

### Terminals and topology views

A `Terminal` is an equipment's connection point in a voltage level. It carries the variant-dependent state values (`getP()`, `getQ()`) and gives access to the topology through three nested views, mirroring the three levels of detail at which the topology can be read:

- `Terminal.NodeBreakerView` — the connection `getNode()` in a node/breaker topology, plus `moveConnectable(node, voltageLevelId)`.
- `Terminal.BusBreakerView` — `getBus()` / `getConnectableBus()` in the bus/breaker topology (variant-dependent), plus `setConnectableBus` and `moveConnectable`.
- `Terminal.BusView` — `getBus()` / `getConnectableBus()` in the aggregated bus-only topology (variant-dependent).

The same three levels exist at the `VoltageLevel`. A voltage level carries a **topology model** selected by `TopologyKind` (`NODE_BREAKER` or `BUS_BREAKER`), and the topology can be read through three views ordered from most to least detailed: `getNodeBreakerView()`, `getBusBreakerView()` and `getBusView()`. Each view exposes adders (`newSwitch()`, `newBusbarSection()`, `newBus()`) and a `TopologyTraverser`. The rule is:

- a view **more detailed** than the underlying model is *not available* (calling it throws);
- a view **at the same level** as the model is *modifiable*;
- a **coarser** view is *read-only* and computed from the model.

The bus/breaker and bus views are variant-dependent because the computed buses depend on switch positions. A switch flagged with `Switch.isRetained()` survives the reduction to the bus/breaker view. `VoltageLevel.convertToTopology(TopologyKind)` switches a voltage level from one model to the other.

### Variants (state management)

A single `Network` can hold several **variants** (states): different sets of state values (flows, voltages, tap positions, switch positions, ...) sharing the same topology and structural data. Variants are managed through the `VariantManager` returned by `network.getVariantManager()`. The first variant is named `VariantManagerConstants.INITIAL_VARIANT_ID` (`"InitialState"`). The manager exposes:

- `getVariantIds()` / `getWorkingVariantId()` / `setWorkingVariant(String)` to list and select the active variant;
- `cloneVariant(source, target[, mayOverwrite])` to duplicate a state, and `removeVariant(String)` to drop one;
- `allowVariantMultiThreadAccess(boolean)` / `isVariantMultiThreadAccessAllowed()` to enable a per-thread working variant.

Variant management is *not* thread-safe with concurrent structural changes; the documented pattern is to pre-allocate the variants on the main thread, enable multi-thread access, work on distinct variants from worker threads (writing only attributes flagged as variant-dependent in the Javadoc), then remove the variants on the main thread.

### Obtaining a network: `NetworkFactory` (SPI)

Networks are never instantiated directly either: they are created through a `NetworkFactory`, obtained with `NetworkFactory.findDefault()` or `NetworkFactory.find(name)`. The factory itself is discovered through the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface): a `NetworkFactoryService` (a `PlatformConfigNamedProvider` with `getName()` and `createNetworkFactory()`) is looked up by `PlatformConfigNamedProvider.Finder` under the `"network"` module. This indirection is what lets a different implementation (for instance a database-backed one) be substituted without touching the rest of the codebase. `NetworkFactory` also provides `merge(id, networks...)` to build a network from several sub-networks.

## `iidm-impl` — the default in-memory implementation

`iidm-impl` provides the reference in-memory implementation of the API, in the `com.powsybl.iidm.network.impl` package (with an `extensions` sub-package for the implementations of the standard extensions). It is registered as the default factory by `NetworkFactoryServiceImpl`, annotated `@AutoService(NetworkFactoryService.class)`, whose `getName()` returns `NetworkFactoryConstants.DEFAULT` and whose `createNetworkFactory()` returns a `NetworkFactoryImpl`. This is why `NetworkFactory.findDefault()` returns the in-memory factory with no explicit dependency on `iidm-impl`.

### `NetworkImpl` and the object index

`NetworkImpl extends AbstractNetwork implements VariantManagerHolder, MultiVariantObject`. Its main collaborators are:

- a `NetworkIndex` providing O(1) lookup of every `Identifiable` by id (and alias);
- a `VariantManagerImpl` controlling the variant lifecycle;
- a map of `SubnetworkImpl` (each also an `AbstractNetwork`) for merged sub-networks;
- `RefChain` references that let objects keep a stable, relocatable handle to their owning network across merges and splits;
- a `NetworkListenerList` dispatching `NetworkListener` events.

### Variant storage

State-dependent attributes are kept in per-variant arrays rather than copied wholesale. The contract is the `MultiVariantObject` interface, whose callbacks (`extendVariantArraySize`, `reduceVariantArraySize`, `deleteVariantArrayElement`, `allocateVariantArrayElement`) are invoked by the `VariantManagerImpl` whenever variants are created, cloned or removed. `VariantManagerImpl` maps variant ids to array indices (a `BiMap<String, Integer>`), recycles freed slots, and holds a `VariantContext` giving the current variant index for the calling thread.

Objects that store such state use a `VariantArray<S>` — a generic, thread-aware container of per-variant slots whose `get()` returns the slot for the current variant. For scalar attributes, primitive arrays from Trove are used directly: for instance `AbstractTerminal` keeps `TDoubleArrayList p`/`q`, and `NodeTerminal` adds `v`, `angle` and the connected/synchronous component numbers, all indexed by variant. Cloning a variant therefore only duplicates the indexed slots, and switching the working variant only changes which slot is read — this is what lets a security analysis explore many contingency states in parallel without copying the whole structure.

### Topology models

The two topology kinds are implemented by `NodeBreakerTopologyModel` and `BusBreakerTopologyModel`, both extending `AbstractTopologyModel` (which implements the internal `TopologyModel` contract and holds a reference to its `VoltageLevelExt`):

- **`NodeBreakerTopologyModel`** stores the topology as an `UndirectedGraphImpl<NodeTerminal, SwitchImpl>` (from `com.powsybl.math.graph`): connection nodes are vertices and switches are edges. Buses are *computed* on demand by traversing the graph: the inner `CalculatedBusTopology` builds `CalculatedBusImpl` instances by walking through closed switches (validated by a `BusChecker`), while `CalculatedBusBreakerTopology` (a subclass) additionally stops at *retained* switches so that retained breakers remain visible in the bus/breaker view. Terminals in this model are `NodeTerminal` instances.
- **`BusBreakerTopologyModel`** stores explicit `ConfiguredBusImpl` buses joined by switches; terminals are `BusTerminal` instances carrying their connectable-bus id and connection state per variant.

`AbstractTerminal` and its subclasses `NodeTerminal` / `BusTerminal` provide the three terminal views, delegating the coarser ones to the calculated-bus topologies.

### Adder implementations

Adders are implemented by abstract base classes mirroring the API hierarchy: `AbstractIdentifiableAdder` (id, name, fictitious flag, and the `checkAndGetUniqueId()` logic honouring `setEnsureIdUnicity`) and, for one-terminal equipment, `AbstractInjectionAdder` (node / bus / connectable-bus selection and terminal construction via a `TerminalBuilder`). Each concrete adder validates its attributes in `add()` before instantiating the object and registering it in the `NetworkIndex`.

## `iidm-extensions` — standard extensions

The model can be enriched without modifying its core interfaces through **extensions**: typed data attached to an `Identifiable` (an `Extension<T>` where `T` is the extended type). `iidm-extensions` holds the *standard* extensions in `com.powsybl.iidm.network.extensions`, each defined as an extension interface plus its adder (and an implementation in `iidm-impl`'s `extensions` sub-package). Their serializers live in `iidm-serde`. Representative extensions include:

- diagram/position data — `ConnectablePosition`, `BusbarSectionPosition`, `SubstationPosition`, `LinePosition`;
- control data — `ActivePowerControl`, `CoordinatedReactiveControl`, `RemoteReactivePowerControl`, `SecondaryVoltageControl`, `VoltagePerReactivePowerControl`, `HvdcAngleDroopActivePowerControl`, `ReferencePriority`/`ReferencePriorities`;
- short-circuit data — `GeneratorShortCircuit`, `LineFortescue`;
- state-estimation data — `Measurement`/`Measurements`, `DiscreteMeasurement`/`DiscreteMeasurements`, `BranchObservability`, `InjectionObservability`.

Extensions are added through an adder (`object.newExtension(XxxAdder.class)...add()`); their adders and serializers are themselves discovered through the plugin mechanism. See the [extension mechanism](../index.md#the-extension-mechanism) overview.

## `iidm-serde` — serialization

`iidm-serde` (package `com.powsybl.iidm.serde`) reads and writes a `Network` in the native IIDM serialization, in three interchangeable formats backed by a common *tree-data* abstraction (`TreeDataReader`/`TreeDataWriter`, defined in `commons`):

| Format | `getFormat()` | `TreeDataFormat` |
|--------|---------------|------------------|
| XIIDM | `XIIDM` | `XML` |
| JIIDM | `JIIDM` | `JSON` |
| BIIDM | `BIIDM` | `BIN` |

The central class is `NetworkSerDe`, holding the read/write logic shared by all three formats. Its static `read(...)` and `write(...)` overloads take an `InputStream`/`OutputStream`, a `Path` or a `DataSource`, together with `ImportOptions` / `ExportOptions` and an `Anonymizer`; `validate(...)` checks an input against the IIDM XSD. Reading and writing thread a `NetworkDeserializerContext` / `NetworkSerializerContext` (both extending `AbstractNetworkSerDeContext`, which carries the `IidmVersion` and the `Anonymizer`) through a set of per-equipment SerDe classes — `GeneratorSerDe`, `VoltageLevelSerDe`, `LineSerDe`, `LoadSerDe`, ... — that derive from `AbstractIdentifiableSerDe` via `AbstractSimpleIdentifiableSerDe` (object creatable before reading its sub-elements) or `AbstractComplexIdentifiableSerDe` (objects whose model is read before instantiation, e.g. loads with exponential/ZIP models).

Each format also exposes an `Importer` / `Exporter` pair, all `@AutoService`-registered and built on `AbstractTreeDataImporter` / `AbstractTreeDataExporter`: `XMLImporter`/`XMLExporter`, `JsonImporter`/`JsonExporter`, `BinaryImporter`/`BinaryExporter`. In practice these are reached transparently through the generic `Network.read(...)` / `Network.write(...)` entry points of `iidm-api`. The binary format is recognized by a magic-number header.

### Versioning

The model is versioned by the `IidmVersion` enum, from `V_1_0` (in the historical iTesla namespace) through `V_1_17`, the current version held by `IidmSerDeConstants.CURRENT_IIDM_VERSION`. Each version maps to an XML namespace URI and an XSD (`getNamespaceURI()`, `getXsd()`), so older files remain readable; from `V_1_7` onward a separate "equipment-only" namespace/XSD supports partial (equipment-level) validation. `ExportOptions.version` selects the version to write.

`ImportOptions` and `ExportOptions` both extend `AbstractOptions` (carrying the `TreeDataFormat` and the included/excluded extension sets). `ExportOptions` adds, among others, `topologyLevel` (`TopologyLevel`: node/breaker, bus/breaker or bus/branch), `withBranchSV`, `onlyMainCc`, `anonymized`, `indent`, `sorted` and per-extension version overrides; `ImportOptions` adds flags such as `throwExceptionIfExtensionNotFound`, `withAutomationSystems` and a minimal validation level.

### Extension serialization and anonymization

Extensions are serialized by their own providers, discovered through the plugin mechanism via `ExtensionProviders` (looking up `ExtensionSerDe` providers in the `"network"` category). The SPI interface `ExtensionSerDe` (defined in `commons`, `com.powsybl.commons.extensions`) covers XML and binary; JSON uses `ExtensionJsonSerializer`. Versioned IIDM extensions derive from `AbstractVersionableNetworkExtensionSerDe`, mapping each extension version to a namespace/serialization name. The `com.powsybl.iidm.serde.anonymizer` package can mask identifiers on export: `Anonymizer` is the contract (`anonymizeString`/`deanonymizeString`, `anonymizeCountry`, plus read/write of the mapping), with `SimpleAnonymizer` and `FakeAnonymizer` implementations.

## `iidm-modification` — network modifications

`iidm-modification` (package `com.powsybl.iidm.modification`) provides reusable, named transformations of a network. They all implement the `NetworkModification` interface — a family of `apply(Network, ...)` overloads (optionally taking a `ComputationManager`, a `ReportNode`, a `NamingStrategy`, a `throwException` flag and a `dryRun` flag) plus `hasImpactOnNetwork(Network)`, which returns a `NetworkModificationImpact` (`CANNOT_BE_APPLIED`, `NO_IMPACT_ON_NETWORK`, `HAS_IMPACT_ON_NETWORK`). Concrete modifications extend `AbstractNetworkModification` (which implements all the apply overloads and delegates to `getName()` and the impact check); `NetworkModificationList` composes several into one.

The modifications are grouped by concern:

- **topology** (`topology` package) — structural changes, most exposed through a fluent `*Builder`: `CreateCouplingDevice`, `CreateVoltageLevelTopology`, `CreateFeederBay`, `CreateBranchFeederBays`, `CreateLineOnLine`, `ConnectVoltageLevelOnLine`, `ReplaceTeePointByVoltageLevelOnLine`, `ConnectFeedersToBusbarSections`, `MoveFeederBay`, `RemoveFeederBay`, `RemoveVoltageLevel`, `RemoveSubstation`, `RemoveHvdcLine`, and the `Revert*` counterparts. A `NamingStrategy` (default `DefaultNamingStrategy`, pluggable via `NamingStrategiesServiceLoader`) names the objects these modifications create.
- **tripping** (`tripping` package) — disconnecting equipment by simulating the opening of the switches around it. The `Tripping` interface (extending `NetworkModification`) computes the set of switches to open and terminals to disconnect (`traverse(...)`, with DC variants); implementations include `BranchTripping`, `LineTripping`, `TwoWindingsTransformerTripping`, `GeneratorTripping`, `LoadTripping`, `BatteryTripping`, `BusbarSectionTripping`, `SwitchTripping`, `HvdcLineTripping`, ... sharing `AbstractTripping` / `AbstractInjectionTripping`. This is the mechanism used to apply contingencies.
- **scalable** (`scalable` package) — the `Scalable` abstraction, an injection (or combination of injections) whose active power can be increased or decreased. `GeneratorScalable` and `LoadScalable` scale a single injection; `ProportionalScalable`, `StackScalable` and `UpDownScalable` combine several, a `ScalingConvention` (`GENERATOR` vs `LOAD`) fixes the sign convention, and a `ScalingParameters` (with `ScalingType` and `Priority` enums, JSON-serializable) tunes the scaling. Scalables are used to shift generation/consumption, for example to build study cases.
- **tap changers** (`tapchanger` package) — `RatioTapPositionModification` and `PhaseTapPositionModification` (sharing `AbstractTapPositionModification`, supporting two- and three-winding transformers and relative position changes).
- **equipment parameter changes** (root package) — `GeneratorModification`, `LoadModification`, `PercentChangeLoadModification`, `BatteryModification`, `ShuntCompensatorModification`, `StaticVarCompensatorModification`, `HvdcLineModification`, `OpenSwitch`, `CloseSwitch`, the phase-shifter helpers (`PhaseShifterOptimizeTap`, `PhaseShifterSetAsFixedTap`, `PhaseShifterShiftTap`), and structural replacements such as `ReplaceTieLinesByLines` and the 3×2WT ↔ 3WT converters.

## The other `iidm` submodules

### `iidm-criteria`

Defines criteria to select network elements by their characteristics, with JSON (de)serialization. A `NetworkElementCriterion` (base `AbstractNetworkElementCriterion`) targets a kind of element — `LineCriterion`, `TwoWindingsTransformerCriterion`, `ThreeWindingsTransformerCriterion`, `TieLineCriterion`, `IdentifiableCriterion`, `NetworkElementIdListCriterion` — and combines `Criterion`s such as `SingleCountryCriterion` / `TwoCountriesCriterion`, the nominal-voltage criteria (`SingleNominalVoltageCriterion`, `TwoNominalVoltageCriterion`), `PropertyCriterion` and `RegexCriterion`. The package also defines limit-duration criteria (permanent, interval/equality temporary). These criteria are reused by criterion-based contingency lists and by limit/duration filtering.

### `iidm-geodata`

Loads geographical data (substation points, line paths) from GeoJSON and applies it to the network as the `SubstationPosition` and `LinePosition` extensions (defined in `iidm-extensions`), using a `Coordinate` for latitude/longitude. Key classes are `GeoJsonDataParser`, `GeoJsonDataAdder`, `NetworkGeoDataExtensionsAdder` and the `GeoJsonAdderPostProcessor` (an import post-processor), with helpers such as `LineGraph`, `LineCoordinatesOrdering` and `DistanceCalculator`.

### `iidm-reducer`

Shrinks a network to a subset of interest. `NetworkReducer` (built through `DefaultNetworkReducerBuilder`, implemented by `DefaultNetworkReducer` over `AbstractNetworkReducer`) keeps the substations and voltage levels accepted by a `NetworkPredicate` — `NominalVoltageNetworkPredicate`, `SubNetworkPredicate`, `IdentifierNetworkPredicate`, `DefaultNetworkPredicate` — with `ReductionOptions` controlling boundary handling and a `NetworkReducerObserver` (default `DefaultNetworkReducerObserver`) reporting what is removed.

### `iidm-comparator`

Compares two states of a network. `NetworkStateComparator` takes a network and the name of an alternate variant and produces an Excel report of the differences in voltages, angles, flows and tap positions per equipment type.

### `iidm-tck`

The Technology Compatibility Kit: a suite of abstract tests (over a hundred `Abstract*Test` classes under `src/test/java`, e.g. `AbstractNodeBreakerTest`, `AbstractTopologyTraverserTest`, `AbstractMinMaxReactiveLimitsTest`, plus extension TCK tests) that any IIDM implementation must pass to prove it conforms to the API contract. `iidm-impl` and alternative implementations subclass these tests to validate their behaviour.

### `iidm-test`

Provides ready-made sample networks used throughout the test suites and documentation, exposed as factories such as `EurostagTutorialExample1Factory`, `FourSubstationsNodeBreakerFactory`, `BatteryNetworkFactory`, `SvcTestCaseFactory`, `PhaseShifterTestCaseFactory`, `ThreeWindingsTransformerNetworkFactory` and `FictitiousSwitchFactory`.

### `iidm-scripting`

Adds Groovy extension methods (registered through Groovy's `ExtensionModule` descriptor) that make the model more pleasant to script — concise property access and adder syntax on `Network`, `VoltageLevel`, the transformers, batteries and other types — and a `GroovyScriptPostProcessor` to run a Groovy script as an import post-processor.
