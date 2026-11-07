# Addendum 219 — Temporal break-glass MVP (part 1): specification, decisions, architecture and data model

**Date:** 2026-11-07
**Status:** PROPOSED. This is the specification only; no implementation exists.
**Scope:** a standalone demonstration app under `sidequest/temporal-breakglass-mvp/`. It is not part
of the production build, and it does not replace the inline transition table in
`crates/ffs-server/src/workflow_machine.rs` or the XState table in `server/workflow-machine.tsx`.
**Supersedes:** nothing. It extends the user's draft prompt "Advanced Implementation Guide: End-to-End Workflows
with Variables, Fork/Join, and Rule Evaluation"; the real break-glass process replaces its procurement example.
**Excluded:** every AWS path (credential exchange, secret delivery, tenant sweep), Slack, Entra, email delivery,
export/import and TOTP. In this MVP every credential item is a **manual (physical-custody)** item.

---

## §0 Document map

| § | Content |
|---|---|
| §1 | Context and goal |
| §2 | **How the app relates to Temporal, where data (remarks, decisions, evidence) is stored, and long-term retention (§2.7)** |
| §3 | Ground truth: what the current app does, the gaps, and the AWS exclusions |
| §4 | MVP decisions D-1 … D-19 and D-38 (D-20 … D-27 and D-39 … D-43 in part 4 §20.4; D-28 … D-33 in part 6; D-34 … D-37 in part 9; D-44 … D-48 in part 11; D-49 … D-61 in part 12) |
| §5 | Architecture, processes, sequence diagrams |
| §6 | Domain model, as an MVP DDL outline |
| §7 | Workflow specification: parent and child workflows, updates, signals, queries, timers, activities |
| §8 | Authorization matrix |
| §9 | Custody state machine and exception-resolution matrix |
| §10 | Timers and SLAs |
| §11 | API contract |
| §12 | UI specification (Next.js and React Flow) |
| §13 (part 3) | Seed data and demo personas |
| §14 (part 2) | Subtasks ST-0 … ST-14: user stories and acceptance criteria |
| §15 (part 3) | Demo scenarios S1 … S13 |
| §16 (part 3) | Verification, test matrix, Definition of Done |
| §17 (part 3) | Out of scope / Future |
| §18 (part 3) | Open points |
| §19 (part 3) | **The implementation prompt**, a fleshed-out replacement for the draft prompt |
| §20–§26 (part 4) | **Per-system workflow definition vs the actualised request plan** (R1–R10), node and port catalogue (including the `embed` node and **embeddable organisation workflow templates**, D-39 … D-43), connector rules, validation catalogue V-01…V-52, definition and plan DDL, the generic interpreter, the **graphical workflow editor**, simulation, the seeded HRIS v1 demo workflow and feature coverage, and the supersession table (§26) |
| §14 cont. (part 5) | Subtasks ST-15 … ST-24 and ST-33: editor, validation, versioning, variables and rules, parallel review, per-item embed, timers, simulation, determinism, one SR per request, embeddable workflow templates |
| §27–§29 (part 6); §14 cont. (part 7); §30–§34 (part 8); §35–§37 (part 9); §38–§40 (part 10) | Fulfilment exception handlers, cloning, coverage mandate; ST-25 … ST-30; module architecture, Rust conventions; exception runtime, races, effects, cloning algorithm, coverage CI (D-34 … D-37, ST-31); exception and cloning UI, PM walkthroughs, S10–S12 state tables (ST-32) |
| §41 (part 11) | **Variable scope and data flow**: static scope per node, platform catalogue, copy-in `in`, copy-out `out` and per-item reducers, additive `ref` aliases across lanes, V-53 … V-56 (D-44 … D-48, ST-34) |
| §42 (part 12) | **Expression (f(x)) and Function (JS) nodes**: FX, a ZEN-compatible expression subset; JavaScript `handler` in the `mvp-fn-runner` QuickJS-on-WASM sandbox; jailbreak controls; inputs, snapshot, outputs as variables, error targets and rule port labels; `FOUR_EYES_PUBLISH`; V-57 … V-69 (D-49 … D-61, ST-35, ST-36, S14) |
| Appendix A | The user's illustrative procurement example, verbatim, marked **not our domain** |

> **Amended 2026-11-08 and 2026-11-10.** Part 4 adds the graph editor and a definition-driven interpreter; parts 6, 8, 11 and 12 amend further. A later part wins over this one; part 4 §26, part 6 §27.13 and part 8 list every superseded clause.

Notation:
- AC format is **G**iven / **W**hen / **T**hen.
- "No side effect" means two things together: no app-DB row was inserted or updated, and no workflow
  history event was appended other than a rejected-update record.
- Every time is stored as UTC and shown as SGT (UTC+8).
- **Terminology.** "Organisation" is the tenant: the app's `organizations` concept. Prose uses the British
  spelling; identifiers keep the app's `organization*` spelling from `server/db/schema.ts`. "Agency" keeps
  only its existing app meanings: user affiliation `agency`, cluster type `agency`, custody state
  `received_by_agency`, and agency teams or staff.

---

## §1 Context and goal

The goal is to demonstrate a Temporal workflow engine driving **our** break-glass request lifecycle end
to end, using the identity and authorization model the app already has. That model is: organisation (the
tenant), clusters, teams at levels L1–L5, team memberships with the team_admin role, admin roles and
grants, system owners (primary, backup and delegate) and the PA team. The demo covers:

0. **A graphical workflow editor and a definition-driven engine** (part 4). Each system has one
   versioned workflow definition drawn in the editor; each request runs an actualised plan of the
   pinned version, with the per-item embed expanded into one lane per item (§20). The seeded HRIS
   workflow demonstrates variables, fork/join with mutation, quorum, configurable rules, timers,
   return loops and multiple terminals (§25). **One request = one SR / reference / activity on one
   system** (D-24); every reviewer may reject or return (D-25).
1. **Sequential configurable approval chain** (now one valid shape of a definition, part 4 §26).
   Each system's chain is an ordered list of
   `(cluster, level)` steps that is snapshotted onto the request, followed by a terminal owner stage.
   Any member of a step's team may decide that step, and the first decision wins. A reviewer on a later
   step may override.
2. **Branches.** Approve, return-to-vendor (which loops to resubmit), reject (terminal), admin reset,
   auto-cancel when the submitter is tombstoned, and auto-expire when the activity window elapses.
3. **Fork/join of fulfilment.** On owner approval, one physical-custody child workflow starts per item.
   The request becomes `credential_issued` when the **join predicate** holds, which is the
   break-glass replacement for the draft's `total_spend <= budget` rule.
4. **Unhappy custody edges** (drawn, configurable handlers: part 6 §27).
   - Multiple items, where one fails and the others proceed.
   - An item lost before issuance: replacement, or write-off.
   - An item lost or damaged after issuance: issuance stands and custody stays attention-required.
   - Disputes, false reports, and overdue returns.
5. **Durable timers.** Approval deadline, per-stage SLA, return due, return overdue.
6. **Resilience.** Survives a worker crash, deduplicates retries, and stays consistent between the
   engine and the record.

---

## §2 How our app relates to Temporal, and where the data lives

### §2.1 The one-sentence rule

> **Temporal owns *progress*. Our Postgres owns *facts*.**

Temporal knows what the request is waiting for, which timers are armed and what runs next. It does
not hold the system of record for anything a person typed, decided or uploaded.

### §2.2 What Temporal's persistence actually is

Temporal has a database: SQLite for `temporal server start-dev`, and Postgres, MySQL or Cassandra in
production. It **is durable**. If a worker or the server restarts, nothing is lost. What it stores is
an **event history per workflow execution**, an append-only log that includes:

- `WorkflowExecutionStarted` (with input payload)
- `WorkflowExecutionUpdateAccepted` / `…Completed` (with update args and result)
- `WorkflowExecutionSignaled` (with signal payload)
- `ActivityTaskScheduled` / `…Completed` (with activity input and output)
- `TimerStarted` / `TimerFired`
- `StartChildWorkflowExecution…`, `ChildWorkflowExecutionCompleted`
- `WorkflowExecutionContinuedAsNew`, `WorkflowExecutionCompleted`

When a worker picks a workflow up again, it **replays** the workflow code against this history to
rebuild in-memory state. That is the only purpose of the history. It is *not* a system of record,
for four reasons:

