# distribution-core

`distribution-core` (`powsybl-distribution-core`) is the aggregation module that produces the official PowSyBl Core iTools distribution. It is a `pom`-packaging module and contains **no Java source code** — only its `pom.xml`. Its single responsibility is to declare *what* ships and to invoke the [`itools-packager`](itools-packager.md) plugin to assemble it. See the [distribution layer overview](../distribution.md) for the broader picture.

Being source-free, there is nothing to describe class-by-class here; the module is entirely defined by its dependency set and its `build`/`profiles` configuration.

## Declaring the contents of the distribution

The module depends on essentially every shippable PowSyBl Core module. Through the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface), whatever ends up on the runtime/compile classpath is exactly what the packager copies into `share/java`, and is therefore auto-discovered at run time. The dependency list includes, among others:

- the IIDM model and its sub-modules — `iidm-api`, `iidm-impl`, `iidm-serde`, `iidm-extensions`, `iidm-modification`, `iidm-criteria`, `iidm-geodata`, `iidm-reducer`, `iidm-comparator`, `iidm-scripting`;
- every converter — CGMES (`cgmes-conversion`, `cgmes-model`, `cgmes-extensions`, `cgmes-completion`, `cgmes-conformity`, `cgmes-gl`, `cgmes-measurements`, `cgmes-shortcircuit`), UCTE, PSSE, MATPOWER, PowerFactory, IEEE-CDF, AMPL (`ampl-converter`, `ampl-executor`), plus `cim-anonymiser` and `entsoe-util`;
- the simulation APIs and their tooling — `loadflow-api`, `loadflow-scripting`, `loadflow-validation`, `loadflow-results-completion`, `security-analysis-api`, `sensitivity-analysis-api`, `shortcircuit-api`, `dynamic-simulation-api`, `dynamic-security-analysis`;
- the contingency / action / DSL / scripting stack — `contingency-api`, `contingency-dsl`, the `action-ial-*` modules, `dsl`, `dynamic-simulation-dsl`, `dynamic-simulation-tool`, `time-series-dsl`, `scripting`;
- the foundation modules — `commons`, `math`, `computation`, `computation-local`, `config-classic` (runtime), `tools`, `time-series-api`, `triple-store-api` and `triple-store-impl-rdf4j` (runtime);
- the runtime logging dependencies `logback-classic` and `log4j-over-slf4j`.

### Scopes: shipped vs. not shipped

Scope is used deliberately to keep the final `share/java` classpath clean while still letting the module participate in the aggregated build and coverage. Modules that must be present for build/coverage but **not** shipped are pulled in `provided` or `test` scope and so stay out of the packaged classpath — for example the technology-compatibility-kit and test artifacts (`iidm-tck`, `cgmes-model-test`, `psse-model-test`, `commons-test`, `config-test`, `scripting-test`, `tools-test`, `triple-store-test`), and the `cgmes-model-alternatives` artifact (test scope, with an inline comment noting it is there for coverage only). The `itools-packager` plugin itself is declared as a `provided` dependency in addition to being bound as a plugin.

## Invoking the packager

The `build` section binds `powsybl-itools-packager-maven-plugin`'s `package-zip` goal to the `package` phase, with:

- `packageName` = `powsybl`;
- `javaXmx` = `8G`;
- a `copyToEtc` adding the IIDM XSD schemas `iidm_V1_3.xsd`, `iidm_V1_2.xsd`, `iidm_V1_1.xsd` and `iidm_V1_0.xsd` (taken from `iidm/iidm-serde/src/main/resources/xsd`) to the package's `etc` directory.

Building this module therefore yields the runnable `powsybl` iTools distribution archive.

## Profiles

Two profiles complete the module:

- **`checks`** — active by default (via the `powsyblchecks` property), it runs the `maven-enforcer-plugin` `dependencyConvergence` rule in the `validate` phase, but with `fail=false`, so convergence problems are reported without breaking the build.
- **`jacoco`** — not active by default; it adds a `report-aggregate` execution of the `jacoco-maven-plugin` in the `verify` phase, producing a coverage report aggregated across the whole distribution. This aggregation is the reason several test-only artifacts are listed as `provided`/`test` dependencies above.
