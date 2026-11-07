# Addendum 219 (part 11) — Variable scope, name resolution and data flow across embedded workflows

**Date:** 2026-11-09
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12; §0 maps every part).
- Part 4 `…-part-4-workflow-definition-and-editor.md` (§20–§26: embeds, templates, validation, DDL, interpreter).
- Part 5 `…-part-5-editor-stories.md` (ST-15–ST-24, ST-33).
- Part 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27 handlers, §29 coverage gate).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34, RC-01…RC-19).
- Part 9 `…-part-9-exception-runtime-and-cloning-detail.md` (§35 payloads, §36 cloning).
- **This part (§41, ST-34).**

**Why this part exists.** On 2026-11-09 the user asked three questions:
1. How does a node know which variables it has in scope, and how is that evaluated?
2. How do variables pass from the outer workflow into an embedded one?
3. How do return values pass back to the caller?

The user then asked to "plan properly how these variables flow in and out". Part 4 §21.5 names the
binding kinds (`in`, `out`, `ref`). It leaves open their timing, storage, concurrency and lifetimes
(per run, per lane and per call). This part closes those gaps.

**Precedence.** This part is normative for variable scope and data flow. Part 4 §21.4, §21.5, §22,
§23 and §25.1 summarise it and point here; where they differ, this part wins.

---

## §41 Variable scope and data flow

### §41.0 The short answer

> **Scope is static.** A node sees two sets of names: the variables declared in **its own
> definition**, and a fixed set of **platform variables** for its context (root, template or
> handler). `compile` builds this set once (`scope_table`). The editor's pickers, the validator and
> the evaluator all use that one table. Nothing is looked up by name at run time.
>
> **Values cross an embed boundary only through declared params.**
> - `in` is **copied in** when the call starts and is read-only inside the template.
> - `out` is **copied out** when the call leaves by a normal exit. For a per-item embed, the outputs
>   of all lanes are **reduced** into one value when the `completed` port fires.
> - `ref` is an **alias** to one caller variable, resolved at compile time. A lane may only add to
>   it; the parent applies each addition in order.

### §41.1 Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-44 | **Static lexical scoping.** Every expression operand references either a variable declared in the node's own definition (`wf_expressions.variable_id`) or a platform key (`wf_expressions.platform_var`). `mvp_engine::scope_table(&CompiledPlan, Context)` is the single source of what a node can see | A node knows its scope from where it sits, not from what has already run. Validation, pickers and evaluation cannot disagree |
| D-45 | **Platform variables come from an engine catalogue, not `wf_variables` rows.** They are `item_count`, `window_hours`, `activity_start_at`, `activity_end_at`, `item.*`, `lane.*`, `exc.*` and `substitute_available`, all from `mvp_engine::catalogue()`. They are read-only, never copied by cloning, and shown read-only in the editor | The platform owns these facts. As rows, a clone, import or edit could change their meaning. This also gives `exc.*` and `lane.*` a home: they did not fit the old `request`/`item` scope enum |
| D-46 | **Copy in, copy out, alias.** `in` is evaluated once per call and then frozen. `out` is copied only on a normal exit. `ref` resolves to one owning slot at compile time | Values are predictable, and caller and template share nothing that is not declared |
| D-47 | **Lanes add; the parent decides.** A lane writes a root request variable only through a `ref`, and only additively (`x := x ± e`, on int or decimal). The parent applies each delta in `seq` order and is the only writer of the authoritative value. It pushes new values only to the lanes whose template reads that variable | Additions commute, so concurrent lanes cannot lose an update. Plain assignment from concurrent lanes would |
| D-48 | **Fresh values per run, lane and call.** A plan run starts every request variable at its initial value. Every call starts each call variable at its initial value; a lane is a call. A replacement or supplementary lane is a new call, so it also starts fresh | Resubmit and reset re-evaluate everything (R6, R7). A substitute is a different physical item |

### §41.2 Scopes, owners and lifetimes