| Property | Temporal history | What a system of record needs |
|---|---|---|
| Retention | Deleted after the namespace retention period (MVP: 7 days after the workflow closes). The MVP copies each closed history into Postgres first (D-18) | 7-year audit retention (FDS key decision 1/2) |
| Query model | Per workflow ID; visibility search on a few search attributes only | Relational joins, lists, reports, exports |
| Shape | Opaque, serialized payloads tied to code version | Typed, constrained, FK-enforced rows |
| Mutability | Append-only per run; continue-as-new starts a fresh history | Corrections, soft-delete, tombstones |

### §2.3 Where each kind of data lives

| Data | Lives in | Written by | Read by |
|---|---|---|---|
| Users, organisations, clusters, teams, memberships, roles, systems, chains, custody configs, assets | App Postgres | API (admin CRUD) | API, activities |
| Request content (justification, SR number, activity window, items, hostname descriptions) | App Postgres | Activity `persist_submission` / `apply_approver_edit` / `apply_resubmit` | API lists/detail |
| Decisions and **remarks** (step `decision_notes`, `owner_decision_notes`, `closing_notes`, reset notes) | App Postgres | Lifecycle activities | Timeline, audit |
| Custody events, handoffs, exceptions, exception updates, resolution notes | App Postgres | Custody activities | Timeline, dashboards |
| Evidence files (bytes) | App Postgres `bytea` | **API directly**, as pending rows before the engine call | Detail page |
| Audit log | App Postgres | Same transaction as every business write | Audit views |
| Request status / ordinal / case state (projection) | App Postgres columns **and** workflow state | Activities only (D-3) | API lists (DB), live graph (query) |
| "What am I waiting for", armed timers, decision lock, child handles | **Temporal workflow state** | Workflow code | Query `projection` |
| Update and signal payloads (including a transient copy of remark text) | **Temporal history** (live for 7 days after close) | Temporal | Replay only |
| Archived copy of each closed workflow history | App Postgres `workflow_history_archive` (compressed bytea, SHA-256), kept 7 years (D-18) | History archiver | Auditors (download), replay tests |
| Issuance and return of each item: who handed over, who received, when returned, verified and restocked | App Postgres: `custody_events` (ledger) and the timestamp columns on `custody_cases` (D-19) | Custody activities | Issuance and returns panel, reports, asset history |

### §2.4 Life of a remark (example)

Claire returns request `BG-HRIS-20261107-004` with the remark "SR number does not match change
ticket".

```
Browser ──POST /api/requests/{id}/step-decision {decision:"return", notes:"SR number…", expectedOrdinal:4}
  │        X-Idempotency-Token: 01J…
  ▼
axum API
  1. session → actor = Claire; validate notes (1–2000, non-blank)          ── 422 if invalid, nothing else happens
  2. DB read: can Claire see request? (404 if not) Is she in step-4 team? (403 if not)
  3. Temporal: ExecuteUpdate(workflow "bg-AUR-{id}", update "step_decision",
               updateId = idempotency token, args = {actor, decision, notes, expectedOrdinal})
  ▼
Workflow (worker)
  4. validator: status ∈ {submitted, under_review}; ordinal == 4; no decision in flight;
                actor ≠ submitter                                           ── rejected update → 409, no history change
  5. handler: set decision_lock; call activity record_step_decision(...)
  ▼
Activity (worker → app Postgres, ONE transaction)
  6. INSERT workflow_ops(op_ref) ON CONFLICT DO NOTHING → if exists, return stored result
  7. re-check Claire ∈ live team(cluster, level) of step 4                  ── non-retryable NOT_STEP_REVIEWER
  8. UPDATE request_approval_steps SET status='returned', decided_by, decided_at, decision_notes='SR number…'
  9. UPDATE breakglass_requests SET status='returned_to_vendor', return_count = return_count+1
     WHERE id=… AND status IN (…) AND current_step_ordinal=4                 ── guarded
 10. INSERT audit_log(action='request.step_returned', actor=Claire, details={ordinal:4, notes:'SR number…'})
 11. COMMIT  (if 10 fails → ROLLBACK, activity fails, update returns 500 AUDIT_WRITE_FAILED)
  ▼
Workflow
 12. apply result to projection; clear lock; return {status:'returned_to_vendor'}
  ▼
API → 200 {data:{status:'returned_to_vendor'}} → UI refetches detail (DB) and graph (query)
```

The remark's **authoritative copy** is `request_approval_steps.decision_notes`, along with the audit
row, both kept for 7 years (§2.7). A **second copy** sits in Temporal history as the update argument and
the activity input. That copy is live for 7 days after the workflow closes and is also archived to
Postgres (D-16, D-18). The UI never reads remarks from Temporal.

### §2.5 Why the workflow isn't simply the database

- Temporal adds durable waiting (for a person, a timer or a child), crash-safe sequencing, retries
  with backoff, exactly-once orchestration with idempotent activities, and visibility into "what is
  stuck where".
- Postgres adds integrity, relational authorization, reporting, retention and correction.
- Business rules that need relational facts, such as "is this user in that team right now?", are
  evaluated **in activities against the DB**. Workflow code must stay deterministic and cannot query
  the DB.
- The workflow decides *whether the state machine allows* an action. The DB transaction decides
  *whether this person may do it now* and records it.

### §2.6 Consistency contract

1. A lifecycle column (`status`, `current_step_ordinal`, `custody_cases.state`, exception `status`)
   changes **only** inside an activity transaction that also writes the audit row and the `op_ref`.
2. The workflow updates its projection **only from activity results**, so the projection and the DB
   cannot disagree after an activity completes.
3. If an activity is retried after a commit that was never acknowledged, the stored `op_ref` result is
   returned, so the write happens exactly once.
4. `GET /api/requests/:id/engine` and `POST /api/ops/reconcile` compare the projection with the DB and
   report divergence. They never auto-repair.

### §2.7 Long-term storage and retention: where audit data lives for 7 years

**Every long-term record is in the app's Postgres. Temporal is never relied on for anything older
than its 7-day window.** That is the FDS rule: the audit log is the single source of truth, kept for
7 years, and an operation commits only if its audit row is written (FDS chunk 01, key decisions).

| Data | Long-term store | Kept for | Protection |
|---|---|---|---|
| Audit log (every action, actor, time, before/after values, remark text in `details`) | `audit_log` | 7 years after the request closes | Append-only: the app DB role has INSERT and SELECT only. Written in the same transaction as the change, or the change rolls back |
| Request content, decisions and remarks | `breakglass_requests`, `request_approval_steps`, `request_credential_items` | 7 years after the request closes | Soft delete only (`expires_after`); no hard DELETE |
| Custody ledger: issuance, handover, receipt, return, verification, restock, exceptions, resolutions | `custody_events`, `custody_handoffs`, `custody_exceptions`, `custody_exception_updates`, `custody_resolution_proposals` | 7 years after the request closes | `custody_events` and `custody_exception_updates` are append-only (as audit) |
| Evidence files | `custody_evidence_files` (bytea, SHA-256) | 7 years after the request closes | Never edited; hash checked on download |
| Users, organisations, teams, roles, systems, assets | Their own tables | For as long as referenced, then 7 years | Tombstone or soft delete; names stay readable in history |
| Temporal workflow history (engine trail) | Live in Temporal for 7 days after close, then **archived** to `workflow_history_archive` (D-18) | 7 years after the request closes | Archive rows are append-only; SHA-256 recorded; zstd-compressed protobuf |

Rules for the MVP:
1. **Nothing is purged.** A purge job that deletes records older than 7 years is Future (§17). The
   MVP only proves that nothing is lost when Temporal's copy goes.
2. **Backups.** `make demo-backup` runs `pg_dump` of the app DB. A production deployment adds
   point-in-time recovery; the Temporal database needs no long-term backup, because its closed
   histories are archived.
3. **Reconstruction test.** With Temporal's database deleted entirely, the request page, timeline,
   audit export and the issuance and returns report still render in full (AC-1.5.2).

---

## §3 Ground truth from the current app

### §3.1 What the MVP mirrors

