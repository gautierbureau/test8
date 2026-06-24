# Action model

The action layer models the **remedial actions** an operator can take to solve security issues, and the **strategies** deciding when to take them. It is the response side of the security workflow: where the [contingency model](contingency.md) describes what fails, the action model describes what is done about it. Two modules cover it, with complementary roles:

| Module | Role |
|--------|------|
| `action-api` | The portable model: `Action` and its subtypes, `ActionList`, and the operator-strategy / condition classes (the latter physically reside in `contingency-api`). It is consumed by security analysis to apply curative actions. |
| `action-ial` | The *IIDM action local* implementation — an aggregator of submodules providing a Groovy action DSL, an extensible modification SPI, utility DSL keywords, and a standalone load-flow-based action simulator with its iTools command. |

Both sit in the [grid model layer](../grid-model.md). `action-api` depends on `iidm-modification`: every action ultimately turns into a `NetworkModification`, so applying an action reuses the modification machinery.

## `action-api`

### `Action` and its subtypes

An `Action` (package `com.powsybl.action`) is a single elementary remedial action:

```java
public interface Action {
    String getType();
    String getId();
    default NetworkModification toModification() {
        throw new UnsupportedOperationException("toModification not implemented");
    }
}
```

`getType()` is a string discriminator (each concrete class exposes a `NAME` constant), and `toModification()` is the bridge to `iidm-modification`. For example, `GeneratorAction.toModification()` builds a `GeneratorModification`, and `SwitchAction.toModification()` returns an `OpenSwitch` or `CloseSwitch` depending on its flag. Most actions extend `AbstractAction`, which holds the id and implements `equals`/`hashCode` on it.

The available action types are:

| `NAME` (type) | Class | Effect |
|---------------|-------|--------|
| `SWITCH` | `SwitchAction` | Open or close a switch. |
| `TERMINALS_CONNECTION` | `TerminalsConnectionAction` | Connect or disconnect an equipment's terminal(s). |
| `GENERATOR` | `GeneratorAction` | Change a generator's targetP (absolute or relative), targetQ, targetV or voltage-regulation status. |
| `LOAD` | `LoadAction` | Change a load's P0 / Q0 (absolute or relative). |
| `PCT_LOAD_CHANGE` | `PercentChangeLoadAction` | Change a load by a percentage of its current value. |
| `SHUNT_COMPENSATOR_POSITION` | `ShuntCompensatorPositionAction` | Set a shunt compensator's section count. |
| `STATIC_VAR_COMPENSATOR` | `StaticVarCompensatorAction` | Change an SVC's regulation mode and setpoint. |
| `HVDC` | `HvdcAction` | Change HVDC active-power setpoint / mode (AC emulation). |
| `AREA_INTERCHANGE_TARGET_ACTION` | `AreaInterchangeTargetAction` | Set an area's interchange target. |
| `BOUNDARY_LINE` | `BoundaryLineAction` | Act on a boundary (dangling) line. |
| `PHASE_TAP_CHANGER_TAP_POSITION` | `PhaseTapChangerTapPositionAction` | Set a phase tap changer's tap position. |
| `RATIO_TAP_CHANGER_TAP_POSITION` | `RatioTapChangerTapPositionAction` | Set a ratio tap changer's tap position. |
| `PHASE_TAP_CHANGER_REGULATION` | `PhaseTapChangerRegulationAction` | Change a phase tap changer's regulation. |
| `RATIO_TAP_CHANGER_REGULATION` | `RatioTapChangerRegulationAction` | Change a ratio tap changer's regulation. |
| `MULTIPLE_ACTIONS` | `MultipleActionsAction` | An ordered composite of several actions. |

The tap-changer actions share abstract bases (`AbstractTapChangerAction`, `AbstractTapChangerTapPositionAction`, `AbstractTapChangerRegulationAction`), and the load actions share `AbstractLoadAction`, factoring out the common attributes.

### Builders

Every action type has a matching fluent builder (`SwitchActionBuilder`, `GeneratorActionBuilder`, `LoadActionBuilder`, ...) implementing the generic `ActionBuilder<T>` interface:

```java
public interface ActionBuilder<T extends ActionBuilder<T>> {
    String getType();
    T withId(String id);
    String getId();
    T withNetworkElementId(String elementId);
    Action build();
}
```

The builder pattern is what allows actions to be deserialized polymorphically (Jackson instantiates a builder, populates it from JSON, then calls `build()`) and what enables `IdentifierActionList` to defer the resolution of an action's target equipment until a network is available.

### `ActionList` and JSON

Actions are collected in an `ActionList`, a simple container with versioned JSON read/write (`readJsonFile` / `writeJsonFile`, `VERSION = "1.3"`). `IdentifierActionList` is a variant that holds, alongside concrete actions, a map of unresolved builders keyed by a `NetworkElementIdentifier`; its `getActions(Network)` resolves each identifier against the network (requiring exactly one match) and finishes building those actions.

