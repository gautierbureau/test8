# Triple store

The `triple-store` module is an abstraction over [RDF](https://www.w3.org/RDF/) triple stores. It lets the rest of PowSyBl load CIM/RDF data into a queryable graph and run [SPARQL](https://www.w3.org/TR/sparql11-query/) queries against it, without depending on a particular RDF engine. It is the storage and query layer on which [CGMES](cgmes.md) is built: a `CgmesModel` is essentially a set of SPARQL queries run over a `TripleStore`.

The module sits in the [grid exchange formats](../grid-exchange-formats.md) layer and, like the rest of the framework, relies on the [plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface) to discover its implementations at runtime.

## Submodule structure

| Submodule | ArtifactId | Role |
|-----------|------------|------|
| `triple-store-api` | `powsybl-triple-store-api` | The `TripleStore` API, its factory and the value types (`PropertyBag`, `QueryCatalog`...). |
| `triple-store-impl-rdf4j` | `powsybl-triple-store-impl-rdf4j` | The default implementation, built on [Eclipse RDF4J](https://rdf4j.org/). |
| `triple-store-test` | `powsybl-triple-store-test` | A shared test harness run against every implementation. |

The API is deliberately decoupled from any implementation: a consumer depends only on `triple-store-api`, while the concrete engine (RDF4J today) is pulled in at runtime through the classpath.

## The `TripleStore` API

`com.powsybl.triplestore.api.TripleStore` is the central interface. It models a set of *named graphs* (called *contexts*, typically one per CIM/RDF instance file) and exposes operations to load, query, update and serialize them. The main methods are:

| Concern | Methods |
|---------|---------|
| Loading RDF | `read(InputStream is, String base, String contextName)` |
| Querying | `PropertyBags query(String query)`, `defineQueryPrefix(String prefix, String namespace)` |
| Updating | `update(String queryText)`, `add(String contextName, String namespace, String type, PropertyBag/PropertyBags)`, `add(TripleStore source)` |
| Namespaces | `addNamespace(String prefix, String namespace)`, `getNamespaces()` |
| Contexts | `contextNames()`, `clear(String contextName)` |
| Serializing | `write(DataSource ds)`, `write(DataSource ds, String contextName)`, `print(PrintStream out)` / `print(Consumer<String> liner)` |
| Misc | `getOptions()`, `getImplementationName()` |

`read` writes RDF/XML or Turtle statements into the named context; `write` serializes each context back to a separate file inside a `DataSource` (the same `com.powsybl.commons.datasource` abstraction used by importers and exporters). Queries return a `PropertyBags` result set.

`com.powsybl.triplestore.api.AbstractPowsyblTripleStore` is the abstract base shared by implementations. It holds the `TripleStoreOptions`, caches the query prefixes (prepended to every query through `adjustedQuery`), and provides helpers for generating RDF identifiers and resolving output streams.

### Query results: `PropertyBag` and `PropertyBags`

SPARQL results are returned as plain Java collections rather than RDF objects, which keeps the consuming code (the CGMES conversion) free of any RDF dependency.

- `com.powsybl.triplestore.api.PropertyBag` extends `HashMap<String, String>`: one bag is one result row, keyed by the SELECT variable names. Beyond the map it adds typed accessors (`asDouble`, `asInt`, `asBoolean`, `asOptionalDouble`), identifier/local-name helpers (`getId`, `getLocal`, `getLocals`) that strip RDF namespace prefixes and leading underscores, flags for resource/class/multivalued properties, and `tabulate`/`tabulateLocals` for debugging output.
- `com.powsybl.triplestore.api.PropertyBags` extends `ArrayList<PropertyBag>`: the full result set, with column-extraction helpers (`pluck`, `pluckLocals`, `pluckIdentifiers`) and `pivot` to transpose rows.

### Named SPARQL queries: `QueryCatalog`

Rather than embedding SPARQL strings in Java, queries are externalized in resource files and loaded through `com.powsybl.triplestore.api.QueryCatalog` (a `HashMap<String, String>` keyed by query name). A catalog file marks each query with a `# query: <name>` header and can compose other catalogs with `# include:`. `com.powsybl.triplestore.api.TripleStoreUtils.queryTripleStore(queryKey, queryCatalog, tripleStore)` is the convenience entry point used to run a named query. This is how `cgmes-model` keeps its (large) set of CIM queries.

### Configuration: `TripleStoreOptions`

`com.powsybl.triplestore.api.TripleStoreOptions` carries the behavioural knobs used when extracting identifiers from RDF, notably `removeInitialUnderscoreForIdentifiers` (strip the leading `_` of RDF IDs, default `true`) and `unescapeIdentifiers` (URL-decode escaped identifiers, default `true`), plus an optional `queryCatalog` path. They are fluent setters.

Errors are reported through `com.powsybl.triplestore.api.TripleStoreException`, which extends `PowsyblException`. Namespace bindings are carried by `com.powsybl.triplestore.api.PrefixNamespace`.

## Factory and the plugin mechanism

Implementations are not referenced directly; they are created through `com.powsybl.triplestore.api.TripleStoreFactory`:

```java
TripleStore ts = TripleStoreFactory.create();                 // default implementation ("rdf4j")
TripleStore ts = TripleStoreFactory.create("rdf4j", options); // named implementation, with options
TripleStore copy = TripleStoreFactory.copy(ts);               // copy using the source implementation
```

The factory discovers the available engines through a `ServiceLoaderCache<TripleStoreFactoryService>`. `com.powsybl.triplestore.api.TripleStoreFactoryService` is the Service Provider Interface each implementation must provide: `create()`, `create(TripleStoreOptions)`, `copy(TripleStore)`, `getImplementationName()` and `isWorkingWithNestedGraphClauses()` (a capability flag, since SPARQL nested `GRAPH` clauses are not handled identically by every engine). The default implementation name is `"rdf4j"`.

The factory also exposes introspection helpers used by the tests: `allImplementations()`, `onlyDefaultImplementation()`, `defaultImplementation()`, and the `implementationsWorkingWithNestedGraphClauses()` / `implementationsBadNestedGraphClauses()` partitions based on that capability flag.

## The RDF4J implementation

`triple-store-impl-rdf4j` is the only implementation shipped today. Its key classes are:

- `com.powsybl.triplestore.impl.rdf4j.TripleStoreRDF4J` extends `AbstractPowsyblTripleStore`. It is backed by an in-memory RDF4J repository (`SailRepository` over a `MemoryStore`) and implements the API on top of RDF4J's `RepositoryConnection`, `TupleQuery`/`TupleQueryResult` (for SELECT queries) and parser/writer settings. Its implementation name constant is `NAME = "rdf4j"`.
- `com.powsybl.triplestore.impl.rdf4j.TripleStoreFactoryServiceRDF4J` is the `TripleStoreFactoryService`, annotated `@AutoService(TripleStoreFactoryService.class)` so it is registered automatically. It reports `isWorkingWithNestedGraphClauses()` as `true`.
- `com.powsybl.triplestore.impl.rdf4j.PowsyblWriter` extends RDF4J's `RDFXMLWriter` to apply CGMES-specific RDF/XML serialization rules (choosing `rdf:ID` vs `rdf:about`, handling fragment references) so that the files written back are compatible with the CGMES profiles.

Because discovery is by `ServiceLoader`, adding an alternative engine (for example a Jena-based store) is just a matter of providing a new `triple-store-impl-*` module with its own `TripleStoreFactoryService` on the classpath — no change to the API or to CGMES.

## Test support

`triple-store-test` contains a shared harness (built around a `TripleStoreTester` helper) that loads the same RDF resources into every registered implementation, runs the same queries, updates and copies, and asserts identical results. Concrete tests cover SPARQL features that differ across engines (optionals, named graphs, subqueries with `LIMIT`, updates) as well as RDF/XML export, so that any new implementation can be validated against the same behavioural contract.
