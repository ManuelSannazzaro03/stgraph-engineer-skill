# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What STGraph Is

STGraph is a Java Swing desktop application for creating, editing, and executing models of dynamical systems using the state-variable approach to Systems Theory. Developed by Luca Mari at Universita Cattaneo - LIUC. Licensed under GNU GPL 2.0.

Models are directed graphs of nodes connected by edges. Each node holds an expression in the **STEL** (STGraph Expression Language). The graph topology determines evaluation order. Simulation advances discrete time steps, updating state variables via transition expressions.

---

## Workspace Layout

### Information Hierarchy

When working in this workspace, consult sources in this priority order:

1. **Source code** (`stgraph-main/stgraph/src/`, `stgraph-main/myjep/src/`) — the ground truth for how STEL actually works, what operators exist, how expressions are parsed and evaluated
2. **Test models** (`stgraph/test/*.stg`) — verified reference models with known correct outputs
3. **Function libraries** (`stgraph/fun_lib/*.stf`) — canonical examples of custom function syntax
4. **Properties files** (`src/datafiles/functions_en.properties`, `variables_en.txt`) — function and variable documentation strings
5. **PDF documentation** (`Documentazione/`) — conceptual introduction to STGraph and STEL; useful for understanding intent but may lag behind the code
6. **User models** (`Progetto Stagno/`) — real-world models; useful for patterns but may contain domain-specific assumptions

**Never guess STEL syntax.** If uncertain whether a function or operator exists, verify against the source code (`it.liuc.stgraph.fun/` for functions, `myjep/src/org/nfunk/jep/Parser.jjt` for operators, `STInterpreter.java` for operator wiring).

### Multi-Project Structure

| Project | Purpose | Output JAR |
|---|---|---|
| **stgraph** | Main application | `stgraph.jar` |
| **myjep** | Forked JEP 2.4 math parser (Tensor type, custom grammar, operators) | `jep.jar` |
| **stgraphfun** | User-defined function loader (reads `.stf` XML files at runtime) | `stgraphfun.jar` |
| **stgraphdoc** | Function documentation generator (`STFunctionDocumenter`) | -- |
| **jgraph** | Forked JGraph library (graph visualization) | `jgraph.jar` |
| **ekit** | HTML editor component | `ekit.jar` |
| **steel** | Gauge/dial UI components | `steel.jar` |
| **surfaceplotter** | 3D surface visualization | `surfaceplotter.jar` |
| **trident** | Animation framework | `trident.jar` |

**Critical:** `myjep` is NOT standard JEP. It adds `Tensor` and `Matrix` as first-class types, modifies the parser grammar, adds `#` (concatenate), `##` (decatenate), `@` (size), `{...}` (subexpression), and meta-operators (`f/x`, `f\x`, `f|x`). Changes are documented in `myjep/readme.txt`.

### Key Source File Locations

| What | Where |
|---|---|
| Parser grammar (operator definitions) | `myjep/src/org/nfunk/jep/Parser.jjt` |
| Tensor data type | `myjep/src/org/nfunk/jep/type/Tensor.java` |
| Interpreter + operator wiring | `stgraph/src/it/liuc/stgraph/STInterpreter.java` |
| Simulation engine | `stgraph/src/it/liuc/stgraph/STGraphExec.java` |
| Graph data model + file I/O | `stgraph/src/it/liuc/stgraph/STGraphImpl.java` |
| Node base class | `stgraph/src/it/liuc/stgraph/node/STNode.java` |
| Value node (algebraic/state) | `stgraph/src/it/liuc/stgraph/node/ValueNode.java` |
| Model node (submodel) | `stgraph/src/it/liuc/stgraph/node/ModelNode.java` |
| Custom function engine | `stgraph/src/it/liuc/stgraph/fun/CustomFun2.java` |
| Built-in functions | `stgraph/src/it/liuc/stgraph/fun/` (one class per function) |
| Function descriptions | `stgraph/src/datafiles/functions_en.properties` |
| System variable docs | `stgraph/src/datafiles/variables_en.txt` |
| Custom function libraries | `stgraph/fun_lib/*.stf` |
| Test models | `stgraph/test/*.stg` |