| Scope | Declared in | Readable by | Writable by | Lifetime and storage |
|---|---|---|---|---|
| `request` | a system root (`wf_variables.scope='request'`; roots only) | nodes of the root. A template reads one only through an `in` or `ref` param | root `set_variable` nodes, root fields, `out` bindings of root embeds, `ref` aliases (D-47) | one value per plan run: `request_variables (plan_run_id, variable_id, embed_call_id NULL)` |
| `call` | a template (`scope='call'`; templates only), including its params | nodes of that template and of its handlers | that template's `set_variable` nodes and fields, `out` bindings of embeds inside it, and `ref` aliases that resolve to it. `in` params are never writable | one value per call: `(plan_run_id, variable_id, embed_call_id)`. For a lane template, the call is the lane |
| `item.*` (platform) | catalogue | nodes of a `subject=item` template; the input bindings of a `per_item` embed | nobody | fixed from the asset at lane start; carried in `LaneInput` |
| `lane.*`, `exc.*`, `substitute_available` (platform) | catalogue | handler nodes only (part 6 §27.4) | nobody | computed per exception (part 6 §27.4) |
| root facts (platform) | catalogue | root nodes only | nobody; refreshed by activities (§41.8) | derived from the request, its items and the run; stored as `request_variables` rows keyed by `platform_var` so `/variables` keeps them (R8) |

**Every slot always holds a value; there is no unset state.**
- Request and call variables take their `init_*` value when their run or call starts.
- `in` params take their binding value, or their default.
- Platform facts exist before the run starts.

So an evaluation never fails on a missing value. A read that comes before any designer write is
caught statically instead (V-56).

### §41.3 Name resolution: how a node knows what is in scope

`compile` builds one `ScopeTable` per **context**. A node's context is derived from its own row
alone.

| Context | Determined by | Visible names |
|---|---|---|
| Root node | `wf_nodes.definition_id` is a system root | the root's `request` variables, root facts, aggregates (R2) |
| Template node | the definition is a template and `handler_id` is NULL | that template's `call` variables and params; `item.*` when `subject=item` |
| Handler node | `handler_id` is set (always inside a `subject=item` template) | as for a template node, plus `lane.*`, `exc.*` and `substitute_available` |
| Input binding of an embed | a `wf_embed_bindings` row of an `in` param | the **caller's** context. A `per_item` binding also sees `item.*` of the lane being started |
| Output or `ref` binding target | `wf_embed_bindings.caller_variable_id` | a writable variable of the caller's definition |

Rules:
- **Resolution is by ID, not by name.** Each `wf_expressions` operand stores a `variable_id` or a
  `platform_var`, so renaming a variable changes no expression. A composite foreign key
  `(definition_id, variable_id) → wf_variables (definition_id, id)` makes a cross-definition
  reference impossible.
- **Equal names do not collide.** A caller and a template may both declare `review_score`. They are
  different rows in different definitions, and only a binding connects them.
- **The editor never re-implements scope.** The graph DTO carries `scopes[{context, names[{ref,
  name, type, writable, origin}]}]`, built by `scope_table`. Every variable picker and the rule
  builder filter by the selected node's context. The server re-checks with V-20 and V-22.

### §41.4 Where a value lives at run time

`mvp-engine` addresses every value through a **slot**. Slots are resolved at compile time wherever
possible.

```rust
pub enum Slot {
    Request(VarId),              // root request variable; the parent workflow owns it
    Call(CallId, VarId),         // template variable of one call (a lane, or a single-embed call)
    Platform(PlatformVar),       // catalogue fact; read-only
}
pub struct Frame { template: DefId, parent: Option<CallId>, lane: Option<LaneId>,
                   vars: BTreeMap<VarId, Value>, aliases: BTreeMap<VarId, Slot> }  // aliases: ref params
fn read(state: &State, frame: &Frame, op: &Operand) -> Value   // pure; total over valid plans
```

| Host | Holds | Owns (authoritative for) |
|---|---|---|
| `BreakglassRequestWorkflow` (the parent) | the root snapshot; the frames of single embeds that run in the root; `last_seq` per lane | the `Request(_)` slots, and the call slots of calls that run in the parent |
| `LaneWorkflow` | the lane's frame; the frames of single embeds nested in the lane; the item facts; a **ref cache** of the request variables its template reads | its own `Call(_)` slots |
| Postgres | `request_variables` and the append-only `request_variable_changes`, written by `persist_settlement` and `persist_lane_settlement` | the durable record (R8: `/variables` still renders after Temporal retention) |

