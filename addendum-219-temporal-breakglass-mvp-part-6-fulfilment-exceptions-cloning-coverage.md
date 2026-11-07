# Addendum 219 (part 6) — Drawn fulfilment-exception handlers, workflow cloning and the 100% coverage mandate

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12); part 2 `…-part-2-stories.md` (§14, ST-0–ST-14); part 3 `…-part-3-verification-and-prompt.md` (§13, §15–§19).
- Part 4 `…-part-4-workflow-definition-and-editor.md` (§20–§26); part 5 `…-part-5-editor-stories.md` (ST-15–ST-24, ST-33).
- **This part (§27–§29).** Part 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34); part 9 `…-part-9-exception-runtime-and-cloning-detail.md` (§35–§37, ST-31; refines this part).
- Part 10 `…-part-10-exception-ui-and-walkthroughs.md` (§38–§40, ST-32); Appendix A `…-appendix-a-illustrative-example.md`.

**Why this part exists.** The user asked for five things (2026-11-08):
- The fulfilment unhappy paths after approval must be **visible and customisable**. Parts 1–4 left
  them platform-owned and undrawn (part 4 §21.3), so a team could not see how a request flows through
  an exception, nor assign its own people to it.
- They must be **fleshed out**: an item missing and impossible to issue, lost or damaged after
  issuance, and their permutations. The demo workflow must cater for all of them.
- A **new system can clone its workflow** from any version of an existing system.
- Every Rust crate and frontend module must reach **100% coverage**; only the gate is specified.
- The design is sharpened by modularity review (part 8).

**Precedence.** This part is normative and wins over parts 1–5. §27.13 lists what it supersedes.

---

## §27 Fulfilment exception handlers

### §27.0 The problem and the split of responsibility

Part 1 §9 defines what happens to a custody case when something goes wrong. Part 4 §21.3 left that
"lane-level platform behaviour" undrawn. The user rejected two consequences:
- Nobody can see **which path** an exception took, who held it, or why.
- A team cannot say "damaged items in custody go to the Asset Store; the Ops Leads approve the replacement".

> **The split.** The **designer owns who acts, in what order, with which approvals and SLAs.** The
> **platform owns what each resolution does** to custody, items, assets and the per-item join. The
> platform's effects are not hidden: every `resolution` terminal shows an **effect card** (§27.10)
> generated from the same code that executes the effect.

Exceptions become **handlers**: sub-graphs of the **lane template** (the `subject=item` template that
a root embeds once per item). Each has a mini-canvas, a catch filter, and node instances that render
on the request graph. Handlers belong to the template's definition; call-site flags (§27.5.1) tune
the limits per root.

### §27.1 Decisions added by this part (continued in §28.1 and §29.1)

| ID | Decision | Rationale |
|---|---|---|
| D-28 | **Drawn exception handlers.** A lane template holds an ordered list of handlers. Each is an `exception_start` with a catch filter `(type set, phase set, severity set)`, followed by `investigate`, `resolution_approval`, `resolution` and the permitted generic nodes. The lowest ordinal whose filter matches handles the exception. A catch-all fallback handler is mandatory. Routing is a pure function of the pinned `CompiledPlan` | Requested by the user. Makes the path visible, auditable and customisable |
| D-29 | **Exception phases** `pre_issuance`, `in_custody`, `return`, derived from the case state (§27.2). Resolution effects depend on `(resolution, phase, type class)` (§27.5). A resolution that makes no sense in a phase is invalid at validate time (V-35) and refused at runtime | Losing an item before the requester holds it differs physically and legally from losing it afterwards |
| D-30 | **Post-issuance replacement through supplementary lanes.** After a lane has crossed receipt, an approved replacement does not revoke issuance: the original lane ends or continues (§27.5) and a lane with `origin=supplementary` is added. It never gates `issued`; it only gates `completed`. Damaged items can end in `quarantine_return`. Replacements are bounded by `max_replacements_per_item` (0..3) and `allow_post_issuance_replacement` | Issuance cannot be revoked (V-16, §9.3). Without this the user's "lost or damaged after issuance" cases had no replacement path |
| D-31 | **Platform interrupts are drawn as locked nodes.** Auto-expire, tombstone auto-cancel and admin reset appear as three locked `platform_interrupt` nodes in an **Interrupts band**, and render on the request graph when they fire. A designer still cannot edit or remove them. This refines D-26 (undrawn) without weakening it | Removes the last invisible part of the lifecycle |

### §27.2 Phase derivation

`ExceptionPhase::of(state)` is a total function over the custody state. The phase is computed once,
when the exception opens, and stored in `custody_exceptions.phase`.

| Case state when the exception is reported | Phase | `lane.issued` |
|---|---|---|
| `awaiting_preparation`, `prepared`, `handover_pending` | `pre_issuance` | false |
| `in_requester_custody` | `in_custody` | true |
| `return_pending`, `received_by_agency` | `return` | true |
| `exception_open` (an exception is already open) | not applicable: one open exception per case (§9.2) | — |
| `returned_to_inventory`, `resolved_exception`, `cancelled` | not applicable: terminal, no exception can open | — |

`lane.issued` is the **lane's own** milestone, `crossed_receipt(c)`. It is not the request's status.
Consequently:
- A lane can be `in_custody` while the request is still `approved`, because a sibling lane has not
  yet crossed receipt.
- A lane (for example a supplementary lane) can be `pre_issuance` while the request is already
  `credential_issued`.

For an exception detected while a case is `exception_open`, the engine uses
`state_before_exception`, as `crossed_receipt` already does (§9.1).

**Type classes** (used by the effects matrix):

| Class | Exception types |
|---|---|
| `lost` | `missing`, `unavailable` |
| `damaged` | `damaged` |
| `other` | `wrong_item`, `identity_mismatch`, `quantity_mismatch`, `handover_disputed`, `receipt_disputed`, `return_disputed`, `other` |

**Impossible pairs** (greyed in the coverage grid, excluded from V-32): `handover_disputed` and
`receipt_disputed` occur only in `pre_issuance`; `return_disputed` only in `return`.

### §27.3 Node kinds inside a handler

Handler nodes carry `handler_id` (§27.7). Edges never leave a handler (V-33).

| Kind | In | Out-ports | Configuration | Exception status shown |
|---|---|---|---|---|
| `exception_start` (exactly 1 per handler) | none | `caught` | the catch filter (below); `ordinal`; `label` | `open` |
| `investigate` | 1 | a **non-empty subset** of `recovered`, `false_report`, `replacement`, `write_off`, `waiver`, `quarantine` | assignee team `(agency cluster, level)`; `sla_hours` plus `escalate_to_team`; `allow_info_hold` (default true) | `investigating`; `awaiting_information` during an info hold |
| `resolution_approval` | 1 | `approve`, `reject` | `assignee_kind` ∈ {`owner`, `team`}. `owner` is the primary or backup owner or an active delegate. `team` is an agency team of the system's organisation. Optional `sla_hours` | `awaiting_resolution_approval` |
| `resolution` (terminal, ≥1 per handler) | 1 | none | `resolution_kind` ∈ {`resume`, `cancel_false_report`, `replace`, `write_off`, `waive`, `quarantine_return`}; for `resume`, `resume_state_mode` ∈ {`state_before_exception`, `investigator_choice`} | `resolved`, or `cancelled_as_false_report` |
| `rule`, `switch`, `set_variable`, `task`, `wait`, `notify` | as in §21.2 | as in §21.2 | `task` assignee is the submitter or a team. They may read the `exception` scope (§27.4) and `item.*` | unchanged |

**Catch filter.** One row per allowed value in `wf_exception_handler_filters`, with `dimension` ∈
{`exception_type`, `phase`, `severity`}. A dimension with no rows matches every value. Severity
values are `low, medium, high, critical`.

**Matching and order.**
- The engine tests handlers in ascending `ordinal`. The first handler for which all three dimensions
  match wins; later handlers are not consulted.
- Exactly one handler per lane template has `is_fallback=true`. It has no filter rows and the highest
  ordinal, so it always matches. A definition without one is invalid (V-32).

