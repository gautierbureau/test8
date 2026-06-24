# Math

The `math` module gathers the numerical and graph utilities that the rest of PowSyBl Core builds on without ever touching the grid model. Everything lives under `com.powsybl.math` and falls into three families: a generic undirected-graph data structure (reused by the IIDM topology code), a matrix abstraction with dense and sparse implementations backed by native solvers, and a thin wrapper around the KINSOL nonlinear solver. A handful of these classes bind to native libraries through JNI, which the module loads at class-load time.

This page is part of the [foundation layer](../foundation.md).

## Package structure

| Package | Content |
|---------|---------|
| `com.powsybl.math` | `AbstractMathNative` (native-library loading), `MathException` |
| `com.powsybl.math.graph` | the `UndirectedGraph` data structure, traversal callbacks and `GraphUtil` |
| `com.powsybl.math.matrix` | the `Matrix` abstraction, `DenseMatrix`/`SparseMatrix`, factories, LU decomposition, `ComplexMatrix` |
| `com.powsybl.math.matrix.serializer` | `SparseMatrixMatSerializer` (MATLAB `.mat` import/export) |
| `com.powsybl.math.solver` | the KINSOL binding (`Kinsol` and friends) |
| `com.powsybl.math.casting` | `Double2Float` |

### Native code

