# scripting

The `scripting` module runs ordinary Groovy scripts (not DSLs) against PowSyBl and exposes that capability on the command line through the `run-script` iTools command. It also defines the `GroovyScriptExtension` SPI that lets any module enrich the scripting environment with extra bindings. See the [scripting layer overview](../scripting.md) for how it relates to `dsl` and `iidm-scripting`.

It is a small mixed module: the script runner and its bundled extension are Groovy (`com.powsybl.scripting.groovy`), the iTools command and the SPI are Java.

## Running scripts

`GroovyScripts` (Groovy, `com.powsybl.scripting.groovy`) is the entry point. It is a family of overloaded static `run(...)` methods accepting a `Path` or a `Reader`, with optional `Binding`, optional output `PrintStream`, an optional `Iterable<GroovyScriptExtension>` and an optional `Map<Class<?>, Object>` of context objects. The shorter overloads delegate to the canonical one:

```groovy
static void run(Reader codeReader, Binding binding,
                Iterable<GroovyScriptExtension> extensions,
                PrintStream out, Map<Class<?>, Object> contextObjects)
```

When this method runs it:

1. builds a `CompilerConfiguration` with the `ThreadInterrupt` AST customiser, giving scripts the same loop-level cancellation behaviour as the DSL shell;
2. loads a `ComputationManager` from `DefaultComputationManagerConfig` (`createShortTimeExecutionComputationManager()`), binds it as `computationManager` and also stores it in `contextObjects` under `ComputationManager.class`;
3. binds the supplied output stream as `out` when one is provided;
4. calls `load(binding, contextObjects)` on every `GroovyScriptExtension` (by default discovered through `ServiceLoader`), evaluates the script in a `GroovyShell`, and — crucially in a `finally` block — calls `unload()` on each extension;
5. checks `Thread.currentThread().isInterrupted()` immediately before evaluation so an already-cancelled run never starts.

The default `run(Reader, Binding, PrintStream)` overload supplies the `ServiceLoader`-discovered extensions, so callers normally do not have to manage extensions themselves.

## The `run-script` iTools command

`RunScriptTool` is a `Tool` (annotated `@AutoService(Tool.class)`, see [the plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface)) registering the `run-script` command under the *Script* theme. Its command declares a single required `--file` option. `run(...)`:

- resolves the file against the tool's `FileSystem`;
- accepts only `.groovy` files — any other extension raises `IllegalArgumentException("Script type not supported")`;
- builds a `Binding`, exposes the remaining command-line arguments as `args`, and delegates to `GroovyScripts.run(file, binding, context.getOutputStream())`;
- on failure, sanitises the root cause with Groovy's `StackTraceUtils.sanitizeRootCause(e)` and prints it to the tool's error stream rather than letting a noisy Groovy stack trace escape.

The user-facing reference for the command is [run-script](../../user/itools/run-script.md).

## The script-extension SPI

`GroovyScriptExtension` is the SPI through which modules contribute to the scripting environment. It has two methods:

- `load(Binding binding, Map<Class<?>, Object> contextObjects)` — bind functions or variables, typically derived from context objects (`ComputationManager`, `Writer`, ...);
- `unload()` — release whatever was set up.

Implementations declare themselves with `@AutoService(GroovyScriptExtension.class)` and are picked up by `GroovyScripts` via `ServiceLoader`. The module ships one implementation, `LogsGroovyScriptExtension` (Groovy), which looks up a `Writer` in `contextObjects` and, when present, rebinds the script's `out` to it — letting an embedding application redirect script output to its own writer. Other modules add their own extensions discovered the same way, for instance `NetworkLoadSaveGroovyScriptExtension` in [`iidm-scripting`](iidm-scripting.md) and `LoadFlowGroovyScriptExtension` in `loadflow-scripting`.

## Interactive shell metadata

The module also carries the resource `com/powsybl/scripting/groovy/InitPowsybl.properties`, holding the command metadata (`command.description`, `command.usage`, `command.help`) used to initialise PowSyBl inside the interactive `powsyblsh` Groovy shell — the `:register com.powsybl.scripting.groovy.InitPowsybl` / `:init_powsybl` step run by the `powsyblsh` launcher shipped by [`itools-packager`](../distribution/itools-packager.md).