JSON (de)serialization is centralized in `ActionJsonModule`, a Jackson `SimpleModule`. Polymorphism is driven by the `type` property: the module registers, for every action type, a `NamedType` pairing the action class and its builder class under the type name, plus the type's serializer and the builder's deserializer. Deserialization therefore always goes through the builder. A backward-compatibility alias maps the old `DANGLING_LINE` type to `BoundaryLineActionBuilder`.

### Operator strategies and conditions

When to apply actions is described by an `OperatorStrategy`. These classes live in `contingency-api` (packages `com.powsybl.contingency.strategy` and `...strategy.condition`) so they can be used without depending on `action-ial`, but they belong functionally to the action model and are presented here.

An `OperatorStrategy` (extending `AbstractExtendable`) binds a `ContingencyContext` (see the [contingency model](contingency.md)) to one or more **stages** of `ConditionalActions`. Each `ConditionalActions` pairs a `Condition` with the ordered ids of the actions to run if that condition holds. A single-stage strategy is the common case; a multi-stage strategy is evaluated stage by stage, each condition being checked on the network with the previous stages' actions already applied. `OperatorStrategyList` is the versioned JSON container (`VERSION = "1.2"`), and it registers both `ContingencyJsonModule` and `ActionJsonModule` so a strategy file can reference action ids resolved against an `ActionList`.

A `Condition` evaluates the post-contingency / post-action state to decide whether to act:

```java
public interface Condition {
    String getType();
}
```

| `NAME` (type) | Class | Triggers when |
|---------------|-------|---------------|
| `TRUE_CONDITION` | `TrueCondition` | Always. |
| `ANY_VIOLATION_CONDITION` | `AnyViolationCondition` | Any limit violation exists (optionally filtered by `LimitViolationType`). |
| `ALL_VIOLATION` | `AllViolationCondition` | All listed equipments are in violation. |
| `AT_LEAST_ONE_VIOLATION` | `AtLeastOneViolationCondition` | At least one of the listed equipments is in violation. |
| `BRANCH_THRESHOLD_CONDITION` | `BranchThresholdCondition` | A branch variable crosses a threshold on a given side. |
| `THREE_WINDINGS_TRANSFORMER_THRESHOLD_CONDITION` | `ThreeWindingsTransformerThresholdCondition` | A three-windings transformer variable crosses a threshold. |
| `INJECTION_THRESHOLD_CONDITION` | `InjectionThresholdCondition` | An injection variable crosses a threshold. |
| `AC_DC_CONVERTER_THRESHOLD_CONDITION` | `AcDcConverterThresholdCondition` | An AC/DC converter variable crosses a threshold. |

The violation conditions extend `AbstractFilteredCondition` (filtering on a set of `LimitViolationType`). The threshold conditions extend `AbstractThresholdCondition`, which carries an `equipmentId`, a `Variable` (`ACTIVE_POWER`, `REACTIVE_POWER`, `CURRENT`, `TARGET_P`), a `ComparisonType` (`EQUALS`, `GREATER_THAN`, ... `NOT_EQUAL`) and a `threshold`; the sided variants extend `AbstractSidedThresholdCondition`. The limit-violation classes they reason about (`LimitViolation`, `LimitViolationType`, `LimitViolationFilter`, `ViolationLocation`) also live in `contingency-api`, under `com.powsybl.contingency.violations`.

Operator strategies are consumed by [security analysis](../../simulation/security/index.md) to model curative actions; the user-facing way to author them as a script is the [action DSL](../../simulation/security/action-dsl.md), implemented by `action-ial`.

## `action-ial`

`action-ial` (artifact `powsybl-action-ial`, described as the *action aggregator module*) is an older, self-contained engine that predates the `action-api` / operator-strategy model: it parses a Groovy DSL of contingencies, rules and actions into an in-memory database and runs it through repeated load flows. It is a pure aggregator POM with four submodules.

### `action-ial-dsl` — the action DSL

