# entsoe-util

`entsoe-util` (`powsybl-entsoe-util`) is a small library of helpers shared by the ENTSO-E-related formats — chiefly UCTE and CGMES. It is not an importer or exporter itself; it gathers cross-format ENTSO-E conventions (geographical codes, file-name parsing, the area extension, boundary points) in one place so that the format modules do not duplicate them. All classes live in the `com.powsybl.entsoe.util` package.

## EntsoeGeographicalCode

`com.powsybl.entsoe.util.EntsoeGeographicalCode` is an enum of the ENTSO-E geographical (country/control-area) codes used in UCTE-DEF, each mapped to an IIDM `Country`. Most entries map one-to-one (`FR` → `Country.FR`), but a few special cases exist: Germany has several control-area codes (`DE`, `D1`, `D2`, `D4`, `D7`, `D8`) all mapping to `Country.DE`; Kosovo (`KS`) maps to `Country.XK`; and the codes `UC`/`UX` have no associated country (mapped to `null`).

The reverse lookup `forCountry(Country)` returns all geographical codes for a country. It lazily builds a `Country` → codes `Multimap` on first use, guarded by a `ReentrantLock` for thread-safety.

## EntsoeArea extension

`com.powsybl.entsoe.util.EntsoeArea` is an IIDM **extension** on `Substation` (`Extension<Substation>`, name `"entsoeArea"`) that records the substation's `EntsoeGeographicalCode`. It is wired into the IIDM extension framework with the usual collaborators:

- `EntsoeAreaAdder` / `EntsoeAreaAdderImpl` — the adder used to attach the extension (`substation.newExtension(EntsoeAreaAdder.class).withCode(...).add()`);
- `EntsoeAreaImpl` — the implementation;
- `EntsoeAreaAdderImplProvider` — the `@AutoService(ExtensionAdderProvider.class)` that registers the adder (implementation name `"Default"`);
- `EntsoeAreaSerDe` — the `@AutoService(ExtensionSerDe.class)` XML serializer (element `entsoeArea`, schema `entsoeArea.xsd`), which writes/reads the geographical code as the node content.

Because both the adder provider and the serializer are `@AutoService`-registered, the extension is discovered automatically through the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface) — putting the jar on the classpath is enough for `EntsoeArea` to be addable and (de)serializable.

## EntsoeFileName

`com.powsybl.entsoe.util.EntsoeFileName` parses the standard ENTSO-E case file-name convention. `parse(String)` extracts, with a single regex:

- the **date/time** (`yyyy mm dd hh mn`, interpreted in the `Europe/Paris` zone), defaulting to "now" when the name does not match;
- the **forecast distance** (in minutes), derived from the `FO` (day-ahead forecast) or `SN` (snapshot) marker in the name;
- the **geographical code**, read from characters 19–20 of the name (a two-letter `EntsoeGeographicalCode`), or `null` when absent or unknown.

The parsed object exposes `getDate()`, `getForecastDistance()`, `getGeographicalCode()` and a convenience `getCountry()`. The constructor is `protected`, so the class can be subclassed by a format module that needs to attach additional metadata while reusing the parsing.

## Boundary points

`com.powsybl.entsoe.util.BoundaryPoint` is a simple value object describing a cross-border boundary point: a name and the two bordering `Country`s (`borderFrom`, `borderTo`). `com.powsybl.entsoe.util.BoundaryPointXlsParser` loads the official ENTSO-E boundary-point list from an Excel (HSSF) workbook through Apache POI, mapping the human-readable country names to IIDM `Country` values and producing a map of `BoundaryPoint`s keyed by name. This is used to resolve the countries on either side of a tie line / X-node.

## Design notes

`entsoe-util` is deliberately a thin, dependency-light helper module sitting below the ENTSO-E format converters. Its only framework integration point is the `EntsoeArea` IIDM extension (with its `@AutoService`-registered adder provider and serializer); everything else is plain utility code (`EntsoeGeographicalCode`, `EntsoeFileName`, `BoundaryPoint`) that the UCTE and CGMES modules call directly.
