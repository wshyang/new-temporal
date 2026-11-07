# Addendum 219 (part 7) — Exception handlers, cloning, coverage and module boundaries: user stories and acceptance criteria

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only.
**Normative sources:** part 6 (`addendum-219-temporal-breakglass-mvp-part-6-fulfilment-exceptions-cloning-coverage.md`,
§27–§29) for ST-25–ST-29, with parts 9 (ST-31) and 10 (ST-32) elaborating them, and part 8 (`addendum-219-temporal-breakglass-mvp-part-8-module-architecture-and-rust-conventions.md`,
§30–§34) for ST-30.
**Conventions:** these continue §14 (part 2) and part 5, with the same AC format. **Every negative AC
also asserts no side effect**: no row changed, no workflow history event, no audit row, no
notification. AC IDs are `AC-{subtask}.{story}.{n}`. The Rust test names `sim_pm_NN` and `db_pm_NN`
are defined in part 6 §27.12.

**Personas:**
- Bob (requester) submits HRIS requests and reports problems in custody.
- Grace and Leo (Asset Store L1) prepare and hand over assets. Grace is the reporter in most scenarios.
- Frank (Ops Leads L2) investigates. Claire, Emma and Ivan (Ops Reviewers L1) verify returns.
- Carol owns HRIS; Dan is the backup owner.
- Amanda (`org_admin` @AUR) and Greg (`full_admin`) author workflows.
- Betty (Beta Engineers L1, vendor) submits requests on PAYROLL.
- Gina (SEN) is the cross-organisation attacker.

**Assets (HRIS):** SP-01, SP-02, SP-03 (secure phones), HT-01, HT-02 (hardware tokens), LT-01, LT-02
(laptops). Handlers H1–H4 and the fallback are those of part 6 §27.11.

**Part 6 read-through.** Part 2 ST-9 ACs that name `resolve`, `propose` or `decide` are read as `act`
on the named port of a handler instance (part 6 §27.13); their assertions are unchanged. Recovered
(PM-07, PM-15) and false-report resolutions are asserted by AC-9.3.1–AC-9.3.3 and now run through the
H1/H2 ports; waivers by AC-9.6.1–AC-9.6.2; the return dispute by AC-9.7.2.

---

### ST-25 Exception handler editor and validation

**US-25.1** As an author, I want to see and draw the exception handlers of a lane template (D-28).
- AC-25.1.1
  - G: Amanda opens `standard_custody_lane` v1 (a `subject=item` template) from `/admin/templates` and
    starts a draft. In the HRIS root editor the same handlers are visible only by drilling into the
    `items` embed read-only, and the root has the **Interrupts** band.
  - T: the template editor has the bottom tabs **Interface**, **Eligibility** and **Handlers**, with the
    lane graph as the canvas. The Handlers tab lists
    H1, H3, H2, H4 and the fallback in ordinal order. Selecting a handler opens its mini-canvas with
    `exception_start`, `investigate`, `resolution_approval`, `resolution` nodes and generic nodes
    (for example the `loss_report` task in H2). The fallback is pinned last, shows no ordinal handle,
    and its delete button is disabled.
- AC-25.1.2: the palette enables `exception_start`, `investigate`, `resolution_approval` and
  `resolution` only in the Handlers tab, and disables them on the lane canvas (the **Lane** graph).
  `custody_stage` is disabled in a handler (V-41). Dragging an edge out of a handler, or into one from outside, is blocked
  by the UI; a crafted `PUT` containing it returns V-33 or V-39, and the draft revision is unchanged.
- AC-25.1.3: the `investigate` inspector offers the ports `recovered`, `false_report`, `replacement`,
  `write_off`, `waiver` and `quarantine` as checkboxes. Unchecking every port is blocked (a non-empty
  subset). Removing a port that has an edge asks for confirmation and deletes the edge with it.
- AC-25.1.4: handler editing is possible by keyboard alone. A Playwright test adds a handler, sets its
  filter, adds an `investigate` and a `resolution`, and connects them with the "Connect *port* to…"
  dropdown, using no pointer.

**US-25.2** As an author, I want to set the catch filter and order of a handler.
- AC-25.2.1
  - G: a new handler with no filter rows.
  - T: the filter form shows "All types · all phases · all severities". Adding `damaged` to the type
    dimension creates one `wf_exception_handler_filters` row (dimension `exception_type`). A dimension
    with no rows matches every value.
- AC-25.2.2
  - W: Amanda drags H2 above H3.
  - T: the ordinals are saved (`unique (definition_id, ordinal)` holds), the coverage grid now shows
    H2 winning `in_custody` damaged, and V-32 reports `HANDLER_SHADOWED` (warning) on H3. Dragging a
    handler below the fallback is refused.
- AC-25.2.3: a crafted `PUT` with a filter value outside its dimension's domain (type `exploded`,
  phase `after`, severity `urgent`), a filter row on the fallback, or two fallbacks in one template
  returns 422 with a path, and the draft revision is unchanged. The database CHECK and the partial
  unique index refuse the same rows via direct SQL.

**US-25.3** As an author, I want a coverage grid that shows which handler wins each case.
- AC-25.3.1: the grid has 10 exception types as rows and 3 phases as columns. On HRIS v1 every
  `pre_issuance` cell shows H1, every `in_custody` damaged or identity_mismatch cell shows H3, every
  other `in_custody` cell shows H2, and every `return` cell shows H4. Clicking a cell focuses the
  winning handler's mini-canvas.
- AC-25.3.2: impossible pairs are greyed and not clickable: `handover_disputed` and `receipt_disputed`
  outside `pre_issuance`, `return_disputed` outside `return`. They never produce V-32.
- AC-25.3.3
  - G: a draft in which H3 is limited to severity {high, critical}.
  - W: Amanda sets the severity selector to `low`, then `high`.
  - T: the `in_custody` damaged cell shows H2 for `low` and H3 for `high`. With the selector on
    "any", it shows "varies".
- AC-25.3.4: removing the only handler that catches `return` leaves the fallback as the winner. Making
  the fallback non-fallback (a crafted `PUT`) leaves a cell uncovered, and V-32 (error) names the
  (type, phase) pair.

**US-25.4** As an author, I want effect cards and team pickers so that I know what a handler does.
- AC-25.4.1
  - G: Amanda selects the H3 `resolution` node of kind `replace`.
  - T: the inspector shows one effect card per phase the handler can receive (here `in_custody`),
    fetched from `GET /api/admin/platform/exception-effects`, for example "Replace · after issuance ·
    damaged: original resumes in custody expecting a damaged return; substitute gets a supplementary
    lane". The editor contains no copy of the matrix (the fixture endpoint is changed in a test and the
    card changes with it).
