# PowerFactory

The `powerfactory` module imports DIgSILENT PowerFactory study cases into IIDM. It is **import-only**: there is no `Exporter`. PowerFactory stores a study case as a tree of typed objects (`ElmNet`, `ElmTerm`, `ElmLne`, `ElmSym`, ...), and the module's design closely mirrors that object/attribute structure rather than forcing it into format-specific records.

The module is split into a generic object model, two interchangeable file backends, and the IIDM converter:

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `powerfactory-model` | `powsybl-powerfactory-model` | The generic data-object model and the `PowerFactoryDataLoader` SPI. |
| `powerfactory-dgs` | `powsybl-powerfactory-dgs` | Backend reading DGS text files (`.dgs`). |
| `powerfactory-db` | `powsybl-powerfactory-db` | Backend reading a live PowerFactory database through a native (JNI) bridge. |
| `powerfactory-converter` | `powsybl-powerfactory-converter` | The `PowerFactoryImporter` and the per-equipment converters mapping to IIDM. |

## powerfactory-model

This submodule defines a generic, schema-driven representation of a PowerFactory data model — independent of the file format it came from — plus the loader SPI that the backends implement.

### The data-object model

The model is a small object database:

- `com.powsybl.powerfactory.model.DataObject` is a node in the object tree. It carries a numeric `id`, a `parent` and `children`, a reference to its `DataClass`, the owning `DataObjectIndex`, and a map of attribute values. Helpers expose typed accessors (`findStringAttributeValue`, `getLocName`, ...), parent/child navigation (`getChildrenByClass`, `getChild`) and a `search(String regex)` that matches descendants by their qualified name. `getDataClassName()` returns the PowerFactory class name (e.g. `"ElmTerm"`), which the converter dispatches on.
- `com.powsybl.powerfactory.model.DataClass` is a class definition: a name plus an ordered list of `DataAttribute`s (with a by-name index).
- `com.powsybl.powerfactory.model.DataAttribute` is an attribute definition (name, `DataAttributeType`, description); it defines the well-known names `loc_name`, `fold_id`, `for_name`.
- `com.powsybl.powerfactory.model.DataAttributeType` is the value-type enum: `INTEGER`, `DOUBLE`, `DOUBLE_MATRIX`, `OBJECT`, `STRING`, `INTEGER64`, `FLOAT` and their vector variants.
- `com.powsybl.powerfactory.model.DataScheme` is the collection of `DataClass`es (the schema) for a model; it can be built from the objects or (de)serialized to JSON.
- `com.powsybl.powerfactory.model.DataObjectIndex` indexes all objects by id, by class and by foreign key (`for_name`), and lazily maintains backward links so that object references can be resolved both ways. `DataObjectRef`/`DataObjectRefKey` model an unresolved reference until the index resolves it.

`com.powsybl.powerfactory.model.PowerFactoryData` is the common interface for a loaded data set, implemented through `AbstractPowerFactoryData` by:

- `com.powsybl.powerfactory.model.StudyCase` — a study case: a name, a timestamp, and the list of `ElmNet` root networks (`getElmNets()`). This is what the importer consumes.
- `com.powsybl.powerfactory.model.Project` — a whole project tree, from which the active study case can be extracted (used by the DB backend).

Both can be read from and written to JSON (`JsonStudyCaseLoader`, `JsonProjectLoader`, and the `writeJson`/`parseJson` methods), which gives a portable, backend-independent serialization of the intermediate model.

### The `PowerFactoryDataLoader` SPI

`com.powsybl.powerfactory.model.PowerFactoryDataLoader<T extends PowerFactoryData>` is the extension point that decouples the importer from the physical file format. A loader declares:

```java
Class<T> getDataClass();          // e.g. StudyCase.class
String getExtension();            // e.g. "dgs"
boolean test(InputStream is);     // can this loader handle the content?
T doLoad(String fileName, InputStream is);
```

