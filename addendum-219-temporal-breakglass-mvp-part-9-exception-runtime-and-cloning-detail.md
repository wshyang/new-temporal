# Addendum 219 (part 9) — Exception runtime, cloning and coverage gate in detail

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12).
- Part 2 `…-part-2-stories.md` (§14, ST-0–ST-14).
- Part 3 `…-part-3-verification-and-prompt.md` (§13, §15–§19).
- Part 4 `…-part-4-workflow-definition-and-editor.md` (§20–§26).
- Part 5 `…-part-5-editor-stories.md` (ST-15–ST-24).
- Part 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27–§29).
- Part 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34).
- **This part (§35–§37, ST-31).**
- Part 10 `…-part-10-exception-ui-and-walkthroughs.md` (§38–§40, ST-32). UI layout is specified there, not here.
- Appendix A `…-appendix-a-illustrative-example.md`.

**Why this part exists.** The user asked (2026-11-08) to "flesh out in detail more" three things that
parts 6–8 state but cannot hold at the 800-line cap:
- the unhappy paths after approval: an item missing and impossible to issue, lost after issuance,
  damaged after issuance, and their permutations (how each runs, end to end);
- cloning a workflow from an existing system's version;
- the coverage mandate.

**Precedence.** This part is a normative **elaboration** of part 6. It adds mechanism and closes gaps; it
does not change what part 6 decides. Where it resolves an ambiguity it says "refines part 6 §x", and the
refinement wins. Anywhere else part 6 wins. Part 8 conventions (RC-01…RC-19) apply to every sketch here.

### §35.0 Decisions and refinements added by this part

| ID | Decision | Rationale |
|---|---|---|
| D-34 | **An info hold does not pause the `investigate` SLA** (§35.6). The clock measures the team's time to resolution, and holding is the team's own decision | Keeps timers a pure function of node entry; a stalled hold is visible as "Overdue" and escalates |
| D-35 | **Return-due and return-overdue timers keep running while an exception is open** and flag against the *effective* state (§35.6) | The requester's return obligation does not pause because an investigation does |
| D-36 | **One transaction per settlement.** `apply_resolution` is a transaction function called inside `persist_lane_settlement`, never a separate commit (§35.2) | The instance chain and the physical effects can never disagree |
| D-37 | **Creating a system and cloning into it is atomic** (§36.4) | No half-created system in the wizard |

| # | Gap in part 6 | Resolution (refines part 6 §…) |
|---|---|---|
| G-1 | `route_exception` named as an activity candidate | Routing is the **pure** function; its persistence is part of `open_exception` (§27.8) |
| G-2 | Stage SLA on resume stated only for `recovered` | Stage SLA is cancelled when an exception opens and re-armed with its **full** duration on every resume (§27.5.2, §35.6) |
| G-3 | `resume` to `received_by_agency` leaves D-19 timestamps open | Sets `returned_at`/`return_verified_by` and writes `return_received` (§35.7, E-01) |
| G-4 | `expected_return_condition` could flip back to `good` | Monotonic: `good → damaged` only (§35.7) |
| G-5 | `lane.replacement_count` as chain length lets one slot exceed `max_replacements_per_item` (SP-01 replaced by SP-03, then SP-01 reported lost) | The count is **slot-wide**: lanes in the `replaces_lane_id` component minus one, via new column `request_plan_lanes.slot_root_lane_id` (§27.4, §27.7) |
| G-6 | `replaced{origin=supplementary}` would mark the original lane replaced | Separate milestone `supplementary_started`; `replaced` is only for `origin=replacement` (§27.8, §35.3) |
| G-7 | Second report on an open exception is "409" with no code | `EXCEPTION_ALREADY_OPEN` (§35.8) |
| G-8 | Substitute check at `act` cannot read the DB in a validator | The API attaches a `candidate_snapshot`; the activity re-checks in the transaction (§35.4) |
| G-9 | `state_before_exception` after resume | Cleared to NULL on resume; CHECK ties it to `exception_open`/`resolved_exception` (§35.7) |
| G-10 | Wizard cannot preview before the system exists | Preview route that takes the target facts inline; atomic create (§36.4) |
| G-11 | `content_sha256` canonical form unspecified | Canonical form contains no UUID (§36.1 step 12) |
| G-12 | Lane events unordered/undeduplicated across continue-as-new | `LaneEvent.seq`, parent FIFO queue (§35.3, §35.5) |

---

## §35 Exception runtime in detail

### §35.1 End-to-end sequences

**Notation.** `U` = Temporal Update; `V` = validator (pure, on workflow state); `W` = workflow code
between activities; `A#` = activity with its single transaction (`op_ref` = `{workflow_id}:{run_id}:{activity_id}`, written
to `workflow_ops` first); `S→P` = `lane_event` signal to the parent; `P` = parent. Every `A#` writes its
audit rows in the same transaction or rolls back (D-5, §2.6). Workflow IDs: lane `bg-AUR-R4-lane-L1`, parent
`bg-AUR-R4`. Every `act` passes the three D-5 layers (API 404/403, validator, activity re-check).

**(a) Missing before issuance → investigate → replace → join (PM-01; S10 step 1).**

```
Grace ─POST /api/cases/C1/exceptions {missing, high, desc, evidence[2]} T1─▶ api: authorize(stage team) ✓
U1 report_exception(updateId=T1, expectedRevision=v3) ─▶ lane-L1
V  not terminal ✓ · handler=None ✓ · lock free ✓ · rev ✓ → accept; handler:=pending; lock:=T1
W  facts{missing, high, pre_issuance, awaiting_preparation, slot_count 0} → route_exception → H1 (candidates [H1 ✓])
A1 open_exception: custody_exceptions(open, phase pre_issuance, no 1) · custody_cases(→exception_open, before
   awaiting_preparation, v4) · custody_events(exception_reported) · request_exception_routes · _route_candidates ·
   node_instances(exception_start done, investigate#1 active) · taken_edges(caught) · evidence attached
   · audit custody.exception_reported, custody.exception_routed
W  A2 compute_substitute_availability → [HT-02]; arm investigate SLA (48 h); lock:=None        ◀ 200 {exceptionId}
Frank ─POST /api/exceptions/E1/update {investigating, notes}─▶ U exception_update → A3 (exception_updates row,
   status investigating; audit custody.exception_updated)
Frank ─POST …/nodes/N5/act {replacement, fields{substituteAssetId:HT-02}, notes, expectedRevision:4} T3─▶ api:
   authorize(Ops Leads L2 ✓); live candidates [HT-02] → candidate_snapshot
V  N5 active · status investigating · replacement offered (policy ✓ · slot_count 0<1 ✓ · HT-02 ∈ snapshot ✓) · notes ✓
W  settle: leave investigate(replacement) → enter resolution_approval#1 (owner); effects RecordProposal, SetStatus
A4 persist_lane_settlement: custody_resolution_proposals(proposed, HT-02) · exception status
   awaiting_resolution_approval + update row · node_instances · taken_edges · cancel investigate SLA · audit custody.resolution_proposed
Carol ─POST …/nodes/N6/act {approve, expectedRevision:4} T4─▶ V: active ✓ · SoD: Carol ∉ {Bob, Grace, Frank} ✓
W  settle: approve → resolution(replace) → ApplyResolution{Replace{HT-02}}
A5 persist_lane_settlement ⊃ apply_resolution_tx (rows: §35.7 E-04): original case resolved_exception · item excluded `replaced`
   · asset retired `lost_replaced` · hold released · new item/hold/case/snapshots/lane L5 (origin replacement, depth 1,
   slot_root L1) · lane L1 excluded · proposal approved · exception resolved · event replacement_authorized
   · audit custody.resolution_approved, custody.exception_resolved                          ◀ 200 {status: resolved}
S→P lane_event{L1, seq 2, milestone Replaced{new_lane_id L5}}  (sent only after A5 committed)
P  queue → settle: L1 := Replaced(L5) and L5 := InProgress in ONE step; join live={L2,L3,L4,L5} → Wait
   persist_settlement(plan_revision+1, join_trace; audit workflow.lane_event_applied) → start child bg-AUR-R4-lane-L5
   (token branch: reset_pin task first). When L5 sends Issued and all others are crossed → join Issue → mark_issued.
```

**(b) Lost in custody → write-off → loss report → terminal; issuance stands (PM-10; S4, S10 step 4).**

```
Bob ─POST /api/cases/C3/exceptions {missing}─▶ U1 … V ✓ (in_requester_custody)   W route → H3? no: missing∉{damaged,identity_mismatch}
   → H2 (cands [H1 ✗ phase, H3 ✗ type, H2 ✓]); A1 as (a) with phase in_custody, reported_by_role submitter
Frank takes (A3), acts write_off ─▶ V ✓ ; W: leave investigate(write_off) → enter task loss_report (assignee submitter)
A4 persist_lane_settlement: node_instances(task active, due_at) · proposal NOT yet created (the drawn path puts the task first)
   · exception status stays `investigating`; Carol's approval instance is not yet active
Bob ─act {done, fields{loss_report_done:true}}─▶ V ✓ ; W: SetVariable(Call(lane)) → enter resolution_approval(owner)
A5 persist_lane_settlement: request_variables(L·loss_report_done, call scope) + request_variable_changes · proposal(proposed, write_off) ·
   status awaiting_resolution_approval · audit workflow.variable_set, custody.resolution_proposed
Carol approves ─▶ V ✓ ; A6 ⊃ apply_resolution_tx (E-09): case resolved_exception · asset retired `written_off` · hold released ·
   item NOT excluded · lane NOT excluded · event asset_written_off · audit custody.resolution_approved, custody.exception_resolved
S→P lane_event{L3, seq 4, Terminal{outcome WrittenOffAfterIssuance}}; P: lane terminal; status stays credential_issued;
   join already Issue; parent completes only when every lane is terminal and the request is closed.
```