- AC-25.4.2: `investigate` and `resolution_approval` pickers list only live **agency** teams of the
  system's organisation (V-24). `resolution_approval` additionally offers "Owner class" (primary or
  backup owner or active delegate). `escalate_to_team` lists agency teams. A SEN team ID or a vendor
  team in a crafted `PUT` returns V-24, and nothing changes.
- AC-25.4.3: choosing the same team for `investigate` and `resolution_approval` shows V-36 (warning)
  on the approval node, and Publish stays enabled.

**US-25.5** As the platform, I validate each handler rule on its own fixture.
- AC-25.5.1 (one fixture per rule). A table test loads the fixture below; `validate` returns exactly
  the named code with the expected `node_key` or `edge_id`, and no other code.

  | Code | Sev | Fixture violating only that rule |
  |---|---|---|
  | V-32 | error | lane template without a fallback handler; and a fallback that is not the highest ordinal |
  | V-32 | warning | `HANDLER_SHADOWED`: H3 filter wholly covered by an earlier handler |
  | V-33 | error | `investigate` port `write_off` wired to a `resolution` of kind `replace`; a path that ends nowhere; two `exception_start` nodes |
  | V-34 | error | `investigate.replacement` wired straight to `resolution(replace)` with no approval |
  | V-35 | error | `replace` in a handler whose phase filter includes `return`; `quarantine_return` in one that includes `pre_issuance` |
  | V-36 | warning | approval team equal to the investigate team |
  | V-37 | error | a cycle other than `reject` → `investigate`; `reject` wired to a `resolution` |
  | V-38 | error | `replacement` port with neither `write_off` (with `allow_write_off=true`) nor `recovered` |
  | V-39 | error | a handler edge to a top-level node |
  | V-40 | error | `sla_hours` of 0 and of 721 |
  | V-41 | error | a `custody_stage` inside a handler |
  | V-42 | error | an unresolved assignee on `investigate` (and one on an escalation target) |
  | V-43 | warning | `allowed_credential_types` includes a type with no live asset |

- AC-25.5.2: `resume` with `investigator_choice` never raises V-35 (the allowed set is derived at
  runtime); `resume` with `state_before_exception` on a handler catching `in_custody` is valid.
- AC-25.5.3 (publish gate): a draft with any error is refused by `publish` with 409
  `DEFINITION_INVALID` and `issues[]`. Warnings (V-32 shadowed, V-36, V-43) never block publish. A
  draft with an unresolved placeholder cannot be published even by a crafted request: the publish
  trigger rejects it (§27.7).

**US-25.6** As an operator, I want the seeded handlers to validate clean and route as stated.
- AC-25.6.1: HRIS v1 (H1, H3, H2, H4, fallback) validates with zero issues. FINDB v1, OPSTOOL v1 and
  PORTAL v1 each validate with zero issues and have exactly one handler, the fallback.
- AC-25.6.2: for every (type, phase) pair of HRIS v1, the simulator injects the exception
  (`{kind:'exception', lane_no, type, severity, at_state, substitutes_available, replacement_count}`)
  and `SimTrace.RouteDecision` names the winner shown in AC-25.3.1, with every earlier candidate
  marked unmatched and its reason.
- AC-25.6.3: the simulator can inject every resolution of the effects matrix (§27.5.2), and for a
  refused one it returns the refusal code (`NO_SUBSTITUTE`, `REPLACEMENT_LIMIT`,
  `POST_ISSUANCE_REPLACEMENT_DISABLED`). Simulation equals execution: the same injection run through
  the workflow gives the same route, effects and summary counts (AC-22.1.2).

**US-25.7** As the platform, I draw the interrupts and keep them locked (D-31).
- AC-25.7.1: the Interrupts tab shows three padlocked `platform_interrupt` nodes (`expire`,
  `tombstone_cancel`, `admin_reset`) with no handles. Selecting one shows a read-only description of
  what it cancels.
- AC-25.7.2: a crafted `PUT` that edits, moves, deletes or rewires an interrupt node returns 422
  `PLATFORM_NODE_LOCKED`, and the revision is unchanged. Direct SQL `UPDATE`/`DELETE` of a node with
  `is_platform_locked` in a draft raises from the trigger. The `expired` terminal keeps
  `PLATFORM_TERMINAL_LOCKED`.
- AC-25.7.3: "New draft from vN" and clone re-seed the locked nodes from the current platform
  template; they are not copied from the source.

---

### ST-26 Runtime routing and visibility

**US-26.1** As the platform, I route each exception to exactly one handler (D-28).
- AC-26.1.1 (first match)
  - G: Bob's SP-01 is `in_requester_custody` on HRIS v1.
  - W: Bob reports `damaged`, then (in a second scenario) `missing`.
  - T: `damaged` routes to **H3** (ordinal 2), and `missing` to **H2** (ordinal 3), though H2 also
    matches `damaged`. Each `request_exception_routes` row holds `handler_id`, `matched_ordinal`, and
    `rendered_match` such as "H3 Damaged in custody (ordinal 2): type damaged ∈ {damaged,
    identity_mismatch}; phase in_custody ∈ {in_custody}; severity any".
- AC-26.1.2: `request_exception_route_candidates` holds one row per handler tested, in order, up to the
  winner (H1 `matched=false`, reason "phase pre_issuance ≠ in_custody"). Later handlers are not
  tested and have no row.
- AC-26.1.3 (fallback)
  - G: a FINDB request, whose only handler is the fallback.
  - W: any exception is reported at any phase.
  - T: it routes to the fallback with `matched_ordinal` equal to the fallback's ordinal.
- AC-26.1.4 (purity and pinning): `route_exception` reads neither the database nor the clock. The
  route is computed from the run's pinned `CompiledPlan`: if Amanda publishes a v2 whose H3 is changed
  while the exception is open, the exception keeps v1's handler and instances. A route row is written
  once and never updated (an `UPDATE` fails).
- AC-26.1.5 (atomic opening): `open_exception` writes the exception, `phase`, `reported_by_role`,
  `case_exception_no`, the case move to `exception_open`, the route and candidate rows, the
  `exception_start` instance (completed), the first `investigate` instance (active), evidence, and
  the audit rows `custody.exception_reported` and `custody.exception_routed` in one transaction. With
  a fault injected at the audit write, no row exists.
- AC-26.1.6: the phase is derived once from the case state (§27.2) and stored. For a case at
  `exception_open` the engine uses `state_before_exception`. A lane at `in_requester_custody` is
  `in_custody` even while a sibling lane has not crossed receipt and the request is still `approved`.

**US-26.2** As a participant, I act on a handler node and the platform enforces who and when.
- AC-26.2.1: only the active `investigate` instance's team (here Ops Leads L2 for H1) can `act`. A
  member of another team (Grace on H1) gets 403. An actor outside the organisation (Gina) gets 404,
  the same body as a random UUID.