---

## Running, Building, Testing

**Run** (from `stgraph/`): `./stgraph.sh` (Linux), `./stgraph.command` (macOS), `stgraph.bat` (Windows).
CLI: `-l=en|it` (language), `-x=e|w` (easy/web mode), `-f=<file>` (open model), `-s` (no splash). Requires JRE 1.8+.

**Build** via Eclipse `.jardesc` files. Order matters: `myjep` -> `stgraphfun` -> `stgraphdoc` -> `stgraph`. When modifying functions, run `STFunctionDocumenter` before rebuilding `stgraph.jar`.

**Test** via `it.liuc.stgraph.tools.STTester` -- standalone `main()` that runs `stgraph/test/*.stg` models headless (`EXECMODE_NOGUIENGINE`), executes each to completion, and asserts specific node final values.

---

## STEL Language Reference

### Fundamental Design Rules

STEL is a **pure functional expression language**. Strict rules:

1. **No imperative statements.** No assignments, no loops, no side effects. Only expressions that return values.
2. **Single data type.** Everything is a `Tensor` -- scalars, vectors, matrices, and higher-order arrays are all the same type. Scalars are order-0 tensors. Vectors are order-1. Matrices are order-2.
3. **Boolean encoding.** `true` = value > 0 (specifically `1.0`). `false` = value <= 0 (specifically `0.0`). There is no boolean type.
4. **Polymorphic operations.** All operators and functions automatically broadcast across tensor dimensions (APL-style). Scalar + vector = element-wise. Matching-dimension arrays = element-wise. Incompatible dimensions = exception.
5. **Graph-driven evaluation.** Expression evaluation order is determined by the directed graph topology, not by textual order. A node can reference any other node it has an incoming edge from, by name.
6. **`this` is reserved.** Inside state transition expressions, `this` refers to the current state value. It is meaningless in algebraic expressions.

### Operator Precedence (lowest to highest)

| Priority | Operators | Notes |
|---|---|---|
| 1 | `\|\|` | Logical OR |
| 2 | `&&` | Logical AND |
| 3 | `==` `!=` | Equality |
| 4 | `<` `>` `<=` `>=` | Relational |
| 5 | `+` `-` | Additive |
| 6 | `*` `/` `%` `#` `##` | Multiplicative, modulus, concatenation, decatenation |
| 7 | `!` unary`+` unary`-` `@` meta-ops | Unary prefix, size operator, meta-operators |
| 8 | `^` | Power (right-associative) |

### Array and Structural Operators

| Operator | Meaning | Example |
|---|---|---|
| `[a,b,c]` | Vector literal | `[1,2,3]` |
| `[[a,b],[c,d]]` | Matrix literal | `[[1,2],[3,4]]` |
| `[]` | Empty vector | |
| `[x:y]` | Range vector (step 1) | `[1:5]` -> `[1,2,3,4,5]` |
| `[x:y:z]` | Range vector (step z) | `[0:1:0.25]` -> `[0,0.25,0.5,0.75,1]` |
| `x[v0,v1,...]` | Indexing (= `get(x,v0,...)`) | `m[[0],[1,2]]` |
| `x#y` | Concatenation (= `conc(x,y)`) | `[1,2]#[3]` -> `[1,2,3]` |
| `x##y` | Decatenation (= `dec(x,y)`) | |
| `@x` | Size (= `size(x)`) | `@[1,2,3]` -> `3` |
| `{expr}` | Subexpression, stores result in `$w0`..`$w3` | |

### Meta-Operators (Higher-Order, APL-style)

These take a dyadic operator or function `f` and apply it across an array. They are a distinctive STEL feature. **`f` must be a dyadic operator** (`+`, `-`, `*`, `/`, `%`, `^`, `&&`, `||`, `<`, `>`, `==`, `<=`, `>=`, `!=`) **or a named dyadic function in parentheses** (e.g., `max/(v)`, `min/(v)`).

