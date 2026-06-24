# Computation

The computation layer abstracts *where* and *how* an external computation runs, so that the rest of PowSyBl can submit work without knowing whether it executes as a local process or on a remote cluster. It spans two modules: `computation` (under `com.powsybl.computation`) defines the abstraction, and `computation-local` (under `com.powsybl.computation.local`) provides the default implementation that runs commands as local OS processes. Putting `computation-local` on the classpath is what makes `itools` compute locally out of the box.

This page is part of the [foundation layer](../foundation.md). The choice of implementation is itself driven by the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface).

## Package structure

| Module / package | Content |
|------------------|---------|
| `computation` / `com.powsybl.computation` | `ComputationManager`, the job and command model, the factory SPI |
| `computation` / `com.powsybl.computation.util` | `AbstractExecutor` lifecycle helper |
| `computation-local` / `com.powsybl.computation.local` | `LocalComputationManager` and the OS-specific command executors |

## The central abstraction: `ComputationManager`

`ComputationManager` is an `AutoCloseable` whose two `execute(...)` overloads submit a job and return a `CompletableFuture<R>`, making execution asynchronous and cancellable (cancelling the future asks the manager to free the underlying resources):

```java
<R> CompletableFuture<R> execute(ExecutionEnvironment environment, ExecutionHandler<R> handler);

default <R> CompletableFuture<R> execute(ExecutionEnvironment environment,
                                         ExecutionHandler<R> handler,
                                         ComputationParameters parameters);
```

It also exposes `getVersion()`, `newCommonFile(String)` (a file shared across executions), `getResourcesStatus()`, `getExecutor()` (an in-JVM executor for expensive local processing), and `getLocalDir()`. The manager guarantees each submitted job a temporary working directory, named from `ExecutionEnvironment`'s prefix plus a UUID, and kept on disk when `ExecutionEnvironment.isDebug()` is `true` (otherwise discarded). The submitted `CommandExecution`s run sequentially, but a single command with an execution count greater than one may run its instances in parallel.

### Defining a job: `ExecutionHandler`

`ExecutionHandler<R>` is the **template** for a job, splitting it into pre- and post-processing around the actual command runs:

```java
List<CommandExecution> before(Path workingDir) throws IOException;
void onExecutionStart(CommandExecution execution, int executionIndex);
void onExecutionCompletion(CommandExecution execution, int executionIndex);
R after(Path workingDir, ExecutionReport report) throws IOException;
```

`before(...)` writes the input files into the working directory and returns the commands to run; `after(...)` reads the results back and produces the typed business result `R`. `AbstractExecutionHandler<R>` is the usual base: it provides empty `onExecution*` hooks and an `after(...)` that inspects the `ExecutionReport` and raises a `PowsyblException` if any command failed, so subclasses normally override only `before` and `after`. Progress can also be observed externally via `ExecutionListener` (no-op base `DefaultExecutionListener`).

### Describing the environment and the report

`ExecutionEnvironment` carries the run context: a map of environment `variables`, a `workingDirPrefix`, a `debug` flag and an optional `dumpDir`. `ExecutionEnvironment.createDefault()` returns an empty environment with the `itools` prefix and debug off, and fluent setters tune it.

`ExecutionReport` reports the outcome: `getErrors()` returns a list of `ExecutionError` (each pairing a `Command`, an execution index and an exit code), and `getStdOut`/`getStdErr` give optional access to the captured streams. `DefaultExecutionReport` reads those streams from the working directory. For richer failure propagation, `ComputationException` (a `PowsyblException`) bundles captured out/err logs and arbitrary files — built fluently with `ComputationExceptionBuilder` and serializable to a zip via `toZipBytes()`. `ComputationResourcesStatus` reports the snapshot date, available and busy cores.

## The command model

Commands are modelled separately from their execution. `Command` describes *what* to run — `getId()`, `getType()`, `getInputFiles()`, `getOutputFiles()` and `toString(int executionNumber)` — and the `CommandType` enum distinguishes the two kinds:

- `SIMPLE` — a single program with arguments. `SimpleCommand` adds `getProgram()` and `getArgs(int executionNumber)`, implemented by `SimpleCommandImpl`.
- `GROUP` — a sequence of sub-commands run in order. `GroupCommand` exposes `getSubCommands()`, each `SubCommand` having its own program, args and timeout; implemented by `GroupCommandImpl`.

