# dsl

The `dsl` module is the reusable infrastructure shared by every PowSyBl Groovy DSL. It defines no DSL of its own: it provides the pieces that the concrete DSLs (contingencies, actions, dynamic simulation, time series, ...) build on — a pre-configured Groovy shell, a compile-time AST transformation, an extension SPI, and a small expression-tree library that lets a script *capture* an arithmetic/boolean expression instead of evaluating it immediately. See the [scripting layer overview](../scripting.md) for how it fits with the other scripting modules.

It is a mixed Java/Groovy module: the shell wiring and operator overloading are written in Groovy (`dsl/src/main/groovy`), while the AST transformation, the extension interface and the expression tree are Java (`dsl/src/main/java`).

## Building and configuring a shell

`DslLoader` (Groovy) is the base class for DSL loaders. A loader wraps a `GroovyCodeSource` — built from a `File`, a `String` or, indirectly, the static helpers in `GroovyScripts` (`load(InputStream)` / `load(Path)`, both UTF-8). The key contribution is the static factory `createShell(Binding[, ImportCustomizer])`, which every DSL uses to obtain a consistently configured `GroovyShell`:

```groovy
static GroovyShell createShell(Binding binding, ImportCustomizer imports) {
    def astCustomizer = new ASTTransformationCustomizer(new PowsyblDslAstTransformation())
    def config = new CompilerConfiguration()
    config.addCompilationCustomizers(astCustomizer, imports)
    // Add a check on thread interruption in every loop (for, while) in the script
    config.addCompilationCustomizers(new ASTTransformationCustomizer(ThreadInterrupt.class))
    ExpressionDslLoader.prepareClosures(binding)
    new GroovyShell(binding, config)
}
```

Three things happen here:

- `PowsyblDslAstTransformation` is installed as a compilation customiser — the hook through which comparison/logical expressions are rewritten (see below);
- Groovy's own `ThreadInterrupt` transformation is added, injecting a thread-interruption check into every loop so a long-running or runaway script can be cancelled by interrupting its thread;
- `ExpressionDslLoader.prepareClosures(binding)` installs the operator overloads used by the expression AST, on *every* shell `DslLoader` builds.

`GroovyScripts` (Java) is the small UTF-8 `GroovyCodeSource` loader. `GroovyDslConstants` holds the single shared constant `SCRIPT_IS_RUNNING` (`"scriptIsRunning"`). `GroovyUtil` (Groovy) exposes two dynamic-dispatch helpers, `callProperty` and `callMethod`. `DslException` is the module's runtime exception.

## The AST transformation

`AbstractPowsyblDslAstTransformation` is an `ASTTransformation` parameterised by a `Function<SourceUnit, ClassCodeExpressionTransformer>`. Its `visit` walks the script's `ModuleNode`, applying the supplied expression transformer to the body of every method and to the top-level statement block. This is the generic mechanism any DSL can reuse to rewrite expressions at compile time.

`PowsyblDslAstTransformation` is the concrete transformation installed by `DslLoader`. Its inner `CustomClassCodeExpressionTransformer` rewrites binary and unary expressions into method calls, so the result can be intercepted by metaclass methods at run time:

| Source operator | Rewritten to |
|-----------------|--------------|
| `>`, `>=`, `<`, `<=`, `==`, `!=` | `left.compareTo2(right, "<op>")` |
| `&&` | `left.and2(right)` |
| `\|\|` | `left.or2(right)` |
| `!x` (a `NotExpression`) | `x.not()` |

Anything else is left untouched and visited recursively. The point is that ordinary numeric Groovy code keeps its normal semantics, but when one operand is an `ExpressionNode` the rewritten `compareTo2`/`and2`/`or2`/`not` methods (defined by `prepareClosures`) build an AST node instead of computing a value.

## The extension SPI

