# Addendum 219 (part 12) — Expression (f(x)) and Function (JavaScript) nodes, and the sandbox that contains them

**Date:** 2026-11-10
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12; §0 maps every part).
- Part 4 `…-part-4-workflow-definition-and-editor.md` (§20–§26: node catalogue, expressions, validation, DDL, interpreter, editor, API).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34, RC-01…RC-19).
- Part 11 `…-part-11-variable-scope-and-data-flow.md` (§41: scope, bindings, data flow).
- **This part (§42, D-49…D-61, V-57…V-69, ST-35, ST-36, S14, RC-20).**
- Addendum 220 Prompts 23, 26, 27, 29 and the new Prompt 31 show the screens.

**Why this part exists.** On 2026-11-10 the user asked for an f(x) function node in the workflow
engine, "similar to GovTech's Opus / Workflow app", that evaluates variable expressions, with
research on the language Opus uses and an assessment of what stops a user from jailbreaking out of
the node. The user then chose to ship **both** of Opus's node types in the MVP: an expression node
and a JavaScript function node.

**Precedence.** This part is normative for the `expression` and `function` node kinds, the FX
language, the function sandbox, node data flow and the `FOUR_EYES_PUBLISH` setting. Where part 4 §21.4, §22, §23 or §24
differ, this part wins; §42.17 lists every superseded clause.

---

## §42 Expression and Function nodes

### §42.0 The short answer

> **Two new node kinds, two very different trust levels.**
> - **`expression` ("f(x)")** evaluates up to 16 named lines of **FX**, a small ZEN-compatible
>   expression language. FX is parsed and type-checked at validate time into a closed AST and run by
>   the pure engine. It has no loops, no eval, no host calls and no dynamic member access. **There
>   is nothing to escape into**; the residual risks are parser denial of service and arithmetic
>   faults, both bounded statically.
> - **`function` ("JS")** runs an author's JavaScript `handler(input)`, as Opus does. It is
>   **arbitrary code**, so it never runs in the workflow, the API or the worker. A Temporal activity
>   sends it to **`mvp-fn-runner`**, a separate process with no network, no credentials and no
>   files. That process runs QuickJS **compiled to WebAssembly** inside `wasmtime`, in a fresh
>   instance per call, with deterministic fuel, a memory cap and declared typed inputs and outputs
>   only. Any definition editor may write function code. Every change is audited, and one app-wide
>   setting, `FOUR_EYES_PUBLISH`, can require a second person to publish it (D-55).
> - Both nodes have the ports `next` and `error`. A runtime fault takes `error`; it never corrupts
>   state and never needs an admin reset. The node may store the fault's class and message in two
>   variables of the author's choice, so a later `switch` can route by cause (D-59).
> - **Data flow.** FX reads any name in scope; a function reads only its declared inputs. Both read
>   one snapshot taken when the node is entered. Results are ordinary variables, written all or
>   nothing. To branch on a result, add a `rule` (ports `true`/`false` with author-defined labels) or
>   a `switch` after `next` (D-58, D-59, D-60).

### §42.1 Research: what Opus and Plumber do

Read 2026-11-10. `opus.gov.sg` redirects (307) to `https://workflow.tech.gov.sg/`, GovTech's
visual workflow builder for public officers. The node pages are under
`docs.developer.tech.gov.sg/docs/workflow-documentation/nodes/`.

| Opus node | What it does | Language and limits | Confidence |
|---|---|---|---|
| **Expression node** | One or more named expressions; returns values. Inputs by bare name (`household_income <= 9000`); earlier results of the same node by `$.key` (`$.incomeOk and $.citizenOk`); a loop mode applies the expressions to each item of a list | "Expression Syntax", with pages for String, Number, Boolean, Date and Time, Array and Context | High for the behaviour. That the language is **ZEN** is inferred (medium-high): the Switch node page names the ZEN Expression Language and uses the same syntax |
| **Switch node** | Routes by conditions | ZEN: `(input.status == "Approve" or …) and input.amount > 1000` | High |
| **Decision node** | References a decision table; nests up to 5 levels | GoRules-style decision tables (inferred) | Medium |
| **Function node** | `export const handler = async (input) => { …; return {…} }` | JavaScript ES2020 on **QuickJS through the `rquickjs` Rust bindings**; `dayJS` and `bigJS` preloaded; module imports and external API calls blocked; **5,000 ms** execution limit; no documented memory limit; `console.log` only in simulation; optional AI coding assistant | High |

**ZEN** (`docs.gorules.io/learn/zen-language/syntax`) has `+ - * / % ^`, `and`/`or`/`not`, `??`,
the ternary `c ? a : b`, ranges (`x in [1..5]`), assignment with `;` between statements, and
closures `map`, `filter`, `some`, `all` with `#` for the current element. It has no loop construct.

**Plumber** (`github.com/opengovsg/plumber`, OGP) has **no expression language**:
- `{{step.<id>.<path>}}` tokens are resolved with `lodash.get` (`helpers/compute-parameters.ts`);
- a form-driven Calculator handles two operands on `big.js`;
- a Formatter handles dates on `luxon`.

It is a useful lower bound: no eval anywhere.

**Candidate engines assessed for our Rust stack** (crates.io and GitHub advisories, 2026-11-10):

| Candidate | Fits pure `mvp-engine`? | Main reason |
|---|---|---|
| `zen-expression` 2.1.6 (GoRules) | **No** | Its dependencies include `fastrand`, `ahash`, `iana-time-zone`, `chrono-tz`, `serde_json` dynamic values, `recursive` and `bumpalo`. That breaks the part 8 §30.3 purity rules and RC-08 (`BTreeMap` only), and dynamic typing defeats V-21 |
| `cel` 0.15 (CEL) | Partly | Non-Turing and well specified, but the syntax is unfamiliar to Opus users and it has no documented depth or cost limits |
| `evalexpr` 13 | Partly | No loops, but dynamically typed and has no limits |
| `rhai` 1.27 | No | Turing-complete; CVE-2024-36760 (stack overflow on deep nesting) |
| **Own FX parser in `mvp-engine::fx`** | **Yes (chosen, D-50)** | ZEN syntax, our static types (part 4 §21.4), `rust_decimal`, no new crates, an explicit depth counter |
| `vm2`, `isolated-vm` (Node) | No (for JS) | Repeated critical sandbox escapes (vm2: many 2023–2026 CVEs; isolated-vm: CVE-2022-39266, CVE-2021-21413) |
| `rquickjs` native in the worker | No (for JS) | QuickJS is C. A memory-safety bug would run in a process that holds DB and Temporal credentials |
| `boa_engine` 0.22 | No (for JS) | CVE-2024-43367; incomplete conformance |
| **QuickJS on WASM in `wasmtime`, sidecar process (chosen, D-53)** | n/a (separate binary) | Opus-compatible JavaScript. A QuickJS bug stays inside WASM linear memory, so an escape also needs a wasmtime bug, and then still lands in a process with no network or secrets |

No advisories were found for `cel`, `evalexpr`, `rquickjs`, `starlark` or `zen-expression`. Absence
of advisories is not evidence of safety.