- AC-26.2.2: `act` before the first `open → investigating` update returns 409
  `INVALID_EXCEPTION_STATUS`. While the exception is `awaiting_information`, `act` on the same
  `investigate` instance also returns 409 `INVALID_EXCEPTION_STATUS`; `POST …/update` moves it back
  to `investigating` on the same instance (no new instance).
- AC-26.2.3: an `act` on an inactive instance → 409 `NODE_NOT_ACTIVE`; an outcome that is not a port
  of the node or is currently disabled → 409 `OUTCOME_NOT_ALLOWED`; a stale `expectedRevision` (case
  `version`) → 409 `STALE_STATE`; a second concurrent `act` on the same instance → exactly one
  succeeds (decision lock), the other is 409.
- AC-26.2.4: `GET /api/exceptions/:id` returns the route, the candidates, the instance chain,
  `offeredPorts[{port, enabled, disabledReason}]`, the candidate substitutes and the effect card for
  each reachable resolution. For the replacement port the reason follows the order of §27.4 (policy,
  then limit, then substitute).
- AC-26.2.5: `resume` with `resumeState` outside the allowed set of the phase (for example
  `awaiting_preparation` for an `in_custody` exception) → 422 `RESUME_STATE_INVALID`, nothing written.
- AC-26.2.6: a reject at `resolution_approval` (notes required) sets the proposal `rejected`, returns
  to `investigating`, and creates a **new** `investigate` instance; the earlier instances remain
  visible.

**US-26.3** As a requester or reviewer, I see the exception on the request graph (F-8, §27.10).
- AC-26.3.1
  - G: S10 step 5 (SP-01 damaged, replaced by SP-03).
  - T: the request graph shows a band "Exception #1" beneath the SP-01 lane with the route chip
    "H3 · Damaged in custody", the reason on expand (with H1 shown as skipped), its node instances in
    order with actor, remarks, SLA badge and the taken edges, and the `resolution` terminal's effect
    card in plain language.
- AC-26.3.2 (numbered bands): two exceptions on one lane render as "Exception #1" and "Exception
  #2" in `case_exception_no` order. `lane.replacement_count` carries across them. A third report while
  #2 is open is 409 (AC-9.1.2) and adds no band.
- AC-26.3.3: a supplementary lane is drawn with a dashed border and a "supplements" edge from the
  original lane. A replacement lane (`origin=replacement`) is drawn with the existing "replaced by" edge.
- AC-26.3.4 (interrupts): the Interrupts band is always drawn. When an admin reset with `force`
  cancels a pre-issuance exception, the `admin_reset` node is highlighted and draws a "cancelled by"
  edge to each cancelled handler instance. Un-fired interrupts show no highlight.
- AC-26.3.5 (after retention): after the workflow has completed and Temporal history is archived or
  deleted, `GET /api/requests/:id/plan` renders identical `exceptionBands[]`, `routes[]` and
  `interrupts` from the database rows alone, and the Engine panel shows "history archived" with the
  archive link rather than an error.
