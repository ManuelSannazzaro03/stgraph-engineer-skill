# Common STGraph Errors

Catalog of frequent mistakes in STGraph models, organized by error source. Each entry includes the error manifestation, root cause analysis (traced to specific source code), and the precise fix.

---

## 1. Expression Errors

### 1.1 `this` Used Outside State Transition

**Manifestation:** Validation error `ERR.THIS_NOT_ALLOWED`. Or, if validation is bypassed: `this` silently evaluates to whatever value was last assigned to the interpreter variable `this` by a different node's evaluation, producing wrong results with no error message.

**Root cause:** `ValueNode.checkVariable()` (line ~265) only allows `this` in `stateTrans` of a state node (valueType 1 or 2) and in `expression` of a state-with-output node (valueType 2). In algebraic nodes (valueType 0) or in `stateInit`, `this` has no defined meaning.

**Why it happens:** Modeler intends "the previous value of this node" but uses an algebraic node instead of a state node. Or modeler puts `this` in `stateInit` thinking it refers to an initial state.

**Fix:** Change the node to `valueType=1` (state) or `valueType=2` (state with output). Move the expression to `stateTrans`. Provide a concrete `stateInit` value that does NOT reference `this`.

---

### 1.2 `this` Used in `stateInit`

**Manifestation:** At step 0, `this` has no defined prior value. It evaluates to whatever was last stored in the interpreter variable -- typically `0.0` from a previous node's evaluation or from initialization. The result is an incorrect initial state that appears to work but produces subtly wrong dynamics.

**Root cause:** At step 0 (`firstCall=true`), `computeStateValues()` evaluates `stateInit` via `topOfTreeValue`. At this point, `this` has not been set to the node's own state (there is no prior state). The interpreter returns whatever residual value is in `this` from the most recent state node evaluation.

**Why it happens:** Modeler writes `stateInit: this * 2` intending a doubling of some initial condition, but `this` has no meaning at t=0.

**Fix:** Replace `this` in `stateInit` with a concrete value, a reference to another node, or a constant expression. `stateInit` should be fully determined without self-reference.

---

### 1.3 `integral()` Used in Algebraic Node

**Manifestation:** `integral(x)` expands to `this + x * timeD`. In an algebraic node, `this` is not the node's own prior value -- it is the residual interpreter variable from whatever state node was last evaluated. The result is numerically wrong with no error message.

**Root cause:** `integral()` is implemented as a standard function that references `this`. The function itself does not check whether it is being called inside a state transition. `ValueNode.checkVariable()` catches `this` usage in algebraic nodes during validation, but if `integral()` is used, the error is one level of indirection away and may not be caught if the model skips full validation.

**Why it happens:** Modeler wants to accumulate a value over time and uses `integral()` in an algebraic node expression instead of creating a state node.

**Fix:** Create a state node (valueType=1). Set `stateInit` to the desired initial value (e.g., `0`). Set `stateTrans` to `integral(x)` (or equivalently `this + x * timeD`).

---

### 1.4 `me` Used Outside State-With-Output Expression

**Manifestation:** Validation error `ERR.ME_NOT_ALLOWED`.

**Root cause:** `ValueNode.checkVariable()` only allows `me` in the `expression` field of a state-with-output node (valueType=2). `me` is set by the engine to the result of evaluating the state transition expression, and is only meaningful in that specific context.

**Fix:** Only use `me` in the `expression` field of a node with `valueType=2`.

---

### 1.5 Empty Expression Field

**Manifestation:** Validation error `ERR.EMPTY_EXPRESSION` (algebraic/state-with-output), `ERR.EMPTY_STATEINIT` (state node stateInit), or `ERR.EMPTY_STATETRANSITION` (state node stateTrans).

**Root cause:** `ValueNode.checkDefinition()` calls `STTools.isEmpty()` on each required field. `isEmpty()` returns true for null, blank strings, or the literal string `"null"`.

**Why it happens:** Node was created but its expression was never filled in. Or an XML edit accidentally deleted the content.

**Fix:** Provide a valid STEL expression for every required field based on `valueType`.

---

### 1.6 Referencing a Non-Existent Function

