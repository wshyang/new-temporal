# Addendum 219 (part 5) — Workflow editor, definition and actualisation: user stories and acceptance criteria

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only.
**Normative source:** part 4 (`addendum-219-temporal-breakglass-mvp-part-4-workflow-definition-and-editor.md`, §20–§26).
**Later parts:** part 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27–§29), part 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30), part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34), part 9 (§35–§37, ST-31), part 10 (§38–§40, ST-32) and part 11 (§41, ST-34: variable scope and data flow). ST-33 (embeddable workflow templates) closes this part.
**Conventions:** these continue §14 (part 2), with the same AC format and the same "no side effect"
rule for every negative.

**Personas:**
- Greg (`full_admin`) and Amanda (`org_admin` @AUR) author workflows and the AUR template library.
- Carol owns HRIS.
- Gina (SEN) is the cross-organisation attacker.

---

### ST-15 Editor canvas and connector rules

**US-15.1** As an organisation admin, I want to build a system's workflow by drawing it.
- AC-15.1.1
  - G: Amanda opens `/admin/systems/{HRIS}/workflow/edit` on a new draft from v1.
  - T: the canvas shows the HRIS v1 graph (§25.2) laid out from the stored positions. Every node
    shows label, kind icon, assignee chip and labelled port handles. The `items` embed shows
    `standard_custody_lane · v1`, a "per item (1..5)" badge and its input and output chips. "Open"
    drills into the pinned template read-only (breadcrumb `HRIS v1 › standard_custody_lane v1`).
- AC-15.1.2
  - W: she drags an `approval` from the palette and connects `ops_l1.approve` to it.
  - T: the previous edge from `ops_l1.approve` is not duplicated. The connection is refused with the
    tooltip "This port already has a connection — delete it first" (one edge per out-port, §21.1).
- AC-15.1.3
  - In a root, the palette disables `start`, `amend` and `owner_approval` while one exists. Its
    "Embedded workflow" kind opens the template picker. The per-item cardinality is disabled once the
    root has a per-item embed.
  - In template mode (`/admin/templates/[id]/workflow/edit`), the palette swaps `start`, `owner_approval`,
    `amend` and `terminal` for `entry` and `exit`. `entry` is disabled while one exists.
  - `custody_stage` appears only when the template's subject is `item`.
- AC-15.1.4: every editor action is possible by keyboard. A Playwright test builds a two-approval
  linear definition using only the keyboard and the inspector's "Connect *port* to…" dropdown, and
  publishes it.

**US-15.2** As the platform, I enforce the connector rules the user stated.
- AC-15.2.1 (one initial, many terminals)
  - G: the HRIS v1 draft.
  - T: it has exactly one `start` and four terminals (`completed`, `rejected`, `closed_unfulfilled`,
    plus the locked `expired`).
  - W: Amanda adds a second `terminal(rejected)` and rewires `sec_review.reject` to it.
  - T: the draft remains valid (many terminals are allowed).
- AC-15.2.2: dragging any edge **into** `start` or `entry`, or **out of** any terminal or `exit`, is
  blocked by the UI. A crafted `PUT` containing such an edge returns the draft with issues V-02 or
  V-06 (error).
- AC-15.2.3
  - G: a node with no inbound edge.
  - T: V-04 is shown on that node ("This node is never entered"), and its in-handle is outlined red.
- AC-15.2.4
  - G: a non-terminal node with an unconnected out-port.
  - T: V-05 names the port ("`ops_l1.reject` is not connected"), and the port handle is outlined red.
- AC-15.2.5: an edge that crosses an embed boundary (from a node of the embedded template to a node of
  the caller, or the reverse) is blocked by the UI, which offers only the embed node's own ports.
  Via the API it returns V-07.

**US-15.3** As an author, I don't lose work and don't overwrite colleagues.
- AC-15.3.1: autosave runs 1 s after the last change and increments `revision`. Reloading the page
  restores the exact graph, including positions.
- AC-15.3.2
  - G: Greg and Amanda both have the draft open at revision 7.
  - W: Amanda saves (revision 8), then Greg saves with `expectedRevision:7`.
  - T: 409 `STALE_DRAFT`. Greg sees "Someone else changed this draft — reload", and his unsaved
    changes stay in local undo history.
- AC-15.3.3: undo and redo cover add, delete, move, connect, disconnect and inspector edits (50 steps).

**US-15.4** As the platform, I keep editor access scoped.
- AC-15.4.1: Gina, or any SEN admin, calling any `/api/admin/workflow-definitions/:defId*` or
  `/api/admin/systems/{HRIS}/workflow*` gets 404, the same body as a random UUID.
- AC-15.4.2: Carol (owner) can view the editor read-only: the palette is hidden, the inspector is
  disabled, and `PUT`/`publish` return 403. Bob gets 404.
