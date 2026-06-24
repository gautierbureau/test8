# IEEE-CDF

The `ieee-cdf` module imports the [IEEE Common Data Format](../../grid_exchange_formats/ieee/ieee.md) (CDF), the historical text format used to distribute the IEEE test systems (the 14-, 30-, 57-, 118-, 300-bus cases, ...). The format is a fixed-width, section-based text file: a title line, then sections introduced by header lines such as `BUS DATA FOLLOWS` / `BRANCH DATA FOLLOWS` and terminated by a `-999` footer line. This module is **import-only** — there is no `Exporter` exposed.

It is split following the layer's usual `*-model` / `*-converter` split:

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `ieee-cdf-model` | `powsybl-ieee-cdf-model` | The intermediate model and the fixed-width text reader (and a matching writer). |
| `ieee-cdf-converter` | `powsybl-ieee-cdf-converter` | The `IeeeCdfImporter` and the mapping to IIDM. |

## ieee-cdf-model

This submodule is a plain-Java representation of a CDF file plus the column-based parser. It has no IIDM dependency.

### The model

`com.powsybl.ieeecdf.model.IeeeCdfModel` is the parsed case. It holds a mandatory `IeeeCdfTitle` and five lists, one per section:

- `IeeeCdfBus` — a bus record (number, name, area, loss zone, `Type`, final voltage/angle, load and generation MW/MVAR, base kV, desired volts, MVAR/voltage limits, shunt G/B, remote-controlled bus). The bus `Type` enum encodes the CDF type code: `UNREGULATED` (0, PQ), `HOLD_MVAR_GENERATION_WITHIN_VOLTAGE_LIMITS` (1, PQ), `HOLD_VOLTAGE_WITHIN_VAR_LIMITS` (2, PV) and `HOLD_VOLTAGE_AND_ANGLE` (3, the swing bus).
- `IeeeCdfBranch` — a branch record (tap-bus / z-bus numbers, R/X/B, ratings, branch `Type` and side, transformer ratio and angle, control parameters).
- `IeeeCdfLossZone`, `IeeeCdfInterchangeData`, `IeeeCdfTieLine` — the remaining (less load-bearing) sections.

The element classes all extend `com.powsybl.ieeecdf.model.elements.AbstractIeeeElement`, and the column layout of each record is documented in the class Javadoc (with reference to the canonical `cdf.txt` specification).

### Fixed-width parsing

The reading entry point is `com.powsybl.ieeecdf.model.IeeeCdfReader.read(BufferedReader)`. It first parses the title with `IeeeCdfTitleReader`, then loops over the lines, recognizing each section's header (e.g. `BUS DATA FOLLOWS    14 ITEMS`), extracting the announced item count with a regex, and delegating the body to the section reader (`IeeeCdfBusReader`, `IeeeCdfBranchReader`, `IeeeCdfLossZoneReader`, `IeeeCdfInterchangeDataReader`, `IeeeCdfTieLineReader`).

Every section reader extends `com.powsybl.ieeecdf.model.reader.AbstractIeeeCdfReader`, which provides the fixed-width machinery shared across sections:

- `readLines(reader, footerValue, constructor, expectedItemsNumber)` reads body lines until the footer (e.g. `-999`) is reached, applies a per-line constructor, and warns if the count does not match the announced number.
- `readInteger` / `readDouble` / `readString` extract a value from a 1-based `[start, end]` column range, trimming and skipping empty fields, and feed it to a setter through an `IntConsumer` / `DoubleConsumer` / `Consumer`.

A section reader is therefore a thin declaration of column ranges. For example `IeeeCdfBusReader.parseBus` reads the bus number from columns 1–4, the name from 6–17, the type from 25–26 (through `BusTypeConversion`), and so on. The `com.powsybl.ieeecdf.model.conversion` package holds the small value converters used while parsing/writing: `BusTypeConversion`, `BranchTypeConversion`, `BranchSideConversion`, `SeasonConversion` and `LocalDateConversion`.

A symmetric writer side exists (`com.powsybl.ieeecdf.model.IeeeCdfWriter` and the `*Writer` classes under `writer/`, extending `AbstractIeeeCdfWriter`) producing the fixed-width output; it is used internally and for tests, but is not exposed as an IIDM `Exporter`.

## ieee-cdf-converter

`com.powsybl.ieeecdf.converter.IeeeCdfImporter` is the `@AutoService(Importer.class)` implementation (format `"IEEE-CDF"`, extension `txt`). `exists(...)` does a lightweight signature check on the title line (the `mm/dd/yy` slashes at columns 4 and 7, and an `S`/`W` season marker around column 44). `importData(...)` parses the file with `IeeeCdfReader` and calls `convert(...)`.

The conversion mirrors the other power-flow importers in the layer:

- `ContainersMapping` (`com.powsybl.iidm.network.util`) groups buses and branches into IIDM voltage levels (`VL...`) and substations (`S...`), treating zero-impedance branches as bus merges and using `isTransformer` (a branch with a non-unit ratio or a phase angle) to decide branch grouping.
- A `PerUnitContext` carries the MVA base (from the title) and the `ignore-base-voltage` flag, so per-unit quantities are converted to engineering units. The nominal voltage of a bus comes from its base kV, or 1 when the base voltage is absent or ignored.
- `createBuses` creates the bus-breaker buses, loads, shunts and generators; the swing bus (`HOLD_VOLTAGE_AND_ANGLE`) gets a `SlackTerminal` extension. `createBranches` creates `Line`s and `TwoWindingsTransformer`s, with phase-shifting / voltage-control / active-power-control tap changers depending on the branch control type.
- The case date is taken from the title when present.

The single import parameter is `ignore-base-voltage` (boolean, default `false`). The constructor also accepts a custom `ToDoubleFunction<IeeeCdfBus>` nominal-voltage provider, overriding the default base-kV-or-1 rule; `IeeeCdfNetworkFactory` is a small helper that builds the standard IEEE test networks in memory for use in tests across the codebase.

## Design notes

The module is a clean illustration of the layer's `*-model` / `*-converter` separation: the fixed-width parsing in `ieee-cdf-model` knows nothing about IIDM and could be reused on its own, while `ieee-cdf-converter` only contributes the `Importer` SPI implementation and the IIDM mapping. There are no module-specific extension points beyond the standard import parameter.