**Port-to-resolution pairing** (V-33). Each `investigate` port may lead only to a `resolution` of its
own kind, directly or through other handler nodes:

| Port | Leads to `resolution_kind` | Needs `resolution_approval.approve` on the path |
|---|---|---|
| `recovered` | `resume` | no |
| `false_report` | `cancel_false_report` | no |
| `replacement` | `replace` | **yes** (V-34) |
| `write_off` | `write_off` | **yes** (V-34) |
| `waiver` | `waive` | **yes** (V-34) |
| `quarantine` | `quarantine_return` | no (a designer may add one) |

A `resolution_approval` has exactly one `approve` edge, so it leads to one resolution kind. The
**proposal** recorded in `custody_resolution_proposals` takes its `resolution` from the kind that the
approval's `approve` path reaches (computed at compile time). `reject` must lead back to an
`investigate` node of the same handler (V-37); that visit creates a new node instance.

**Info hold.** `POST /api/exceptions/:id/update` (kept) moves the exception between `investigating`
and `awaiting_information` on the **same** `investigate` instance. While on hold, `act` on it returns
409 `INVALID_EXCEPTION_STATUS`. The first `open → investigating` update is also required before any
`act` (AC-9.2.1 is unchanged).

**Separation of duties on `resolution_approval`.** The approver must not be the submitter, the
reporter, or **any actor who acted on an `investigate` instance of this exception** (the
proposer or investigator). Denial is 403 `SOD_VIOLATION`, re-checked inside the activity
transaction (D-5). This widens D-6, which named only the proposer, to cover team approvers.

### §27.4 The `exception` variable scope (read-only)

Platform variables from the engine catalogue, not `wf_variables` rows (part 11 D-45). Available to
`rule`, `switch`, `task` visibility conditions and `notify` inside handlers. Never writable (V-22).

| Name | Type | Meaning |
|---|---|---|
| `exc.type` | enum (the 10 exception types) | the reported type |
| `exc.severity` | enum {low, medium, high, critical} | the reported severity |
| `exc.phase` | enum {pre_issuance, in_custody, return} | §27.2 |
| `exc.detected_state` | enum (case states) | `detected_during_state` |
| `exc.reported_by_role` | enum {submitter, stage_team, verifier} | `verifier` when reported by the `verify_return` team, including the automatic report of §27.5.3 |
| `lane.replacement_count` | int | length of this lane's `replaces_lane_id` chain (`replacement_depth`); refined by part 9 §35.0 G-5: slot-wide, via `request_plan_lanes.slot_root_lane_id` |
| `lane.issued` | bool | `crossed_receipt` of this lane |
| `substitute_available` | bool | at least one substitute asset is free (below) |

**`substitute_available`** is system-computed at `investigate` entry **and re-checked inside the
activity** that applies a replacement (the race with another request is real). A substitute is a
live, non-retired, **non-quarantined**, unheld, manual asset in the same system with the same
`credential_type` as the item. The original asset does not count. V-20 is extended: `exc.*`,
`lane.replacement_count`, `lane.issued` and `substitute_available` may be used only inside handlers.

**Offered ports.** At runtime the replacement port is shown disabled, with a reason, when any
applies. The first match in this order is reported:

| Order | Condition | Code on attempt |
|---|---|---|
| 1 | `lane.issued` and `allow_post_issuance_replacement=false` | 409 `POST_ISSUANCE_REPLACEMENT_DISABLED` |
| 2 | `lane.replacement_count >= max_replacements_per_item`, or `allow_replacement=false` | 409 `REPLACEMENT_LIMIT` |
| 3 | `substitute_available=false` | 422 `NO_SUBSTITUTE` |

A disabled port performs no write when attempted. `substituteAssetId` (field on the replacement
port) must belong to the candidate set; otherwise 422 `SUBSTITUTE_INVALID`.

### §27.5 Platform effects

#### §27.5.1 Replacement bounds and call-site flags

| Property (on the per-item embed call site) | Range | Default | Meaning |
|---|---|---|---|
| `max_replacements_per_item` | 0..3 | 1 | maximum chain length per original item slot (slot-wide, part 9 §35.0 G-5). 0 forbids replacement |
| `allow_post_issuance_replacement` | bool | true | gates replacement when `lane.issued` |
| `allow_replacement`, `allow_write_off` (part 4) | bool | true | master switches. `allow_replacement=false` equals a limit of 0. `allow_write_off=false` removes the `write_off` port from the runtime offer |

A supplementary lane and a replacement lane both lengthen the chain by one.

#### §27.5.2 The effects matrix

This extends part 1 §9.3. Rows marked **base** keep their §9.3 meaning. "Lane excluded" means
`request_plan_lanes.excluded_at` is set and the lane drops out of the per-item join; "issuance stands"
means the lane is **not** excluded and still counts as crossed.

| Resolution | Phase · class | Case | Item and asset | Lane, join, summary | Ledger events |
|---|---|---|---|---|---|
| `resume` (recovered) | any | resume at the investigator's choice. Allowed set: `pre_issuance` {`state_before_exception`}; `in_custody` {`in_requester_custody`, `received_by_agency`}; `return` {`return_pending`, `received_by_agency`}. Anything else: 422 `RESUME_STATE_INVALID` | none | unchanged. Stage SLA re-armed | `asset_recovered` |
| `cancel_false_report` | any | resume at `state_before_exception` | none | unchanged | `exception_cancelled` |
| `waive` | any | resume at `state_before_exception` | none. `expected_return_condition` is **not** changed | unchanged | `waiver_granted` |
| `replace` | `pre_issuance` · any (**base**) | original → `resolved_exception` | original item excluded (`replaced`). Asset retired `lost_replaced` if class is lost or damaged, otherwise hold released. New item (`replaces_item_id`), hold on substitute, new case at `awaiting_preparation` | original lane excluded. New lane `origin=replacement`, `replaces_lane_id`, `replacement_depth+1`. `items_replaced`+1 | `replacement_authorized` |
| `replace` | `in_custody` · lost | original → `resolved_exception` | original item **not** excluded. Asset retired `lost_replaced`, hold released. New item, hold and case as above | issuance stands. New lane `origin=supplementary`. Outcome `replaced`. `items_replaced_after_issuance`+1 | `replacement_authorized` |
| `replace` | `in_custody` · damaged | original resumes at `in_requester_custody` with `expected_return_condition=damaged` | asset keeps its hold; it is quarantined at return (§27.5.3). New item, hold and case for the substitute | issuance stands. New lane `origin=supplementary`. `items_replaced_after_issuance`+1 | `replacement_authorized` |
| `replace` | `in_custody` · other | original resumes at `in_requester_custody`; `expected_return_condition` stays `good` | as the damaged row, but the asset is not quarantined unless found damaged | as the damaged row | `replacement_authorized` |
| `replace` | `return` | **not permitted** (V-35; the engine refuses it) | — | — | — |
| `write_off` | `pre_issuance` (**base**) | → `resolved_exception` | item excluded (`written_off`); asset retired `written_off`; hold released | lane excluded. If no live lane remains: `AllExcluded` (D-13) | `asset_written_off` |
| `write_off` | `in_custody` or `return` | → `resolved_exception` | item **not** excluded; asset retired `written_off`; hold released | issuance stands; join unchanged. `items_written_off`+1 | `asset_written_off` |
| `quarantine_return` | `in_custody` (damaged or other) | resume at `in_requester_custody` with `expected_return_condition=damaged` | none yet; quarantine happens at return (§27.5.3) | unchanged | none |
| `quarantine_return` | `return` | → `resolved_exception`. If `returned_at` is null it is set now, with `return_verified_by` = the acting investigator | `assets.quarantined_at/by/reason='damaged_on_return'`; hold released; the asset is **not** restocked (`restocked_*` stay null) | outcome `returned_damaged`. `items_returned`+1, `items_returned_damaged`+1 (at resolution, not at `return_received`) | `return_received` (if newly set), `return_condition_recorded`, `asset_quarantined` |
| `quarantine_return` | `pre_issuance` | **not permitted** (V-35) | — | — | — |

