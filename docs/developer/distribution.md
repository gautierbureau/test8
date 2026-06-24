# Distribution layer

The distribution layer turns the many PowSyBl Core modules into a single, self-contained [iTools](../user/itools/index.md) installation that an end user can unzip and run. It has two modules:

| Module | Role |
|--------|------|
| `itools-packager` | A Maven plugin (`powsybl-itools-packager-maven-plugin`) that assembles an iTools package and archives it. |
| `distribution-core` | An aggregation module that lists the jars to ship and invokes the packager to produce the official PowSyBl distribution. |

```{toctree}
:hidden:
distribution/itools-packager
distribution/distribution-core
```

## The `itools-packager` module

`itools-packager` is a Maven plugin packaged as `maven-plugin` under the artifact id `powsybl-itools-packager-maven-plugin`. Its single goal builds the on-disk layout of an iTools installation and compresses it.

### The `package-zip` goal

The whole plugin is one Mojo, `ItoolsPackagerMojo`, bound to the `package-zip` goal. It is declared with `requiresDependencyResolution = RUNTIME`, so Maven resolves the project's runtime classpath and hands it to the Mojo as `project.getArtifacts()`. Despite its name the goal can produce either a `zip` or a `tgz` archive (selected by `packageType`).

Running `execute()` produces the following layout under `target/<packageName>/`:

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

1. **`share/java`** — copies every resolved artifact (`project.getArtifacts()`) into `share/java`. This is the runtime classpath of the distribution: thanks to the PowSyBl plugin mechanism, every tool, importer/exporter, simulation provider and extension on the classpath is auto-discovered at run time (see [the plugin mechanism](index.md#the-plugin-mechanism-service-provider-interface)).
2. **`bin`** — copies the launcher scripts `itools`, `itools.bat` and `powsyblsh` (bundled as plugin resources) and marks them executable using POSIX file permissions where the filesystem supports it. Extra binaries listed under `copyToBin` are added.
3. **`etc`** — writes the two bundled logback configurations (`logback-itools.xml`, `logback-powsyblsh.xml`) and generates `itools.conf`. Extra files from `copyToEtc` are added.
4. **`lib`** — created and filled from `copyToLib` (e.g. native libraries).
5. **package root** — files from `copyToPackageRoot` are copied to the top of the package.
6. **license files** — `addLicenseFiles` copies a license and a third-party-notice file into the package. Their paths can be set with `licenseFile` / `thirdPartyFile`; otherwise the packager searches for `LICENSE` / `LICENSE.txt` and `THIRD-PARTY` / `THIRD-PARTY.txt` in the project directory and its parent.
7. **archive** — finally the directory is compressed into `<archiveName>.zip` (`zip`) or `<archiveName>.tgz` (`targz`); any other `packageType` raises an error. Both writers preserve Unix file modes, marking executable files `0770` and the rest `0660`.

### Configuration parameters

The Mojo exposes the following Maven parameters:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `packageName` | project final name | Name of the package directory under `target`. |
| `archiveName` | `packageName` | Base name of the produced archive. |
| `packageType` | `zip` | `zip` or `tgz`. |
| `javaXmx` | `8G` | Written into `itools.conf` as the JVM max heap. |
| `configName` | `config` | Written into `itools.conf` as `powsybl_config_name`. |
| `copyToBin`, `copyToLib`, `copyToEtc`, `copyToPackageRoot` | — | Extra files to drop into the matching folder (each holds a `files` list). |
| `licenseFile`, `thirdPartyFile` | auto-detected | Explicit license / third-party files. |

The generated `itools.conf` contains a commented `powsybl_config_dirs`, the `powsybl_config_name` and the `java_xmx` values; it is read by the `itools` launcher. The user-facing reference is [itools-packager](../user/itools/itools-packager.md).

> Note: the module's `README.md` also mentions `mpiTasks`/`mpiHosts` parameters and a `tools-mpi-task.sh` script; these are not present in the current `ItoolsPackagerMojo` and only the parameters listed above are implemented.

### The launcher scripts

The bundled `bin` scripts are thin shell wrappers, not Java:

- **`itools`** (Bash) resolves `JAVA_HOME`/`java`, sources `etc/itools.conf`, supports a `--config-name` argument, sets `-Dpowsybl.config.dirs` (default `~/.itools:<install>/etc`), `-Dpowsybl.config.name` and `-Dlogback.configurationFile`, then launches `com.powsybl.tools.Main` with `-Xmx$java_xmx` and the classpath `share/java/*`. `com.powsybl.tools.Main` is the iTools command dispatcher (from the foundation `tools` module).
- **`itools.bat`** is the Windows equivalent.
- **`powsyblsh`** launches an interactive Groovy shell (`groovysh`, via `GROOVY_HOME` or the `PATH`) configured against the same distribution, using `logback-powsyblsh.xml`.

## The `distribution-core` module

`distribution-core` is a `pom`-packaging aggregation module (`powsybl-distribution-core`); it contains **no Java source code** — only its `pom.xml`. Its job is twofold:

1. **Declare the contents of the distribution.** It depends on essentially every shippable PowSyBl Core module (the IIDM model and its serde/extensions/modification/criteria/geodata sub-modules, every converter — CGMES, UCTE, PSSE, MATPOWER, PowerFactory, IEEE-CDF, AMPL —, the simulation APIs, the contingency/action/DSL/scripting modules, `triple-store`, `commons`, `math`, `computation(-local)`, `tools`, ...), plus the `logback-classic` and `log4j-over-slf4j` runtime logging dependencies. These runtime/compile dependencies are exactly what the packager copies into `share/java`. Modules that should be present for build/coverage but **not** shipped are pulled in `provided` or `test` scope (e.g. the various `*-tck` / `*-test` artifacts), so they stay out of the final classpath.
2. **Invoke the packager.** Its `build` section binds `powsybl-itools-packager-maven-plugin`'s `package-zip` goal to the `package` phase, with `packageName=powsybl` and `javaXmx=8G`, and uses `copyToEtc` to add the IIDM XSD schemas (`iidm_V1_0`..`iidm_V1_3`) to `etc`. Building this module therefore yields the runnable `powsybl` iTools distribution archive.

The module also defines a `checks` profile (dependency-convergence enforcement, non-failing) and a `jacoco` profile producing an aggregated coverage report across the whole distribution — which is why some test-only artifacts are listed as dependencies.