### §42.2 Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-49 | **Two node kinds: `expression` and `function`.** Each has one in-connector and the out-ports `next` and `error`, both mandatory (V-05). They are allowed in roots and templates, on any path, but **not inside exception handlers** in the MVP (V-64). Neither projects a status | Opus parity for both Expression and Function. Handlers keep their fixed palette (part 6 §27) |
| D-50 | **FX is a ZEN-compatible subset with our own parser and static typing** (§42.4). It is parsed at save, type-checked against `scope_table` (D-44) at validate, and compiled to a typed AST in `CompiledPlan`. Text that also parses as ZEN means the same thing in both. The reverse does not hold | Familiar to Opus authors. Keeps `mvp-engine` pure and deterministic and keeps V-21 static typing |
| D-51 | **Division only in FX.** `/` always yields `decimal`; `%` is int only. Divide-by-zero, overflow, an out-of-range decimal or a text result over 200 characters is an `FxFault`. The node then leaves by `error` and **none** of its lines is applied (all or nothing) | Rule, switch and set_variable stay as part 4 §21.4 says. Faults become routable instead of a failed node |
| D-52 | **FX source is stored as text with resolved reference spans** (`wf_fx_lines`, `wf_fx_refs`). Each identifier span stores a `variable_id` or `platform_var` (D-44). Renaming a variable rewrites the spans in drafts in the same transaction. Published rows are immutable | No JSONB. Resolution by ID survives renames (AC-34.1.3) |
| D-53 | **The Function node runs as the Temporal activity `run_function`, never in workflow code.** The activity sends the source, the inputs and `now` to `mvp-fn-runner`, which runs QuickJS-on-WASM in `wasmtime` with a fresh instance per call (§42.8). The activity result is in history, so replay never re-executes code | Arbitrary code needs a hard boundary and must not break determinism |
| D-54 | **Function I/O contract.** At most 8 declared typed inputs, each bound to a variable or platform key in scope, or to a typed literal; at most 8 declared typed outputs, each written to a writable variable that is not a `ref`. Decimals and timestamps cross as strings; enums as their value string. The handler must return a plain object with exactly the declared keys (§42.7) | The node behaves like a typed pure function. Least data in, validated data out |
| D-55 | **No separate code-author role; four-eyes publish is one app-wide setting.** Any definition editor (D-38, D-40) may write function code, as for every other node. The environment variable `FOUR_EYES_PUBLISH` (read once by `mvp-api` at start; the API refuses to start on any other value) applies to every publish of every system workflow and template in every organisation: `off` (default) means no second person is needed; `code` means a draft whose function change set (§42.10) is non-empty must be published by an editor who is not one of its authors; `all` means every publish needs such a second editor. Request-level separation of duties (submitter ≠ approver, proposer ≠ resolution approver) is not affected and is always on | User decision 2026-11-10 (OP-10 resolved). The sandbox (§42.9 L1, L2) is the boundary in every mode; four-eyes is a governance choice per deployment, not a security dependency |
| D-56 | **Failure classes.** User-code faults are non-retryable and take `error`: `thrown`, `fuel_exhausted`, `memory_exceeded`, `stack_exceeded`, `timeout`, `invalid_output`, `input_unrepresentable`. Infrastructure faults are retried (`runner_unavailable`, `runner_busy`; 3 attempts, backoff 1 s → 4 s) and then take `error` with `runner_unavailable`. `runner_refused` (an integrity mismatch) is non-retryable and raises an ops alert | Faults are routable and deterministic. The infrastructure is retried, not the code |
| D-57 | **Limits are fixed by the platform, not per node** (§42.9 table). Hard storage bounds return 422 at `PUT`; semantic limits are V-59 issues | One table to review; authors cannot raise them |
| D-58 | **Input scoping and the entry snapshot.** An `expression` node may read any name in its context's `scope_table` (part 11 D-44, D-45) with no declaration step; its reads list is derived from the resolved spans (`wf_fx_refs`) and shown read-only. A `function` node reads **only** its declared inputs, each bound to a variable or platform key in scope, or to a typed literal; an FX expression is not a binding (compute it in an expression node first). Both kinds read one snapshot of their inputs, taken when the token enters the node; inside an expression node a later line sees an earlier line's value only as `$.name`. A line without a target is node-local and is never visible downstream | One evaluator per node; replay-safe (the inputs are in the event); least data to code |
| D-59 | **Outputs are declared variables only, written all or nothing; error targets are configurable.** Results pass on through ordinary `request` or `call` variables that later nodes read by ID. There are no node-scoped outputs such as `risk_fx.score`. On `next` every target is written; on `error` none is, except the node's optional **error targets**: `error_class_target` (a text variable; receives the fault key, for example `divide_by_zero` or `fuel_exhausted`) and `error_message_target` (a text variable; receives a message of at most 200 characters: the FX fault rendered by the engine, or the thrown JavaScript message with control and bidi characters removed). On `next` they are left unchanged. Neither may be a `ref` or an `in` param (V-66). The variable picker offers "+ New variable", so naming an output feels as in Opus, but the variable still belongs to the definition | Keeps "every slot always holds a value" (part 11 §41.2), rename safety (D-44, D-52) and V-56 unchanged; node outputs would need nulls, a new reference kind and a new dominance check |
| D-60 | **Ports stay `next` and `error`; branching uses `rule` or `switch`; rule port labels are author-defined.** True/false: write a bool, then a `rule`. Several outcomes or thresholds: write a value, then a `switch` (cases plus `default`). The editor action "Branch on this result" inserts the right node after `next`, pre-filled (bool: rule `target == true`; enum: one switch case per value; number or timestamp: one threshold case). The `rule` ports keep the keys `true` and `false`, but each carries a label of 1..40 characters chosen by the author from presets ("True / False" (default), "Yes / No", "Pass / Fail", "Met / Not met") or typed freely; the label is display only, and the trace, history and API use the key. Rule and switch conditions stay in the visual builder; they do not accept FX text | Matches Opus (its Switch routes; Expression and Function do not). One way to write a condition. Custom labels help readers; presets help beginners |
| D-61 | **Variables are created at edit time from the node, never at run time.** Writing to an undeclared name never creates a variable: an unknown name is V-58 when read and V-60 when targeted. In the inspector, an FX line whose name matches no variable offers the chip "Create variable `name` (Type)"; a function output offers "+ New variable" pre-filled with the output's name and type. The type is inferred from the line (int asks "Whole number or decimal?"; a line with errors needs an explicit type); the initial value is pre-filled by type (0, 0.0000, false, ""), and an enum or timestamp needs an explicit one. The variable is a `request` variable in a root and a private `call` variable in a template, never an `in`, `out` or `ref` param (those change the template interface, part 4 §21.5). It is created in the same graph `PUT` as the target and undone with it. A name equal to an existing variable offers "Use existing"; a name equal to a platform key is refused. Deleting the node offers to delete variables that only it writes | Keeps every slot declared, typed and holding a value on every path (part 11 §41.2); typos stay errors; no null; rename-safe. Opus-like convenience without runtime symbol creation |

### §42.3 Node catalogue rows (extend part 4 §21.2)