**Manifestation:** Parse error during expression parsing. The parser treats unknown identifiers as variable names, so the actual error is typically `ERR.VAR_NOT_FOUND` or a confusing parse failure depending on context.

**Root cause:** Function names are case-sensitive. `STInterpreter` registers functions by exact name. `Cos` is not `cos`. User-defined functions from `.stf` files must be loaded at startup.

**Why it happens:** Typo in function name, wrong case, or the `.stf` library containing the function is not in `fun_lib/`.

**Fix:** Check the exact function name against `functions_en.properties` (built-in) or the `.stf` files (user-defined). Function names are lowercase by convention.

---

## 2. Topology Errors

### 2.1 Circular Dependency Among Algebraic Nodes

**Manifestation:** `ERR.WRONG_TOPOLOGY` with `ERR.WRONG_TOPOLOGY2` listing the node names forming the cycle.

**Root cause:** `STGraphExec.setupProperties()` runs iterative priority relaxation. If after convergence, `notAssignedCount > 0`, the remaining nodes have mutual dependencies that cannot be resolved. This is detected when the algorithm makes no progress in an iteration (`assignedCount == 0 && notAssignedCount > 0`).

**Why it happens:** Node A's expression references node B, and node B's expression references node A (directly or transitively). This is valid for state nodes (via `this`) but not for algebraic nodes.

**Fix:** Break the cycle by converting one of the nodes to a state node. The state node uses `this` to reference its own previous value, breaking the algebraic dependency.

---

### 2.2 Missing Edge for Referenced Node

**Manifestation:** Validation error `ERR.NON_CONNECTED_NODE`. At runtime (if validation is bypassed): the referenced variable resolves to whatever residual value is in the interpreter's symbol table -- often `0.0` or a stale value from a previous step.

**Root cause:** `ValueNode.checkVariable()` checks that every non-system, non-global, non-private variable referenced in an expression has a corresponding entry in `getDefiningNodesByEdges()` (the `connectedFrom2` list built from graph edges).

**Why it happens:** Expression was edited to reference a new node but the edge was never drawn. Or a node was renamed but the edge still points to the old name.

**Fix:** Add an `<edge source="referenced_node" target="this_node">` in the XML, or draw the arrow in the STGraph GUI. If the node was renamed, update the edge's `source` or `target` attribute.

---

### 2.3 Unused Edge (Edge Exists But Not Referenced)

**Manifestation:** Not a runtime error. Flagged by `STGraphChecker.checkConnections()`: the edge exists from node A to node B, but node B's expression does not reference node A.

**Root cause:** The check compares `getDefiningNodesByEdges()` (edge-connected nodes) against `getDefiningNodesByExpressions()` (expression-referenced nodes). Edges not backed by expression references are flagged.

**Why it happens:** Expression was edited to remove a reference but the edge was left in place. Or the edge was drawn speculatively during model design.

**Fix:** Remove the unnecessary edge from the XML or the GUI. Unused edges add visual clutter and may confuse future analysis.

---

### 2.4 Dead-End Node (No Outgoing Edges, Not Output)

**Manifestation:** Not a runtime error. Flagged by `STGraphChecker.checkTerminals()`.

**Root cause:** A ValueNode with `getDefinedNodes() == null || size == 0` and `isOut=false` computes a value that nothing else reads.

**Why it happens:** The node was part of an earlier design and is now orphaned. Or the modeler forgot to mark it as output or connect it to a downstream node.

**Fix:** Either mark it `isOut=true` if the value should be observed, connect it to a downstream node, or remove it.

---

## 3. Submodel Errors

### 3.1 Wrong `superExpression` Positional Binding

**Manifestation:** Submodel inputs receive wrong values. The simulation runs without errors but produces wrong results. This is the most dangerous submodel error because it is silent.

**Root cause:** `superExpression0` binds to the first entry in the alphabetically-sorted `subVarNames` list, `superExpression1` to the second, etc. If the sort order is misunderstood, expressions are bound to the wrong inputs.

**Why it happens:** The modeler assumes `superExpression0` corresponds to the first input declared in the submodel, or the first one listed in `subVarNames`. But `subVarNames` is always alphabetically sorted regardless of declaration order.

