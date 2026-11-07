# Addendum 219 (part 4) — Per-system workflow definitions, the template library, the actualised request plan, the graph editor and the interpreter

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12).
- Part 2 `…-part-2-stories.md` (§14, ST-0–ST-14).
- Part 3 `…-part-3-verification-and-prompt.md` (§13, §15–§19).
- **This part (§20–§26).**
- Part 5 `…-part-5-editor-stories.md` (§14 continued, ST-15–ST-24).
- Part 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27–§29).
- Part 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34).
- Part 9 `…-part-9-exception-runtime-and-cloning-detail.md` (§35–§37, ST-31).
- Part 10 `…-part-10-exception-ui-and-walkthroughs.md` (§38–§40, ST-32).
- Part 11 `…-part-11-variable-scope-and-data-flow.md` (§41, ST-34): variable scope and data flow; normative over §21.4, §21.5 and §23 on those points.
- Appendix A `…-appendix-a-illustrative-example.md`.

**Why this part exists.** The user asked for five things:
- A **graphical workflow editor** in the demo, pre-populated with a break-glass workflow that
  exercises every engine feature.
- Connector rules: every node's *in* is joined to another node's *out*, except the single initial
  node; the converse holds for terminals (there may be many).
- **One request = one SR / reference / activity on one system.** Every reviewer may reject
  (terminal) or return to the requester for amendment.
- An answer to the tension that **a system's workflow is defined without knowing how many items a
  requester will draw**, so the system's workflow and the request's actualised workflow differ.
- **Reuse.** Any eligible workflow in the organisation's template library can be embedded in
  another workflow as one node, so that a review pattern or a custody lane is authored once.

**Precedence.** This part is normative. Where it conflicts with parts 1–3, this part wins. §26 lists
every clause it supersedes.

---

## §20 Definition versus actualised request: expressing and reconciling the tension

### §20.0 The short answer

> A system's workflow is a **cardinality-free graph with one per-item embed**. The per-item embed
> names a published **lane template** from the organisation library and carries call-site settings,
> not a count. A request's workflow is a **plan**: the pinned root version and the template versions
> it pins, with the per-item embed **expanded into one lane per item at the moment the token enters
> it**. The two are reconciled by four things: late binding, joins that range over the lane
> collection rather than a fixed number of in-edges, lanes that change only through custody
> resolutions, and a per-run version pin.

### §20.1 Three distinct objects

| Object | What it is | Cardinality | Stored in | Lifetime |
|---|---|---|---|---|
| **Workflow definition** | Versioned, in two classes (D-21): a **system root** (one per system) and an **organisation template** (embeddable, §21.5). A draft is editable; a published version is immutable | Cardinality-free. A root holds exactly one per-item embed with call-site bounds `min_items..max_items` (1..10) and a join policy. A template holds no count | Postgres, normalised (§23.1) | Until retired; never deleted (7-year audit, §2.7) |
| **Request plan** (actualised workflow) | One **plan run** per request run (a resubmit or admin reset starts a new run). Each run pins one root version and, through it, the template versions it embeds. It holds embed calls, node instances, lanes, taken edges and variable values | Concrete. N lanes created at per-item embed entry; +1 lane per approved replacement; lanes excluded by write-off | Postgres (§23.2) | 7 years |
| **Engine state** | Tokens, call frames, armed timers, join counters, child handles, decision lock | Mirrors the current plan run | Temporal (§23.3) | Run plus 7 days; history archived (D-18) |

The definition answers "what *may* happen on this system". The plan answers "what *did* happen to
this request". Temporal answers "what is this request waiting for right now".

### §20.2 Reconciliation rules

| Rule | Statement | Why |
|---|---|---|
| R1 Late binding | The lane count is bound when the token **enters the per-item embed**, not when the definition is written and not at submit. Until then, items can change: by approver edit (US-5.6), or by return and resubmit | Reviewers may add or remove items. The plan must match the approved items, not the first draft |
| R2 Aggregates before the per-item embed | Nodes of the root before the per-item embed may reference items only through aggregates: `item_count`, `any(item.attr op value)`, `all(item.attr op value)`, `count(item.attr op value)`. They may not reference "item 2". A template sees an aggregate only as a bound input (E6) | The graph shape before the per-item embed must not depend on N |
| R3 Per-item logic inside the lane template | Logic that differs per item (for example "tokens need a PIN reset") lives in the lane template and branches on **lane variables** (`item.*`) | Each lane runs the same template with its own bindings |
| R4 Collection joins | The per-item embed's out-ports are evaluated over the lane collection, not by counting in-edges. Ports: `issued`, `completed`, `unfulfilled`. Policy: `all_live` (default, which is D-13), `all`, or `k_of_n` with a fixed k | A static join needs a known in-degree; a per-item join does not |
| R5 Lanes change only by resolution | After per-item embed entry, a lane is **added** only by an approved replacement (D-12, §9.3) and **excluded** only by an approved write-off. Each change recomputes the per-item join. The call-site flags `allow_replacement` and `allow_write_off` gate both | Keeps the plan auditable. N changes only through an owner-approved decision |
| R6 Consumed inputs lock | When a rule, set-variable node or embed input binding has evaluated, the variables it read are recorded (§23.2). An approver edit that would change one (for example `item_count` after the high-risk rule ran) is refused with 409 `RULE_INPUT_LOCKED`. The reviewer must **return** the request instead, which starts a new run and re-evaluates | Prevents a path that was decided on stale inputs |
| R7 Per-run version pin | Each plan run pins the root version that was **current-published** when the run started. The pin covers the root and every template version it pins (`wf_embed_pins`, D-42). Publishing a new root or template version never changes a running plan. A resubmit or admin reset starts a new run and re-pins the then-current root (OP-7) | Same as today's snapshot rebuild on resubmit (§3.1). Definition changes need no deploy |
| R8 One rendering rule | The request graph is the pinned root. A single embed is drawn as a collapsible group with a breadcrumb chip; the per-item embed is drawn once per lane. Taken edges are highlighted and untaken edges greyed. A run selector shows earlier runs. It renders from Postgres (`GET /api/requests/:id/plan`), not from Temporal | The view stays correct after Temporal retention |
| R9 Bounds at the boundary | `min_items..max_items` and `allowed_credential_types` come from the per-item embed of the pinned (or, at submit, the current) root. They are enforced at submit, approver edit and resubmit (422) | The definition constrains what a requester may draw |
| R10 Simulation before publish | The editor dry-runs a draft root, with its pinned templates, for any N (1..max_items), chosen item attributes and chosen outcomes, and shows the plan that would result. It also dry-runs a template on its own with chosen inputs. It uses the same engine code as production (§23.3) | Lets the designer see the actualised workflow for N items before any request exists |

### §20.3 Worked example (HRIS v1, §25)

- **Definition.** 16 top-level nodes, two of which are embeds. `review` is a single embed of
  `dual_review` v1 (12 nodes). `items` is the per-item embed of `standard_custody_lane` v1 (11
  nodes), which itself embeds `pin_reset` v1 (3 nodes). No item count appears anywhere.
- **Bob submits SP-01, HT-01, LT-01.** `item_count=3`. The plan run is `#1`, pinned to HRIS v1 and
  its three template versions. The high-risk rule reads `item_count` (R2) and records it as
  consumed (R6).
- **Aaron's approver edit** removes LT-01. It is allowed while no rule has consumed `item_count`.
  Once the rule has evaluated, the same edit gets 409 `RULE_INPUT_LOCKED`.
- **Owner approves; the token enters `items`.** Lanes L1 (SP-01) and L2 (HT-01) are created (R1).
  L2 is a token, so its lane takes the `pin_reset` branch and enters the nested embed (R3).
- **HT-01 is lost and replaced by HT-02.** Lane L3 is added with `origin=replacement`, and L2 is
  excluded (R5). `all_live` now waits for L1 and L3 (R4).
- **The graph** shows three lane bands. L2 is struck through, with a "replaced by" edge to L3 (R8).

### §20.4 Decisions added by this part