| Area | Current behaviour | Source |
|---|---|---|
| Request statuses | `submitted, under_review, returned_to_vendor, rejected, approved, credential_issued, closed`. `rejected` and `closed` are terminal. No expired, withdrawn or cancelled status. | `server/db/schema.ts:88`; `setup/sql/001_schema.sql:110-118` |
| Step statuses | `pending, in_review, approved, rejected, returned, skipped, submitted` (`in_review` and `skipped` are never written) | `schema.ts:98` |
| Position | `(status, current_step_ordinal)`: ordinal 1 is the submitter's own step (status `submitted`, decided by the submitter); the first reviewer is ordinal 2; `under_review` with ordinal 0 means the owner stage | addendum-027; `crates/ffs-server/src/workflow_machine.rs:31-72` |
| Transition table | StepApprove → next ordinal or 0; StepReturn → `returned_to_vendor` (ordinal kept); StepReject → `rejected` (ordinal kept); OwnerApprove/Return/Reject; Submit from returned; IssueCredential; Close; AdminReset → `returned_to_vendor` | `workflow_machine.rs`; `server/workflow-machine.tsx:154-196` |
| Step decision | Any member of the live team at the snapshot's `(required_cluster_id, required_level)`; the vendor must serve a linked org; notes required for return/reject (≤2000); override by a reviewer of a **later** step (`decided_via_override`) | `crates/ffs-server/src/routes/requests.rs:1018-1164`, `:205`, `:572` |
| Owner decision | Primary or backup owner (`system_owners`), or delegate inside `active_from..active_until`; notes required for return/reject; approval instantiates custody and notifies PA | `requests.rs:1174-1309`, `:226` |
| Approver edit | Current-step reviewer only, no override; edits justification, SR number, window and items | `requests.rs:1449-1562` |
| Resubmit | Submitter only, from `returned_to_vendor`; snapshot **rebuilt from the current system chain** | `requests.rs:1605-1753` |
| Admin reset | Privilege `RequestAdminReset`; any status except `closed` → `returned_to_vendor`, steps >1 back to `pending`, owner decision cleared, custody run voided | `requests.rs:1994-2079`; addendum-217 |
| Tombstone auto-cancel | Tombstoning a submitter → their `submitted`/`under_review`/`returned_to_vendor` requests become `rejected` (System actor, `request.auto_cancelled`) | `crates/ffs-server/src/routes/admin/users_lifecycle.rs:97-160` |
| Visibility | Submitter; global admin; pre-owner reviewer teams; owners/delegates only from the owner stage | `requests.rs:1809-1826` (`req_can_view`) |
| Custody (non-AWS fulfilment) | One case per manual item on owner approval; per-item states; 5 per-system stage configs (agency team, instructions, `due_in_hours`); requester stages bound to the submitter; combined action when consecutive stages share a team | addenda 144/168/177; `schema.ts:123, 1377`; `server/custody/service.tsx:195-240`; `crates/ffs-server/src/domains/custody.rs` |
| Issuance | Custody-only request auto-issues when every case has reached requester custody (counting `state_before_exception` for cases in exception) | `domains/custody.rs` `should_auto_issue_custody_request` |
| Close | Requester or reviewer from `credential_issued`; independent of custody return | `requests.rs:1375-1435`; addendum-168 |
| Exceptions | 10 types; statuses `open, investigating, awaiting_information, awaiting_resolution_approval, resolved, cancelled_as_false_report`; one open exception per case; evidence as bytea (jpeg/png/webp/pdf/txt, ≤10 MiB) | `schema.ts:149-192, 1690-1726`; addendum-144 |
| Audit | Audit write failure rolls the operation back; trigger writes `audit_log` + `audit_log_changes`; System actor `00000000-0000-7000-8000-000000000001` | `crates/ffs-server/src/audit.rs:12-65`; schema 2379 |
| Idempotency | `X-Idempotency-Token`, single use, consumed in the request transaction | `crates/ffs-server/src/middleware/idempotency.rs:44-163` |
| Request number | `BG-{SHORT_CODE}-{YYYYMMDD}-{NNN}` per system per day | `crates/ffs-server/src/utils.rs:97-100` |
| Identity | users (affiliation agency\|vendor, status, tombstone, is_system/protected); organizations; clusters (vendor\|agency\|pa); teams (L1–5, unique (cluster, level)); team_memberships (member\|team_admin); system_owners; system_delegates; system_owner_pa_mappings; system_vendor_clusters; system_workflow_steps; admin_roles(capabilities[]); user_admin_role_grants (global or org-scoped) | `schema.ts:244-2899` |
| Sessions | HttpOnly cookie; 24 h lifetime; 15 min idle; 5 attempts → 30 min lockout | `crates/ffs-server/src/app_settings_defaults.rs:4-12` |

### §3.2 Gaps in today's app that this MVP closes (it specifies new behaviour)

| Gap | Today | MVP |
|---|---|---|
| G-1 Runtime self-approval | Enforced only when a chain is saved (`server/utils.tsx:354`); no decision handler compares the decider with `submitted_by` | Enforced on every step, owner and resolution decision (D-6) |
| G-2 Timers | `due_in_hours` and the activity window are display-only (FDS chunk-04:87) | Durable Temporal timers (D-9, D-10) |
| G-3 Exception resolution | Report and investigate only; recovered, replacement, write-off and waiver unimplemented | Full resolution matrix (§9.3, D-12) |
| G-4 Rust owner return/reject | UPDATE has no status guard | Guarded in validator and SQL |
| G-5 Rust semantic audit | Return, reject, issue, close and edit write only the generic `breakglass_request.updated` | A semantic action for every transition |
| G-6 Rust approver edit | Silently skips invalid assets | Full validation, 422 |
| G-7 Reset after handover | Voids the run even when physical items are out | Blocked unless forced; post-handover cases continue (D-14) |
| G-8 Request-number date | The FDS says SGT; the code uses UTC | SGT (D-15) |

### §3.3 AWS branch points excluded

- `aws::secret_delivery::publish_request_non_fatal` (`requests.rs:1272`)
- `publish_request_close_non_fatal` (`requests.rs:1432`)
- `aws/secret_delivery.rs:168-208` reconciliation
- the `tenant-inbound-sweep` job
- `request_key_material.encrypted_secret_payload` / `secretMaterialReady`
- the exchange route in `domains/fulfillment.rs:17-26`
- all of `crates/ffs-server/src/aws/`

`request_key_material` is omitted entirely because the MVP issues no secrets.

---

## §4 MVP decisions