- AC-15.4.3: inspector assignee pickers list only live teams of AUR clusters of a valid type
  (V-24). Typing a SEN team ID into a crafted `PUT` returns V-24.

### ST-16 Validation and publish

**US-16.1** As an author, I see every problem live and cannot publish a broken workflow.
- AC-16.1.1: for each V-code marked "error" in §22, there is a fixture definition violating only that
  rule. `validate` returns exactly that code, with the expected `node_key` or `edge_id`. The table
  test covers V-01…V-27, V-31 and V-44…V-51.
- AC-16.1.2: the Validation tab updates within 500 ms of a change. Clicking an issue selects and
  centres the node or edge.
- AC-16.1.3: "Publish" is disabled while any error exists. A crafted `POST …/publish` returns
  409 `DEFINITION_INVALID` with `issues[]`, and nothing changes.
- AC-16.1.4: warnings (V-28…V-30 and V-52) are shown but do not block publishing.
- AC-16.1.5 (single implementation): the editor renders server `issues[]` only. A bite check that
  removes V-12 from `mvp_engine::validate` makes AC-24.1.1 fail, which shows there is no separate UI
  copy of the rule.

**US-16.2** As an author, I publish a version with notes.
- AC-16.2.1
  - W: Amanda publishes the draft with notes "Add ops lead review".
  - T: the draft becomes `published` with `version = max+1`, `published_by=Amanda`, and
    `content_sha256` set. Audit `workflow_definition.published{system, version, sha}`.
- AC-16.2.2: publishing with blank or >2000-char notes → 422.
- AC-16.2.3: after publishing, any `PUT`, or any direct SQL `UPDATE`/`DELETE` on that definition's
  `wf_*` rows (including `wf_embed_pins` and `wf_embed_bindings`), fails. The API returns
  409 `DEFINITION_IMMUTABLE`; the trigger raises in SQL.
- AC-16.2.4 (template publish)
  - G: a template draft that violates E1–E5 (V-44…V-47).
  - W: Amanda publishes it.
  - T: 422 `TEMPLATE_NOT_ELIGIBLE` with the V-codes in `issues[]`, and no `version` is assigned. A
    template with only non-eligibility errors (for example V-05) returns 409 `DEFINITION_INVALID`.
    Audit `workflow_definition.published{template, version, sha}` on success.
- AC-16.2.5 (root publish)
  - G: a root draft with an embed whose pin is a draft, a retired version, or unset.
  - T: V-49 (error) blocks "Publish". A crafted `POST …/publish` returns 409 `DEFINITION_INVALID`.
    A published root has every pin resolved in `wf_embed_pins`.

### ST-17 Versioning and pinning

**US-17.1** As an author, I manage versions.
- AC-17.1.1: "New draft from vN" deep-copies every node, port, edge, variable, expression and field.
  A structural diff against vN is empty, and the IDs are all new. A second draft gets
  409 `DRAFT_EXISTS`. For a root, the copy keeps every embed's pin and bindings (still pointing at
  the same published template versions).
- AC-17.1.2: the Diff tab against v1 lists added, removed and changed nodes, edges and variables, and
  colours them on the canvas (added green, removed red ghost, changed amber).
- AC-17.1.3: retiring the only published version → 409. Retiring v1 after v2 is published succeeds.
  New submissions pin v2. Running v1 plans are unaffected. Retiring a template version still pinned
  by a published root is allowed, because pins are frozen; it only stops new pins (V-49 on drafts).