- AC-26.3.6: a viewer without access (Gina, or Bob for another requester's request) gets 404 on
  `/plan` and on `GET /api/exceptions/:id`.

**US-26.4** As a participant, I see the exception in the engine panel, timeline, SLAs and dashboards.
- AC-26.4.1: the Engine panel for the lane child lists the update `act` calls as Temporal updates,
  the activity `open_exception` and `apply_resolution` with their `op_ref`, and the run chain. The
  timeline interleaves `custody.exception_routed`, each `act` and the resolution.
- AC-26.4.2 (SLA)
  - G: H3 `investigate` SLA 8 h, demo unit 10 s, escalation to Ops Leads L2.
  - W: 80 s pass with no `act`.
  - T: `overdue_at` is set, the event is `sla_breached`, the escalation target is notified, and Grace's
    dashboard and that of the escalation team (Ops Leads L2) show "Overdue". The timer is cancelled on leaving the node (a later
    `act` leaves no stray timer).
- AC-26.4.3 (approval SLA and wait): an optional `resolution_approval.sla_hours` and a `wait` node
  in a handler use `SLA_UNIT_SECONDS` and are cancelled on leaving the node.
- AC-26.4.4 (dashboards): Frank's dashboard lists exceptions awaiting his team's `investigate`; Carol's
  lists exceptions at `resolution_approval` with her as owner-class assignee; Frank's also lists H3
  approvals (team approver). Emma's lists none. A user in no assignee set sees nothing for the
  exception, and the exception is not counted for them.
- AC-26.4.5: while an exception is open the request shows "Attention required: N custody
  exceptions", with N counting open exceptions of that request only.

---

### ST-27 Unhappy-path permutations

The scenarios S10 and S11 (part 6 §27.12.1–§27.12.2) are the end-to-end proofs. The rows PM-01…PM-31
(part 6 §27.12) are the cells each AC below must cover. Per row there is a `sim_pm_NN` and a
`db_pm_NN` test (AC-27.10.1).

**US-27.1** As the Asset Store, I deal with an item that is missing before issuance.
- AC-27.1.1 (replace, with a substitute; PM-01, PM-05; S3, S10 step 1, S11 step 1)
  - G: HT-01 is `awaiting_preparation`, HT-02 is free, HRIS v1 allows one replacement.
  - W: Grace reports HT-01 `missing` (severity high); Frank takes it (`open → investigating`) and acts
    `replacement` with `substituteAssetId` HT-02; Carol approves.
  - T: the route is H1. In one transaction (`apply_resolution`): the HT-01 case is `resolved_exception`;
    the HT-01 item is excluded (`replaced`); the HT-01 asset is retired `lost_replaced`; a new item
    (`replaces_item_id`), a hold on HT-02 and a case at `awaiting_preparation` exist; the HT-01 lane has
    `excluded_at`; a new lane has `origin=replacement`, `replacement_depth=1`; `items_replaced`=1; the
    event is `replacement_authorized`. The join counts HT-02, not HT-01. For a `damaged` item (PM-05) the
    result is identical.
- AC-27.1.2 (no substitute; PM-02; S10 step 2)
  - G: no free SP device exists (R0 holds LT-02 and SP-03).
  - W: Grace reports SP-02 `missing`; Frank opens `GET /api/exceptions/:id`.
  - T: `replacement` is listed disabled with the reason `NO_SUBSTITUTE`. An attempted `act` returns 422
    `NO_SUBSTITUTE`; no proposal row, no history event. A `substituteAssetId` outside the candidate set
    (held, retired, quarantined, other credential type or other system) returns 422 `SUBSTITUTE_INVALID`.
- AC-27.1.3 (write-off, partial; PM-06; S10 step 2)
  - W: Frank acts `write_off` for SP-02 and Carol approves.
  - T: the SP-02 item is excluded (`written_off`), the asset is retired `written_off`, the hold is
    released, the lane is excluded, the event is `asset_written_off`, and the join is `live=3`
    (SP-01, LT-01, HT-02). With `allow_write_off=false` the port is absent from the offer.
- AC-27.1.4 (dispute at handover; PM-09)
  - G: LT-01 is `handover_pending`.
  - W: Bob disputes the handover ("not the laptop I asked for"); H1 routes it; Frank resolves
    `recovered`.
  - T: the case resumes at `handover_pending` (the only allowed set in `pre_issuance`), the event is
    `asset_recovered`, the stage SLA is re-armed, and the item did not count as received meanwhile.
    A `resumeState` other than `handover_pending` is 422 `RESUME_STATE_INVALID`.

**US-27.2** As the investigator, I deal with an item that is lost after issuance.
- AC-27.2.1 (write-off after issuance; PM-10, PM-20; S4, S10 step 4)
  - G: the request is `credential_issued`; LT-01 is `in_requester_custody`.
  - W: Bob reports LT-01 `missing`; Frank takes it in H2 and acts `write_off`; Carol's decision is not
    yet offered; Bob completes the `loss_report` task; Carol approves.
  - T: before the task completes, Carol's `act` is 409 `NODE_NOT_ACTIVE`. After approval: the LT-01
    case is `resolved_exception`; the asset is retired `written_off`; the item is **not** excluded; the
    lane is not excluded; the request stays `credential_issued`; `items_written_off`+1; the event is
    `asset_written_off`; Bob may still close. The same write-off of a damaged item through H3 (owner
    approval) has the same effects.
- AC-27.2.2 (replace a lost item after issuance; PM-11, PM-12)
  - G: as above; HT-02 free (PM-11) or no spare token (PM-12).
  - W (PM-11): Frank acts `replacement` with HT-02 and Carol approves (H2).
  - T: the original case is `resolved_exception`; the original item is not excluded; the asset is
    retired `lost_replaced`; HT-02 is held with a new item and case at `awaiting_preparation`; a new lane
    has `origin=supplementary`; the outcome of the original is `replaced`;
    `items_replaced_after_issuance`+1; audit `custody.supplementary_lane_started`. `issued` is
    unaffected and the status stays `credential_issued`.
  - W (PM-12): the port is disabled and the attempt is 422 `NO_SUBSTITUTE` with no write.

**US-27.3** As the Asset Store, I deal with an item that is damaged after issuance.
- AC-27.3.1 (custom team assignment, H3; PM-16, PM-17, PM-22)
  - G: SP-01 is `in_requester_custody`; HRIS v1 H3 investigates with **Asset Store L1** and approves
    replacements with **Ops Leads L2**.
  - W: Bob reports SP-01 `damaged`; Frank (Ops Leads) tries to act on the `investigate` instance;
    Grace takes it and acts `replacement`; Grace tries to approve; Carol tries to approve; Frank
    approves.
  - T: Frank on `investigate` → 403, Grace on `resolution_approval` → 403, Carol on `resolution_approval`
    → 403 (a team assignee, not owner class); Frank approves and the replacement proceeds. The
    investigate SLA is 8 h. `waiver` goes to owner approval (Carol approves, Frank gets 403).
    For `identity_mismatch` (PM-22) with `replace` or `waiver`, the original resumes at
    `in_requester_custody` with `expected_return_condition=good` and the asset is not quarantined unless
    found damaged at return. With no substitute (PM-17) the replacement attempt is 422 `NO_SUBSTITUTE`.
- AC-27.3.2 (post-issuance replacement through a supplementary lane; PM-16; S10 step 5)
  - T: after Frank's approval the SP-01 case resumes at `in_requester_custody` with
    `expected_return_condition=damaged` (not excluded, hold kept); SP-03 gets a new item, hold and case
    at `awaiting_preparation` on a lane with `origin=supplementary`, `replacement_depth=1`;
    `items_replaced_after_issuance`+1; event `replacement_authorized`; audit
    `custody.supplementary_lane_started`. While SP-03 is not yet received the request stays
    `credential_issued` (a supplementary lane never counts in `issued`) but cannot complete until SP-03
    is terminal.

**US-27.4** As the Ops Reviewers, we handle damage found at return.
- AC-27.4.1 (unexpected damage at verify-return; PM-23, PM-20 waive; S11 variant b)
  - G: SP-01 is `return_pending`; `expected_return_condition=good`.
  - W: Claire verifies with `condition=damaged`.
  - T: in one transaction the case moves to `received_by_agency` (event `return_received`), the event
    `return_condition_recorded` is written, and an exception is opened with `type=damaged`,
    `severity=medium`, `reported_by_role=verifier`, `phase=return`, routed to H4. Claire (Ops Reviewers
    L1) takes it and acts `quarantine` (direct): the asset gets `quarantined_at/by/reason='damaged_on_return'`,
    the hold is released, the asset is **not** restocked, the case is `resolved_exception`, the outcome
    is `returned_damaged`, the summary counts `items_returned` and `items_returned_damaged` at this
    resolution (not at `return_received`), and `/requests/new` shows SP-01 disabled; submitting it is 409
    `ASSET_QUARANTINED`. The same applies when a waiver for in-custody damage (PM-20) had left
    `expected_return_condition` at `good`: the later damaged return raises this new exception.
- AC-27.4.2 (expected damage; PM-21, PM-24; S10 steps 5–6)
  - G: SP-01 has `expected_return_condition=damaged` (after H3 replacement, or after an in-custody
    `quarantine_return`, which resumes the case at `in_requester_custody` with that expectation).
  - W: Claire verifies with `condition=damaged`; Grace performs `return_to_inventory`.
  - T: no exception opens; `return_condition_recorded` is written; at `return_to_inventory` the asset is
    quarantined instead of restocked (`restocked_*` stay null); outcome `returned_damaged`. With
    `condition=good` against an expected `damaged`, the condition is recorded as `good` and the asset is
    restocked. The combined `verify_and_restock` with an unexpected `damaged` performs only the verify
    half and opens the exception.
- AC-27.4.3 (return-phase write-off and false report; PM-25)
  - W: in H4, Emma (Ops Reviewers L1) acts `write_off` and Carol approves; separately, a damaged
    report is judged `false_report`.
  - T: write-off in `return`: the item is **not** excluded, the asset is retired `written_off`, the hold
    is released, `items_written_off`+1, the join is unchanged. `false_report` resumes at
    `state_before_exception` and writes `exception_cancelled`.
- AC-27.4.4 (replace refused at return; PM-27)
  - T: H4 and the fallback have no `replace` resolution (V-35). The `replacement` port is not offered in
    any `return` exception, and a crafted `act` with outcome `replacement` is 409 `OUTCOME_NOT_ALLOWED`.
    A definition whose `return`-catching handler reaches `replace` fails V-35 at publish.

**US-27.5** As the platform, I bound replacements and finish cascades.
- AC-27.5.1 (replacement limit; PM-03, PM-04, PM-13, PM-18, PM-28; S11 step 2)
  - G: HT-02 replaced HT-01 (depth 1); `max_replacements_per_item=1`.
  - W: Bob disputes the handover of HT-02 ("token cracked"); H1 routes it; Frank opens the offer.
  - T: `replacement` is disabled with `REPLACEMENT_LIMIT` (reported before `NO_SUBSTITUTE`); an attempt
    is 409 and writes nothing. The offered ports are `write_off`, `recovered`, `false_report` and
    `waiver`. The same refusal holds `in_custody` for lost and damaged items. With limit 3, three
    replacements succeed (depths 1, 2, 3) and the fourth is `REPLACEMENT_LIMIT`; with limit 0, or
    `allow_replacement=false`, the first is refused. A supplementary lane also counts for the chain.
- AC-27.5.2 (exhaustion → write-off → closed_unfulfilled; PM-06; S11 step 3)
  - W: Frank proposes `write_off` for HT-02 and Carol approves.
  - T: HT-02 is excluded; both lanes are excluded; the join is `AllExcluded`; `close_unfulfilled` sets
    `closed` with `decision_reason='unfulfilled_all_items_written_off'`; audit
    `request.closed_unfulfilled`; the workflow completes at `closed_unfulfilled`. Summary: total 2,
    issued 0, replaced 1, written off 1. Cascade after issuance: a supplementary lane's replacement
    item lost in custody is `REPLACEMENT_LIMIT` at limit 1, is written off, the issuance stands and the
    request completes once every lane is terminal.
- AC-27.5.3 (post-issuance replacement disabled; PM-14, PM-19)
  - G: `allow_post_issuance_replacement=false`; HT-02 free.
  - W: Bob reports an in-custody item `missing`, then `damaged`; Frank opens the offer.
  - T: `replacement` is disabled with `POST_ISSUANCE_REPLACEMENT_DISABLED` (checked before limit and
    substitute); an attempt is 409 and writes nothing. A `pre_issuance` replacement is still allowed.

**US-27.6** As the platform, I keep exceptions on different lanes independent.
- AC-27.6.1 (S10 mixed lanes; PM-29)
  - G: S10 preconditions (R0 issued and holding LT-02 and SP-03; HT-02 the only free token).
  - W: S10 steps 1–6 are run through the public API.
  - T: the four lanes take four different paths concurrently; handlers are independent and the parent
    applies lane events serially. After step 2 the join is `live=3`; after step 3 the `items` embed emits
    `issued` (status `credential_issued`) while exceptions are still to come; the graph shows four
    exception bands (HT-01, SP-02, LT-01, SP-01), each numbered per lane. Two proposals for the same
    substitute on different lanes: the second approval fails in the activity with 409 `ASSET_IN_USE`,
    writes nothing, and the instance stays active.

**US-27.7** As the platform, I define how exceptions interact with close, expiry and reset (§27.8).
- AC-27.7.1 (close; PM-30)
  - G: `credential_issued`, one exception open on LT-01.
  - W: Bob closes the request; Frank continues the exception and Carol approves.
  - T: close is allowed and does not stop handlers (AC-9.5.4); reporting and resolving continue after
    close; the parent stays Running until every lane is terminal.
- AC-27.7.2 (auto-expire and tombstone; PM-30)
  - T: auto-expire (D-10) and tombstone auto-cancel have no effect once owner approval has happened.
    If Bob is tombstoned after approval, handlers continue; a `loss_report` task assigned to the
    submitter is actionable by an active member of the ordinal-1 team of the run's submit step, recorded
    `acting_as=submitter_team`; any other user gets 403.
- AC-27.7.3 (admin reset; PM-30)
  - G: LT-01 has an open `pre_issuance` exception; SP-01 is `in_requester_custody` with an open exception.
  - W: an admin resets without `force`, then with `force=true` and notes.
  - T: without force → 409 `CUSTODY_IN_PROGRESS` listing the cases, nothing changed. With force: the
    LT-01 case is `cancelled`, its handler instances are `cancelled` by the `admin_reset` node, the
    exception is `resolved` with `resolution_notes='cancelled_by_admin_reset'` and no resolution value,
    the event is `exception_cancelled`, and the audits `custody.case_cancelled` and
    `workflow.interrupt_fired` exist. SP-01's exception and handler continue unchanged.

**US-27.8** As an auditor, I want separation of duties on every resolution approval (D-6, widened).
- AC-27.8.1 (PM-31; S3, S10 step 5)
  - G: a handler whose approval team equals its investigate team (a V-36 warning fixture), and H3.
  - W: the investigator proposes and then tries to approve; the reporter, granted owner-class rights
    through a fixture, tries to approve; the submitter, granted a delegate window on the owner, tries to
    approve; an unrelated member of the approval team approves.
  - T: each of the first three is 403 `SOD_VIOLATION`, re-checked inside the activity transaction (D-5);
    nothing changes and the instance stays active. The unrelated approver succeeds. In S10 step 5 Grace
    (investigator, not in the approval team) and Carol (team step) get a plain 403 and Frank (team member who did not investigate) succeeds. A reject by an
    eligible actor needs notes (422 without) and is subject to the same rule.

**US-27.9** As an auditor, I want the summary, CSV and asset history to agree (D-19, ST-11).
- AC-27.9.1 (PM-01…PM-31; S10 final summary)
  - T: after S10, the Issuance and returns panel shows: total 6, issued 4, outstanding 0, returned 3,
    returned damaged 1, replaced 1, replaced after issuance 1, written off 2. The CSV export has the
    same rows and counts. `/assets/:id/history` shows, per asset: SP-01 `returned_damaged` and
    `quarantined_at`; LT-01 `written_off`; HT-01 `replaced` (`lost_replaced`); SP-02 `written_off`;
    HT-02 and SP-03 restocked. For every PM row the three views report the same per-item outcome. A
    quarantined asset cannot be selected: `/requests/new` disables it and submit is 409
    `ASSET_QUARANTINED`. An organisation-scoped read by Gina gets 404 on all three views.

**US-27.10** As the team, I want proof that every permutation is covered.
- AC-27.10.1 (meta-test)
  - G: `fixtures/permutation-matrix.csv` and the table in part 6 §27.12.
  - W: the meta-test runs.
  - T: it fails if the two differ, if any row id lacks `sim_pm_NN` or `db_pm_NN`, or if a named test
    does not pass. It also checks that S10 and S11 exercise every row that designates them, and that the
    Playwright runs of S10 and S11 produce videos (the e2e videos of §16). Deleting `db_pm_16` turns the
    test red, and so does removing a row from the CSV.

---

### ST-28 Cloning a workflow from an existing system's version

**US-28.1** As an organisation admin, I want to clone a workflow so that I don't rebuild it (D-32).
- AC-28.1.1 (S12 end to end)
  - G: PAYROLL is a new AUR system, linked to the vendor cluster Beta, with its own owner and one live
    asset per allowed credential type (device and token), so V-43 is not raised; HRIS v1 is published.
  - W: Amanda opens `/admin/systems/new`, chooses **Clone from system version**, picks AUR → HRIS → v1,
    and previews. She maps Alpha Leads L2 to Beta Leads L2 (the ordinal suggestion), keeps the agency
    teams, and leaves the vendor team Alpha Engineers L1 (no default) unresolved. She confirms. The editor opens on the draft.
  - T: a draft exists with lineage "cloned from HRIS v1, AUR". V-42 lists one error, on the `alpha_l2`
    escalation that uses that team ("Choose team (was …)"). `simulate` and `publish` return 409 `DEFINITION_INVALID`. Amanda chooses
    the team, V-42 clears, V-43 is not raised, and she publishes PAYROLL v1. Betty
    submits a PAYROLL request; the run is pinned to PAYROLL v1 and renders the cloned graph, whose
    `items` embed keeps its pin on `standard_custody_lane` v1 (same organisation), so H1–H4 apply, with
    Beta in the vendor step. The HRIS definition is unchanged.
- AC-28.1.2: the wizard's Workflow step offers **Start blank (starter template)**, **Clone from system
  version** and **Import file**. Blank creates `start → approval → owner_approval → per-item
  embed (1..5) of the organisation's default lane template (`basic_custody_lane`, one fallback
  handler) → terminals`, with unresolved assignees, so it cannot publish until chosen.
- AC-28.1.3: `/admin/systems/[id]/workflow` has **New draft from another system's version** beside "New
  draft from vN". With a live draft on the target → 409 `DRAFT_EXISTS`, and nothing is created.
- AC-28.1.4: the source picker lists only published and retired versions (with publisher, date and
  `content_sha256` prefix). An `org_admin` sees only the target system's organisation (cross-organisation cloning is `full_admin`
  only, D-38); a `full_admin` sees all.