| Meta-op | Name | Semantics | Example |
|---|---|---|---|
| `f/x` | Reduction | Fold left: `f(f(x[0],x[1]),x[2])...` -> scalar | `+/[1:4]` = `10` |
| `f\x` | Scan | Cumulative fold: `[x[0], f(x[0],x[1]), ...]` | `+\[1:3]` = `[1,3,6]` |
| `f\|x` | PairScan | Pairwise: `[f(x[0],x[1]), f(x[1],x[2]), ...]` | `+\|[1:4]` = `[3,5,7]` |

Dimensional variants: `f/[n]x`, `f\[n]x`, `f|[n]x` operate along dimension `n`.

### System Variables

**Time/simulation (read-only during execution):**

| Variable | Description |
|---|---|
| `time0` | Start time |
| `time1` | End time |
| `timeD` | Time step |
| `time` | Current time |
| `index` | Current step index |
| `numSteps` | Total steps |
| `vTime` | Vector of all time values |
| `vIndex` | Vector of all index values |

**Constants:** `pi`, `e`

**Node self-reference:** `this` (current state, only in state transition expressions), `me` (the node itself)

**Iteration variables (inside `iter(x,e,z)`):** `$i` (index), `$0` (accumulator), `$1` (current element)

**Array constructor variables (inside `array(v,e)`):** `$i0`..`$i5` (dimension indices), `$p0`..`$p2` (previous elements)

**Custom function variables:** `$a0`..`$a3` (arguments), `$numArgs` (argument count), `$v0`..`$vN` (locals in `.stf` functions)

**Subexpression variables:** `$w0`..`$w3` (results of `{...}` blocks, assigned in order)

**Batch variables:** `__batch` (current batch step), `__vBatch` (batch vector)

### Built-in Function Categories

**Math:** `acos`, `asin`, `atan`, `cos`, `sin`, `tan`, `exp`, `log` (natural or with base), `sqrt`, `int` (floor), `sign`, `deg2rad`, `rad2deg`, `mod`, `wrap`, `round`, `max`, `min`, `FFT`, `integral`

**Interpolation:** `line`, `bline` (bounded), `pline` (polyline), `spline`, `sigmoid`

**Arrays:** `array(v,e)`, `get`/`set`, `getData`, `getIndex`, `size`/`order`, `conc`/`dec`, `sort`, `shuffle`, `remove`, `resize`, `shift`, `transpose`, `frequency`

**Control:** `if(c1,v1,...,cn,vn,default)` (generalized conditional, odd arg count >= 3), `iter(x,e,z)` (fold/loop), `function(x)` (define node as callable function), `isNumber`, `getCProp`, `readFromXLS`, `sysTime`, `indexOrigin`

**Distributions** (all share calling convention: `f()` = random, `f(v,x,0)` = PDF, `f(v,x,1)` = CDF, `f(v,x,2)` = inverse CDF): `gaussian`, `uniform`, `exponential`, `poisson`, `gamma`, `chiSquare`, `tDistribution`, `beta`, `binomial`. Also: `rand()`, `randInt(x)`.

**User-defined (from `.stf` libraries):**
- `mathFunctions.stf`: `abs`, `pos`, `even`, `map3to2d`
- `arrayFunctions.stf`: `vector`, `matrix`, `numRows`, `numCols`, `lastDim`, `numEl`, `prod` (matrix product), `sumif`, `select`, `filter`
- `statFunctions.stf`: `rank`, `range`, `median`, `mean`, `stdDev`, `kurtosis`, `skewness`, `correl`, `autocorrel`, `slope`, `intercept`

### Polymorphism Rules

**Monadic functions:** scalar -> scalar; array -> element-wise array of same shape.

**Dyadic functions/operators:**
- scalar + scalar -> scalar
- scalar + array -> broadcast (scalar applied to each element)
- same-shape arrays -> element-wise
- order-n + order-(n-1) with matching first (n-1) dims -> broadcast along last dim
- either operand is empty non-scalar -> `0.0`
- incompatible dimensions -> exception

### Common STEL Idioms

