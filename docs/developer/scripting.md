# Scripting layer

The scripting layer lets users drive PowSyBl from [Groovy](https://groovy-lang.org/) scripts and domain-specific languages (DSLs) instead of writing Java. It is made of three modules:

| Module | Role |
|--------|------|
| `dsl` | Reusable infrastructure for building Groovy DSLs (shell configuration, an extension SPI, an arithmetic-expression AST). |
| `scripting` | Runs plain Groovy scripts over PowSyBl, including the `run-script` iTools command and a script-extension SPI. |
| `iidm-scripting` (`iidm/iidm-scripting`) | Groovy *extension methods* and helper bindings that make the IIDM model more pleasant to manipulate from scripts. |

All of them build on Groovy's compiler customisers and metaclass mechanism, and they are wired together through the same `ServiceLoader`/`@AutoService` plugin mechanism used elsewhere in PowSyBl (see [the plugin mechanism](index.md#the-plugin-mechanism-service-provider-interface)). The DSLs that actually use this infrastructure (contingencies, actions, dynamic simulation, ...) live in their own modules and are out of scope here.

```{toctree}
:hidden:
scripting/dsl
scripting/scripting
scripting/iidm-scripting
```

## The `dsl` module

`dsl` provides the building blocks shared by every PowSyBl Groovy DSL. It contains no DSL of its own; it is a small library mixing Java and Groovy sources.

### Building and configuring a DSL

`DslLoader` (Groovy) is the base class for DSL loaders. It wraps a `GroovyCodeSource` (built from a `File`, a `String` or a stream) and exposes a static `createShell(Binding, ImportCustomizer)` factory that builds a pre-configured `GroovyShell`. The shell is configured through a `CompilerConfiguration` to which two AST transformation customisers are added:

- `PowsyblDslAstTransformation`, the PowSyBl-specific transformation;
- Groovy's own `ThreadInterrupt` transformation, which injects a thread-interruption check into every loop so that long-running or runaway scripts can be cancelled.

`GroovyScripts` (Java) is a small helper that loads a `GroovyCodeSource` from an `InputStream` or a `Path` using UTF-8. `GroovyDslConstants` and `GroovyUtil` hold shared constants and helper closures. `DslException` is the module's runtime exception.

### The AST transformation

`AbstractPowsyblDslAstTransformation` is an `ASTTransformation` that applies a `ClassCodeExpressionTransformer` (supplied as a `Function<SourceUnit, ...>`) to every method body and to the top-level statement block of a script. `PowsyblDslAstTransformation` is the concrete transformation installed by `DslLoader`. This is the hook through which a DSL can rewrite expressions at compile time — most notably to turn arithmetic and comparison expressions into the expression AST described below.

### The extension SPI

DSLs are made extensible through `ExtendableDslExtension<E extends Extendable<E>>`. An implementation declares the extended type via `getExtendableClass()` and contributes to the DSL through `addToSpec(MetaClass extSpecMetaClass, List<Extension<E>> extensions, Binding binding)`, i.e. it adds methods/properties to a Groovy `MetaClass` and entries to the script `Binding`. Concrete DSLs sub-interface this SPI and discover implementations with `ServiceLoader`. For example, the contingency DSL (in the `contingency-dsl` module) defines `ContingencyDslExtension extends ExtendableDslExtension<Contingency>`, letting third parties extend the contingency DSL without modifying it.

### The expression AST

The `com.powsybl.dsl.ast` package is a self-contained tree representation of arithmetic/boolean expressions, used by DSLs that need to capture an expression and evaluate it later (rather than evaluating it immediately in Groovy). The central type is `ExpressionNode`, with:

- literal nodes — `IntegerLiteralNode`, `DoubleLiteralNode`, `FloatLiteralNode`, `BigDecimalLiteralNode`, `StringLiteralNode`, `BooleanLiteralNode` (sharing `AbstractLiteralNode` / `LiteralType`);
- operator nodes — `ArithmeticBinaryOperatorNode`, `ComparisonOperatorNode`, `LogicalBinaryOperatorNode`, plus unary nodes, with their operator enums (`ArithmeticBinaryOperator`, `ComparisonOperator`, `LogicalBinaryOperator`, `LogicalNotOperator`);
- the visitor machinery — `ExpressionVisitor` / `DefaultExpressionVisitor`, with `ExpressionEvaluator` (evaluate a tree) and `ExpressionPrinter` (render it back to text) as the main visitors;
- `ExpressionHelper`, a factory of static helpers used to build the nodes.

`ExpressionDslLoader` (Groovy) bridges Groovy syntax to this AST: its `prepareClosures(Binding)` method installs operator overloads on `ExpressionNode` (and on `Integer`, `Float`, `Double`, ...) so that writing `someNode + 2` in a script produces an `ArithmeticBinaryOperatorNode` instead of an immediate numeric result. `DslLoader.createShell` calls `prepareClosures` for every shell it builds.

## The `scripting` module

`scripting` runs ordinary Groovy scripts (not DSLs) against PowSyBl and exposes them on the command line.

### Running scripts

`GroovyScripts` (Groovy, in `com.powsybl.scripting.groovy`) is the entry point, with a family of overloaded `run(...)` methods accepting a `Path` or a `Reader`, an optional `Binding`, an optional output `PrintStream` and an optional set of extensions. When run, it:

1. builds a `CompilerConfiguration` with the `ThreadInterrupt` customiser (same cancellation behaviour as the DSL shell);
2. loads a `ComputationManager` from `DefaultComputationManagerConfig` and binds it as `computationManager` (also placing it in the `contextObjects` map);
3. binds the output stream as `out` when one is provided;
4. loads all `GroovyScriptExtension` services (by default through `ServiceLoader`), calling `load` on each before evaluation and `unload` afterwards (in a `finally` block);
5. checks for thread interruption and evaluates the script in a `GroovyShell`.

### The `run-script` iTools command

`RunScriptTool` is a `Tool` (annotated `@AutoService(Tool.class)`) that registers the `run-script` command under the *Script* theme. It takes a required `--file` option; only `.groovy` files are supported (any other extension raises `IllegalArgumentException`). The command builds a `Binding`, exposes the remaining command-line arguments as `args`, and delegates to `GroovyScripts.run(...)`, sanitising the root cause with Groovy's `StackTraceUtils` on failure. The user-facing documentation is in [run-script](../user/itools/run-script.md).

### The script-extension SPI

`GroovyScriptExtension` is the SPI that lets modules enrich the scripting environment. It declares:

- `load(Binding binding, Map<Class<?>, Object> contextObjects)` — bind functions or variables, typically derived from context objects (`ComputationManager`, `Writer`, ...);
- `unload()` — release whatever was set up.

Implementations declare themselves with `@AutoService(GroovyScriptExtension.class)`. `scripting` itself ships `LogsGroovyScriptExtension`, which redirects the script's `out` to a `Writer` supplied in `contextObjects` when present. Other modules contribute their own extensions discovered the same way — for instance `LoadFlowGroovyScriptExtension` (in `loadflow-scripting`) and `NetworkLoadSaveGroovyScriptExtension` (described below).

The module also carries the `InitPowsybl.properties` resource (command metadata) and a `GroovyShellExtension`-style init used by the interactive shell.

## The `iidm-scripting` module

`iidm-scripting` makes the IIDM model nicer to use from Groovy. It is mostly Groovy code and contains two distinct kinds of contribution.

### Groovy extension methods (metaclass additions)

The bulk of the module is a set of Groovy *extension modules*: classes whose static methods (taking the receiver as the first `self` argument) are grafted onto existing IIDM types at runtime. They are registered through the descriptor `META-INF/groovy/org.codehaus.groovy.runtime.ExtensionModule` (module name `powsybl-iidm-scripting-module`), which lists the extension classes:

| Extension class | Extends |
|-----------------|---------|
| `IdentifiableExtension` | `Identifiable` |
| `NetworkExtension` | `Network` (e.g. `getShunts()` / `getShunt(id)` aliases over shunt compensators) |
| `SubstationExtension` | `Substation` |
| `VoltageLevelExtension` | `VoltageLevel` |
| `ShuntCompensatorExtension` | `ShuntCompensator` |
| `TwoWindingsTransformerExtension`, `ThreeWindingsTransformerExtension` | the transformer types |
| `BatteryExtension`, `BatteryAdderExtension` | `Battery` and its adder |
| `FlowLimitsHolderExtension` | flow-limits holders |
| `BranchExtension` | `Branch` |

These add convenience accessors and shorthand syntax (helped by `AdderSpec`, a small Groovy helper for configuring adders). Because they are wired through Groovy's extension-module mechanism rather than `@AutoService`, they take effect automatically for any Groovy code that has the jar on its classpath — no explicit import or registration is needed. The corresponding `*ExtensionTest.groovy` files under `src/test` exercise each one.

### Script-driven network I/O and import post-processing

The module also provides two `ServiceLoader`-discovered contributions:

- `NetworkLoadSaveGroovyScriptExtension` — a `GroovyScriptExtension` (`@AutoService`) that adds `loadNetwork` and `saveNetwork` closures to the script `Binding`, wrapping `Network.read(...)` / `Network.write(...)` so scripts can read and write networks in one line.
- `GroovyScriptPostProcessor` — an `ImportPostProcessor` (`@AutoService`, name `groovyScript`) that runs a Groovy script after every network import. The script path comes from the `groovy-post-processor` configuration module (`script` property) or defaults to `import-post-processor.groovy` in the configuration directory; the script is evaluated with `network` and `computationManager` bound, again using the `ThreadInterrupt` customiser. This lets a deployment customise imported networks declaratively.
