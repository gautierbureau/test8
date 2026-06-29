# Design: Bundled default `config.yml` in jars

## Status

Proposed.

## Summary

Allow powsybl libraries to **bundle a default `config.yml` inside their jars**, so that
a module ships sensible default configuration out of the box. These bundled defaults form
the **lowest-precedence layer** of the platform configuration and can be overridden, exactly
as today, by:

- the distribution config (`$installDir/etc/config.yml`),
- the user config (`~/.itools/config.yml`),
- environment variables,
- and, for typed parameters, the per-tool `--parameters-file` at runtime.

The inspiration is Spring Boot's
[externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html),
where configuration packaged inside the application is the base layer and external sources
override it.

## Motivation

Today a powsybl module has no way to ship a default `config.yml`. Defaults must either be
hardcoded in Java, dropped into `$installDir/etc` by the packager, or — for a few typed
parameter objects only — provided through a type-specific `*DefaultParametersLoader` SPI
(e.g. `LoadFlowDefaultParametersLoader`).

We want a **single, general mechanism**: any jar can carry curated defaults for any
configuration module, and those defaults compose with and are overridden by the existing
configuration sources.

## Current architecture (recap)

The layering machinery already exists.

- `ModuleConfigRepository` is the source abstraction; `StackedModuleConfigRepository`
  merges a list of repositories **at property granularity** — the first repository that
  defines a property wins, otherwise resolution falls through to the next
  (`StackedModuleConfig`).
- `ClassicPlatformConfigProvider` (module `config-classic`) builds the stack from
  highest to lowest precedence:
  1. `EnvironmentModuleConfigRepository` (`MODULE__PROPERTY` environment variables),
  2. one repository per config directory, in declared order.
- The itools launcher seeds the config directories with
  `powsybl_config_dirs="${HOME}/.itools:$installDir/etc"`, so a user file
  (`~/.itools/config.yml`) already overrides a distribution file
  (`$installDir/etc/config.yml`).
- Each directory repository reads `config.yml` (YAML), else `config.xml` (XML), else
  `.properties` files (`PlatformConfig.loadModuleRepository`).

Separately, typed parameter objects resolve in their own chain. For example
`LoadFlowParameters.load()`:

1. hardcoded constructor defaults,
2. `LoadFlowDefaultParametersLoader` bundled JSON (selected by name), applied in the
   constructor,
3. the `load-flow-default-parameters` module read from `PlatformConfig`,
4. and finally, in the tool, `--parameters-file` JSON via `JsonLoadFlowParameters.update`.

Step 3 reads the **entire** platform-config stack described above, and step 4 (the per-tool
`--parameters-file`) already overrides it. **No change is required to typed-parameter
resolution.**

## The gap

Every existing `ModuleConfigRepository` reads from a filesystem `Path`. Nothing reads a
`config.yml` from the **classpath** (jar resources). That is the only missing capability.

## Design

### Resource convention

A jar ships its defaults at a fixed classpath location:

```
META-INF/powsybl/config.yml
```

(`config.xml` and `.properties` are out of scope for the bundled mechanism; bundled defaults
are YAML only. The on-disk directory repositories continue to support all three formats.)

### Discovery SPI

Bundled defaults are discovered through a `ServiceLoader`-based SPI, mirroring the existing
`LoadFlowDefaultParametersLoader` pattern. This is preferred over a blind
`ClassLoader.getResources("META-INF/powsybl/config.yml")` scan because it gives
**deterministic ordering** and makes each contribution **attributable** (we know which
provider supplied which default when reporting overlaps).

```java
package com.powsybl.commons.config;

public interface DefaultConfigProvider {

    /** Stable name used in logs and to attribute overlapping properties. */
    String getName();

    /**
     * Ordering among bundled defaults. Higher priority wins when two providers
     * define the same property. Ties are broken by name for determinism.
     */
    int getPriority();

    /** Classpath resource, e.g. "/META-INF/powsybl/config.yml". */
    String getResourceName();
}
```

A typical implementation is annotated with `@AutoService(DefaultConfigProvider.class)`.

### New repository

Add a classpath-backed repository in `commons`:

- `ClasspathModuleConfigRepository` — parses a bundled YAML resource into module configs.

To avoid duplicating the YAML parsing, refactor `YamlModuleConfigRepository` (and, where it
helps, `XmlModuleConfigRepository`) to accept an `InputStream`/`URL` in addition to the
current `Path` constructor. The classpath repository reuses that parsing.

