# MATPOWER

The `matpower` module reads and writes [MATPOWER](../../grid_exchange_formats/matpower/index.md) case files. MATPOWER is a MATLAB-based power-system simulation package, and its cases are MATLAB `.mat` binary files (MAT-File level 5) containing a single structure named `mpc` (the "MATPOWER case"). The module supports both directions: an `Importer` and an `Exporter`.

It is split in two submodules, following the layer's usual `*-model` / `*-converter` split:

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `matpower-model` | `powsybl-matpower-model` | The intermediate model (`MatpowerModel` and its records) and the MAT-file reader/writer. |
| `matpower-converter` | `powsybl-matpower-converter` | The `MatpowerImporter` / `MatpowerExporter` and the mapping to and from IIDM. |

## matpower-model

This submodule is a faithful, table-oriented representation of an `mpc` structure, plus the binary MAT-file serialization. It depends only on `commons`, Jackson (for JSON test fixtures) and the MAT-File library `us.hebi.matlab:mat-file` rather than on IIDM, so it can be reused independently of the IIDM conversion.

### The model

`com.powsybl.matpower.model.MatpowerModel` is the in-memory case. It holds the case name, the `baseMVA` (base apparent power), the format `version`, and four lists mirroring the `mpc` matrices:

- `MBus` — a bus row (number, `MBus.Type`, real/reactive power demand, shunt conductance/susceptance, area, voltage magnitude/angle, base voltage, loss zone, min/max voltage magnitude, optional name).
- `MGen` — a generator row (bus number, real/reactive power output, Q limits, voltage setpoint, MVA base, status, P limits and, in v2, the capability-curve and ramp fields `pc1/pc2`, `qc1Min/Max`, `qc2Min/Max`, `rampAgc`, ...).
- `MBranch` — a branch row (from/to bus, `r`, `x`, `b`, ratings A/B/C, ratio, phase-shift angle, status, angle min/max).
- `MDcLine` — a DC line row (from/to, status, `pf/pt/qf/qt/vf/vt`, P/Q limits, `loss0/loss1`).

`MatpowerModel` keeps lookup indexes alongside the lists: `busByNum` (bus number → `MBus`) and `generatorsByBusNum` (bus number → its generators), so that the converter can resolve cross-references cheaply.

`MBus.Type` is the MATPOWER bus type enum: `PQ` (1), `PV` (2), `REF` (3, the slack) and `ISOLATED` (4), with `fromInt`/`getValue` to map to and from the integer encoding.

`com.powsybl.matpower.model.MatpowerFormatVersion` is an enum of the two supported case-format versions, `V1` and `V2`, each carrying the expected number of generator columns (10 for V1, 21 for V2). It is JSON-mapped via `@JsonCreator`/`@JsonValue` to the strings `"1"` and `"2"`.

### Reading and writing the .mat file

`com.powsybl.matpower.model.MatpowerReader` and `MatpowerWriter` are stateless utility classes (private constructor, static `read`/`write`). They wrap the third-party `us.hebi.matlab.mat` library (`Mat5`) to deal with the binary MAT-File format.

`MatpowerReader.read(InputStream, caseName)` opens the MAT file with an entry filter that keeps only the `mpc` struct (`MATPOWER_STRUCT_NAME = "mpc"`), checks that the mandatory fields (`version`, `baseMVA`, `bus`, `gen`, `branch`) are present, validates the version (only `V2` is supported on read, `MATPOWER_SUPPORTED_VERSION`), and then reads each matrix column-by-column into the model records. The optional `bus_name` cell and `dcline` matrix are read when present. The reader is tolerant about column counts: `checkNumberOfColumns` enforces minimums (`MATPOWER_BUSES_COLUMNS = 13`, `MATPOWER_BRANCHES_COLUMNS = 13`, `MATPOWER_DCLINES_COLUMNS = 17`) and downgrades the generator reading to the v1 layout (10 columns) when fewer than 21 generator columns are present, logging a warning.

`MatpowerWriter.write(MatpowerModel, OutputStream, withBusNames)` does the reverse: it fills the `Mat5` `Struct` (`version`, `baseMVA`, `bus`, `gen`, `branch`, and optionally `dcline` and `bus_name`), allocates a native-order `ByteBuffer` sized to the serialized length, and writes it to a `WritableByteChannel`. Generators are written with the column count of the model's `version`.