**Rules that apply to every row.**
- The ledger event, the D-19 timestamps, the item/asset/lane columns and the audit row are written
  in **one** transaction by the activity `apply_resolution`, guarded by `custody_cases.version`.
- A lane in `resolved_exception` is terminal: its child completes and it counts for `completed`.
- A **supplementary lane never counts** in the `issued` predicate of D-13, whether it is live or not.
  It counts for `completed`. A replacement lane (`origin=replacement`) counts in `issued` as today.
- Replacement of a lane whose own `lane.issued` is false gets `origin=replacement`; of a lane with
  `lane.issued` true, `origin=supplementary`.
- **Quarantine** sets `assets.quarantined_at/by` and a reason. A quarantined asset is excluded from
  `/requests/new` (409 `ASSET_QUARANTINED` on submit) and from substitutes, and is never restocked.
  There is no release path in the MVP (repair is Future, part 3 §17).
- A waiver given for damage after issuance does not alter the expected return condition. A later
  damaged return is therefore unexpected and raises a **new** return-phase exception (§27.8).

#### §27.5.3 The return condition

`custody_action verify_return` gains the field `condition` ∈ {`good`, `damaged`} (default `good`).
The case records `custody_cases.return_condition`.

| Expected | Actual | Result |
|---|---|---|
| `good` | `good` | as today |
| `damaged` | `damaged` | recorded; no exception. At `return_to_inventory` the asset is **quarantined instead of restocked**; outcome `returned_damaged` |
| `good` | `damaged` | **unexpected**. In the same transaction as `return_received`: record `return_condition_recorded`, then open an exception (`type=damaged`, `severity=medium`, reporter = the verifier, phase `return`, `detected_during_state=received_by_agency`). It is routed like any exception (usually to a `return` handler) |
| `damaged` | `good` | recorded as `good`; the asset is restocked. Quarantine follows the **actual** condition |

The combined action `verify_and_restock` with `damaged` performs the verify half only when the
damage is unexpected (no restock, exception opens); when expected, it quarantines instead of
restocking. Combined actions with `good` are unchanged.

### §27.6 Validation: V-32 … V-43

These extend §22. They use the same `Issue` shape and run in the same pure function. Prior-art
column: no code is ported from `sidequest/src/validation.ts`.

| Code | Sev | Rule | Prior art |
|---|---|---|---|
| **V-32** | error / warning | Exactly one fallback handler per lane template: highest ordinal, no filter rows. Every possible (type × phase) pair, excluding the impossible pairs of §27.2, is caught by some handler. A handler that can never win because earlier handlers cover its whole filter is **warning** (`HANDLER_SHADOWED`) | — |
| **V-33** | error | Handler well-formed: one `exception_start`; every node reachable from it; every path ends at a `resolution`; no edge enters or leaves the handler; every `investigate` port leads only to its own resolution kind (§27.3) | — |
| **V-34** | error | `replace`, `write_off` and `waive` resolutions are reachable only through a `resolution_approval.approve` edge | — |
| **V-35** | error | Resolution kind fits the phases the handler can catch (its phase filter, or all three): no `replace` if `return` is catchable; no `quarantine_return` if `pre_issuance` is catchable. `resume` with `investigator_choice` is always valid because the allowed set is derived from the runtime phase | — |
| **V-36** | warning | A `resolution_approval` team equals the `investigate` team of the same handler | — |
| **V-37** | error | The only cycle inside a handler is `resolution_approval.reject` → `investigate`; each `reject` is wired to an `investigate` node | — |
| **V-38** | error | An `investigate` node with a `replacement` port also has a fallback port: `write_off` (with `allow_write_off=true`) or `recovered` | — |
| **V-39** | error | No handler path reaches a top-level node or terminal | — |
| **V-40** | error | `sla_hours`, `duration_hours` and task due values inside handlers are in 1..720 (extends V-25) | — |
| **V-41** | error | No `custody_stage` node inside a handler | — |
| **V-42** | error | `ASSIGNEE_UNRESOLVED`: a node still carries an unresolved assignee or escalation placeholder (§28.4). Blocks simulate and publish | — |
| **V-43** | warning | An `allowed_credential_types` entry of a per-item embed has no live asset in the system (typical after a clone, since assets are never copied) | — |

Extensions of earlier codes: V-04…V-09 apply inside each handler with `exception_start` as its
start and `resolution` as its terminals; V-20 and V-22 are extended as in §27.4; V-25 range checks
include handler nodes; V-31 also protects the platform-locked interrupt nodes (error code
`PLATFORM_NODE_LOCKED`; the `expired` terminal keeps `PLATFORM_TERMINAL_LOCKED`).

### §27.7 Data model additions (no JSONB)

These follow §6 and §23.1 conventions: UUIDv7, liveness-aware foreign keys, and immutability of
published definitions (the same trigger covers every new definition table).

**New enums.** `exception_phase` (`pre_issuance, in_custody, return`); `return_condition`
(`good, damaged`); `handler_resolution_kind` (`resume, cancel_false_report, replace, write_off,
waive, quarantine_return`); `resume_state_mode` (`state_before_exception, investigator_choice`);
`filter_dimension` (`exception_type, phase, severity`); `interrupt_kind` (`expire, tombstone_cancel,
admin_reset`); `reported_by_role` (`submitter, stage_team, verifier`).

**Enum additions.**
- `wf_node_kind` gains `exception_start`, `investigate`, `resolution_approval`, `resolution`,
  `platform_interrupt`.
- `exception_resolution` gains `quarantine`.
- `asset_custody_event_type` gains `asset_quarantined`, `return_condition_recorded`.
- The history outcome gains `returned_damaged`.
- `request_plan_lanes.origin` gains `supplementary`.

**New definition tables.**