| ID | Decision | Rationale | Divergence from app |
|---|---|---|---|
| D-1 | **Organisation is the tenant.** The app's existing organisation concept (`organizations`, addenda 114/200/206) becomes the hard data boundary. `organizations(short_code, name)`; every cluster, system, asset, request and audit row carries `organization_id`. Users are global identities linked by `organization_memberships` (many-to-many, as today). Admin grants are global (`organization_id NULL`) or organisation-scoped, as `user_admin_role_grants` is today; an `org_admin` is always scoped to one or more named organisations (D-38). Vendor clusters are anchored (NOT NULL) to the organisation they serve (addendum-206 D2), so a vendor company serving two organisations is two clusters. Each system has exactly one owning organisation (`systems.organization_id`) | User decision (the term "organisation" is retained); it is the clearest demonstration of isolation | Today organisations scope admin powers and the vendor service gate, while agency-affiliated users may participate across organisations through `system_organizations` participant edges (addendum-206). The MVP makes the organisation a hard boundary and defers participant edges (§17) |
| D-2 | Temporal owns progress; Postgres owns facts (§2) | Retention, querying, integrity | — |
| D-3 | **Single writer.** After creation, lifecycle columns are written only by workflow activities | No dual-writer drift | — |
| D-4 | **Human actions are Temporal Updates with validators.** Update ID = client `X-Idempotency-Token`. Signals only for system events (`submitter_tombstoned`, child → parent milestones) | Synchronous result for the UI; deduplication | — |
| D-5 | **Three-layer authorization**: (1) API checks visibility (404) and eligibility (403) from the DB; (2) the workflow validator checks the state precondition, expected position, lock and self-approval; (3) the activity re-checks membership inside the transaction (TOCTOU) and fails non-retryably on denial | Defence in depth; deterministic workflow | — |
| D-6 | **Runtime self-approval prevention**: actor ≠ `submitted_by` for step, override, owner and resolution decisions; resolution approver ≠ proposer and ≠ reporter | Closes G-1 | New |
| D-7 | One parent workflow per request, `BreakglassRequestWorkflow`, ID `bg-{ORG}-{request_uuid}`. One child per custody case, `CustodyCaseWorkflow`, ID `bg-{ORG}-{request_uuid}-case-{case_uuid}`. Single namespace `breakglass-mvp`; search attributes `OrgCode` (Keyword), `SystemCode` (Keyword), `RequestStatus` (Keyword) | Simple ops; organisation filter via search attribute | — |
| D-8 | **Continue-as-new** on resubmit, admin reset, and when the history length is over 2,000 events. The new run reloads its projection from the DB with the activity `load_projection`; children are re-attached by ID | Bounded history; proves the DB is the source of truth | — |
| D-9 | **Durable timers**: approval deadline; per-stage SLA from the case's snapshotted `due_in_hours`; exception SLA; return due; return overdue. `SLA_UNIT_SECONDS` (default 3600; demo 10) converts hour-valued configs | Closes G-2 | New |
| D-10 | **Auto-expire**: if `activity_end_at` passes before owner approval, the System actor sets `rejected` with `decision_reason='activity_window_elapsed'`, audit `request.auto_expired` | No new enum value; mirrors tombstone auto-cancel | New |
| D-11 | Submit, resubmit and edit require `activity_end_at > now()` in addition to `end > start` | Prevents dead-on-arrival requests | New |
| D-12 | **Exception resolution** per addendum-144 §5: recovered and false report decided by the investigator; replacement, write-off and waiver need owner approval (D-6 applies) | Closes G-3 | New |
| D-13 | **Issuance join predicate**: `live = cases where not excluded`; issue when `live ≠ ∅ ∧ ∀c∈live: crossed_receipt(c) ∧ pending_replacements = 0`. If `live = ∅` because every item was written off, the System actor sets `closed` with `decision_reason='unfulfilled_all_items_written_off'`, audit `request.closed_unfulfilled` | Deterministic rule for partial fulfilment | New transition `approved → closed` (System only) |
| D-14 | **Admin reset after handover**: 409 `CUSTODY_IN_PROGRESS` if any case is at or past `handover_pending` and not terminal, unless `force=true` with notes. Force cancels pre-handover cases only; post-handover cases continue to return | Physical reality can't be voided | Stricter (G-7) |
| D-15 | **Parity choices**: request number uses the SGT date; `return_count` counts every return to vendor (step and owner returns), `resubmit_count` counts resubmissions, and issuance and returns are tracked per item (D-19); owner return/reject guarded; a semantic audit action for every transition; the submitter is any active member of the ordinal-1 team whose affiliation matches the cluster type | Fixes G-4/5/6/8 | Minor |
| D-16 | Remark text crosses update payloads, so a copy also lives in Temporal history. That copy is **not** the audit record: the authoritative copy and the audit row are in Postgres for 7 years (§2.7), and the history itself is archived to Postgres at close (D-18). Temporal's namespace retention stays at 7 days to keep the engine DB small. The UI reads only the DB. Payload-codec encryption is Future. **Evidence bytes never enter Temporal**; only file-row IDs do | Audit retention comes from Postgres, not Temporal | — |
| D-18 | **History archive.** A history archiver (a loop in the worker binary, every 60 s) finds closed workflows with no archive row, fetches the full history with `GetWorkflowExecutionHistory`, and inserts one `workflow_history_archive` row per run (zstd-compressed protobuf, SHA-256, event count, close status). The namespace retention period (7 days) leaves several days of margin. An archive failure raises an ops alert on `/api/health` (`archive: "lagging"`) | Keeps the engine trail for audit even though Temporal deletes it | New |
| D-19 | **Issuance and return tracking.** Each custody case records who issued it and when (`handed_over_by/at`, `issued_to/at`, set when the requester confirms receipt), and the return (`return_initiated_at`, `returned_at`, `return_verified_by`, `restocked_by/at`). These columns are written only by the custody activity that writes the matching ledger event. A replacement creates a second issuance for the request. The views `request_issuance_summary` and `asset_custody_history` derive counts from these columns and the ledger: items issued, outstanding (issued and not returned), returned, overdue, replaced and written off; and, per asset, every issue and return across requests | Requested by the user (OP-1) | New |
| — | **D-20 … D-27** (editor, per-system versioned definition, generic interpreter, late-bound actualisation R1–R10, one SR per request, mandatory return/reject, undrawn platform interrupts, typed variables) are in part 4 §20.4 | User request 2026-11-08 | New |
| D-17 | Next.js proxies `/api/*` to axum through same-origin rewrites. HttpOnly, SameSite=Lax session cookie plus an Origin check on mutating routes. **No permissive CORS** | The draft's `CorsLayer::permissive()` is unsafe | — |
| D-38 | **`org_admin` is scoped to named organisations, never to all.** (1) *Grant shape:* `admin_roles.grant_scope` (`global` \| `organisation`, CHECK) is `global` for `full_admin` and `organisation` for `org_admin`; a `full_admin` grant has `organization_id NULL` and an `org_admin` grant has `organization_id NOT NULL`, otherwise 422 `GRANT_SCOPE_INVALID` (API) and a trigger on `user_admin_role_grants` (DB). (2) *Membership:* each `org_admin` grant requires a live `organization_memberships(organization_id, user_id)` row for the same organisation, otherwise 422 `GRANT_NOT_MEMBER` (API and trigger). (3) *Multi-select:* `POST /api/admin/grants {userId, roleKey, organizationIds[]}` writes one grant row per organisation in one transaction (non-empty for `org_admin`, absent or empty for `full_admin`); an organisation that already has a live grant is skipped (idempotent, no duplicate); each row is revoked on its own (`POST /api/admin/grants/:id/revoke`), so revoking one organisation leaves the others. (4) *Admin scope:* `admin_scope(user)` = the organisations with a live `org_admin` grant, read from the DB on every request (AC-2.6.2). Wherever this addendum or addendum 220 says an organisation admin acts on their "own" organisation, it means an organisation in the caller's admin scope: lists filter `organization_id = ANY(scope)`; reads and writes resolve the object's organisation first and return 404 if it is outside the scope (existence not revealed); create forms require an organisation from the scope. (5) *Membership removal* revokes that user's `org_admin` grant for that organisation only, in the same transaction, with audit `admin_grant.revoked` (reason `membership_removed`, `revoked_by` = the actor). (6) *Identity actions* (tombstone, US-2.4) need the target's memberships ⊆ the caller's scope (addendum-137 identity-subset rule), otherwise 403 `ADMIN_SCOPE_INSUFFICIENT`; a target with no membership is global-plane (addendum-142). (7) *Global-only:* create and delete organisations (an `org_admin` may rename an organisation in scope, never change its short code); grant and revoke admin roles; `/api/ops/reconcile`; cross-organisation cloning (OP-8), even for an `org_admin` of both organisations. (8) *No auto-inclusion:* an organisation created later is outside every `org_admin` scope until it is granted | Requested by the user (2026-11-08): the earlier wording ("own organisation") left the boundary undefined because users belong to several organisations, and read literally gave `org_admin` every organisation. Requiring membership keeps an organisation admin accountable to the organisation they administer | **Reverses addendum-142 "cross-org support grants: ALLOW"** for this MVP (a scoped grant there may name an organisation the grantee is not a member of). The schema already allows N scoped rows per user and role (`user_admin_role_grants_user_role_org_uniq`); `grant_scope` and both triggers are new |

---

## §5 Architecture

### §5.1 Components and ports

| Process | Tech | Port | Responsibility |
|---|---|---|---|
| `web` | Next.js 15 (App Router, TypeScript), `@xyflow/react` 12 | 3000 | UI; `/api/*` rewrite to `api` |
| `api` | Rust 1.99.1, axum 0.7, sqlx 0.8 (Postgres, compile-checked), `temporalio-client` 1.1.x | 3001 | Sessions, authorization pre-checks, reads from the DB, Temporal update/start/query calls, admin CRUD, evidence upload |
| `worker` | Rust, `temporalio-sdk` 1.1.x | — | Hosts both workflows and all activities; task queue `breakglass` |
| `temporal` | `temporal server start-dev --namespace breakglass-mvp --db-filename .temporal/dev.db` | 7233 / UI 8233 | Engine |
| `postgres` | PostgreSQL 16 | 5433 | App DB (system of record) |
| `fn-runner` | Rust `mvp-fn-runner`, `wasmtime`, QuickJS compiled to WASM | Unix socket `/run/fn-runner/sock` | Runs Function-node JavaScript in a fresh sandbox per call; no network, no secrets, no DB (part 12 §42.8, §42.9) |

`api` and `worker` are two binaries in one Cargo workspace; the crate and module map is in part 8 §30 (it supersedes any earlier crate list). The worker never serves HTTP. The API never runs workflow code.

### §5.2 Sequence: submit (Update-With-Start)

```
web → api POST /requests (token T)
api: validate body (422) → visibility/eligibility (404/403) → mint request_id (UUIDv7)
api → temporal: UpdateWithStart(workflowId=bg-AUR-{id}, startArgs={request_id, org, mode:Fresh},
                                update "submit"(updateId=T, SubmitPayload), conflict=FAIL)
worker: validator(state=Empty) ok → activity persist_submission (tx: request, number, snapshot, items,
        asset holds, audit request.submitted, op_ref) → projection := result → arm approval-deadline timer
api ← {requestId, requestNumber, status}  → 201
```