Loaders are discovered with `ServiceLoader` through the static `find(Class<T>)` helper, and the static `load(...)` helpers iterate the registered loaders, pick the first whose extension matches the file name and whose `test(...)` accepts the content, and call `doLoad`. New backends are added simply by registering a new `@AutoService(PowerFactoryDataLoader.class)` implementation — exactly the [plugin recipe](../index.md#the-plugin-mechanism-service-provider-interface) used elsewhere.

## The file backends

### powerfactory-dgs

`powerfactory-dgs` reads the DGS text export. `com.powsybl.powerfactory.dgs.DgsStudyCaseLoader` is the `@AutoService(PowerFactoryDataLoader.class)` loader for the `.dgs` extension (its `test` always returns true). It delegates to `com.powsybl.powerfactory.dgs.DgsReader`, which parses the file (default charset ISO-8859-1) building `DataClass`es and `DataObject`s into a `DataObjectIndex` and returning a `StudyCase`. The low-level parsing is split between `DgsParser` (token/record scanning) and `DgsHandler` (a SAX-like callback that materializes classes, objects and attribute values).

### powerfactory-db

`powerfactory-db` reads directly from a running PowerFactory installation through its C++ API. `com.powsybl.powerfactory.db.DbStudyCaseLoader` is the `@AutoService` loader, but its `getExtension()` is `"properties"`: the data source is a small `.properties` file (`ActiveProjectConfig`) naming the active project, not the data itself. `test(...)` succeeds only when the native bridge is available *and* the property file is valid.

The native access is hidden behind the `com.powsybl.powerfactory.db.DatabaseReader` interface (`isOk()`, `read(homeDir, projectName, builder)`). Its production implementation `JniDatabaseReader` loads a native library (`powsybl-powerfactory-db-native`) via JNI and is only functional on Windows (where PowerFactory runs); on any other platform `isOk()` returns false, so the loader silently declines. `DataObjectBuilder` is the callback the native side uses to reconstruct `DataObject`s, and `DbProjectLoader`/`PowerFactoryAppUtil` drive the project load. The DB backend therefore loads a `Project` and returns its active `StudyCase`.

This is the `powerfactory-db` API the task refers to as the "db-api": the `DatabaseReader` SPI plus the `DataObjectBuilder` callback, kept abstract so the JNI implementation can be substituted (for example in tests).

## powerfactory-converter

`com.powsybl.powerfactory.converter.PowerFactoryImporter` is the `@AutoService(Importer.class)` implementation (format `"POWER-FACTORY"`). Notably, `getSupportedExtensions()` is derived dynamically from the registered loaders, and `exists`/`importData` locate the right `PowerFactoryDataLoader` for the data source, load a `StudyCase`, and build the network.

The conversion walks the PowerFactory object tree:

1. Gather the `ElmTerm` terminals across all `ElmNet` networks; build IIDM containers (substations / voltage levels) with `ContainersMapping` via `ContainersMappingHelper`.
2. Build the topology, deciding HVDC handling first: a `ReducedHvdcConverter` (HVDC as simple lines) or, with the `powerfactory.import.dgs.HVDC-import-detailed` parameter, a `DetailedHvdcConverter` (full multi-terminal DC subgrids). DC nodes/links are then excluded from AC processing.
3. Create nodes from terminals (`NodeConverter`), optionally forcing all `ElmTerm`s to busbars (`powerfactory.import.dgs.force-all-elmTerms-as-busbars`).
4. Convert each AC object by dispatching on its `getDataClassName()` in `processEquipment`: `ElmSym`/`ElmAsm`/`ElmGenstat` → `GeneratorConverter`, `ElmXnet` → `ExternalGridConverter`, `ElmLod`/`ElmLodmv` → `LoadConverter`, `ElmShnt` → `ShuntConverter`, `ElmLne`/`ElmTow` → `LineConverter`, `ElmTr2`/`ElmTr3` → `TransformerConverter`, `ElmCoup` → `SwitchConverter`, `ElmZpu` → `CommonImpedanceConverter`. A large list of relay/measurement/graphics classes is explicitly ignored, and unknown classes are logged.
5. Build the HVDC subgrids, attach slack buses (where an `ElmGenstat`/`ElmSym`/`ElmAsm`/`ElmXnet` is flagged slack) via the `SlackTerminal` extension, and finally set bus voltages and angles (`VoltageAndAngle`).

The per-equipment converters share `com.powsybl.powerfactory.converter.AbstractConverter` (and `AbstractHvdcConverter` for the DC side), and `DataAttributeNames` centralizes the PowerFactory attribute-name constants they read. The two boolean import parameters above are the only configuration.

## Design notes

The defining design choice is the `PowerFactoryDataLoader` SPI: the importer is written entirely against the generic `DataObject`/`DataClass` model, so adding a new way to obtain PowerFactory data (a new file format, a new database bridge) only requires a new loader on the classpath. The JSON loaders also make the intermediate model directly serializable, which is convenient for testing and for caching a parsed case.