**US-28.2** As an administrator, I want a preview and a mapping before anything is written.
- AC-28.2.1: `POST …/workflow/clone/preview` writes nothing (row counts of every table unchanged). It
  returns the source, `crossOrganisation`, `counts{nodes, edges, handlers, variables, references}` and
  one entry per distinct `(cluster, level)` reference with `usedBy[]`, `defaultTarget` and
  `suggestions[]`.
- AC-28.2.2: same-organisation defaults: an agency reference defaults to the same team; a vendor
  reference defaults to the same team only if its cluster is linked to the target system
  (`system_vendor_clusters`). Suggestions are by `name_level` (agency) and `ordinal_level` (vendor).
  Cross-organisation previews return no `defaultTarget` for any reference.
- AC-28.2.3: a mapping to a team outside the target's organisation, a `pa` cluster, a deleted team, or a
  vendor team for an agency reference (or the reverse) → 422 `CLONE_MAPPING_INVALID` with a `fields`
  path per entry; no draft, no audit row.
- AC-28.2.4 (unresolved placeholders): a reference with no default and no mapping becomes
  `assignee_unresolved=true` (or `escalate_unresolved`), cluster id NULL, hint holding the source label.
  The draft is saved and listed. V-42 gives one error per node and slot, and blocks `simulate` and
  `publish`. Choosing a team clears the flag. A crafted publish is refused by the trigger.