```
-- Sum of vector:            +/v
-- Mean:                     (+/v)/@v
-- Dot product:              +/(x*y)
-- Range:                    max/(v)-min/(v)
-- Reverse vector:           iter(x,$1#$0,[])
-- Filter zeros:             iter(x,if($1!=0,$0#$1,$0),[])
-- Discrete derivative:      (input-this)/timeD          (state-with-output node)
-- Discrete integral:        this+input*timeD             (or use integral(input))
-- Counter:                  stateInit=0, stateTrans=this+1
-- Exponential growth:       stateInit=N0, stateTrans=this+this*rate*timeD
-- Seasonal cycle:           Lmin+(1-Lmin)*(0.5+0.5*sin(2*pi*time/period))
-- Beer-Lambert decay:       I0*exp(-k*depth)
-- Matrix product:           array([numRows(x),numCols(y)],+/(x[[$i0],[]]*y[[],[$i1]]))
```

---

## Node Types and Expression Fields

### ValueNode (`type="ValueNode"`)

| `valueType` | Name | Fields used | Semantics |
|---|---|---|---|
| 0 | Algebraic | `expression` | `y(t) = f(inputs)` |
| 1 | State | `stateInit`, `stateTrans` | `x(0) = init; x(t+dt) = g(this, inputs); y = x` |
| 2 | State with output | `stateInit`, `stateTrans`, `expression` | `x(0) = init; x(t+dt) = g(this, inputs); y = f(x, inputs)` |

**Flags:** `isIn` (input node, no incoming edges), `isOut` (output, observable), `isGlobal` (visible across submodel boundary), `isVectorOut` (keep full time series).

### ModelNode (`type="ModelNode"`)

Embeds a submodel `.stg` file as a nested `STGraphExec`. The parent binds to the submodel's inputs via `superExpression0`..`superExpressionN`, positionally matched to the alphabetically sorted `subVarNames` list.

**Access submodel outputs:** `SubmodelNodeName.outputNodeName` (e.g., `Luce.L_eff_bottom`).

### Evaluation Order (per simulation step)

1. `computeInputs()` -- algebraic nodes in topological order (`sortedNodeList1`)
2. `computeStateValues()` -- state node expressions using current state
3. `computeSync()` -- algebraic nodes depending on states (`sortedNodeList2`)
4. `computeNextStates()` -- advance states via transition expressions

Integration methods: **Euler** (0) = single pass, **RK2** (1) = 3 phases with save/restore, **RK23** (2) = 5 phases.

---

## .stg XML Format

### Document Structure

```xml
<stgraph class="STGraph.decoder" version="STGraph build XX.YY">
    <head ... />
    <nodes> <node .../> ... </nodes>
    <texts> <text .../> ... </texts>
    <edges> <edge .../> ... </edges>
    <widgets> <widget .../> ... </widgets>
    <groups> ... </groups>
    <reports> ... </reports>
</stgraph>
```

**Element order is mandatory** -- the parser reads children sequentially with `nextElement()`. Groups and reports are optional (backward compatibility).

### `<head>` Attributes

| Attribute | Type | Description |
|---|---|---|
| `systemName` | String | Model display name |
| `description` | String | Model description |
| `timeFrame` | int | 0=standard, 1=instantaneous, 2=windowed, 3=playmode |
| `time0`, `time1`, `timeD` | double | Time bounds and step |
| `integrationMethod` | int | 0=Euler, 1=RK2, 2=RK23 |
| `indexOrigin` | int | 0 or 1 |
| `exceptionHandling` | int | 0=ignore, 1=halt |
| `maxSteps` | int | Max steps (windowed/playmode) |
| `batchSteps` | int | Batch repetitions |
| `simulationDelay` | int | Delay between steps (ms) |
| `stepsBeforePause` | int | Steps per animation tick |
| `width`, `height` | int | Window size |
| `scale` | double | Zoom level |
| `isGraphVisible`, `areWidgetsVisible` | boolean | Panel visibility |
| `isDataSaved` | boolean | Auto-save data |
| `forWeb` | boolean | Web mode |
| `withInterrupts` | boolean | Enable interrupt conditions |