The fallback when Update-With-Start is unavailable in the SDK version: start the workflow with
`StartInput`, then call update `submit` with the same update ID. A duplicate start is rejected because
the ID is already in use.

### §5.3 Sequence: owner approval, fork and join

```
api → update owner_decision{approve}
worker: validator → activity record_owner_decision (tx) → status approved
        → activity instantiate_custody (tx: run, cases, stage snapshots, audit custody.run_started)
        → for each case: start child CustodyCaseWorkflow(case_id) (ParentClosePolicy=ABANDON)
        → cancel approval-deadline timer
child: wait for actions (updates sent by api directly to child id) … on reaching in_requester_custody:
        signal parent "case_milestone"{case_id, milestone:"crossed_receipt"}
parent: on milestone → evaluate join (D-13) → if true and status==approved → activity mark_issued (tx, once)
```

### §5.4 Sequence: timer

```
parent: timer(approval_deadline = activity_end_at - now) races wait_condition(status ∉ pre-approval)
  fired first → activity auto_expire (tx guarded WHERE status IN pre-approval) → status rejected → complete
  condition first → timer cancelled
```

---

## §6 Domain model (MVP DDL outline)

> **Amended (part 4 §23).** `system_workflow_steps` and `system_custody_stage_configs` are replaced
> by the definition tables in §23.1; `request_approval_steps` becomes a view over
> `request_node_instances` (§23.2); request plan and variable tables are added in §23.2.

Migrations are sqlx files in `sidequest/temporal-breakglass-mvp/migrations/`. The repo invariants
apply:
- UUIDv7 primary keys minted in one place (`Uuid::now_v7()`);
- soft delete through `expires_after TIMESTAMPTZ`, with partial unique indexes `WHERE expires_after IS NULL`;
- liveness-aware FKs, enforced by a trigger that rejects references to expired rows;
- a seeded System actor row;
- **no JSONB for business data**. `audit_log.details` and `workflow_ops.result` are metadata or
  bookkeeping, which matches today's `audit_log.details`.

Enums are copied from the app: `request_status`, `step_status`, `step_kind`, `cluster_type`,
`team_role`, `user_affiliation`, `user_status`, `system_owner_kind`, `custody_stage`,
`asset_custody_state` (+ `cancelled`), `asset_custody_event_type` (+ `exception_cancelled`,
`return_due`, `return_overdue`, `sla_breached`, `asset_written_off`, `waiver_granted`),
`asset_custody_exception_type`, `asset_custody_exception_status`, `asset_custody_exception_severity`,
`asset_handoff_type`, `asset_handoff_status`, `custody_evidence_media_type`.

New enums:
- `exception_resolution` (`recovered, false_report, replacement, write_off, waiver`)
- `resolution_proposal_status` (`proposed, approved, rejected`)
- `decision_reason` (`activity_window_elapsed, submitter_tombstoned, unfulfilled_all_items_written_off`)

| Table | Key columns (beyond id, created_at, updated_at, expires_after) | Constraints |
|---|---|---|
| `organizations` | short_code varchar(10), name | short_code uppercased, `^[A-Z0-9]{1,10}$`, unique live (addendum-200); `lower(name)` unique live |
| `users` | email, full_name, password_hash, password_salt, affiliation, status, is_system, is_protected, tombstoned_at/by | email unique where status not in (tombstoned, rejected) |
| `organization_memberships` | organization_id, user_id | unique live (organization, user) |
| `clusters` | organization_id, name, type | type immutable once teams exist (trigger) |
| `teams` | cluster_id, name, level smallint | level 1..5; unique live (cluster_id, level) |
| `team_memberships` | team_id, user_id, team_role | unique live (team, user); trigger: one level per user per cluster; affiliation matches cluster type (vendor ↔ vendor; agency ↔ agency/pa) |
| `admin_roles` | key (PK), name, capabilities text[], grant_scope | seeded `full_admin` (`global`), `org_admin` (`organisation`); grant_scope CHECK in (`global`, `organisation`) (D-38) |
| `user_admin_role_grants` | user_id, role_key, organization_id NULL, granted_by, revoked_at/by, revocation_reason | grantee active, agency-affiliated, not system/protected; unique live (user, role, coalesce(organization_id)); trigger: `organization_id` NULL iff the role's grant_scope is `global`; trigger: an organisation-scoped grant needs a live membership of the grantee in that organisation (D-38) |
| `systems` | organization_id, short_code, name, description, agency_cluster_id | short_code unique live per organisation |
| `system_vendor_clusters` | system_id, cluster_id, ordinal | cluster type vendor, same organisation |
| `system_owners` | system_id, user_id, kind | one live per (system, kind); primary ≠ backup |
| `system_delegates` | system_id, owner_id, delegate_user_id, active_from, active_until | until > from |
| `system_owner_pa_mappings` | system_owner_id, pa_team_id | team in pa cluster, same organisation |
| `system_workflow_steps` | system_id, ordinal, cluster_id, level, step_kind, notes | unique live (system, ordinal); contiguous from 1 (validated in the API) |
| `system_custody_stage_configs` | system_id, stage, agency_team_id, instructions, due_in_hours | unique live (system, stage); team in agency cluster; due 1..720 or NULL |
| `assets` | system_id, hostname, credential_name, credential_type, description, retired_at, retired_reason | unique live (system, hostname, credential_name) |
| `asset_holds` | asset_id, request_id, request_item_id, released_at | **unique (asset_id) WHERE released_at IS NULL** (prevents double booking) |
| `request_number_sequences` | system_id, seq_date (SGT), last_value | PK (system_id, seq_date) |
| `breakglass_requests` | organization_id, system_id, request_number, submitted_by, status, current_step_ordinal, justification(2000), sr_number(100), activity_start_at, activity_end_at, return_count, resubmit_count, owner_decision_by/at/notes(2000), credential_issued_at, closed_by/at, closing_notes(2000), decision_reason, workflow_id, workflow_run_id | request_number unique |
| `request_approval_steps` | request_id, ordinal, required_cluster_id, required_team_type, required_level, step_kind, status, decided_by, decided_at, decision_notes(2000), decided_via_override | unique live (request, ordinal) |
| `request_credential_items` | request_id, asset_id, credential_type, hostname_description(200), replaces_item_id, excluded_at, excluded_reason | — |
| `custody_runs` | request_id, status (`in_progress, completed, void`) | one open run per request |
| `custody_cases` | run_id, request_id, request_item_id, asset_id, state, state_before_exception, version, child_workflow_id, stage_due_at, overdue_at, return_due_at, return_overdue_at, handed_over_by/at, issued_to, issued_at, return_initiated_at, returned_at, return_verified_by, restocked_by/at | unique (request_item_id); timestamps set once and never cleared (trigger) |
| `custody_case_stage_snapshots` | case_id, stage, agency_team_id, due_in_hours | unique (case, stage); snapshotted at fork |
| `custody_handoffs` | case_id, handoff_type, status, initiated_by/at, initiation_notes, responded_by/at, response_notes | one pending per case |
| `custody_events` | case_id, event_type, from_state, to_state, actor_id, notes, op_ref | **append-only**: app role has INSERT/SELECT only |
| `custody_exceptions` | case_id, request_id, exception_type, status, severity, detected_during_state, reported_by, description(4000), expected_location, last_known_location, assigned_team_id, resolution, resolved_by/at, resolution_notes, overdue_at | unique (case_id) WHERE status open-ish (as today) |
| `custody_exception_updates` | exception_id, from_status, to_status, actor_id, notes | append-only |
| `custody_resolution_proposals` | exception_id, resolution, proposed_by, notes, substitute_asset_id, status, decided_by/at, decision_notes | one `proposed` per exception |
| `custody_evidence_files` | exception_id NULL, event_id NULL, status (`pending, attached`), uploaded_by, original_filename, media_type, byte_count, sha256, bytes bytea | exactly one of exception_id/event_id when attached; ≤10 MiB |
| `audit_log` | organization_id, actor_id, action, target_type, target_id, details jsonb, op_ref, created_at | append-only: app role has INSERT/SELECT only; kept 7 years (§2.7) |
| `workflow_history_archive` | workflow_id, run_id, request_id, organization_id, close_status, event_count, history_zstd bytea, sha256, archived_at | unique (workflow_id, run_id); append-only (D-18) |
| view `request_issuance_summary` | request_id, items_total, items_issued, items_outstanding, items_returned, items_return_overdue, items_replaced, items_written_off, first_issued_at, last_returned_at | derived; read-only (D-19) |
| view `asset_custody_history` | asset_id, request_id, request_number, issued_to, issued_at, returned_at, outcome (`returned`, `outstanding`, `written_off`, `replaced`) | derived; read-only (D-19) |
| `workflow_ops` | op_ref (PK), request_id, activity, result jsonb, created_at | idempotency ledger |
| `sessions`, `login_attempts`, `idempotency_tokens` | as in the app | — |