- AC-28.2.5: V-43 is a warning: a cloned per-item embed that allows a credential type with no live asset in the
  target warns but does not block publish.

**US-28.3** As the platform, I keep cloning scoped (OP-8).
- AC-28.2.6 (embed placeholders)
  - G: Greg (full_admin) clones HRIS v1 (AUR) into PORTAL (SEN).
  - T: the preview lists the referenced templates (`dual_review` v1, `standard_custody_lane` v1). After
    the clone each embed is an unresolved placeholder: `embed_unresolved=true`, pin cleared,
    `embed_hint` "standard_custody_lane v1 (from AUR)", with bindings and call-site properties copied.
    V-42 reports `EMBED_UNRESOLVED` once per embed and blocks `simulate` and `publish`. A same-organisation
    clone (HRIS → PAYROLL) keeps the pins and raises none. Choosing `sen_custody_lane` v1 clears
    the flag on the per-item embed and V-48 re-checks its bindings.
- AC-28.2.7 (also clone templates)
  - W: Greg ticks "Also clone these templates into the target library" in the wizard.
  - T: `definition_copy` creates template drafts in SEN (new `template_key` rows, handlers and filters
    included, teams mapped like any reference) in the same transaction as the system draft, and the
    placeholders point at them. Publishing the root is refused by V-49 until those template drafts are
    published. Without the tick, no `wf_templates` row is created. A `template_key` collision in SEN
    → 409 and nothing is written.
- AC-28.3.1: Amanda (org_admin @AUR) can clone within AUR. Greg (full_admin) can clone within or across
  organisations; across organisations every reference needs a mapping (no defaults).
- AC-28.3.2: Amanda cloning a SEN source → 404, the same body as a random UUID, and nothing is read or
  written (no `clone_source_read` audit row). Gina cloning an AUR source into a SEN system → 404.
  Owners, vendors and agency staff → 404.
- AC-28.3.3: the caller needs edit rights on the **target** system. Carol (owner, view only) → 403;
  Bob → 404.
- AC-28.3.4: a source that is a draft → 409 `CLONE_SOURCE_NOT_PUBLISHED`. A retired source is allowed.
  A soft-deleted or unknown source → 404.
- AC-28.3.5: a source that belongs to the target system → 422 `CLONE_SAME_SYSTEM` (use draft-from-version).
- AC-28.3.6: Mona (org_admin @AUR and @SEN, D-38) can clone within AUR and within SEN. Mona
  cloning an AUR source into PORTAL (SEN) → 403 `CLONE_CROSS_ORGANISATION`, nothing written.

**US-28.4** As an author, I want a faithful, traceable copy.
- AC-28.4.1 (deep copy)
  - G: Amanda clones HRIS v1 with identity-equivalent mappings into a draft.
  - T: every `wf_variables`, `wf_nodes` (including handler nodes), `wf_node_ports`, `wf_edges`,
    `wf_expressions`, `wf_node_fields`, `wf_allowed_credential_types`, `wf_exception_handlers` and
    filter row is copied with a new UUIDv7. A structural diff against the source is empty modulo the
    remapped assignees. Positions are kept.
- AC-28.4.2 (not copied): owners, delegates, PA mappings, vendor-cluster links, assets, requests,
  runs, history, variable values, `change_notes`, version number and `content_sha256` are not copied.
  The draft has `version` NULL and `revision` 1; the sha is computed at publish.
