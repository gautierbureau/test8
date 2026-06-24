# iidm-scripting

The `iidm-scripting` module (`iidm/iidm-scripting`) makes the IIDM grid model nicer to manipulate from Groovy. It contains two distinct kinds of contribution: *Groovy extension methods* that graft convenience accessors onto IIDM types, and two `ServiceLoader`-discovered contributions (a script-binding extension and an import post-processor). It is mostly Groovy code with one Java class. See the [scripting layer overview](../scripting.md) for context.

## Groovy extension methods (metaclass additions)

The bulk of the module is a set of Groovy *extension modules*: classes whose static methods take the receiver as a first `self` argument and are grafted onto existing IIDM types at run time. They are registered through the descriptor `META-INF/groovy/org.codehaus.groovy.runtime.ExtensionModule` (module name `powsybl-iidm-scripting-module`, version `1.0`), which lists the extension classes. Because they go through Groovy's extension-module mechanism rather than `@AutoService`, they take effect automatically for any Groovy code that simply has the jar on its classpath — no import or registration is needed.

| Extension class | Extends | Adds |
|-----------------|---------|------|
| `IdentifiableExtension` | `Identifiable` | `propertyMissing` / `methodMissing` magic (see below) and a guarded `setId` |
| `NetworkExtension` | `Network` | `getShunt(id)`, `getShunts()`, `getShuntStream()`, `getShuntCount()` aliases over shunt compensators |
| `SubstationExtension` | `Substation` | `getCountry()` forwarding to `getNullableCountry()` |
| `VoltageLevelExtension` | `VoltageLevel` | `getSubstation()` forwarding to `getNullableSubstation()` |
| `ShuntCompensatorExtension` | `ShuntCompensator` | linear-model shorthands (`getMaximumB`, `bPerSection`, `maximumSectionCount`) and `currentSectionCount`/`currentB` aliases |
| `TwoWindingsTransformerExtension`, `ThreeWindingsTransformerExtension` | the transformer types | `getSubstation()` forwarding to `getNullableSubstation()` |
| `BatteryExtension` | `Battery` | `p0`/`q0` aliases over `targetP`/`targetQ` |
| `BatteryAdderExtension` | `BatteryAdder` | `setP0`/`setQ0` aliases over `setTargetP`/`setTargetQ` |
| `FlowLimitsHolderExtension` | `FlowsLimitsHolder` | `getActivePowerLimits`/`getApparentPowerLimits`/`getCurrentLimits` forwarding to the `getNullable...` variants |
| `BranchExtension` | `Branch` | per-side and per-`TwoSides` `get...Limits` accessors forwarding to the `getNullable...` variants |

A recurring theme is replacing the model's `Optional`/exception-throwing getters with null-returning ones (`getNullableXxx`) that read more naturally from a script.

### Dynamic property and extension access

`IdentifiableExtension` is the most elaborate. It uses Groovy's missing-member hooks to expose IIDM properties and extensions as if they were fields:

- `propertyMissing(self, name)` first tries `self.getExtensionByName(name)` and otherwise returns `self.properties[name]`, so `network.getGenerator("g").myProp` reads either an extension or a string property;
- `propertyMissing(self, name, value)` writes (or, for `null`, removes) a string property;
- `methodMissing(self, name, args)` recognises `xxxAdder()` and `newXxx { ... }` calls and resolves the matching `ExtensionAdderProvider` (via `ExtensionAdderProviders.findCachedProvider`) to create — and, when a configuration closure is supplied, populate and `add()` — an extension adder;
- `setId(self, id)` is overridden to always throw, forbidding id modification (and working around Groovy issue GROOVY-3010 on the private field).

The `newXxx { ... }` form is backed by `AdderSpec`, a small Groovy helper whose `methodMissing` maps a call like `targetP 100` onto the adder's `withTargetP`/`setTargetP` setter (trying both `with` and `set` prefixes, and stripping a leading `_` to dodge Groovy keyword collisions). This is what gives the adder-configuration closures their declarative feel.

The matching `*ExtensionTest.groovy` files under `src/test` exercise each extension class.

## Script-driven network I/O and import post-processing

The module also provides two `ServiceLoader`-discovered contributions (see [the plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface)):

- `NetworkLoadSaveGroovyScriptExtension` (Groovy, `@AutoService(GroovyScriptExtension.class)`) implements the [`scripting`](scripting.md) module's `GroovyScriptExtension`. Its `load` adds two closures to the script `Binding`: `loadNetwork(file[, parameters])` wrapping `Network.read(...)` (with `LocalComputationManager.getDefault()`, an `ImportConfig`, and an `ImportersServiceLoader`), and `saveNetwork(format, network[, parameters], file)` wrapping `network.write(...)` (with an `ExportersServiceLoader`). A script can therefore read and write networks in one line. The class is constructor-injectable (import config, importers/exporters loaders, file system) for testing.
- `GroovyScriptPostProcessor` (Java, `@AutoService(ImportPostProcessor.class)`, name `groovyScript`) is an `ImportPostProcessor` that runs a Groovy script after every network import. The script path comes from the `groovy-post-processor` configuration module's `script` property, or defaults to `import-post-processor.groovy` in the platform configuration directory; if neither resolves it raises a `PowsyblException`. When the script file exists, `process` evaluates it in a `GroovyShell` with the `ThreadInterrupt` customiser and `network` / `computationManager` bound, again checking for interruption before evaluation. This lets a deployment customise imported networks declaratively.