- AC-17.1.4: export downloads a canonical JSON file whose SHA-256 equals `content_sha256`. Importing
  it into FINDB creates a FINDB draft (teams resolved by cluster name and level within AUR; embeds
  resolved by `(template_key, version)` in the target organisation's library). Unresolvable teams
  or templates → 422 listing paths. No JSONB column exists in any `wf_*` table (schema test).

**US-17.2** As a requester, my in-flight request keeps the workflow it started with.
- AC-17.2.1 (S7)
  - G: Bob's request is in run #1 pinned to HRIS v1, at `ops_l1`.
  - W: Amanda publishes v2 (3-branch quorum).
  - T: Bob's request still renders and executes v1. The `/plan` response carries `definitionVersion:1`.
    `ops_l1` approval follows v1 edges.
- AC-17.2.2
  - G: the same request.
  - W: Claire returns it and Bob resubmits.
  - T: run #2 starts pinned to v2 (OP-7). The run selector shows both runs. Run #1's graph is
    unchanged and readable.
- AC-17.2.4 (pin tree)
  - G: Bob's run #1 pinned to HRIS v1, whose pin tree is `dual_review` v1 and
    `standard_custody_lane` v1 › `pin_reset` v1 (`wf_embed_pins`, depth 1..2).
  - W: Amanda publishes `dual_review` v2 and `standard_custody_lane` v2.
  - T: run #1 executes the v1 tree. The `/plan` response lists the pinned template versions. HRIS v1 is
    unchanged (same `content_sha256`). Run #2 (resubmit) pins HRIS's current version and so its
    pins, not the newest template versions.
- AC-17.2.3: a system whose only version is retired, or which has none published, refuses submit
  with 409 `SYSTEM_WORKFLOW_INVALID`. The new-request form hides it.

### ST-18 Variables, fields and rules

**US-18.1** As an author, I declare typed variables.
- AC-18.1.1 (INIT)
  - W: a request is submitted on HRIS v1.
  - T: `request_variables` holds every request-scope variable of §25.1 at its init value
    (`review_score=0`, `items_received=0`, `pins_reset=0`, …) and the root platform facts
    (`item_count=3`, …); `required_score` appears only when the `review` call starts (part 11 §41.9). Each has a
    `request_variable_changes` row with `source='init'` or `'system'`. The Variables panel lists
    them.
- AC-18.1.2: variable names that fail `^[a-z][a-z0-9_]{0,39}$`, duplicate names, or an init value of
  the wrong type → V-26 or V-21. A `decimal` init with more than 4 decimal places → 422.
- AC-18.1.3: `required_score` is constant. A `set_variable` targeting it → V-22, and so does a field
  bound to it.

**US-18.2** As a reviewer, my input feeds the workflow's variables.
- AC-18.2.1
  - G: `ops_l1` has required field `risk_level` on `approve`.
  - W: Claire approves without it.
  - T: 422 `fields.risk_level`.
  - W: with `risk_level=high`.
  - T: the variable is set (change source `field`, actor Claire, node instance `ops_l1`).
- AC-18.2.2: field values outside bounds (`security_score=6`), or an enum value not declared → 422.
  The validator in the workflow rejects the same input sent directly to Temporal.

**US-18.3** As an author, I configure rules with a comparison operator.
- AC-18.3.1 (CONDITION)
  - G: high-risk path with `security_score=3` and `logistics_score=2`.
  - W: the join completes.
  - T: `score_ok` evaluates `review_score(5) >= required_score(6) → false`, recorded in
    `request_rule_evaluations` and shown on the edge tooltip. The token goes to `rejected`, the
    request is `rejected` with `decision_reason='rule_rejected'`, and audit `workflow.rule_evaluated`
    and `request.rule_rejected` are written.
- AC-18.3.2 (operator is configurable)
  - G: v2 draft with `score_ok` changed to `>` and `required_score=5`.
  - W: simulated with scores 3 and 2.
  - T: `5 > 5 → false`.
  - W: changed to `>=` and re-simulated.
  - T: `true`. Each operator in {`>`, `<`, `>=`, `<=`, `==`, `!=`} has a unit test on int, decimal
    and timestamp operands. `==` and `!=` also have tests on bool, enum and text.
- AC-18.3.3: comparing `risk_level` (enum) with an int, or ordering a bool → V-21 at validate.
- AC-18.3.4 (R2): a rule outside the lane template (in the root) that references `item.credential_type` → V-20. Using
  `any(item.credential_type == 'token')` is accepted.
- AC-18.3.5 (R6)
  - G: `assess` has consumed `item_count`.
  - W: Claire's approver edit removes an item.
  - T: 409 `RULE_INPUT_LOCKED` "Item changes would alter a decision already taken. Return the
    request instead." An edit that only changes the justification succeeds.

### ST-19 Parallel review, joins and quorum

**US-19.1** As the platform, I fork, mutate and merge. In HRIS the split, approvals and join live in
the embedded `dual_review` template (v1), so each embed call has its own `review_score`.
- AC-19.1.1 (FORK)
  - G: `high_risk=true`.
  - T: the `dual_review` call's `split_review` creates two active instances, `sec_review` (Ops Leads L2) and `log_review`
    (Asset Store L1). Both teams see the request in "Needs my action". The graph shows both blue.
- AC-19.1.2 (MERGE)
  - W: only `sec_review` approves.
  - T: `join_review` waits (graph shows "1 of 2"). After `log_review` approves, the join fires once,
    and `score_ok` runs.
  - The same holds in the opposite order, and when both approve concurrently (within 10 ms): exactly
    one join firing and one rule evaluation.
- AC-19.1.3 (branch mutation of a shared variable)
  - W: both branches add to `review_score` concurrently.
  - T: the final value is the sum (5 for 3+2). There are two `request_variable_changes` rows with
    distinct `node_instance_id`s and contiguous `revision`s. No lost update. Repeated 50 times in a
    workflow test.
- AC-19.1.4 (D-25 interrupt)
  - W: `sec_review` rejects while `log_review` is active.
  - T: the template's `exit(exit_kind=reject)` propagates to the call site, so the request is
    `rejected`. `log_review` becomes `cancelled` (audit `workflow.node_cancelled`), and Asset Store's
    action list no longer shows it. A return instead of a reject → `amend`, with the same
    cancellation.
- AC-19.1.5: a split whose two branches are both assigned to teams the same person belongs to still
  requires two distinct decisions. That person may decide at most one branch (US-19.3).

**US-19.2** As an author, I use a quorum.
- AC-19.2.1 (S9)
  - G: HRIS v2, which pins `dual_review` v2 (three branches, `k_of_n` k=2).
  - W: two branches approve.
  - T: the join fires and the third branch's instance is `cancelled`. With k=3, all three are
    needed. k=0 or k=4 → V-18.
- AC-19.2.2: `any` policy fires on the first arrival and cancels the rest.

**US-19.3** As the platform, I prevent one person from satisfying a parallel review twice.
- AC-19.3.1: Ivan (in Ops Reviewers and, in a fixture, Ops Leads) approves `sec_review`. Then his
  `act` on `log_review` → 403 `ALREADY_DECIDED_IN_SPLIT`. The check runs in the validator and in the
  activity, and a bite check covers both layers.

### ST-20 Per-item embed and actualisation

**US-20.1** As the platform, I expand the per-item embed into lanes at entry (R1).
- AC-20.1.1
  - G: owner approval on a 3-item request.
  - W: the token reaches `items`.
  - T:
    - three `request_plan_lanes` rows (`origin=initial`, lane_no 1..3);
    - three custody cases with stage snapshots copied from the `custody_stage` nodes of the pinned
      lane template (`standard_custody_lane` v1);
    - three `request_embed_calls` rows and three `LaneWorkflow` children.
    
    The graph shows three lane bands labelled with asset names.
- AC-20.1.2: submit with 0 items, or more than `max_items` (5) → 422 `ITEM_BOUNDS`. Submit with an
  asset whose `credential_type` is not allowed (password on HRIS) → 422. The same checks apply at
  approver edit and resubmit (R9).
- AC-20.1.3: an approver edit before the embed is entered that changes the items (when R6 doesn't block it)
  is reflected in the lanes created later. Lanes are never created at submit.

**US-20.2** As the platform, I run per-item logic inside lanes (R3).
- AC-20.2.1
  - G: HT-01 (token) and SP-01 (device) lanes.
  - T: HT-01's lane activates `reset_pin` (Asset Store must record `pin_done=true` to
    finish; the binding copies it to the lane's `pin_reset_done`). SP-01's lane goes straight to `prepare`. The lane `is_token` evaluation is recorded per
    lane.

**US-20.3** As the platform, I reconcile lane changes after the fork (R4, R5).
- AC-20.3.1 (cross-lane mutation)
  - W: Bob confirms receipt on two lanes within 10 ms.
  - T: `items_received` becomes 2. The parent applied two `lane_event` deltas serially, with two
    change rows and no lost update.
- AC-20.3.2 (replacement adds a lane)
  - W: Carol approves replacing HT-01 with HT-02.
  - T:
    - lane 2 gets `excluded_at` (reason `replaced`);
    - lane 4 is created (`origin=replacement`, `replaces_lane_id`=lane 2) with a new child;
    - `item_count` is recomputed to the live count;
    - the per-item join now requires lane 4.
    
    The graph draws lane 2 struck through, with a "replaced by" edge to lane 4.
- AC-20.3.3 (write-off excludes a lane)
  - W: a write-off before receipt is approved.
  - T: that lane is excluded and `issued` fires when the remaining live lanes have crossed receipt.
  - W: this was the last live lane.
  - T: `unfulfilled` fires, and the token reaches `closed_unfulfilled`, with D-13 behaviour.
- AC-20.3.4: a per-item embed with `allow_replacement=false` (a call-site property) makes the "Propose replacement" action
  unavailable (and a crafted call → 409 `OUTCOME_NOT_ALLOWED`). The same holds for write-off.
- AC-20.3.5: join policy `all` on the per-item embed fires `unfulfilled` on the first exclusion. Policy `k_of_n` k=1
  fires `issued` after the first receipt (simulation tests).

**US-20.4** As the platform, I complete only when people and custody are both done.
- AC-20.4.1 (COMPLETED)
  - `issued` activates `close_req`.
  - After Bob closes, the request is `closed` but `join_done` still waits for the `items` embed's
    `completed`.
  - When the last lane reaches the lane template's `exit`, `join_done` fires, the token reaches `completed`, the
    plan run ends with `end_kind=completed`, and the parent workflow completes.

**US-20.5** As any viewer, I see the actualised workflow live.
- AC-20.5.1
  - The request graph (from `/plan`) shows active instances with a blue border and glow, completed
    instances green, taken edges solid and coloured, untaken edges grey dashed.
  - It updates within 3 s of another persona's action.
  - The Variables panel shows current values and, per variable, its change history (actor, node,
    time in SGT).
  - These are Appendix A's live-view conventions on our data.
- AC-20.5.2: after the Temporal workflow is deleted (retention), `/plan` and `/variables` still render
  fully from Postgres.
- AC-20.5.3: a 5-item request with one replacement renders in under 400 ms (performance mark). Lane
  bands collapse to a summary row when there are more than 3 lanes, and can be expanded.

### ST-21 Timers in the graph

**US-21.1** As an author, I can delay a step until a time.
- AC-21.1.1
  - G: `wait_window` = until `activity_start_at − 30 min` (demo: `SLA_UNIT_SECONDS=10` applies to
    hour-valued configuration only; absolute times are real).
  - W: the owner approves 2 minutes before `activity_start_at − 30 min`.
  - T: the token waits, the Engine panel lists the timer, and the `items` embed is entered when it
    fires. If the time has already passed, it is entered immediately.
- AC-21.1.2: a `wait` with `duration_hours=0` or more than 720 → V-25.

**US-21.2** As the platform, I escalate overdue approvals.
- AC-21.2.1
  - G: `alpha_l2` SLA 4 h with escalation to Alpha Engineers L1 (demo 40 s).
  - W: there is no decision.
  - T: at 40 s ± 2 s the instance gets `overdue_at`, an `sla_breached` audit row, and the escalation
    team gains the request in "Needs my action" with an "Escalated" badge.
  - Escalation adds eligibility; it does not reassign. The original team can still decide.
  - Self-approval (D-6) still applies.
- AC-21.2.2: the platform deadline (D-10) applies to every definition without being drawn. Removing
  the locked `expired` terminal is impossible in the UI, and through the API it returns V-31
  (`PLATFORM_TERMINAL_LOCKED`).

### ST-22 Simulation

**US-22.1** As an author, I preview how a request with N items would run before publishing (R10).
- AC-22.1.1 (S9)
  - W: Amanda simulates the HRIS v2 draft with 1 item, then with 5 items, scripting approvals with
    scores 3 and 3.
  - T: the actualised graph shows 1 lane, then 5 lanes. The trace lists every effect, the variables
    after each step and the rule renderings. The terminal reached is shown. The editor animates the
    trace (blue active, green done).
- AC-22.1.2: the simulation of a published definition equals the real run. A test runs the S1 script
  against a real worker and against `simulate`, and asserts identical node-instance sequences,
  taken edges and final variables.
- AC-22.1.3: simulation writes nothing and starts nothing. Row counts of every table and the Temporal
  workflow list are unchanged.
- AC-22.1.4: simulating a draft with errors → 422 with `issues[]`. A script step naming a non-active
  node → the trace stops with "Step 4: `log_review` is not active".

### ST-23 Interpreter determinism and replay

**US-23.1** As an engineer, I trust that the generic interpreter is deterministic.
- AC-23.1.1: `mvp-engine` has no dependency that performs I/O (a `cargo tree` gate) and uses no
  `HashMap` in any decision path (a lint or grep gate).
- AC-23.1.2: property test. For random valid definitions (generator obeying §22) and random event
  scripts, `settle` is a pure function: the same inputs give the same `(State, Effects)` 1,000 times
  out of 1,000. No token is left stranded: every run reaches a terminal or a human, timer or
  milestone wait.
- AC-23.1.3: the `CompiledPlan` is loaded by activity and present in history. Mutating the
  definition tables directly after a run started (test-only superuser) does not change the run's
  behaviour on replay.
- AC-23.1.4: replay tests (AC-13.2.1) include S7–S9 histories.

### ST-24 One SR per request; return and reject at every approval

**US-24.1** As the platform, I guarantee every reviewer can reject or return (D-25).
- AC-24.1.1: a definition in which any `approval` or `owner_approval` lacks a wired `return` or
  `reject` fails V-12. The publish API refuses it, and `reject` wired to a non-rejected terminal
  fails V-13.
- AC-24.1.2: for every approval node instance in S1–S9, `/actions` for an eligible reviewer offers
  exactly Approve, Return and Reject (plus Override where `allow_override`).

**US-24.2** As a reviewer, my reject ends the request and my return sends it back for amendment.
- AC-24.2.1 (APPROVED / REJECTED)
  - Reject at any approval node (`alpha_l2`, `ops_l1`, `sec_review`, `log_review`, `owner`) with
    notes takes its `reject` edge to `rejected`. The request becomes `rejected`, holds are released,
    sibling instances are cancelled, timers are cancelled and the workflow completes.
  - A rule-driven rejection (`score_ok` false) reaches the same terminal, with
    `decision_reason='rule_rejected'`.
- AC-24.2.2 (return loop)
  - Return at any approval node with notes activates `amend`, and the request is
    `returned_to_vendor`.
  - Bob amends (full validation) and resubmits. A new plan run starts at `amend.resubmitted` →
    `alpha_l2`, re-pins the current version (R7), re-initialises variables, and increments
    `resubmit_count`.
  - Remarks from run #1 remain visible in the timeline and in run #1's graph.
- AC-24.2.3: blank notes on return or reject → 422 (API) and are rejected by the validator.

**US-24.3** As the platform, I hold one SR / reference / activity on one system per request (D-24).
- AC-24.3.1: a request has exactly one `system_id`, one `sr_number` and one activity window. The
  request body has no array of systems or SRs (schema test). Items must all belong to that system
  (422).
- AC-24.3.2
  - G: a live HRIS request with SR `CHG-1001`.
  - W: Alice submits another HRIS request with `CHG-1001`.
  - T: 409 `DUPLICATE_SR` naming the existing request number (only if visible to Alice; otherwise
    the message is generic).
  - After the first request is rejected or closed, the SR can be reused. The same SR on FINDB is
    allowed.
- AC-24.3.3: two concurrent submits with the same SR → exactly one succeeds (partial unique index).
  The other gets 409 and no request row.

---

### ST-33 Embeddable workflow templates

Templates are published, versioned workflows in an organisation library (D-39, D-40). A root embeds
one as a node (`embed`, cardinality `single` or `per_item`). Pins freeze at root publish (D-42).
Nesting is bounded (D-43). Normative source: part 4 (interface and eligibility rules E1…E6, D-41;
V-44…V-52 in §22).

**Personas here:** Amanda (`org_admin` @AUR) owns the AUR library; Mona (`org_admin` @AUR and @SEN)
sees both libraries; Gina (SEN) has no AUR access.

**US-33.1** As an author, I see which templates I can embed, and why not the others.
- AC-33.1.1 (E1 entry, E2 exits)
  - G: a template draft with two `entry` nodes, or with no `exit`, or with nine distinct exit outcomes.
  - T: `validate` returns V-44 on the entry or exit node, and the Eligibility tab lists "E1" or "E2"
    with the node. A draft with one `entry` and exits `passed`, `failed` (1..8 outcomes) raises none.
- AC-33.1.2 (E2 outputs): an `out` parameter with no init value is refused with V-44 ("output is not
  defined on every exit"). A `ref` parameter that names no caller request variable is refused at
  the embed (V-48), not in the template.
- AC-33.1.3 (E3 root-only kinds)
  - G: a template that contains `start`, `owner_approval`, `amend`, `platform_interrupt` or a root
    `terminal` (crafted `PUT`; the template palette never offers them).
  - T: V-45 on each such node. Publish is refused (AC-16.2.4).
- AC-33.1.4 (E4 propagating exits)
  - G: a template whose approval `reject` goes to `exit(exit_kind=reject)`.
  - T: it is eligible as a `single` embed, but not as `per_item` and not downstream of
    `owner_approval`. Both placements return V-46 on the embed node.
- AC-33.1.5 (E5 subject)
  - G: `custody_stage` nodes or exception handlers in a template with `subject=none`.
  - T: V-47. Changing `subject` from `item` to `none` while custody stages exist returns V-47 too.
  - A `subject=item` template embeds `per_item` at a root, or `single` inside another
    `subject=item` template, which then may hold no custody stages and no handlers.
- AC-33.1.6 (E6 privacy): template-local variables are not offered in the embed's binding pickers, and
  a crafted binding to one → 422 `EMBED_BINDING_INVALID` (V-48).
- AC-33.1.7 (picker)
  - G: the AUR library holds `pin_reset`, `standard_custody_lane`, `dual_review`, `basic_custody_lane`,
    plus a draft-only template and a template containing `owner_approval`.
  - W: Amanda opens the "Embedded workflow" picker on HRIS with cardinality `single`.
  - T:
    - published, eligible templates are selectable, with `name · vN`, subject and interface summary;
    - the ineligible one is greyed, with the reason "E3 · V-45 `owner_approval` is root-only";
    - the draft-only template is greyed, with "no published version";
    - a `per_item` embed lists only `subject=item` templates without propagating exits.
  - A crafted `GET …/workflow-templates/eligible?forDefinition=&cardinality=` returns the same set.

**US-33.2** As an author, I embed a template once and wire its inputs and outputs.
- AC-33.2.1 (S13 setup)
  - G: HRIS root with a `review` embed of `dual_review` v1 (`single`).
  - T: the node's out-ports are the template's normal exit outcomes (`passed`, `failed`) and are
    wired to `owner` and `rejected`. The Inputs table lists `required_score` (int, required) and the
    Outputs table lists `review_score` (int).
- AC-33.2.2 (V-48)
  - G: `required_score` unbound; or bound to a `text` variable; or `review_score` bound to a constant
    variable (not writable).
  - T: V-48 on the embed, naming the parameter. A literal `6`, a request variable or a rule-builder
    expression of type int is accepted. Publish is blocked.
- AC-33.2.3 (call isolation)
  - G: the same template embedded twice in one root (two `single` embeds).
  - T: each call has its own template-local variables (`request_embed_calls`, call frame), and one
    call's `review_score` never leaks into the other's.
- AC-33.2.4 (runtime)
  - W: the request reaches `review`.
  - T: `EnterEmbed` binds `required_score=6`. The `dual_review` split runs inline. At `exit(passed)`
    the output `review_score` is copied to the bound variable and the token leaves by `passed`.
    Audit `workflow.embed_entered` and `workflow.embed_exited{exit_outcome}`. The request page shows a
    collapsible group with the breadcrumb chip `dual_review v1`.

**US-33.3** As an author, I embed a template once per item and set the call-site properties.
- AC-33.3.1 (V-51)
  - G: a system root with no per-item embed, with two, or with one whose template has no custody stage.
  - T: V-51 in each case. A root with exactly one per-item embed of a template with custody stages is
    valid. A second per-item embed on any root-to-leaf path (also through nesting) → V-51.
- AC-33.3.2 (call-site props)
  - W: Amanda sets, on the HRIS `items` embed, `min_items=1`, `max_items=5`, join policy `all_live`,
    `allow_replacement`, `allow_write_off`, `max_replacements_per_item`,
    `allow_post_issuance_replacement` and `allowed_credential_types={device, token}`.
  - T: they are stored on the `wf_nodes` row of the embed, not in the template. FINDB's `items` embed
    of `basic_custody_lane` can carry other values. `min_items` above `max_items`, or `max_items` > 10
    → V-51.
- AC-33.3.3 (reducer and ref): a per-item output binding without a reducer (`sum`, `count_true`,
  `any`, `all`, `min`, `max`) → V-48. The `ref` input `received_counter` bound to request variable
  `items_received` is applied serially by the parent (AC-20.3.1 holds).
- AC-33.3.4 (flags are enforced from the call site): with `allow_replacement=false` on the call
  site, "Propose replacement" is unavailable and a crafted call → 409 `OUTCOME_NOT_ALLOWED`, though
  the same lane template is embedded elsewhere with `allow_replacement=true`.

**US-33.4** As the platform, I bound nesting and refuse cycles.
- AC-33.4.1 (depth)
  - G: HRIS root › `standard_custody_lane` › `pin_reset` (template depth 2).
  - T: valid. A chain of four template levels below the root (depth 4) → V-50, and the picker greys
    the template at the deepest level with "depth limit 3".
- AC-33.4.2 (cycle)
  - G: Amanda drafts `pin_reset` v2 that embeds `standard_custody_lane`, which embeds `pin_reset`.
  - T: V-50 names the cycle (`pin_reset › standard_custody_lane › pin_reset`). A self-embed is refused
    at once. Cycles are detected by template identity across versions, so embedding
    `pin_reset` v1 from `pin_reset` v2 → V-50.
- AC-33.4.3: a crafted `PUT` that builds the cycle through `embed_definition_id` returns the draft with
  V-50 and cannot be published; the immutability trigger prevents a later edge from creating one in a
  published version.

**US-33.5** As an author, I upgrade a pin on purpose and nothing else changes (D-42).
- AC-33.5.1 (S13)
  - G: HRIS v1 is current and pins `dual_review` v1. Bob's request is in run #1 at `ops_l1`.
  - W: Amanda publishes `dual_review` v2 (3 branches, `k_of_n` k=2).
  - T: the library row for `dual_review` shows "Used by HRIS v1". Bob's run still executes
    v1. HRIS v1 keeps its `content_sha256`. The HRIS v2 draft shows "v2 available · Upgrade" on the
    `review` embed (V-52 warning).
- AC-33.5.2 (upgrade)
  - W: Amanda clicks "Upgrade" in the HRIS v2 draft.
  - T: the pin in the draft moves to `dual_review` v2, bindings are re-validated, and V-52 now lists
    "binding drift" if an interface parameter changed (for example `required_score` still unbound
    → V-48 error). No other root changes.
- AC-33.5.3 (V-49): a pin that points at a draft, a retired version, another organisation's template,
  or a template ineligible for the node's cardinality → V-49 on the embed and `EMBED_PIN_INVALID`
  (422) from a crafted `PUT`. Retired template versions stay readable in pinned roots.
- AC-33.5.4 (pin tree): publishing a root writes one `wf_embed_pins` row per template in the resolved
  tree with its depth. Run #1 and a later run each record the pin tree they used in `/plan`.
- AC-33.5.5: "Upgrade" never changes a published root. It is available on drafts only, and a crafted
  call on a published root → 409 `DEFINITION_IMMUTABLE`.

**US-33.6** As a reviewer, my reject or return in an embedded template ends or returns the request.
- AC-33.6.1 (propagating reject)
  - G: a request in the `dual_review` call, `sec_review` active and `log_review` active.
  - W: Frank rejects `sec_review` with notes.
  - T: the request is `rejected`, `log_review` is `cancelled`, the embed call ends with
    `exit_outcome` null and reason `reject`, and the audit shows `request.rejected` and
    `workflow.embed_exited`. Holds are released and timers cancelled (AC-24.2.1).
- AC-33.6.2 (propagating return): the same with `return` → `returned_to_vendor` (via `amend`). Bob's
  resubmit starts run #2 and re-initialises the template call's variables.
- AC-33.6.3: the normal exit `failed` (from `score_ok` false) is not propagating. The embed's `failed`
  port is wired to a `rejected` terminal in the root, and the request ends with
  `decision_reason='rule_rejected'`.
- AC-33.6.4: blank notes on a reject or return inside a template → 422 (D-25 applies inside templates).

**US-33.7** As an author, I clone workflows that embed templates.
- AC-33.7.1 (same organisation)
  - W: Amanda clones HRIS v1 into another AUR system.
  - T: the draft keeps each embed's pin, its bindings and its call-site properties. No `embed_unresolved`
    flag is set. Templates are not copied.
- AC-33.7.2 (cross organisation)
  - W: Mona clones HRIS v1 into a SEN system.
  - T: each embed becomes an unresolved placeholder: `embed_unresolved=true` and `embed_hint`
    "standard_custody_lane v1 (from AUR)". Bindings and call-site properties are copied. V-42
    (extended to embeds) blocks publish until each placeholder is replaced by an eligible SEN template
    or the templates are cloned.
- AC-33.7.3 (also clone templates): with "Also clone these templates into the target library" ticked,
  the wizard creates template drafts in SEN through `definition_copy`, re-points the placeholders to
  them (version "draft") and V-49 asks for them to be published first. Teams in the copied templates
  follow the §28 assignee mapping.
- AC-33.7.4: templates themselves can be cloned from `/admin/templates` into the same organisation
  (new `template_key`) or into another one the actor administers. `template_key` collision → 409.

**US-33.8** As the platform, I scope the template library and resist abuse.
- AC-33.8.1 (permissions)
  - Edit, publish and retire of a template: `full_admin` or `org_admin` whose scope includes the
    template's organisation (D-38). Everyone else gets 404.
  - Read-only drill-in ("Open" on an embed, breadcrumbs): anyone who can view the embedding root,
    including the system's owners (Carol: palette hidden, `PUT` and `publish` → 403).
  - A caller who cannot view the root gets 404 on the template, the same body as a random UUID.
- AC-33.8.2 (IDOR negative)
  - G: Gina (SEN) and a crafted `embed_template_id` or `embed_definition_id` of an AUR template.
  - T: every `/api/admin/workflow-templates/:id*` and `…/workflow-definitions/:defId*` call with that
    ID returns 404. A `PUT` on a SEN draft that embeds the AUR template returns V-49 and reveals
    nothing about it. `GET …/eligible` never lists templates of another organisation. No audit row
    reveals the template's name.
- AC-33.8.3 (cross-org listing): `GET /api/admin/organizations/:orgId/workflow-templates` for an
  organisation outside the actor's scope → 404. Mona sees two libraries only through two `orgId`
  calls.
- AC-33.8.4 (OWASP A03 injection)
  - W: a template is created with `template_key = "a'; --"` or `"x\"; select pg_sleep(5);--"`.
  - T: 422 (`^[a-z][a-z0-9_]{0,39}$`), with no timing difference. A `name` of
    `<script>alert(1)</script>` or `'; --` is stored as text, rendered escaped in the library, picker,
    breadcrumb and inspector, and the schema is intact. Names over 80 characters and descriptions
    over 500 → 422.
- AC-33.8.5 (OWASP A01/A04 mass assignment)
  - W: a `POST …/workflow-templates` or `PUT …/workflow-definitions/:defId` body carries
    `template_id`, `system_id`, `organization_id`, `version`, `status`, `content_sha256` or
    `embed_definition_id` outside the documented graph fields.
  - T: the extra fields are ignored or refused with 422, and the stored row keeps the values derived
    from the route. A draft cannot be moved to another organisation or turned into a system workflow.
- AC-33.8.6 (OWASP A05 oversized import): an import larger than 1 MiB → 413; a file with more than
  300 nodes, a depth-4 template tree or an unknown `template_key` reference → 422 listing paths. No
  draft is created.
- AC-33.8.7 (wrong state): a `PUT` on a published or retired template → 409 `DEFINITION_IMMUTABLE`; a
  second live draft of the same template → 409 `DRAFT_EXISTS`. Retiring the only published version
  of a template that is not pinned by any root is allowed; one still pinned by a draft raises V-49 in
  that draft.
- AC-33.8.8 (fuzz and bounds): `eligible?cardinality=` outside {`single`, `per_item`} → 422. A
  non-UUID `:id` → 404. Pin to a random UUID → V-49 / 422 `EMBED_PIN_INVALID`.