A proposal row is created when the path reaches `resolution_approval` (not at the investigate port) if a
`task` sits between them. Refines part 6 §27.8 "Settlement" (which created it at the port).

**(c) Damaged in custody → H3 → team-approved supplementary replacement (PM-16; S10 step 5).**

```
Bob reports SP-01 damaged ─▶ route: H1 ✗ (phase) · H3 ✓ (in_custody ∧ type∈{damaged,identity_mismatch}) → A1 as (a)
Grace (Asset Store L1) takes it, acts replacement{SP-03} ─▶ V ✓ (policy ✓, slot_count 0<1, SP-03 ∈ snapshot) → proposal; approval
   instance = Ops Leads L2 team. Grace tries to approve → V: SoD (investigator) → 403 SOD_VIOLATION, nothing written.
Frank approves ─▶ V: SoD ✓ (not Bob/Grace) → A5 ⊃ apply_resolution_tx (E-06): original case → in_requester_custody
   (before := NULL, expected_return_condition := damaged), hold on SP-01 KEPT · new item/hold(SP-03)/case/snapshots · lane L6
   (origin supplementary, depth 1, slot_root L1) · original lane NOT excluded · event replacement_authorized ·
   audit custody.resolution_approved, custody.exception_resolved, custody.supplementary_lane_started
lane-L1 → P (L1 itself continues; this is its only event for this step):
   lane_event{L1, seq 5, SupplementaryStarted{new_lane_id L6}}
P  register L6 (live for `completed`, ignored by `issued`); persist_settlement; start child L6; status stays credential_issued.
Later: Claire verify_return(condition=damaged) with expected damaged → no exception (E-14). Grace return_to_inventory →
   asset quarantined, never restocked; outcome returned_damaged.
```

**(d) Verify-return finds unexpected damage → auto-opened return-phase exception (PM-23; S11 variant b).**

```
Claire ─POST /api/cases/C1/actions/verify_return {condition:damaged, notes}─▶ api: authorize(verify team ✓)
U1 custody_action(verify_return, damaged) ─▶ lane-L1
V  action legal in return_pending ✓ · no open exception ✓
W  expected_return_condition (state) = good ∧ condition = damaged → unexpected: facts{damaged, medium, return, received_by_agency,
   reported_by_role verifier} → route_exception → H4; builds OpenExceptionPlan{route} BEFORE the activity (determinism)
A1 apply_custody_action ⊃ open_exception_tx, ONE transaction (E-15): custody_cases(return_pending→received_by_agency→exception_open,
   return_condition=damaged, returned_at/return_verified_by set once, before received_by_agency, v+2) · events return_received,
   return_condition_recorded, exception_reported · exception(open, phase return, verifier) · route rows · node instances
   · audit custody.return_received, custody.return_condition_recorded, custody.exception_reported, custody.exception_routed
   guard: `expected_return_condition = 'good'` re-read in the transaction; if the DB says `damaged` → STALE_STATE (non-retryable),
   the workflow re-evaluates against fresh state (no exception) and retries once.
Claire acts quarantine (direct to resolution quarantine_return) ─▶ A2 ⊃ apply_resolution_tx (E-12): case resolved_exception ·
   asset quarantined_at/by/reason='damaged_on_return', hold released, restocked_* NULL · exception resolved(quarantine) ·
   events asset_quarantined · audit custody.exception_resolved, custody.asset_quarantined
S→P lane_event{L1, seq n, Terminal{ReturnedDamaged}}
```

**(e) Replacement disputed at handover → limit → only write-off → last live lane excluded → closed_unfulfilled (S11 variant a).**

```
HT-01 replaced by HT-02 as in (a): L1 excluded, L2 (replacement, depth 1, slot_root L1) live. L2 reaches handover_pending.
Bob ─POST /api/cases/C2/dispute {notes}─▶ U dispute_handoff ─▶ V: state handover_pending ✓ ; route H1; A1 as (a) (type handover_disputed)
Frank opens GET /api/exceptions/E2 → offeredPorts: replacement disabled REPLACEMENT_LIMIT (slot_count 1 ≥ max 1; reported
   before NO_SUBSTITUTE), write_off/recovered/false_report/waiver enabled. act replacement → V: 409 REPLACEMENT_LIMIT, no row.
Frank acts write_off ─▶ A4 proposal; Carol approves ─▶ A5 ⊃ apply_resolution_tx (E-08): L2 case resolved_exception · item
   excluded `written_off` · asset retired · lane L2 excluded · event asset_written_off
S→P lane_event{L2, seq 3, Excluded{WrittenOff}}
P  settle: live=∅ ∧ lanes≠∅ → AllExcluded → activity close_unfulfilled (status closed, decision_reason
   unfulfilled_all_items_written_off; audit request.closed_unfulfilled) → the per-item embed emits `unfulfilled` → workflow completes.
```

**Ordering guarantees (all sequences).** (1) A lane sends a signal only after its activity commits. (2) The parent applies
events one at a time from a FIFO queue, ordered per lane by `seq`. (3) A lane with an open `pre_issuance` exception is not
crossed, so the join cannot say `Issue` until the parent has applied that lane's `Replaced`/`Excluded`.

### §35.2 Activity catalogue additions

Common rules of part 1 §7.3 apply (one transaction, `op_ref`, audit-or-rollback, retry 1 s ×2 to 30 s). "Tx fn" is a function
called inside an activity's transaction; it is not a separate commit (D-36). Part 6's name `apply_resolution` means the tx fn.

| Name | Kind | Inputs | Writes | Guards | Audit | Non-retryable |
|---|---|---|---|---|---|---|
| `open_exception` (extended) | Activity | `ReportExceptionInput`, `RouteDecision`, phase | exception, case, event, route + candidate rows, `exception_start` and first `investigate` instance, taken edge, evidence | `custody_cases WHERE version=:v AND state=:from`; unique open exception per case; evidence `status='pending' AND uploaded_by=:actor` | `custody.exception_reported`, `custody.exception_routed` | `STALE_STATE`, `EXCEPTION_ALREADY_OPEN`, `NOT_AUTHORIZED`, `VALIDATION_FAILED` |
| `compute_substitute_availability` | Activity (read) | system, credential type, original asset | nothing; returns candidates (≤20, by hostname) and slot-wide count | live, `retired_at IS NULL`, `quarantined_at IS NULL`, no open hold, manual, same system and type, ≠ original | none | none (retryable) |
| `persist_lane_settlement` | Activity | `op_ref`, effects, case version, actor | node instances, taken edges, item variables and changes, rule evaluations, notifications, exception status and update rows, proposals; calls `apply_resolution_tx` | case `version`; instance `status='active'`; proposal `status='proposed'`; `authorize` and `separation_of_duties` over facts loaded in the transaction | `custody.exception_updated`, `custody.resolution_proposed`, `custody.resolution_rejected`, `workflow.variable_set`, `workflow.node_cancelled` | `NOT_AUTHORIZED`, `DELEGATION_INACTIVE`, `SOD_VIOLATION`, `STALE_STATE`, `NODE_NOT_ACTIVE` |
| `apply_resolution_tx` | Tx fn | `Resolution`, case, exception | per §35.7 | case `version`/`state`; item `excluded_at IS NULL`; asset `quarantined_at IS NULL`; hold insert hits `unique(asset_id) WHERE released_at IS NULL`; slot count re-read `<` max; substitute predicate re-run | `custody.resolution_approved`, `custody.exception_resolved` | `ASSET_IN_USE`, `SUBSTITUTE_INVALID`, `NO_SUBSTITUTE`, `REPLACEMENT_LIMIT`, `POST_ISSUANCE_REPLACEMENT_DISABLED`, `RESUME_STATE_INVALID` |
| `create_lane_tx` | Tx fn | substitute, origin, original lane | new item (`replaces_item_id`), hold, case (`awaiting_preparation`, v1), stage snapshots, `request_plan_lanes` row (`lane_no`=max+1, `origin`, `replaces_lane_id`, `replacement_depth`, `slot_root_lane_id`) | called only by `apply_resolution_tx` | `custody.supplementary_lane_started` (supplementary only) | as above |
| `quarantine_asset_tx` | Tx fn | asset, reason | `assets.quarantined_at/by/reason`, hold released, event `asset_quarantined` | `quarantined_at IS NULL`; set-once trigger | `custody.asset_quarantined` | `STALE_STATE` |
| `apply_custody_action` (extended: `record_return_condition`) | Activity | action, `condition`, optional `OpenExceptionPlan` | `return_condition` (set once), events, optionally `open_exception_tx`, and at `return_to_inventory` with `damaged` calls `quarantine_asset_tx` instead of restock | `return_condition IS NULL`; `expected_return_condition` re-read; version/state | `custody.return_condition_recorded` (+ the `open_exception` rows) | `STALE_STATE`, `INVALID_CUSTODY_STATE` |
| `flag_overdue` (extended) | Activity | node instance id | `request_node_instances.overdue_at`, event `sla_breached` | instance active ∧ `overdue_at IS NULL` | `custody.sla_breached` | none |
| `cancel_case` (extended) | Activity | case, reset node | case `cancelled`; handler instances `cancelled` (`cancelled_by_node_instance_id`); exception `resolved` with `cancelled_by_admin_reset`; event `exception_cancelled` | case not past `handover_pending`; **cases are re-read in the transaction, not taken from the parent's list** | `custody.case_cancelled`, `workflow.interrupt_fired` | `STALE_STATE` |
| `persist_settlement` (parent, lane events) | Activity | event, expected `plan_revision` | request variables, `plan_revision`+1, `join_trace` | `plan_revision=:expected`; `last_seq[lane] < event.seq` | `workflow.lane_event_applied` (new) | `STALE_STATE` |

`start_replacement_lane` and `start_supplementary_lane` are **not activities** (refines part 6 §27.8): the rows are written by
`create_lane_tx` inside the resolution transaction, and the parent starts the child (`bg-AUR-R4-lane-L5`, a deterministic ID) after it
applies the lane event. A child that already exists is a no-op.