---

## §7 Workflow specification

> **Amended (part 4 §23.3).** The parent becomes a generic interpreter host using `mvp-engine`;
> `step_decision`, `owner_decision` and `close` are replaced by the single update `act` with
> `expected_revision`; `CustodyCaseWorkflow` becomes `LaneWorkflow`. Determinism rules, D-2…D-6,
> idempotency and the activity catalogue below still apply.

> **SDK note.** These names are from `temporalio-sdk` 1.1.x: `#[workflow]`, `#[workflow_methods]`,
> `#[init]`, `#[run]`, `#[signal]`, `#[update]` (with validator), `#[query]` (sync, read-only),
> `#[activities]`, `ctx.wait_condition`, timers, child workflows, continue-as-new,
> `temporalio_sdk::workflows::select!` (deterministic). The implementer **must verify exact
> signatures on docs.rs** before coding.
>
> Forbidden inside workflow code: `tokio::select!`, `tokio::time`, `std::time::SystemTime::now`,
> random numbers, DB/HTTP calls, `Rc<RefCell<…>>` shared state (state lives on the workflow struct),
> and `HashMap` iteration that affects decisions (use `BTreeMap`).

### §7.1 `BreakglassRequestWorkflow` (parent)

**Start input:** `StartInput { request_id, org_code, mode: Fresh | Resumed { reason } }`.
On `Resumed`, which is the case after continue-as-new, the first action is the activity
`load_projection(request_id)`.

**State (workflow struct):**

```
projection: Projection {
  request_id, request_number, org_code, system_code, submitted_by,
  status: RequestStatus, ordinal: i16, steps: Vec<StepView{ordinal, cluster_id, level, status}>,
  activity_start_at, activity_end_at, return_count,
  cases: BTreeMap<CaseId, CaseView{state, crossed_receipt, excluded, terminal, pending_replacement}>,
  timers: Vec<TimerView{kind, fires_at}>, join_trace: Option<String>,
  custody_outstanding: u32, history_hint: u32
}
decision_lock: Option<UpdateId>   // set synchronously at handler entry
seen_signals: BTreeSet<String>    // dedupe child milestones
```

**Updates (human actions).** Each carries `actor_id` plus `acting_as`. The API resolves both.

| Update | Validator (pure, on state) | Handler (activities) | Result |
|---|---|---|---|
| `submit` | state empty | `persist_submission` | projection |
| `step_decision{expected_ordinal, decision, notes?, override}` | status ∈ {submitted, under_review} ∧ ordinal>0 ∧ ordinal==expected ∧ lock none ∧ actor≠submitter ∧ (decision≠approve ⇒ notes non-blank) | `record_step_decision` | status, ordinal |
| `owner_decision{decision, notes?}` | status==under_review ∧ ordinal==0 ∧ lock none ∧ actor≠submitter ∧ notes rule | `record_owner_decision`; if approve: `instantiate_custody`, start children, cancel deadline | status |
| `approver_edit{expected_ordinal, content}` | status ∈ pre-owner ∧ ordinal==expected ∧ lock none | `apply_approver_edit`; re-arm deadline if the window changed | projection |
| `resubmit{content}` | status==returned_to_vendor ∧ actor==submitter | `apply_resubmit` → continue-as-new(Resumed) | status |
| `close{notes?}` | status==credential_issued | `close_request` | status (workflow keeps running while custody is outstanding) |
| `admin_reset{notes?, force}` | status≠closed ∧ (¬post_handover_active ∨ force∧notes) | cancel pre-handover children → `apply_admin_reset` → continue-as-new | status |

**Signals (system events):**
- `submitter_tombstoned{}`: if status ∈ pre-approval, run `auto_cancel`; otherwise no-op.
- `case_milestone{case_id, milestone: crossed_receipt | excluded | replaced{new_case_id} | terminal}`
  from children, deduplicated by `(case_id, milestone)`.

**Query:** `projection() -> Projection`.

**Main loop (pseudocode):**

```
run:
  if mode == Resumed: projection = activity load_projection
  loop:
    match projection.status:
      submitted | under_review | returned_to_vendor:
         select! { wait_condition(status changed by an update/signal) ,
                   timer(until activity_end_at) => activity auto_expire (guarded) }
         // returned_to_vendor still waits for resubmit; deadline still applies (D-10)
      approved | credential_issued | closed:
         wait_condition(join changed | status changed | all cases terminal)
         if status==approved && join(projection) == Issue  => activity mark_issued
         if status==approved && join(projection) == AllExcluded => activity close_unfulfilled
         if status ∈ {closed} && all cases terminal => return Ok(Completed)
      rejected => cancel children (pre-handover only; post-handover ones none exist pre-approval) ; return Ok
    if history_len > 2000 => continue_as_new(Resumed{reason:"history"})
```

`join(projection)`:

```
live = cases.filter(!excluded)
if live.is_empty() && cases.non_empty()           -> AllExcluded
if live.any(pending_replacement)                  -> Wait
if live.non_empty() && live.all(crossed_receipt)  -> Issue
else                                              -> Wait
join_trace = "live={l} crossed={c} excluded={e} pendingReplacement={p} -> {verdict}"
```

### §7.2 `CustodyCaseWorkflow` (child, one per item)

**Input:** `{ request_id, case_id, parent_workflow_id, submitted_by }`. The stage teams and due hours
are loaded by `load_case` from `custody_case_stage_snapshots`.

**State:** `{ state, state_before_exception, open_exception: Option<ExceptionView>, proposal: Option<ProposalView>, stage_timer: Option<TimerView>, return_timers }`.

**Updates.** The API sends these to the child ID directly:

| Update | Validator | Activity | Notes |
|---|---|---|---|
| `custody_action{action, notes?, evidence_ids[]}` | plan_for_state(state) contains action (§9.1) ∧ no open exception | `apply_custody_action` (multi-event for combined actions) | re-checks stage team, or submitter for requester stages |
| `dispute_handoff{notes}` | state==handover_pending (actor=submitter) or state==return_pending (actor=verify team) | `open_exception(handover_disputed / return_disputed)` | |
| `report_exception{type, severity, description, locations?, evidence_ids[]}` | no open exception ∧ state not terminal | `open_exception` | pauses this case only |
| `exception_update{to_status, notes}` | open exception ∧ transition allowed (§9.2) | `record_exception_update` | investigate team |
| `resolve_direct{resolution: recovered{resume_state} \| false_report, notes}` | open exception ∧ status ∈ {investigating, awaiting_information} | `resolve_exception_direct` | investigate team |
| `propose_resolution{resolution: replacement{substitute_asset_id} \| write_off \| waiver, notes}` | open exception ∧ no live proposal | `create_resolution_proposal` | investigate team |
| `decide_resolution{approve, notes?}` | live proposal ∧ actor ∉ {submitter, proposer, reporter} | `decide_resolution_proposal` | owner/backup/delegate |

**Timers:**
- Stage SLA: `due_in_hours × SLA_UNIT_SECONDS` from stage entry. When it fires,
  `flag_overdue(stage)` runs only if the case is still in that stage.
- Exception SLA: the investigate stage's due hours from when the exception opened.
- Return due: at `activity_end_at` if the state is `in_requester_custody`.
- Return overdue: at `activity_end_at + RETURN_GRACE_HOURS × SLA_UNIT_SECONDS`. It only flags; it
  never auto-opens an exception.

**Milestones signalled to the parent:**
- `crossed_receipt`: first entry into `in_requester_custody`.
- `excluded`: write-off approved before receipt.
- `replaced{new_case_id}`: the parent starts the new child.
- `terminal`: `returned_to_inventory`, `resolved_exception` (terminal), `cancelled`, or written off.

**Completion:** the child completes when its state is terminal.

### §7.3 Activity catalogue

Common rules for every activity:
- One DB transaction.
- `op_ref = {workflow_id}:{run_id}:{activity_id}` inserted into `workflow_ops`.
- The semantic audit row is written in the same transaction.
- Retry policy: initial 1 s, ×2 backoff, max 30 s, schedule-to-close 10 min.
- Non-retryable error types: `NOT_AUTHORIZED`, `NOT_STEP_REVIEWER`, `STALE_STATE`,
  `VALIDATION_FAILED`, `ASSET_UNAVAILABLE`, `SYSTEM_WORKFLOW_INVALID`. Any audit failure is retryable,
  and is surfaced as `AUDIT_WRITE_FAILED` once retries are exhausted.

