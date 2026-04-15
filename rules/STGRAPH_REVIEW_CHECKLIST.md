# STGraph Model Review Checklist

Step-by-step procedure for analyzing any `.stg` model. Derived from the validation logic in `STNode.checkName()`, `ValueNode.checkDefinition()`, `ValueNode.checkVariable()`, `STNode.checkExpressionDefinition()`, `STGraphExec.setupProperties()`, and `STGraphChecker.java`.

---

## Phase 1: Structural Inspection

Parse the `.stg` XML and collect the following inventory.

### 1.1 Model Header (`<head>`)

- [ ] `time0`, `time1`, `timeD` are present and form a valid range: `time1 > time0` and `timeD > 0`
- [ ] `numSteps` is consistent: should equal `(int)(1 + (time1 - time0) / timeD)`
- [ ] `integrationMethod` is 0 (Euler), 1 (RK2), or 2 (RK23). RK2/RK23 only matter if the model has state nodes
- [ ] `exceptionHandling` is noted: 0 = continue (masks errors), 1 = halt (recommended for debugging)
- [ ] `indexOrigin` is noted: 0 or 1. Affects `[x:y]` range generation and array indexing semantics
- [ ] `timeFrame` is noted: 0 = standard, 1 = instantaneous, 2 = windowed, 3 = playmode
- [ ] If `timeFrame` is 2 or 3: `maxSteps` is set and > 0
- [ ] XML element order is correct: head, nodes, texts, edges, widgets, groups, reports (parser reads sequentially with `nextElement()`)

### 1.2 Node Inventory

For each `<node>`:

- [ ] `name` is valid: not empty, not starting with digit, no forbidden chars (space `.?!%&+-*/#,()[]{}'"`) , no non-ASCII, not a reserved name (system variable or function name), not duplicated
- [ ] `type` is `ValueNode` or `ModelNode`
- [ ] `pos-x`, `pos-y`, `width`, `height` are present (integer pixel values)

**For ValueNodes:**
- [ ] `valueType` is present: 0 (algebraic), 1 (state), or 2 (state with output)
- [ ] If `valueType=0`: `expression` is present and non-empty
- [ ] If `valueType=1`: both `stateInit` and `stateTrans` are present and non-empty
- [ ] If `valueType=2`: all three of `stateInit`, `stateTrans`, and `expression` are present and non-empty
- [ ] `isIn`, `isOut`, `isGlobal`, `isVectorOut` are `true` or `false` (not missing, though defaults apply)

**For ModelNodes:**
- [ ] `systemName` contains a valid `.stg` filename
- [ ] `subVarNames` lists all input node names from the submodel, comma-separated and alphabetically sorted
- [ ] `superExpression0`..`superExpressionN` count matches the number of entries in `subVarNames`
- [ ] Each `superExpressionN` is a valid STEL expression in the parent model's scope

### 1.3 Edge Inventory

For each `<edge>`:

- [ ] `source` matches an existing `<node name="...">`
- [ ] `target` matches an existing `<node name="...">`
- [ ] No self-loops (source != target)
- [ ] `numpoints` is consistent with the number of `pNx`/`pNy` attributes present

### 1.4 Widget Inventory

For each `<widget>`:

- [ ] `type` is a known widget class: `ChartWidget`, `DataTableWidget`, `SliderWidget`, `KnobWidget`, `GaugeWidget`, `InputTableWidget`, `InputTextWidget`, `OutputTextWidget`, `ToggleButtonWidget`, `ToggleIndicatorWidget`, `FourButtonWidget`, `MatrixViewerWidget`
- [ ] `ysourcena` / `sourcena` node references match existing node names
- [ ] `xsourcena` references are valid (typically `vTime` for time-series charts)
- [ ] For chart widgets: nodes referenced in `ysourcena` are marked `isOut=true`
- [ ] For chart widgets showing time series: referenced output nodes have `isVectorOut=true` if they produce vector values

---

## Phase 2: Syntax Validation

