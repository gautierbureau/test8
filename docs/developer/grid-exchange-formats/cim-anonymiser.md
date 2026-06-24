# cim-anonymiser

`cim-anonymiser` (`powsybl-cim-anonymiser`) is a small tooling module that anonymizes CIM/CGMES files. Real ENTSO-E cases carry identifiable information (object names, descriptions and the RDF identifiers that reference them); the anonymizer replaces these strings with neutral values so that a case can be shared — for example to reproduce a bug — without exposing sensitive data. It keeps a mapping dictionary so the substitution is consistent across files and reversible.

Unlike the other modules in this layer, it does **not** implement `Importer`/`Exporter` and does not go through the IIDM model: it streams the RDF/XML directly. It contributes an iTools `Tool` instead. The module has two classes, both in package `com.powsybl.cim`.

## CimAnonymizer — the engine

`com.powsybl.cim.CimAnonymizer` does the work. The public entry point is:

```java
void anonymizeZip(Path cimZipFile, Path anonymizedCimFileDir, Path dictionaryFile,
                  Logger logger, boolean skipExternalRef)
```

It loads (or creates) a mapping dictionary from `dictionaryFile`, opens the input ZIP archive, rewrites every entry into a new ZIP of the same name in the output directory, and finally saves the (possibly enriched) dictionary back. Each archive entry is processed as a stream of XML events.

The anonymization itself is a StAX event pipeline. `XmlAnonymizer` extends `EventWriterDelegate`: it intercepts the XML event stream and substitutes the sensitive parts before they are written out:

- the **content** of `IdentifiedObject.name` and `IdentifiedObject.description` elements (tracked with the `identifiedObjectName` / `identifiedObjectDescription` flags), except a small exclusion set (`PATL`, `TATL` operational-limit names are kept);
- the RDF identifier attributes `rdf:ID`, `rdf:resource` and `rdf:about` (recognized by their qualified names against the RDF namespace).

Replacement values come from a `com.powsybl.commons.util.StringAnonymizer` "dictionary": each original string is mapped to a stable anonymized token, so the same value always maps to the same replacement and cross-references stay consistent. The dictionary is persisted between runs (`loadDic`/`saveDic`), which is what makes the anonymization repeatable across a set of files and, with the saved mapping, reversible.

The `skipExternalRef` flag drives the handling of **external** references: when set, the anonymizer first collects the `rdf:ID` values actually defined in the archive (`getRdfIdValues`) and only anonymizes references that point inside it, leaving references to objects defined elsewhere (boundaries, other model parts) untouched. Strings that were skipped are reported to the `Logger`.

`CimAnonymizer.Logger` is a tiny callback interface (`logAnonymizingFile`, `logSkipped`) so that the caller controls how progress is reported; `DefaultLogger` is a no-op implementation.

## CimAnonymizerTool — the iTools command

`com.powsybl.cim.CimAnonymizerTool` exposes the engine as a command-line tool. It is an `@AutoService(Tool.class)` implementation, so it is discovered through the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface) and appears in an iTools distribution automatically.

The command is named `cim-anonymizer` (theme "Data conversion") and accepts:

| Option | Required | Meaning |
|--------|----------|---------|
| `--cim-path` | yes | CIM ZIP file, or a directory of `.zip` files. |
| `--output-dir` | yes | Directory where anonymized ZIP files are written. |
| `--mapping-file` | yes | File storing the id mapping dictionary. |
| `--skip-external-refs` | no | Do not anonymize external references. |

`run(...)` parses the options through `ToolOptions`, builds a `CimAnonymizer` with a `Logger` that prints to the tool's output stream, and calls `anonymizeZip`. When `--cim-path` is a directory it iterates over every `.zip` file it contains; otherwise it processes the single file.

## Design notes

`cim-anonymiser` is the layer's clearest example of a `Tool`-only contribution: it has no `Importer`/`Exporter` and never builds an IIDM `Network`. The whole transformation is a streaming StAX rewrite driven by a persistent `StringAnonymizer` dictionary, which keeps it memory-light on large cases and guarantees consistent, reversible substitutions. The only framework integration point is the `@AutoService(Tool.class)` registration of `CimAnonymizerTool`.