Commands are assembled with the **builder** pattern on the `AbstractCommandBuilder` / `AbstractCommand` bases. `SimpleCommandBuilder` offers `program(...)`, `args(...)`, and the conveniences `arg`, `flag(name, condition)` and `option(name, value)`; `GroupCommandBuilder` nests a `SubCommandBuilder` (with `add()` to append a sub-command). Both produce immutable command objects.

A `CommandExecution` wraps a `Command` with the number of times to run it (`executionCount`), a priority, optional `tags`, and `overloadedVariables` that override the environment for that command. The static `getExecutionVariables(...)` merges the base and overloaded variables.

### Files and partitioning

Input and output files are first-class so the manager can move and transform them around the run:

- `InputFile` and `OutputFile` carry a name and an optional processor. `FilePreProcessor` (`ARCHIVE_UNZIP`, `FILE_GUNZIP`) is applied to inputs before the run; `FilePostProcessor` (`FILE_GZIP`) to outputs after. File names are computed through the `FileName` strategy — `StringFileName` (a fixed name, where the `${EXEC_NUM}` token is substituted with the execution number) and `FunctionFileName` (a name computed from the execution number) — so a parallel command can read and write per-execution files.
- `Partition` models a "`taskIndex/taskCount`" split (`Partition.parse("2/5")`), with `startIndex(size)` / `endIndex(size)` to carve a workload across parallel executions.

`ComputationParameters` (an `Extendable`, so it is itself extensible) carries optional technical parameters — per-command `getTimeout` (execution time) and `getDeadline` (including queue wait) — built with `ComputationParametersBuilder`; `ComputationParameters.empty()` is the neutral value.

## Choosing the implementation

Which `ComputationManager` is used is configurable. `ComputationManagerFactory` is the SPI (`ComputationManager create()`), and `DefaultComputationManagerConfig` reads from `PlatformConfig` which factory to use for short- and long-time executions (defaulting the short-time factory to `LocalComputationManagerFactory` by class name, to avoid a hard dependency on `computation-local`). `LazyCreatedComputationManager` wraps a factory and defers creation until first use. The `tools` module's `Main` builds its two managers from `DefaultComputationManagerConfig`.

## The local implementation

`computation-local` is the default backend: it runs each command as a local OS process. `LocalComputationManager` implements `ComputationManager`, and `LocalComputationManagerFactory` is the `@AutoService(ComputationManagerFactory.class)` registration that makes it discoverable — so adding this module to the classpath is all it takes to compute locally. `LocalComputationManager.getDefault()` returns a shared, shutdown-hooked instance.

Configuration comes from `LocalComputationConfig` (module `computation-local`): the working directory (`tmp-dir`, defaulting to the JVM temp dir) and the number of available cores (`available-core`, defaulting to 1; a non-positive value means use `Runtime.availableProcessors()`). The manager enforces that core count with a `Semaphore`, tracks usage through `LocalComputationResourcesStatus` (a `ComputationResourcesStatus`), and runs jobs on an `Executor` (a `ForkJoinPool` by default). For each `CommandExecution` it runs the per-execution pre-process → process → post-process pipeline, applying the file processors (gunzip/unzip on the way in, gzip on the way out), and collects an `ExecutionReport`.

### Running the process: `LocalCommandExecutor`

Process launching is abstracted behind `LocalCommandExecutor`, a **strategy** selected per operating system:

```java
int execute(String program, List<String> args, Path outFile, Path errFile,
            Path workingDir, Map<String, String> env) throws IOException, InterruptedException;
default int execute(String program, long timeoutSecondes, List<String> args, /* … */);
void stop(Path workingDir);
void stopForcibly(Path workingDir) throws InterruptedException;
```

`AbstractLocalCommandExecutor` holds the common machinery: it builds a `ProcessBuilder` redirecting stdout/stderr to files, tracks the live `Process` per working directory (so `stop` can `destroy()` it and `stopForcibly` `destroyForcibly()` it), and logs non-zero exit codes. `UnixLocalCommandExecutor` and `WindowsLocalCommandExecutor` specialize the command line — Unix wraps it as `bash -c "VAR=val … program 'arg'…"`, Windows as `cmd /c "setlocal & set VAR=val & … program "arg"… & endlocal"` — and set the temp-directory variables appropriately. `LocalComputationManager` picks the right executor from the detected OS, and `ProcessHelper.runWithTimeout(...)` enforces per-command timeouts (returning exit code `124` when a process is killed for exceeding its deadline).
