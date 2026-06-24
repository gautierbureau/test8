# CGMES

**CGMES** (Common Grid Model Exchange Specification) is the ENTSO-E standard for exchanging grid models, based on the IEC CIM (Common Information Model) expressed in RDF. It is by far the largest format module in PowSyBl, and the most complex: a CGMES case is a set of RDF/XML *profile* files (equipment, topology, steady-state hypothesis, state variables, boundaries...) rather than a single flat file, and the conversion to and from the [IIDM grid model](../../grid_model/index.md) involves a great deal of domain logic.

The user-facing description lives in the [CGMES](../../grid_exchange_formats/cgmes/index.md) section. This page covers the internal design. CGMES is built on top of the [triple store](triple-store.md): the RDF data is loaded into a `TripleStore` and queried with SPARQL, and the resulting query rows drive the conversion. CGMES also extends the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface) with two CGMES-specific extension points — import pre/post-processors and naming strategies.

## Submodule structure

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `cgmes-model` | `powsybl-cgmes-model` | The CGMES (CIM/RDF) model, backed by a triple store. |
| `cgmes-conversion` | `powsybl-cgmes-conversion` | Bidirectional conversion between the CGMES model and IIDM (`CgmesImport` / `CgmesExport`). |
| `cgmes-extensions` | `powsybl-cgmes-extensions` | IIDM extensions carrying CGMES information not represented natively in IIDM. |
| `cgmes-gl` | `powsybl-cgmes-gl` | Import/export of the GL (Geographical Location) profile. |
| `cgmes-measurements` | `powsybl-cgmes-measurements` | Import of measurement data through a post-processor. |
| `cgmes-shortcircuit` | `powsybl-cgmes-shortcircuit` | Import of short-circuit data through a post-processor. |
| `cgmes-completion` | `powsybl-cgmes-completion` | Completion of missing data in incomplete instance files (through a pre-processor). |
| `cgmes-conformity` | `powsybl-cgmes-conformity` | Test support based on ENTSO-E conformity assessment data sets. |
| `cgmes-model-test` | `powsybl-cgmes-model-test` | A reusable tester for `CgmesModel` implementations. |
| `cgmes-model-alternatives` | `powsybl-cgmes-model-alternatives` | Experiments around alternative model queries. |

## The CGMES model (`cgmes-model`)

`com.powsybl.cgmes.model.CgmesModel` is the central interface. It exposes the CIM content as a large set of typed query methods, each returning a `PropertyBags` (the triple-store result type): `substations()`, `voltageLevels()`, `terminals()`, `acLineSegments()`, `transformers()`, `transformerEnds()`, `ratioTapChangers()`, `phaseTapChangers()`, `regulatingControls()`, `energyConsumers()`, `shuntCompensators()`, `synchronousMachines()`, `svVoltages()` and many more. It also gives access to model-level metadata (`modelId()`, `fullModels()`, `isNodeBreaker()`, `hasBoundary()`, `hasEquipmentCore()`) and to the underlying store through `tripleStore()`.

The default implementation is `com.powsybl.cgmes.model.triplestore.CgmesModelTripleStore`, which extends `com.powsybl.cgmes.model.AbstractCgmesModel`. Each `CgmesModel` query method is a named SPARQL query run against the `TripleStore`; the queries are kept in a `QueryCatalog` resource, chosen by CIM version and by an init/update query-catalog suffix.

`CgmesModel` instances are created from a `DataSource` through the factory:

```java
CgmesModel cgmes = CgmesModelFactory.create(dataSource);
// overloads add: implementation name, ReportNode, TripleStoreOptions,
// and a separate data source for the boundary files
```

`com.powsybl.cgmes.model.CgmesModelFactory` determines the CIM namespace present in the data source (`obtainCimNamespace`), creates a `TripleStore` of the requested implementation (default `rdf4j`), loads the RDF/XML files into it and wraps it as a `CgmesModelTripleStore`.

Supporting types in `cgmes-model` include:

- `com.powsybl.cgmes.model.CgmesOnDataSource` — detects whether a data source contains CGMES (by inspecting RDF namespaces) and lists the CGMES file names. It backs `CgmesImport.exists`.
- `com.powsybl.cgmes.model.CgmesNamespace` — the known CIM/RDF namespaces (`CIM_16_NAMESPACE`, `CIM_100_NAMESPACE`, the ENTSO-E/EU/MD namespaces) and the standard profile URIs.
- `com.powsybl.cgmes.model.CgmesSubset` — the profile/subset enum (`EQUIPMENT` "EQ", `TOPOLOGY` "TP", `STATE_VARIABLES` "SV", `STEADY_STATE_HYPOTHESIS` "SSH", `DYNAMIC` "DY", `DIAGRAM_LAYOUT` "DL", `GEOGRAPHICAL_LOCATION` "GL", plus the `EQ_BD` / `TP_BD` boundary subsets).
- `CgmesNames`, `CgmesTerminal`, `CgmesContainer`, `PowerFlow`, `FullModel`, `CgmesMetadataModel` — vocabulary and small value types.
- `com.powsybl.cgmes.model.InMemoryCgmesModel` / `EmptyTripleStore` — lightweight implementations used in tests.

## Conversion to and from IIDM (`cgmes-conversion`)

### Import: `CgmesImport` and `Conversion`

`com.powsybl.cgmes.conversion.CgmesImport` is the `Importer` (`@AutoService(Importer.class)`): `getFormat()` → `"CGMES"`, `getComment()` → `"ENTSO-E CGMES version 2.4.15"`, supported extension `xml`. `importData(...)` builds a `CgmesModel` through `CgmesModelFactory` and then delegates the actual mapping to `com.powsybl.cgmes.conversion.Conversion`.

`Conversion.convert(ReportNode)` walks the CGMES query results and builds the IIDM `Network` through the IIDM adders. It is driven by a `Conversion.Config` object and a per-conversion `com.powsybl.cgmes.conversion.Context`, and is organized into specialized helpers, for example:

- node/topology mapping: `NodeMapping`, `TerminalMapping`, `NodeContainerMapping`, `CgmesBoundary`;
- regulating controls: `RegulatingControlMapping` and its per-equipment variants (`...ForGenerators`, `...ForShuntCompensators`, `...ForStaticVarCompensators`, `...ForTransformers`, `...ForVscConverters`), plus `RegulatingTerminalMapper`;
- branches and transformers, with the many interpretation choices that CGMES allows captured by the `Config` enums `Xfmr2RatioPhaseInterpretationAlternative`, `Xfmr2ShuntInterpretationAlternative`, `Xfmr3...`, etc.

`CgmesImport` exposes a large set of `Parameter`s (all prefixed `iidm.import.cgmes.`), including `boundary-location`, `convert-boundary`, `import-control-areas`, `naming-strategy`, `source-for-iidm-id`, `powsybl-triplestore`, `cgm-with-subnetworks`, `store-cgmes-model-as-network-extension`, `store-cgmes-conversion-context-as-network-extension`, and the `pre-processors` / `post-processors` selectors described below. `Conversion.update(Network, ReportNode)` supports re-applying SSH/SV updates to an already imported network.

### Export: `CgmesExport`

`com.powsybl.cgmes.conversion.CgmesExport` is the `Exporter` (`@AutoService(Exporter.class)`, `getFormat()` → `"CGMES"`). `export(...)` serializes an IIDM network back into the CGMES profile files. The export of each profile lives in `com.powsybl.cgmes.conversion.export`, with one entry class per subset: `EquipmentExport`, `SteadyStateHypothesisExport`, `TopologyExport` and `StateVariablesExport`, coordinated by a `CgmesExportContext` (helped by `CgmesExportUtil` and `ReferenceDataProvider`).

Its `Parameter`s (prefixed `iidm.export.cgmes.`) control the output: `profiles` (which subsets to write), `cim-version`, `topology-kind`, `naming-strategy`, `base-name`, `modeling-authority-set`, `export-boundary-power-flows`, `cgm_export` (common-grid-model export), the boundary EQ/TP identifiers, and others.

### Naming strategy (conversion SPI)

IIDM and CGMES identifiers do not coincide, and CGMES requires master resource identifiers (mRIDs) that are valid UUIDs. The mapping between the two id spaces is pluggable through `com.powsybl.cgmes.conversion.naming.NamingStrategy`, which converts in both directions (`getIidmId`, `getIidmName`, and the several `getCgmesId(...)` overloads taking an `Identifiable`, a raw identifier, or `CgmesObjectReference`s). `com.powsybl.cgmes.conversion.naming.NamingStrategyFactory.create(impl, uuidNamespace)` builds one from a name, the built-in choices being `IDENTITY` (`IdentityNamingStrategy`, ids passed through unchanged) and `CGMES` (`CgmesNamingStrategy`, which generates stable UUID-based mRIDs). Implementations are discovered through `NamingStrategyProvider` / `NamingStrategiesServiceLoader` and selected by the import/export `naming-strategy` parameter.