### §35.3 Update and signal payloads

Rust-ish sketches in `mvp-contract`. Temporal payloads do not deny unknown fields; new fields are `#[serde(default)]` (RC-14).
The HTTP DTO is flat and has `deny_unknown_fields`; the API maps `fields` to `ActFields` using the **node kind of the instance**, so a field not valid for
that kind is 422 `VALIDATION_FAILED`.

```rust
pub struct ActorRef { pub user: UserId, pub acting_as: ActingAs }          // resolved by the API (D-4)
pub struct ReportExceptionInput {
    pub actor: ActorRef, pub exception_type: ExceptionType, pub severity: Severity,
    pub description: Description,                 // 1..=4000 chars, non-blank
    pub expected_location: Option<Location>,      // <= 200 chars
    pub last_known_location: Option<Location>,    // <= 200 chars
    pub evidence_ids: Vec<EvidenceId>,            // 0..=10, unique, pending, owned by actor
    pub expected_revision: CaseVersion,
}
pub struct ActInput {
    pub actor: ActorRef, pub node_instance_id: NodeInstanceId, pub outcome: PortKey,
    pub fields: ActFields, pub notes: Option<Notes>,            // Notes: 1..=2000, non-blank
    pub expected_revision: Revision,
    pub candidate_snapshot: Vec<AssetId>,         // API-supplied, <= 20; used only for early refusal (§35.4)
}
pub enum ActFields {
    Investigate { substitute_asset_id: Option<AssetId>, resume_state: Option<LiveState> },
    Approval,                                     // approve | reject carry notes only
    Task(Vec<FieldValue>),                        // typed per wf_node_fields (required_on_port_key honoured)
}
```

| Node kind | Outcome | `fields` | Notes |
|---|---|---|---|
| `investigate` | `replacement` | `substitute_asset_id` **required**, in the candidate set | required |
| `investigate` | `recovered` | `resume_state` required iff the reached `resume` has `investigator_choice`; must be in the phase's allowed set; forbidden otherwise | required |
| `investigate` | `false_report`, `write_off`, `waiver`, `quarantine` | none | required |
| `resolution_approval` | `approve` / `reject` | none | `reject` required, `approve` optional |
| `task` | `done` | typed values for the node's fields | optional |
| any handler instance | — | `override` is not accepted (no override inside handlers) | — |

`exception_update{to_status ∈ {investigating, awaiting_information}, notes (required)}` is the info-hold update (§35.5, part 6 §27.3).

```rust
pub struct LaneEvent { pub lane_id: LaneId, pub seq: u32, pub kind: LaneEventKind }   // seq: per lane, from 1
pub enum LaneEventKind {
    VariableDelta { var: VarId, add: Num },                  // additive only (part 11 D-47, V-53); Num: Int | Decimal
    Milestone(MilestoneEvent),
}
pub enum MilestoneEvent {
    Issued,                                  // lane crossed receipt (ignored by the issued predicate if origin=supplementary)
    Excluded { reason: ExclusionReason },    // WrittenOff | Cancelled, before receipt
    Replaced { new_lane_id: LaneId },        // origin=replacement only; excludes the sender and registers the successor atomically
    SupplementaryStarted { new_lane_id: LaneId },   // the sender continues; successor is live for `completed` only
    Terminal { outcome: LaneOutcome, outputs: Vec<(VarId, Value)> },   // out params, reduced at `completed` (part 11 §41.6)
}
```

The parent drops an event whose `seq` ≤ `last_seq[lane]` (duplicate) or whose lane is unknown to the current run (stale after
reset): no write, no audit. `last_seq` is part of the continue-as-new state. `variable_delta` is how supplementary lanes keep
`items_received` correct: each physical receipt counts, including a replacement's (refines part 4 §23.3 with `seq`).

### §35.4 Validator rules and in-transaction re-checks

**`act` on a handler instance, in order** (first failure wins; a failure writes nothing and appends no history event):

| # | Rule (validator, on `LaneWorkflow` state) | Code |
|---|---|---|
| 1 | `exception_id` is the lane's live `handler.exception_id` and the instance is `active` | `NODE_NOT_ACTIVE` 409 |
| 2 | decision lock free; `expected_revision` = case `version` | `STALE_STATE` 409 |
| 3 | exception status allows acting (`investigating`; for approval `awaiting_resolution_approval`); not `open`, not `awaiting_information` | `INVALID_EXCEPTION_STATUS` 409 |
| 4 | `outcome` is a port of the node | `OUTCOME_NOT_ALLOWED` 409 |
| 5 | port currently offered: `write_off` with `allow_write_off=false` → absent, `OUTCOME_NOT_ALLOWED`; `replacement` by §27.4 order: policy → `POST_ISSUANCE_REPLACEMENT_DISABLED`; slot count/`allow_replacement` → `REPLACEMENT_LIMIT`; empty snapshot → `NO_SUBSTITUTE` | as listed |
| 6 | approval kinds: `separation_of_duties(actor, {submitter, reporter, every investigate actor of this exception})` | `SOD_VIOLATION` 403 |
| 7 | notes rule of §35.3 | `VALIDATION_FAILED` 422 (`fields.notes`) |
| 8 | field rules: `substitute_asset_id` present and ∈ snapshot; `resume_state` ∈ allowed set; task fields typed | `SUBSTITUTE_INVALID`, `RESUME_STATE_INVALID`, `VALIDATION_FAILED` |

The API layer runs before: unknown or invisible → 404; not eligible → 403. `report_exception` validator: not terminal
(`INVALID_CUSTODY_STATE`), no open exception (`EXCEPTION_ALREADY_OPEN`), lock free and revision (`STALE_STATE`).

**Re-checks inside the activity transaction (D-5, layer 3).** Facts are loaded inside the transaction by the same `load_actor_facts`:

| TOCTOU case | What happens between the layers | Outcome |
|---|---|---|
| Substitute taken | Request R9 holds HT-02 after the offer or the proposal | Approval: the hold insert hits the unique index → **409 `ASSET_IN_USE`** (`fields.substituteAssetId`); nothing written, instance and proposal stay active/`proposed`. `GET /api/exceptions/:id` reports `proposal.substituteStatus: unavailable` and a refreshed `candidateSubstitutes`; Carol may `reject` with notes (back to a new `investigate` instance) and Frank proposes again |
| Substitute retired or quarantined | An admin retires HT-02 (or it is quarantined) | Predicate re-run fails → **422 `SUBSTITUTE_INVALID`**; same no-write rule |
| Original asset already retired | An admin retired the original concurrently | Not an error: the `UPDATE … WHERE retired_at IS NULL` affects 0 rows, the reason is kept, the audit row notes `asset_already_retired` |
| Approver loses team membership | Frank removed from Ops Leads L2 after the API check | `authorize` fails → **403 `NOT_AUTHORIZED`**, non-retryable; no row |
| Owner delegation expires | Emma's window ends mid-approval | **403 `DELEGATION_INACTIVE`**; no row |
| Self-approval appears late | A fixture grants the investigator owner-class rights after the API check | **403 `SOD_VIOLATION`** from the transaction |
| Limit reached late | Another lane of the same slot was approved first | Slot count re-read → **409 `REPLACEMENT_LIMIT`** |

`candidate_snapshot` is untrusted input, used only to refuse early without a database read in the validator; correctness is
always decided by the transaction (refines part 6 §27.4). Rule nodes and `substitute_available` in handlers use the value from
`compute_substitute_availability` at `investigate` entry, which is recorded in history and so replays deterministically.

### §35.5 Races and concurrency

| Situation | Outcome | Mechanism |
|---|---|---|
| Two reports on the **same lane** at once | Exactly one is accepted; the other is 409 `EXCEPTION_ALREADY_OPEN`, no row | The handler sets `handler=pending` synchronously before its first await, so the second validator sees it. Backstop: `unique(case_id) WHERE status open-ish` and case `version` |
| Reports on **different lanes** | Both accepted, independent handlers | Separate lane workflows. Shared resources are serialised elsewhere: assets by the hold index, request variables and join by the parent queue |
| Two `act`s on one instance | One succeeds, the other 409 `STALE_STATE` (in flight) or `NODE_NOT_ACTIVE` (done) | Decision lock; instance `status='active'` guard |
| `exception_update` (hold) racing `act` | Serial in the lane workflow; `act` after a hold → `INVALID_EXCEPTION_STATUS` | Update handlers run in arrival order on one history |
| Approval racing **admin reset (force)** | If `apply_resolution_tx` commits first, `apply_admin_reset` re-reads cases and also cancels the new `awaiting_preparation` replacement case; if reset commits first, the approval gets 409 `STALE_STATE` and the child is cancelled | Case `version` guard on both; `cancel_case` re-reads cases in its transaction; lane events from a stale run are dropped (§35.3) |
| Approval racing **auto-expire / tombstone auto-cancel** | Cannot happen: both exist only before owner approval; the deadline timer is cancelled at approval and `auto_expire` is guarded `WHERE status IN (pre-approval)` (0 rows if it ever fired) | Part 6 §27.8 |
| Approval racing **close** | Both succeed independently; the parent completes when closed ∧ every lane terminal, in either order | `close_request` writes request rows only; `apply_resolution_tx` writes case rows only |
| Approval racing a **tombstone of the submitter** | A `loss_report` completed by a tombstoned Bob is refused; the ordinal-1 team member acts `acting_as=submitter_team` | `authorize` over facts loaded in the transaction (403 `NOT_AUTHORIZED`) |
| **Replacement approval vs the last other lane's receipt** | The join cannot issue before the replacement lane is counted | The original lane stays live and uncrossed until the parent applies `Replaced{new}`, which excludes it and registers the successor in one `settle` step; events are processed one at a time in order (guarantee 2–3 of §35.1) |
| `mark_issued` in flight while a lane event arrives | The event waits in the queue | The queue is drained by the parent's main loop, which awaits `mark_issued` before the next event; handlers only enqueue |
| **Verify-return (damaged) vs a new report** | One outcome, never two exceptions. Report first: `verify_return` → `INVALID_CUSTODY_STATE`. Verify first (unexpected damage): the report → `EXCEPTION_ALREADY_OPEN` | Lane serialisation plus case `version` and the unique-open-exception index |
| **Two proposals, one substitute**, two lanes | The second approval 409 `ASSET_IN_USE`, no row | Hold index (§35.4) |
| Duplicate token or retried activity | First result returned; one write | Update ID = `X-Idempotency-Token`; `workflow_ops(op_ref)` |
| New definition published during an open exception | The exception keeps the pinned plan | `CompiledPlan` in history (R7) |