**Example:** If `subVarNames="birth_rate,death_rate,food_req"`, then `superExpression0` goes to `birth_rate`, `superExpression1` to `death_rate`, `superExpression2` to `food_req`. If the modeler thinks the order is `food_req, birth_rate, death_rate`, every input gets the wrong value.

**Fix:** Always verify the alphabetical sort of `subVarNames` before assigning `superExpression` values. Use a systematic mapping: sort the names, then assign expressions in that order.

---

### 3.2 Missing `isGlobal=true` on Submodel Output

**Manifestation:** Parent model references `SubmodelName.outputNode` but gets a variable-not-found error or `0.0`.

**Root cause:** `checkVariable()` resolves dot-notation references by looking up the ModelNode, getting its submodel desk, and searching for the output node. If the output node exists but is not marked `isGlobal=true`, it is not accessible from the parent.

**Fix:** In the submodel `.stg` file, set `<isGlobal>true</isGlobal>` on the node that needs to be accessed from the parent.

---

### 3.3 Submodel File Not Found

**Manifestation:** Model fails to load or the submodel area appears empty. NanoXML parser may throw an exception during `STGraphImpl` load.

**Root cause:** `ModelNode.MODELPATH` defaults to `./mod_lib`. Submodel files are searched in the same directory as the parent model first, then in `mod_lib/`.

**Fix:** Place the submodel `.stg` file in the same directory as the parent model, or in the `mod_lib/` directory.

---

### 3.4 Adding Submodel Input Without Updating All Parents

**Manifestation:** When a new input node is added to a submodel, the `subVarNames` list changes (because it is alphabetically sorted), which shifts all positional `superExpression` bindings. Every parent model using this submodel now has wrong bindings.