### `<node>` Element

**Attributes:** `name`, `type` (`ValueNode` or `ModelNode`), `pos-x`, `pos-y`, `width`, `height`.

**Child elements (ValueNode):**

| Element | Description |
|---|---|
| `valueType` | 0, 1, or 2 |
| `expression` | Raw algebraic/output expression |
| `stateInit` | Raw initial state expression |
| `stateTrans` | Raw state transition expression |
| `fExpression`, `fStateInit`, `fStateTrans` | Formatted versions (backticks as whitespace markers) |
| `isIn`, `isOut`, `isGlobal`, `isVectorOut` | `true`/`false` flags |
| `font` | `"Family,size"` (e.g., `"Serif,12"`) |
| `fontcol`, `forecol`, `backcol` | `"R,G,B"` (e.g., `"0,0,255"`) |
| `format` | Number format (e.g., `"0.0##"`) |
| `customprops` | Semicolon-separated `Key=Value` pairs |
| `documentation` | Free-text documentation |

**Child elements (ModelNode):**

| Element | Description |
|---|---|
| `systemName` | Submodel `.stg` filename |
| `subvisible` | `true`/`false` |
| `subVarNames` | Comma-separated alphabetically-sorted input names |
| `superExpression0`..`N` | Parent expressions bound to submodel inputs (positional) |

### `<edge>` Element

**Attributes:** `source` (node name), `target` (node name), `label` (always `"(...)"`), `p0x`/`p0y`..`pNx`/`pNy` (control points), `numpoints`, `labpos-x`/`labpos-y`. Edges reference nodes by name.

### `<widget>` Element

**Attributes:** `type` (class name), `pos-x`, `pos-y`, `width`, `height`. Widget types: `ChartWidget`, `DataTableWidget`, `SliderWidget`, `KnobWidget`, `GaugeWidget`, `InputTableWidget`, `InputTextWidget`, `OutputTextWidget`, `ToggleButtonWidget`, `ToggleIndicatorWidget`, `FourButtonWidget`, `MatrixViewerWidget`.

Key children for `ChartWidget`: `title`, `ysourcena` (comma-separated node names), `xsourcena` (typically `vTime` repeated), `linecolors` (`__RED,__GREEN,...`), `showline`/`showdots`/`showbars` (comma-separated booleans).

### `<text>` Element

All data in attributes: `name` (prefixed with `*`), `pos-x`, `pos-y`, `width`, `height`, `content`.

### Formatted vs Raw Expressions