### §35.6 Timers in handlers

All durations are hours × `SLA_UNIT_SECONDS`, read once at worker start (demo 10 s: 8 h = 80 s, 24 h = 240 s, 48 h = 480 s). Timers
live in the lane child, so they survive parent continue-as-new. A firing is checked against the current instance before it acts.

| Timer | Armed | Cancelled / re-armed | On fire |
|---|---|---|---|
| `investigate` SLA | entering an `investigate` instance with `sla_hours`; **a new instance (after `reject`) gets a full new timer** | cancelled on leaving the node. **Not paused by an info hold (D-34)** | `flag_overdue(instance)`: `overdue_at`, event `sla_breached`, escalation team notified. **Escalation adds eligibility, it does not reassign** (part 5 AC-21.2.1): the `escalate_to_team` may now `act`; SoD still applies. No second level |
| `resolution_approval` SLA | entering the instance with `sla_hours` | cancelled on `approve`/`reject` | same; `escalate_to` must be a team |
| `task` due (for example `loss_report`) | entering with `due_in_hours` (V-40: 1..720) | cancelled on `done` | `flag_overdue`; shows on the assignee's dashboard; no escalation |
| `wait` in a handler | entering | cancelled on leaving | proceeds along `elapsed` |
| Lane stage SLA | entering a stage | **cancelled when an exception opens** (G-2); **re-armed with the full duration on every resume** (recovered, false report, waive, `quarantine_return` in custody) | `flag_overdue(stage)` if still in the stage |
| Return due / overdue | as part 1 §10 | **keep running while an exception is open (D-35)**; cancelled on `initiate_return`, and by any resolution that makes the case terminal | The firing reads the **effective state** (`state`, or `state_before_exception` when `exception_open`). `flag_return_due`/`flag_return_overdue` run only if it is `in_requester_custody`; otherwise no-op. A supplementary lane arms its own pair on entering `in_requester_custody`; a timer whose instant has passed fires immediately |
| Approval deadline, auto-expire | parent; not applicable after owner approval | cancelled at approval | — |

### §35.7 Effects in data

Row IDs `E-nn` are the cells of part 6 §27.5.2 plus the return-condition table of §27.5.3. Every row also: case `version`+1; the
event, D-19 timestamps, item/asset/lane columns and audit rows commit in one transaction (D-36); `custody_exceptions` gets
`resolved_by/at` and notes; `state_before_exception` is **NULL whenever the case is not in `exception_open`/`resolved_exception`**
(CHECK, G-9).

**E-table A: case, exception, proposal, ledger.** `exc` = `custody_exceptions.status/resolution`.

| Row | Cell | `custody_cases` (original) | `exc` and proposal | `custody_events` in order | Audit |
|---|---|---|---|---|---|
| E-01 | `resume` (recovered), any | `state` := chosen; if chosen = `received_by_agency` and `returned_at` NULL: `return_initiated_at`, `returned_at` := now, `return_verified_by` := actor (G-3) | resolved / recovered | `asset_recovered`; + `return_received` if `returned_at` newly set | `exception_resolved` |
| E-02 | `cancel_false_report`, any | `state` := `state_before_exception` | `cancelled_as_false_report` / false_report | `exception_cancelled` | `exception_resolved` |
| E-03 | `waive`, any | as E-02; `expected_return_condition` unchanged | resolved / waiver; proposal approved | `waiver_granted` | `resolution_approved`, `exception_resolved` |
| E-04 | `replace`, pre_issuance | `resolved_exception` | resolved / replacement; proposal approved with `substitute_asset_id` | `replacement_authorized` | as E-03 |
| E-05 | `replace`, in_custody, lost | `resolved_exception`; return timers cancelled | as E-04 | `replacement_authorized` | as E-03 + `supplementary_lane_started` |
| E-06 | `replace`, in_custody, damaged | `in_requester_custody`; `expected_return_condition` := `damaged` | as E-04 | `replacement_authorized` | as E-05 |
| E-07 | `replace`, in_custody, other | `in_requester_custody`; expected unchanged | as E-04 | `replacement_authorized` | as E-05 |
| E-08 | `write_off`, pre_issuance | `resolved_exception` | resolved / write_off; proposal approved | `asset_written_off` | as E-03 |
| E-09 | `write_off`, in_custody | `resolved_exception`; timers cancelled | as E-08 | `asset_written_off` | as E-03 |
| E-10 | `write_off`, return | `resolved_exception`; `returned_at` untouched | as E-08 | `asset_written_off` | as E-03 |
| E-11 | `quarantine_return`, in_custody | `in_requester_custody`; expected := `damaged` | resolved / quarantine | none | `exception_resolved` |
| E-12 | `quarantine_return`, return | `resolved_exception`; `returned_at`, `return_verified_by` := now/actor if NULL | resolved / quarantine | `return_received` (if new), `return_condition_recorded`, `asset_quarantined` | `exception_resolved`, `asset_quarantined` |
| E-13 | verify good vs expected good | `received_by_agency`; `return_condition` := good | none | `return_received` | `return_received` |
| E-14 | verify damaged vs expected damaged; later `return_to_inventory` | `received_by_agency`, then terminal `returned_to_inventory`; `restocked_*` NULL | none | `return_received`, `return_condition_recorded`; then `returned_to_inventory`, `asset_quarantined` | `return_condition_recorded`, `asset_quarantined` |
| E-15 | verify damaged vs expected good (unexpected) | `exception_open`, before `received_by_agency`; `return_condition` := damaged | new exception (open, phase return, `verifier`, medium) | `return_received`, `return_condition_recorded`, `exception_reported` | as sequence (d) |
| E-16 | verify good vs expected damaged | `received_by_agency`; `return_condition` := good; expected stays `damaged` (monotonic, G-4) | none | `return_received`, `return_condition_recorded`; restock follows | `return_condition_recorded` |

**E-table B: item, asset, lane, summary, history.** `Δ` is the change to `request_issuance_summary`.

| Row | `request_credential_items` | `assets` and `asset_holds` | `request_plan_lanes` | Summary Δ | History outcome |
|---|---|---|---|---|---|
| E-01…E-03 | none | none | none | none | unchanged |
| E-04 | original `excluded_at`, `excluded_reason='replaced'`; new item (`replaces_item_id`) | original: retired `lost_replaced` if lost/damaged, else hold released only; new hold on substitute | original `excluded_at`; new row `origin=replacement`, `replacement_depth`+1, `replaces_lane_id`, `slot_root_lane_id` | `items_total`+1, `items_replaced`+1 | original `replaced` |
| E-05 | original **not** excluded; new item | original retired `lost_replaced`, hold released; new hold | original untouched; new row `origin=supplementary` | `items_total`+1, `items_replaced_after_issuance`+1, `items_outstanding`−1 | original `replaced` |
| E-06 | original not excluded; new item | original keeps hold; new hold | new `supplementary` row | `items_total`+1, `items_replaced_after_issuance`+1 | `returned_damaged` after return |
| E-07 | as E-06 | as E-06 | as E-06 | as E-06 | `returned` (or `returned_damaged` if found damaged) |
| E-08 | excluded `written_off` | retired `written_off`, hold released | `excluded_at`, reason `written_off` | `items_written_off`+1 | `written_off` |
| E-09, E-10 | **not** excluded | retired `written_off`, hold released | none | `items_written_off`+1; E-09 also `items_outstanding`−1 | `written_off` |
| E-11 | none | none | none | none | pending |
| E-12 | none | `quarantined_at/by/reason='damaged_on_return'`, hold released, `restocked_*` NULL | none | `items_returned`+1, `items_returned_damaged`+1 | `returned_damaged` |
| E-13 | none | restocked at `return_to_inventory` | none | `items_returned`+1 | `returned` |
| E-14, E-15→E-12 | none | quarantined at `return_to_inventory` (E-14) | none | `items_returned`+1, `items_returned_damaged`+1 | `returned_damaged` |
| E-16 | none | restocked | none | `items_returned`+1 | `returned` |

`items_outstanding` = issued items with no `returned_at` whose history outcome is neither `written_off` nor `replaced`. The three views
(panel, CSV, `/assets/:id/history`) classify from these columns only (part 8 §30.9 classification test).

### §35.8 Error-code matrix

Layer: **API** (pre-check), **V** (workflow validator), **Tx** (activity transaction), **DB** (constraint or trigger). The test of
this table is AC-31.8.1; `fixtures/error-matrix.csv` must equal it. The table lists the codes parts 6–9 introduce or reuse; codes defined only in parts 1–5 (for example `ENGINE_UNAVAILABLE`, `IDEMPOTENCY_TOKEN_USED`, `STALE_DRAFT`) keep their definitions there, and the CSV holds the union of both, so the `ErrorCode` enum equals that union (AC-12.1.1). Every error leaves no row changed and no history event.