Evaluation reads only the host's own state (part 4 §21.4, "evaluation is pure"). A lane never reads
the parent's state directly; it reads its ref cache (§41.7).

### §41.5 Inputs (`in`): from the caller into the template

| Step | Single embed | Per-item embed |
|---|---|---|
| When | at `EnterEmbed`, in the same `settle` that moves the token onto the embed | at each lane start: the initial lanes at entry (R1), and later every replacement or supplementary lane |
| Evaluated by | the host of the embed node | the parent, because it starts the lanes |
| Environment | the caller's context (§41.3) and its current values, including aggregates at a root | the caller's context plus `item.*` of that lane |
| Result | written to the new frame's param slot with `source=binding` | as for single, in the lane's frame; sent in `LaneInput.in_values` |
| Unbound param | a required param without a binding is invalid (V-48). An optional one takes its `init_*` default | same |
| Inside the template | read-only (V-22) | same |
| Caller changes later | not seen: the value was copied | not seen. A replacement lane evaluates afresh and may see a newer value |
| Audit and locks | each caller variable read is a consumed input (`request_rule_inputs`, R6) | same, recorded per lane |

Example: the `review` embed binds `required_score := 6`, a literal. Inside `dual_review`, the rule
`score_ok` reads the template's own `required_score`, never a variable of HRIS.

### §41.6 Outputs (`out`): from the template back to the caller

**Single embed.**
- Inside the template, `out` params are ordinary call variables. `set_variable` nodes, fields and
  nested `out` bindings may write them. Each starts at its `init_*` (E2), so it has a value at
  every exit.
- On a **normal exit**, `ExitEmbed` copies every bound `out` param to its caller slot, and the token
  then leaves by the exit's outcome port. Both happen in one `settle` and one transaction. Each
  copy is recorded with `source=binding` and its `binding_id` in `request_variable_changes`.
- An unbound `out` param is discarded. The inspector shows it as "Not used by this caller".
- **An early end writes nothing.** The call can end early through a propagating reject or return
  (E4), an admin reset, cancellation after a sibling's reject, or auto-expire. None of these copies
  outputs, so the caller keeps its values.
- **Re-entry is a new call.** A second visit creates a new frame (for example after `amend`, which
  also starts a new run). Its outputs overwrite the caller's values when it exits.

**Per-item embed.**
- Each lane's frame holds its own `out` values. They become **final** when the lane is terminal: at
  its `exit`, or when a resolution ends it (`resolved_exception`).
- The lane sends them with its terminal milestone, `Milestone(Terminal{outcome, outputs})`, so the
  values and the milestone arrive in one event. Excluded lanes send `Excluded` and no outputs.
- The parent **reduces once**, when `completed` fires, over every lane that sent `Terminal`.
  Supplementary lanes are included. The result is written before the `completed` port is taken.
- `unfulfilled` reduces nothing, so the caller keeps its values.
- A variable written by a per-item reduction may be read only on paths that start at `completed`,
  or after a join that includes it (V-55). On the `issued` path it still holds its old value.

| Reducer | Param type | Caller type |
|---|---|---|
| `sum` | int, decimal | same as the param |
| `count_true` | bool | int |
| `any`, `all` | bool | bool |
| `min`, `max` | int, decimal, timestamp | same as the param |

`completed` needs at least one terminal lane, so a reduction never runs over zero lanes. A `sum`
that overflows is an evaluation error (part 4 §21.4).

### §41.7 References (`ref`): shared state between caller and template

- **Aliases resolve at compile time.** A `ref` param resolves to its final owning slot: a root
  `Request(var)`, or a `Call(call, var)` of an enclosing template. When a `ref` is bound to an outer
  template's own `ref` param, `compile` follows the chain, so nesting never adds a hop.
- **Same host: direct.** When the alias and the writer run in the same workflow, reads and writes go
  straight to the slot. This covers a single embed in the root, and a nested embed whose alias
  stays inside the same lane. `settle` is serial, so any assignment form is allowed, subject to
  V-54.