| Table | Key columns | Constraints |
|---|---|---|
| `wf_exception_handlers` | definition_id (the lane template's), handler_key `^[a-z][a-z0-9_]{0,39}$`, label(80), ordinal smallint, is_fallback bool | unique (definition_id, handler_key); unique (definition_id, ordinal); partial unique one `is_fallback` per definition. V-32 checks "fallback has the highest ordinal" |
| `wf_exception_handler_filters` | handler_id, dimension, value varchar(40) | unique (handler_id, dimension, value); CHECK the value belongs to the dimension's domain; none allowed for the fallback |

**Columns added to `wf_nodes`.**
- `handler_id` NULL (set for every handler node).
- Investigate and approval: `allow_info_hold bool`. `assignee_kind` is reused (`owner` or `team` on
  `resolution_approval`), as are `assignee_cluster_id/level`, `sla_hours` and `escalate_to_*`.
- Resolution: `resolution_kind`, `resume_state_mode` (NULL unless `resume`).
- Per-item embed call site: `max_replacements_per_item smallint CHECK (0..3)`, `allow_post_issuance_replacement bool`.
- Interrupts: `interrupt_kind`, `is_platform_locked bool` (a trigger forbids edits and deletes of locked
  nodes in a draft).
- Cloning (§28.4): `assignee_unresolved bool`, `assignee_hint varchar(160)`, `escalate_unresolved bool`,
  `escalate_hint varchar(160)`; CHECK unresolved ⇒ the matching cluster id is NULL; a publish trigger
  rejects any unresolved flag.
- **Dropped:** `investigate_cluster_id`, `investigate_level`, `investigate_due_hours` (part 4 §23.1).

A trigger on `wf_edges` requires both endpoints to share the same `handler_id` (both NULL, or equal).

**New request tables.**

| Table | Key columns | Notes |
|---|---|---|
| `request_exception_routes` | exception_id (unique), plan_run_id, lane_id, handler_id, matched_ordinal, rendered_match varchar(500), routed_at, op_ref | `rendered_match` example: `H3 Damaged in custody (ordinal 2): type damaged ∈ {damaged, identity_mismatch}; phase in_custody ∈ {in_custody}; severity any`. Written once, never updated |
| `request_exception_route_candidates` | route_id, handler_id, ordinal, matched bool, reason varchar(200) | one row per handler tested, in order, so the UI can show "H1 skipped: phase pre_issuance ≠ in_custody" |

**Columns added to existing tables.**

| Table | Columns |
|---|---|
| `request_node_instances` | `exception_id` NULL (handler instances); `cancelled_by_node_instance_id` NULL (interrupts) |
| `custody_exceptions` | `phase exception_phase NOT NULL`, `reported_by_role`, `case_exception_no smallint` (unique per case: numbers the bands) |
| `custody_cases` | `expected_return_condition return_condition NOT NULL DEFAULT 'good'`, `return_condition NULL`. Both obey the D-19 "set once" trigger once `return_condition` is written |
| `assets` | `quarantined_at`, `quarantined_by`, `quarantine_reason varchar(200)`; CHECK all three set together; set once |
| `request_plan_lanes` | `origin` gains `supplementary`; `replacement_depth smallint NOT NULL DEFAULT 0` |
| view `request_issuance_summary` | adds `items_returned_damaged`, `items_replaced_after_issuance`. `items_returned` counts returns of either condition, at resolution (quarantine) or restock (part 9 §35.7) |
| view `asset_custody_history` | `outcome` gains `returned_damaged`; adds `return_condition`, `quarantined_at` |

**New audit actions:** `custody.exception_routed`, `custody.asset_quarantined`, `custody.return_condition_recorded`, `custody.supplementary_lane_started`, `workflow.interrupt_fired`.

### §27.8 Runtime in `LaneWorkflow`

Handlers run **inside the lane's child workflow**, so they survive parent continue-as-new (D-8).

**Reporting and routing.**
1. `report_exception` (and the automatic reports of §27.5.3 and the disputes of §9.1) passes the
   update validator unchanged: no open exception, state not terminal.
2. The workflow builds `ExceptionFacts` and calls the **pure** function
   `mvp_engine::route_exception(&CompiledPlan, &ExceptionFacts) -> RouteDecision` (selected handler
   plus every candidate with its match result). Nothing in it reads the database or the clock.
3. One activity, `open_exception` (extended), writes in one transaction: the exception (with
   `phase`, `reported_by_role`, `case_exception_no`), the case move to `exception_open`, the
   `request_exception_routes` and candidate rows, the `exception_start` instance (completed) and
   the first `investigate` instance (active), the evidence attachments, the event and the audit rows
   `custody.exception_reported` and `custody.exception_routed`.
4. The lane's state holds `handler: Option<HandlerRun{exception_id, handler_id, tokens}>`.
   `mvp-engine::settle` runs handler nodes with the same rules as lane nodes (automatic nodes run
   inside `settle`; humans, timers and tasks wait).

**Acting.** `POST /api/requests/:id/nodes/:nodeInstanceId/act` on a handler instance is sent to the
lane's child workflow (its `exception_id` identifies the child). For a lane-scoped instance,
`expectedRevision` is the case `version`; a mismatch is 409 `STALE_STATE`. The validator checks: the
instance is active, the outcome is a port of the node and currently offered (§27.4), the exception
status allows it, the decision lock is free, and notes are present where §9.2 requires them.

**Settlement.** An `investigate` port `replacement`, `write_off` or `waiver` creates the
`custody_resolution_proposals` row (and the status `awaiting_resolution_approval`). A direct port
(`recovered`, `false_report`, `quarantine`) proceeds along the drawn path. Reaching a `resolution`
terminal yields the effect `ApplyResolution{kind, params}`, executed by the activity
`apply_resolution` (§27.5.2), which also closes the exception and writes the update row. `reject`
sets the proposal `rejected`, returns to `investigating`, and creates a new `investigate` instance.

**Milestones to the parent** (extends §23.3 `lane_event`): `milestone{issued | excluded |
replaced{new_lane_id, origin} | terminal}`. `replaced` with `origin=supplementary` does not change the
`issued` predicate (refined by part 9 §35.0 G-6: a separate milestone `supplementary_started`). The parent starts the new lane's child on receipt.

**Rules for interactions.**

| Situation | Behaviour |
|---|---|
| One open exception per case | A report while one is open is 409 `EXCEPTION_ALREADY_OPEN` (AC-9.1.2) |
| Repeated exceptions on one lane | Each is routed afresh, gets the next `case_exception_no`, and renders as a numbered band. `lane.replacement_count` persists across them |
| Concurrent exceptions on different lanes | Independent handlers. The parent applies lane events serially. Two proposals for the same substitute: the second approval fails in the activity with 409 `ASSET_IN_USE` and writes nothing; the instance stays active |
| Close | Closing the request does not stop handlers; reporting and resolving continue (AC-9.5.4). The parent stays Running until every lane is terminal |
| Auto-expire (D-10), tombstone auto-cancel | Both apply only before owner approval, when no lane exists. They have **no effect** after approval |
| Submitter tombstoned after approval | Handlers continue. A `task` assigned to the submitter becomes actionable by an active member of the ordinal-1 team of the run's submit step, recorded `acting_as=submitter_team` |
| Admin reset, no force | 409 `CUSTODY_IN_PROGRESS` as D-14 |
| Admin reset, `force=true` | Cases before `handover_pending` are cancelled, including those with an open `pre_issuance` exception: its handler instances become `cancelled` (by the reset interrupt node), the exception is `resolved` with `resolution_notes='cancelled_by_admin_reset'` and no resolution value, event `exception_cancelled`. Cases at or past `handover_pending`, including those with open exceptions, **continue** and their handlers keep running |
| Handler timers | SLA and `wait` timers live in the lane child, are cancelled on leaving the node, and use `SLA_UNIT_SECONDS`. The investigate SLA replaces the exception SLA of §10 |

### §27.9 API

The §11 envelope and idempotency rules apply.

| Route | Change |
|---|---|
| `POST /api/requests/:id/nodes/:nodeInstanceId/act` | Now also acts on handler instances. Body `{outcome, fields?, notes?, expectedRevision}`. `fields` carries `substituteAssetId`, `resumeState` and task fields |
| `POST /api/exceptions/:id/resolve`, `/propose`, `/decide` | **Superseded** by `act`: `recovered`/`false_report`/`quarantine` ports replace resolve; `replacement`/`write_off`/`waiver` replace propose; `approve`/`reject` replace decide |
| `POST /api/cases/:caseId/exceptions` | Kept (report) |
| `POST /api/exceptions/:id/update` | Kept (info hold, `open → investigating`) |
| `POST /api/cases/:caseId/actions/verify_return` | Gains `condition` (§27.5.3) |
| `GET /api/exceptions/:id` | New. Route, candidates, the instance chain, `offeredPorts[{port, enabled, disabledReason}]`, the candidate substitutes, and the effect card for each reachable resolution |
| `GET /api/admin/platform/exception-effects` | New. The effects matrix (§27.5.2) from `mvp-custody`, so the UI never re-implements it |
| `GET /api/requests/:id/plan` | Adds `exceptionBands[]` (per lane: exception no, route, instances, taken edges), `routes[]` and the `interrupts` band with `fired` flags |

New error codes: `NO_SUBSTITUTE` (422), `SUBSTITUTE_INVALID` (422), `RESUME_STATE_INVALID` (422),
`REPLACEMENT_LIMIT` (409), `POST_ISSUANCE_REPLACEMENT_DISABLED` (409), `INVALID_EXCEPTION_STATUS`
(409), `ASSET_QUARANTINED` (409), `SOD_VIOLATION` (403), `PLATFORM_NODE_LOCKED` (422), `EXCEPTION_ALREADY_OPEN` (409), `RATE_LIMITED` (429); part 9 §35.8 is authoritative. Existing
`NODE_NOT_ACTIVE`, `OUTCOME_NOT_ALLOWED`, `STALE_STATE`, `ASSET_IN_USE` keep their meanings. Every
error above leaves no row changed and no workflow history event (the update validator or the guarded
activity refuses it).

### §27.10 Request-graph rendering and editor

**Request graph (R8, from `/plan`).**
- Each exception renders as a **numbered band beneath its lane** ("Exception #2"). The band shows the
  route chip ("H3 · Damaged in custody"), with the reason from `rendered_match` and the skipped
  handlers on expand; then its node instances with actor, remarks, SLA badge and taken edges.
- The `resolution` terminal shows its **effect card** in plain language, for example "Replace · after
  issuance · lost: original lane ends, HT-01 retired, issuance stands, supplementary lane HT-02
  started".
- Supplementary lanes are drawn with a dashed border and a "supplements" edge from the original lane.
- The **Interrupts band** sits at the bottom. A fired interrupt is highlighted and draws a "cancelled
  by" edge to each cancelled instance.

**Editor.**
- In the **template editor** of a `subject=item` template, the **Handlers** tab (beside Interface and
  Eligibility; the lane graph is the canvas). Each handler has a mini-canvas, a catch-filter form and
  an ordinal drag handle. The fallback is pinned last and cannot be deleted. The root editor has the
  **Interrupts** band only. Opening a per-item embed drills into the template read-only.
- **Coverage grid.** 10 types × 3 phases; each cell shows the chip of the handler that wins
  (or "fallback"). Impossible pairs are greyed. A severity selector re-evaluates the grid; a cell
  whose winner depends on severity shows "varies". Clicking a cell focuses the handler.
- **Inspector effect cards.** Selecting a `resolution` node shows the card for each phase it can
  receive, fetched from `GET /api/admin/platform/exception-effects`.
- The palette adds the handler kinds, enabled only in the Handlers tab. The "Connect port to…"
  dropdown and ARIA labels of §24.1 apply to them.
- **Simulation** (R10) accepts exception injection: `{kind:'exception', lane_no, type, severity,
  at_state, substitutes_available, replacement_count}` followed by `act` steps on handler nodes.
  `SimTrace` includes the `RouteDecision`. The simulator can inject every (type, phase) pair and
  every resolution of §27.5.2; the permutation matrix (§27.12) relies on this.
- Interrupt nodes are drawn locked (padlock, no handles).

### §27.11 Seeded HRIS v1 handlers

The HRIS handlers are those of `standard_custody_lane` v1 (AUR library), embedded once per item by
HRIS's `items` embed, whose call site sets `max_replacements_per_item=1` and
`allow_post_issuance_replacement=true`. Handlers are evaluated in this order (so H3 sits before H2):

| Ordinal | Handler | Catch | Investigate | Ports → approval → resolution |
|---|---|---|---|---|
| 1 | **H1** Before issuance | `pre_issuance`, all types and severities | Ops Leads L2, 48 h | `recovered` → `resume` (`state_before_exception`); `false_report` → `cancel_false_report`; `replacement`, `write_off`, `waiver` → owner approval → `replace`, `write_off`, `waive` |
| 2 | **H3** Damaged in custody | `in_custody`, types {`damaged`, `identity_mismatch`} | **Asset Store L1**, 8 h | `false_report`; `replacement` → approval by **Ops Leads L2 team** → `replace`; `waiver` → owner → `waive`; `write_off` → owner → `write_off`; `quarantine` → `quarantine_return` (direct; PM-21). The investigator team differs from the approver team |
| 3 | **H2** Lost in custody | `in_custody`, all (after H3) | Ops Leads L2, 24 h | `recovered` → `resume` (`investigator_choice`); `false_report`; `write_off` → `task loss_report` (submitter, requires `loss_report_done`) → owner approval → `write_off`; `replacement` → owner → `replace` |
| 4 | **H4** Problems at return | `return`, all | Ops Reviewers L1, 24 h | `recovered` → `resume` (`investigator_choice`); `false_report`; `quarantine` → `quarantine_return` (direct); `write_off` → owner → `write_off` |
| 5 | `fallback` | everything | Ops Leads L2, 48 h | `recovered`, `false_report`, `write_off` → owner, `waiver` → owner. No `replace` (V-35: it also catches `return`) |

Seed additions: assets **SP-03** (secure phone, device) and **LT-02** (laptop, device) on HRIS; the
call variable `loss_report_done` of `standard_custody_lane` (bool, false, field on `loss_report`). The §25.3 matrix
gains rows for every handler feature (part 4 is updated to point here).

FINDB v1 and OPSTOOL v1 embed `basic_custody_lane` v1 (AUR); PORTAL v1 embeds `sen_custody_lane` v1
(SEN). Each template holds **one handler**, the fallback: `investigate` by the agency investigate team
(AUR: Ops Leads L2; SEN: SEN Ops L1), 48 h, ports `recovered`, `false_report`, `write_off` (owner) and
`waiver` (owner). A system that wants replacement embeds a template with a phase-limited handler.

S3 (replacement before issuance) runs through H1; S4 (write-off after issuance) through H2, which
adds the `loss_report` task for Bob before Carol's decision (§27.13).

### §27.12 Unhappy-path permutation matrix

**Dimensions.** Phase (3) × type class (lost, damaged, other) × resolution (6) × substitute
availability (yes, no; relevant to `replace` only) × replacement-limit state (`below`, `at_limit`,
`post_issuance_disabled`; relevant to `replace` only). Impossible and refused combinations are rows
too, so that the refusal is proved.

> **Coverage rule.** The seeded workflow and the scenarios must cover **every row**, to show the
> engine's capability. Each row has (1) a named engine-level simulation test `sim_pm_NN` that injects
> the exception and asserts route, effects, summary counts and join, **and** (2) a real-database
> `apply_resolution` test `db_pm_NN` asserting the §27.5.2 effects. Where a scenario is named, it adds
> end-to-end proof. A meta-test (AC-27.10.1) fails if any row id lacks either test or if
> `fixtures/permutation-matrix.csv` and this table differ.

| ID | Phase | Class | Resolution | Substitute | Limit state | Scenario | AC |
|---|---|---|---|---|---|---|---|
| PM-01 | pre | lost | replace | yes | below | S3, S10 step 1, S11 step 1 | AC-27.1.1 |
| PM-02 | pre | lost | replace | no | below | S10 step 2 (refused `NO_SUBSTITUTE`) | AC-27.1.2 |
| PM-03 | pre | lost | replace | yes | at_limit | — | AC-27.5.1 |
| PM-04 | pre | other | replace | no | at_limit | S11 step 2 (`REPLACEMENT_LIMIT` wins) | AC-27.5.1 |
| PM-05 | pre | damaged | replace | yes | below | — | AC-27.1.1 |
| PM-06 | pre | lost, damaged, other | write_off | n/a | n/a | S10 step 2, S11 step 3 | AC-27.1.3, AC-27.5.2 |
| PM-07 | pre | lost, other | resume, cancel_false_report | n/a | n/a | — | AC-9.3.1, AC-9.3.3 |
| PM-08 | pre | other | waive | n/a | n/a | — | AC-9.6.1, AC-9.6.2 |
| PM-09 | pre | other (dispute) | resume | n/a | n/a | — | AC-27.1.4 |
| PM-10 | in | lost | write_off | n/a | n/a | S4, S10 step 4 | AC-27.2.1 |
| PM-11 | in | lost | replace | yes | below | — | AC-27.2.2 |
| PM-12 | in | lost | replace | no | below | — | AC-27.2.2 |
| PM-13 | in | lost | replace | yes | at_limit | — | AC-27.5.1 |
| PM-14 | in | lost | replace | yes | post_issuance_disabled | — | AC-27.5.3 |
| PM-15 | in | lost | resume, cancel_false_report | n/a | n/a | — | AC-9.3.2, AC-9.3.3 |
| PM-16 | in | damaged | replace | yes | below | S10 step 5 | AC-27.3.1, AC-27.3.2 |
| PM-17 | in | damaged | replace | no | below | — | AC-27.3.1 |
| PM-18 | in | damaged | replace | yes | at_limit | — | AC-27.5.1 |
| PM-19 | in | damaged | replace | yes | post_issuance_disabled | — | AC-27.5.3 |
| PM-20 | in | damaged | write_off, waive | n/a | n/a | — | AC-27.2.1, AC-27.4.1 |
| PM-21 | in | damaged | quarantine_return | n/a | n/a | — | AC-27.4.2 |
| PM-22 | in | other | replace, waive | yes | below | — | AC-27.3.1 |
| PM-23 | return | damaged (unexpected) | quarantine_return | n/a | n/a | S11 variant b | AC-27.4.1 |
| PM-24 | return | damaged (expected) | none: no exception; quarantine at restock | n/a | n/a | S10 steps 5–6 | AC-27.4.2 |
| PM-25 | return | damaged, lost | write_off, cancel_false_report | n/a | n/a | — | AC-27.4.3 |
| PM-26 | return | other (dispute) | resume | n/a | n/a | — | AC-9.7.2 |
| PM-27 | return | any | replace (refused) | any | any | — | AC-27.4.4 |
| PM-28 | any | any | replacement chain up to limit 3, then refusal | yes | below → at_limit | — | AC-27.5.1 |
| PM-29 | mixed | four lanes, four paths at once | mixed | mixed | mixed | S10 | AC-27.6.1 |
| PM-30 | any | any | close, expire and admin-reset interactions (§27.8) | n/a | n/a | — | AC-27.7.1, AC-27.7.2, AC-27.7.3 |
| PM-31 | any | any | separation of duties on every approval | n/a | n/a | S3, S10 step 5 | AC-27.8.1 |

Part 7 (ST-27) also asserts for every row that the Issuance and returns panel, the CSV export and
`/assets/:id/history` agree on the per-item outcome (AC-27.9.1). **S10-X** (sidecar request R2, after step 6; part 10 §40.1): SP-01 at AP with an open H1 exception, SP-02 at HP; admin reset without force is 409 `CUSTODY_IN_PROGRESS`, with force cancels SP-01's case (PM-30) and SP-02 continues.

#### §27.12.1 Scenario S10: Mixed fulfilment exceptions (end-to-end on HRIS v1)

**Preconditions (set up through the public API, not SQL).** A background request R0 by Bob is issued
and holds **LT-02 and SP-03**, so no spare device is free. HT-02 is the only free token.

| Step | Script | Expected | AC |
|---|---|---|---|
| 1 | Bob submits HRIS with SP-01, SP-02, HT-01, LT-01 (4 lanes) and the approvals complete. Grace reports **HT-01 missing** at `awaiting_preparation`. Frank takes it, proposes `replacement` with HT-02. Carol approves | Routed to H1 (`rendered_match` shown). HT-01 lane excluded; asset retired `lost_replaced`. New lane HT-02 `origin=replacement`, depth 1, takes the token branch (`reset_pin`) | AC-27.1.1 |
| 2 | Grace reports **SP-02 missing**. Frank sees `replacement` disabled: `NO_SUBSTITUTE`; an attempt is 422 with no row changed. He proposes `write_off`. Carol approves | SP-02 item excluded, asset retired `written_off`. Join `live=3` (SP-01, LT-01, HT-02) | AC-27.1.2, AC-27.1.3 |
| 3 | SP-01, LT-01 and HT-02 are prepared, handed over and received (Bob does not close yet) | The `items` embed emits `issued`; status `credential_issued`; Bob's close allowed | AC-27.6.1 |
| 4 | Bob reports **LT-01 missing** (`in_requester_custody`). H2: Frank takes it, chooses `write_off`; Bob completes `loss_report`; Carol approves | LT-01 case `resolved_exception`; asset retired `written_off`; item **not** excluded; issuance stands; `items_written_off`=2 | AC-27.2.1 |
| 5 | R0 returns SP-03 (normal custody) so it is free. Bob reports **SP-01 damaged**; **S10-T** idles 80 s so the H3 SLA breach and escalation show (AC-26.4.2). H3: Grace takes it, proposes `replacement` with SP-03. Grace (not in the approval team) and Carol (team step, not owner class) trying to approve get plain 403 (`SOD_VIOLATION` is for the §27.3 rules); Frank approves | SP-01 lane resumes at `in_requester_custody` with `expected_return_condition=damaged`. Supplementary lane SP-03 (`origin=supplementary`, depth 1) starts. `issued` unaffected | AC-27.3.1, AC-27.3.2, AC-27.8.1 |
| 6 | SP-03 is prepared, handed over and received by Bob (6a); Bob closes the request (6b; status `closed`, 3 items outstanding). SP-01 is returned: Claire verifies with `condition=damaged` (expected: no exception); Grace's `return_to_inventory` quarantines it (6c). HT-02 and SP-03 are returned and restocked normally (6d) | SP-01 `quarantined_at` set, not restocked, outcome `returned_damaged`. All lanes terminal and the request closed: workflow completes. The graph shows 4 exception bands (HT-01, SP-02, LT-01, SP-01), numbered per lane | AC-27.4.2 |

**Final summary (AC-27.9.1).**

| items_total | items_issued | items_outstanding | items_returned | items_returned_damaged | items_replaced | items_replaced_after_issuance | items_written_off |
|---|---|---|---|---|---|---|---|
| 6 (SP-01, SP-02, HT-01, LT-01, HT-02, SP-03) | 4 (SP-01, LT-01, HT-02, SP-03) | 0 | 3 (SP-01, HT-02, SP-03) | 1 (SP-01) | 1 (HT-01) | 1 (SP-01) | 2 (SP-02 before, LT-01 after issuance) |

#### §27.12.2 Scenario S11: Cascade and exhaustion

**Variant a (closed unfulfilled).** Bob submits HRIS for **one token** (HT-01). HT-02 is the only
other token.

| Step | Script | Expected | AC |
|---|---|---|---|
| 1 | Grace reports HT-01 missing. H1: Frank proposes `replacement` (HT-02); Carol approves | HT-01 lane excluded. HT-02 lane `origin=replacement`, `replacement_depth=1` | AC-27.1.1 |
| 2 | HT-02 is prepared and handed over. Bob **disputes the handover** ("token cracked"). H1 routes it. Frank's offer shows `replacement` disabled: `REPLACEMENT_LIMIT` (depth 1 = max 1; also no substitute, but the limit is reported first). An attempt returns 409 with nothing written | Only `write_off`, `recovered`, `false_report` and `waiver` are offered | AC-27.5.1 |
| 3 | Frank proposes `write_off`; Carol approves | HT-02 excluded. Both lanes excluded: join `AllExcluded`. `close_unfulfilled` sets `closed`, `decision_reason='unfulfilled_all_items_written_off'`. The embed emits `unfulfilled`; workflow completes at `closed_unfulfilled`. Summary: total 2, issued 0, replaced 1, written off 1 | AC-27.5.2 |

**Variant b (returned damaged is quarantined).** Bob submits for SP-01 and the request is issued.
Bob initiates the return. Claire verifies with `condition=damaged`.

| Step | Script | Expected | AC |
|---|---|---|---|
| 1 | The verification is unexpected damage | One transaction: `return_received`, `return_condition_recorded`, and an auto-opened `damaged` exception (`reported_by_role=verifier`, `severity=medium`), routed to H4 | AC-27.4.1 |
| 2 | Claire (Ops Reviewers L1) takes it and acts on the `quarantine` port | `resolution` `quarantine_return`: SP-01 `quarantined_at/by/reason` set, hold released, never restocked; case `resolved_exception`; outcome `returned_damaged`. `/requests/new` shows SP-01 disabled; submitting it is 409 `ASSET_QUARANTINED` | AC-27.4.1 |

### §27.13 Supersession

| Clause | Replaced by |
|---|---|
| Part 1 §9.3 "Resolution effects" | §27.5.2. The `pre_issuance` rows are the §9.3 rows unchanged (**base**); the rest are new |
| Part 1 §9.2 exception transitions | Still the status vocabulary, now **projected** from handler instances (§27.3) |
| Part 1 §7.2 child updates `resolve_direct`, `propose_resolution`, `decide_resolution` | The update `act` on handler instances. `report_exception`, `exception_update`, `dispute_handoff` stay; `custody_action.verify_return` gains `condition` |
| Part 1 §7.3 activities `resolve_exception_direct`, `create_resolution_proposal`, `decide_resolution_proposal` | `apply_resolution` (+ `act` settlement). `open_exception` gains routing (§27.8) |
| Part 1 §8 "Exception update / resolve direct / propose" and "Decide resolution" rows | Eligible: the assignee of the active `investigate` (team) or `resolution_approval` (owner class or team) instance, with the §27.3 separation of duties |
| Part 1 §10 "Exception SLA" | The `investigate` node `sla_hours` (and optional approval `sla_hours`) |
| Part 1 §11 `POST /api/exceptions/:id/resolve`, `/propose`, `/decide` | `act` (§27.9). `/update` stays |
| Part 1 §12 "Exception side node" | Exception bands and the Interrupts band (§27.10) |
| Part 1 D-12 "replacement, write-off and waiver need owner approval" | They need approval by the `resolution_approval` assignee, which the designer chooses (owner class or agency team). D-6 is widened (§27.3) |
| Part 1 D-13 | The `issued` predicate ignores `origin=supplementary` lanes (§27.5.2). Nothing else changes |
| Part 4 D-26 "Platform interrupts are not drawn" | D-31 (drawn, locked). The behaviour (non-removable lifecycle guarantees) is unchanged |
| Part 4 §21.3 "Exceptions are not drawn" and the `investigate_team` property | §27.3. The `investigate_*` columns are dropped |
| Part 4 §21.2 per-item `embed` row | Adds call-site `max_replacements_per_item`, `allow_post_issuance_replacement`; handlers belong to the lane template |
| Part 4 V-20, V-22, V-25, V-31 | Extended as stated in §27.6 |
| Part 4 §23.2 `request_plan_lanes.origin ∈ {initial, replacement}` | Adds `supplementary` (§27.7) |
| Part 3 §15 S4 | Gains the `loss_report` task by Bob between Frank's proposal and Carol's decision. Its other assertions are unchanged |
| Part 2 ST-9 ACs naming `resolve`, `propose`, `decide` | Read as `act` on the named port; assertions unchanged |

---

## §28 Cloning a workflow from an existing system's version

### §28.0 The short answer

> An administrator can **clone any published or retired version of any system's workflow** into a new
> draft of the target system. Every definition row is copied and every team reference is
> **remapped**. Unmapped references become visible **unresolved placeholders** that fail V-42 and so
> block simulate and publish. Clone, draft-from-version and import share **one** copy function.

### §28.1 Decision

| ID | Decision | Rationale |
|---|---|---|
| D-32 | **Workflow cloning.** `POST …/clone` deep-copies an immutable source version into a draft of the target system, with an explicit `AssigneeMapping`. Same-organisation cloning needs an `org_admin` whose scope includes that organisation (D-38); cross-organisation cloning needs `full_admin` (OP-8, recorded in part 3 §18), even when an `org_admin`'s scope includes both organisations. The copy is atomic, idempotent and audited, and records lineage | Requested by the user. Saves rebuilding a graph; the unresolved-placeholder model prevents silently inheriting another system's teams |

### §28.2 Entry points

| Where | Action | Notes |
|---|---|---|
| Create-system wizard `/admin/systems/new`, step "Workflow" | Three choices: **Start blank (starter template)**, **Clone from system version**, **Import file** | Blank creates the draft from a built-in starter: `start → approval → owner_approval → per-item embed (1..5) of the organisation's default lane template (`basic_custody_lane`) → terminals`, with assignees as unresolved placeholders. The clone wizard also offers cloning the referenced templates (§28.4) |
| `/admin/systems/[id]/workflow` | **New draft from another system's version** (beside "New draft from vN") | Subject to the `DRAFT_EXISTS` rule: 409 if the target has a live draft |

Wizard flow for the clone choice:
1. Pick the source: organisation (a `full_admin` sees all; an `org_admin` sees only the target system's organisation, because cross-organisation cloning is `full_admin` only, D-38), system,
   then a published or retired version (shown with publisher, date and `content_sha256` prefix).
2. **Preview** shows counts, a mapping table and the referenced templates (tick "Also clone these
   templates"). The administrator maps each reference or leaves it unresolved, then confirms.
3. **Clone** runs, and the editor opens on the new draft with the V-42 issues listed.

### §28.3 What is and is not copied

| Copied (new UUIDv7 for every row) | Not copied |
|---|---|
| `wf_variables` and enum values; `wf_nodes` (all kinds, including handlers, fields and positions); `wf_node_ports`; `wf_edges`; `wf_expressions`; `wf_node_fields`; `wf_allowed_credential_types`; `wf_exception_handlers` and filters (of templates, when templates are cloned); embed pins, bindings and call-site properties; labels, SLAs, bounds and flags | Owners, delegates, PA mappings, vendor-cluster links, assets, request history, runs, variables' values, `change_notes`, version number, `content_sha256` (recomputed at publish). **Templates are not copied** unless the administrator ticks "Also clone these templates" |

**Locked platform nodes** (the `expired` terminal, the three interrupt nodes) are **re-seeded** from
the current platform template, never copied.

### §28.4 Mapping rules and placeholders

A **reference** is a distinct `(cluster, level)` pair used as an assignee or escalation target by a
node. A reference of kind owner, submitter or any-reviewer needs no mapping. Each distinct pair gets
one mapping row that applies to every use.

| Reference | Default target (same organisation) | Default (cross-organisation) | Suggestion offered |
|---|---|---|---|
| Agency team `(cluster, level)` | the **same** team | none | in the target organisation: cluster with the same name (case-insensitive) and the same level: `name_level` |
| Vendor team `(cluster, level)` | the same team **only if** its cluster is linked to the target system (`system_vendor_clusters`) | none | the cluster at the **same ordinal** in the target's `system_vendor_clusters`, same level: `ordinal_level` |

Rules:
- A mapping target must be a live team of the target system's organisation with the right cluster
  type (vendor for vendor references, agency for agency references) and never a `pa` cluster.
  Violations are 422 `CLONE_MAPPING_INVALID` with a `fields` path per entry.
- A reference with no default and no mapping becomes an **unresolved placeholder**:
  `assignee_unresolved=true` (or `escalate_unresolved`), the cluster id NULL, and the hint holds the
  source label (for example "Alpha Leads L2 (from HRIS v1, AUR)"). The editor shows a red chip
  "Choose team (was …)". Choosing a team clears the flag.
- **V-42** reports one error per node and slot. A draft with placeholders is saved and listed, but
  `simulate` and `publish` refuse it (V-42 is an error). Published definitions never contain
  placeholders (§27.7 trigger).
- **Embeds.** Same organisation: each embed keeps its pin and bindings. Cross organisation: each embed
  becomes an unresolved placeholder (`embed_unresolved=true`, pin cleared, `embed_hint` such as
  "standard_custody_lane v1 (from AUR)"); bindings and call-site properties are kept. V-42 is extended
  to report it (`EMBED_UNRESOLVED`). Choosing an eligible published template of the target
  organisation clears the flag (V-48 re-checks the bindings). Templates are cloneable too
  (`definition_copy`); "Also clone these templates" creates drafts in the target library and re-points
  the placeholders.
- **V-43** warns when a cloned per-item embed allows a credential type for which the target has no asset.

### §28.5 API

All routes: §11 envelope, `X-Idempotency-Token` on mutations.

**Preview** `POST /api/admin/systems/:id/workflow/clone/preview` with `{sourceDefinitionId}`.
Writes nothing. Response `200`:

```
{ data: {
    source: { definitionId, systemId, systemName, organisationId, organisationCode,
              version, status, contentSha256 },
    crossOrganisation: bool,
    counts: { nodes, edges, handlers, variables, references },
    references: [ { sourceClusterId, sourceClusterName, sourceLevel, kind: "vendor"|"agency",
                    usedBy: [ { nodeKey, slot: "assignee"|"escalation" } ],
                    defaultTarget: { clusterId, level }?,
                    suggestions: [ { clusterId, clusterName, level, basis: "same_team"|"name_level"|"cluster_level"|"ordinal_level", confidence, ambiguous } ] } ]
} }
```

**Clone** `POST /api/admin/systems/:id/workflow/clone` with
`{sourceDefinitionId, mappings: [{sourceClusterId, sourceLevel, targetClusterId, targetLevel}]}`.
Response `201`:
`{ data: { definitionId, status: "draft", revision, clonedFromDefinitionId, issues: [Issue] } }`.
`issues` always includes the V-42 and V-43 results.

### §28.6 One copy path

`mvp_db::definition_copy(source_rows, AssigneeMapping, Mode)` is the single implementation. It
reads the source rows, mints new ids, remaps references, resets status to `draft`, `version` to NULL
and `revision` to 1, and writes the target rows in the caller's transaction.

| Caller | Mapping | Mode for an unmapped reference |
|---|---|---|
| Draft from version (`POST …/workflow/draft`) | identity (same system) | not applicable |
| Import (`POST …/workflow/import`) | resolved by `(cluster name, level)` in the organisation | `Strict`: 422 with paths (part 4 §24.2) |
| Clone | the request's mappings plus defaults (§28.4) | `Placeholder` |

A unit test asserts that the three callers produce identical rows for identical inputs (finding F-6, part 8).

### §28.7 Lineage, audit, idempotency and atomicity

- **Lineage columns** on `workflow_definitions`: `cloned_from_definition_id` (NULL, FK) and
  `clone_source_sha256 char(64)`, the source's `content_sha256` at clone time. They are set once and
  shown on the version list ("cloned from HRIS v1, AUR").
- **Audit.** `workflow_definition.cloned` in the target organisation's audit, with source ids,
  version, sha, the number of mapped and unresolved references. When the organisations differ, an
  additional row `workflow_definition.clone_source_read` is written in the **source** organisation's
  audit. Both are in the same transaction as the copy.
- **Atomic.** One transaction writes the draft, child rows, lineage and audit. Any failure rolls
  everything back; no half-built draft exists.
- **Idempotent.** The same `X-Idempotency-Token` returns the first response. A different token
  while the draft exists is 409 `DRAFT_EXISTS`. Cloning never changes the source.

### §28.8 Permissions and errors

| Caller | Same organisation | Another organisation |
|---|---|---|
| `full_admin` | allowed | allowed; every reference needs mapping (no defaults) |
| `org_admin` whose scope includes the target's organisation but not the source's | allowed | **404** (the source's existence is not revealed) |
| `org_admin` whose scope includes both organisations (D-38) | allowed | **403** `CLONE_CROSS_ORGANISATION` (the source is visible to them, so 404 would be false; OP-8) |
| Owners, vendors, agency staff, others | 404 | 404 |

The caller also needs edit rights on the **target** system (§24.1). A source that does not exist, is soft-deleted or is not visible is 404.

| Code | Status | When |
|---|---|---|
| `CLONE_SOURCE_NOT_PUBLISHED` | 409 | the source is a draft |
| `CLONE_SAME_SYSTEM` | 422 | the source belongs to the target system (use draft-from-version) |
| `CLONE_MAPPING_INVALID` | 422 | a mapping target is wrong (§28.4) |
| `CLONE_CROSS_ORGANISATION` | 403 | an `org_admin` whose scope includes both organisations clones across them (OP-8, D-38) |
| `DRAFT_EXISTS` | 409 | the target already has a live draft |

---

## §29 Test coverage mandate

### §29.0 Scope of this section

> This section states a **gate**, not a test design. Coverage is necessary for done, never sufficient.

### §29.1 Decision

| ID | Decision | Rationale |
|---|---|---|
| D-33 | **100% coverage gate.** Every Rust crate in the MVP workspace must reach 100% line, function and region coverage under `cargo-llvm-cov`, binaries included. The frontend must reach 100% lines, branches, functions and statements, per file, under Vitest with the v8 provider and jsdom. The only exclusions are generated files listed in `coverage-exclusions.md`. The gate cannot be bypassed by directives or by lowering a threshold | Requested by the user. Unexecuted code in a break-glass system is unverified code |

### §29.2 Rust gate

- Scope: **every crate** in the workspace at `sidequest/temporal-breakglass-mvp/` (part 8 §30):
  libraries, binaries and the test-support crate.
- Measured over the **whole** Rust suite combined (§16 layers 1–5 and 7): unit, workflow, real-DB
  activity, API integration and engine tests.
- Thresholds: `--fail-under-lines 100 --fail-under-functions 100 --fail-under-regions 100`.
- **Binaries.** `api` and `worker` run in the black-box harness (§16 layer 4) as instrumented binaries
  (`cargo llvm-cov show-env`) and stop by a signal that runs the covered shutdown path. `SIGKILL` writes
  no profile and fails the gate.
- **Branch coverage** is report-only: stable Rust 1.99.1 offers no branch gate.
- Unreachable code is **deleted** or made unrepresentable, not excluded. `fn main` stays thin (it
  calls `run`) and is measured.

### §29.3 Frontend gate

- Scope: every file under `web/src/**` via `coverage.include`, so a file with no test counts as 0%.
- Vitest with jsdom, React Testing Library, provider `v8`:

```ts
coverage: {
  provider: 'v8',
  include: ['src/**/*.{ts,tsx}'],
  exclude: [/* only entries from coverage-exclusions.md */],
  thresholds: { lines: 100, branches: 100, functions: 100, statements: 100, perFile: true },
}
```

- `perFile: true` is mandatory: an average must not hide one uncovered file.
- **Playwright runs are not counted**; they remain required by §16 for e2e videos.

### §29.4 Allowed exclusions

Only **generated** code, each entry in `sidequest/temporal-breakglass-mvp/coverage-exclusions.md`:

| Allowed | Condition |
|---|---|
| The generated OpenAPI client (`web/src/lib/api/generated/**`) | carries an `@generated` header; the file lists the generator command |
| Generated protobuf or `build.rs` output | same |

Each entry records the glob, generator, command and reason. A CI step regenerates the file and fails
on any diff, so a hand-edited file cannot hide under an exclusion.

### §29.5 Forbidden escape hatches

A grep gate (`make coverage-gate`) fails the build on any match outside `coverage-exclusions.md`:

| Pattern | Reason |
|---|---|
| `c8 ignore`, `istanbul ignore`, `v8 ignore` | frontend directives |
| `#[cfg(not(coverage))]`, `cfg(coverage`, `coverage(off)`, `coverage_nightly`, `LCOV_EXCL` | Rust directives |
| any `--ignore-filename-regex` or `exclude` entry not backed by §29.4 | hidden exclusion |
| a `fail-under` value or a `thresholds` value below 100 in the Makefile, CI files, `vitest.config.*` or `Cargo` config | lowering a threshold |
| production code moved into a `tests` path or a `*-test-support` crate to avoid measurement | relocation |

### §29.6 What coverage does not replace

Coverage **adds to** (replaces none of) the following (§16, part 3):
- Every AC maps to a named test in `docs/ac-traceability.md`.
- Real-database tests for activities (no mocks at the database boundary).
- The **bite checks** (AC-14.1.2): removing a guard must make a test fail. A line that executes
  without an assertion that depends on it is not accepted as covered in review.
- Replay tests and simulation-equals-execution tests.

### §29.7 Commands

Run by `make coverage-rust`, `make coverage-web` and `make coverage-gate` (both plus the grep gate), in CI on every pull request and locally before the Definition of Done:

```bash
cargo llvm-cov clean --workspace
source <(cargo llvm-cov show-env --export-prefix)
cargo build --workspace --bins
cargo test --workspace --all-targets          # includes the black-box harness
cargo llvm-cov report --fail-under-lines 100 --fail-under-functions 100 --fail-under-regions 100
pnpm --filter web exec vitest run --coverage   # thresholds come from vitest.config.ts
sh scripts/check-no-coverage-escapes.sh         # §29.5 grep gate
```

The Definition of Done (part 3 §16) gains: "coverage gate green". Stories: ST-29 (part 7).