| Code | HTTP | Layer | Message | AC |
|---|---|---|---|---|
| `NO_SUBSTITUTE` | 422 | V, Tx | No free substitute asset of this credential type | AC-27.1.2 |
| `SUBSTITUTE_INVALID` | 422 | V, Tx | Substitute is not an available asset for this item | AC-31.4.2 |
| `RESUME_STATE_INVALID` | 422 | V, Tx | Resume state not allowed in this phase | AC-27.1.4 |
| `REPLACEMENT_LIMIT` | 409 | V, Tx | Replacement limit for this item reached | AC-27.5.1, AC-31.3.4 |
| `POST_ISSUANCE_REPLACEMENT_DISABLED` | 409 | V, Tx | Replacement after issuance is disabled | AC-27.5.3 |
| `INVALID_EXCEPTION_STATUS` | 409 | V | Exception is not in a status that allows this | AC-26.2.2 |
| `EXCEPTION_ALREADY_OPEN` (new) | 409 | V, DB | This case already has an open exception | AC-31.5.1 |
| `ASSET_QUARANTINED` | 409 | API, Tx | Asset is quarantined | AC-27.4.1 |
| `ASSET_IN_USE` | 409 | Tx, DB | Asset is held by another request | AC-31.4.1 |
| `SOD_VIOLATION` | 403 | V, Tx | You acted on this exception and cannot approve it | AC-27.8.1 |
| `NOT_AUTHORIZED`, `DELEGATION_INACTIVE` | 403 | API, Tx | Not eligible / delegation not active | AC-31.4.3, AC-31.4.4 |
| `NODE_NOT_ACTIVE`, `OUTCOME_NOT_ALLOWED`, `STALE_STATE` | 409 | V, Tx | as part 4 §24.2 | AC-26.2.3 |
| `INVALID_CUSTODY_STATE` | 409 | V, Tx | Action not legal in this custody state | AC-31.5.6 |
| `CUSTODY_IN_PROGRESS` | 409 | V | Reset needs `force` | AC-27.7.3 |
| `VALIDATION_FAILED` | 422 | API, V | field-level (`fields` paths) | AC-31.3.1 |
| `PLATFORM_NODE_LOCKED`, `PLATFORM_TERMINAL_LOCKED` | 422 | API, DB | Platform node cannot be changed | AC-25.7.2 |
| `DEFINITION_INVALID` | 409 | API | Draft has errors | AC-25.5.3 |
| `DRAFT_EXISTS` | 409 | API, DB | A draft already exists | AC-28.1.3 |
| `CLONE_SOURCE_NOT_PUBLISHED` | 409 | API | Source version is a draft | AC-28.3.4 |
| `CLONE_SAME_SYSTEM` | 422 | API | Use draft-from-version | AC-28.3.5 |
| `CLONE_CROSS_ORGANISATION` | 403 | API | Cross-organisation cloning is global-admin only, even for an `org_admin` of both organisations (D-38, OP-8) | AC-28.3.6 |
| `CLONE_MAPPING_INVALID` | 422 | API | Not a live team of the target organisation (same text for any invalid target) | AC-28.2.3, AC-31.9.10 |
| `RATE_LIMITED` (new) | 429 | API | Too many clone requests | AC-31.9.11 |
| `NOT_FOUND` | 404 | API | Not found (identical body for hidden and random IDs) | AC-28.3.2 |

`ASSIGNEE_UNRESOLVED` (V-42) and `HANDLER_SHADOWED` (V-32) are `issues[]` codes, not HTTP errors.

---

## §36 Cloning in detail

### §36.1 The algorithm: `definition_copy`

`mvp_db::definition_copy(tx, CopySource, &AssigneeMapping, DraftTarget) -> DefinitionId` (part 8 §30.9). The steps:

1. **Lock and gate.** `SELECT … FOR UPDATE` on the target `systems` row serialises concurrent drafts; a live draft → `DRAFT_EXISTS`
   (the partial unique index is the backstop). `DraftTarget::New(system_id)` is used inside system creation (§36.4).
2. **Load** every `wf_*` row of the source definition, including `wf_embed_bindings` and `wf_embed_pins` (immutable once published, so a plain read suffices). `wf_templates` rows are not copied unless the caller chose "also clone templates" (step 5a). Recompute the source's canonical
   sha (step 12) and compare with `content_sha256`; a mismatch aborts with `DbError::SourceCorrupt` (HTTP 500, audit
   `AUDIT_WRITE_FAILED` is not used).
3. **Build ID remaps first.** One `BTreeMap<OldId, NewId>` per table kind (`NodeId`, `VariableId`, `ExpressionId`, `HandlerId`, `PortKey`
   stays), minted by `mint()` from source rows sorted by `(node_key, ordinal)`. All remaps exist before any insert, so insertion
   order is free of mapping concerns.
4. **Skip platform nodes.** Rows with `is_platform_locked` and the `expired` terminal are not copied (re-seeded in step 11).
5. **Insert in FK order** (the draft row first, status `draft`, `version` NULL, `revision` 1, lineage set once):
   `workflow_definitions` → `wf_variables` → `wf_variable_enum_values` → `wf_expressions` (parents before children, by depth) →
   `wf_nodes` (embed nodes included, `handler_id` NULL) → `wf_exception_handlers` (`definition_id` of the copied template) →
   `wf_exception_handler_filters` → handler `wf_nodes` (`handler_id` now resolvable) → `wf_node_ports` → `wf_node_fields` →
   `wf_embed_bindings` → `wf_embed_pins` → `wf_allowed_credential_types` → `wf_edges` (last: they reference ports). The `wf_edges` trigger (both ends share `handler_id`)
   holds because every node exists.
5a. **Embeds and templates.** Same-organisation target: every embed keeps its `embed_template_id`, `embed_definition_id` and pins; bindings are
   re-pointed to the new node and variable IDs. Cross-organisation target: each embed becomes an unresolved placeholder (`embed_unresolved`,
   `embed_hint` "{template_key} v{n} (from {org})"; no pin; bindings kept by `param_name` with `param_variable_id` NULL), which V-42 reports
   as `EMBED_UNRESOLVED`. Choosing a template resolves the bindings by name (V-48 reports any that do not match). With "also clone templates", template drafts
   are created first by this same function (`DraftTarget::Template`, handlers and custody stages included, steps 3 to 14 recursively), and the
   root's embeds are then re-pointed to those drafts; the root stays unpublishable until the drafts are published and re-pinned.
6. **Expression trees.** Copied by depth with `parent_id` and `ordinal` preserved. Roots are re-pointed from `wf_nodes.condition_expression_id`,
   `set_expression_id` and `wf_node_ports.case_expression_id`. `variable_id` goes through the variable map; this includes
   declared `request` and `call` variables only. Platform keys (`item_count`, `item.*`, `exc.*`, `lane.*`, `substitute_available`) are
   `platform_var` values from the engine catalogue and are copied verbatim, never mapped (part 11 D-45).
7. **Ports and edges.** Ports keep `(node, port_key)`; edges keep `from_port_key`; only node IDs are remapped. Handler membership is
   preserved by the node copy, so no edge crosses a handler.
8. **Handlers and filters.** `handler_key`, `ordinal`, `is_fallback`, label and every filter row are copied; `definition_id` is remapped to the copied definition.
9. **Assignee slots.** For every node and each of two slots (`assignee`, `escalation`), where the kind is `team`: look up
   `(source cluster, level)` in the mapping. `Mapped::Team(t)` writes `cluster_id`, `level` of `t`. `Mapped::Unresolved` sets
   `assignee_unresolved` (or `escalate_unresolved`), the cluster NULL, the level kept as a picker hint, and `assignee_hint`/`escalate_hint`
   := "{cluster} L{level} (from {system} v{n}, {org})". `owner`, `submitter` and `any_reviewer` copy unchanged. This covers `approval`,
   `investigate`, `resolution_approval`, `custody_stage`, `task` and every `escalate_to_*`. The old `investigate_*` columns no longer
   exist (part 6 §27.7), so there is nothing else to remap. A reference's mapping applies to every use (one row per distinct pair).
10. **Positions** are copied verbatim (no offset: a draft is a separate canvas). **Per-item embed bounds and flags** are copied; `allowed_credential_types`
    too. Nothing in the target's assets is consulted at copy time.
11. **Re-seed** platform nodes from the current template via `seed_platform_nodes(tx, new_def)`, wiring by `node_key`.
12. **Canonical sha (G-11).** Computed at publish over rows ordered by `node_key`/`ordinal`, with **no UUID**: node references by
    `node_key`, team references by `(cluster name, level)`, labels and flags included, `change_notes`, `version` and positions excluded. A clone made
    with a name-identical mapping therefore publishes with the same `content_sha256` as its source; a changed label changes it.
13. **Validate** the copy with the pure engine (placeholders compile as `Unresolved`) and return `issues[]` including V-42 (one error
    per node and slot) and V-43 (warning for each allowed credential type with no live asset in the target system).
14. **Lineage and audit** (§36.4). Any failure rolls the transaction back.

### §36.2 Suggestion heuristics for the preview

This section refines part 6 §28.4 and §28.5: it adds the `same_team` and `cluster_level` bases and the `confidence` and `ambiguous` fields. Suggestions are advice. **The server never applies a suggestion**; only an `exact` default (same team, same organisation) is applied when
the request lists no mapping for it. The UI pre-selects the first suggestion for convenience, and the administrator confirms.

| Reference | Rank | Rule | `basis` | Confidence |
|---|---|---|---|---|
| Agency team | 1 | Same organisation and the same team id | `same_team` | `exact` (default applied) |
| Agency team | 2 | Target organisation has an agency team with identical team name and level (case-insensitive, whitespace collapsed) | `name_level` | `inferred` |
| Agency team | 3 | Target organisation has an agency cluster with the same name and a live team at the same level | `cluster_level` | `inferred` |
| Vendor team | 1 | Same organisation and its cluster is linked to the target (`system_vendor_clusters`) | `same_team` | `exact` (default applied) |
| Vendor team | 2 | The cluster at the same **ordinal** in the target's `system_vendor_clusters`, same level | `ordinal_level` | `inferred` |
| any | — | nothing matches | — | `none` |