- AC-28.4.3: the `expired` terminal and the three interrupt nodes are re-seeded from the platform
  template: after changing the template in a test, a new clone carries the change.
- AC-28.4.4 (lineage and audit): `cloned_from_definition_id` and `clone_source_sha256` are set once
  (an update fails) and shown on the version list. Audit `workflow_definition.cloned` is written in the
  target organisation with source ids, version, sha and the mapped and unresolved counts. For a
  cross-organisation clone an additional `workflow_definition.clone_source_read` row is written in the
  **source** organisation's audit, in the same transaction.

**US-28.5** As the platform, I make cloning atomic, idempotent and single-path (F-6).
- AC-28.5.1 (atomic): with a fault injected at any child table, at lineage, or at the audit write,
  the response is an error and no definition, node, edge, handler, lineage or audit row exists.
- AC-28.5.2 (idempotent): the same `X-Idempotency-Token` returns the first response and creates one
  draft. A different token while that draft exists → 409 `DRAFT_EXISTS`. A missing token is refused under the §11 idempotency rule, with no draft created.
  The source version is byte-identical before and after (same `content_sha256`).
- AC-28.5.3 (one copy path): draft-from-version, import and clone all call
  `mvp_db::definition_copy`. A test over the three callers asserts identical rows for identical inputs.
  A bite check adds a new `wf_*` table to the copy and removes it from `definition_copy`: the
  draft-from-version, import and clone tests all fail, which shows there is no second implementation.
  Import maps references by `(cluster name, level)` in `Strict` mode (unresolvable → 422 with paths);
  clone uses `Placeholder` mode.
- AC-28.5.4: a cloned and published definition passes every V-rule and the replay corpus (ST-30)
  accepts its `CompiledPlan` format.

---

### ST-29 Coverage gate

This story states the gate. It does not state which tests to write (part 6 §29.0).

**US-29.1** As the team, I want CI to fail when Rust coverage drops below 100%.
- AC-29.1.1
  - G: `make coverage-gate` on a green tree.
  - T: `cargo llvm-cov` over the whole workspace (every crate of part 8 §30.2, libraries, binaries and
    the test-support crate) reports 100% lines, functions and regions with
    `--fail-under-lines 100 --fail-under-functions 100 --fail-under-regions 100`.
  - W: one test that is the only cover of a function is deleted.
  - T: the gate fails, naming the file and function, and the exit code is non-zero.
- AC-29.1.2 (binaries): `api` and `worker` are exercised by the black-box harness under the environment
  from `cargo llvm-cov show-env` and are stopped with a signal that runs the graceful-shutdown path.
  A binary stopped with `SIGKILL` writes no profile; the gate then fails (proved by a deliberate run).
  `fn main` stays thin, calls `run`, and is measured.
- AC-29.1.3: branch coverage appears in the report but has no gate (stable Rust 1.99.1). Unreachable code
  is deleted or made unrepresentable by the types; no `unreachable!` is added to satisfy coverage.

**US-29.2** As the team, I want CI to fail when frontend coverage drops below 100%.
- AC-29.2.1: Vitest runs with jsdom, React Testing Library and provider `v8`, with `include:
  ['src/**/*.{ts,tsx}']` and thresholds 100 for lines, branches, functions and statements, all with
  `perFile: true`. A new file under `web/src/**` with no test counts as 0% and fails the gate.
- AC-29.2.2: one file at 99% fails the run even when the average is 100%. Playwright runs are not
  counted in these figures; they remain required for the e2e videos (§16).

**US-29.3** As a reviewer, I want exclusions limited to generated code and escape hatches blocked.
- AC-29.3.1: the only exclusions are listed in `coverage-exclusions.md` with glob, generator, command
  and reason (the generated OpenAPI client, generated protobuf or `build.rs` output). A CI step
  regenerates each file and fails on a diff, so a hand-edited file cannot hide under an exclusion.
- AC-29.3.2: `make coverage-gate` runs `scripts/check-no-coverage-escapes.sh`. For each of these
  patterns, a fixture that adds it outside `coverage-exclusions.md` makes the gate fail: `c8 ignore`,
  `istanbul ignore`, `v8 ignore`, `#[cfg(not(coverage))]`, `cfg(coverage`, `coverage(off)`,
  `coverage_nightly`, `LCOV_EXCL`, an `--ignore-filename-regex` or `exclude` entry not backed by an
  exclusion record.
- AC-29.3.3: a fixture that lowers any `fail-under` or `thresholds` value below 100 in the Makefile, CI
  files, `vitest.config.*` or Cargo config fails the gate. A fixture that moves production code into a
  `tests` path or a `*-test-support` crate to avoid measurement fails it too.

**US-29.4** As an auditor, I want the reports kept and coverage not mistaken for verification.
- AC-29.4.1: CI publishes the Rust (lcov, HTML, summary JSON) and frontend (lcov, HTML) reports as build
  artefacts on every pull request, and the gate's pass or fail is shown on the pull request.
- AC-29.4.2: coverage does not replace the other gates (§29.6). `docs/ac-traceability.md` still maps
  every AC to a named test; an AC with no named test fails the traceability check even at 100% coverage.
  The bite checks (AC-14.1.2) and the real-database rule still apply: an activity test that mocks the
  database does not satisfy the AC, whatever it covers.

---

### ST-30 Module boundaries and Rust conventions

The checks live in the test-only workspace member `mvp-fitness` (part 8 §30.3) and in the workspace
lints. Each negative below is proved by a fixture that makes the check fail with a message naming the
rule.

**US-30.1** As an architect, I want the dependency rules of §30.3 enforced by tests.
- AC-30.1.1 (edge check)
  - G: `cargo metadata` of the workspace.
  - T: every workspace edge is in the §30.3 table. The test fails if `mvp-api` depends on `mvp-worker`
    (F-4), if any crate depends on `mvp-api` or `mvp-worker`, or if `mvp-engine` depends on
    `mvp-db`, `mvp-worker` or `mvp-api`.
- AC-30.1.2 (no I/O in pure crates): the dependency closure of `mvp-kernel`, `mvp-custody`, `mvp-authz`
  and `mvp-engine` contains none of `sqlx`, `tokio`, `axum`, `reqwest`, `temporalio-*`, `rand`.
  `mvp-custody` and `mvp-authz` have no DB dependency (F-3); adding `sqlx` to `mvp-custody` fails.
- AC-30.1.3 (`vocab` only): a source scan fails when `mvp-engine` or `mvp-contract` contains any path
  `mvp_custody::` that is not `mvp_custody::vocab` (risk R-1).
- AC-30.1.4 (`lifecycle`): a source scan fails when `mvp-api` imports any `lifecycle` module (D-3 in
  the type system). No function in a `lifecycle` module takes a pool (RC-12).
