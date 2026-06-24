# itools-packager

`itools-packager` is the Maven plugin that turns a set of resolved jars into a self-contained [iTools](../../user/itools/index.md) installation an end user can unzip and run. It is packaged as `maven-plugin` under the artifact id `powsybl-itools-packager-maven-plugin` and exposes a single goal. See the [distribution layer overview](../distribution.md) for how it pairs with `distribution-core`, and the user-facing reference at [itools-packager](../../user/itools/itools-packager.md).

## The `package-zip` goal

The whole plugin is one Mojo, `ItoolsPackagerMojo` (`com.powsybl.itools`), bound to the `package-zip` goal. It is annotated:

```java
@Mojo(name = "package-zip",
      requiresDependencyCollection = ResolutionScope.RUNTIME,
      requiresDependencyResolution = ResolutionScope.RUNTIME)
```

so Maven resolves the project's runtime classpath before invoking it and exposes it to the Mojo as `project.getArtifacts()`. Despite the goal name, the produced archive can be a `zip` or a `tgz`, selected by `packageType`.

### Produced layout

`execute()` builds the following tree under `target/<packageName>/`:

```text
<package-name>
    bin
        itools
        itools.bat
        powsyblsh
        <files from copyToBin>
    lib
        <files from copyToLib>
    etc
        logback-itools.xml
        logback-powsyblsh.xml
        itools.conf
        <files from copyToEtc>
    share
        java
            <every runtime/compile jar of the project>
    <files from copyToPackageRoot>
    LICENSE(.txt)
    THIRD-PARTY(.txt)
```

Step by step, `execute()`:

1. **`share/java`** — copies every resolved artifact (`project.getArtifacts()`) into `share/java`. This is the runtime classpath of the distribution: thanks to the PowSyBl plugin mechanism, every tool, importer/exporter, simulation provider and extension on it is auto-discovered at run time (see [the plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface)).
2. **`bin`** — copies the three launcher scripts `itools`, `itools.bat` and `powsyblsh` (bundled as plugin resources) and, where the filesystem supports `PosixFileAttributeView`, sets their POSIX permissions to owner read/write/execute plus group read/execute. Extra binaries from `copyToBin` are added.
3. **`etc`** — writes the two bundled logback configurations (`logback-itools.xml`, `logback-powsyblsh.xml`) and generates `itools.conf` (see below). Extra files from `copyToEtc` are added.
4. **`lib`** — created and filled from `copyToLib` (e.g. native libraries).
5. **package root** — files from `copyToPackageRoot` are copied to the top of the package.
6. **license files** — `addLicenseFiles` copies a license and a third-party-notice file into the package.
7. **archive** — the directory tree is compressed; any `packageType` other than `zip`/`tgz` raises `IllegalArgumentException`.

`copyFiles` (used for all the `copyToXxx` options) logs and skips files that do not exist rather than failing the build.

### `itools.conf`

`writeItoolsConf` emits three lines: a commented-out `#powsybl_config_dirs=`, `powsybl_config_name=<configName>` and `java_xmx=<javaXmx>`. The file is read by the `itools` launcher at startup.

### License resolution

`addLicenseFiles` handles two files, the license and the third-party notice. For each, `getFilePathList` builds a candidate list: if an explicit `licenseFile` / `thirdPartyFile` is set, it looks for that name in the project directory and its parent; otherwise it searches the default base names `LICENSE` / `THIRD-PARTY` with and without a `.txt` suffix, in the project directory and its parent. The first existing candidate is copied; if none is found a warning is logged (the build is not failed). The bundled test projects under `src/test/resources` cover the configured-name, default-name, parent-directory and not-found cases.

### Archiving and file modes

`zip` and `targz` both walk the package tree and preserve Unix file modes, marking executable files `0100770` and the rest `0100660`; the tar writer additionally enables `LONGFILE_POSIX` and `BIGNUMBER_POSIX` to tolerate long paths and large GIDs. Compression uses Apache Commons Compress.

## Configuration parameters

The Mojo exposes the following Maven parameters:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `packageName` | project final name | Name of the package directory under `target`. |
| `archiveName` | `packageName` | Base name of the produced archive. |
| `packageType` | `zip` | `zip` or `tgz`. |
| `javaXmx` | `8G` | Written into `itools.conf` as `java_xmx` (JVM max heap). |
| `configName` | `config` | Written into `itools.conf` as `powsybl_config_name`. |
| `copyToBin`, `copyToLib`, `copyToEtc`, `copyToPackageRoot` | — | Extra files to drop into the matching folder (each holds a `files` list, via the nested `CopyTo` type). |
| `licenseFile`, `thirdPartyFile` | auto-detected | Explicit license / third-party files. |

> Note: the module's `README.md` also mentions `mpiTasks`/`mpiHosts` parameters and a `tools-mpi-task.sh` script; these are not present in the current `ItoolsPackagerMojo`, which implements only the parameters above.

## The launcher scripts

The bundled `bin` scripts are thin shell wrappers, not Java:

- **`itools`** (Bash) resolves `JAVA_HOME`/`java`, derives the install directory from its own location, sources `etc/itools.conf`, and supports a `--config-name` argument that overrides `powsybl_config_name`. It sets `-Dpowsybl.config.dirs` (default `~/.itools:<install>/etc`), `-Dpowsybl.config.name` and `-Dlogback.configurationFile` (the first `logback-itools.xml` found across the config dirs, else `etc/logback-itools.xml`), then launches `com.powsybl.tools.Main` with `-Xmx$java_xmx` and the classpath `share/java/*`. `com.powsybl.tools.Main` is the iTools command dispatcher from the foundation `tools` module.
- **`itools.bat`** is the Windows equivalent.
- **`powsyblsh`** (Bash) launches an interactive Groovy shell (`groovysh`, via `GROOVY_HOME` or the `PATH`) configured against the same distribution and `logback-powsyblsh.xml`, registering and running the `com.powsybl.scripting.groovy.InitPowsybl` initialiser shipped by the [`scripting`](../scripting/scripting.md) module.