The DSL is loaded by `ActionDslLoader` (Groovy, package `com.powsybl.action.ial.dsl`, extending `com.powsybl.dsl.DslLoader`). `load(Network)` evaluates a script and returns an `ActionDb`, the database of parsed objects: it collects `Contingency` objects (reusing the [contingency DSL](../../simulation/security/contingency-dsl.md)'s `contingency` keyword), `Rule`s and `Action`s, and `checkUndefinedActions()` validates that every rule references a defined action. The script has three top-level blocks:

```groovy
contingency('contingency1') {
    equipments 'NHV1_NHV2_1'
}

rule('rule1') {
    description '...'
    when isOverloaded(['NHV1_NHV2_1'])
    apply 'action1'
    life 5
}

action('action1') {
    description '...'
    tasks {
        openSwitch 'switchId'
        generatorModification('GEN') { targetP 100.0 }
    }
}
```

Here a `Rule` associates a condition with a list of actions, with a `RuleType` (`APPLY` or `TEST`) and a `life` counter; an `Action` wraps an ordered list of `NetworkModification`s (its `tasks`). Conditions are expressed as Groovy expressions parsed into an abstract syntax tree under `com.powsybl.action.ial.dsl.ast`. The AST is built around an `ExpressionNode` hierarchy and an `ActionExpressionVisitor` (with `DefaultActionExpressionVisitor`, an evaluator `ActionExpressionEvaluator` and a printer `ActionExpressionPrinter`). The action-specific nodes back the DSL's condition vocabulary: `ActionTakenNode` (`actionTaken(id)`), `ContingencyOccurredNode` (`contingencyOccurred()`), `MostLoadedNode` (`mostLoaded(...)`), `IsOverloadedNode` / `AllOverloadedNode` / `LoadingRankNode`, and the network-access nodes `NetworkComponentNode` / `NetworkPropertyNode` / `NetworkMethodNode` (for expressions such as `line('L').p1 > 500`). A `Rule`'s condition is wrapped as an `ExpressionCondition` over the AST root.

The `tasks` block is itself extensible (see the SPI below), and a built-in `script { network, computationManager -> ... }` keyword runs arbitrary Groovy as a `ScriptNetworkModification`, contributed by the `@AutoService`-registered `ScriptDslModificationExtension`.

This module also exposes `GroovyDslContingenciesProvider` and its `@AutoService(ContingenciesProviderFactory.class)` factory, so an action DSL script's contingencies can feed a standard security analysis through the same `ContingenciesProvider` SPI used everywhere else.

### `action-ial-dsl-spi` — the modification SPI

A tiny module (artifact `powsybl-action-ial-dsl-spi`) defining a single interface, `DslModificationExtension`, the extension point for adding new keywords to the DSL's `tasks` block:

```java
void addToSpec(MetaClass modificationsSpecMetaClass,
               List<NetworkModification> modifications,
               Binding binding);
```

`ActionDslLoader` discovers the implementations through `ServiceLoader` and lets each one inject closure methods into the modifications spec's metaclass. This is how new task keywords are contributed without touching the loader.

### `action-ial-util` — built-in DSL keywords

A set of `DslModificationExtension` implementations (Groovy, each `@AutoService(DslModificationExtension.class)`), each contributing one task keyword that maps to an `iidm-modification`:

| Keyword | Extension | Modification |
|---------|-----------|--------------|
| `generatorModification(id) { ... }` | `GeneratorModificationModificationExtension` | `GeneratorModification` |
| `openSwitch(id)` | `OpenSwitchModificationExtension` | `OpenSwitch` |
| `closeSwitch(id)` | `CloseSwitchModificationExtension` | `CloseSwitch` |
| `phaseShifterTap(id, delta)` | `PhaseShifterTapModificationExtension` | `PhaseShifterShiftTap` |
| `phaseShifterFixedTap(id, position)` | `PhaseShifterFixedTapModificationExtension` | `PhaseShifterSetAsFixedTap` |
| `optimizePhaseShifterTap(id)` | `PhaseShifterOptimizerModificationExtension` | `PhaseShifterOptimizeTap` |

### `action-ial-simulator` — the load-flow action simulator

The engine that runs an `ActionDb`. `ActionSimulator` is the entry interface (`getName()`, `start(actionDb, contingencyIds)`); the concrete implementation is `LoadFlowActionSimulator` (named `"loadflow"`), which iterates: run a load flow, evaluate the rules against the resulting `LimitViolation`s, apply the matching actions, and repeat until no violation remains or `maxIterations` is reached. `LocalLoadFlowActionSimulator` restricts the work to a `Partition` of the contingencies, and `ParallelLoadFlowActionSimulator` distributes the work by submitting the `action-simulator` iTools command through the `ComputationManager`.

Between contingencies the simulator restores the base case through a `NetworkCopyStrategy`, selected by the `CopyStrategy` enum: `CopyStateStrategy` (`STATE`) clones an IIDM variant, while `DeepCopyStrategy` (`DEEP`, the default) round-trips the network through a gzipped serialization. `LoadFlowActionSimulatorConfig` (read from the `load-flow-action-simulator` configuration module) carries `maxIterations` (default 30), the optional load-flow provider name, the copy strategy and debug flags. Progress is reported through the `LoadFlowActionSimulatorObserver` interface (with an empty `DefaultLoadFlowActionSimulatorObserver` and a log-printing implementation), whose callbacks cover the pre-/post-contingency phases, each round, load-flow convergence, rule checks and action application.

`ActionSimulatorTool` is the `@AutoService(Tool.class)` exposing all of this as the **`action-simulator`** iTools command (theme *Computation*). It takes a case file and a DSL file, with options to select contingencies, control output (`--output-file`, `--output-case-folder`, `--output-case-format`, ...) and export the network after each round.