### Wiring

In `ClassicPlatformConfigProvider.loadModuleRepository`, append a single bundled-defaults
repository as the **last** (lowest-precedence) entry of the stack:

```
[ env vars ]                                 (highest)
[ ~/.itools/config.yml ]
[ $installDir/etc/config.yml ]
[ bundled defaults: META-INF/powsybl/config.yml from all jars ]   (lowest)   <-- NEW
```

The bundled-defaults repository is itself a `StackedModuleConfigRepository` built from all
discovered `DefaultConfigProvider`s, ordered by descending priority (ties broken by name).

### Overlap policy

When two providers define the **same property of the same module**, resolve by priority
(higher wins, ties broken by name) and emit a `WARN` log naming both providers, the module,
and the property, so collisions are visible. Properties that belong to different modules — or
different properties of the same module — simply compose with no warning.

### Interaction with `*DefaultParametersLoader`

A jar can now ship defaults two ways, at different precedence:

| Mechanism | Enters resolution at | Relative precedence |
|---|---|---|
| `*DefaultParametersLoader` bundled JSON (existing) | typed-param constructor (step 2) | lower |
| Jar-bundled `config.yml` (new) | platform-config stack → typed-param step 3 | **higher** |

**Decision:** the jar-bundled `config.yml` sits **above** `*DefaultParametersLoader`. A
bundled `config.yml` value for a module such as `load-flow-default-parameters` therefore
overrides that module's `*DefaultParametersLoader` JSON default. This matches Spring Boot's
"packaged configuration overrides code defaults" model and keeps a single, predictable
ordering. The two mechanisms coexist: `*DefaultParametersLoader` remains available for
typed JSON defaults, while the general `config.yml` mechanism covers any module uniformly.

## Full precedence (highest wins)

```
Platform configuration (module key/values):
  1. Environment variables          MODULE__PROPERTY
  2. User config dir                 ~/.itools/config.yml
  3. Distribution config dir         $installDir/etc/config.yml
  4. Jar-bundled defaults            META-INF/powsybl/config.yml         (NEW, lowest)

Typed-parameter resolution at tool runtime (e.g. LoadFlowParameters.load()):
  i.   hardcoded constructor defaults
  ii.  *DefaultParametersLoader bundled JSON
  iii. PlatformConfig module override  <-- the entire platform stack above (incl. jar config.yml)
  iv.  --parameters-file JSON          (highest, per-tool; already exists)
```

## Scope

### In scope

- `DefaultConfigProvider` SPI in `commons`.
- `ClasspathModuleConfigRepository` in `commons`, with `InputStream`/`URL` parsing reuse.
- Wiring of the bundled-defaults layer as the lowest entry in `ClassicPlatformConfigProvider`.
- Overlap detection with `WARN` + priority resolution.
- Unit tests and user documentation update.

### Out of scope (possible future work)

- Profiles (`config-{profile}.yml` selected by `powsybl.config.profiles`). The resource
  resolution should be written so profiles can be added later, but they are not built now.
- A generic `-Dmodule.property` system-property source.
- Full `${other.property}` cross-property placeholder resolution (the existing
  `PlatformEnv.substitute` for `${user.home}` etc. is unchanged).
- A new global JSON config override for itools — explicitly **not** needed: the per-tool
  `--parameters-file` is the intended runtime override and already overrides `config.yml`.
- Bundled defaults in XML or `.properties` form.

## Backward compatibility

The change is additive. The new layer is the lowest in the stack, so existing deployments
that rely on `~/.itools`, `$installDir/etc`, environment variables, or `--parameters-file`
keep their current behaviour. Jars that do not ship `META-INF/powsybl/config.yml`
contribute nothing.

## Testing

- Single jar: a bundled `config.yml` value is read when no other source defines it.
- Multi-jar merge: non-overlapping modules/properties compose; overlapping properties
  resolve by priority and emit a `WARN`.
- Override precedence: `~/.itools/config.yml`, `$installDir/etc/config.yml`, environment
  variables, and `--parameters-file` each override a bundled default.
- Interaction: a bundled `config.yml` value overrides the corresponding
  `*DefaultParametersLoader` JSON default.

## Open questions

None blocking. Profiles and placeholder resolution are deferred by design.
