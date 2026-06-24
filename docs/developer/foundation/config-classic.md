# `config-classic`

`config-classic` is a deliberately tiny module: it contains a single class, `ClassicPlatformConfigProvider` (package `com.powsybl.config.classic`). Its only job is to provide the standard implementation of the `PlatformConfigProvider` SPI defined in [`commons`](commons.md#configuration-framework) — that is, to decide *which* `PlatformConfig` `PlatformConfig.defaultConfig()` returns in a normal iTools distribution.

## Purpose and role

Recall from the [configuration framework](commons.md#configuration-framework) that `PlatformConfig.defaultConfig()` looks up, through `ServiceLoader`, the single `PlatformConfigProvider` on the classpath. `commons` itself ships no provider, so without one the default configuration is empty. `config-classic` fills that role with the "classic" provider, registered with `@AutoService(PlatformConfigProvider.class)` and named `"classic"`.

Keeping this provider in its own jar is an intentional application of the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface): an application can swap configuration strategies simply by replacing the jar on the classpath. For tests, the separate `powsybl-config-test` artifact provides an in-memory provider instead; in production the distribution bundles `config-classic`.

## How configuration is located and loaded

`ClassicPlatformConfigProvider.getPlatformConfig()` builds the configuration in three steps.

### 1. Resolve the configuration directories

The list of directories comes from system properties, in order of precedence:

- `powsybl.config.dirs`, else `itools.config.dir` — a path-separator-separated list of directories;
- if neither is set, it defaults to a single directory, `${HOME}/.itools`.

Each entry is passed through `PlatformEnv.substitute(...)` (from `commons`), so keywords such as `${user.home}` / `$HOME` and `${app.root}` are expanded before the path is resolved. The directory-resolution logic is exposed as the static, testable `getDefaultConfigDirs(fileSystem, directories, userHome, pathSeparator)`. The configuration **name** (the file base name, default `config`) likewise comes from `powsybl.config.name` / `itools.config.name`.

### 2. Load one repository per directory

For each directory, the provider delegates to `commons`' `PlatformConfig.loadModuleRepository(configDir, configName)`, which picks the backend by file existence: `<configName>.yml` → `YamlModuleConfigRepository`, else `<configName>.xml` → `XmlModuleConfigRepository`, else the directory's `.properties` files → `PropertiesModuleConfigRepository`. So YAML wins over XML, which wins over properties, **per directory**.

### 3. Stack the repositories (with environment variables on top)

The per-directory repositories are combined into a single `StackedModuleConfigRepository`, with an `EnvironmentModuleConfigRepository` placed **first** so that environment variables override file-based configuration. Within the file repositories, directories earlier in the list take precedence over later ones (this is the *stacking* behaviour described in the user-facing [configuration](../../user/configuration/index.md) section). The resulting precedence, highest to lowest:

1. environment variables (`MODULE_NAME__PROPERTY_NAME`);
2. the first configuration directory;
3. subsequent configuration directories in order.

Finally `getPlatformConfig()` wraps the stacked repository in a `PlatformConfig`, using the first configuration directory as the config directory (the location relative to which path properties and bundled resources are resolved).

```java
// Effective behaviour of getPlatformConfig()
String directories = System.getProperty("powsybl.config.dirs",
                                         System.getProperty("itools.config.dir"));
String configName  = System.getProperty("powsybl.config.name",
                                         System.getProperty("itools.config.name", "config"));
Path[] configDirs = getDefaultConfigDirs(fileSystem, directories,
                                         System.getProperty("user.home"), File.pathSeparator);
ModuleConfigRepository repository = loadModuleRepository(configDirs, configName);
return new PlatformConfig(repository, configDirs[0]);
```

## Design notes

The whole module is essentially a thin assembly of `commons` building blocks (`PlatformEnv`, `PlatformConfig.loadModuleRepository`, `StackedModuleConfigRepository`, `EnvironmentModuleConfigRepository`); it adds no new configuration model of its own. The two helper methods that carry the logic — `getDefaultConfigDirs(...)` and the package-private `loadModuleRepository(...)` — are written as pure, file-system-injectable functions, which is what lets the test suite exercise directory resolution and stacking against an in-memory file system (Jimfs) without touching the real environment.