| Kind | In | Out-ports | Configuration | Status projection |
|---|---|---|---|---|
| `expression` | 1 | `next`, `error` | 1..16 ordered **lines**. Each has a `name` (`^[a-z][a-z0-9_]{0,39}$`, unique in the node), an optional **target** (a writable variable of the node's scope; part 4 V-22 rules) and FX source (≤1000 characters). A line without a target is a node-local temporary that later lines read as `$.name`. Optional error targets (D-59) | — |
| `function` | 1 | `next`, `error` | Source (JavaScript ES2020, ≤20 KiB UTF-8); inputs (name, type, binding: variable, platform key or typed literal); outputs (name, type, target variable); optional error targets (D-59) | — |

In a template, both kinds may write call variables. From a lane, writing a root request variable
goes only through a `ref` and only in the additive form `x := x ± e` (part 11 D-47):
- An FX line may do that.
- A function output may never target a `ref` (V-60).

### §42.4 The FX language

**Grammar** (EBNF; whitespace and `//` line comments are allowed between tokens):

```
line      = expr ;
expr      = ternary ;
ternary   = or_expr [ "?" expr ":" expr ] ;
or_expr   = and_expr { "or" and_expr } ;
and_expr  = not_expr { "and" not_expr } ;
not_expr  = [ "not" ] cmp_expr ;
cmp_expr  = add_expr [ ( "==" | "!=" | "<" | "<=" | ">" | ">=" ) add_expr
                     | [ "not" ] "in" ( list | range ) ] ;
list      = "[" expr { "," expr } "]" ;                       (* 1..16 elements *)
range     = ( "[" | "(" ) add_expr ".." add_expr ( "]" | ")" ) ;
add_expr  = mul_expr { ( "+" | "-" ) mul_expr } ;
mul_expr  = pow_expr { ( "*" | "/" | "%" ) pow_expr } ;
pow_expr  = unary [ "^" unary ] ;                             (* right operand: int literal 0..64 *)
unary     = [ "-" ] primary ;
primary   = literal | ref | call | "(" expr ")" ;
ref       = ident { "." ident } | "$" "." ident | "#" "." ident ;
call      = fn_name "(" [ arg { "," arg } ] ")" ;
arg       = expr | closure ;                                  (* closure only in items functions *)
closure   = expr ;                                            (* may use "#"; evaluated per item *)
literal   = int | decimal | "true" | "false" | string | "d(" string ")" ;
ident     = /[a-z][a-z0-9_]{0,39}/ ;
string    = '"' { char | '\"' | '\\' } '"' ;                  (* no other escapes; ≤200 chars *)
```

**Names.** An identifier must be one of the following, otherwise the line gets V-58:
- a variable in the node's `scope_table` (D-44);
- a platform key of the catalogue (`item_count`, `window_hours`, `item.credential_type`, …,
  part 11 §41.2);
- `$.name` of an **earlier** line of the same node;
- `#.attr` inside an items closure;
- `items` as the first argument of an items function in a root.

There is no computed member access (`a[b]`), no assignment operator, no user-defined function and no
access to any object model. `__proto__`, `constructor` or `prototype` are just unknown names
(V-58).

**Types.** These are the types of part 4 §21.4 and part 11 (`int` i64, `decimal` numeric(18,4),
`bool`, `text` ≤200, `timestamp`, `enum`). The only implicit conversion is int to decimal
promotion.
- A string literal compared with an enum must be a declared value of that enum (V-23).
- Ordering applies to int, decimal, timestamp and text (by code point). Ordering a bool or enum is
  invalid (V-58).
- `+` on text concatenates. `-` on two timestamps is invalid; use `hours_between`.
- The ternary needs a bool condition and branches of one type.
- A line's type must equal its target's type, or be int for a decimal target.

**Functions (closed allow-list; every one is total and pure):**

| Group | Functions |
|---|---|
| Number | `abs(n)`, `min(n, n, …)` and `max(n, n, …)` (2..8 arguments of one type), `round(d, dp)` (half-even; `dp` int literal 0..4), `floor(d)` and `ceil(d)` → int |
| Text | `len(t)` → int, `upper(t)`, `lower(t)`, `trim(t)`, `contains(t, t)`, `starts_with(t, t)`, `ends_with(t, t)` → bool |
| Time | `now()` (the workflow clock, RC-17), `hours_between(ts, ts)` → decimal, `add_hours(ts, n)` → ts |
| Items (root only, part 4 R2) | `count(items, pred)` → int, `some(items, pred)` and `all(items, pred)` → bool, `len(items)` = `item_count` |

ZEN features **left out on purpose**:
- `??` (FX has no null; part 11 has no unset state).
- Regex and `matches` (ReDoS risk).
- Template strings.
- `map`/`filter` returning lists (FX has no list type).
- Statement chaining with `;` (use lines and `$.name`).
- Every date function that needs a time-zone database (SGT formatting belongs in a Function node).

**Evaluation (D-58).** Lines run in order over one snapshot of the variables taken when the token enters the node. A bare name always reads the entry value, even after an earlier line of the node has targeted that variable; `$.name` reads a line's new value:
- A line's value is checked against its target's bounds.
- The node commits all targets together, or none if any line faults.

`FxFault` is one of:
- `divide_by_zero`;
- `overflow` (checked i64 and `rust_decimal`);
- `decimal_out_of_range` (outside numeric(18,4) after half-even rounding to 4 dp);
- `text_too_long`;
- `timestamp_out_of_range`.

**Static cost.** There are no loops, so worst-case cost is known at compile time:

`cost = Σ node weights + 10 × (closure body weights)` (an items closure runs at most `max_items` ≤ 10
times).

A line over 10,000 units, an AST deeper than 32 or larger than 256 nodes is V-59. Evaluation
therefore needs no run-time fuel.

**Examples** (seeded HRIS v2 draft, S14):
```
per_item    = window_hours / item_count                                 // decimal, temporary
risk_score  = round($.per_item * (risk_level == "high" ? 2 : 1), 1)     // → risk_score (decimal)
needs_dual  = count(items, #.credential_type == "token") >= 2 or $.per_item > 4   // → needs_dual (bool)
```

### §42.5 FX storage, parsing and rename

- `PUT` parses each line in `mvp-engine::fx::parse`, an iterative Pratt parser with an explicit
  depth counter. It never recurses on the Rust stack per nesting level, never panics (RC-11) and
  returns `Vec<Issue>` with byte spans.
  - Unresolvable names are kept as unresolved spans. The draft saves, and the issues are V-57 or
    V-58.
  - A line over 1,000 characters, more than 16 lines, or a string over 200 characters is refused
    with **422 `VALIDATION_FAILED`** (a hard bound, D-57). Nothing is stored.
- `compile` maps the lines and spans to a typed `FxProgram`: `Vec<FxLine{target: Option<Slot>,
  ast: FxAst, ty: Ty}>`. `FxAst` is an exhaustive enum (RC-03, RC-04).
- **Rename.** Renaming a variable in a draft rewrites every `wf_fx_refs` span that points to it, and
  the matching `source_text`, in the same transaction. A published definition is never rewritten
  (part 4 §23.1 trigger).
- **Source hygiene** (both kinds): the source is NFC-normalised. It is refused (422) if it holds C0
  or C1 control characters other than tab and newline, or any Unicode bidirectional override or
  isolate (U+202A–U+202E, U+2066–U+2069; "Trojan Source", CVE-2021-42574).

### §42.6 FX runtime, records and simulation

- `settle` evaluates the program when the token enters the node, then emits
  `Effect::SetVariable{…, source: ChangeSource::Expression}` per target and
  `Effect::RecordExpression{line, rendered, value | fault}`. It then emits `LeaveNode{next}`, or
  only the records and `LeaveNode{error}` on a fault.
- On a fault the only writes are the error targets (D-59): `error_class_target` := the fault key and
  `error_message_target` := the rendered message, for example "Division by zero in line per_item".
  The same rule applies to a function's `Event::FunctionCompleted` with an error class.
- Consumed reads go to `request_rule_inputs` (R6, so `approver_edit` locks them like a rule input).
  Variable changes record `source='expression'`.
- The rendered text uses the part 4 §24.3 trace format. Example:
  `risk_score := round((window_hours (6) / item_count (3)) * 2, 1) → 4.0`.
- Simulation is unchanged (part 4 §24.3): FX is pure and runs in-process.

### §42.7 Function contract

```js
// ES2020. The module must export `handler`; it may be async.
export const handler = async (input) => {
  const start = dayjs(input.start).utcOffset(480);     // dayjs is preloaded (utc plugin)
  const end   = dayjs(input.end).utcOffset(480);
  const risk  = new Big(input.risk).toFixed(1);         // big.js is preloaded as Big
  return { window_label: `${start.format('ddd D MMM HH:mm')}–${end.format('HH:mm')} SGT · risk ${risk}` };
};
```

**Input value mapping.** `input` is a frozen plain object holding exactly the declared input names, read from the entry snapshot (D-58).

| Type | Sent as | Received output must be | Otherwise |
|---|---|---|---|
| `int` | number (\|x\| ≤ 2^53−1) | an integer number with \|x\| ≤ 2^53−1 | input outside the safe range → `input_unrepresentable`; bad output → `invalid_output` |
| `decimal` | string `^-?\d{1,14}(\.\d{1,4})?$` | a string of that form, or a `Big` (its `toJSON` gives that form) | `invalid_output` (numbers are refused, so floats never reach a decimal) |
| `bool` | boolean | boolean | `invalid_output` |
| `text` | string ≤200 | a string ≤200 characters, no C0/C1 controls or bidi overrides | `invalid_output` |
| `timestamp` | ISO-8601 UTC string | ISO-8601 string with an offset, normalised to UTC | `invalid_output` |
| `enum` | the value string | one of the target enum's values | `invalid_output` |

**Output rules.**
- The returned value (after awaiting) must be a plain object (prototype `Object.prototype` or
  `null`) with exactly the declared output keys and no getters.
- The host parses the serialised JSON (≤16 KiB) against the declared schema, with unknown keys
  refused. `__proto__` as a key is an unknown key.
- `NaN`, `Infinity`, `undefined` and nested objects are refused.

**Time.** `Date.now()`, `new Date()` and `dayjs()` return `now`, the workflow time when the token
entered the node (RC-17). `Math.random()` throws `Error("random is not available")`.

**Console.** `console.log`, `info`, `warn` and `error` lines are captured **only** in simulation
and Test run (at most 64 lines of 200 characters). In a real run they are discarded and never
written to any server log (§42.9 L4).

### §42.8 Function runtime

```
LaneWorkflow / BreakglassRequestWorkflow (deterministic)
  settle → Effect::RunFunction{instance, node, fn_sha, inputs, now}   token waits
  └─ activity run_function(op_ref, definition_id, node_id, fn_sha, inputs, now, org_id)
       1. read wf_function_sources(node_id) from the DB (published, immutable); check sha256 == fn_sha
          (mismatch → runner_refused, non-retryable, ops alert)
       2. connect to /run/fn-runner/sock (Unix socket, group fnrun), one job per connection,
          u32-length-prefixed JSON frame ≤ 64 KiB: {kind:"run", fn_sha, source, inputs, now, org_id}
  mvp-fn-runner (separate process; no DB, no Temporal, no network)
       3. admission: global semaphore 8, per-organisation share 4, queue wait ≤ 2 s (else runner_busy)
       4. verify sha256(source) == fn_sha; new wasmtime Store from the pre-compiled QuickJS module
          (its SHA-256 is pinned and checked at runner start)
       5. bootstrap realm (§42.9 L1) → evaluate source → call handler(input) → drain the job queue
       6. reply {ok, outputs_json, fuel_used, duration_ms, runner_build} | {error_class, message}
  activity returns the reply (non-retryable error types per D-56)
  settle(Event::FunctionCompleted{instance, result}) → SetVariable…(source=Function) + LeaveNode(next|error)
  persist_settlement → request_function_runs row, variable changes, audit workflow.function_executed
```

- **Activity options:** start-to-close 10 s; retry policy maximum 3 attempts, initial interval 1 s,
  backoff ×2. The non-retryable types are every user-code class of D-56 plus `runner_refused`.
- **Idempotency:** the function is deterministic given `(fn_sha, inputs, now, runner_build)`, so a
  retry is safe. `persist_settlement` deduplicates by `op_ref` as before.
- **Determinism and replay:** workflow code never sees the source. Replay reads the activity result
  from history. `runner_build` is recorded per run, so "simulation equals execution" is asserted per
  build (R-7).
- **The API also uses the runner** for `check` (compile only: the source parses as a module and
  exports a function `handler`) and for simulation and Test run (§42.13). The worker and the API
  link no WASM or JS engine (RC-20).

### §42.9 Threat model and jailbreak controls

**Attacker models.**
- **T1** a malicious definition editor, or one whose session is compromised (any editor may write code, D-55).
- **T2** an insider who wants a logic change published without review.
- **T3** a request participant who controls variable *values* (field input, item choice) but no
  code.
- **T4** a supply-chain compromise of QuickJS, wasmtime or the preloaded libraries.

**Assets.** DB and Temporal credentials (worker and API); other organisations' data; request
integrity (status, approvals); availability of the worker and runner.

| Layer | Control | Defeats |
|---|---|---|
| L0 FX language | Closed grammar and allow-list, with no eval, host calls, object model, computed access or loops. Names resolve only through `scope_table`, so another definition's or organisation's variable cannot be named (V-58, AC-34.1.2). Values are typed, so input from T3 is data, never syntax | Code execution and data reach through FX: there is no mechanism to escape into |
| L0 FX limits | Line ≤1,000 characters, ≤16 lines, AST depth ≤32 and ≤256 nodes, static cost ≤10,000; iterative parser with an explicit depth counter (the Rhai CVE-2024-36760 lesson); checked arithmetic; `cargo fuzz` target on `fx::parse` and `fx::eval` | Parser and evaluator DoS, panics |
| L0 source hygiene (both) | NFC; control characters, bidi overrides and isolates refused (§42.5) | Reviewer deception (Trojan Source), log injection |
| L1 JS realm | **Fresh WASM instance and QuickJS runtime per call**, so no state survives between calls or tenants. A bootstrap runs before user code: it deletes `eval`; replaces the `constructor` of `Function`, `AsyncFunction`, `GeneratorFunction` and `AsyncGeneratorFunction` prototypes with throwing stubs; registers no module loader (`import()` and `require` fail); omits the `std` and `os` modules, `WebAssembly`, `SharedArrayBuffer` and `Atomics`; fixes `Date` to `now`; makes `Math.random` throw; then freezes every intrinsic and the preloaded `dayjs` and `Big`, SES-lockdown style. These are **hardening**: the boundary is L1-WASM and L2 | Prototype pollution across runs, `this.constructor.constructor('return globalThis')()`, dynamic code, non-determinism |
| L1 WASM boundary | QuickJS (`quickjs-ng`, pinned) built for `wasm32-wasi`. The host provides **no** preopened directories, environment, arguments, sockets or clock (`clock_time_get` returns `now`; `random_get` returns a fixed seed). The only I/O is one input buffer and one output buffer through stdin and stdout pipes in memory. wasmtime configuration: `consume_fuel` (the budget is deterministic, so simulation equals execution), epoch interruption, `max_wasm_stack` 512 KiB, a `ResourceLimiter` capping linear memory at 32 MiB and tables at 10,000 elements, threads off, default Spectre mitigations on | Escape from QuickJS memory-safety bugs (C code), infinite loops, memory bombs, recursion, never-settling promises (the job queue is drained under the same fuel) |
| L1 data minimisation | A function sees only its declared inputs. No platform key exposes justification text, remarks, user identities, evidence or credentials. Outputs are schema-checked (§42.7) and written only to declared targets | Exfiltration (there is nothing more to read), mass assignment |
| L1 logic containment | A function or expression can only compute values. It cannot emit effects, call activities, signal, change status, skip a node or write a `ref` (V-60). Approvals, owner dominance (V-14) and return or reject wiring (V-12, V-13) are structural and validated, so no computed value can bypass an approval: at most it chooses between drawn routes, exactly as a rule can | "Jailbreaking" the business process through the node |
| L2 process | `mvp-fn-runner` runs as its own container: `network_mode: none`, read-only root filesystem, non-root UID, `cap_drop: [ALL]`, `no-new-privileges`, the default seccomp profile, `pids_limit` 64, memory 512 MiB, CPU 2. It holds **no** secrets: no DB URL, no Temporal address, no session key. Its only interface is the Unix socket on a shared volume (mode 0660, group `fnrun`). A native (non-container) dev run is allowed but `/api/health` then reports `fn_runner: "unhardened"` and the editor shows a banner | A wasmtime escape lands in an empty, offline, unprivileged box; SSRF (OWASP A10); lateral movement |
| L3 authoring | Every change to function source or I/O is audited in the `PUT` transaction (`workflow_definition.function_source_changed{node_key, old_sha, new_sha}`), and every author of the draft is recorded in `wf_draft_authors`. The Diff tab shows code as a line diff. With `FOUR_EYES_PUBLISH=code` or `all`, publishing needs a second editor (§42.10). In every mode the sandbox (L1, L2) is the boundary, so code published by one person can at most compute wrong values on drawn routes (L1 logic containment) | T1 hiding a change; T2 publishing alone where the deployment requires review |
| L3 integrity | `fn_sha` in `CompiledPlan`. The activity and the runner both check the SHA-256 of the source (`runner_refused` on mismatch, plus an ops alert). The runner checks its pinned QuickJS module SHA-256 at start and refuses to serve otherwise. `content_sha256` at publish covers the source (part 4 §23.1) | Tampering with stored code or the runtime (OWASP A08) |
| L4 audit and logs | `workflow.function_executed{node_key, fn_sha, input_sha256, outputs, fuel_used, duration_ms, error_class}` in the settlement transaction. Test runs write `workflow_definition.function_test_run`. Console output is never logged outside simulation and Test run. Runner logs carry no input or output values | Forensics (OWASP A09) without log leakage |
| L4 abuse | Test run and function-executing simulations: 10 per minute per user (429 `RATE_LIMITED`). Runner admission per §42.8. A `check` is cached by `(source_sha256, runner_build)` | Resource exhaustion through the editor |
| Supply chain | RC-20: `wasmtime`, the QuickJS module, `dayjs` and `big.js` exist **only** in `mvp-fn-runner`, vendored and SHA-pinned. `mvp-fitness` fails if any other crate links them. A `cargo deny check advisories` gate covers the vendored tree. Upgrading wasmtime or QuickJS bumps `runner_build` | T4 (OWASP A06) |
| UI | FX and JS source render as text in CodeMirror 6 (syntax highlighting only, no linting worker that evaluates code). Rendered expressions, outputs and console lines are text nodes; `dangerouslySetInnerHTML` is banned by lint in `web/features/editor` and `web/features/request`. The browser never evaluates FX or JS (part 8 I-11) | Stored XSS through code, outputs or names (OWASP A03) |

**Limits table (D-57).**

| Limit | Value | Breach |
|---|---|---|
| FX lines per node / characters per line / string literal | 16 / 1,000 / 200 | 422 `VALIDATION_FAILED` |
| FX AST depth / nodes / static cost | 32 / 256 / 10,000 | V-59 |
| Function source | 20 KiB UTF-8 | 422 `VALIDATION_FAILED` |
| Function inputs / outputs | 8 / 8 | V-59 |
| Input or output JSON | 16 KiB each | `input_unrepresentable` / `invalid_output` |
| Fuel | 500,000,000 units (calibrated in ST-36 so that the S14 function uses under 1%) | `fuel_exhausted` |
| Wall clock (backstop only) | 2 s epoch deadline (Opus allows 5 s) | `timeout` |
| Linear memory / WASM stack | 32 MiB / 512 KiB | `memory_exceeded` / `stack_exceeded` |
| Console in simulation | 64 lines × 200 characters | truncated, with a marker |
| Runner concurrency | 8 overall, 4 per organisation, 2 s queue wait | `runner_busy` (retryable) |

**Red-team corpus** (`mvp-fn-runner/tests/redteam/*.js` and `mvp-engine/tests/fx_redteam.rs`).
Every entry must fail closed with the stated class, and every run must leave no state behind for
the next:

| # | Input | Expected |
|---|---|---|
| RT-01 | `import('os')`, `import fs from 'fs'`, `require('fs')` | `check` fails (V-62), or `thrown` |
| RT-02 | `eval('1')`, `new Function('return this')()`, `(async()=>{}).constructor('…')` | `thrown` |
| RT-03 | `this.constructor.constructor('return globalThis')()`, `Object.getPrototypeOf(async function*(){}).constructor` | `thrown` |
| RT-04 | `Object.prototype.polluted = 1`, then a second call reads `({}).polluted` | first: `thrown` (frozen); second sees `undefined` |
| RT-05 | `while (true) {}`; `for (;;) Promise.resolve()`; `new Promise(() => {})` | `fuel_exhausted`; `fuel_exhausted`; `invalid_output` (the promise never settles) |
| RT-06 | `'a'.repeat(2 ** 30)`; `new Array(1e9).fill(0)`; `const f = () => f(); f()` | `memory_exceeded`; `memory_exceeded`; `stack_exceeded` or `thrown` (RangeError) |
| RT-07 | `Atomics.wait(…)`, `new SharedArrayBuffer(8)`, `WebAssembly.compile(…)` | `thrown` (ReferenceError) |
| RT-08 | `return {__proto__: {x: 1}}`; extra key; missing key; `NaN`; `1e400`; getter | `invalid_output` |
| RT-09 | `Math.random()`; then `Date.now()` compared with input `now` | `thrown`; equal |
| RT-10 | source with U+202E, or a NUL byte | 422 at `PUT` |
| RT-11 | FX: `((((…` 10,000 deep; `1e400`; `"a" * 3`; `constructor`; `__proto__.x`; `a[b]` | 422 or V-57, V-58, V-59; no panic, no stack overflow |
| RT-12 | FX referencing a variable ID of another organisation in a crafted `PUT` | V-58, with no timing difference and the same body as a random UUID (as AC-33.8.4) |
| RT-13 | Runner frame over 64 KiB, malformed length prefix, a second job on one connection, an `fn_sha` that does not match | refused, connection closed; `runner_refused` |
| RT-14 | `console.log` 10,000 times in a real run | no server log line; run succeeds or exhausts fuel |

### §42.10 Authoring, review and publish

- **Who may do what.** Any definition editor (part 4 §24.1, D-38, D-40) may add, edit, wire and
  delete both kinds, including function source and I/O, and may Test run. Owners and other
  read-only viewers see FX and code read-only. Everyone else gets 404, as today.
- **Draft authors.** Every `PUT` that changes a draft adds the caller to `wf_draft_authors`
  (`touched_function_code` is set when the change touched function source or I/O). Import and clone
  record the importer or cloner as an author with `touched_function_code` set when the file or
  source holds function nodes.
- **Function change set** of a draft: every function node whose `(source_sha256, inputs, outputs)`
  differs from `based_on_version`, including added and removed nodes. A draft with no base (a new
  system, a clone or an import) counts every function node.
- **Four-eyes publish (D-55).** `mvp-api` reads `FOUR_EYES_PUBLISH` once at start:

  | Value | Publish needs |
  |---|---|
  | `off` (default) | Edit rights only |
  | `code` | If the function change set is non-empty: a publisher who is not an author with `touched_function_code`. Otherwise edit rights only |
  | `all` | A publisher who is not any author of the draft |
  | any other value | The API does not start; it logs `FOUR_EYES_PUBLISH must be off, code or all` |

  - A refused publish returns 409 `FOUR_EYES_REQUIRED` with the reason and, for `code`, the node
    keys; nothing changes.
  - The publish audit row adds `four_eyes_mode`, `function_change_set[{node_key, sha}]` and, when a
    second person was needed, `reviewed_by = publisher`.
  - The mode applies to system workflows and templates alike, in every organisation. Changing it
    needs a restart and applies to the next publish; published versions are never affected.
  - `GET /api/auth/me` returns `policy.fourEyesPublish`, so the editor shows the review section only
    when it applies.
- **Check at publish.** Publish runs the runner `check` synchronously for every function in the
  definition and its pinned templates; any failure is V-62. If the runner is down, publish returns
  503 `FN_RUNNER_UNAVAILABLE`.
- **Export, import and clone** carry the source verbatim.

### §42.11 Validation codes (extend part 4 §22)

| Code | Sev | Rule |
|---|---|---|
| V-57 | error | FX syntax error. The issue carries `line` and the byte `span` |
| V-58 | error | FX name or type error: unknown name, a name outside the node's scope, `$.name` of a later line, `#` outside an items closure, an items function outside a root, mixed types, an ordered bool or enum, a line type that does not match its target. Refines V-20, V-21, V-23 |
| V-59 | error | Limits: AST depth, node count or static cost (§42.4); more than 8 function inputs or outputs |
| V-60 | error | Targets. An FX target or function output that is not writable (V-22 rules). A function output on a `ref`. An FX write to a `ref` from a lane that is not `x := x ± e`. Two lines or outputs of one node with the same target. Refines V-22, V-53 |
| V-61 | error | Function params: names `^[a-z][a-z0-9_]{0,39}$`, unique per direction; a binding of the same type (an enum input binds an enum variable or one of its values); every output has a target |
| V-62 | error | The runner `check` failed for this source and runner build (syntax, no exported `handler`, `handler` not a function). In `validate` the result comes from `wf_function_checks` (passed in `DefinitionRows`), so `validate` stays pure. A missing check result is a V-62 **warning** "Not yet checked"; publish forces the check |
| V-63 | warning | Lint for `import`, `require`, `eval`, `Function(`, `constructor`, `WebAssembly`, `Atomics`, `globalThis`, `Math.random`. This is a lexical hint for reviewers only; L1 enforces |
| V-64 | error | An `expression` or `function` node inside an exception handler (D-49) |
| V-65 | error | The node saves nothing: an `expression` node with no targeted line, or a `function` node with no outputs |
| V-66 | error | An error target is not a writable text variable of the node's context, is a `ref` or `in` param, or is also a target of a line or output of the same node |
| V-67 | warning | A temporary line (no target) is never read as `$.name` by a later line |
| V-68 | error | A `rule` port label is blank, longer than 40 characters, or equal (ignoring case) to the other port's label |
| V-69 | warning | A variable is written by an `expression` or `function` node but never read by any node, binding, field or template output (the variable-level form of V-67) |

V-56 (read before write) and the dataflow report (part 11 §41.12) include FX reads, function input
bindings, both kinds' writes and the error targets.

### §42.12 Data model (no JSONB; extends part 4 §23.1 and §23.2)

Every `wf_*` table below is covered by the part 4 §23.1 immutability trigger.

| Table | Key columns | Constraints |
|---|---|---|
| `wf_nodes.kind` | adds `expression`, `function` | CHECKs tie the new kinds to their child rows |
| `wf_fx_lines` | node_id, ordinal 1..16, name(40), target_variable_id NULL, source_text varchar(1000) | unique (node_id, ordinal) and (node_id, name); the node is an `expression`; the target lies in the node's definition (composite FK) |
| `wf_fx_refs` | line_id, span_start, span_end, variable_id NULL, platform_var(40) NULL, line_ref_ordinal NULL | exactly one of the three; `0 ≤ span_start < span_end ≤ octet_length(source_text)`; composite FK keeps `variable_id` in the definition |
| `wf_function_sources` | node_id (PK), language (`js_es2020`), source_text text, source_sha256 bytea | `octet_length(source_text) ≤ 20480`; `octet_length(source_sha256) = 32`; the node is a `function` |
| `wf_function_params` | node_id, direction (`in, out`), ordinal 1..8, name(40), type, in_variable_id NULL, in_platform_var(40) NULL, lit_* typed NULL, out_variable_id NULL | unique (node_id, direction, name); `in`: exactly one of variable, platform key and literal; `out`: target required |
| `wf_node_error_targets` | node_id (PK), class_variable_id NULL, message_variable_id NULL | the node is an `expression` or `function`; at least one set; composite FKs keep both in the definition |
| `wf_node_ports.label` | `rule` ports `true` and `false` take the author's label (D-60) | 1..40 characters after trim; hygiene as §42.5 (422 otherwise); default "True" and "False" |
| `wf_draft_authors` | definition_id, author_id, touched_function_code bool, first_changed_at, last_changed_at | PK (definition_id, author_id); draft definitions and templates only (trigger) |
| `wf_function_checks` | source_sha256, runner_build(40), ok bool, message(200), checked_at | PK (source_sha256, runner_build); a cache, not owned by a definition |
| `request_expression_evaluations` | node_instance_id, line_ordinal, rendered(500), value_rendered(200) NULL, fault NULL, op_ref | `fault` ∈ §42.4 `FxFault`; exactly one of value and fault |
| `request_function_runs` | node_instance_id, fn_sha bytea, runner_build(40), input_sha256 bytea, outcome (`ok, error`), error_class NULL, message(200) NULL, fuel_used bigint, duration_ms int, attempts smallint, op_ref | `error_class` ∈ D-56; unique (node_instance_id); append-only (INSERT/SELECT grant) |
| `request_variable_changes.source` | adds `expression`, `function` | — |

`CompiledPlan` additions (additive serde, so `PLAN_FORMAT` is unchanged, RC-14 and RC-15):
- `NodeKind::Expression(FxProgram)`;
- `NodeKind::Function{fn_sha, inputs: Vec<FnInput>, outputs: Vec<FnOutput>}` (no source text);
- `Effect::RecordExpression`, `Effect::RunFunction`;
- `Event::FunctionCompleted`;
- `ChangeSource::{Expression, Function}`.

`LaneInput` and the `act` payload are unchanged.

### §42.13 API (extends part 4 §24.2)

| Method & path | Notes |
|---|---|
| `PUT /api/admin/workflow-definitions/:defId` | The graph DTO gains `expression.lines[{name, target?, source}]` and `function{source, inputs[], outputs[]}`. Hard bounds → 422. Nodes may carry `errorTargets{class?, message?}`; function `inputs[].binding` is `{variable | platform | literal}`; `rule` ports carry `label`. Each change adds the caller to `wf_draft_authors`. `issues[]` carry `line` and `span` for V-57/V-58. A changed source triggers an asynchronous `check`; its result appears in the next `validate` |
| `POST /api/admin/workflow-definitions/:defId/nodes/:nodeKey/test-run {inputs{}, now?}` | Draft or published; needs edit rights. Inputs are validated against the declared types (422). Returns `{outcome, outputs | errorClass, message, fuelUsed, durationMs, runnerBuild, console[]}`. 429 `RATE_LIMITED`; 503 `FN_RUNNER_UNAVAILABLE`. Writes only the audit row `workflow_definition.function_test_run` |
| `POST …/simulate` | `script[]` entries may stub a function: `{node_key, function_result: {outputs} | {errorClass}}`. Unstubbed functions call the runner (rate limit as Test run). Still writes no rows and never contacts Temporal (part 4 §24.3) |
| `POST …/publish` | Adds 409 `FOUR_EYES_REQUIRED` (when `FOUR_EYES_PUBLISH` requires a second person) and 503 `FN_RUNNER_UNAVAILABLE` (§42.10) |
| `GET /api/auth/me` | Adds `policy{fourEyesPublish: "off" | "code" | "all"}` |
| `GET /api/requests/:id/plan`, `/variables` | Function instances carry `{errorClass?, fuelUsed, durationMs}`. Expression instances carry their rendered lines. Visibility per part 1 §8; 404 otherwise |
| `GET /api/health` | Adds `fn_runner: "ok" | "degraded" | "unhardened"` and `four_eyes_publish: "off" | "code" | "all"` |

New error codes for `mvp-contract::ErrorCode`:
- `FOUR_EYES_REQUIRED` 409;
- `FN_RUNNER_UNAVAILABLE` 503, also the retryable activity error type `runner_unavailable`.

`runner_refused` is a non-retryable activity error with no HTTP mapping. New audit actions:
- `workflow.expression_evaluated`;
- `workflow.function_executed`;
- `workflow_definition.function_source_changed`;
- `workflow_definition.function_test_run`.

### §42.14 Editor, simulation and request page

- **Palette** (root and template mode): "Expression · f(x)" and "Function · JS" after "Set variable".
  Neither appears in the handler palette.
- **Both inspectors** (220 Prompt 31) show a read-only **Reads** list (FX: derived from the spans;
  function: the input bindings), an **On error** section with "Store error type in" and "Store error
  message in" text-variable pickers (each with "+ New variable"), and "Branch on this result"
  beside each targeted line or output (D-60). The node card subtitle reads "Writes risk_score,
  needs_dual" or "In 3 · Out 1".
- **Create variable from the node** (D-61): the "Create variable" chip on an FX line with an
  unknown name, "+ New variable" on outputs and error targets, a small form (name, type, initial
  value), and on node delete the option "Also delete risk_band, which only this step sets".
- **Rule inspector** (part 4 §24.1, 220 Prompt 23) gains "Port labels": a preset select ("True /
  False" (default), "Yes / No", "Pass / Fail", "Met / Not met", "Custom…"); "Custom…" shows two
  text fields (1 to 40 characters). The canvas, trace and request page show the labels; the keys
  stay `true` and `false`.
- **Expression inspector** (220 Prompt 31). An ordered line list: name, optional target picker
  (writable variables only, with "+ New variable") and a single-line CodeMirror field. Variable chips insert names from
  `scopes[]`, and `$.` chips insert earlier lines. Under each line, V-57 and V-58 are underlined at
  their span, and the rendered sample value from the last simulation is shown.
- **Function inspector** (220 Prompt 31):
  - Inputs and outputs tables in the style of the embed binding tables.
  - A full-height code editor (read-only for owners and viewers).
  - A "Test run" drawer with typed input fields, outputs, fuel bar, duration and console.
  - A "Last check" chip ("Checked with runner r2026.11.1", or the V-62 message).
- **Simulation trace** lines:
  - "Risk score: risk_score := round((6 / 3) * 2, 1) → 4.0".
  - "Format ticket (JS): ok, fuel 3.1 M, 14 ms → window_label".
  - On a fault, the class and the `error` port taken.
- **Diff** shows function source as a line diff. **Publish** lists the function change set and its
  authors, and when `FOUR_EYES_PUBLISH` requires it, the second-person rule (Prompt 29).
- **Request page.** The node card shows "f(x)" or "JS" and, after a run, its outcome. The Variables
  panel shows the source "expression" or "function (JS)" on change rows.

### §42.15 Modules and Rust conventions (extend part 8)

- `mvp-engine::fx` holds `parse`, `check` (types and scope), `cost` and `eval`. All are pure,
  exhaustive and covered by the 100% gate. A `cargo fuzz` target covers `parse` and `eval`.
- **New binary `mvp-fn-runner`** (Generic subdomain, low volatility). It depends on `mvp-contract`
  only (for the frame types and `runner_build`), plus vendored `wasmtime` and the pinned QuickJS
  `.cwasm`. Nothing depends on it, because it is reached only through the socket.
- `mvp-worker` (`run_function` activity) and `mvp-api` (Test run, simulate host loop, `check`) talk
  to it through `mvp-contract::fnrun::{Frame, RunRequest, RunReply, CheckReply}`.

**RC-20.** The JS engine, `wasmtime`, `dayjs` and `big.js` are allowed only in `mvp-fn-runner`. No
other crate may link a WASM runtime, a JavaScript engine or an expression-language crate. The
`mvp-fitness` banned list adds `wasmtime*`, `rquickjs*`, `boa*`, `quickjs*`, `rhai`, `evalexpr`,
`cel*` and `zen-*`. `#![forbid(unsafe_code)]` holds in `mvp-fn-runner` as well; wasmtime's internal
`unsafe` is a dependency's, as for `sqlx`.

**Risks (extend part 8 §34).**

| ID | Risk | Trigger | Response |
|---|---|---|---|
| R-6 | Building QuickJS for `wasm32-wasi` with the bootstrap of §42.9 L1, and freezing intrinsics without breaking dayjs, is unverified | First runner build | Spike first (ST-36 US-36.1). Prior art: Bytecode Alliance `javy`. If freezing breaks dayjs, preload a frozen copy built before lockdown |
| R-7 | Fuel is deterministic per wasmtime and QuickJS build, not across builds | A runner upgrade | `runner_build` is recorded per run and per check. The replay corpus pins expected fuel per build. An upgrade re-baselines the corpus and re-runs `check` |
| R-8 | With `FOUR_EYES_PUBLISH=off` (the default), one editor can write and publish function code alone | Deployment review | Production deployments should set `code` or `all`; the setting is shown in `/api/health` as `four_eyes_publish` so an operator can see it |

### §42.16 Seed and scenario S14

- **Seed.** The demo compose file sets `FOUR_EYES_PUBLISH=code`. The HRIS v2 draft gains:
  - request variables `risk_score` (decimal, init 0), `needs_dual` (bool, init false),
    `risk_error` (text, init ""), `window_label` (text, init "") and `window_error` (text, init "");
  - an `expression` node "Risk score" [`risk_fx`] after Ops Reviewers L1 `approve` and before
    `assess`, with the three lines of §42.4 and error class target `risk_error`. Its `next` goes to
    a `rule` "Many tokens?" [`dual_rule`] (`needs_dual == true`, port labels preset "Yes / No"); its
    `error` and the rule's `true` ("Yes") go to a Task "Check risk data" [`fix_risk`] (Ops Reviewers
    L1), whose `done` returns to `assess`; the rule's `false` ("No") goes to `assess`;
  - a `function` node "Format window" [`window_fn`] after the owner `approve`, with inputs `start` ←
    `activity_start_at`, `end` ← `activity_end_at` (platform keys), `risk` ← `risk_score`, the output
    `window_label`, the error class target `window_error`, and the source of §42.7. Its `error` goes
    to `notify_pa`, so a fault only loses the label.
- **S14 Expression and Function nodes.** Script:
  1. Amanda adds `risk_fx`, mistypes `risk_levle` and sees V-58 at the span. She fixes it and
     simulates 3 items: `risk_score` 4.0, `needs_dual` false, and `dual_rule` takes "No".
  2. On `needs_dual` she presses "Branch on this result": a rule is inserted after `next`, and she
     picks the labels "Yes / No". She sets the error target `risk_error` (V-66 clears).
  3. Amanda adds `window_fn` and first types `while(true){}`. Test run returns `fuel_exhausted`; the
     simulation takes `error` and the trace shows `window_error := "fuel_exhausted"`. She then types
     `eval("1")`: V-63 warns, and Test run returns `thrown`.
  4. Amanda pastes the §42.7 source. Test run returns "Tue 10 Nov 14:00–18:00 SGT · risk 4.0".
  5. Amanda presses Publish and gets 409 `FOUR_EYES_REQUIRED` (mode `code`).
  6. Greg opens the Diff, reads the code and publishes v2. The audit row shows
     `four_eyes_mode=code` and `reviewed_by=Greg`.
  7. Bob submits on v2, and the request's Variables panel shows `window_label ← function (JS)`.

  Shows: Opus-style f(x) and JS nodes, results passed as variables, branching with custom-labelled
  rule ports, error ports with stored error types, the sandbox failing closed, and configurable
  four-eyes.

### §42.17 Effect on other parts (supersession)

| Clause | Replaced or extended by |
|---|---|
| Part 4 §21.2 node table | Adds `expression` and `function` (§42.3) |
| Part 4 §21.4 "Division is absent" | Still true for rule, switch, set_variable and bindings. FX allows `/` and `%` (D-51) |
| Part 4 §21.4 "Overflow is an evaluation error … admin reset is required" | Still true for those kinds. In `expression` and `function` nodes a fault takes the `error` port (D-51, D-56) |
| Part 4 §24.1 "There is no free-text expression entry" | Still true for the rule builder and bindings. The `expression` node is FX text (D-50) |
| Part 4 §22 | Adds V-57…V-69 (§42.11) |
| Part 4 §23.1, §23.2 | §42.12 tables; `source` gains `expression` and `function` |
| Part 4 §24.2 publish and PUT; part 4 §24.3 simulation | §42.10, §42.13. Simulation may call the runner, but still writes nothing and never contacts Temporal |
| Part 4 §24.1 permissions | Unchanged: any definition editor edits both kinds (D-55). Publish follows `FOUR_EYES_PUBLISH` (§42.10) |
| Part 4 §21.2 `rule` ports; part 4 §23.1 `wf_node_ports.label` | Author-defined labels on `true` and `false` with presets (D-60, V-68); keys unchanged |
| Part 1 §5.1 processes | Adds `fn-runner` (§42.8) |
| Part 5 AC-33.8.6 import; part 6 §28 cloning | Unchanged permissions; the importer or cloner is recorded as a draft author (§42.10) |
| Part 8 §30.2, §30.3, §33.1 | Adds `mvp-fn-runner`, the RC-20 banned list and fitness test entries (§42.15) |
| Part 3 §15, §16, §19 | Adds S14, the red-team corpus and FX fuzzing, and prompt rule 23 |
| Addendum 220 Prompts 23, 26, 27, 29 | Palette, rule port labels, codes, trace lines and the publish review section; the new Prompt 31 holds both inspectors |

---

## Stories

### ST-35 Expression (f(x)) node

**US-35.1** As a designer, I write Opus-style expressions with live checking.
- AC-35.1.1
  - G: the HRIS v2 draft and the `risk_fx` node of §42.16.
  - W: the line `risk_score = round($.per_item * (risk_levle == "high" ? 2 : 1), 1)` is saved.
  - T: `PUT` returns 200 with V-58 at the span of `risk_levle`. Publish and simulate are refused.
    After the fix, `issues[]` has no V-57 or V-58.
- AC-35.1.2: `$.per_item` used in line 1 before it is defined in line 2 → V-58. `#.credential_type`
  outside `count(items, …)` → V-58. `count(items, …)` in a template → V-58.
- AC-35.1.3 (types): `risk_level > "low"` (ordering an enum), `true + 1`, `"a" * 3`, an int line
  targeting a bool, and `risk_level == "extreme"` (not an enum value) each give V-58 or V-23.
  `item_count / 2` targeting a decimal passes.

**US-35.2** As the platform, I evaluate expressions deterministically and atomically.
- AC-35.2.1 (S14): 3 items and `window_hours` 6 give `risk_score` 4.0. The change row has
  `source=expression`, and `request_expression_evaluations` holds 3 rendered lines.
  `request_rule_inputs` lists `window_hours`, `item_count` and `risk_level`.
- AC-35.2.2 (fault): with a draft line `x = window_hours / (item_count - 3)` and 3 items, the node
  takes `error`. **No** target of the node changes, the fault is `divide_by_zero`, and the request
  is not `failed`.
- AC-35.2.3: an i64 overflow, a decimal outside numeric(18,4), and a concatenation over 200
  characters each take `error` with their class.
- AC-35.2.4: `settle` applied twice to equal inputs gives equal effects. `now()` equals the workflow
  clock; replay yields the same values.
- AC-35.2.5: an approver edit that removes an item after `risk_fx` has evaluated → 409
  `RULE_INPUT_LOCKED` (R6).

**US-35.3** As a designer, renames do not break expressions.
- AC-35.3.1: renaming `risk_score` to `risk_index` in the draft rewrites `wf_fx_lines.source_text`
  and the spans in the same transaction. `validate` shows no new issue, and a published version's
  rows are unchanged.

**US-35.4** Security and bounds (OWASP A03, A04, A05).
- AC-35.4.1 (RT-11): a 1,001-character line, 17 lines, a 201-character string, or a NUL or U+202E
  character → 422 with field paths; nothing is stored. 10,000 nested parentheses within 1,000
  characters → V-59, with no panic and no stack overflow (property test plus fuzz corpus).
- AC-35.4.2 (RT-12): a crafted `wf_fx_refs.variable_id` of another organisation, or a name of a
  variable of another definition → V-58. The response is identical to a random UUID and has no
  timing difference.
- AC-35.4.3: `constructor`, `__proto__.x`, `a[b]` and `x = 1` → V-57 or V-58; nothing can reach an
  object model.
- AC-35.4.4: an expression node placed in an exception handler by a crafted `PUT` → V-64.

**US-35.5** As a viewer, I see how a value was computed.
- AC-35.5.1: the simulation trace and `/plan` show
  `risk_score := round((window_hours (6) / item_count (3)) * 2, 1) → 4.0`. The text is rendered
  escaped.

**US-35.6** As a designer, I know which values a node reads (D-58).
- AC-35.6.1: the inspector's Reads list equals the resolved spans: `window_hours`, `item_count`,
  `risk_level`, `items`. A variable not in the node's `scope_table` never appears in the picker.
- AC-35.6.2 (snapshot): with line 1 `risk_score = 9` (target `risk_score`) and line 2
  `x = risk_score + $.risk_score` (target `x`), and `risk_score` 0 on entry, `x` is 9 (0 + 9).
- AC-35.6.3: a temporary is not offered to any downstream node; a temporary never read later gives
  V-67 (warning).

**US-35.7** As a designer, I pass results on and branch on them (D-59, D-60).
- AC-35.7.1: a downstream `rule` on `needs_dual` reads the value written by `risk_fx`; the change row
  has `source=expression` and the node instance id.
- AC-35.7.2: "Branch on this result" on a bool inserts a `rule` (`needs_dual == true`) wired from
  `next`; on an enum, a `switch` with one case per value plus `default`; on a number, a `switch` with
  one threshold case plus `default`. It is one `PUT` and validates like any edit.
- AC-35.7.3 (labels): a rule saved with preset "Yes / No" shows those labels on the canvas, in the
  trace and on the request page; history and API use `true`/`false`. A custom label of 41
  characters, a blank label, or two equal labels → V-68; a label with U+202E → 422.
- AC-35.7.4: an expression node with no targeted line → V-65.
- AC-35.7.5 (error targets): on `divide_by_zero`, only `risk_error` changes, to `"divide_by_zero"`,
  and the message target (if set) gets "Division by zero in line per_item". A `switch` after
  `error` with case `risk_error == "divide_by_zero"` takes that case. An error target that is a
  number, a `ref`, an `in` param, or also a line target → V-66.

**US-35.8** As a designer, I create the variables a node needs without leaving it (D-61).
- AC-35.8.1: line `risk_band = risk_score >= 4 ? "high" : "low"` with no variable `risk_band` shows
  "Create variable risk_band (Text)"; until it is chosen the line is V-60 and nothing is created.
  Choosing it adds `risk_band` (text, init "") and sets the target in one `PUT`; Undo removes both.
- AC-35.8.2: a whole-number line asks "Whole number or decimal?"; an enum or timestamp needs an
  initial value before Create is enabled; a line with errors needs an explicit type.
- AC-35.8.3: in a template the new variable is a private `call` variable and never appears in the
  Interface tab; in a root it is a `request` variable. The name `item_count` (a platform key) is
  refused; the name of an existing variable offers "Use existing".
- AC-35.8.4: a crafted run-time write to an undeclared name is impossible: a `PUT` with a target
  `variable_id` not in the definition → V-60, and `CompiledPlan` holds no unresolved target.
- AC-35.8.5: `risk_band` set but never read → V-69 (warning); deleting `risk_fx` offers to delete
  `risk_band`, and declining keeps it with the Variables tab note "Unused".

### ST-36 Function (JS) node and sandbox

**US-36.1** As the platform, I run author code only inside the runner sandbox.
- AC-36.1.1 (spike, R-6): the runner builds QuickJS for `wasm32-wasi`, verifies its module SHA-256
  at start, and runs the §42.7 source with dayjs and big.js frozen. A tampered module makes the
  runner refuse to start, and `/api/health` shows `fn_runner: "degraded"`.
- AC-36.1.2: `cargo metadata` shows no `wasmtime*` or `*quickjs*` crate in the closure of
  `mvp-worker`, `mvp-api`, `mvp-engine` or any pure crate (RC-20).
- AC-36.1.3 (L2): the demo compose file runs the runner with `network_mode: none`, a read-only root
  filesystem, a non-root UID, `cap_drop: [ALL]`, `no-new-privileges`, no environment secrets and
  only the socket volume. A test inside the container cannot open a TCP connection or write outside
  `/tmp`.

**US-36.2** As the platform, I fail closed on every hostile function (red-team corpus).
- AC-36.2.1: RT-01 … RT-09 and RT-13 … RT-14 each return the class in §42.9. After each one, a
  benign run in the same runner process returns the expected output: no state persists (RT-04).
- AC-36.2.2: `while(true){}` returns `fuel_exhausted` within the 2 s backstop. In a request, the
  node takes `error`, the activity is not retried, and `request_function_runs.attempts=1`.
- AC-36.2.3: the same function and inputs give the same `fuel_used` in Test run, simulation and a
  real run on one `runner_build` (simulation equals execution).

**US-36.3** As the platform, I keep data in and out typed and minimal.
- AC-36.3.1: `input` holds exactly the declared names. `Object.keys(input)` equals the declared
  list, and `input` is frozen.
- AC-36.3.2 (RT-08): a returned extra key, a missing key, `__proto__`, `NaN`, a number for a
  decimal, a 201-character text or an unknown enum value → `invalid_output`. No variable changes,
  and the node takes `error`.
- AC-36.3.3: an int input above 2^53−1 → `input_unrepresentable` before the runner is called.

**US-36.4** As the platform, I keep function nodes deterministic and replay-safe.
- AC-36.4.1: killing the worker between `run_function` completing and `persist_settlement` gives,
  after restart, one `request_function_runs` row, one variable change and one audit row. The runner
  is not called again on replay.
- AC-36.4.2: stopping the runner makes the activity retry 3 times, then take `error` with
  `runner_unavailable`, with an ops alert on `/api/health`. Restarting it before the third attempt
  completes the node normally.
- AC-36.4.3: changing the stored source after publish (a DB-level tamper that bypasses the trigger
  in a test) → `runner_refused`, non-retryable, with an ops alert. No output is written.

**US-36.5** As an operator, I choose whether publishing needs a second person (D-55).
- AC-36.5.1: with `FOUR_EYES_PUBLISH` unset or `off`, Amanda edits `window_fn` and publishes alone;
  the audit rows `workflow_definition.function_source_changed` (old and new SHA) and the publish row
  (`four_eyes_mode=off`) are written.
- AC-36.5.2 (S14, `code`): Amanda's publish → 409 `FOUR_EYES_REQUIRED` listing `window_fn`, and
  nothing changes. Greg's publish succeeds, and the audit row lists the change set and
  `reviewed_by`. A draft with no function change is published by Amanda alone.
- AC-36.5.3 (`all`): any draft change by Amanda makes her publish → 409; Greg's succeeds. A template
  publish follows the same rule.
- AC-36.5.4: `FOUR_EYES_PUBLISH=maybe` → the API exits at start with
  "FOUR_EYES_PUBLISH must be off, code or all". `/api/auth/me` and `/api/health` report the mode.
- AC-36.5.5: importing or cloning a definition with function nodes records the importer as a draft
  author with `touched_function_code`; under `code` the importer cannot publish it alone.
- AC-36.5.6: request-level separation of duties is unaffected by the setting (Ivan still gets 403 on
  his own request, S6).

**US-36.6** As an author, I test functions safely.
- AC-36.6.1: Test run with typed inputs returns the outputs, fuel, duration, `runner_build` and up
  to 64 console lines. No request, variable or definition row changes; only the test-run audit row
  is written.
- AC-36.6.2: the 11th Test run in a minute by one user → 429 `RATE_LIMITED`. A Test run on a
  definition the caller cannot view → 404 (IDOR disguise).
- AC-36.6.3: a real run that logs 10,000 lines writes no server log line containing them (RT-14).

**US-36.8** As a designer, I route function failures by cause (D-59).
- AC-36.8.1: on `fuel_exhausted`, only `window_error` changes (to `"fuel_exhausted"`); `window_label`
  is unchanged and the node takes `error`. On `thrown` with `throw new Error("bad\u202Einput")`, the
  message target holds "badinput" (bidi removed, at most 200 characters).
- AC-36.8.2: a function input bound to the platform key `activity_start_at` receives its ISO value;
  binding a platform key not in the node's scope → V-58.
- AC-36.8.3: a function with no outputs → V-65.

**US-36.7** Endpoint test mandate (CLAUDE.md OWASP rule). For `PUT`, `test-run`, `simulate`,
`publish` and `GET /plan`, the suite has:
- positive and negative tests;
- fuzz tests of the graph DTO's new fields;
- bounds tests (§42.9 limits table);
- wrong state (published draft → 409 `DEFINITION_IMMUTABLE`);
- wrong session (expired → 401, another organisation → 404);
- OWASP A01, A03, A04, A05, A08 and A09 cases mapped in `docs/ac-traceability.md`.

The XSS case: a function output `<img src=x onerror=alert(1)>` renders as text in the Variables
panel and the trace.