| Activity | Writes | Audit action |
|---|---|---|
| `persist_submission` | request, number, steps snapshot, items, asset holds | `request.submitted` |
| `load_projection` | — (read) | — |
| `record_step_decision` | step, request (guarded) | `request.step_approved` / `request.step_override_approved` / `request.step_returned` / `request.step_rejected` |
| `record_owner_decision` | request owner_decision_* (guarded) | `request.owner_approved` / `request.owner_returned` / `request.owner_rejected` |
| `apply_approver_edit` | request content, items (replace), holds | `request.edited_by_approver` (old/new) |
| `apply_resubmit` | content, items, holds, snapshot rebuilt, owner decision cleared, `resubmit_count`+1 | `request.resubmitted` |
| `apply_admin_reset` | steps >1 → pending, owner decision cleared, run void if no post-handover case | `request.admin_reset` |
| `auto_expire` | status rejected, decision_reason, holds released | `request.auto_expired` |
| `auto_cancel` | status rejected, decision_reason, holds released | `request.auto_cancelled` |
| `instantiate_custody` | run, cases, stage snapshots | `custody.run_started` |
| `mark_issued` | status credential_issued (guarded `WHERE status='approved'`) | `request.credential_issued` |
| `close_request` | closed_by/at/notes | `request.closed` |
| `close_unfulfilled` | status closed, decision_reason | `request.closed_unfulfilled` |
| `load_case` | — | — |
| `apply_custody_action` | case state, version+1, events, handoff, and the matching D-19 timestamp (`handed_over_*`, `issued_*`, `return_initiated_at`, `returned_at`, `return_verified_by`, `restocked_*`) | `custody.{event_type}` |
| `open_exception` | exception, case → exception_open, state_before_exception, event, evidence attach | `custody.exception_reported` |
| `record_exception_update` | exception status, update row | `custody.exception_updated` |
| `resolve_exception_direct` | exception resolved / cancelled_as_false_report; case resumes | `custody.exception_resolved` |
| `create_resolution_proposal` | proposal; exception → awaiting_resolution_approval | `custody.resolution_proposed` |
| `decide_resolution_proposal` | approve: replacement / write-off / waiver effects (§9.3); reject: exception → investigating | `custody.resolution_approved` / `custody.resolution_rejected` |
| `cancel_case` | case cancelled, hold released | `custody.case_cancelled` |
| `flag_overdue` / `flag_return_due` / `flag_return_overdue` | case/exception overdue columns, event | `custody.sla_breached` / `custody.return_due` / `custody.return_overdue` |

---

## §8 Authorization matrix

Legend:
- Vis = may see the request (otherwise 404).
- Elig = DB-evaluated eligibility (otherwise 403).
- All checks are organisation-scoped: any cross-organisation object returns 404.

| Action | Eligible actor | State precondition | Notes required | Self-approval rule |
|---|---|---|---|---|
| Submit | Active member of the live team at ordinal-1 `(cluster, level)`, affiliation matches cluster type, member of the system's organisation | — | — | — |
| Step approve / return / reject | Member of the live team at the current snapshot step | submitted\|under_review, ordinal>0 | return/reject | actor ≠ submitter |
| Step override | Member of a team at a **later** snapshot step | same | as above | actor ≠ submitter |
| Approver edit | Current-step team member (no override) | pre-owner | — | — |
| Owner approve / return / reject | Primary or backup owner; delegate inside window | under_review, ordinal 0 | return/reject | actor ≠ submitter |
| Resubmit | Original submitter, still eligible to submit | returned_to_vendor | — | — |
| Close | Submitter or any member of a snapshot reviewer team | credential_issued | — | — |
| Admin reset | Grant with `request_admin_reset`, global or scoped to the request's organisation (D-38) | ≠ closed (D-14 rules) | if force | — |
| Prepare / initiate handover / verify return / return to inventory | Member of the case's snapshotted stage team | per §9.1 | — | — |
| Confirm receipt / initiate return / dispute handover | Submitter | per §9.1 | dispute: yes | — |
| Report exception | Submitter or any member of any of the case's stage teams | non-terminal, no open exception | description | — |
| Exception update / resolve direct / propose | Member of the investigate team | per §9.2 | yes | — |
| Decide resolution | Owner, backup owner or active delegate | proposal live | reject: yes | actor ∉ {submitter, reporter, proposer} |
| View request | Submitter; reviewer teams (any snapshot step) pre-owner; owners/delegates from owner stage; stage teams from approval; PA of primary owner from approval (read-only); `org_admin` whose scope includes the request's organisation (D-38); global admin | — | — | — |
| Admin CRUD | `full_admin` (global) or `org_admin` whose scope includes the object's organisation (D-38); organisation create and delete stay `full_admin` | — | — | — |
| Grant / revoke admin role | `full_admin` only; an `org_admin` grant names one or more organisations the grantee is a member of (D-38) | — | — | — |
| Team membership | Global admin, `org_admin` whose scope includes the team's organisation (D-38), or team_admin of that team | — | — | sole team_admin can't self-demote |

---

## §9 Custody state machine and exception resolution

> Resolution routing is drawn as handlers (part 6 §27); §9.3 is extended by §27.5.

### §9.1 Case transitions (from `crates/ffs-server/src/domains/custody.rs`, plus disputes and cancel)

| From | Action | Actor | Event(s) | To |
|---|---|---|---|---|
| awaiting_preparation | prepare | stage `prepare` team | preparation_confirmed | prepared |
| awaiting_preparation | prepare_and_handover (if prepare team == initiate_handover team) | that team | preparation_confirmed, handover_initiated | handover_pending |
| prepared | initiate_handover | `initiate_handover` team | handover_initiated (handoff pending) | handover_pending |
| handover_pending | confirm_receipt | submitter | handover_accepted (handoff accepted) | in_requester_custody |
| handover_pending | dispute_handoff | submitter | exception_reported (handover_disputed; handoff disputed) | exception_open |
| in_requester_custody | initiate_return | submitter | return_initiated (handoff pending) | return_pending |
| return_pending | verify_return | `verify_return` team | return_received | received_by_agency |
| return_pending | verify_and_restock (if verify team == restock team) | that team | return_received, returned_to_inventory | returned_to_inventory |
| return_pending | dispute_handoff | `verify_return` team | exception_reported (return_disputed) | exception_open |
| received_by_agency | return_to_inventory | `return_to_inventory` team | returned_to_inventory | returned_to_inventory (terminal) |
| any non-terminal | report_exception | see §8 | exception_reported | exception_open |
| awaiting_preparation \| prepared | (system) cancel | System (reset force / reject) | case_cancelled | cancelled (terminal) |

`crossed_receipt(c)` is true when the state ∈ {in_requester_custody, return_pending, received_by_agency,
returned_to_inventory}, or when the state is exception_open or resolved_exception and
`state_before_exception` is in that set.

### §9.2 Exception status transitions

| From | To | Actor |
|---|---|---|
| open | investigating | investigate team |
| investigating | awaiting_information | investigate team |
| awaiting_information | investigating | investigate team |
| investigating \| awaiting_information | resolved (recovered) | investigate team (direct) |
| investigating \| awaiting_information | cancelled_as_false_report | investigate team (direct) |
| investigating \| awaiting_information | awaiting_resolution_approval | investigate team (propose) |
| awaiting_resolution_approval | resolved | owner (approve) |
| awaiting_resolution_approval | investigating | owner (reject, notes) |

### §9.3 Resolution effects

| Resolution | Approval | Case effect | Item/asset effect | Join effect |
|---|---|---|---|---|
| recovered | investigator | resume at chosen state ∈ {state_before_exception, and for post-receipt losses `received_by_agency`}; event asset_recovered | — | unchanged |
| false_report | investigator | resume at state_before_exception; event exception_cancelled | — | unchanged |
| replacement | owner | original → resolved_exception (terminal); event replacement_authorized | original item `excluded_at`, reason `replaced`; original asset `retired_reason='lost_replaced'` if missing/damaged, else hold released; **new item** (`replaces_item_id`) + hold on substitute + new case at awaiting_preparation + new child | original excluded; new case live (pending_replacement until its child starts) |
| write_off | owner | → resolved_exception (terminal); event asset_written_off | item excluded (reason `written_off`); asset retired | if pre-receipt: excluded; if post-receipt: still counts as crossed (issuance stands) |
| waiver | owner | resume at state_before_exception; event waiver_granted (deviation in notes) | — | unchanged |