- **Lane to parent: additive deltas only (D-47, V-53).** When a lane writes an alias to a root
  request variable:
  1. The write must be `x := x + e` or `x := x - e`, on int or decimal. `compile` turns it into
     `VariableDelta{var, add: e}`, negated for minus. The self-reference does not count as a read
     of `x`.
  2. The lane adds the delta to its ref cache at once (read your own writes). It records the delta
     as pending and sends `lane_event{seq, VariableDelta}`.
  3. The parent drops duplicates by `seq` (part 9 §35.3). It applies deltas in arrival order through
     `persist_settlement` and increments the variable's `revision`. An overflow is a parent
     evaluation error (part 4 §21.4).
- **Parent to lane: pushed only to readers.** `compile` records which root request variables each
  lane template **reads** (`ref_reads`).
  - After the parent changes such a variable, it signals every live lane whose template reads it:
    `ref_values{values[(var, value, revision)], acked_seq}`.
  - `acked_seq` is the receiving lane's last applied `seq`. The lane's view becomes the pushed value
    plus its own pending deltas with a higher `seq`, so read-your-writes survives the push.
  - When a lane reads such a value, it records the `revision` it saw in `request_rule_inputs`.
- `LaneInput.ref_cache` seeds the cache at lane start with the current values of `ref_reads`.
- The parent may also write an alias while lanes run. Those writes are pushed the same way.

Example (S1): three lanes run `count_received` (`received_counter := received_counter + 1`). No
other node of the template reads `received_counter`, so `ref_reads` is empty and the parent pushes
nothing. Two receipts within 10 ms arrive as two deltas, and the parent writes `items_received` as
1, then 2 (AC-20.3.1).

### §41.8 Platform variables and when they refresh

| Key | Type | Context | Refreshed |
|---|---|---|---|
| `item_count` | int | root | at submit, approver edit and resubmit; also on replacement and write-off (it counts live items) |
| `window_hours`, `activity_start_at`, `activity_end_at` | decimal, timestamp | root | at submit, approver edit and resubmit |
| `item.credential_type`, `item.asset_tag` | enum, text | `subject=item` templates; per-item input bindings | fixed at lane start. A replacement lane gets its substitute's facts |
| `lane.issued`, `lane.replacement_count` | bool, int | handlers | on every lane milestone |
| `exc.type`, `exc.severity`, `exc.phase`, `exc.detected_state`, `exc.reported_by_role` | enum | handlers | when the exception opens |
| `substitute_available` | bool | handlers | at `investigate` entry, and re-checked inside the activity (part 6 §27.4) |

The activity that changes a fact returns the new values. The workflow passes them to `settle` as
`Event::FactsChanged(PlatformFacts)`. R6 still applies: an approver edit that would change a fact
that a rule or binding has already consumed is refused with 409 `RULE_INPUT_LOCKED`.

### §41.9 Initialisation points

| Event | Values set |
|---|---|
| A plan run starts (submit, resubmit, admin reset) | every root `request` variable to its `init_*` value; the root facts |
| A single-embed call starts | every `call` variable of the template to its `init_*` value; then the `in` params from their bindings |
| A lane starts (initial, replacement or supplementary) | the lane frame, as for a call; the `item.*` facts; the `ref_cache` |
| A field is submitted on `act` | the field's variable, in the acting node's context (V-27) |
| A call ends normally | single embed: the caller slots of bound `out` params. Per item: the lane's outputs, held for reduction |
| `completed` fires | the per-item reductions |

### §41.10 Worked traces (seeded HRIS v1)

**S8, high-risk review.** Calls are numbered `C1`, `C2` and so on.