Excluded from every rule: `pa` clusters, deleted teams, teams of another organisation, and the wrong cluster type. **Ties:** if a rank has more than one
candidate, all are listed ordered by team name then id with `ambiguous=true`, and the reference gets no default. The reference's overall
`confidence` is the best one available. Cross-organisation previews have no rank 1 and so no defaults at all (OP-8).

### §36.3 Request and response examples (IDs shortened)

Preview, `POST /api/admin/systems/S9/workflow/clone/preview`:

```json
{ "sourceDefinitionId": "d01" }
→ 200 { "data": {
  "source": { "definitionId": "d01", "systemName": "HRIS", "organisationCode": "AUR", "version": 1, "status": "published", "contentSha256": "9f2c…" },
  "crossOrganisation": false,
  "counts": { "nodes": 61, "edges": 70, "handlers": 5, "variables": 18, "references": 6 },
  "references": [
    { "sourceClusterId": "c-alpha", "sourceClusterName": "Alpha Pte", "sourceLevel": 2, "kind": "vendor",
      "usedBy": [ { "nodeKey": "alpha_l2", "slot": "assignee" } ], "confidence": "inferred", "defaultTarget": null,
      "suggestions": [ { "clusterId": "c-beta", "clusterName": "Beta Pte", "level": 2, "basis": "ordinal_level", "confidence": "inferred", "ambiguous": false } ] },
    { "sourceClusterId": "c-ops", "sourceClusterName": "AUR Ops", "sourceLevel": 2, "kind": "agency",
      "usedBy": [ { "nodeKey": "sec_review", "slot": "assignee" }, { "nodeKey": "h1_investigate", "slot": "assignee" } ],
      "confidence": "exact", "defaultTarget": { "clusterId": "c-ops", "level": 2 }, "suggestions": [] } ] } }
```

Clone as in S12 (part 10 §40.3): Alpha Leads L2 is mapped to Beta Leads L2; the escalation team Alpha Engineers L1 has no default and `mappings` omits it, so it stays unresolved; agency references take their same-team defaults:

```json
POST /api/admin/systems/S9/workflow/clone   X-Idempotency-Token: 01J…
{ "sourceDefinitionId": "d01", "mappings": [ { "sourceClusterId": "c-alpha", "sourceLevel": 2, "targetClusterId": "c-beta", "targetLevel": 2 } ] }
→ 201 { "data": { "definitionId": "d77", "status": "draft", "revision": 1, "clonedFromDefinitionId": "d01",
  "issues": [ { "code": "ASSIGNEE_UNRESOLVED", "severity": "error", "nodeKey": "alpha_l2", "slot": "escalation",
                "message": "Choose team (was Alpha Engineers L1, from HRIS v1, AUR)" },
              { "code": "V-43", "severity": "warning", "nodeKey": "items", "message": "No live asset of type 'password' in PAYROLL" } ] } }
```

Error, a mapping that targets a team of another organisation (or any non-live or wrong-type team; the message is identical):

```json
→ 422 { "error": { "code": "CLONE_MAPPING_INVALID", "message": "Not a live team of the target organisation",
        "fields": { "mappings[0].targetClusterId": "Not a live team of the target organisation" } } }
```

### §36.4 Lifecycle

- **Wizard (D-37, G-10).** The wizard collects the system fields and vendor clusters first. The preview step calls
  `POST /api/admin/systems/clone-preview {sourceDefinitionId, organisationId, vendorClusterIds[ordered]}`, which takes the target facts
  inline (the system does not yet exist) and runs the same pure mapping code. The final call is a **single**
  `POST /api/admin/systems {…system fields, workflow: {mode: "blank"|"clone"|"import", …}}`. The server creates the system row,
  its vendor-cluster links and the draft in **one transaction**, with `DraftTarget::New`. If the copy or the audit write fails, no system exists.
  `DRAFT_EXISTS` cannot occur for a new system.
- **Existing system without a draft:** `POST /api/admin/systems/:id/workflow/clone` creates the draft. **With a draft:** 409 `DRAFT_EXISTS`, nothing read from the source
  beyond the authorisation check.
- **Order of checks:** 404 (visibility) → 403 (target edit rights, then `CLONE_CROSS_ORGANISATION`) → 422 `CLONE_SAME_SYSTEM` → 409 `CLONE_SOURCE_NOT_PUBLISHED` →
  422 `CLONE_MAPPING_INVALID` → 409 `DRAFT_EXISTS` (inside the transaction, under the lock).
- **Retiring the source afterwards** changes only the source's status; the clone is independent and its lineage columns are untouched (set once).
- **Source later superseded** (HRIS v2 published): the clone still says "cloned from HRIS v1, AUR". Lineage records a definition id and a sha, not
  "latest". If the source system is later soft-deleted, the version list shows the lineage text with "(source deleted)".
- **Audit rows.** `workflow_definition.cloned` (target organisation): source definition, system, organisation, version, sha, counts of nodes, defaults
  applied, mapped and unresolved references, and the new definition id. `workflow_definition.clone_source_read` (source organisation, cross-organisation only):
  actor, source definition, target organisation. Preview writes none.

### §36.5 Security

- **IDOR.** Every source lookup is by `(id, caller-visible)`. A source that is hidden, deleted or unknown returns the same 404 body as a random UUID,
  before any source data is read. An `org_admin` asking for a source in an organisation outside their scope (D-38) gets that 404 and no `clone_source_read` row.
