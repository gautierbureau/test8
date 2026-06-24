# UCTE

The `ucte` module handles the **UCTE-DEF** format (`.uct` files), the historical text exchange format of the former UCTE (now ENTSO-E continental Europe) network. It provides both an importer and an exporter to and from the [IIDM grid model](../../grid_model/index.md). The user-facing description of the format lives in the [UCTE](../../grid_exchange_formats/ucte/index.md) section; this page covers the internal design.

Like every format in the [grid exchange formats](../grid-exchange-formats.md) layer, it follows the `*-network` (intermediate model) / `*-converter` (IIDM mapping) split and plugs into the framework through the `Importer` / `Exporter` [SPIs](../index.md#the-plugin-mechanism-service-provider-interface).

## Submodule structure

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `ucte-network` | `powsybl-ucte-network` | The UCTE-DEF object model and its `.uct` reader/writer. |
| `ucte-util` | `powsybl-ucte-util` | Shared helpers (notably alias creation). |
| `ucte-converter` | `powsybl-ucte-converter` | The `UcteImporter` / `UcteExporter` and the IIDM mapping, including the naming strategy. |

## The UCTE-DEF model (`ucte-network`)

The format is record-oriented: a `.uct` file is a sequence of fixed-width blocks (comments, nodes, lines, transformers, regulations) flagged by a record type letter. The model in `com.powsybl.ucte.network` mirrors this structure.

`com.powsybl.ucte.network.UcteNetwork` is the root interface (default implementation `UcteNetworkImpl`). It aggregates the records and exposes a `fix(ReportNode)` method that validates and repairs inconsistent data. The main record classes are:

| Class | Role |
|-------|------|
| `UcteNode` | A node: voltage reference, active/reactive load and generation, generation limits, primary control, short-circuit data, power-plant type. |
| `UcteLine` | A line (or busbar coupler), extends `UcteElement`. |
| `UcteTransformer` | A two-winding transformer, extends `UcteElement`; adds rated voltages, nominal power and conductance. |
| `UcteRegulation` | The regulation attached to a transformer, combining a `UctePhaseRegulation` and a `UcteAngleRegulation`. |
| `UctePhaseRegulation` / `UcteAngleRegulation` | Tap-changer data: step, number of taps, current position, voltage/active-power target. |

`com.powsybl.ucte.network.UcteElement` is the abstract base of `UcteLine` and `UcteTransformer` (id, status, R/X/B, current limit, element name). These records implement the `UcteRecord` interface, whose contract is the single `fix(ReportNode)` method.

Identifiers and coded fields are first-class typed objects, which is where most of the format's domain logic lives:

- `com.powsybl.ucte.network.UcteNodeCode` — the structured node identifier (`UcteCountryCode`, a 5-character geographical spot, a `UcteVoltageLevelCode`, and an optional busbar character), with `parseUcteNodeCode(String)`.
- `com.powsybl.ucte.network.UcteElementId` — a branch identifier (two node codes plus an order character), with `parseUcteElementId(String)`.
- Enums encode the coded fields: `UcteCountryCode` (the one-character country codes, e.g. `FR`, `DE`, `XX`), `UcteVoltageLevelCode` (`VL_380`, `VL_220`...), `UcteElementStatus`, `UcteNodeStatus` (`REAL`/`EQUIVALENT`), `UcteNodeTypeCode` (`PQ`, `QT`, `PU`, `UT`), `UcteAngleRegulationType` (`ASYM`/`SYMM`) and `UctePowerPlantType`. The format version is carried by `UcteFormatVersion`.

### Reading and writing `.uct`

The I/O lives in `com.powsybl.ucte.network.io`:

- `UcteReader.read(BufferedReader, ReportNode)` parses a `.uct` file into a `UcteNetworkImpl`, block by block, and `checkHeader` is used to detect whether a stream is UCTE.
- `UcteWriter.write(BufferedWriter)` serializes a `UcteNetwork` back, grouping nodes by country.
- Because UCTE-DEF is a fixed-column format, the field-level parsing/formatting is factored into `UcteRecordParser` and `UcteRecordWriter` (parse/write a string, double, int, char or enum at given column positions), driven by the `UcteRecordType` enum (`C`, `N`, `L`, `T`, `R`...).

A separate `com.powsybl.ucte.network.ext` package (`UcteNetworkExt`, `UcteSubstation`, `UcteVoltageLevel`) derives the substation/voltage-level grouping that IIDM needs but UCTE-DEF does not represent explicitly.

## Import and export (`ucte-converter`)

### `UcteImporter`

`com.powsybl.ucte.converter.UcteImporter` is annotated `@AutoService(Importer.class)`. It declares:

- `getFormat()` → `"UCTE"`, `getComment()` → `"UCTE-DEF"`, `getSupportedExtensions()` → `uct`/`UCT`;
- `exists(ReadOnlyDataSource)` checks the `.uct` header;
- `importData(...)` reads the file into a `UcteNetworkExt` and builds the IIDM `Network` — creating buses, substations and voltage levels, then lines (handling couplers and boundary X-nodes), then transformers and their tap changers, merging paired boundary lines into IIDM tie lines, and optionally creating control areas.

Its `Parameter`s include `ucte.import.combine-phase-angle-regulation`, `ucte.import.create-areas` and `ucte.import.areas-dc-xnodes`.

### `UcteExporter`

`com.powsybl.ucte.converter.UcteExporter` is annotated `@AutoService(Exporter.class)` (`getFormat()` → `"UCTE"`). Its `export(...)` walks the IIDM network and builds a `UcteNetwork` (converting buses to `UcteNode`s, lines and transformers to UCTE elements, tap changers to phase/angle regulations), then writes it with `UcteWriter`. Its parameters include `ucte.export.naming-strategy` and `ucte.export.combine-phase-angle-regulation`; the boolean and the selected naming strategy are bundled into a `UcteExporterContext` passed through the conversion.

## Naming strategy (export SPI)

UCTE node and element identifiers are strongly constrained (country, geographical spot, voltage level, busbar / order code). IIDM identifiers are free-form, so on export they must be mapped to valid UCTE codes. This mapping is pluggable through `com.powsybl.ucte.converter.NamingStrategy`, whose methods produce a `UcteNodeCode` or a `UcteElementId` from an IIDM object or id.

- `com.powsybl.ucte.converter.AbstractNamingStrategy` is the cached base class.
- `com.powsybl.ucte.converter.DefaultNamingStrategy` (`@AutoService(NamingStrategy.class)`, name `"Default"`) assumes the IIDM ids are already valid UCTE codes and parses them directly.
- `CounterNamingStrategy` is an alternative that generates conforming codes with counters when the ids are not UCTE-compliant.

The implementations are discovered through a `ServiceLoaderCache<NamingStrategy>` and selected by the `naming-strategy` export parameter (matched on `getName()` by `UcteExporter`).

## Utilities (`ucte-util`)

`com.powsybl.ucte.util.UcteAliasesCreation.createAliases(Network)` enriches an IIDM network with aliases built from UCTE element names (for branches, tie lines, switches and boundary lines), so that equipment can be looked up by its UCTE identifier after import.