| # | Event | Writes (slot ← value, source) |
|---|---|---|
| 1 | Bob submits 4 items | root facts `item_count`=4, `window_hours`=6; root variables at their init values |
| 2 | Claire approves `ops_l1` with field `risk_level=high` | `Request(risk_level)` ← high (field) |
| 3 | `assess` runs | `Request(high_risk)` ← true (set_variable); consumed inputs `risk_level`, `item_count`, `window_hours` |
| 4 | `review` enters `dual_review` v1 | new call C1; C1's variables at their init values; `C1.required_score` ← 6 (binding, literal) |
| 5 | Frank approves `sec_review` with field `security_score=3`; then `add_sec` | `C1.security_score` ← 3 (field); `C1.review_score` ← 3 (set_variable) |
| 6 | Grace approves `log_review` with `logistics_score=2`; then `add_log` | `C1.logistics_score` ← 2; `C1.review_score` ← 5 (additive on a parallel branch, so V-54 passes) |
| 7 | `score_ok` evaluates `C1.review_score(5) >= C1.required_score(6)` → false | rule evaluation recorded |
| 8 | exit `failed` | `Request(review_score)` ← 5 (binding, out); the token leaves `review` by `failed` and reaches `rejected` |

**S1, lanes.**

| # | Event | Writes |
|---|---|---|
| 1 | The token enters `items`; 3 lanes start | lane calls L1–L3; `item.*` from SP-01, HT-01 and LT-01; `pin_reset_done` ← false in each |
| 2 | In the HT-01 lane, `is_token` is true and `pin_reset` is entered | new call C(L2·1); `pin_done` ← false |
| 3 | Grace completes `reset_pin` with `pin_done=true`; exit `done` | `C(L2·1).pin_done` ← true; `L2.pin_reset_done` ← true (binding, same host) |
| 4 | Bob confirms receipt on L1 and L3 | two `VariableDelta{items_received, add 1}`; the parent writes 1, then 2 |
| 5 | The last lane returns and `completed` fires | `Request(pins_reset)` ← `count_true(pin_reset_done)` over L1–L3 = 1 (reduction) |

### §41.11 Validation

These codes extend part 4 §22. The same pure `validate` emits them.

| Code | Sev | Rule |
|---|---|---|
| **V-53** | error | A write from a lane-hosted node to a `ref` alias that resolves to a root `Request` slot must be additive (`x := x ± e`) on int or decimal. An `out` binding or field of a lane-hosted node may not target such an alias |
| **V-54** | error | When several writers target the same slot on different branches of one split, before its join, every one of them must be additive. Writers here are nodes and `out` bindings. Otherwise the last writer would win silently |
| **V-55** | error | A caller variable written by a per-item reduction may be read only by nodes reachable from `completed`, or downstream of a join that includes `completed` |
| **V-56** | warning | `READ_BEFORE_WRITE`: a designer variable is read on some path from the definition's `start` or `entry` with no writer before it, so its initial value is used. Params, platform variables and the self-reference of an additive write are exempt. The simulator flags the same read |

Extensions to existing rules:
- **V-20:** operands resolve through the `scope_table` of the node's context (§41.3). A
  `platform_var` key must be allowed in that context. Per-item input bindings may read `item.*`.
- **V-22:** platform variables, constants and `in` params are never writable.
- **V-27:** a field writes a writable variable of the acting node's context.
- **V-48:** reducers and types must pair as in §41.6.

### §41.12 Data model and payloads (no JSONB)

