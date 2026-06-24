# PSS/E

The `psse` module reads and writes Siemens **PSS/E** power-flow cases, in both the legacy text `.raw` format and the newer JSON-based `.rawx` format. It converts them to and from the [IIDM grid model](../../grid_model/index.md). The user-facing description lives in the [PSS/E](../../grid_exchange_formats/psse/index.md) section; this page covers the internal design.

The module follows the [grid exchange formats](../grid-exchange-formats.md) layer conventions: a `*-model` submodule isolates the PSS/E data model and its file readers/writers, and a `*-converter` submodule maps it to IIDM through the `Importer` / `Exporter` [SPIs](../index.md#the-plugin-mechanism-service-provider-interface).

## Submodule structure

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `psse-model` | `powsybl-psse-model` | The PSS/E power-flow model and the RAW/RAWX read/write framework. |
| `psse-converter` | `powsybl-psse-converter` | The `PsseImporter` / `PsseExporter` and the IIDM mapping. |
| `psse-model-test` | `powsybl-psse-model-test` | Test cases and shared test resources. |

## The power-flow model (`psse-model`)

`com.powsybl.psse.model.pf.PssePowerFlowModel` is the in-memory representation of a PSS/E case. It aggregates a `PsseCaseIdentification` (base MVA, frequency, revision, title) and lists of the data records, each modelled by its own class in `com.powsybl.psse.model.pf`:

| Record class | Content |
|--------------|---------|
| `PsseBus` | Buses (number, name, base kV, area, zone, owner, voltage). |
| `PsseLoad`, `PsseFixedShunt`, `PsseSwitchedShunt` | Loads and shunt compensation. |
| `PsseGenerator` | Generators with reactive/active limits. |
| `PsseNonTransformerBranch` | AC lines (R, X, B, ratings). |
| `PsseTransformer` | Two- and three-winding transformers. |
| `PsseArea`, `PsseZone`, `PsseOwner` | Area, zone and owner definitions. |
| `PsseTwoTerminalDcTransmissionLine`, `PsseVoltageSourceConverterDcTransmissionLine`, `PsseMultiTerminalDcTransmissionLine` | HVDC links (LCC two-terminal, VSC, multi-terminal). |
| `PsseFacts`, `PsseInductionMachine`, `PsseGneDevice` | FACTS devices, induction machines, general network elements. |
| `PsseSubstation` | Node-breaker substation topology. |
| `PsseTransformerImpedanceCorrection`, `PsseLineGrouping`, `PsseInterareaTransfer` | Impedance correction tables, multi-section lines, inter-area transfers. |

Field-level access uses Jackson and univocity annotations on the record fields, so the same record classes serve both the legacy fixed/CSV text format and the JSON format.

### Versions

`com.powsybl.psse.model.PsseVersion` represents the format revision and supports major versions 32, 33 and 35 (the `Major` enum `V32`/`V33`/`V35`). `PsseVersion.fromRevision(float)` derives the version from the case revision number, and `isSupported()` / `supportedVersions()` report the supported set. Many record classes declare **version-specific field-name arrays** (v32, v33, v35, and the RAWX/JSON variant), selected at parse/write time from the current version — this is how a single record class supports several revisions of the format.

### Validation

`com.powsybl.psse.model.pf.PsseValidation` (constructed from a `PssePowerFlowModel` and a `PsseVersion`) checks the model before it is converted: positive base MVA and frequency, unique and in-range bus numbers, referenced buses existing for loads, shunts, generators, branches and transformers, consistent reactive/active limits, valid regulating buses, non-zero branch reactance, and so on. Several checks are version-aware (e.g. the regulating-bus field differs between v35 and earlier). It exposes `isValidCase()` and `getValidationErrors()`.

### RAW vs RAWX: the I/O framework

The reading/writing framework lives in `com.powsybl.psse.model.io` and is shared by all record groups:

- `com.powsybl.psse.model.io.FileFormat` distinguishes `LEGACY_TEXT` (the `.raw` fixed/CSV format, with single-quote string delimiters and comma separators) from `JSON` (the `.rawx` format).
- `com.powsybl.psse.model.io.Context` carries the format, the detected `PsseVersion`, the per-record-group field names and the Jackson reader/writer handles. It auto-detects the CSV delimiter for legacy text.
- `com.powsybl.psse.model.io.AbstractRecordGroup<T>` is the generic handler for one record group (buses, loads, ...). It parses CSV records into objects and serializes them back, picking the version-specific field names. The actual read/write strategy is selected by the `RecordGroupIO<T>` interface, with `RecordGroupIOLegacyText` for `.raw` and `RecordGroupIOJson` for `.rawx` (the JSON reader converts the `"fields"` / `"data"` JSON arrays into CSV-like records so the same parsing code is reused).
- `com.powsybl.psse.model.io.LegacyTextReader` wraps the input reader, normalizes CSV lines and detects end-of-block markers (`0` / `Q`).
- `com.powsybl.psse.model.io.RecordGroupIdentification` names each group both as a JSON node (`"bus"`, `"load"`...) and as a legacy section header (`BUS DATA`...), and tags it as a single parameter set or a data table.

The power-flow-specific assembly is in `com.powsybl.psse.model.pf.io`. The `PowerFlowData` interface (`isValidFile`, `readVersion`, `read`, `write`) has version-specific implementations — `PowerFlowRawData32` / `PowerFlowRawData33` / `PowerFlowRawData35` for legacy text and `PowerFlowRawxData35` for JSON. `com.powsybl.psse.model.pf.io.PowerFlowDataFactory` selects the right one:

```java
PowerFlowData data = PowerFlowDataFactory.create(extension);              // any supported version
PsseVersion version = data.readVersion(dataSource, ext);
PssePowerFlowModel model = PowerFlowDataFactory.create(ext, version)
                                               .read(dataSource, ext, context);
```

The `.raw` vs `.rawx` choice is driven by the **file extension**: `.raw` (case-insensitive) maps to `LEGACY_TEXT`, `.rawx` to `JSON`.

## Import and export (`psse-converter`)

### `PsseImporter`

`com.powsybl.psse.converter.PsseImporter` is annotated `@AutoService(Importer.class)`:

- `getFormat()` → `"PSS/E"`, `getComment()` → `"PSS/E Format to IIDM converter"`, supported extensions `raw`/`RAW`/`rawx`/`RAWX`;
- `exists(...)` iterates the extensions and validates the file (this is also how the RAW/RAWX format is detected);
- `importData(...)` reads the version, parses the `PssePowerFlowModel`, validates it with `PsseValidation`, and converts it to an IIDM `Network`. The imported model and its `Context` are attached to the network as extensions so that a later export can reproduce the original format.

Its `Parameter`s are `psse.import.ignore-base-voltage` and `psse.import.ignore-node-breaker-topology`.

### `PsseExporter`

`com.powsybl.psse.converter.PsseExporter` is annotated `@AutoService(Exporter.class)` (`getFormat()` → `"PSS/E"`). It supports two modes through its parameters:

- `psse.export.update` (default `true`) — when the network was imported from PSS/E, update the stored model in place (preserving its original layout) rather than rebuilding it from scratch;
- `psse.export.raw-format` (default `true`) — write legacy `.raw` text, or JSON `.rawx` when `false`.

It then writes through the appropriate `PowerFlowRawData*` / `PowerFlowRawxData35` implementation.

### Equipment converters

The IIDM mapping is split per equipment type. `com.powsybl.psse.converter.AbstractConverter` is the common base (holding the `Network` and the `ContainersMapping` that derives IIDM substations/voltage levels from PSS/E buses). Concrete converters include `BusConverter`, `LoadConverter`, `GeneratorConverter`, `LineConverter`, `TransformerConverter` (two- and three-winding), `FixedShuntCompensatorConverter`, `SwitchedShuntCompensatorConverter`, `TwoTerminalDcConverter`, `VscDcTransmissionLineConverter`, `FactsDeviceConverter`, `VoltageLevelConverter`, `SubstationConverter`, `TieLineConverter` and `BoundaryLineConverter`. On export, `com.powsybl.psse.converter.ContextExport` tracks the bus/node numbering and circuit identifiers (with distinct `UpdateExport` and `FullExport` states for the two export modes).