- AC-30.1.5 (`mint`): the cargo feature `mint` of `mvp-kernel` is enabled only by `mvp-db`. Enabling it
  from `mvp-engine`, `mvp-custody`, `mvp-authz` or workflow code fails the test (RC-17); `Uuid::now_v7`,
  `Utc::now` and `SystemTime::now` are rejected by `disallowed-methods` in those crates.
- AC-30.1.6 (web): `dependency-cruiser` fails when a `web/features/*` module imports another feature, or
  anything but `lib/api` and `lib/ui`. ESLint `no-restricted-syntax` fails on a custody-state or
  exception-type literal in `features/**`.
- AC-30.1.7 (size): a file over 800 lines, or a function over 50 lines (`clippy::too_many_lines`), fails
  (RC-19). `mvp-kernel` over 1,500 lines fails (risk R-2).

**US-30.2** As the implementer, I want lint and type rules that make illegal code unrepresentable.
- AC-30.2.1 (lints): every crate sets `[lints] workspace = true`. A fixture with `unwrap()`, `expect()`,
  `panic!`, `todo!`, `unimplemented!` or `dbg!` in non-test code fails `cargo clippy`. The same call in a
  `cfg(test)` module passes. A wildcard `_` arm in a `match` over `RequestStatus`, `CustodyState`,
  `LiveState`, `ExceptionPhase`, `Resolution`, `ResolutionKind`, `Effect`, `NodeKind`, `LaneMilestone`
  or `ErrorCode` fails (`wildcard_enum_match_arm`); an `#[allow]` without a reason fails (RC-04, RC-11).
- AC-30.2.2 (`unsafe`): every crate root has `#![forbid(unsafe_code)]`. A fixture with an `unsafe`
  block fails to compile. A grep gate fails if `anyhow` appears outside a binary's `main.rs` (RC-05, RC-10).
- AC-30.2.3 (newtype IDs): a `trybuild` compile-fail test passes a `ItemId` where an `AssetId` is
  expected (and `LaneId` for `RequestId`, `TeamId` for `ClusterId`), and each fails to compile. A grep
  or `disallowed-types` gate fails on a bare `Uuid`, `String` or `i64` identifying an entity in a
  signature (RC-01).
- AC-30.2.4 (parse, don't validate): `Notes`, `SrNumber`, `ShortCode`, `SlaHours` and `RequestNumber`
  have private fields and fallible constructors; a compile-fail test that builds one with a struct
  literal fails. Boundary tests cover 0, 1, max, max+1, whitespace-only and control characters, and a
  `serde` round trip rejects invalid strings (RC-02).
- AC-30.2.5 (exhaustiveness): a compile-fail fixture adds a variant to `ExceptionPhase` and one to
  `ResolutionKind` and shows the build breaking at every `match` that must handle them (RC-04).
- AC-30.2.6 (determinism): `mvp-engine`, `mvp-custody` and workflow code use `BTreeMap` and `BTreeSet`
  only (`HashMap` is rejected); decimals use `rust_decimal`, never `f32` or `f64` (RC-07, RC-08).
  `settle` applied twice to equal inputs gives equal outputs.

**US-30.3** As an operator, I want one error vocabulary and one wire contract (F-4).
- AC-30.3.1 (error mapping): a test enumerates every `ErrorCode` in `mvp-contract` and every code in
  §11, §24.2, §27.9 and §28.8, and asserts each has an HTTP status (for example `NO_SUBSTITUTE` 422,
  `REPLACEMENT_LIMIT` 409, `SOD_VIOLATION` 403, `CLONE_MAPPING_INVALID` 422). There is exactly one
  `From<DomainError> for ApiError`, and it has no `_` arm. Adding a `DbError` variant without a mapping
  fails to compile. A fake `EngineGateway` drives one test per route: a `GatewayError::Rejected(code)`
  becomes that code's status and envelope; `Unavailable` and `TimedOut` become 503.
- AC-30.3.2 (Temporal names): every update, signal, query, workflow and task-queue name is a `const` in
  `mvp-contract`. A literal grep fails when a registered name appears as a string literal in `mvp-api` or
  `mvp-worker`. The registered-name test asserts the SDK registrations equal the constants (risk R-3).
  Golden payload tests fail when a payload field is renamed, removed or retyped (RC-14, RC-16).
- AC-30.3.3: every inbound HTTP DTO has `deny_unknown_fields` (a request with an extra field → 422 and
  no side effect); Temporal payloads and `CompiledPlan` do not, and new fields are `#[serde(default)]`.

**US-30.4** As an operator, I want history to replay across engine versions (F-5).
- AC-30.4.1
  - G: the replay corpus holds a golden `CompiledPlan` for every `PLAN_FORMAT` plus recorded
    event-to-effect traces, and the exports of S1–S12 (AC-13.2.1).
  - T: `CompiledPlan` carries `plan_format: u16`; the corpus replays against the current worker with
    no non-determinism error.
  - W: a fixture changes a `CompiledPlan` field in a breaking way without bumping `PLAN_FORMAT`.
  - T: the replay test fails. The fix, a bump with a reader for the earlier format and a new golden file,
    passes. Behaviour changes use `patched()` (§23.3).

**US-30.5** As the maintainer, I want custody and authorization rules to live in one place (F-1, F-2, F-3, F-8).
- AC-30.5.1 (single transition table): `mvp_custody::transition` is the only custody transition table.
  The child-update validator, the activity guard and `/actions` all call it. A bite check removes one
  cell (for example `prepared → handover_pending`): a validator test, an activity test and an `/actions`
  test all fail. SQL guards check only `version` and `state = from`. A grep fails on a custody-state
  transition literal outside `mvp-custody` and the `mvp-db` row mapping.
- AC-30.5.2 (server-driven actions): `GET /api/requests/:id/actions` returns the action list from
  `available_actions` and `authorize`. A fixture where the server returns an action for a state the UI
  has never seen renders it; a fixture where the server omits an action hides it. The UI contains no
  custody rule (ESLint, AC-30.1.6). A new custody state needs no `web` change.
- AC-30.5.3 (single authorization): `mvp_authz::authorize` and `separation_of_duties` are called by the
  API pre-check, the validator and the activity, each over `ActorFacts` from one `load_actor_facts`. A
  bite check that breaks the delegate-window test in `authorize` makes the API test, the validator test
  and the activity test all fail (F-3).
- AC-30.5.4 (milestone contract): the engine's per-item verdict is a function of `LaneMilestone`, lane
  origin and join policy only. A test replaces every custody state with a milestone table and
  `per_item_verdict` gives the same answer as S10 and S11. A supplementary lane never changes `issued` (F-2).
- AC-30.5.5 (effect cards): `/plan` carries exception bands, the Interrupts band and effect cards built
  by `describe_effects`; the web renders them without re-implementing the matrix (F-8, AC-25.4.1).