## Import pre- and post-processors

CGMES adds two CGMES-specific extension points on top of the generic `Importer`, both discovered by `ServiceLoader` and selected at import time by the `pre-processors` / `post-processors` parameters of `CgmesImport`:

- `com.powsybl.cgmes.conversion.CgmesImportPreProcessor` — `getName()` and `process(CgmesModel cgmes)`, run **before** the conversion, to enrich or repair the CGMES model.
- `com.powsybl.cgmes.conversion.CgmesImportPostProcessor` — `getName()` and `process(Network network, CgmesModel cgmesModel)`, run **after** the IIDM network has been built, to attach extra data.

`cgmes-conversion` itself ships a couple of post-processors (`EntsoeCategoryPostProcessor`, `RemoveGroundsPostProcessor`, both `@AutoService(CgmesImportPostProcessor.class)`); the optional submodules below contribute the others. (This is the generic `CgmesImportPostProcessor` SPI listed in the [developer guide](../index.md#the-plugin-mechanism-service-provider-interface).)

## CGMES IIDM extensions (`cgmes-extensions`)

`cgmes-extensions` defines the IIDM extensions used to carry CGMES information that has no native IIDM representation, so that a network can be imported and re-exported without loss. The main extension interfaces are `CgmesMetadataModels` (the per-profile `FullModel` metadata), `CgmesTapChangers` / `CgmesTapChanger`, `BaseVoltageMapping`, `CimCharacteristics` (CIM version and topology kind, via `CgmesTopologyKind`), `CgmesLineBoundaryNode`, `CgmesBoundaryLineBoundaryNode` and `Source`. Each comes with its adder and serializer, registered through the [extension mechanism](../index.md#the-extension-mechanism).

The conversion also stores two extensions on the imported network (when the corresponding parameters are enabled), defined in `cgmes-conversion`: `com.powsybl.cgmes.conversion.CgmesModelExtension` (gives access to the underlying `CgmesModel`) and `com.powsybl.cgmes.conversion.CgmesConversionContextExtension` (the conversion `Context`), both `Extension<Network>`. They are what allows a later `CgmesExport` to reproduce the source model faithfully.

## Optional profile and data submodules

These submodules plug into the import/export through the SPIs above:

- **`cgmes-gl`** handles the Geographical Location profile. `com.powsybl.cgmes.gl.CgmesGLImporter.importGLData()` reads substation and line positions into the IIDM geographical-position extensions, and `CgmesGLExporter.exportData(DataSource)` writes them back; `CgmesGLImportPostProcessor` (`@AutoService(CgmesImportPostProcessor.class)`, name `cgmesGLImport`) wires the import into the CGMES post-processing chain.
- **`cgmes-measurements`** provides `com.powsybl.cgmes.measurements.CgmesMeasurementsPostProcessor` (`@AutoService`, name `measurements`), which adds Analog/Discrete measurement data to the imported network.
- **`cgmes-shortcircuit`** provides `com.powsybl.cgmes.shortcircuit.CgmesShortCircuitPostProcessor` (`@AutoService`, name `shortcircuit`), importing short-circuit parameters (generators, busbar sections) into the relevant IIDM extensions.
- **`cgmes-completion`** provides `com.powsybl.cgmes.completion.CreateMissingContainersPreProcessor` (`@AutoService(CgmesImportPreProcessor.class)`), which completes instance files lacking mandatory containers (substations/voltage levels) before the conversion runs.

## Test support

- **`cgmes-conformity`** packages the ENTSO-E conformity assessment data sets as reusable catalogs (`CgmesConformity1Catalog`, `CgmesConformity2Catalog`, `CgmesConformity3Catalog`, the `Cgmes3Catalog`, their `*ModifiedCatalog` variants...). Each entry exposes a `com.powsybl.cgmes.model.GridModelReference` — an interface (`name()`, `dataSource()`) implemented by `GridModelReferenceResources` — so tests across the codebase can import a standard CGMES case from a known data source.
- **`cgmes-model-test`** provides `com.powsybl.cgmes.model.test.CgmesModelTester`, constructed from a `GridModelReference`, whose `test()` validates a `CgmesModel` implementation by loading a reference model and exercising its queries.
- **`cgmes-model-alternatives`** holds experiments around alternative model query strategies.
