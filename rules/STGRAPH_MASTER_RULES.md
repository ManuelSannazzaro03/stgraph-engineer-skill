# STGraph Master Rules

Authoritative reference for STGraph modeling principles, derived from source code analysis of the STGraph engine (`STGraphExec.java`, `STInterpreter.java`, `ValueNode.java`, `STNode.java`, `Parser.jjt`, `Tensor.java`).

---

## 1. Modeling Principles

### 1.1 What a Model Is

An STGraph model is a directed graph of nodes connected by edges. Each node holds one or more STEL expressions. The graph topology determines the evaluation order. The simulation engine advances discrete time steps, evaluating all nodes in dependency order at each step.

### 1.2 The Three Fundamental Laws

**Law 1: Everything is a Tensor.**
There is no integer type, no boolean type, no string type in STEL expressions. Every value is a `Tensor` -- a generalized n-dimensional array of `double`. Scalars are order-0 tensors. Vectors are order-1. Matrices are order-2. The `Tensor` class (`org.nfunk.jep.type.Tensor`) stores values in a flat `double[]` array in row-major order.

**Law 2: Data flows through edges only.**
A node can only read the value of another node if a directed edge exists from that node to the reading node. The only exceptions are: system variables (`time`, `index`, etc.), constants (`pi`, `e`), private variables (`$0`, `$1`, `$a0`..`$a3`, etc.), global nodes (`isGlobal=true`), and submodel outputs accessed via dot notation (`SubmodelName.nodeName`). Referencing a node without an edge will either fail validation (`ERR.NON_CONNECTED_NODE`) or produce undefined behavior.

**Law 3: Evaluation order is determined by topology, not by text.**
The engine performs a topological sort on the dependency graph. Nodes with no dependencies are evaluated first; nodes that depend on others are evaluated after their dependencies. Circular dependencies among algebraic nodes are detected and reported as `ERR.WRONG_TOPOLOGY`. The sort algorithm is an iterative priority-relaxation scheme: each node's priority = max(defining node priorities) + 1.

### 1.3 The Simulation Cycle

Each simulation step executes four phases in strict order:

```
Phase 1: computeInputs()      -- evaluate algebraic nodes NOT depending on states (sortedNodeList1)
Phase 2: computeStateValues()  -- read current state of state nodes (stateInit on step 0, nextState after)
Phase 3: computeSync()         -- evaluate algebraic nodes that DO depend on states (sortedNodeList2)
Phase 4: computeNextStates()   -- compute next state via state transition expressions
```

Within each phase, nodes are evaluated in topological priority order. After all four phases, `handleOutput()` accumulates value histories for output nodes.

### 1.4 Execution Modes

| Mode | Method | Behavior |
|---|---|---|
| Immediate | `exec()` | Runs all steps in a blocking loop |
| Stepped | `steppedExec()` | Runs `stepsBeforePause` steps, then pauses |
| Timed | `timedExec()` | Runs steps at `simulationDelay` ms intervals via `javax.swing.Timer` |
| Single | `singleExec()` | Runs exactly one step |
| Batch | `batchExec()` | Runs the full simulation `batchSteps` times, incrementing `__batch` |

### 1.5 Exception Handling Modes

Controlled by `exceptionHandling` in the model header:

- **Mode 0 (Continue):** `continueWithExceptions()` returns true. Any null evaluation result is silently replaced with `Tensor.ZERO` (scalar `0.0`). Errors are swallowed. This masks bugs.
- **Mode 1 (Halt):** Any null evaluation result triggers `errorHandler()`, which displays the error and stops execution by returning `false` from `compute()`.

---

## 2. Node Types

### 2.1 ValueNode -- Algebraic (valueType=0)

**Fields used:** `expression`

**Semantics:** `y(t) = f(u(t))`

The node computes a pure function of its inputs at each time step. It has no memory. The expression is evaluated once per step, after all nodes it depends on have been evaluated.

**Evaluation:** The expression is pre-parsed into a tree (`topOfTreeValue`) during `preCompute()`. At each step, `interpreter.evalExpression(node, topOfTreeValue)` evaluates it. The result is stored on the node and in the interpreter's symbol table.

**Input detection:** If an algebraic node has zero incoming edges AND references no other nodes in its expression, `setInput()` automatically marks it as an input node. Nodes explicitly marked `isIn=true` can be controlled by input widgets.

### 2.2 ValueNode -- State (valueType=1)

**Fields used:** `stateInit`, `stateTrans`

**Semantics:** `x(0) = init; x(t+dt) = g(this, u(t)); y(t) = x(t)`

The node has internal state. At step 0, `stateInit` is evaluated to set the initial state. At subsequent steps, `stateTrans` is evaluated with `this` bound to the current state value. The output is the state itself.