| ID | Decision | Rationale |
|---|---|---|
| D-20 | **Graphical workflow editor** (Next.js and React Flow) is part of the MVP. It is the only place a workflow is authored. It replaces the chain builder and the custody stage configuration screens | Requested by the user |
| D-21 | **Two workflow definition classes.** A **system root** has one family of versions per system. An **organisation template** (`wf_templates`) has one family per template and is embeddable; a root never is. Both are versioned (draft, published, retired), stored in normalised tables (no JSONB) and share one editor. A published version is immutable (trigger). There is at most one live draft per owner (system or template). "Current" means the highest published version that is not retired | Repo rule: no JSONB for business data. Versions make pinning possible (R7) |
| D-22 | **A generic interpreter replaces the hard-coded parent workflow.** The pure crate `mvp-engine` holds `compile`, `validate`, `settle` and `simulate`. The same code runs in the Temporal workflow, in the API's validate/simulate endpoints and in replay tests | One semantics everywhere. Simulation cannot drift from execution |
| D-23 | **Actualisation by late-bound per-item embed expansion** with reconciliation rules R1–R10 (§20.2) | Answers the user's tension question |
| D-24 | **One request = one system, one `sr_number` (the SR / reference), one activity window.** Items: `min_items..max_items` assets of that system. A second live request with the same `(system_id, sr_number)` gets 409 `DUPLICATE_SR` (OP-6) | Requested by the user ("to avoid complications") |
| D-25 | **Every approval and owner-approval node has the out-ports `approve`, `return` and `reject`, all mandatory.** At a root, `reject` connects only to a Rejected terminal and `return` only to the Amend node. Inside a template they connect to `exit` nodes that propagate to the root (E4). A reject or return on any parallel branch interrupts the whole request: sibling active instances become `cancelled` | Requested by the user. Enforced by the editor (V-12, V-13, V-46) and again by the server |
| D-26 | **Platform interrupts are not drawn.** Auto-expire (D-10), tombstone auto-cancel and admin reset (D-14) apply to every root. The editor shows them as a fixed, read-only legend and as a locked `Expired` terminal | They are lifecycle guarantees that a designer must not be able to remove |
| D-27 | **Typed workflow variables.** Variables are declared in the definition with scope `request` (a root) or `call` (a template), and type `int`, `decimal`, `bool`, `text`, `timestamp` or `enum`. Values live in `request_variables` (typed columns), with an append-only `request_variable_changes` history. They are written only by activities: by a set-variable node, a reviewer-filled field, a binding, a reduction or a lane delta. Platform facts (`item_count`, `item.*`, `exc.*` and others) come from an engine catalogue, not from rows (part 11 D-44 … D-48) | The engine needs variables, fork mutation and rules (Appendix A features) without JSONB |
| D-39 | **A workflow is reused as one node.** The node kind `embed` places a pinned template version in another workflow. Its `cardinality` is `single` (the template runs once, inline) or `per_item` (one lane per item). The template's inputs become the node's inputs; its exits and outputs become the node's out-ports and outputs. There is no inline per-item section: custody stages and exception handlers live in the lane template | One mechanism instead of a special region. A review pattern or custody lane is authored once |
| D-40 | **Organisation template library.** A template belongs to one organisation, is created, versioned, published and retired in the same editor (template mode), and can be embedded only by workflows of that organisation. Edit and publish: `full_admin`, or an `org_admin` whose scope includes the organisation (D-38). Anyone who can view an embedding root may open the pinned template read-only. Others get 404 | Keeps authority where D-38 already puts it |
| D-41 | **Interface and eligibility contract E1–E6** (§21.5). A template is embeddable only if it passes; `validate` reports each rule and publishing a failing template is refused | The whole template must behave like one node: declared inputs in, declared outcomes and outputs out |
| D-42 | **Explicit pinning.** A draft pins an explicit published template version per embed. Publishing a definition freezes the closure of its pins in `wf_embed_pins`. Publishing a template never changes any root. An "Upgrade" action re-pins in the draft and re-validates bindings | Reproducible plans (R7); no silent behaviour change |
| D-43 | **Nesting.** Depth is at most 3 template levels below the root. No template is reachable from itself (by template identity, across versions). A nesting path holds at most one `per_item` embed; in the MVP that is the root's own | Bounded, acyclic, and one lane expansion per path |

---

## §21 Node and port catalogue, and connector rules

### §21.1 Connector model

- Every node has **at most one in-connector**. It accepts any number of edges, except the Start node
  of a root and the `entry` node of a template, which have no in-connector.
- Each node kind has a fixed set of **named out-ports**. Split and switch nodes have a configurable
  list. Terminals and `exit` nodes have none.
- **An out-port carries exactly one edge** (unique `(from_node, from_port)`). Fan-out is expressed
  only with a Split node, so that "every out connector is joined" has one meaning.
- Edges never cross an embed boundary. An `embed` node has an in-connector and out-ports like any
  other node; the template it names is a separate graph. Inside a template, `entry` plays the role
  of `start` and `exit` the role of a terminal, and the same connector rules apply.

### §21.2 Node kinds

| Kind | In | Out-ports | Configuration | Status projection |
|---|---|---|---|---|
| `start` (root, exactly 1) | none | `submitted` | — | `submitted` |
| `approval` | 1 | `approve`, `return`, `reject` | assignee team `(cluster, level)`; optional fields; optional `sla_hours` plus `escalate_to_team`; `allow_override` | `under_review` once the token is past the first approval |
| `owner_approval` (root) | 1 | `approve`, `return`, `reject` | primary or backup owner, or an active delegate (fixed); optional fields | `under_review` (owner stage); `approve` → `approved` |
| `amend` (root, exactly 1) | 1 (edges only from `return` ports and propagating exits) | `resubmitted` | re-entry: its edge goes to an approval node | `returned_to_vendor` |
| `rule` | 1 | `true`, `false` | a condition expression (§21.4) | — |
| `switch` | 1 | one per case plus `default` | an ordered list of conditions | — |
| `set_variable` | 1 | `next` | target variable, expression | — |
| `expression` ("f(x)"), `function` ("JS") | 1 | `next`, `error` | FX lines, or JavaScript source with typed inputs and outputs (part 12 §42.3) | — |
| `split` | 1 | `branch_1`…`branch_n` (n 2..5) | branch labels | — |
| `join` | 1 (≥2 edges) | `joined` | policy `all`, `any`, or `k_of_n` with k | — |
| `wait` | 1 | `elapsed` | `duration_hours`, or `until` a timestamp variable plus an offset | — |
| `notify` | 1 | `next` | target: PA of primary owner, submitter, or a team. It records a notification row; there is no delivery in the MVP | — |
| `task` | 1 | `done` | a human task: assignee is the submitter, any snapshot reviewer, or a team; optional fields | `task.kind=close` sets `closed` |
| `embed` | 1 | `single`: one port per normal exit outcome of the pinned template. `per_item`: `issued`, `completed`, `unfulfilled` | pinned template version; `cardinality`; input and output bindings (§21.5). `per_item` adds the call-site settings: `min_items`, `max_items`, `allowed_credential_types`, join policy (R4), `allow_replacement`, `allow_write_off`, `max_replacements_per_item`, `allow_post_issuance_replacement`, investigate team and due hours | `per_item`: entry → custody starts; `issued` → `credential_issued`. `single`: none |
| `terminal` (root, ≥1) | 1 | none | `terminal_kind`: `completed`, `rejected`, `closed_unfulfilled`. The platform-owned `expired` terminal is locked (D-26) | `completed` and `closed_unfulfilled` → `closed`; `rejected` → `rejected` |
| `entry` (template, exactly 1) | none | `next` | declares nothing itself; the inputs are the template's `in` and `ref` variables | — |
| `exit` (template, ≥1) | 1 | none | `exit_outcome` key (1..8 distinct) and `exit_kind` ∈ {`normal`, `reject`, `return`} | — |

**Per-item embed port semantics.**
- The per-item embed emits **exactly one** of `issued` or `unfulfilled`, once.
- `completed` is emitted only after `issued`, when every lane is terminal.
- `issued` fires when the join policy holds over the live lanes:
  - `all_live` is D-13.
  - `all` means any exclusion leads to `unfulfilled`.
  - `k_of_n` means k lanes have crossed the issuance milestone.
- `unfulfilled` fires when the live set becomes empty before issuance. For `all` it also fires on
  the first exclusion.

**Single embed port semantics.** The embed leaves through the port named by the exit outcome the
template reached. Exits with `exit_kind` `reject` or `return` are not ports: they are propagating
(§21.5).

### §21.3 Template node rules

A template has a `subject`: `none` (no item) or `item` (runs for one item). Which kinds may appear
depends on it.

| Kind | `subject=none` | `subject=item` | Notes |
|---|---|---|---|
| `entry`, `exit` | yes | yes | exactly one `entry`; `exit` as in §21.2 |
| `approval` | yes | yes | `reject` → `exit(reject)`, `return` → `exit(return)` (E4) |
| `rule`, `switch`, `set_variable`, `task`, `wait`, `notify`, `split`, `join` | yes | yes | `item.*` references only when `subject=item`. A lane may write `item.*` and, through `ref` params, caller request variables (the parent serialises writes, §23.3) |
| `custody_stage` | no | yes | `stage` ∈ {prepare, initiate_handover, confirm_receipt, initiate_return, verify_return, return_to_inventory}. The team comes from an agency cluster of the same organisation, or is the submitter for `confirm_receipt` and `initiate_return`. Has `due_in_hours`. Exactly one of each stage, in this canonical order (§9.1). `confirm_receipt` carries `is_issuance_milestone=true`. Only in a template embedded `per_item` (E5) |
| `embed` | `single` of a `subject=none` template | `single` of a `subject=item` or `subject=none` template | nesting limits in D-43 |
| `start`, `owner_approval`, `amend`, `terminal`, `expired` | no | no | root-only (E3) |