DSLs are made extensible through `ExtendableDslExtension<E extends Extendable<E>>`. An implementation declares the extended type via `getExtendableClass()` and contributes to the DSL through `addToSpec(MetaClass extSpecMetaClass, List<Extension<E>> extensions, Binding binding)` — i.e. it adds methods/properties to a Groovy `MetaClass` and entries to the script `Binding`. Concrete DSLs sub-interface this SPI (for example `contingency-dsl` defines `ContingencyDslExtension extends ExtendableDslExtension<Contingency>`) and discover implementations with `ServiceLoader`, so third parties can extend a DSL without modifying it (see [the plugin mechanism](../index.md#the-plugin-mechanism-service-provider-interface)).

## The expression AST

The `com.powsybl.dsl.ast` package is a self-contained tree representation of arithmetic/boolean expressions. It is used by DSLs that need to capture an expression and evaluate it later (against changing data) rather than evaluating it once in Groovy. The root type is the `ExpressionNode` interface, whose single method is a typed visitor accept:

```java
public interface ExpressionNode {
    <R, A> R accept(ExpressionVisitor<R, A> visitor, A arg);
}
```

### Node hierarchy

- **Literals** — `AbstractLiteralNode` (with `getType()` returning a `LiteralType` and `getValue()`) is the base of `IntegerLiteralNode`, `DoubleLiteralNode`, `FloatLiteralNode`, `BigDecimalLiteralNode`, `BooleanLiteralNode` and `StringLiteralNode`. `LiteralType` enumerates `FLOAT`, `DOUBLE`, `INTEGER`, `BOOLEAN`, `BIG_DECIMAL`, `STRING`. All literals dispatch to `visitLiteral`.
- **Binary operators** — `AbstractBinaryOperatorNode` holds non-null `left`/`right` children and is extended by `ArithmeticBinaryOperatorNode` (operator enum `ArithmeticBinaryOperator`: `PLUS`, `MINUS`, `MULTIPLY`, `DIVIDE`), `ComparisonOperatorNode` (`ComparisonOperator`: `LESS_THAN`, `LESS_THAN_OR_EQUALS_TO`, `GREATER_THAN`, `GREATER_THAN_OR_EQUALS_TO`, `EQUALS`, `NOT_EQUALS`) and `LogicalBinaryOperatorNode` (`LogicalBinaryOperator`: `AND`, `OR`).
- **Unary operators** — `AbstractUnaryOperatorNode` and `LogicalNotOperator` (with `LogicalNotOperator`'s operator enum), wrapping a single child.

### Visitors

`ExpressionVisitor<R, A>` has one method per node family (`visitLiteral`, `visitComparisonOperator`, `visitLogicalOperator`, `visitArithmeticOperator`, `visitNotOperator`). `DefaultExpressionVisitor` provides a no-op/recursive default so subclasses override only what they need. Two visitors ship with the module:

- `ExpressionEvaluator` — evaluates a tree to a value (`evaluate(node)`). Arithmetic and comparison operands are coerced to `double`; logical/not operands must be `Boolean`; a non-conforming operand raises a `PowsyblException`.
- `ExpressionPrinter` — renders a tree back to a fully-parenthesised string (e.g. `(($x + 2) > 3)`), with `toString(node)`, `print(node[, OutputStream])` static helpers and `Writer`/`OutputStream`/charset constructors.

`ExpressionHelper` is the factory of static `newXxx` methods (`newIntegerLiteral`, `newArithmeticBinaryOperator`, `newComparisonOperator`, `newLogicalBinaryOperator`, `newLogicalNotOperator`, ...) used to build all of the above.

### Bridging Groovy syntax to the AST

`ExpressionDslLoader` (Groovy, extends `DslLoader`) is where Groovy operators become AST nodes. Its static `prepareClosures(Binding)` installs metaclass overloads:

- arithmetic — for each of `plus`/`minus`/`multiply`/`div`, it overloads the method on `ExpressionNode` and on `Integer`, `Float`, `Double`, `BigDecimal` so that, when either operand is an `ExpressionNode`, the result is an `ArithmeticBinaryOperatorNode` (numeric literals on the other side are wrapped automatically). `someNode + 2` therefore yields a node, while `1 + 2` stays numeric;
- comparison — `compareTo2(value, op)` (the target of the AST transformation's rewrite) builds a `ComparisonOperatorNode` when an `ExpressionNode` is involved, and falls back to the plain Groovy comparison on ordinary objects;
- logical — `and2`/`or2`/`not` build `LogicalBinaryOperatorNode` / `LogicalNotOperator`, again falling back to native boolean logic when no node is involved.

`createExpressionNode(value)` wraps a raw Groovy value into the matching literal node (or returns it unchanged if already a node), and `load()` evaluates the wrapped `GroovyCodeSource` in a `createShell` shell and converts the result with `createExpressionNode`, turning compilation failures into a `PowsyblException`. Because `DslLoader.createShell` calls `prepareClosures` for every shell, any DSL gets this behaviour for free.