Every expression field has a dual: `expression`/`fExpression`, `stateInit`/`fStateInit`, `stateTrans`/`fStateTrans`. The `f`-prefixed version uses backticks (`` ` ``) as visual whitespace markers (e.g., `death_rate`+`death_starv`). XML special characters are entity-escaped in both forms.

**When editing .stg XML directly, always update both the raw and formatted versions of an expression, or remove the formatted version entirely** (STGraph will regenerate it).

---

## .stf Custom Function Format

Java XML Properties format:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <entry key="en:[user]Category__it:[user]Categoria|funcName">
        EXPRESSION // <![CDATA[en:English docs__it:Italian docs]]>
    </entry>
</properties>
```

**Key format:** `en:[user]EnglishCategory__it:[user]ItalianCategory|functionName`

**Expression body** uses `$a0`..`$a3` for arguments, `$numArgs` for count. Validate args first:
```
if($numArgs!=1,"en:One argument required__it:Richiesto un argomento",
    if($a0>=0,$a0,-$a0)
)
```

Semicolons separate sequential sub-statements; intermediate results go to `$v0`..`$vN`. Error strings use `"en:...__it:..."` pattern for bilingual messages.

---

## Workflow for Analyzing and Fixing .stg Models

### Step 1: Read the Model Structure

1. Parse the XML and identify: `<head>` params (time range, integration method, time step), all `<node>` elements (name, type, valueType), all `<edge>` elements (dependency graph), all `<widget>` elements (what's displayed).
2. Classify nodes: inputs (`isIn=true`), states (`valueType=1` or `2`), algebraic (`valueType=0`), outputs (`isOut=true`), submodels (`type="ModelNode"`).
3. Reconstruct the dependency graph from edges. Check that every node referenced in an expression has an incoming edge from that node.

### Step 2: Validate Expressions

For each node, check:
- **Algebraic nodes:** `expression` must be a valid STEL expression referencing only nodes with edges pointing to this node, system variables, and constants.
- **State nodes:** `stateInit` must not reference `this`. `stateTrans` may reference `this` (the current state). Both must be valid STEL.
- **State-with-output nodes:** all three fields (`stateInit`, `stateTrans`, `expression`) must be valid.
- **Submodel nodes:** every `superExpressionN` must be valid in the parent model's scope. The count of `superExpression*` elements must match the length of `subVarNames`.

### Step 3: Check for Common Errors

| Error | Symptom | Fix |
|---|---|---|
| `this` used in algebraic node | Runtime error or wrong value | Change `valueType` to 1 or 2, add `stateInit` |
| `this` used in `stateInit` | Circular at t=0 | Replace with a concrete initial value |
| Missing edge for referenced node | "Variable not found" at runtime | Add `<edge source="referenced" target="this_node">` |
| `integral(x)` in algebraic node | Only meaningful in state transition | Move to `stateTrans` of a state node |
| Division by zero without guard | NaN propagation | Use `if(denom!=0, num/denom, fallback)` |
| Empty vector operation | Silent `0.0` result | Check `@v > 0` before operating on `v` |
| Wrong `superExpression` index | Submodel gets wrong parameter | Verify positional match against alphabetically-sorted `subVarNames` |
| Formatted expression out of sync | No runtime effect, but confusing in GUI | Delete `fExpression`/`fStateTrans`/`fStateInit` -- STGraph regenerates them |

### Step 4: Test Changes

After modifying a `.stg` file:
1. Open in STGraph and run the simulation (key `1` for immediate run, `3` for stepped).
2. Check output widgets for expected behavior.
3. For headless validation, use `STTester` patterns: load model in `EXECMODE_NOGUIENGINE`, run `exec()`, check specific node values.

---

## Architecture (Source Code)

### Core Class Hierarchy

```
STGraph (entry point, CLI args, exec modes)
  +-- STGraphC (JFrame -- window, Spring context, i18n, undo)
       +-- manages STGraphExec instances (one per open model tab)
             +-- STGraphExec extends STGraphImpl (simulation engine)
                   +-- STGraphImpl extends JGraph (graph data model, file I/O)
```

`STGraphC` is the singleton: `STGraph.getSTC()`. Holds the Spring `ApplicationContext`.

### Simulation Execution

`STGraphExec` has five execution modes: `exec()` (immediate), `steppedExec()` (step-by-step), `timedExec()` (animated via Timer), `singleExec()` (one step), `batchExec()` (repeated runs).

All funnel into `compute()`, which runs the four-phase evaluation pipeline per step. For RK2/RK23, `compute()` saves/restores computational state and repeats phases with modified time deltas.

### Expression Engine

`STInterpreter extends JEP` replaces all standard JEP operators with tensor-aware versions (`it.liuc.stgraph.fun.Op*` classes), registers ~90 built-in functions from the Spring-injected list, loads custom functions from `.stf` files via `UserFunReader`, and sets up private variables for meta-functions.

### Spring Wiring

- `stgraph.spring.xml.properties` -- core beans (i18n, dialogs, comparators)
- `actions.base.spring.xml.properties` -- shared action beans
- `actions.default.spring.xml.properties` -- standard mode
- `actions.easy.spring.xml.properties` / `actions.web.spring.xml.properties` -- reduced sets

### i18n

Two locales: `en`, `it`. Bundles: `menus_*.properties`, `dialogs_*.properties`, `functions_*.properties`, `interpreter_*.properties`. Access: `STGraphC.getMessage(key)`.

### Adding a New Built-in Function

1. Create class in `it.liuc.stgraph.fun` extending `STFunction` (or `STMonadicFun`/`STDyadicFun`/`STDistributionFun`/`STInterpolationFun`)
2. Implement `run(Stack stack)` -- pop inputs, push Tensor result
3. Register in Spring function list
4. Add entries to `functions_en.properties` and `functions_it.properties`
5. Run `STFunctionDocumenter`, rebuild

### Adding a New Action

1. Create class in `it.liuc.stgraph.action` extending `AbstractActionDefault`
2. Add bean to `actions.base.spring.xml.properties` and/or `actions.default.spring.xml.properties`
3. Add menu text to `menus_en.properties` and `menus_it.properties`

---

## Best Practices

### Writing STEL Expressions

- **Use `integral(x)` only in `stateTrans`** -- it expands to `this + x * timeD`, which requires a state context.
- **Guard divisions:** `if(d!=0, n/d, 0)` rather than bare `n/d`.
- **Prefer meta-operators over `iter` for simple reductions:** `+/v` is clearer and faster than `iter(v,$0+$1,0)`.
- **Use `if` for piecewise, not nested ternaries:** `if(c1,v1,c2,v2,default)` accepts any odd number of args >= 3.
- **Name convention for callable functions:** prefix with `_` (e.g., `_factorial`). The node's expression uses `function(...)`.
- **Avoid referencing nodes without edges** -- the expression may parse but will use stale or zero values because the topological sort won't guarantee evaluation order.
- **State nodes need both fields:** `stateInit` for t=0, `stateTrans` for t>0. Omitting either causes runtime errors.
- **Submodel parameter binding is positional, not by name.** `superExpression0` maps to the first name in the alphabetically-sorted `subVarNames`, not necessarily to the first input declared in the submodel. Always verify the sort order.

### Editing .stg XML Directly

- **Preserve element order:** head, nodes, texts, edges, widgets, groups, reports. The parser reads sequentially.
- **Keep raw and formatted expressions in sync**, or delete the `f`-prefixed version and let STGraph regenerate it.
- **Node names are identifiers** used in expressions and edge references. Renaming a node requires updating: all expressions referencing it, all edges with it as source/target, all widget source references, all `superExpression` values in parent models if it's a submodel output.
- **Edge `source`/`target` are node names, not indices.** Adding a node without corresponding edges means no data flows.
- **XML entities in expressions:** `<` must be `&lt;`, `>` must be `&gt;`, `&` must be `&amp;` in raw XML. The parser unescapes them.
- **Test after every structural change.** Open the model in STGraph, run the simulation, verify outputs match expectations before making further changes.

### Working with Submodels (ModelNode)

- Submodel `.stg` files must be in the same directory as the parent model, or in the `mod_lib/` directory.
- Nodes marked `isGlobal=true` in the submodel are visible from the parent as `SubmodelName.nodeName`.
- The `subVarNames` list is **always alphabetically sorted**. When adding a new input to a submodel, all `superExpressionN` indices in every parent model must be recalculated.
- To check the binding: sort `subVarNames` alphabetically, then `superExpression0` feeds the first name, `superExpression1` the second, etc.

### Common Mistakes

1. **Using `this` outside state transitions** -- silently evaluates to the last-set value, producing subtle bugs.
2. **Forgetting `isVectorOut=true`** on nodes used as chart Y-sources -- the chart shows only the last value.
3. **Wrong `integrationMethod` for the model** -- RK2/RK23 only matter for state nodes using `integral()`. Algebraic-only models are unaffected.
4. **Circular dependencies in algebraic nodes** -- STGraph detects these during topological sort and reports an error, but the error message may be unclear. Check the dependency graph.
5. **Off-by-one in `indexOrigin`** -- models with `indexOrigin=0` vs `indexOrigin=1` produce different results for `[x:y]` ranges and array indexing. Check the `<head>` attribute.
6. **Empty vector edge cases** -- `@[]` returns `0`, `+/[]` returns `0.0`, but `max/([])` or accessing `[][0]` may error. Always guard with size checks.
7. **Modifying a `.stf` library function signature** -- existing models calling the old signature will break silently (wrong `$a` bindings). Check all models that use the function.