Check every expression against STEL syntax rules.

### 2.1 Expression Parsing

For each expression (`expression`, `stateInit`, `stateTrans`, and every `superExpressionN`):

- [ ] Expression parses without error (balanced parentheses, brackets, braces; valid operator usage; correct function call syntax)
- [ ] All referenced variable names either:
  - Are names of other nodes in the same graph
  - Are system variables (`time`, `time0`, `time1`, `timeD`, `index`, `numSteps`, `vTime`, `vIndex`, `pi`, `e`)
  - Are private variables (`$0`, `$1`, `$i`, `$a0`..`$a3`, `$numArgs`, `$v0`..`$vN`, `$w0`..`$w3`, `$i0`..`$i5`, `$p0`..`$p2`) and are used in the appropriate context
  - Are submodel output references in `SubmodelName.nodeName` format
  - Are global nodes from submodels (`isGlobal=true`)
  - Are custom function calls (names starting with `_`)
- [ ] All function calls use valid function names (built-in: ~90 in `it.liuc.stgraph.fun/`; user-defined: from `.stf` libraries; in-model: nodes prefixed with `_`)
- [ ] All function calls have the correct number of arguments

### 2.2 Context-Sensitive Variable Rules

These rules are enforced by `ValueNode.checkVariable()`:

- [ ] **`this` is ONLY used in:**
  - `stateTrans` of a state node (valueType 1 or 2)
  - `expression` of a state-with-output node (valueType 2)
  - NOT in `stateInit` (ERR.THIS_NOT_ALLOWED)
  - NOT in algebraic node expressions (ERR.THIS_NOT_ALLOWED)
- [ ] **`me` is ONLY used in:**
  - `expression` of a state-with-output node (valueType 2)
  - NOT anywhere else (ERR.ME_NOT_ALLOWED)
- [ ] **`integral(x)` is ONLY used in `stateTrans`** -- it expands to `this + x * timeD`, which requires a state context
- [ ] **Every non-system, non-global, non-private variable referenced in an expression has a corresponding incoming edge** from that node (ERR.NON_CONNECTED_NODE)

### 2.3 Formatted Expression Sync

- [ ] If `fExpression` is present, it matches the semantic content of `expression` (ignoring backtick whitespace markers and XML entity differences)
- [ ] Same for `fStateInit`/`stateInit` and `fStateTrans`/`stateTrans`
- [ ] If in doubt, removing the `f`-prefixed elements is safe -- STGraph regenerates them on load

---

## Phase 3: Logic Validation

Check the model's computational logic.

### 3.1 Dependency Graph Analysis

- [ ] **No circular dependencies among algebraic nodes.** Trace the dependency graph: if node A references node B references node C references node A, the topological sort will fail with `ERR.WRONG_TOPOLOGY`. (State nodes are allowed to form cycles via `this`.)
- [ ] **Every edge is used.** For each edge, check that the target node's expression actually references the source node. Unused edges are flagged by `STGraphChecker.checkConnections()`.
- [ ] **No dead-end nodes.** Every non-output node should have at least one outgoing edge (another node depends on it) or be marked `isOut=true`. Nodes with no outgoing edges and not marked as output are wasted computation. Flagged by `STGraphChecker.checkTerminals()`.
- [ ] **No missing edges.** For each variable referenced in an expression, check that an incoming edge exists from that node. (Exception: system variables, global nodes, private variables.)

### 3.2 State Node Logic

- [ ] **`stateInit` does NOT reference `this`.** There is no prior state at t=0. This creates a circular dependency that the engine won't catch cleanly.
- [ ] **`stateInit` does NOT depend on other state nodes' initial values** unless the dependency order is explicitly controlled. State nodes are initialized in list order, not dependency order.
- [ ] **`stateTrans` correctly models the intended dynamics.** For accumulation: `this + flow * timeD` or `integral(flow)`. For discrete update: `this + delta`.
- [ ] **State nodes using `integral()` with RK2/RK23:** The integration method is model-wide. If only some state nodes need high-order integration, the others still get multi-phase evaluation (harmless but wasteful).