**Example:** Adding input `alpha` to a submodel with inputs `birth_rate, death_rate` changes the sort order to `alpha, birth_rate, death_rate`. Now `superExpression0` (which was `birth_rate`'s value) goes to `alpha`, and all subsequent indices shift by one.

**Fix:** After adding or removing a submodel input, update `subVarNames` and ALL `superExpressionN` indices in EVERY parent model that uses this submodel.

---

## 4. Numerical Errors

### 4.1 Division by Zero

**Manifestation:** If `exceptionHandling=1` (halt): `ERR.COMP.INPUTS` or `ERR.COMP.ALGEBRAICS` error. If `exceptionHandling=0` (continue): division produces `Infinity` or `NaN`, which propagates silently through subsequent computations.

**Root cause:** `OpQuotient.run()` performs standard IEEE 754 division. `1.0/0.0` = `Infinity`, `0.0/0.0` = `NaN`. Flagged as risky by `STGraphChecker.checkIssuesInDefs()`.

**Fix:** Guard with `if(denom!=0, num/denom, fallback)`. Choose a fallback that is physically meaningful (0, a large number, the previous value via `this`).

---

### 4.2 NaN Propagation

**Manifestation:** One or more output nodes show `NaN`. Charts display gaps or flat lines. The simulation may complete without errors if `exceptionHandling=0`.

**Root cause:** Any operation involving `NaN` produces `NaN` (IEEE 754 semantics). A single `NaN` in one node propagates to all downstream nodes via expression evaluation.

**Why it happens:** Common sources: `0.0/0.0`, `sqrt(-1)`, `log(0)`, `log(-1)`, `Infinity - Infinity`, `0 * Infinity`.

**Fix:** Find the first node that produces `NaN` (use stepped execution, key `3`, and inspect node values at each step). Guard the operation. Use `isNumber(x)` to check for `NaN`/`Infinity` in expressions: `if(isNumber(x), x, fallback)`.

---

### 4.3 State Variable Blowup (Numerical Instability)

**Manifestation:** State variables grow without bound, reaching `Infinity` within a few steps. Charts show exponential curves shooting off the axis.

**Root cause:** Euler integration with `timeD` too large for the dynamics. Forward Euler is conditionally stable: for a linear system `x' = -k*x`, stability requires `timeD < 2/k`. If `timeD` is larger, the state oscillates with growing amplitude.

**Why it happens:** `timeD` was chosen for convenience (e.g., 1.0) without checking stability constraints. Or the model has fast dynamics (large rate constants) that require small time steps.

**Fix:** Reduce `timeD`. Or switch to RK2/RK23 (`integrationMethod=1` or `2`), which have larger stability regions. Or restructure the model to use smaller effective rate constants.

---

### 4.4 Off-by-One from `indexOrigin`

**Manifestation:** Array indexing produces unexpected results. `v[1]` returns the first element in some models and the second element in others.

**Root cause:** `indexOrigin` in the model header sets whether arrays are 0-indexed or 1-indexed. `[1:5]` with `indexOrigin=0` produces `[1,2,3,4,5]` (the literal values 1 through 5), but `v[0]` accesses the first element. With `indexOrigin=1`, `v[1]` accesses the first element.

**Fix:** Check `indexOrigin` in the `<head>` element. Be consistent. Most STGraph models use `indexOrigin=0`.

---

### 4.5 Empty Vector Edge Case

**Manifestation:** Unexpected `0.0` results. `+/[]` returns `0.0` silently (by polymorphism rules: empty non-scalar operand -> `0.0`). But `max/([])` may throw an error. `[][0]` throws an array index exception.

**Root cause:** `Tensor` with `order=1, dimensions=[0], value=null` is a valid empty vector. Most operations handle it via the polymorphism fallback, but some functions (especially user-defined ones) may not check for empty input.

**Fix:** Guard operations on potentially empty vectors: `if(@v>0, operation(v), fallback)`.

---

## 5. XML and File Errors

### 5.1 XML Element Order Violation

**Manifestation:** Model fails to load. May produce a NullPointerException or ClassCastException during parsing.

**Root cause:** `STGraphImpl` reads XML children sequentially with `theEnum.nextElement()`. It expects: head, nodes, texts, edges, widgets. If elements are reordered, the parser casts the wrong element to the wrong type.

**Fix:** Ensure element order is: `<head>`, `<nodes>`, `<texts>`, `<edges>`, `<widgets>`, `<groups>`, `<reports>`.

---

### 5.2 Formatted Expression Out of Sync

**Manifestation:** The STGraph GUI shows a different expression than what actually executes. The `expression` (raw) is correct but the `fExpression` (formatted) shows the old version.

**Root cause:** The engine evaluates `expression`, not `fExpression`. The formatted version is for GUI display only. If the raw expression is edited in XML without updating the formatted version, they diverge.

**Fix:** When editing expressions in XML, either update both `expression` and `fExpression`, or delete `fExpression` entirely. STGraph regenerates it on the next load/save.

---

### 5.3 XML Entity Escaping in Expressions

**Manifestation:** Expression containing `<`, `>`, or `&` causes XML parse failure or wrong expression evaluation.

**Root cause:** These characters have special meaning in XML. Inside element content, `<` must be `&lt;`, `>` must be `&gt;`, `&` must be `&amp;`.

**Why it happens:** Modeler edits the raw XML and writes `if(x<0, -x, x)` instead of `if(x&lt;0, -x, x)`.

**Fix:** Always escape XML special characters in expression element content. Alternatively, wrap expression content in `<![CDATA[...]]>` (though STGraph's own save does not use CDATA).

---

### 5.4 Node Name Collision After Rename

**Manifestation:** After renaming a node in the XML, expressions referencing the old name fail with `ERR.VAR_NOT_FOUND`. Edges using the old name produce orphaned connections.

**Root cause:** Node names are used as identifiers everywhere: in expressions, edge `source`/`target` attributes, widget `ysourcena`/`sourcena`/`source`, `superExpression` values in parent models, and `SubmodelName.nodeName` references.

**Fix:** When renaming a node, search-and-replace the old name in ALL of:
1. Every `expression`, `stateInit`, `stateTrans` in the same model
2. Every `<edge source="...">` and `<edge target="...">`
3. Every widget `ysourcena`, `xsourcena`, `sourcena`, `source` element
4. Every `superExpression` in parent models if this is a submodel output/input
5. Every `SubmodelName.oldName` reference in parent models

---

## 6. Widget and Display Errors

### 6.1 Chart Shows Only Last Value (Missing `isVectorOut`)

**Manifestation:** Chart widget connected to a node that produces vector values only shows the final value, not a time series.

**Root cause:** In `handleOutput()`, if `isVectorOutput() == false` and the node's value is a vector (order >= 1), only the current value is stored -- no time series accumulation occurs. The chart receives a single vector instead of a matrix of vectors over time.

**Fix:** Set `<isVectorOut>true</isVectorOut>` on the node.

---

### 6.2 Chart Shows Nothing (Missing `isOut`)

**Manifestation:** Chart widget references a node but displays empty.

**Root cause:** `handleOutput()` only processes nodes in `outputNodeList`. A node is in this list only if `isOut=true`. If not marked as output, no value history is accumulated, and the chart has no data to display.

**Fix:** Set `<isOut>true</isOut>` on the node.

---

### 6.3 Slider/Knob Has No Effect

**Manifestation:** Moving a slider or knob widget doesn't change simulation results.

**Root cause:** The widget's `source` must reference an input node (`isIn=true`). The widget writes to the node's value via `((InputWidget)node.getInputWidget()).getNextValue()`. If the node is not marked as input, or the widget `source` doesn't match the node name, the binding doesn't exist.

**Fix:** Ensure the widget `source` matches an existing node name AND that node has `<isIn>true</isIn>`.

---

## 7. Validation Errors Summary Table

| Error Code | Source | Meaning |
|---|---|---|
| `ERR.BLANK_NODE_NAME` | `STNode.checkName()` | Empty node name |
| `ERR.FORBIDDEN_NODE_NAME` | `STNode.checkName()` | Invalid characters or starts with digit |
| `ERR.RESERVED_NAME` | `STNode.checkName()` | Conflicts with system variable or function |
| `ERR.DUPLICATED_NODE_NAME` | `STNode.checkName()` | Name already used by another node |
| `ERR.EMPTY_EXPRESSION` | `ValueNode.checkDefinition()` | Missing expression on algebraic/state-with-output node |
| `ERR.EMPTY_STATEINIT` | `ValueNode.checkDefinition()` | Missing stateInit on state node |
| `ERR.EMPTY_STATETRANSITION` | `ValueNode.checkDefinition()` | Missing stateTrans on state node |
| `ERR.THIS_NOT_ALLOWED` | `ValueNode.checkVariable()` | `this` used outside valid context |
| `ERR.ME_NOT_ALLOWED` | `ValueNode.checkVariable()` | `me` used outside state-with-output expression |
| `ERR.NON_CONNECTED_NODE` | `ValueNode.checkVariable()` | Referenced variable has no incoming edge |
| `ERR.VAR_NOT_FOUND` | `STNode.checkExpressionDefinition()` | Blank variable in parse tree |
| `ERR.WRONG_TOPOLOGY` | `STGraphExec.setupProperties()` | Circular dependency among algebraic nodes |
| `ERR.COMP.NULLVALUE` | `STGraphExec.compute*()` | Expression evaluated to null |
| `ERR.COMP.NULLVALUE.FIRSTCALL` | `STGraphExec.computeInputs()` | Expression evaluated to null at step 0 |
| `ERR.COMP.INPUTS` | `STGraphExec.computeInputs()` | Uncaught exception during input evaluation |
| `ERR.COMP.STATES` | `STGraphExec.computeStateValues()` | Uncaught exception during state evaluation |
| `ERR.COMP.ALGEBRAICS` | `STGraphExec.computeSync()` | Uncaught exception during sync evaluation |
| `ERR.COMP.NEXTSTATES` | `STGraphExec.computeNextStates()` | Uncaught exception during state transition |
| `ERR.COMP.HISTORY` | `STGraphExec.compute()` | `handleOutput()` failed (null history vector) |
| `ERR.FUN.GENERIC` | `STInterpreter.preParseExpression()` | Unclassified throwable during parsing |
| `ERR.FUN.EMPTY_STACK` | `STFunction.checkStack()` | Function received empty operand stack |