---

## §10 Timers and SLAs

| Timer | Workflow | Armed | Cancelled / re-armed | On fire |
|---|---|---|---|---|
| Approval deadline | parent | at submit, until `activity_end_at` | cancelled on owner approval or on a terminal state; re-armed on approver edit or resubmit with a new window | `auto_expire` if still pre-approval |
| Stage SLA | child | on entering a stage with an agency actor and non-null `due_in_hours` | cancelled on leaving the stage | `flag_overdue` if still in that stage |
| Exception SLA | child | on open | cancelled on resolve | `flag_overdue` on exception |
| Return due | child | at `activity_end_at` while state = in_requester_custody | cancelled on initiate_return | `flag_return_due` |
| Return overdue | child | `activity_end_at + RETURN_GRACE_HOURS` | cancelled on initiate_return | `flag_return_overdue` (no automatic exception) |

Settings:
- `SLA_UNIT_SECONDS` (3600 in production mode, 10 in demo mode)
- `RETURN_GRACE_HOURS` (default 24)
- `NAMESPACE_RETENTION` (7 days)

All of them are read once at worker start and passed as workflow input, so replay stays deterministic.

---

## §11 API contract (axum)

> **Amended (part 4 §24.2).** Adds definition CRUD/validate/simulate/publish/diff/export/import,
> `/plan`, `/variables`, and `POST …/nodes/:nodeInstanceId/act` (replaces step/owner decision, close).

Envelope:
- Success: `{"data": …}`.
- Error: `{"error": {"code": "…", "message": "…", "fields": {"path": "msg"}?}}`.

Status codes:

| Status | Meaning |
|---|---|
| 401 | No session |
| 403 | Visible but not eligible |
| 404 | Hidden or not found; malformed UUID |
| 409 | State conflict (`STALE_STATE`, `STEP_ALREADY_DECIDED`, `ASSET_IN_USE`, `CUSTODY_IN_PROGRESS`, `INVALID_CUSTODY_STATE`, `SYSTEM_WORKFLOW_INVALID`, `IDEMPOTENCY_TOKEN_USED`) |
| 422 | Validation failure |
| 500 | `AUDIT_WRITE_FAILED` |
| 503 | `ENGINE_UNAVAILABLE` |

Every mutating route requires `X-Idempotency-Token`, which is also used as the Temporal update ID.

| Method & path | Temporal call | Notes |
|---|---|---|
| `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` | — | demo persona login when `DEMO_MODE=1` (`POST /api/auth/demo-login {userId}`) |
| `GET /api/requests?scope=mine\|action\|all&status=` | — (DB) | role-scoped (§8) |
| `POST /api/requests` | UpdateWithStart `submit` | 201 |
| `GET /api/requests/:id` | — (DB) | 404 rule |
| `GET /api/requests/:id/actions` | — (DB + projection) | the actions this viewer may take now |
| `GET /api/requests/:id/timeline` | — (DB) | audit + steps + custody events + exception updates |
| `GET /api/requests/:id/engine` | Query `projection` | plus `consistent` diff and Temporal UI link |
| `POST /api/requests/:id/step-decision` | Update `step_decision` | |
| `POST /api/requests/:id/owner-decision` | Update `owner_decision` | |
| `PUT /api/requests/:id/approver-edit` | Update `approver_edit` | |
| `PUT /api/requests/:id/resubmit` | Update `resubmit` | |
| `POST /api/requests/:id/close` | Update `close` | |
| `POST /api/requests/:id/admin-reset` | Update `admin_reset` | |
| `POST /api/cases/:caseId/actions/:action` | child Update `custody_action` | |
| `POST /api/cases/:caseId/dispute` | child Update `dispute_handoff` | |
| `POST /api/evidence` (multipart) | — | stores a pending file row and returns its id; magic bytes checked |
| `POST /api/cases/:caseId/exceptions` | child Update `report_exception` | |
| `POST /api/exceptions/:id/update` · `/resolve` · `/propose` · `/decide` | child Updates | |
| `POST /api/ops/reconcile` (global admin) | Describe + Query per open request | read-only report |
| Admin: `/api/admin/{organizations,clusters,teams,memberships,users,roles,grants,systems,systems/:id/workflow,systems/:id/custody-stages,systems/:id/owners,systems/:id/delegates,assets}` | — | CRUD, soft delete. Grants: `POST /api/admin/grants {userId, roleKey, organizationIds[]}` (one row per organisation) and `POST /api/admin/grants/:id/revoke` (D-38) |
| `GET /api/requests/:id/issuance` | — (DB view) | per-item issue and return rows plus the `request_issuance_summary` counts |
| `GET /api/reports/issuance-returns?from&to&systemId&outstanding=` | — (DB view) | organisation-scoped; owners, investigate team, `org_admin` (organisations in their scope, D-38) and global admins; CSV with `Accept: text/csv` |
| `GET /api/assets/:id/history` | — (DB view) | every issue and return of the asset across requests |
| `GET /api/requests/:id/history-archive` | — (DB) | global admin, or `org_admin` whose scope includes the request's organisation (D-38); downloads the archived Temporal history (D-18) with its SHA-256 |
| `GET /api/health` | `DescribeNamespace` | `{db, temporal, worker, archive}` |

---

## §12 UI specification (Next.js)

> **Amended (part 4 §24.1, R8).** The graph editor (`/admin/systems/[id]/workflow/edit`) replaces the
> chain builder and custody config. The request graph renders `/plan` (Variables panel, run selector).

| Route | Content |
|---|---|
| `/login` | Email/password. In demo mode, a persona picker grouped by organisation → role |
| `/` | Dashboard: "Needs my action" (with overdue badges), "My requests", PA panel |
| `/requests/new` | System picker (eligible systems only); justification, SR number, window (demo presets +1/+3 min), multi-select assets (in-use assets disabled) |
| `/requests/[id]` | Header (number, status pill, organisation/system); **graph**; action panel; Record vs Engine panels; **Issuance and returns** panel (per item: issued to, issued at, returned at, verified by, restocked at, outcome; plus the summary counts); timeline |
| `/reports/issuance-returns` | Filter by date range, system and "outstanding only"; table and CSV export; overdue rows first |
| `/assets/[id]` | Asset detail with its full issue and return history |
| `/admin/*` | Organisations, clusters/teams (level up/down), memberships, users (tombstone), roles/grants, systems (chain builder with preview graph, custody stage config, owners, delegates, PA), assets |

**Graph generation.** The graph is never hard-coded; it is built from the request's snapshot and its
cases.
- Column 1: `Submit (step 1)`.
- Columns 2…N: `Step k · {cluster} L{level}`.
- Then `Owner`, `Fork`, one **lane per case** (prepare → handover → receipt → return → verify →
  restock, plus an `Exception` side node when one is open), `Join (issue)` and `Close`.
- Side terminals: `Returned` (dashed edge back to Submit), `Rejected`, `Expired`, `Closed –
  unfulfilled`.
- Replacement cases render as a new lane linked from the replaced lane with a "replaced by" edge.

Edge labels are the actions and conditions: "approve", "return (notes)", "reject (notes)", "owner
approve", "all live items received", "timer: window elapsed".

| Node state | Style |
|---|---|
| pending | grey border |
| active | blue border + glow |
| done | green |
| returned | amber |
| rejected/expired | red |
| exception open | red dashed + ⚠ |
| overdue | red "Overdue {n}h" badge |
| excluded (written-off/replaced) | strikethrough, grey fill |

The Join node shows `join_trace`.

**Refresh.** Poll `/engine` and `/actions` every 2 s while the page is visible, with SSE as an
optional upgrade. Stop polling when the workflow has completed.

**Panels.**
- **Record (Postgres):** fields, items, remarks, audit.
- **Engine (Temporal):** current wait, armed timers with countdowns, children and their states,
  history length, run ID chain, and a "Open in Temporal UI" link
  (`http://localhost:8233/namespaces/breakglass-mvp/workflows/{id}`).

---

**Continued in** `addendum-219-temporal-breakglass-mvp-part-2-stories.md` (§14 user stories and acceptance criteria), `addendum-219-temporal-breakglass-mvp-part-3-verification-and-prompt.md` (§13 seed data and §15–§19: demo scenarios, Definition of Done, out of scope, open points, implementation prompt), `addendum-219-temporal-breakglass-mvp-part-4-workflow-definition-and-editor.md` (§20–§26), `addendum-219-temporal-breakglass-mvp-part-5-editor-stories.md` (ST-15–ST-24) and `addendum-219-temporal-breakglass-mvp-appendix-a-illustrative-example.md`.