- **No cross-organisation leakage.** Preview names (`sourceClusterName`, `systemName`) are shown only to a caller who may see the source (a `full_admin`, or an `org_admin` whose scope
  includes the source's organisation). `suggestions` contain target-organisation teams only. `CLONE_MAPPING_INVALID` uses one message for "other organisation", "deleted",
  "wrong type" and "does not exist", so it is not an oracle.
- **`full_admin` cross-organisation clones** write both audit rows (§36.4) in the one transaction; failure of either rolls back.
- **Bounds.** Body ≤ 64 KiB; `mappings` ≤ 200 entries and no duplicate `(sourceClusterId, sourceLevel)`; ≤ 10 clones and ≤ 30 previews per minute
  per admin (429 `RATE_LIMITED`, nothing written). Idempotency per §28.7.

---

## §37 Coverage gate in detail

This section states how the gate runs, not what to test. **Tests are not stipulated** (part 6 §29.0): the implementing AI chooses them.

### §37.1 CI jobs

| Job | Commands | Needs | Artefacts |
|---|---|---|---|
| `rust-cov` | the block below | Postgres 16, Temporal dev server (service containers), pinned `cargo-llvm-cov`, `llvm-tools-preview` | `coverage/rust/{lcov.info, summary.json, html/}` |
| `web-cov` | `pnpm install --frozen-lockfile`; `pnpm --filter web exec vitest run --coverage` | Node LTS | `coverage/web/{lcov.info, coverage-summary.json, html/}` |
| `escape-gate` | `sh scripts/check-no-coverage-escapes.sh`; `make check-generated` | none | `coverage/escapes.txt` |
| `ratchet` | `sh scripts/coverage-ratchet.sh` | the three above | baseline diff |
| `coverage-gate` | aggregates the four exit codes; sets the PR status | all | none |

```bash
cargo llvm-cov clean --workspace
source <(cargo llvm-cov show-env --export-prefix)     # sets RUSTFLAGS, LLVM_PROFILE_FILE (…-%p-%m.profraw), CARGO_TARGET_DIR
cargo build --workspace --bins                         # instrumented api and worker, same env as the tests
cargo test --workspace --all-targets                   # the harness starts $CARGO_TARGET_DIR/debug/{api,worker}
cargo llvm-cov report --fail-under-lines 100 --fail-under-functions 100 --fail-under-regions 100
cargo llvm-cov report --lcov --output-path coverage/rust/lcov.info
cargo llvm-cov report --json --summary-only --output-path coverage/rust/summary.json
cargo llvm-cov report --html --output-dir coverage/rust/html
```

### §37.2 What contributes Rust coverage, and how profiles merge

- **In-process tests** (unit, engine, DB activity, workflow tests on the Temporal time-skipping environment, whose worker and workflows run inside the
  test binary) write one `.profraw` per process.
- **Black-box harness.** `Harness::start` resolves the binary from `$CARGO_TARGET_DIR`, spawns it with the **inherited** environment, so
  `%p` gives each child its own profile. `Harness::stop` sends `SIGTERM` and waits up to 10 s for the graceful-shutdown path (which flushes the profile).
  On timeout it sends `SIGKILL` and **fails the test** `HARNESS_SHUTDOWN_TIMEOUT`; a dropped harness that was never stopped panics.
- The Temporal dev server and Postgres are external and not instrumented.
- **Merging.** `cargo llvm-cov report` merges every `.profraw` in the target directory with `llvm-profdata` and exports once. Split test invocations are
  allowed only under the same `show-env` and without `clean` between them. A profile from a binary built with different flags is rejected.
  The report must show every crate of part 8 §30.2, including `mvp-test-support` and the two binaries' `main.rs`.

### §37.3 Frontend configuration

```ts
// web/vitest.config.ts (excerpt)
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      include: ['src/**/*.{ts,tsx}'],          // an untested file counts as 0%
      exclude: ['src/lib/api/generated/**'],   // only entries backed by coverage-exclusions.md
      reporter: ['text-summary', 'lcov', 'html', 'json-summary'],
      reportsDirectory: '../coverage/web',
      thresholds: { lines: 100, branches: 100, functions: 100, statements: 100, perFile: true },
    },
  },
});
```

### §37.4 `coverage-exclusions.md` and the grep gate

`coverage-exclusions.md` holds one section per entry, with exactly these keys (the gate parses them):

```
## EX-01
- glob: web/src/lib/api/generated/**
- generator: openapi-typescript
- command: pnpm --filter web run gen:api
- header: @generated
- reason: client generated from openapi.json; regenerated and diffed in CI
```

`check-no-coverage-escapes.sh` uses `rg -n` over tracked files, excluding `coverage-exclusions.md` and the script itself:

| Rule id | Pattern |
|---|---|
| `ESC-DIRECTIVE-WEB` | `(c8|istanbul|v8) ignore` |
| `ESC-DIRECTIVE-RUST` | `cfg\(not\(coverage\)\)\|cfg\(coverage\|coverage\(off\)\|coverage_nightly\|LCOV_EXCL` |
| `ESC-IGNORE-REGEX` | `--ignore-filename-regex` or a vitest `exclude` entry with no matching `EX-nn` glob |
| `ESC-THRESHOLD` | any `fail-under*` or `thresholds` value below 100 in the Makefile, CI files, `vitest.config.*`, `.cargo/config.toml` |
| `ESC-RELOCATION` | non-test code under a `tests/` path or in a `*-test-support` crate that is imported by a production crate |
| `ESC-GENERATED-DRIFT` | `make check-generated` shows a diff, or an excluded file lacks its `header` |

**Failure output.** One line per finding: `COVERAGE-GATE FAIL <rule-id> <path>:<line> <detail>`, for example
`COVERAGE-GATE FAIL RUST-REGION crates/mvp-custody/src/resolve.rs:212 region not covered (fn describe_effects)`. Exit codes: 10 Rust
threshold, 11 web threshold, 12 escape gate, 13 ratchet, 14 harness shutdown timeout. `coverage-gate` exits with the first non-zero
and prints all findings.

### §37.5 Ratchet and reporting

- **Never lower.** `coverage-baseline.txt` (committed) records: Rust branch coverage %, the web branch %, the number of `EX-nn` entries, and
  the thresholds (all 100). `coverage-ratchet.sh` fails if branch % falls below the baseline or `EX-nn` count rises. Raising the baseline is a
  normal commit; `make coverage-ratchet-update` refuses to lower any value. A new exclusion needs a generated-code justification (§29.4) in the same change.
- **Reporting.** The implementing AI's final report names the highest layer of part 3 §16 (1–9) that is green and every layer not run, with the reason; and
  prints one line: `coverage: rust L/F/R 100/100/100 (branch 97.4%), web L/B/F/S 100/100/100/100 perFile, escapes 0, exclusions 2`. A gate that was not run is
  reported as **not run**, never as passing. Coverage is necessary for done and never sufficient (§29.6).

---

## ST-31 Exception runtime, cloning algorithm and coverage mechanics

**Conventions** as part 7: every negative AC also asserts no side effect (no row changed, no workflow history event, no audit row, no
notification). Personas and assets as in part 7. "S10", "S11" are part 6 §27.12.1–2.

**US-31.1** As the platform, I run each unhappy path end to end as in §35.1.
- AC-31.1.1 (sequence a; PM-01)
  - G: S10 step 1 on HRIS v1.
  - T: the rows written by `open_exception` (A1), `persist_lane_settlement` (A4, A5) and the audit actions are exactly those of §35.1(a) and E-04; every
    activity row in `workflow_ops` has the `op_ref` form `bg-AUR-R4-lane-L1:run1:A#`; the signal `Replaced{L5}` has `seq`=2 and is applied once by the parent; `join_trace`
    reads `live=4 … → Wait` and later `→ Issue`.
- AC-31.1.2 (sequence b; PM-10): LT-01 lost in custody routes to H2 with candidates `[H1 ✗, H3 ✗, H2 ✓]`; Carol's approval instance does not exist until Bob completes
  `loss_report`; the proposal row is created on reaching the approval; E-09 rows result; the lane is not excluded and the request stays `credential_issued`.
- AC-31.1.3 (sequence c; PM-16): Grace's approval attempt is 403 `SOD_VIOLATION` with no row; Frank's approval writes E-06 and the audit row `custody.supplementary_lane_started`;
  lane L6 is `supplementary`, depth 1, `slot_root_lane_id`=L1; `lane-L1` sends `SupplementaryStarted{L6}`; the `issued` predicate is unchanged.
- AC-31.1.4 (sequence d; PM-23): unexpected damaged verify opens the exception in the **same transaction** (E-15). With a fault injected at the audit write, the case stays
  `return_pending` and no exception, route or event exists. If a concurrent resolution had set `expected_return_condition=damaged`, the activity raises `STALE_STATE`,
  no exception opens, and the retry records E-14.
- AC-31.1.5 (sequence e; S11 a): the offer lists `replacement` disabled with `REPLACEMENT_LIMIT`; after the write-off the parent's `settle` returns `AllExcluded`, `close_unfulfilled` runs
  once, and the workflow completes at `closed_unfulfilled`; the summary matches S11 variant a.
- AC-31.1.6 (crash safety): with `CRASH_AFTER_COMMIT=persist_lane_settlement` after A5 of sequence (a), the worker restarts, the activity returns the stored result, the signal is sent
  once, and the parent applies `Replaced` once (one `workflow.lane_event_applied` row). No duplicate lane, item or hold exists.

**US-31.2** As the platform, I make the new activities safe to retry and refuse bad writes.
- AC-31.2.1: re-running `persist_lane_settlement` with a stored `op_ref` returns the stored result and writes nothing. Re-running with a **different** `op_ref` after the
  case `version` moved fails with `STALE_STATE` and writes nothing.
- AC-31.2.2: `compute_substitute_availability` writes no row (table row counts unchanged) and excludes: held, retired, quarantined, another credential type, another system, the original asset and non-manual
  assets; the list is capped at 20 and ordered by hostname.
- AC-31.2.3: a fault injected at any one write inside `apply_resolution_tx` (item, asset, hold, lane, case, event, audit) leaves **none** of the E-table rows; the instance chain shows no completed
  `resolution` node.

**US-31.3** As the platform, I validate payloads and apply the rules in the stated order.
- AC-31.3.1 (bounds): for `ReportExceptionInput` and `ActInput`: description 0 / 1 / 4000 / 4001 characters, notes 0 / 1 / 2000 / 2001 / whitespace-only, `evidence_ids` 0 / 10 / 11 / duplicate, locations 200 / 201,
  `candidate_snapshot` 20 / 21, an unknown field, and a field not valid for the node kind (`substitute_asset_id` on `approve`) → 422 `VALIDATION_FAILED` with the `fields` path; the boundary-valid cases succeed.
- AC-31.3.2 (precedence): a table test sends an `act` that violates several rules at once and asserts the code of the **first** failing rule of §35.4 (for example inactive instance + stale revision →
  `NODE_NOT_ACTIVE`; disabled replacement + missing notes → the replacement code; approval by an investigator with no notes → `SOD_VIOLATION`).
- AC-31.3.3 (offer reasons): with `allow_post_issuance_replacement=false`, slot count at the limit and no substitute all true together, the reason is `POST_ISSUANCE_REPLACEMENT_DISABLED`; with the first false, `REPLACEMENT_LIMIT`;
  with the first two false, `NO_SUBSTITUTE`.
- AC-31.3.4 (slot-wide limit, G-5)
  - G: SP-01 replaced by SP-03 (supplementary), limit 1.
  - W: Bob reports SP-01 `missing`.
  - T: the offer shows `replacement` disabled with `REPLACEMENT_LIMIT` (the slot count is 1, though SP-01's own chain position is 0); an attempt is 409 with no row.
- AC-31.3.5 (golden payloads): golden JSON for `LaneEvent` (each `MilestoneEvent`), `ActInput` and `ReportExceptionInput` is deserialised by current code; a payload without `seq` is rejected by the parent
  as malformed (no write); a payload with an extra field is accepted.

**US-31.4** As an auditor, I want every TOCTOU window closed in the transaction.
- AC-31.4.1 (substitute taken)
  - G: the proposal names HT-02; request R9 then holds HT-02.
  - W: Carol approves.
  - T: 409 `ASSET_IN_USE` with `fields.substituteAssetId`; no case, item, hold, lane or event row changes; the instance stays active; `GET /api/exceptions/:id` shows
    `substituteStatus: unavailable` and a refreshed candidate list; Carol rejects with notes and a new `investigate` instance exists; Frank proposes another asset and the approval then succeeds.
- AC-31.4.2 (substitute retired or quarantined): the same flow where an admin retires HT-02, and a second run where HT-02 is quarantined → 422 `SUBSTITUTE_INVALID`, no write. The original
  asset already retired concurrently → success, the retirement reason is unchanged, and the audit row carries `asset_already_retired`.
- AC-31.4.3 (membership lost): Frank is removed from Ops Leads L2 after the API check and before the activity → 403 `NOT_AUTHORIZED`, no write, the instance stays active, Carol-class approvers still can act.
- AC-31.4.4 (delegation expires): Emma's delegate window ends between the API check and the activity → 403 `DELEGATION_INACTIVE`, no write. A fixture granting the investigator owner-class rights
  after the API check → 403 `SOD_VIOLATION` from the transaction.

**US-31.5** As the platform, I resolve each race to one outcome.
- AC-31.5.1: two simultaneous reports on HT-01 → one 200 and one 409 `EXCEPTION_ALREADY_OPEN`, one exception row; with the workflow check bypassed in a fixture, the unique index refuses the second. Simultaneous
  reports on HT-01 and LT-01 are both accepted and routed independently.
- AC-31.5.2: two simultaneous `act`s on one instance → one success, one 409 (`STALE_STATE` or `NODE_NOT_ACTIVE`); an `exception_update` to `awaiting_information` racing an `act` leaves either "act
  done, hold refused" or "hold applied, act `INVALID_EXCEPTION_STATUS`", never both.
- AC-31.5.3 (approval vs force reset, both orders)
  - G: LT-01 has a `pre_issuance` exception awaiting Carol; a reset with `force` runs concurrently.
  - T: if the approval commits first, the replacement case is `cancelled` by the reset's transaction and no case is left `awaiting_preparation`; if the reset commits first, the approval is 409
    `STALE_STATE` with no write. In both orders every handler instance is terminal and the stale `Replaced` signal is dropped with no write.
- AC-31.5.4: an approval racing `close` leaves both applied and the workflow completing once; auto-expire and tombstone auto-cancel after approval change nothing (`auto_expire` guarded, 0 rows).
- AC-31.5.5 (replacement vs last receipt)
  - G: three lanes crossed; HT-01's lane has an approved replacement whose `Replaced` event is queued while the parent processes the last lane's `Issued`.
  - T: the parent never calls `mark_issued` until L5 is registered and crossed; at no time does `join_trace` read `→ Issue` with L5 absent. Repeated 200 times with randomised delays, no run issues early.
- AC-31.5.6: verify-return (damaged) racing a report on the same case → exactly one exception; report-first makes the verify 409 `INVALID_CUSTODY_STATE`; verify-first makes the report 409 `EXCEPTION_ALREADY_OPEN`.
- AC-31.5.7: duplicate and out-of-order `LaneEvent`s (`seq` 3 before 2, `seq` 2 twice, an event for a lane of a previous run) are applied in `seq` order exactly once; the duplicate and stale events write nothing.

**US-31.6** As an operator, I want handler timers to behave as §35.6 says.
- AC-31.6.1 (hold does not pause; escalation adds eligibility)
  - G: H3 `investigate` 8 h, demo unit 10 s, escalation to Ops Leads L2; Grace puts the exception on `awaiting_information` at 20 s.
  - T: at 80 s `overdue_at` is set (±2 s) despite the hold; Frank (escalation team) can now `act` (before 80 s he gets 403); Grace remains eligible; SoD still binds Frank if he had acted earlier.
- AC-31.6.2: a `resolution_approval` with `sla_hours` and a `loss_report` task with `due_in_hours` each set `overdue_at` and an `sla_breached` event on time; the task has no escalation; both timers are cancelled on
  leaving the node (a later `act` leaves no stray timer). A `reject` creates a new `investigate` instance with a full new SLA.
- AC-31.6.3 (stage SLA, G-2): opening an exception at `awaiting_preparation` cancels the stage timer; `false_report`, `waive` and `recovered` each re-arm the **full** stage duration (assert `stage_due_at`).
- AC-31.6.4 (return timers, D-35)
  - G: `in_requester_custody`, return due at +30 s, an `in_custody` exception open.
  - T: the timer fires and `flag_return_due` runs (effective state is `in_requester_custody`); after `write_off` (E-09) no later return timer fires for that case. For a supplementary lane entered after
    `activity_end_at`, `flag_return_due` fires immediately.

**US-31.7** As an auditor, I want the exact rows of every resolution cell.
- AC-31.7.1 (E-01…E-03): for each of `resume`, `cancel_false_report` and `waive` in each phase the asset, item and lane columns are unchanged, `state_before_exception` is NULL after resume, and the events match the E-table. `resume`
  to `received_by_agency` from `in_custody` sets `returned_at`, `return_verified_by` and writes `return_received` (G-3); the same resume when `returned_at` is already set does not overwrite it.
- AC-31.7.2 (E-04, E-08): the S10 step 1 and step 2 rows equal the E-tables exactly, including `excluded_reason`, `retired_reason`, hold release, new lane columns and the summary deltas.
- AC-31.7.3 (E-05…E-07): for lost, damaged and other, assert original and new rows column by column; E-06 leaves `expected_return_condition=damaged` and the hold on the original; E-07 leaves it `good`; `items_outstanding`
  decreases only in E-05.
- AC-31.7.4 (E-09, E-10, E-11, E-12): item and lane not excluded for write-offs; `restocked_*` NULL and `quarantined_*` set for E-12; E-11 sets only `expected_return_condition`.
- AC-31.7.5 (E-13…E-16): the four condition combinations produce the listed `return_condition`, events and asset outcomes; `expected_return_condition` never moves from `damaged` to `good`; `return_condition` cannot be
  written twice (trigger).
- AC-31.7.6 (meta): `fixtures/effects-matrix.csv` equals the E-tables A and B; for every row the panel, the CSV export and `/assets/:id/history` give the same per-item outcome; deleting one `db_eff_nn` test or one CSV row
  fails the meta-test.

**US-31.8** As the platform, I keep one error vocabulary.
- AC-31.8.1 (completeness): a test enumerates `ErrorCode` and the table of §35.8; every code has the HTTP status and layer listed, a reproduction in a named test, and an entry in `fixtures/error-matrix.csv`; a code missing from
  the CSV or from the enum fails. Each reproduction asserts no row changed and no history event.
- AC-31.8.2: the same refusal raised at two layers returns the same body: `REPLACEMENT_LIMIT` from the validator and from the transaction (forced by bypassing the validator) have identical status, code and message.

**US-31.9** As an administrator, I want a faithful, safe and leak-free clone.
- AC-31.9.1 (FK order and atomicity): `definition_copy` inserts in the order of §36.1 step 5; with a fault injected at each of the 12 tables in turn, nothing is left (no definition, node, edge, handler, lineage or audit row).
  Source rows are unchanged.
- AC-31.9.2 (expression trees): for a source with nested AND/OR comparisons, arithmetic and an `exc.*` comparison inside a handler `rule`, the clone's trees are isomorphic (same shape, `ordinal`, operators, literals),
  every `variable_id` resolves in the clone's `wf_variables`, and no row of the clone references a source id.
- AC-31.9.3 (remap): ports, edges, handlers (`handler_key`, `ordinal`, filters), `escalate_to_*` and `allowed_credential_types` are copied; a mapped escalation target is remapped the same as an assignee; positions are
  identical; locked nodes are re-seeded, not copied (changing the template changes the next clone).
- AC-31.9.4 (sha): cloning HRIS v1 into PAYROLL with a name-identical mapping and publishing gives the same `content_sha256` as the source; changing one label gives a different one; `clone_source_sha256` equals the
  source's at clone time; both lineage columns reject an `UPDATE`.
- AC-31.9.5 (placeholders): leaving two distinct references unresolved yields one mapping row each applied to all uses, `assignee_unresolved` / `escalate_unresolved` set, cluster NULL, level kept, hint text as in the example;
  `simulate` and `publish` return 409 `DEFINITION_INVALID`; choosing a team clears the flag; a crafted `UPDATE` publishing a placeholder raises from the trigger.
- AC-31.9.6 (heuristics): with fixtures for each rank of §36.2, the preview returns the stated `basis` and `confidence`; same-organisation agency and linked-vendor references get `exact` defaults; cross-organisation previews
  have none; `pa` clusters and deleted teams never appear; nothing is applied server-side without a mapping entry.
- AC-31.9.7 (ties): two target teams that tie at rank 2 are both listed with `ambiguous=true`, ordered by name then id, and the reference has no default.
- AC-31.9.8 (wizard atomicity): `POST /api/admin/systems` with `workflow.mode=clone` and a fault at the copy, the lineage or the audit write returns an error and **no `systems` row, vendor-cluster link or definition exists**;
  success creates all in one transaction; `clone-preview` before creation writes nothing.
- AC-31.9.9 (lifecycle): retiring the source after cloning changes only the source's status; publishing HRIS v2 does not change the clone's lineage text; a second clone into a system with a draft is 409 `DRAFT_EXISTS` with
  no source read beyond authorisation; the check order of §36.4 is asserted with inputs that violate several rules.
- AC-31.9.10 (leakage and audit): Amanda (`org_admin` @AUR) requesting a SEN source and Gina requesting an AUR source both get the 404 body of a random UUID and no audit row; a `CLONE_MAPPING_INVALID` for a SEN team,
  for a deleted AUR team and for a random UUID have identical bodies; Greg's cross-organisation clone writes `workflow_definition.cloned` in the target organisation and `workflow_definition.clone_source_read` in the source
  organisation, and a fault at the second rolls back the first.
- AC-31.9.11 (bounds): a body over 64 KiB, 201 mappings or a duplicate mapping pair → 422 `VALIDATION_FAILED`; the 11th clone or 31st preview in a minute by one admin → 429 `RATE_LIMITED`; nothing written in any case.
- AC-31.9.12 (V-43): cloning a per-item embed that allows `password` into a system with no live `password` asset returns the V-43 warning in `issues[]`, and publish is not blocked.

**US-31.10** As the team, I want the coverage mechanics to be provably real.
- AC-31.10.1 (binaries count): with the harness tests removed, `api`/`worker` lines drop below 100% and the gate fails naming the files; with them present the profiles of both binaries (one `.profraw` per process, distinct `%p`)
  are merged into the single report.
- AC-31.10.2 (shutdown): a deliberate run that stops the harness with `SIGKILL` fails with `HARNESS_SHUTDOWN_TIMEOUT` and exit code 14; a normal `SIGTERM` stop flushes the profile and passes.
- AC-31.10.3 (web config): `vitest.config.ts` has `provider: 'v8'`, `include: ['src/**/*.{ts,tsx}']`, all four thresholds 100 and `perFile: true`; a new untested file and a file at 99% each fail with exit code 11 and a line
  `COVERAGE-GATE FAIL WEB-PERFILE <path> …`.
- AC-31.10.4 (exclusions): hand-editing a generated client file (header intact) fails `ESC-GENERATED-DRIFT`; an excluded path with no `EX-nn` entry fails `ESC-IGNORE-REGEX`; a file in `coverage-exclusions.md` lacking `generator` or
  `command` fails the parse.
- AC-31.10.5 (escape fixtures): one fixture per rule id of §37.4 makes `check-no-coverage-escapes.sh` exit 12 with that rule id and `path:line`; removing the fixture turns the gate green.
- AC-31.10.6 (ratchet): lowering a value in `coverage-baseline.txt`, adding an `EX-nn` entry or dropping Rust branch coverage below the baseline exits 13; raising the baseline passes; `make coverage-ratchet-update` refuses to lower.
- AC-31.10.7 (reporting): the AI's final report contains the one-line `coverage:` summary, names the highest green layer of part 3 §16 and lists each layer not run; a report that claims the gate passed without CI exit code 0
  is a Definition of Done failure (checked by the traceability review, `docs/ac-traceability.md`).