Exceptions, disputes and resolutions (§9.1–§9.3) are **lane-level platform behaviour**. Any
custody stage can enter `exception_open`. Exception handlers belong to the definition of the
`subject=item` lane template (part 6 §27); they are not drawn on the root. The lane band in the
request view shows the exception side node, as §12 already specifies.

### §21.4 Expressions (rules, switch cases, set-variable and embed bindings)

- An expression is a tree (`wf_expressions`, §23.1) with these node kinds:
  - **Comparison:** `cmp(left, op, right)` with `op` ∈ {`>`, `<`, `>=`, `<=`, `==`, `!=`}. Each
    operand is a variable reference or a typed literal. This is a superset of Appendix A's
    `left_var operator right_var`.
  - **Logic:** `and`, `or`, `not`.
  - **Arithmetic** (int and decimal only): `add`, `sub`, `mul`.
  - **Aggregates** (R2): `item_count`, `any(item.attr op lit)`, `all(…)`, `count(…)`.
  - **Built-ins:** `now()` and `hours_between(ts, ts)`.
- **Evaluation is pure.**
  - It reads only the variable snapshot inside workflow state.
  - `now()` is the workflow's deterministic clock.
  - Division is absent (the `expression` node's FX language allows it; part 12 D-51).
  - Overflow is an evaluation error (in `expression` and `function` nodes a fault takes the `error` port instead; part 12 D-51, D-56). The node instance becomes `failed`, the request shows "Workflow
    error", and an admin reset is required.
- **Type checking happens at validate time** (V-20…V-23). Comparing values of different types, or
  ordering a `bool` or `enum`, is invalid.
- The operator is **chosen in the editor** and stored in the definition. Changing it requires a new
  version. A presenter shows Appendix A's "change the operator and re-evaluate" by editing a draft
  and re-simulating (R10).

### §21.5 Embeddable templates: interface and eligibility

A template is **embeddable** when the whole of it behaves like one node: it takes declared inputs
and ends in declared outcomes and outputs. `validate` checks the rules below (D-41); the editor's
Eligibility tab lists each as pass or fail; publishing a failing template is refused with 422
`TEMPLATE_NOT_ELIGIBLE`. A system root is never embeddable.

| Rule | Statement | Code |
|---|---|---|
| E1 Entry | Exactly one `entry` node (port `next`, no in-connector). The inputs are variables with `param_direction=in` (typed, `is_required`, optional default) or `ref` (by reference to a caller request variable; writes go through the parent serially as `lane_event variable_delta`, keeping AC-20.3.1) | V-44 |
| E2 Exits | At least one `exit`, each with an `exit_outcome` key; 1..8 distinct outcomes. Outputs are `param_direction=out` variables; each has an init value, so every exit defines it | V-44 |
| E3 Root-only kinds | No `start`, `owner_approval`, `amend`, `platform_interrupt`, locked `expired` or root `terminal` | V-45 |
| E4 Approvals | `approval` is allowed. `reject` wires to `exit(exit_kind=reject)` and `return` to `exit(exit_kind=return)`. These **propagating exits** are not ports: they raise the root's reject or return at the call site (D-25), resolving to the root's `rejected` terminal and `amend` node. They propagate through nested embeds. A template with propagating exits cannot be embedded `per_item` and cannot sit downstream of `owner_approval` | V-46 |
| E5 Subject | `item.*`, `custody_stage` and exception handlers appear only in `subject=item` templates. A `subject=none` template embeds `single`. A `subject=item` template embeds `per_item` on a root (its custody stages and handlers then apply), or `single` inside another `subject=item` template (it inherits the item and may hold no custody stages or handlers) | V-47 |
| E6 Scope | Template-local variables are private per call. Only params cross the boundary; aggregates (R2) reach a template only as bound inputs. Scope resolution, copy-in, copy-out and `ref` aliasing: part 11 §41 | V-20, V-48, V-53…V-56 |

Connector rules V-01…V-10 apply with `entry` and `exit` in place of `start` and `terminal`. V-17
applies to a `subject=item` template with custody stages.

**Bindings** (full data-flow rules in part 11 §41.5–§41.7).
- An **input binding** gives each `in` param a caller variable, a typed literal or an expression
  (rule builder, §21.4). A required input needs a binding or a default. Binding evaluation is a read
  recorded for R6. It is evaluated once, when the call (or lane) starts, and is read-only inside.
- An **output binding** (`single`) names a writable caller variable of the same type. It is copied
  when the call leaves through a normal exit, never on a propagating exit, reset or cancel.
- A **per-item** output binding also names a reducer: `sum`, `count_true`, `any`, `all`, `min` or
  `max`. It is reduced once, over the lanes that reached a terminal, when `completed` fires (V-55).
- A `ref` binding aliases a caller variable. From a lane it may only be added to (V-53).
- Ports are derived: the editor previews them and `wf_node_ports` is reseeded on every pin or
  upgrade (V-48 re-checks bindings).

**Pinning (D-42).** A draft pins `embed_definition_id`, a **published**, non-retired version of a
template of the same organisation. The inspector shows "v2 available · Upgrade" when a newer
version exists (V-52). Upgrade re-pins the draft and re-validates bindings; a param removed or
retyped surfaces as a V-48 error.

**Nesting (D-43).** Depth is at most 3 template levels below the root (HRIS v1 → `standard_custody_lane`
→ `pin_reset` is depth 2). Cycles are checked by template identity across versions (V-50).

---

## §22 Validation catalogue

A single pure function, `mvp_engine::validate(def) -> Vec<Issue>`, is called by:
- the editor (debounced, on every change);
- `POST …/validate`;
- `POST …/publish`, which refuses any `error`;
- `POST …/simulate`, which refuses any `error`.

It validates a root together with its pinned templates, or a template on its own. Each `Issue`
carries `{code, severity, node_key?, edge_id?, message}`. The UI never reimplements these rules; it
only renders the issues. Codes in **bold** come straight from the user's rules. Codes marked † are
ported from `sidequest/src/validation.ts`.

| Code | Sev | Rule | Prior art |
|---|---|---|---|
| **V-01** | error | A root has exactly one `start` node | † `start-count` |
| **V-02** | error | `start` has no inbound edge | † `entry-not-start` |
| **V-03** | error | A root has at least one `terminal` node | — |
| **V-04** | error | Every non-start, non-entry node has ≥1 inbound edge ("in joined to an out") | † `join-no-inbound` |
| **V-05** | error | Every out-port of every non-terminal, non-exit node has exactly one edge ("out joined to an in"), including every derived port of an `embed` | † `no-outcomes` |
| **V-06** | error | `terminal` and `exit` have no outbound edge | — |
| V-07 | error | Edge endpoints exist; no self-loop; no edge crosses an embed boundary | † `edge-to-unknown`, `edge-from-unknown` |
| V-08 | error | Every node is reachable from `start` (templates: from `entry`) | — |
| V-09 | error | Every node can reach a terminal (templates: an `exit`) | † `no-reachable-end` |
| V-10 | error | In a root the only cycles are those through `amend`. A template has no cycle | † `goto-mixed-rewind` |
| V-11 | error | A root has exactly one `amend` node; its inbound edges come only from `return` ports; its out-edge targets an `approval` node | † `hold-without-goto` |
| **V-12** | error | Every `approval` and `owner_approval` has `approve`, `return` and `reject` wired (D-25) | — |
| **V-13** | error | In a root, `reject` connects only to a `terminal(rejected)` and `return` only to `amend`. In a template, `reject` connects only to `exit(reject)` and `return` only to `exit(return)` | — |
| V-14 | error | A root has exactly one `owner_approval`; it lies on every path from `start` to the per-item embed (dominance) | — |
| V-15 | error | The per-item embed is not inside a split–join pair; all three of its ports are wired; `1 ≤ min_items ≤ max_items ≤ 10`; `k_of_n` has `1 ≤ k ≤ max_items`. Count and placement: V-51 | — |
| V-16 | error | No `rejected` terminal is reachable from any per-item embed port (issuance cannot be revoked, §9.3) | — |
| V-17 | error | A `subject=item` template with custody stages: each stage appears exactly once and in canonical order on every path; one issuance milestone | — |
| V-18 | error | Splits and joins are properly nested. Each `join` has ≥2 inbound edges from the same split; for `k_of_n`, `1 ≤ k ≤ n` | † `quorum-bounds`, `goto-cross-branch` |
| V-19 | error | A `task(kind=close)` lies on the `issued` path, and the `completed` terminal is reachable only via a join of that task with the per-item embed's `completed` | — |
| V-20 | error | Expressions reference declared variables of their own definition, or platform keys allowed in the node's context (part 11 §41.3); `item.*` is used only inside `subject=item` templates; R2 aggregates only in a root | † `operand-unknown-node`, `bad-field-ref` |
| V-21 | error | Expressions type-check (§21.4) | — |
| V-22 | error | The target of a `set_variable` is writable: not a platform variable, not a constant, not an `in` param, and of a matching type | — |
| V-23 | error | Every enum literal is a declared value of that enum | — |
| V-24 | error | Assignee teams are live, belong to the definition's organisation, and have the right cluster type (vendor or agency for approvals; agency for custody stages); none is a `pa` cluster | † `no-assignees` |
| V-25 | error | `sla_hours`, `due_in_hours`, `duration_hours` and `investigate_due_hours` are in 1..720 | † `timeout-nonpositive` |
| V-26 | error | `node_key` matches `^[a-z][a-z0-9_]{0,39}$` and is unique within its definition; labels are 1..80 characters | † `node-id-blank`, `node-id-duplicate` |
| V-27 | error | A field that is `required` on an outcome is a writable variable of the node's scope | † `mandate-bad-ref` |
| V-31 | error | The platform-locked `expired` terminal is present and unmodified in a root (D-26); its removal is reported as `PLATFORM_TERMINAL_LOCKED` | — |
| V-28 | warning | An approval node has the same team as the node immediately before it (likely a duplicate review) | — |
| V-29 | warning | A `rule` whose condition is a constant (always true or always false) | — |
| V-30 | warning | No `sla_hours` on the first approval node | — |
| V-44 | error | Template interface (E1, E2): exactly one `entry`; 1..8 distinct exit outcomes; every `in` and `ref` param is declared and typed; every `out` param has an init value | — |
| V-45 | error | A template contains a root-only kind (E3) | — |
| V-46 | error | Propagating exits (E4): `reject` and `return` wire to the matching `exit` kind; a template holding them, directly or through a nested embed, is not embedded `per_item` and not downstream of `owner_approval`; the root has the `amend` and `rejected` nodes they resolve to | — |
| V-47 | error | Subject rules (E5): `item.*`, `custody_stage` and handlers only in `subject=item`; the embed's cardinality matches the pinned template's subject; no custody stage or handler in a `single`-embedded template | — |
| V-48 | error | Bindings are complete and typed: every required `in` param bound or defaulted; `out` bindings target a writable caller variable of the same type; a reducer on every `per_item` out binding; `ref` bindings target request variables | — |
| V-49 | error | A pin is a published, non-retired version of a template of the same organisation, eligible for the embed's cardinality | — |
| V-50 | error | No template is reachable from itself (by template identity, across versions); depth ≤ 3 | — |
| V-51 | error | A system root has exactly one `per_item` embed, placed directly on the root, whose template has custody stages and one issuance milestone; no template embeds `per_item`, so no path holds a second | — |
| V-52 | warning | A newer published template version is available, or bindings drifted after an upgrade | — |
| V-53…V-56 | — | Data-flow rules (part 11 §41.11): additive-only `ref` writes from lanes; concurrent writers; reads of a per-item reduction off the `completed` path; read before write (warning) | — |

---

## §23 Data model and interpreter

### §23.1 Definition tables (replace `system_workflow_steps` and `system_custody_stage_configs`)

All tables follow §6 conventions: UUIDv7, `created_at`/`updated_at`, liveness-aware foreign keys,
and no JSONB.
- A trigger **rejects every UPDATE and DELETE** on rows that belong to a definition whose status
  is `published` or `retired`. The single permitted change is the status column moving from
  published to retired.
- Saving a draft replaces its child rows in a single transaction.

| Table | Key columns | Constraints |
|---|---|---|
| `wf_templates` | organization_id, template_key `^[a-z][a-z0-9_]{0,39}$`, name(80), description(500), subject (`none, item`) | unique (organization_id, template_key); liveness-aware |
| `workflow_definitions` | system_id NULL, template_id NULL, version int NULL (set at publish), status (`draft, published, retired`), revision bigint (draft autosave counter), based_on_version, created_by, published_by/at, retired_by/at, change_notes(2000), content_sha256 | CHECK exactly one of system_id, template_id; unique (system_id, version) and unique (template_id, version), each partial; one `draft` per owner (partial unique); `content_sha256` is computed at publish over the canonical serialisation |
| `wf_variables` | definition_id, name `^[a-z][a-z0-9_]{0,39}$`, scope (`request, call`), type, param_direction NULL (`in, out, ref`), is_required, is_constant, init_int, init_decimal numeric(18,4), init_bool, init_text(200), init_ts | unique (definition_id, name); `request` only on a root, `call` only on a template (part 11 §41.12); exactly the init column matching `type` (CHECK), none for `ref`; `param_direction` only on a template definition; `is_required` only on `in` |
| `wf_variable_enum_values` | variable_id, ordinal, value(40) | unique (variable_id, value) |
| `wf_nodes` | definition_id, node_key, kind, label(80), pos_x, pos_y, assignee_cluster_id, assignee_level, assignee_kind (`team, owner, submitter, any_reviewer`), allow_override, sla_hours, escalate_to_cluster_id, escalate_to_level, join_policy, join_k, wait_duration_hours, wait_until_variable_id, wait_offset_minutes, notify_target, task_kind (`generic, close`), custody_stage, due_in_hours, is_issuance_milestone, terminal_kind, exit_outcome, exit_kind (`normal, reject, return`), embed_template_id, embed_definition_id, embed_cardinality (`single, per_item`), embed_unresolved, embed_hint(120), min_items, max_items, embed_policy, embed_k, allow_replacement, allow_write_off, max_replacements_per_item, allow_post_issuance_replacement, investigate_cluster_id, investigate_level, investigate_due_hours, condition_expression_id, set_variable_id, set_expression_id | unique (definition_id, node_key); CHECKs tie every column to its kind. `embed_definition_id` references a published definition of `embed_template_id`; `embed_unresolved` (clone placeholder) leaves both NULL and blocks simulate and publish (V-42 `EMBED_UNRESOLVED`, part 6 §28.4) |
| `wf_node_ports` | node_id, port_key(40), ordinal, label(80), case_expression_id NULL | unique (node_id, port_key). Fixed ports are seeded per kind; `split` and `switch` ports are user-defined; `single` embed ports are derived from the pinned template's normal exits |
| `wf_edges` | definition_id, from_node_id, from_port_key, to_node_id, label(80) | **unique (from_node_id, from_port_key)** (§21.1); FK (from_node_id, from_port_key) → `wf_node_ports`; both nodes belong to `definition_id` |
| `wf_expressions` | definition_id, parent_id NULL, ordinal, kind, cmp_op, arith_op, aggregate, variable_id, platform_var(40), lit_type, lit_int, lit_decimal, lit_bool, lit_text, lit_ts | tree; the root is referenced from `wf_nodes`, `wf_node_ports` or `wf_embed_bindings` |
| `wf_node_fields` | node_id, variable_id, label(80), required_on_port_key NULL, min_int, max_int, max_length | unique (node_id, variable_id) |
| `wf_allowed_credential_types` | embed_node_id (a `per_item` embed), credential_type | unique pair |
| `wf_embed_bindings` | node_id (an `embed`), param_name(40), param_variable_id NULL (in the pinned template; NULL only while the embed is unresolved, then resolved by `param_name`, unmatched names are V-48), caller_variable_id NULL, caller_expression_id NULL, reducer NULL (`sum, count_true, any, all, min, max`) | unique (node_id, param_name); exactly one of caller_variable_id, caller_expression_id (expression only for `in`); reducer only for `per_item` `out` |
| `wf_embed_pins` | root_definition_id (the published definition at the top of the tree), template_definition_id, depth (1..3) | written at publish; unique (root_definition_id, template_definition_id); immutable like any child row |

**At most one live draft per owner (system or template).**
- Create a draft with `POST …/workflow/draft {fromVersion}` (system) or
  `POST /api/admin/workflow-templates/:id/draft {fromVersion}` (template), which deep-copies that
  version.
- Publish validates (§22) and computes `content_sha256`. It then sets `status='published'`,
  `version = max(version)+1` within the owner, and writes `wf_embed_pins` for the closure of the
  draft's pins. The draft row becomes that version: draft rows have no version number until publish.
- Retire is allowed only if a newer published version exists. Retiring also stops new pins; a draft
  that still pins a retired version gets V-49.
- Running plans are unaffected (R7). Publishing a template never changes any root (D-42).
- **A system without a current published root cannot accept submissions** (409
  `SYSTEM_WORKFLOW_INVALID`, the same code as AC-7.1.3).

### §23.2 Request plan tables

| Table | Key columns | Notes |
|---|---|---|
| `request_plan_runs` | request_id, run_no, definition_id (pinned root), workflow_run_id, started_reason (`submit, resubmit, admin_reset`), started_at, ended_at, end_kind | unique (request_id, run_no). The current run is the max run_no. Pinned templates are read from `wf_embed_pins` of that root |
| `request_embed_calls` | plan_run_id, embed_node_id, parent_call_id NULL, lane_id NULL, template_definition_id, entered_at, exit_outcome NULL, left_at NULL | one row per `single` call and per lane call of a `per_item` embed; `parent_call_id` nests calls; unique (plan_run_id, embed_node_id, parent_call_id, lane_id) |
| `request_node_instances` | plan_run_id, node_id, lane_id NULL, embed_call_id NULL, seq int, status (`active, completed, cancelled, failed`), outcome_port_key, actor_id, acting_as, notes(2000), decided_via_override, entered_at, left_at, op_ref | `seq` is the entry order within the run. A node in a loop gets a new instance per visit. `node_id` may belong to a pinned template; `embed_call_id` says which call. **Replaces `request_approval_steps`**: a view of that name projects approval and owner instances as `(ordinal=seq, status, decided_by, decision_notes, decided_via_override)` for reports and §3.1 vocabulary |
| `request_taken_edges` | plan_run_id, edge_id, lane_id NULL, embed_call_id NULL, taken_at, op_ref | drives the highlighting in R8 |
| `request_plan_lanes` | plan_run_id, embed_node_id, template_definition_id, lane_no, request_item_id, custody_case_id, origin (`initial, replacement`), replaces_lane_id, excluded_at, excluded_reason | unique (plan_run_id, lane_no); unique (request_item_id) |
| `request_variables` | request_id, plan_run_id, variable_id NULL, platform_var NULL, embed_call_id NULL, value_int, value_decimal, value_bool, value_text(200), value_ts, revision | unique (plan_run_id, variable_id, platform_var, embed_call_id) (NULLS NOT DISTINCT); typed CHECK. Call variables, including lane variables, are keyed by their call (E6, part 11 §41.12) |
| `request_variable_changes` | request_id, plan_run_id, variable_id, platform_var, embed_call_id, old_*/new_* typed, source (`init, set_variable, field, system, binding, reduction, lane_delta`), binding_id NULL, lane_event_seq NULL, node_instance_id, actor_id, op_ref, created_at | **append-only** (INSERT/SELECT grant only) |
| `request_rule_evaluations` | node_instance_id, result_port_key, rendered_expression(500), op_ref | e.g. `review_score(5) >= required_score(6) → false` |
| `request_rule_inputs` | evaluation_id, variable_id NULL, platform_var NULL, embed_call_id NULL, revision, rendered_value(200) | the consumed inputs used by R6, including embed input bindings |
| `request_notifications` | node_instance_id, target_kind, target_id, created_at | `notify` nodes (no delivery) |

**Additions to `breakglass_requests`:**
- `plan_revision bigint`, incremented by every plan-changing activity. Clients send it as
  `expectedRevision`.
- A partial unique index `(system_id, sr_number) WHERE status NOT IN ('rejected','closed')`, for
  D-24 and OP-6.

`custody_case_stage_snapshots` (§6) is filled when a lane is created, from the `custody_stage` nodes
of the pinned lane template (`template_definition_id` on the lane).

### §23.3 The interpreter

**`mvp-engine` (pure, no I/O)** exposes:
- `compile(def_rows) -> CompiledPlan`: resolves the pinned tree of a root (or a template on its own)
  into `CompiledPlan { root, templates: BTreeMap<DefId, CompiledTemplate> }`, with keys resolved and
  adjacency and dominance data built.
- `validate(&CompiledPlan) -> Vec<Issue>` (§22).
- `settle(&CompiledPlan, &State, Event) -> (State, Vec<Effect>)`.
- `simulate(&CompiledPlan, SimInput) -> SimTrace`.

`State` holds:
- active tokens: `BTreeMap<NodeInstanceId, Token{node, lane, call}>`;
- call frames: `BTreeMap<CallId, Frame{template, parent, lane, bindings}>`;
- join arrivals;
- the variable snapshot;
- lanes;
- `revision`.

`Effect` is one of:
- `EnterNode`, `LeaveNode{port}`, `TakeEdge`, `SetVariable`, `RecordRule`, `Notify`,
  `EnterEmbed{call, node, template}`, `ExitEmbed{call, outcome}`,
  `StartLanes{items}`, `ArmTimer`, `CancelTimer`, `CancelInstance`, `ProjectStatus`, `EndRun{kind}`.

`settle` runs every automatic node (rule, switch, set_variable, split, join, notify, a `wait`
whose time has already passed, and embed bookkeeping) until each token waits on one of:
- a human (approval, owner_approval, amend, task);
- a timer;
- a lane milestone.

**Embeds in `settle`.**
- A **`single` embed runs inline** in the host's `settle`. The host is the parent workflow, or a
  `LaneWorkflow` when the embed sits in a lane template.
  - `EnterEmbed` evaluates the input bindings into a new call frame (recorded as consumed inputs,
    R6) and puts the token on the template's `entry`.
  - Nodes of the template settle like any other; a human task inside waits like any other.
  - A normal `exit` emits `ExitEmbed`, copies the output bindings to the caller and leaves through
    the port named by the exit outcome.
  - A propagating exit raises the root's reject or return interrupt: sibling active instances
    become `cancelled` (D-25).
- A **`per_item` embed** uses `StartLanes` and one `LaneWorkflow` per lane, which interprets the lane
  template. Its nested `single` embeds run inline in the lane's own `settle`.

This is the `settle` concept from `sidequest/src/engine.ts`, ported to Rust. The ported prior art is
the condition AST and the JoinSpec/quorum semantics in `sidequest/src/spec.ts`, and the "skipped"
lane semantics, which become `cancelled` here.

**`BreakglassRequestWorkflow` becomes a thin interpreter host** (rewrites §7.1).

*Start.* The activity `load_plan(definition_id)` returns the `CompiledPlan` of the pinned root and
every template it pins. It is recorded in history, so replay never re-reads the database. The
`PLAN_FORMAT` constant is bumped (part 8 RC-15). On `Resumed`, `load_projection` also returns the
`State`.

*Update `act`.* `act{node_instance_id, outcome, fields{var: value}, notes?, expected_revision, override}`
replaces `step_decision`, `owner_decision` and `close`. Its validator checks:
- the instance is active;
- `outcome` is one of the node's ports;
- `expected_revision` equals the current revision;
- the decision lock is free;
- the actor is not the submitter, for approval and owner kinds (D-6);
- notes are present for `return` and `reject`;
- required fields are present and typed.

*Handler.* The handler calls `settle`, then a single activity `persist_settlement(op_ref, effects)`
in one transaction. That transaction writes:
- embed calls, node instances, taken edges, variables and their changes, rule evaluations and
  notifications;
- the status projection;
- the semantic audit rows;
- `revision+1`.

The audit actions are those of §7.3, plus `workflow.rule_evaluated`, `workflow.variable_set`,
`workflow.node_cancelled`, `workflow.embed_entered` and `workflow.embed_exited` (template key,
version, `embed_call_id`, exit outcome). It also re-checks eligibility in-transaction (D-5): a team
member for `approval`; owner or delegate for `owner_approval`; the submitter for `amend` and
requester tasks.

*Effects that need more activities.* `StartLanes` calls `instantiate_custody` and then starts one
child per lane. `EndRun` completes the workflow, or continues-as-new for `amend`, resubmit and
reset.

*Unchanged.* `approver_edit` and `admin_reset` keep the shape in §7.1. `approver_edit` is
additionally checked against R6. `resubmit` is the `act` on the Amend instance with the new
content. It re-pins the root and its templates (R7) and continues-as-new.

**`CustodyCaseWorkflow` becomes `LaneWorkflow`** (rewrites §7.2). It interprets the lane template
from the same `CompiledPlan` (passed as input) for one lane. The custody state machine (§9.1) and
exception handling (§9.2, §9.3) are unchanged.

*Automatic lane nodes.* Rule, task, set_variable, wait, notify and single-embed nodes inside the
lane run in the child.

- **Call-scope writes.** The lane writes its own call variables, through
  `persist_lane_settlement`. `item.*` is read-only.
- **Request-scope writes.** Only through a `ref` alias, and only additively (part 11 D-47). The lane
  **signals the parent** with
  `lane_event{lane_id, seq, kind: variable_delta{var, add} | milestone{issued|excluded|replaced|terminal{outputs}}}`. The parent applies these one at a time, in arrival
  order, through `persist_settlement`, so concurrent lanes never race on a request variable. This
  is how Appendix A's "parallel branches mutate a shared variable" is realised safely.
- **Per-item out bindings** are reduced by the parent once, when `completed` fires, over the
  outputs that lanes sent with `terminal` (part 11 §41.6). The parent pushes `ref_values` only to
  lanes whose template reads the variable (§41.7).

**Determinism** (unchanged from §7).
- No I/O in `mvp-engine`.
- `BTreeMap` only.
- Time comes from the workflow clock.
- Variables are copied into state from activity results only.
- Engine *code* changes use `patched()`. Definition and template changes need no deploy, because
  each run pins its versions and the `CompiledPlan` is in history.

---

## §24 Editor UI and API

### §24.1 Routes and layout

| Route | Content |
|---|---|
| `/admin/systems/[id]/workflow` | Version list (current published, draft, history with publisher and time, and `content_sha256` prefix); "Edit draft" / "New draft from vN"; "Simulate"; "Diff"; "Export" |
| `/admin/systems/[id]/workflow/edit` | **The editor** (below), root mode |
| `/admin/templates` | The organisation template library: name, key, subject, interface summary (inputs, exits), current version, "Used by" (roots and templates that pin it); "New template" |
| `/admin/templates/[id]/workflow` | Version list of one template, with the same actions as a system |
| `/admin/templates/[id]/workflow/edit` | The same editor in **template mode** |
| `/requests/[id]` | The graph now renders `GET /api/requests/:id/plan` (R8). It adds a **Variables panel** (current values, plus a per-variable change history with actor and node) and a run selector. Single embeds render as collapsible groups with a breadcrumb chip |

**Editor layout** (React Flow `@xyflow/react` 12, custom node types, one per kind in §21.2):
- **Left: palette.** Drag a node kind onto the canvas. `start`, `amend`, `owner_approval` and the
  root's per-item embed are disabled once present. In template mode the palette swaps Start, Owner
  approval, Amend and Terminal for Entry and Exit, and `custody_stage` is enabled only when the
  template's subject is `item`. **Embedded workflow** opens a picker of published templates of the
  organisation; ineligible ones are greyed with the failed rule (for example "E5: needs a
  `subject=item` host") and eligibility is judged for the chosen cardinality.
- **Centre: canvas.**
  - Each node shows its label, kind icon, assignee chip, and one handle per port. Port handles are
    labelled and coloured: `approve`/`true` green, `return` amber, `reject`/`false` red, others grey.
  - An `embed` node shows `name · vN`, a cardinality badge (`single`, or "×N per item (min..max)"),
    chips for its inputs and outputs, and "Open", a read-only drill-in with breadcrumbs such as
    `HRIS v1 › standard_custody_lane v1 › pin_reset v1`.
  - Mini-map and zoom controls. Auto-layout button (dagre) that preserves pinned positions.
- **Right: inspector** for the selected node or edge:
  - Assignee pickers, filtered to the definition's organisation and to valid cluster types.
  - Fields, SLA and escalation, join policy and k, wait configuration, task kind.
  - For an embed: template, version with **Upgrade**, cardinality, call-site settings (bounds,
    allowed credential types, replacement and write-off flags, investigate team) for `per_item`, an
    **Inputs** table, an **Outputs** table (reducer column for `per_item`) and a derived ports
    preview.
  - A **visual rule builder**: pick a variable, pick an operator, then pick a variable or a literal;
    with AND/OR groups. There is no free-text expression entry. Input bindings use the same builder.
- **Bottom: tabs.**
  - **Validation**: live issues from §22. Clicking one focuses the node or edge.
  - **Variables**: declare, type, initial value, enum values; in template mode also each param's
    direction and whether it is required. Read-only platform rows; Read by and Written by (part 11 §41.13).
  - **Interface** and **Eligibility** (template mode): inputs, exits, subject; E1–E6 as pass or fail.
  - **Handlers** (template mode, `subject=item`): exception handlers (part 6 §27).
  - **Simulation** (R10, §24.3).
  - **Diff**: against any published version; node, edge and variable changes listed and highlighted
    on the canvas.

**Connection rules enforced while dragging.**
- React Flow's `isValidConnection` blocks the following:
  - a second edge from a port;
  - an edge into `start` or `entry`;
  - an edge out of a terminal or an `exit`;
  - in a root, `reject` going anywhere but a rejected terminal and `return` anywhere but `amend`;
  - in a template, `reject` going anywhere but `exit(reject)` and `return` anywhere but `exit(return)`;
  - edges crossing the embed boundary.
- The server re-validates everything (V-codes). The UI check is a convenience, not the control.

**Saving and publishing.**
- Autosave runs 1 s after the last change and sends the full draft with `expectedRevision`. A stale
  save returns 409 `STALE_DRAFT` and shows "Someone else changed this draft — reload".
- Undo and redo hold 50 steps and are client-side.
- "Publish" is disabled while any error-severity issue exists. It asks for change notes (1..2000).

**Accessibility.**
- Every action has a keyboard path: palette items are buttons.
- The inspector's "Connect *port* to…" dropdown is an alternative to dragging.
- Nodes are focusable and have ARIA labels of the form "Approval: Ops L1, 3 ports, 1 unconnected".

**Permissions.**
- Systems: view is `full_admin`, an `org_admin` whose scope includes the system's organisation
  (D-38), and the system's owners (read-only). Edit and publish is `full_admin`, or an `org_admin`
  whose scope includes the system's organisation (capability `systems_assets`, D-38).
- Templates (D-40): view, edit and publish by `full_admin` or an `org_admin` whose scope includes the
  template's organisation. Anyone who can view an embedding root may open a pinned template
  read-only (drill-in), including a retired version.
- Everyone else gets 404.

### §24.2 API additions (append to §11)

All routes use the §11 envelope. Mutating routes require `X-Idempotency-Token`. All are scoped to
the organisation; cross-organisation access returns 404.

| Method & path | Notes |
|---|---|
| `GET /api/admin/systems/:id/workflow` | `{current: {definitionId, version}, draft: {definitionId, revision}?, versions[]}` |
| `POST /api/admin/systems/:id/workflow/draft {fromVersion}` | creates the draft. 409 `DRAFT_EXISTS` if one is live |
| `GET /api/admin/organizations/:orgId/workflow-templates` | the library: key, name, subject, interface summary, current version, used-by count |
| `POST /api/admin/organizations/:orgId/workflow-templates {templateKey, name, description, subject}` | creates the template and an empty draft |
| `GET /api/admin/workflow-templates/:id` | the template with its versions and "used by" list |
| `POST /api/admin/workflow-templates/:id/draft {fromVersion}` | creates the draft. 409 `DRAFT_EXISTS` if one is live |
| `GET /api/admin/workflow-templates/eligible?forDefinition=&cardinality=` | published templates of the definition's organisation, each with `eligible` and the failed rule codes |
| `GET /api/admin/workflow-definitions/:defId/interface` | params, exits, subject, the E1–E6 results and the derived ports |
| `GET /api/admin/workflow-definitions/:defId` | the full graph DTO: nodes, ports, edges, variables, expressions, fields, bindings and pins. Serves both classes |
| `PUT /api/admin/workflow-definitions/:defId {expectedRevision, graph}` | draft only; whole-graph replace. 409 `STALE_DRAFT`; 409 `DEFINITION_IMMUTABLE` if the definition is published; 422 `EMBED_PIN_INVALID` or `EMBED_BINDING_INVALID` for structural errors. The response includes `issues[]` |
| `POST /api/admin/workflow-definitions/:defId/validate` | `issues[]` |
| `POST /api/admin/workflow-definitions/:defId/simulate {items[], variables{}, script[]}` | `SimTrace` (§24.3). 422 if issues contain errors |
| `POST /api/admin/workflow-definitions/:defId/publish {changeNotes}` | 409 `DEFINITION_INVALID` (with `issues[]`); 422 `TEMPLATE_NOT_ELIGIBLE` for a template failing E1–E6; audit `workflow_definition.published` |
| `POST /api/admin/workflow-definitions/:defId/retire` | 409 if it is current and no newer version exists |
| `DELETE /api/admin/workflow-definitions/:defId` | discards the draft only (soft delete) |
| `GET /api/admin/workflow-definitions/:defId/diff?against=:version` | structural diff |
| `GET /api/admin/workflow-definitions/:defId/export` | file download (`application/json`, canonical serialisation, with `content_sha256`; pins appear as `(template_key, version)`). Not stored |
| `POST /api/admin/systems/:id/workflow/import` (multipart, ≤1 MiB) | parses the file into a **new draft** (normalised rows). Teams are resolved by `(cluster name, level)` and templates by `(template_key, version)` within the organisation; anything unresolved → 422 with paths |
| `GET /api/requests/:id/plan?run=` | the actualised graph: nodes, embed calls, lanes, edges, instance statuses, taken edges and runs (R8). Visibility rules of §8 apply |
| `GET /api/requests/:id/variables?run=` | current values plus change history, grouped by request, call and platform (part 11 §41.13) |
| `POST /api/requests/:id/nodes/:nodeInstanceId/act` | Temporal Update `act`. Replaces `step-decision`, `owner-decision` and `close`. Body: `{outcome, fields?, notes?, expectedRevision, override?}` |

New error codes: `STALE_DRAFT`, `DRAFT_EXISTS`, `DEFINITION_IMMUTABLE`, `DEFINITION_INVALID`,
`RULE_INPUT_LOCKED`, `DUPLICATE_SR`, `ITEM_BOUNDS`, `NODE_NOT_ACTIVE`, `OUTCOME_NOT_ALLOWED`,
`ALREADY_DECIDED_IN_SPLIT`, plus:
- `TEMPLATE_NOT_ELIGIBLE` (422): publishing a template that fails E1–E6, or pinning one that is not
  eligible for the embed's cardinality;
- `EMBED_BINDING_INVALID` (422): a binding names a param the pinned interface lacks, or a variable
  of another definition;
- `EMBED_PIN_INVALID` (422): a pin is not a published version of the named template in the same
  organisation.

New `decision_reason` value: `rule_rejected` (a `rule` or `switch` route reached a rejected
terminal). New audit action: `request.rule_rejected`.

### §24.3 Simulation

**Input:**
- `items[]`: 1..max_items synthetic items with `credential_type`, bounded by the per-item embed's
  `max_items`. For a `subject=item` template simulated on its own, exactly one item.
- Initial variable overrides. For a template simulated on its own, `variables{}` supplies its
  declared inputs.
- `script[]`: an ordered list of `{node_key, embed_path?, lane_no?, outcome, fields?}`. `embed_path`
  is the list of embed node keys from the root, needed when a key repeats in nested calls. A
  `lane_event` entry injects `exception → write_off` or `replacement`.

**Output (`SimTrace`):**
- The step-by-step list of `(effect, state digest)`, including `EnterEmbed` and `ExitEmbed`.
- The final actualised graph, in the same DTO as `/plan`.
- The variables after each step.
- Every rule evaluation with its rendered expression.
- The terminal reached, or "waiting at …".

**Constraints:**
- Simulation runs `mvp-engine` in-process.
- It writes nothing to the database and never contacts Temporal. A test asserts that no rows change
  and no workflow starts.
- The editor animates the trace: the active node is blue, completed nodes are green, and taken edges
  are highlighted. These are the Appendix A visual conventions, plus §12's colours.

---

## §25 The seeded demo workflow (HRIS v1), the template library and feature coverage

### §25.1 Variables

Root variables have scope `request`; template variables have scope `call` and are private per call
(E6). Rows marked *platform* come from the engine catalogue, not `wf_variables` (part 11 D-45).

| Name | Declared in | Scope | Type | Init | Source or param role |
|---|---|---|---|---|---|
| `item_count` | any root | platform | int | — | live count of live items (part 11 §41.8) |
| `window_hours` | any root | platform | decimal | — | `hours_between(activity_start_at, activity_end_at)` |
| `activity_start_at` | any root | platform | timestamp | — | from the request |
| `risk_level` | HRIS v1 | request | enum {low, medium, high} | low | field on `ops_l1` (required on `approve`) |
| `high_risk` | HRIS v1 | request | bool | false | `assess` set_variable |
| `review_score` | HRIS v1 | request | int | 0 | output binding of `review` |
| `items_received` | HRIS v1 | request | int | 0 | target of the `received_counter` ref binding of `items` (cross-lane mutation via the parent) |
| `pins_reset` | HRIS v1 | request | int | 0 | output binding of `items`: `count_true` of `pin_reset_done`, at `completed` |
| `required_score` | `dual_review` | call | int | 6 | **param in**, required; HRIS v1 binds the literal 6 |
| `review_score` | `dual_review` | call | int | 0 | **param out**; `add_sec`, `add_log` set_variable (the fork mutation) |
| `security_score` | `dual_review` | call | int (0..5) | 0 | field on `sec_review` (required on `approve`) |
| `logistics_score` | `dual_review` | call | int (0..5) | 0 | field on `log_review` (required on `approve`) |
| `received_counter` | `standard_custody_lane` | call | int | — | **param ref**; `count_received` (additive only, V-53) |
| `item.credential_type` | any `subject=item` template | platform | enum {device, token, password} | — | from the asset, fixed at lane start |
| `pin_reset_done` | `standard_custody_lane` | call | bool | false | **param out**; output binding of `pin_reset` |
| `loss_report_done` | `standard_custody_lane` | call | bool | false | field on H2 `loss_report` (part 6 §27.11) |
| `pin_done` | `pin_reset` | call | bool | false | **param out**; field on `reset_pin` (required on `done`) |

### §25.2 Graphs

**HRIS v1 root.**

```
submit(start) ──submitted──▶ alpha_l2 [approval · Alpha Leads L2 · SLA 4h → escalate Alpha Engineers L1]
alpha_l2 ──approve──▶ ops_l1 [approval · Ops Reviewers L1 · field risk_level*]
ops_l1 ──approve──▶ assess [set high_risk := risk_level == 'high' OR item_count > 3 OR window_hours > 8]
assess ──next──▶ is_high_risk [rule high_risk == true]
is_high_risk ──true──▶ review [embed single · dual_review v1 · in required_score := 6 · out review_score ▶ review_score]
review ──passed──▶ owner ; review ──failed──▶ rejected
   (propagating: a reject inside dual_review ▶ rejected ; a return ▶ amend; V-46)
is_high_risk ──false──▶ owner [owner_approval · primary/backup/delegate]
owner ──approve──▶ notify_pa [notify PA of primary owner] ──next──▶ wait_window [wait until activity_start_at − 30 min]
wait_window ──elapsed──▶ items [embed per_item · standard_custody_lane v1 · 1..5 · types {device, token} · all_live · replacement ✓ write-off ✓ · investigate Ops Leads L2 48h · ref received_counter ▶ items_received · out pin_reset_done ▶ count_true ▶ pins_reset]
items ──issued──▶ close_req [task kind=close · submitter or any reviewer] ──done──▶ join_done
items ──completed──▶ join_done [join all] ──joined──▶ completed(terminal completed)
items ──unfulfilled──▶ closed_unfulfilled(terminal closed_unfulfilled)
alpha_l2, ops_l1, owner: ──return──▶ amend ; ──reject──▶ rejected(terminal rejected)
amend ──resubmitted──▶ alpha_l2
platform (locked, D-26): expired(terminal) · tombstone auto-cancel → rejected · admin reset → amend
```

**`dual_review` v1** (`subject=none`; in `required_score`; out `review_score`; exits `passed`, `failed`,
`reject`, `return`):

```
entry ──next──▶ split_review [split: branch_1 "Security", branch_2 "Logistics"]
   branch_1 ─▶ sec_review [approval · Ops Leads L2 · field security_score*] ─approve─▶ add_sec [set review_score := review_score + security_score] ─▶ join_review
   branch_2 ─▶ log_review [approval · Asset Store L1 · field logistics_score*] ─approve─▶ add_log [set review_score := review_score + logistics_score] ─▶ join_review
join_review [join all] ──joined──▶ score_ok [rule review_score >= required_score]
score_ok ──true──▶ x_passed(exit passed) ; score_ok ──false──▶ x_failed(exit failed)
sec_review, log_review: ──return──▶ x_return(exit return) ; ──reject──▶ x_reject(exit reject)
```

**`standard_custody_lane` v1** (`subject=item`; ref `received_counter`; out `pin_reset_done`; exit `returned`; handlers H1–H4
and the fallback per part 6 §27.11):

```
entry ─▶ is_token [rule item.credential_type == 'token']
is_token ─true─▶ pin_reset [embed single · pin_reset v1 · out pin_done ▶ pin_reset_done] ─done─▶ prepare
is_token ─false─▶ prepare
prepare [Asset Store, 4h] ─▶ initiate_handover [Asset Store, 2h] ─▶ confirm_receipt [submitter · issuance milestone]
─▶ count_received [set received_counter := received_counter + 1] ─▶ initiate_return [submitter]
─▶ verify_return [Ops Reviewers, 24h] ─▶ return_to_inventory [Asset Store, 8h] ─▶ x_returned(exit returned)
```

**`pin_reset` v1** (`subject=item`; out `pin_done`; exit `done`):

```
entry ─▶ reset_pin [task · Asset Store L1 · field pin_done*] ─done─▶ x_done(exit done)
```

Edges marked `*` are required fields. **Node totals.**
- HRIS v1 root: 16 top-level nodes (3 of them designer terminals) plus the locked `expired`; two are
  embeds.
- `dual_review` v1: 12 nodes. `standard_custody_lane` v1: 11 nodes. `pin_reset` v1: 3 nodes.
- The HRIS v1 tree is therefore 42 nodes in 4 definitions (16 + 12 + 11 + 3), plus `expired`.

**The AUR library** also holds `basic_custody_lane` v1 (8 nodes: `entry`, the six custody stages and
`x_returned`; fallback handler only) and `dual_review` v2 (seeded as a **draft**, published in S13).
`dual_review` v2 has 13 nodes: a third branch `ops_lead_review` (approval, Ops Leads L2) feeds
`join_review`, which becomes `k_of_n` with k=2. The SEN library holds `sen_custody_lane` v1, used by
PORTAL.

**Simple definitions.** FINDB v1 and OPSTOOL v1 are linear chains, each ending in a `per_item` embed
of `basic_custody_lane` v1 (no split, no review embed). They show that a "classic chain" is just a
degenerate graph.

**HRIS v2 (draft only, seeded for S7, S9 and S13):**
- `required_score` input binding becomes the literal 5.
- `items` `max_items` becomes 3.
- The `review` pin moves from `dual_review` v1 to v2 (three branches, `k_of_n` k=2) through
  **Upgrade**, once v2 is published in S13.

### §25.3 Feature coverage matrix

Every engine feature is exercised by the seeded definitions, a scenario (§15) and an AC (part 5).

| Feature (Appendix A equivalent in brackets) | Node(s) | Scenario | AC |
|---|---|---|---|
| Variable initialisation (INIT) | `wf_variables` init, platform facts | S1, S8 | AC-18.1.1 |
| Scope, copy-in, copy-out, reducer, `ref` deltas | `review` bindings; `items` `pins_reset` and `items_received` | S1, S8 | AC-34.1.1 … AC-34.4.4 |
| External input mutates variables (signals add spend) | `ops_l1` field; `sec_review` and `log_review` fields in `dual_review` | S8 | AC-18.2.1 |
| Fork into parallel branches (FORK) | `split_review` in `dual_review` | S8 | AC-19.1.1 |
| Branch-local mutation of a shared variable | `add_sec`, `add_log`; lane `count_received` through the `received_counter` ref | S8, S1 | AC-19.1.3, AC-20.3.1 |
| Join waits for all branches (MERGE) | `join_review` (all) | S8 | AC-19.1.2 |
| Quorum join (k-of-n) | `dual_review` v2 `join_review`, via HRIS v2 | S9, S13 | AC-19.2.1 |
| Configurable rule `left op right` (CONDITION) | `score_ok`, `is_high_risk`, lane `is_token` | S8, S9 | AC-18.3.1, AC-18.3.2 |
| Approved / rejected routing (APPROVED / REJECTED) | `score_ok` true and false; reject ports | S8, S6 | AC-24.2.1 |
| Completion (COMPLETED) | `join_done` → `completed` | S1 | AC-20.4.1 |
| Multiple terminals | `completed`, `rejected`, `closed_unfulfilled`, `expired` | S1, S5, S6 | AC-15.2.1 |
| Dynamic per-item fork (not in Appendix A) | `items` per-item embed | S1, S3 | AC-20.1.1 |
| Per-item branching | lane `is_token` → `pin_reset` embed | S1 | AC-20.2.1 |
| Lane add / exclude after fork | replacement, write-off | S3, S4 | AC-20.3.2, AC-20.3.3 |
| Return-to-requester loop | `amend` | S2 | AC-24.2.2 |
| Timers: wait node, SLA escalation, platform deadline | `wait_window`, `alpha_l2` SLA, D-10 | S1, S5 | AC-21.1.1, AC-21.2.1 |
| Version pinning | HRIS v1 and its template pins running while v2 is published | S7 | AC-17.2.1 |
| Simulation for N items | editor | S9 | AC-22.1.1 |
| Live view, active blue and completed green, plus variables panel | request page | all | AC-20.5.1 |
| Drawn exception handlers with custom teams (investigator team differs from approver team) | H3 `investigate` Asset Store, approval Ops Leads (in `standard_custody_lane`) | S10 | AC-27.3.1 |
| Post-issuance replacement / supplementary lane | H3 `replace` after issuance | S10 | AC-27.3.2 |
| Quarantine on damaged return or in custody | H4 and H3 `quarantine` → `quarantine_return` | S10, S11b | AC-27.4.1, AC-27.4.2 |
| Replacement-limit exhaustion | H1 `replacement` refused at `REPLACEMENT_LIMIT` | S11 | AC-27.5.1 |
| Workflow cloning with assignee mapping | clone HRIS v1 → PAYROLL | S12 | AC-28.1.1 |
| Permutation matrix (every PM row) | H1–H4, fallback | S10, S11 | AC-27.10.1 |
| Handler SLA breach and escalation | H3 `investigate` SLA, `escalate_to_team` | S10-T (step 5a) | AC-26.4.2, AC-31.6.1 |
| Race and TOCTOU safety in resolutions | `apply_resolution` re-checks; approval vs reset, substitute taken | S6, S10 | AC-31.4.1, AC-31.5.2, AC-31.5.3 |
| Walkthrough fixtures for every PM row | `walk_pm_NN` and `ui_pm_NN` from part 10 §39 | S10, S11 | AC-32.12.1 |
| Template library, interface and eligibility (E1–E6) | `/admin/templates`; publish refused with `TEMPLATE_NOT_ELIGIBLE`; picker greys ineligible templates | S13 | AC-33.1.1, AC-33.1.2 |
| Single embed with input and output bindings | `review` (`dual_review` v1): `required_score` in, `review_score` out, ports `passed` and `failed` | S8, S13 | AC-33.2.1, AC-33.2.2 |
| Simulate a template on its own | `dual_review` v1 with inputs | S13 | AC-33.2.3 |
| Per-item embed with call-site settings, reducers and a `ref` binding | `items`: bounds, `all_live`, `received_counter` ▶ `items_received` | S1, S13 | AC-33.3.1, AC-33.3.2 |
| Nested embed (depth 2), cycle and depth refusal | `items` → `pin_reset`; draft embedding itself or nesting deeper than 3 | S1, S13 | AC-33.4.1, AC-33.4.2 |
| Template version upgrade; the older pin stays frozen | `dual_review` v2 published; HRIS v2 draft Upgrade; HRIS v1 still pinned to v1 | S13 | AC-33.5.1, AC-33.5.2 |
| Propagating reject and return from an embed | reject or return on `sec_review` inside `review` ends or returns the whole request | S6, S13 | AC-33.6.1, AC-33.6.2 |
| Clone with unresolved embeds, and same-organisation clone | clone HRIS v1 to a SEN system (placeholders) and within AUR (pins kept) | S12, S13 | AC-33.7.1, AC-33.7.2 |
| Template permissions, IDOR and OWASP negatives | read-only drill-in; other-organisation template routes and pins return 404 | S13 | AC-33.8.1, AC-33.8.2, AC-33.8.3 |

---

## §26 Supersession of parts 1–3

| Clause | Replaced by |
|---|---|
| §1 item 1 "sequential configurable approval chain" | A per-system graph definition (§21). A sequential chain is one valid shape |
| §3.1 "Position (status, current_step_ordinal)" for the MVP | Active node instances plus `plan_revision` (§23.2). `current_step_ordinal` is kept only as a derived column: the `seq` of the oldest active approval instance, or 0 at the owner stage |
| §6 `system_workflow_steps`, `system_custody_stage_configs` | §23.1 definition tables, with lane settings on the per-item `embed` node and custody stages in the lane template |
| §6 `request_approval_steps` | View over `request_node_instances` (§23.2) |
| §7.1 updates `step_decision`, `owner_decision`, `close`; `expected_ordinal` | Update `act` with `expected_revision` (§23.3) |
| §7.1 main loop, `join()` | `mvp_engine::settle`, with per-item join policy `all_live` = D-13 (§21.2) |
| §7.2 `CustodyCaseWorkflow` | `LaneWorkflow`, interpreting the lane template (§23.3). Custody and exception tables in §9 are unchanged |
| §8 "Step override: member of a team at a later snapshot step" | A member of a team of an approval node that is reachable downstream of the active node without passing `amend` or `owner_approval`, and whose node has `allow_override=true`. Override does not cross an embed boundary |
| §11 `step-decision`, `owner-decision`, `close` routes | `POST /api/requests/:id/nodes/:nodeInstanceId/act` (§24.2) |
| §12 `/admin` chain builder and custody stage config; graph generation rules | §24.1 editor and template library; R8 rendering from `/plan` |
| AC-3.2.x, AC-3.3.x (chain builder, custody config) | ST-15–ST-17, ST-20 (part 5); the old IDs are retired |
| Every AC using `step_decision{expectedOrdinal…}` or `owner_decision` | Read as `act{outcome, expectedRevision}` on the named node instance; the assertions are unchanged |
| §17 "`step_kind='peer'` and quorum or all-of approval steps" | Quorum and all-of are now in scope through `split` and `join` (k-of-n). Peer steps remain out |
| Further clauses of parts 1–5 | Part 6 §27.13 and part 8 supersede further clauses; see those tables |
| §21.2, §21.4, §22, §23, §24 for `expression` and `function` nodes | Part 12 §42 (supersession list in §42.17) |