`com.powsybl.matpower.model.MatpowerModelFactory` is a test/utility helper that deserializes bundled JSON fixtures (`ieee9.json`, `ieee14.json`, `ieee30.json`, ... `ieee300.json`, `t_case9_dcline.json`) into `MatpowerModel` instances through Jackson. This is also why `MatpowerModel` carries Jackson annotations.

## matpower-converter

This submodule maps the intermediate model to and from IIDM. `com.powsybl.matpower.converter.MatpowerConstants` defines the shared `FORMAT = "MATPOWER"` and the file extension `EXT = "mat"`.

### MatpowerImporter

`com.powsybl.matpower.converter.MatpowerImporter` is the `@AutoService(Importer.class)` implementation. `exists(...)` checks for a `.mat` data-source entry, `getFormat()` returns `"MATPOWER"`, and `importData(...)` parses the file with `MatpowerReader` and builds a `Network`.

The conversion is essentially a per-bus walk:

- **Containers.** Voltage levels and substations are derived with `com.powsybl.iidm.network.util.ContainersMapping`, which groups buses connected by zero-impedance branches into a single bus-breaker voltage level and groups voltage levels joined by transformers into substations. Ids follow fixed prefixes (`BUS-`, `VL-`, `SUB-`, `LINE-`, `TWT-`, `GEN-`, `LOAD-`, `SHUNT-`, plus `CS1/CS2/HL` for HVDC).
- **Buses, loads, shunts, generators.** Each `MBus` yields a bus-breaker `Bus` (with `v`/`angle` set from the file); a `Load` when there is a non-zero demand; a `ShuntCompensator` when there is a non-zero susceptance; and one `Generator` per associated `MGen`. `REF` buses are collected and a `SlackTerminal` extension is attached to them at the end. Per-unit voltage values are scaled by the voltage level's nominal voltage; shunt admittances are converted to engineering units using `baseMVA`.
- **Branches.** `isLine`/`isTransformer` decide the IIDM type: a branch with no phase shift and either a zero ratio, or a unit ratio between equal base voltages, becomes a `Line`; otherwise a `TwoWindingsTransformer` (with a single-step `PhaseTapChanger` when a phase-shift angle is present). Per-unit `r/x/b` are converted to engineering units; `RateA/B/C` become an `ApparentPowerLimits` group (permanent + temporary limits) on both sides.
- **DC lines.** Each `MDcLine` becomes an `HvdcLine` between two `VscConverterStation`s, with loss factors back-computed from the MATPOWER `loss0`/`loss1` so that an export round-trip is preserved.

Behaviour is tuned by the boolean parameter `matpower.import.ignore-base-voltage` (default `true`): when set, bus base voltages are ignored and a nominal voltage of 1 is used, so all per-unit quantities stay in per-unit.

### MatpowerExporter

`com.powsybl.matpower.converter.MatpowerExporter` is the `@AutoService(Exporter.class)` implementation (`getFormat()` = `"MATPOWER"`). It always writes a `baseMVA` of 100 and a `V2` model. The export walks the IIDM network and builds a `MatpowerModel`, then hands it to `MatpowerWriter`.

The export is more involved than the import because IIDM is richer than the MATPOWER tables. The internal `Context` first computes the synchronous components to export and a slack bus per component (from `SlackTerminal` extensions when available); MATPOWER bus numbers can be preserved from the IIDM ids when possible (`preserveBusIds`). Generators, static var compensators (treated as generators) and dangling-line boundaries are collected and turned into `MGen` rows; buses that end up carrying a generator are promoted from `PQ` to `PV` (and the slack to `REF`). Generators whose limits exceed the configured caps are converted to loads. Export is configured by three parameters:

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `matpower.export.with-bus-names` | boolean | `false` | Write the optional `bus_name` cell. |
| `matpower.export.max-generator-active-power-limit` | double | `10000` | Cap above which a generator is exported as a load. |
| `matpower.export.max-generator-reactive-power-limit` | double | `10000` | Same for reactive power. |

## Design notes and extension points

The module follows the standard `Importer`/`Exporter` SPI recipe (see the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface)). Both implementations are discovered through `ServiceLoader` via `@AutoService`. There are no module-specific extension points beyond the standard parameters; the main reusable asset is `matpower-model`, which can read and write `.mat` MATPOWER cases without any IIDM dependency.