**Evaluation sequence:**
1. Step 0 (`firstCall=true`): `computeStateValues()` evaluates `stateInit` via `topOfTreeValue` (which equals `topOfTreeState` for plain state nodes). `computeNextStates()` re-parses `stateTrans` into `topOfTreeState`, sets `this` to current state, evaluates the transition.
2. Step >0: `computeStateValues()` reads `node.getNextState()` (computed in the previous step's `computeNextStates()`). `computeNextStates()` sets `this` to current state and evaluates `topOfTreeState`.

**Critical rule:** `this` is only valid inside `stateTrans`. Using `this` in `stateInit` creates a circular reference at t=0 (there is no prior state). Using `this` in an algebraic node is a validation error (`ERR.THIS_NOT_ALLOWED`) but may slip past if validation is not run.

### 2.3 ValueNode -- State with Output (valueType=2)

**Fields used:** `stateInit`, `stateTrans`, `expression`

**Semantics:** `x(0) = init; x(t+dt) = g(this, u(t)); y(t) = h(x(t), u(t))`

Same as state node, but the output is computed from a separate expression that can reference both the state (via `this` or `me`) and other inputs. This allows the output to differ from the raw state.

**Evaluation:** State-with-output nodes are placed in either `sortedNodeList1` or `sortedNodeList2` based on whether they depend on other state nodes. The `expression` is evaluated in `computeInputs()` or `computeSync()`. The state transition is evaluated in `computeNextStates()`.

**The `me` variable:** Only valid in the `expression` field of a state-with-output node. It refers to the result of evaluating the state transition expression. Using `me` elsewhere produces `ERR.ME_NOT_ALLOWED`.

### 2.4 ModelNode (type="ModelNode")

**Semantics:** Embeds a submodel (another `.stg` file) as a nested `STGraphExec`.

**Binding mechanism:** The parent model provides values to the submodel's input nodes via `superExpression0`..`superExpressionN`. These are matched positionally to the alphabetically-sorted `subVarNames` list. The binding replaces the submodel input node's expression with the parent's expression (the "saturating expression" mechanism in `getSaturatingExpression()`).

**Output access:** Nodes in the submodel marked `isGlobal=true` can be read from the parent as `SubmodelName.nodeName`. This is validated by `checkVariable()` which resolves dot-notation references.

**Execution:** The submodel inherits the parent's time basis (`time0`, `time1`, `timeD`, `integrationMethod`). Its `initTimeBasis()` copies these from the top graph.

### 2.5 TextNode

Display-only. No computation. Names always start with `*`. Not involved in the simulation engine.

---

## 3. Time Handling

### 3.1 System Time Variables

| Variable | Set by | Meaning |
|---|---|---|
| `time0` | Model header | Simulation start time |
| `time1` | Model header (or computed: `time0 + (numSteps-1) * timeD`) | Simulation end time |
| `timeD` | Model header | Time step size |
| `time` | Engine (each step) | Current simulation time: `time0 + (currStep - 1) * timeD` |
| `index` | Engine (each step) | Current step number (integer, 0-based) |
| `numSteps` | Computed from header | Total number of steps |
| `vTime` | Engine (once at start) | Vector of all time values for the simulation |
| `vIndex` | Engine (once at start) | Vector of all index values |

These are injected into the interpreter via `interpreter.addVariable()` in `initTimeBasis()`. They are read-only from the expression perspective -- expressions cannot assign to them.

### 3.2 The `this` Variable

`this` is a special interpreter variable that holds the current state of a state node. It is set per-node in `computeNextStates()` before evaluating the state transition expression:

```java
interpreter.setVarValue("this", node.getCurrentState());
```

**Scope rules (enforced by `ValueNode.checkVariable()`):**
- `this` is allowed in `stateTrans` (the state transition expression)
- `this` is allowed in `expression` of a state node (state-with-output type)
- `this` is NOT allowed in `stateInit`
- `this` is NOT allowed in algebraic node expressions
- Violation produces `ERR.THIS_NOT_ALLOWED`

### 3.3 The `integral()` Function

`integral(x)` is syntactic sugar for `this + x * timeD` (Euler forward integration). It is ONLY meaningful inside a state transition expression, because it references `this`. Using it in an algebraic node will produce wrong results or errors because `this` has no defined value in that context.

With higher-order integration methods (RK2, RK23), the engine automatically handles the multi-phase evaluation: the `integral()` function's behavior doesn't change, but `compute()` calls the four-phase pipeline multiple times with modified time deltas and saves/restores state between phases.

### 3.4 Time Frame Modes

| `timeFrame` | Name | Behavior |
|---|---|---|
| 0 | Standard | Full simulation from `time0` to `time1` |
| 1 | Instantaneous | Single-step computation (no time progression) |
| 2 | Windowed | Rolling window of `maxSteps` steps; older values are discarded |
| 3 | Play mode | Like windowed but `maxSteps` forced to 1; interactive real-time |

### 3.5 Integration Methods

| `integrationMethod` | Name | Phases per step | Description |
|---|---|---|---|
| 0 | Euler | 1 | Forward Euler: single pass through the four-phase pipeline |
| 1 | RK2 | 3 | Runge-Kutta 2: saves state, evaluates at `0.75*timeD`, restores, evaluates at `timeD`, restores, final evaluation |
| 2 | RK23 | 5 | Runge-Kutta 2-3: five phases with `0.5*timeD` and `0.75*timeD` intermediate steps |

On the first step (`firstCall=true`), all methods reduce to a single Euler pass.

---

## 4. Dependency Graph Logic

### 4.1 How Dependencies Are Tracked

Three separate dependency structures exist per node:

1. **`connectedTo` (defined nodes):** outgoing -- nodes that THIS node's value feeds into. Built by expression parsing.
2. **`connectedFrom` (defining nodes by expression):** incoming -- nodes whose names appear in THIS node's expressions. Built by parsing expressions and extracting variable names via `ParserCheckVisitor`.
3. **`connectedFrom2` (defining nodes by edges):** incoming -- nodes connected by graph edges. Built from the edge topology via `STTools.connect2wrapper()`.

**The critical invariant:** for a model to be valid, every non-system, non-global variable referenced in an expression (`connectedFrom`) must have a corresponding edge (`connectedFrom2`). `checkVariable()` enforces this by checking `getDefiningNodesByEdges()`.

### 4.2 Topological Sort Algorithm

Located in `STGraphExec.setupProperties()`:

1. **Phase 1 -- Priority initialization:** Nodes with no defining nodes (or only system variable dependencies) get `executionPriority = 0`. All others start at `-1` (unassigned).

2. **Phase 2 -- Iterative relaxation:** Repeat until convergence:
   - For each unassigned node: if ALL defining nodes have assigned priorities, set this node's priority = `max(defining priorities) + 1`.
   - If any defining node is still unassigned, defer.
   - Track `assignedCount` and `notAssignedCount` per iteration.

3. **Phase 3 -- Cycle detection:** If `notAssignedCount > 0` after convergence, the remaining nodes form one or more cycles. Error: `ERR.WRONG_TOPOLOGY` with node names listed in `ERR.WRONG_TOPOLOGY2`.

4. **Phase 4 -- List partitioning:** After sorting by priority, nodes are split based on the `fromState` flag:
   - `sortedNodeList1`: algebraic nodes and state-with-output nodes NOT depending on state nodes (computed in Phase 1)
   - `sortedNodeList2`: algebraic nodes and state-with-output nodes depending on state nodes (computed in Phase 3)

### 4.3 The `fromState` Flag

After topological sorting, a second iterative pass propagates the `fromState` flag: a node is marked `fromState=true` if it depends on a plain state node (type 1), a sequential model node, or another node already marked `fromState`. This determines whether the node goes into `sortedNodeList1` (pre-state) or `sortedNodeList2` (post-state).

### 4.4 Submodel Edge Handling

`STTools.connect2wrapper()` handles four cases:

- **ValueNode -> ValueNode:** direct edge connection.
- **ModelNode -> ValueNode:** connects each of the submodel's output nodes to the target.
- **ValueNode -> ModelNode:** connects source to the model node AND to each of the submodel's input nodes.
- **ModelNode -> ModelNode:** connects both model nodes AND cross-connects all outputs of source's submodel to all inputs of target's submodel.

---

## 5. STEL Syntax Rules

### 5.1 Expression Fundamentals

STEL is a pure functional expression language. Every expression evaluates to a Tensor. There are no statements, no assignments (except through the `{...}` subexpression mechanism and the `$v0`..`$vN` locals in custom functions), no control flow beyond `if()`.

### 5.2 Operator Precedence (from `Parser.jjt`)

From lowest to highest binding:

| Level | Operators | Associativity |
|---|---|---|
| 1 | `\|\|` | Left |
| 2 | `&&` | Left |
| 3 | `==` `!=` | Left |
| 4 | `<` `>` `<=` `>=` | Left |
| 5 | `+` `-` | Left |
| 6 | `*` `/` `%` `#` `##` | Left |
| 7 | `!` unary`+` unary`-` `@` meta-operators | Right (prefix) |
| 8 | `^` | Right |

### 5.3 Literals

| Syntax | Result |
|---|---|
| `3.14` | Scalar (order-0 Tensor) |
| `[1,2,3]` | Vector of 3 elements (order-1 Tensor) |
| `[[1,2],[3,4]]` | 2x2 Matrix (order-2 Tensor) |
| `[]` | Empty vector (order-1, dimension=[0], value=null) |
| `[1:5]` | Vector `[1,2,3,4,5]` |
| `[0:1:0.25]` | Vector `[0,0.25,0.5,0.75,1]` |

### 5.4 Indexing

`x[v0,v1,...]` is syntactic sugar for `get(x,v0,v1,...)`. Each `v` can be:
- A scalar: selects a single element/row/column
- A vector: selects multiple elements
- `[]` (empty vector): selects all elements along that dimension

Examples: `m[0,1]` (scalar from matrix), `m[[0],[1,2]]` (submatrix), `v[0]` (first element of vector).

### 5.5 Meta-Operators

Defined in the parser grammar and wired in `STInterpreter.java`. They apply a dyadic function/operator across an array:

- **Reduction `f/x`:** Folds left. `+/[1,2,3,4]` = `((1+2)+3)+4` = `10`. Reduces dimensionality by 1.
- **Scan `f\x`:** Cumulative fold. `+\[1,2,3]` = `[1,3,6]`. Same dimensionality.
- **PairScan `f|x`:** Pairwise. `+|[1,2,3,4]` = `[3,5,7]`. Dimensionality minus 1 element.
- **Dimensional variant:** `f/[n]x` operates along dimension `n`.

Valid `f` values: any binary operator (`+`, `-`, `*`, `/`, `%`, `^`, `&&`, `||`, `<`, `>`, `==`, `<=`, `>=`, `!=`) or a named function in parentheses (e.g., `max/(v)`, `min/(v)`).

### 5.6 The `if()` Function

Generalized conditional. Accepts any odd number of arguments >= 3:

```
if(c1, v1, default)                   -- simple if-else
if(c1, v1, c2, v2, default)           -- if-elseif-else
if(c1, v1, c2, v2, c3, v3, default)   -- if-elseif-elseif-else
```

Conditions are evaluated left to right. The first condition > 0 selects the corresponding value. If no condition is true, the last argument (default) is returned.

### 5.7 The `iter()` Function

`iter(x, e, z)` -- iterative fold over array `x`:
- `$0` = accumulator, initialized to `z`
- `$1` = current element of `x`
- `$i` = current index
- `e` is evaluated for each element; its result becomes the new `$0`
- Returns the final `$0`

Example: `iter([1,2,3,4], $0+$1, 0)` = `10` (equivalent to `+/[1,2,3,4]`)

### 5.8 The `array()` Function

`array(v, e)` -- construct an array of dimensions `v` using expression `e`:
- `v` is a vector of dimension sizes (e.g., `[3,4]` for a 3x4 matrix)
- `$i0`, `$i1`, ... are the current indices along each dimension
- `$p0`, `$p1`, `$p2` are previously computed elements (for recurrence relations)
- `e` is evaluated for each element

Example: `array([5], $i0^2)` = `[0,1,4,9,16]`

### 5.9 The `function()` Mechanism

A node named with `_` prefix (e.g., `_factorial`) can define a reusable function via:
```
function(if($a0==0, 1, $a0 * _factorial($a0-1)))
```

Callable from any other node as `_factorial(5)`. Arguments are passed as `$a0`..`$a3`. `$numArgs` gives the count. Supports recursion.

### 5.10 The `{...}` Subexpression Mechanism

`{expr}` evaluates `expr`, stores the result in `$w0` (first occurrence), `$w1` (second), etc., and returns the value transparently. This allows reusing intermediate results:

```
{a+b} * $w0       -- computes (a+b)^2 without evaluating a+b twice
```

### 5.11 Polymorphism Rules

All operators and functions follow these rules automatically:

**Monadic (one-argument):**
- Scalar -> scalar
- Array -> element-wise array of same shape

**Dyadic (two-argument):**
- Scalar + scalar -> scalar
- Scalar + array -> broadcast (scalar applied element-wise)
- Same-shape arrays -> element-wise
- Order-n + order-(n-1) with matching first (n-1) dimensions -> broadcast along last dimension
- Either operand is empty non-scalar -> result is `0.0`
- Incompatible dimensions -> exception

### 5.12 Boolean Encoding

There is no boolean type. Comparisons and logical operators produce `1.0` (true) or `0.0` (false). Conditions in `if()` and `&&`/`||` interpret values as: true if > 0 (specifically, >= machine epsilon), false if <= 0.

### 5.13 Node Name Rules

Enforced by `STNode.checkName()`:

- Must not be empty
- Must not start with a digit
- Must not contain: space, `.`, `?`, `!`, `%`, `&`, `+`, `-`, `*`, `/`, `#`, `,`, `(`, `)`, `[`, `]`, `{`, `}`, `'`, `"`
- Must not contain characters with code > 127 (non-ASCII)
- Must not match any system variable name (`time`, `index`, `pi`, `e`, `this`, `me`, etc.)
- Must not match any registered function name
- Must not duplicate another node name in the same graph
