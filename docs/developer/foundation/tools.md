# Tools

The `tools` module implements the **iTools** command-line framework — the engine behind the `itools` executable shipped in a PowSyBl distribution. A command is just an implementation of the `Tool` service interface discovered on the classpath, so adding a new sub-command requires no registration beyond dropping a jar in. Everything lives under `com.powsybl.tools`.

This page is part of the [foundation layer](../foundation.md). Command discovery relies on the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface).

## Defining a command: `Tool` and `Command`

The `Tool` SPI is intentionally minimal:

```java
public interface Tool {
    Command getCommand();
    void run(CommandLine line, ToolRunningContext context) throws Exception;
}
```

`getCommand()` returns the command's metadata and `run(...)` is the body, receiving the parsed Apache Commons CLI `CommandLine` and a `ToolRunningContext`. `Command` is the metadata description:

| Method | Meaning |
|--------|---------|
| `String getName()` | the sub-command name typed on the CLI |
| `String getTheme()` | a grouping label used when listing commands in the help |
| `String getDescription()` | one-line description |
| `Options getOptions()` | the Commons CLI options the command accepts |
| `String getUsageFooter()` | extra text printed after the option list (may be null) |
| `default boolean isHidden()` | when `true`, the command is omitted from the help listing |

A tool registers itself with `@AutoService(Tool.class)`; the generated `META-INF/services` descriptor is what lets `ServiceLoader` find it at runtime. The three built-in tools in this module (`VersionTool`, `PluginsInfoTool`, `BashCompletionTool`) all follow this pattern.

## Dispatching: `CommandLineTools` and `Main`

`CommandLineTools` is the dispatcher. By default it loads every `Tool` through `ServiceLoader`; an `Iterable<Tool>` can also be injected (used in tests). Its `run(String[] args, ToolInitializationContext initContext)` drives a run end to end:

1. validate the arguments and resolve the first token to a `Tool` via `findTool(name)`;
2. if the command is unknown, print the usage (commands grouped by theme) and return a status code;
3. otherwise merge the command's `Options` with the global options, parse the line, and — unless `--help` was requested — create the short- and long-time `ComputationManager`s from the `ToolInitializationContext` and invoke `tool.run(...)` inside a `ToolRunningContext`.

The outcome is one of four documented status codes:

| Constant | Value | Meaning |
|----------|-------|---------|
| `COMMAND_OK_STATUS` | 0 | success |
| `COMMAND_NOT_FOUND_STATUS` | 1 | no command matched |
| `INVALID_COMMAND_STATUS` | 2 | argument parsing failed (`ParseException`) |
| `EXECUTION_ERROR_STATUS` | 3 | the tool threw during execution |

`Main` is the `itools` entry point. It loads a `DefaultComputationManagerConfig`, wraps `System.out` / `System.err`, the default `FileSystem` and that config in an inline `ToolInitializationContext`, calls `new CommandLineTools().run(args, initContext)`, and exits with the returned status code if it is non-zero.

## Execution context

The framework separates the context needed to *set up* a run from the context handed to each tool.

`ToolInitializationContext` is the setup contract: it exposes the output and error `PrintStream`s, a `FileSystem`, any `getAdditionalOptions()` to merge into every command (the global options), and two factory methods — `createShortTimeExecutionComputationManager(CommandLine)` and `createLongTimeExecutionComputationManager(CommandLine)` — so computation managers are created lazily, only when a command actually runs.

`ToolRunningContext` is what `Tool.run(...)` receives:

- `getOutputStream()` / `getErrorStream()` — the `PrintStream`s to write to;
- `getFileSystem()` — the `FileSystem` to resolve paths against (so tools are testable on an in-memory filesystem);
- `getShortTimeExecutionComputationManager()` and `getLongTimeExecutionComputationManager()` — the two [computation managers](computation.md). When no long-time manager is configured, the getter transparently falls back to the short-time one.

`ToolOptions` wraps a parsed `CommandLine` with null-safe, typed accessors returning `Optional`s — `getValue`, `getInt`, `getFloat`, `getDouble`, `getValues` (comma-separated list), `getEnum`, `getPath` (resolved against the supplied `FileSystem`) and `hasOption`. `CommandLineUtil.getOptionValue(...)` is a small companion that reads an enum option with a default. `ToolConstants` holds the shared `task` / `task-count` option names used for partitioned executions.

## Built-in tools

Two general-purpose tools ship with the module:

- `VersionTool` (`itools version`) — a hidden tool that prints a table of all PowSyBl component versions. Version information is itself an SPI: `Version` implementations (extending `AbstractVersion`, which carries the repository name, Maven version, git version/branch and build timestamp) are discovered through `ServiceLoaderCache`, and `Version.getTableString(...)` renders the table. A module supplies its own version object — `tools` does so through the generated `PowsyblCoreVersion`, annotated `@AutoService(Version.class)` and filled with Maven build placeholders.
- `PluginsInfoTool` (`itools plugins-info`, theme "Misc") — lists the available plugins by querying `Plugins` from `commons` and printing, for each plugin category, the implementation ids found on the classpath. It is the command-line view of the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface).

## Bash autocompletion

The `com.powsybl.tools.autocompletion` sub-package generates a Bash completion script for the whole tool set. `BashCompletionTool` (`itools generate-completion-script`) introspects every registered `Tool`, converts each `Command`'s Commons CLI `Options` into a simplified model — `BashCommand` holding a list of `BashOption`s, each tagged with an `OptionType` (`FILE`, `DIRECTORY`, `HOSTNAME`, or an `ENUMERATION` with its possible values) — and feeds them to a `BashCompletionGenerator`. The default `StringTemplateBashCompletionGenerator` renders the script from a StringTemplate template (`completion.sh.stg`). `OptionTypeMapper` infers option types from regex rules on option or argument names, so file/directory/hostname options complete appropriately in the shell.