| Change | Detail |
|---|---|
| `wf_variables.scope` | `request` (root definitions only) or `call` (templates only), enforced by a trigger against the owner on `workflow_definitions`. The unique key becomes (definition_id, name). `is_system_computed` and designer `item` variables are dropped (D-45) |
| `wf_expressions` | `item_attribute` is renamed `platform_var` varchar(40) and covers every catalogue key. A CHECK requires exactly one of `variable_id`, `platform_var` and `lit_type` on operand kinds. A composite FK keeps `variable_id` inside the expression's definition |
| `request_variables` | request_id, plan_run_id, variable_id NULL, platform_var NULL, embed_call_id NULL, value_*, revision. Unique (plan_run_id, variable_id, platform_var, embed_call_id) NULLS NOT DISTINCT. A CHECK requires exactly one of variable_id and platform_var. A trigger requires `embed_call_id` to be NULL exactly for `request`-scope variables and root facts. `lane_id` is dropped (the lane's call row carries it) |
| `request_variable_changes` | adds `plan_run_id`, `platform_var NULL`, `binding_id NULL` (set when `source=binding` or `reduction`) and `lane_event_seq NULL`. `source` gains `reduction` and `lane_delta` |
| `request_rule_inputs` | variable_id NULL, platform_var NULL, embed_call_id NULL, revision; CHECK exactly one of variable_id and platform_var. `lane_id` is dropped |

Payloads extend part 9 §35.3. Every new field carries `#[serde(default)]`.

```rust
pub struct LaneInput { pub lane_id: LaneId, pub call_id: CallId, pub item: ItemFacts,
    pub in_values: Vec<(VarId, Value)>, pub ref_cache: Vec<(VarId, Value, Revision)> }
pub struct RefValues { pub values: Vec<(VarId, Value, Revision)>, pub acked_seq: u32 }  // signal parent → lane
pub enum LaneEventKind { VariableDelta { var: VarId, add: Num }, Milestone(MilestoneEvent) }  // Num: Int | Decimal
pub enum MilestoneEvent { /* … */ Terminal { outcome: LaneOutcome, outputs: Vec<(VarId, Value)> } }
pub enum Effect { /* … */ SetVariable { slot: Slot, value: Value, source: ChangeSource } }
pub enum ChangeSource { Init, System, SetVariable, Field, Binding(BindingId), Reduction(BindingId), LaneDelta(u32) }
```

Engine functions (extending part 8 §30.6):
- `scope_table(&CompiledPlan, Context) -> ScopeTable`.
- `dataflow(&CompiledPlan) -> DataflowReport`: for each variable, its declaration, writers, readers,
  bindings and `ref_reads`.

Both are pure and covered by the 100% coverage gate (part 6 §29).

### §41.13 Editor, request page and API

- **Editor, Variables tab.** Rows are grouped into "Declared here" and "Platform (read only)". Each
  row shows **Read by** and **Written by**, from `dataflow`. A param row also shows **Bound at**:
  the embeds that bind it in the organisation's published roots.
- **Editor, embed inspector.** Each binding row reads as a sentence:
  - Inputs: "`required_score` ← 6 (copied when the call starts)".
  - Outputs: "`review_score` → `review_score` (copied on a normal exit)", or, for a per-item embed,
    "(count_true over lanes, when completed)".
  - Ref: "`received_counter` ⇄ `items_received` (shared; lanes can only add)".
- **Rule builder and pickers** list only the `scopes[]` entries of the selected node (§41.3).
- **Simulation trace** lists binding, reduction and delta steps with their slots, in the format of
  §41.10.
- **Request page, Variables panel.**
  - Groups: "Request", one group per call ("Dual review v1 · call 1", "Lane 2 · HT-01"), and
    "Platform".
  - Each change row names its source: field, set, "binding from …", "lane delta #n" or reduction.
- **API.**
  - `GET /api/admin/workflow-definitions/:defId` includes `scopes[]`.
  - `GET /api/admin/workflow-definitions/:defId/dataflow` returns the dataflow report.
  - `GET /api/requests/:id/variables?run=` returns values and changes, grouped by scope and call.
  - `act` refuses a field that is not writable in the acting node's context with 422
    `VALIDATION_FAILED`, and changes no row.

### §41.14 Effect on other parts

- **Part 4.**
  - §21.4 and §21.5 summarise the binding rules and point here.
  - §23.1 `wf_variables` and §23.2 `request_variables` follow §41.12.
  - §25.1 lists the platform facts as "platform" and the template variables as `call`.
  - §25.1 adds the HRIS root variable `pins_reset` (int, init 0), the `count_true` reduction of
    `items` out `pin_reset_done`.
  - `item.pin_reset_done` becomes `pin_reset_done`, an `out` param of `standard_custody_lane`.
- **Part 6** §27.4 and §27.11: the `exception` scope is platform (D-45). `item.loss_report_done`
  becomes the call variable `loss_report_done` of `standard_custody_lane`.
- **Part 9** §36.1: platform variables are never copied; they come from the catalogue.
- **Addendum 220**, Prompts 9, 21, 23, 26 and 27, shows the groups, sentences, codes and trace steps
  above.

---

## Stories

### ST-34 Variable scope and data flow

**US-34.1** As a designer, I see only the variables a node can use.
- AC-34.1.1
  - G: HRIS v1 root and `standard_custody_lane` v1.
  - W: the rule builder opens on `is_high_risk`.
  - T: it lists `risk_level`, `high_risk`, `review_score`, `items_received`, `pins_reset` and the
    root facts. It does not list `pin_reset_done`, `item.*` or `exc.*`. Opened on `is_token`, it
    lists `item.credential_type`, `item.asset_tag`, `pin_reset_done` and `received_counter`, and not
    `risk_level`.
- AC-34.1.2: a crafted PUT whose expression references a variable of another definition, or uses
  `exc.type` outside a handler, returns `issues[]` with V-20, and publish is refused.
- AC-34.1.3: renaming `high_risk` changes no `wf_expressions` row (resolution is by ID).

**US-34.2** As the platform, I copy inputs in once.
- AC-34.2.1 (S8): `C1.required_score` is 6 with `source=binding`. A later change to a caller
  variable does not change it. The binding's caller reads appear in `request_rule_inputs`.
- AC-34.2.2: a template `set_variable` that targets `required_score` (an `in` param) is V-22.
- AC-34.2.3: an unbound required `in` param is V-48. An unbound optional one takes its default.

**US-34.3** As the platform, I copy outputs out only on a normal exit.
- AC-34.3.1 (S8): exit `failed` writes `review_score`=5 to the root in the same transaction as the
  taken edge `review.failed`.
- AC-34.3.2: Frank rejects `sec_review`, a propagating reject. The request is `rejected`, the root
  `review_score` keeps its prior value, and no change row with `source=binding` exists.
- AC-34.3.3 (S1): `pins_reset` is written once, as 1, when `completed` fires, with
  `source=reduction`. An excluded (replaced) lane does not count; a supplementary lane does.
- AC-34.3.4: a node on the `issued` path that reads `pins_reset` is V-55.

**US-34.4** As the platform, I keep shared counters exact across lanes.
- AC-34.4.1 (S1): two receipts within 10 ms give `items_received` 1, then 2. There are two change
  rows, each with its `lane_event_seq`, and no lost update (AC-20.3.1).
- AC-34.4.2: `received_counter := 5` inside the lane template is V-53. So is an `out` binding of
  `pin_reset` that targets `received_counter`.
- AC-34.4.3: a lane template that reads `received_counter` in a rule receives `ref_values` after
  each parent change and records the `revision` it read. A lane that applied a delta not yet acked
  still sees its own delta after the push. A template that does not read the variable receives no
  push.
- AC-34.4.4: a replayed or duplicated `lane_event` (same `seq`) writes nothing.

**US-34.5** As a designer, I am warned about unsafe or suspicious flows.
- AC-34.5.1: two parallel branches that assign `review_score := security_score` and
  `review_score := logistics_score` give V-54. The seeded additive form passes.
- AC-34.5.2: a rule that reads `security_score` on a path that skips `sec_review` raises V-56 in
  the Validation tab and in the simulator. The seeded `add_sec` (`review_score := review_score +
  …`) raises no V-56.

**US-34.6** As any viewer, I can trace where each value came from.
- AC-34.6.1: `/variables` for S8 groups C1 under "Dual review v1 · call 1" and shows
  `review_score: 5 ← binding from dual_review.review_score (exit failed)`.
- AC-34.6.2: after a resubmit, run 2 starts every request variable at its initial value. The run
  selector still shows run 1's values unchanged.

**US-34.7** Security (OWASP and IDOR).
- AC-34.7.1: `GET /api/requests/:id/variables` for a request the caller cannot view returns 404, the
  same response as an unknown ID (IDOR disguise).
- AC-34.7.2: an `act` whose field names a platform variable, a variable of another call or another
  definition, or an `in` param, returns 422 `VALIDATION_FAILED`. No row and no history event is
  written (mass assignment).
- AC-34.7.3: an int field beyond i64, a decimal beyond numeric(18,4), or a text value over 200
  characters returns 422. Evaluation overflow follows part 4 §21.4.
- AC-34.7.4: `GET …/dataflow` and `scopes[]` follow the read permissions of the definition, and
  return 404 otherwise.