### 3.3 Numerical Safety

These are flagged by `STGraphChecker.checkIssuesInDefs()`:

- [ ] **Division operations:** Denominator can never be zero, or guarded with `if(denom!=0, num/denom, fallback)`
- [ ] **`sqrt(x)` operations:** Argument is guaranteed non-negative, or guarded with `if(x>=0, sqrt(x), 0)`
- [ ] **`log(x)` operations:** Argument is guaranteed positive, or guarded with `if(x>0, log(x), fallback)`
- [ ] **Empty vector operations:** Operations on potentially empty vectors (`@v==0`) are guarded. `+/[]` returns `0.0`, but `max/([])` or `[][0]` may error.

### 3.4 Submodel Validation

- [ ] **Submodel file exists** in the same directory as the parent model or in `mod_lib/`
- [ ] **`subVarNames` is alphabetically sorted.** This is the binding contract. If the sort order is wrong, `superExpressionN` will feed values to the wrong inputs.
- [ ] **`superExpression` count matches `subVarNames` count.** Missing expressions leave submodel inputs unbound. Extra expressions are ignored but indicate a mismatch.
- [ ] **Submodel output nodes accessed from the parent are marked `isGlobal=true`** in the submodel. Otherwise they are invisible to the parent model.
- [ ] **Submodel output references use correct dot notation:** `SubmodelNodeName.outputNodeName` (not `SubmodelFileName.outputNodeName`)
- [ ] **If the same submodel file is used multiple times** (e.g., `Specie.stg` for different species), each instance gets its own `superExpression` set. Verify that each instance's parameter bindings are correct and not copy-paste duplicates.

### 3.5 Output and Display

- [ ] **Nodes used in chart `ysourcena` are marked `isOut=true`.** Otherwise their value history is not accumulated and charts show nothing or the last value only.
- [ ] **Nodes producing vector values shown in charts have `isVectorOut=true`.** If `isVectorOut=false`, only the current value is stored per step (no time-series accumulation for vector nodes).
- [ ] **Widget `ysourcena`/`sourcena` references match actual node names.** Typos produce silent failures.
- [ ] **Slider/knob widget `source` matches an input node name.** The widget writes to the input node's value.

---

## Phase 4: Runtime Behavior Check

If the model can be loaded and run:

### 4.1 Pre-Execution

- [ ] Model loads without errors (no XML parse failures, no missing submodel files)
- [ ] Graph checker reports no issues (Analysis menu in STGraph)
- [ ] All nodes have assigned execution priorities (no `ERR.WRONG_TOPOLOGY`)
- [ ] The model is marked `runnable` (no errors, non-empty graph, is top graph)

### 4.2 Step 0

- [ ] All `stateInit` expressions evaluate to non-null values
- [ ] All algebraic node expressions evaluate to non-null values
- [ ] No `ERR.COMP.NULLVALUE.FIRSTCALL` errors
- [ ] Initial state values are physically meaningful (correct sign, correct magnitude, correct dimensions)

### 4.3 Full Execution

- [ ] Simulation runs to completion without `ERR.COMP.*` errors
- [ ] No NaN or Infinity propagation (check output node values at the end)
- [ ] State variables remain bounded (no exponential blowup from numerical instability)
- [ ] If using RK2/RK23: results differ from Euler in the expected direction (smoother, more accurate)
- [ ] Output charts show expected behavior (correct shape, correct scale, correct units)

### 4.4 Interrupt Conditions (if `withInterrupts=true`)

- [ ] Custom properties `Min`/`OnBelowMin`, `Max`/`OnAboveMax`, `OnZero`, `OnTrue`, `OnFalse` are correctly formatted
- [ ] Interrupt action strings use valid syntax: `pause("msg")`, `warn("msg")`, or `end("msg")`
- [ ] Interrupt conditions do not fire spuriously at t=0