`AbstractMathNative` is the common base of every class that calls into native code (`SparseLUDecomposition`'s parent through `AbstractMatrix`, and `Kinsol`). Its static initializer loads the native library named `math` and runs a `nativeInit()` JNI call once. The native side provides the SuiteSparse KLU sparse LU factorization and the SUNDIALS KINSOL solver; the dense path stays pure Java (it uses the bundled Jama library). `MathException` is the unchecked base exception for the module, and the matrix and solver layers derive `MatrixException` and `KinsolException` from it.

## Graph utilities

Package `com.powsybl.math.graph`. `UndirectedGraph<V, E>` is a generic undirected graph that carries an arbitrary object of type `V` on each vertex and `E` on each edge. Vertices and edges are addressed by integer index, and those indices are **not guaranteed to be contiguous** — a removed vertex or edge leaves a hole that may later be reused — so callers iterate through `getVertices()` / `getEdges()` (both return `int[]`) rather than counting from zero. The interface is deliberately wide; the central operations are:

- vertex lifecycle: `addVertex()`, `addVertexIfNotPresent(int v)`, `vertexExists(int v)`, `removeVertex(int v)`, `removeIsolatedVertices()`, plus `getVertexObject(int v)` / `setVertexObject(int v, V obj)`;
- edge lifecycle: `addEdge(int v1, int v2, E obj)`, `removeEdge(int e)`, and the navigation helpers `getEdgeVertex1(int e)` / `getEdgeVertex2(int e)`, `getEdgeObjectsConnectedToVertex(int v)` and `getEdgesConnectedToVertex(int v)`;
- path search: `findAllPaths(int from, Predicate<V> pathComplete, Predicate<? super E> pathCancelled)` (optionally with a `Comparator` to order the results), returning Trove `TIntArrayList`s.

Most mutating methods come in a `(…, boolean notify)` variant so a batch of changes can be applied silently and the listeners notified once at the end.

`UndirectedGraphImpl` is the implementation. Internally it keeps an `ArrayList` of `Vertex` and `Edge` holders, tracks freed slots in a `TIntHashSet` of available vertices and a `TIntLinkedList` of removed edges (so indices can be recycled), and lazily maintains a per-vertex adjacency-list cache (`TIntArrayList[]`, guarded by a `ReentrantLock`) used by traversal and connected-component computations.

### Traversal and observation

Two patterns drive the graph:

- **Observer** — registering an `UndirectedGraphListener<V, E>` (via `addListener`) receives callbacks for every structural change: `vertexAdded`, `vertexRemoved`, `edgeAdded`, `edgeRemoved`, the pre-removal hooks `edgeBeforeRemoval` / `allEdgesBeforeRemoval`, and the bulk `allVerticesRemoved` / `allEdgesRemoved`. `DefaultUndirectedGraphListener` is a no-op base to override selectively. This is the mechanism the IIDM node/breaker topology relies on to stay in sync with its underlying graph.
- **Visitor / callback** — `traverse(int v, TraversalType traversalType, Traverser traverser)` walks the graph from a start vertex (or from several, via the `int[]` overload). `TraversalType` selects `DEPTH_FIRST` or `BREADTH_FIRST`. The `Traverser` functional interface is called for every edge encountered:

  ```java
  TraverseResult traverse(int v1, int e, int v2);
  ```

  and returns a `TraverseResult` controlling the walk: `CONTINUE`, `TERMINATE_PATH` (stop this branch only) or `TERMINATE_TRAVERSER` (stop the whole traversal).

`GraphUtil` holds the connected-component algorithm. `computeConnectedComponents(TIntArrayList[] adjacencyList)` returns a `ConnectedComponentsComputationResult` exposing `getComponentNumber()` (the component index of each vertex) and `getComponentSize()`, with components ordered by decreasing size.

## Matrix and solvers

Package `com.powsybl.math.matrix` provides a matrix abstraction designed so that algorithms can be written once and run against either dense or sparse storage. As the package documentation notes, the dense path is backed by Jama (good for small matrices and tests) and the sparse path by SuiteSparse KLU (for large, sparse systems such as power-flow Jacobians).

### The `Matrix` abstraction

`Matrix` is the interface; `AbstractMatrix` (itself extending `AbstractMathNative`) is the shared base. Beyond `getRowCount()` / `getColumnCount()`, the interface offers:

- element access: `set(int i, int j, double value)`, `add(int i, int j, double value)`, and index-based fast paths (`addAndGetIndex`, `setAtIndex`, `addAtIndex`, and their `Quick` no-bounds-check variants) for performance-critical fill loops. `addAndGetElement(...)` returns an `Matrix.Element` handle whose `set`/`add` can update a known cell repeatedly;
- iteration over non-zeros: `iterateNonZeroValue(ElementHandler handler)` and `iterateNonZeroValueOfColumn(int j, ElementHandler handler)`, where `ElementHandler.onElement(int i, int j, double value)` is the callback;
- linear algebra: `times(Matrix other)`, `add(Matrix other)`, `transpose()`, and `decomposeLU()`;
- conversion: `toDense()`, `toSparse()`, `to(MatrixFactory)`, `copy(MatrixFactory)`, plus the static `createFromColumn` / `createFromRow` helpers.

`DenseMatrix` stores its values column-major in a direct little-endian `ByteBuffer` (cell `(i, j)` at offset `(j * rowCount + i) * Double.BYTES`), which makes it cheap to hand to native code and to convert to a Jama matrix for the linear-algebra operations. `SparseMatrix` uses the **Compressed Sparse Column (CSC)** layout: a `columnStart` array of length `columnCount + 1`, a `rowIndices` array and a `values` array (the last two are Trove-backed). Because of CSC, the fill methods require cells to be written column by column in ascending order. A `rgrowthThreshold` (default `1e-10`) controls reciprocal-pivot-growth checking during factorization.

### Factories and LU decomposition

Two **abstract-factory** chains keep algorithms storage-agnostic:

- `MatrixFactory.create(int rowCount, int columnCount, int estimatedValueCount)` is implemented by `DenseMatrixFactory` and `SparseMatrixFactory` (the latter optionally configured with a custom growth threshold). Code that accepts a `MatrixFactory` works unchanged on either representation — this is how the load-flow and sensitivity engines switch between dense and sparse solvers from configuration.
- `decomposeLU()` returns an `LUDecomposition`, an `AutoCloseable` whose `solve(double[] b)` / `solveTransposed(double[] b)` (and `DenseMatrix` overloads) solve the system in place, and whose `update(boolean allowIncrementalUpdate)` re-factorizes when only the values changed. `DenseLUDecomposition` delegates to Jama; `SparseLUDecomposition` is a thin wrapper over native KLU, tracking the native factorization by a per-instance UUID through `native init/update/solve/release` calls — hence the need to `close()` it to free native memory.

`ComplexMatrix` adds complex-valued support on top of two `DenseMatrix` (real and imaginary parts), with `set`/`get` using Apache Commons Math `Complex`, plus `transpose`, `scale`, and conversion to and from a real Cartesian (2×2-block) `DenseMatrix` via `toRealCartesianMatrix()` / `fromRealCartesian(...)`. `MatrixException` is the dedicated error type.

### MATLAB serialization

`com.powsybl.math.matrix.serializer.SparseMatrixMatSerializer` exports and imports a `SparseMatrix` to and from the MATLAB `.mat` binary format (static `exportMat` / `importMat` over `OutputStream`/`InputStream` or `Path`), bridging through EJML's `DMatrixSparseCSC` and the MFL MAT-file library. This is mainly a debugging and interoperability aid.

## The KINSOL solver

Package `com.powsybl.math.solver` wraps the SUNDIALS KINSOL nonlinear solver. `Kinsol` is constructed with a `SparseMatrix` (the Jacobian) and two caller-supplied callbacks, both inner interfaces of `Kinsol`:

- `FunctionUpdater.update(double[] x, double[] f)` — fills the residual vector `f` for the current iterate `x`;
- `JacobianUpdater.update(double[] x, SparseMatrix j)` — fills the Jacobian for `x`.

`solve(double[] x, KinsolParameters parameters)` (and `solveTransposed`) run the native solver, returning a `KinsolResult` with the final `KinsolStatus` and the iteration count. `KinsolParameters` is a fluent settings object (`maxIters`, `msbset`, `msbsetsub`, `fnormtol`, `scsteptol`, `lineSearch`). `KinsolContext` is the bridge object passed across the JNI boundary that the native code calls back into (`updateFunc`, `updateJac`, plus error/info logging routed to SLF4J). `KinsolStatus` mirrors the SUNDIALS status codes (`KIN_SUCCESS`, `KIN_LINESEARCH_NONCONV`, `KIN_MAXITER_REACHED`, …) with `fromValue(int)` to decode them, and `KinsolException` reports failures.

## Number helpers

`com.powsybl.math.casting.Double2Float` provides `safeCasting(double)` / `safeCasting(Double)` for narrowing `double` to `float`: it maps the special values cleanly (NaN, the infinities, ±`MAX_VALUE`) and throws `IllegalArgumentException` when a finite value would overflow the float range, so loss-of-range is caught rather than silently turning into infinity.
