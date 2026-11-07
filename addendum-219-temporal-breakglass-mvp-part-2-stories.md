# Addendum 219 (part 2) — Subtasks, user stories and acceptance criteria

**Part 1:** `addendum-219-temporal-breakglass-mvp.md` (§0–§13). **Part 3:** `addendum-219-temporal-breakglass-mvp-part-3-verification-and-prompt.md` (§15–§19). Parts 9 and 10 (§35–§40, ST-31 and ST-32) elaborate the exception and cloning stories. All parts are normative; part 1 §0 defines notation (Given/When/Then, "no side effect", terminology).

## §14 Subtasks, user stories and acceptance criteria

> **Amended 2026-11-08.** ST-15–ST-24 (editor, definitions, actualisation) are in part 5. Under
> part 4 §26, `step_decision{expectedOrdinal…}` and `owner_decision` below read as the `act` update
> with `expectedRevision` on the named node instance; assertions are otherwise unchanged. AC-3.2.x
> and AC-3.3.x are **retired** and replaced as noted in ST-3. ST-25–ST-30 (exception handlers,
> cloning, coverage, module boundaries) are in part 7.

Every negative AC also asserts **no side effect** (see §0). AC IDs are stable: `AC-{subtask}.{story}.{n}`.

### ST-0 Environment and toolchain

**US-0.1** As a developer, I want a pinned Rust 1.99.1 toolchain from `wshyang/psd-rust-artifacts`, so builds match the port-parity audit.
- AC-0.1.1
  - G: a machine without Rust 1.99.1.
  - W: `scripts/bootstrap-toolchain.sh`.
  - T: it downloads `toolchain-1.99.1.tar.xz.part-aa` and `-ab` into a fresh empty directory,
    concatenates them, `sha256sum -c toolchain-1.99.1.sha256` passes, it extracts into
    `$HOME/.rust-1.99.1`, and `rustc --version` reports 1.99.1.
- AC-0.1.2
  - G: a corrupted part.
  - W: bootstrap.
  - T: the checksum fails, it exits non-zero, and nothing is extracted.
- AC-0.1.3
  - G: already installed with a matching hash.
  - W: re-run.
  - T: no download, exits 0.
- AC-0.1.4: `rust-toolchain.toml` pins `1.99.1`. The MVP has its own `Cargo.lock`, and
  `.cargo/config.toml` points at its own `vendor/`. After one `cargo vendor`,
  `cargo build --offline --locked` succeeds with the network disabled.

**US-0.2** As a developer, I want one command to start everything.
- AC-0.2.1
  - W: `make demo-up`.
  - T: Temporal on :7233 (UI :8233, namespace `breakglass-mvp`), Postgres on :5433 (migrated and
    seeded), the worker polling `breakglass`, the API on :3001 and Next.js on :3000, and
    `GET /api/health` returns 200 `{db:"ok",temporal:"ok",worker:"polling"}`.
- AC-0.2.2: search attributes `OrgCode`, `SystemCode` and `RequestStatus` are registered
  idempotently. A second `make demo-up` doesn't fail.
- AC-0.2.3
  - G: Temporal stopped.
  - W: `GET /api/health`.
  - T: 503 with `temporal:"down"`, and every page shows the banner "Workflow engine unavailable —
    actions are disabled".
- AC-0.2.4: `make demo-reset` drops and reseeds the app DB **and** deletes `.temporal/dev.db`,
  printing a warning that both stores are wiped.

**US-0.3** As a maintainer, I want the MVP isolated from production code.
- AC-0.3.1: the MVP isn't a member of the root workspace. Running root `cargo build --offline --locked`
  and `deno task check` before and after adding the MVP gives identical results.

### ST-1 Data ownership between Temporal and the app

**US-1.1** As an auditor, I want every remark and decision recorded in the app DB together with its audit row.
- AC-1.1.1
  - G: request at step 3.
  - W: Claire returns it with "SR mismatch".
  - T: in one transaction, `request_approval_steps(ordinal 3).decision_notes='SR mismatch'`, the
    request becomes `returned_to_vendor`, and `audit_log(action='request.step_returned', actor=Claire, details.notes='SR mismatch')`.
    All three rows share one `op_ref`.
- AC-1.1.2
  - G: a closed request whose workflow completed.
  - W: `temporal workflow delete --workflow-id bg-AUR-…`.
  - T: `/requests/[id]` still renders all fields, remarks, the timeline and audit. The Engine panel
    shows "Live history purged; archived copy available" with the archive's SHA-256. No error.
- AC-1.1.3
  - G: a fault flag makes the audit insert raise.
  - W: any decision.
  - T: the business rows are rolled back, the activity retries and then fails, the update returns
    500 `AUDIT_WRITE_FAILED`, and the projection is unchanged.

**US-1.2** As an engineer, I want activities to be idempotent.
- AC-1.2.1: every lifecycle activity writes `workflow_ops(op_ref)` in its transaction. Replaying the
  same op_ref returns the stored result with zero new rows (unit test calls the activity twice).
- AC-1.2.2
  - G: fault flag `CRASH_AFTER_COMMIT=record_step_decision`.
  - W: approve.
  - T: the worker exits after commit. After restart the activity retries and finds the op_ref. There
    is exactly one step update, one audit row and one history `ActivityTaskCompleted`, and the
    request is at the next ordinal.
- AC-1.2.3
  - G: the same `X-Idempotency-Token` sent twice concurrently.
  - T: both responses carry the identical result, and there is one audit row.

**US-1.3** As an operator, I want to prove the engine and the record agree.
- AC-1.3.1: `GET /api/requests/:id/engine` returns `{projection, db:{status, ordinal, cases}, consistent}`.
  Directly altering the DB status in a test makes `consistent:false` and lists the differing field.
- AC-1.3.2: `POST /api/ops/reconcile` returns open requests whose workflow is missing, completed
  while the DB is non-terminal, or diverged. It never changes anything (asserted: no row
  `updated_at` changes).

**US-1.4** As a demo presenter, I want to show which data lives where.
- AC-1.4.1: the request page shows the "Record (Postgres)" and "Engine (Temporal)" panels as listed
  in §12. Every remark shown comes from `/timeline` (the network log shows no remark text in the
  `/engine` response).

**US-1.5** As an auditor, I want every record kept for 7 years, even though Temporal deletes its own
history after 7 days.
- AC-1.5.1
  - G: a request closed and its workflow completed.
  - W: the history archiver runs.
  - T: within 2 minutes there is exactly one `workflow_history_archive` row per run of that workflow
    (resubmits and resets add runs), each with a SHA-256 that matches a fresh download, and the
    event count matches `temporal workflow describe`. Running the archiver again adds no rows.
- AC-1.5.2 (reconstruction)
  - G: S1–S4 completed.
  - W: the Temporal database file is deleted and Temporal is restarted empty.
  - T: every request page, timeline, audit export, issuance and returns panel, report and asset history
    renders the same data as before (snapshot comparison), and each archived history still downloads
    and passes its SHA-256 check.
- AC-1.5.3: `UPDATE` or `DELETE` on `audit_log` and `workflow_history_archive` as the app DB role
  fails with "permission denied". The API has no route that deletes requests, audit rows, custody
  events or evidence.
- AC-1.5.4
  - G: the archiver is failing (fault flag).
  - W: `GET /api/health`.
  - T: `archive: "lagging"` with the number of closed workflows not yet archived, and the dashboard
    shows an ops banner to global admins.
- AC-1.5.5: replaying an archived history against the current worker code succeeds (it feeds the
  replay tests in AC-13.2.1).

### ST-2 Organisation tenancy, identity, roles and personas

**US-2.1** As a global admin, I want to create organisations whose data is isolated.
- AC-2.1.1: create `{shortCode:"AUR", name:"Org Aurora"}`. The short code is uppercased first; a duplicate live short
  code or case-insensitive name → 422 `fields.shortCode` / `fields.name`. A short code that fails `^[A-Z0-9]{1,10}$`
  → 422. Deleting an organisation is a soft delete, and is blocked with 409 while it has
  live systems.
- AC-2.1.2
  - G: Amanda (org_admin @AUR).
  - W: she lists systems, requests, users or audit.
  - T: only AUR rows are returned.
  - W: she GETs any SEN object by a valid ID.
  - T: 404 with the same body as a random UUID.
- AC-2.1.3: every query in the API and activities that reads organisation-owned tables carries an
  `organization_id` predicate. A bite test that removes the predicate from the request list makes
  AC-2.1.2 fail.

**US-2.2** As an organisation admin, I want clusters and tiered teams.
- AC-2.2.1: a second live team at the same `(cluster, level)` → 409. Level 0 or 6 → 422. Changing a
  cluster's type while it has teams → 409.
- AC-2.2.2: adding Bob (Alpha L1) to Alpha L2 → 409 "User already at L1 in Alpha Pte".
- AC-2.2.3: adding a vendor-affiliated user to an agency or pa cluster team → 422, and the reverse
  → 422.

**US-2.3** As a team admin (Alice), I want to manage my team's membership.
- AC-2.3.1: Alice can add, remove, promote and demote only in Alpha Engineers L1. Attempts on Alpha
  Leads L2 → 403.
- AC-2.3.2: Alice, the sole team_admin, demoting or removing herself → 409. The first member added to
  an empty team becomes team_admin automatically.
- AC-2.3.3: Alice renaming or deleting the team, or changing its level → 403.

**US-2.4** As a global admin, I want to tombstone users while preserving history.
- AC-2.4.1
  - G: Bob tombstoned.
  - T: Bob's login → 401 generic, his email can be re-registered as a new user ID, and past timeline
    entries still show "Bob Lim (deactivated)".
- AC-2.4.2: tombstoning yourself, the last active global admin, or the System or protected users →
  409.
- AC-2.4.3: auto-cancel of Bob's open requests is covered by ST-7 (US-7.3).

**US-2.5** As a demo presenter, I want instant persona switching.
- AC-2.5.1: with `DEMO_MODE=1`, `/login` lists the personas grouped by organisation and role (for example
  "AUR · Vendor L1 · Bob Lim"). One click → session → dashboard.
- AC-2.5.2: with `DEMO_MODE` unset, `POST /api/auth/demo-login` → 404. Password login with 5 failures
  locks the account for 30 min (the 6th attempt returns 429 even with the correct password).
- AC-2.5.3: the cookie is `HttpOnly; SameSite=Lax; Path=/`. After 15 min of inactivity any API call
  returns 401, and the UI redirects to `/login?next=…`.

**US-2.6** As a global admin, I want capability-based admin roles.
- AC-2.6.1: `full_admin` holds every capability. `org_admin` holds {org_structure, systems_assets,
  users_org, request_admin_reset}, and only within the organisations named by its grant rows (its
  admin scope, D-38); it never applies to every organisation. Granting to a vendor, pending, system or
  protected user → 422.
- AC-2.6.2: after a grant is revoked, the user's next admin call → 403 (404 for an object of the
  revoked organisation while another grant remains). No cache outlives a request.
- AC-2.6.3 (grant shape and membership, D-38)
  - W: Greg posts `org_admin` with `organizationIds: []`, or `full_admin` with a non-empty
    `organizationIds`.
  - T: 422 `GRANT_SCOPE_INVALID`, no row written.
  - W: Greg grants Amanda (member of AUR only) `org_admin` for `[AUR, SEN]`.
  - T: 422 `GRANT_NOT_MEMBER` naming SEN, and no row is written for AUR either (one transaction).
  - W: a direct INSERT of an `org_admin` row with `organization_id NULL`, or for an organisation the
    user is not a member of.
  - T: the trigger rejects it (bite check: dropping the trigger makes this test fail).
  - W: Greg grants Amanda `org_admin` for `[AUR]` again.
  - T: 200, still exactly one live AUR row.
- AC-2.6.4 (multi-organisation scope)
  - G: Mona (org_admin @AUR and @SEN); Greg then creates organisation TST with system TSTSYS.
  - W: Mona lists systems, users and requests.
  - T: AUR and SEN rows only. Mona GETs TST or TSTSYS by ID → 404 with the same body as a random UUID.
  - T: Mona can create a cluster in AUR and in SEN; creating one in TST → 404. Mona creating or deleting an organisation → 403; renaming SEN → 200;
    renaming TST → 404.
- AC-2.6.5 (per-organisation revoke)
  - W: Greg revokes Mona's SEN grant.
  - T: her AUR grant stays live; her next SEN list omits SEN rows and a SEN object by ID → 404; AUR
    calls still succeed.
- AC-2.6.6 (membership removal)
  - W: Greg removes Mona's SEN membership.
  - T: in the same transaction her SEN `org_admin` grant gets `revoked_at`, `revocation_reason =
    'membership_removed'`, and an `admin_grant.revoked` audit row; her AUR grant is unchanged.
- AC-2.6.7 (identity-subset rule)
  - T: Amanda tombstoning Mona (memberships AUR and SEN, not ⊆ {AUR}) → 403 with
    `ADMIN_SCOPE_INSUFFICIENT`. Mona tombstoning Bob (AUR only) → 200.

### ST-3 System registry, approval chain, custody configuration, assets, delegation

**US-3.1** As an organisation admin, I want to register systems.
- AC-3.1.1: `short_code` must be unique per organisation (`^[A-Z0-9]{2,10}$`), otherwise 422. The primary
  and backup owners must be distinct, active, agency-affiliated members of the same organisation (422).
- AC-3.1.2: the PA team must be in a `pa` cluster of the same organisation (422). The mapping is per owner
  row.

**US-3.2** *(RETIRED 2026-11-08 — superseded by ST-15, ST-16 and ST-17 in part 5; the chain is now
drawn in the workflow editor. Kept for traceability; do not map tests to these IDs.)* As an organisation admin, I want to build the approval chain.
- AC-3.2.1: saving a chain checks each rule below. Every violation → 422 with `fields["steps[i]…"]`.
  - at least one step;
  - ordinals contiguous from 1;
  - no `pa` cluster;
  - vendor steps reference the system's vendor clusters;
  - agency steps reference agency clusters of the same organisation;
  - a live team exists at each `(cluster, level)`;
  - no step after 1 repeats step 1's `(cluster, level)`.
- AC-3.2.2
  - G: an HRIS request at step 3 (Ops L1).
  - W: the admin removes Ops L1 from the chain.
  - T: the request's snapshot and graph still show Step 3 Ops L1 and await it, and new HRIS
    submissions get the new chain.
- AC-3.2.3: the chain builder shows a live preview graph built by the same layout function the
  request page uses.

**US-3.3** *(RETIRED 2026-11-08 — superseded by the `custody_stage` nodes of the `subject=item` lane template the system embeds per item, V-17,
V-24, V-25 and ST-20 in part 5.)* As an organisation admin, I want to configure the custody stages.
- AC-3.3.1: until all 5 stages are configured, submitting to the system → 409 "Custody not configured
  for this system". Each stage team must be in an agency cluster of the same organisation (422).
  `due_in_hours` must be NULL or 1..720 (422).
- AC-3.3.2
  - G: a case forked with prepare due 4 h.
  - W: the admin changes it to 1 h.
  - T: the existing case's SLA timer is unchanged (it uses the snapshot), and new forks use 1 h.

**US-3.4** As an organisation admin, I want to manage manual assets.
- AC-3.4.1: `(system, hostname, credential_name)` is unique among live assets (409).
  `credential_type` can't change after creation (422). Retired assets don't appear in pickers, and
  submitting with one → 409 `ASSET_UNAVAILABLE`.
- AC-3.4.2: an asset with a live hold shows "In use by BG-…" and can't be selected (409
  `ASSET_IN_USE`).

**US-3.5** As an owner, I want to delegate approval for a period.
- AC-3.5.1: the delegate must be an active agency user of the same organisation, and `active_until >
  active_from` (422). Outside the window, the delegate's owner decision → 403 `DELEGATION_INACTIVE`.
  Inside the window it succeeds and is audited with `acting_as='delegate'`.

### ST-4 Submission and workflow start

**US-4.1** As a vendor L1 engineer (Bob), I want to submit an HRIS request with several assets.
- AC-4.1.1: the API validates the body before any engine call, otherwise 422 with field errors:
  - justification 1–2000 (trimmed);
  - SR number 1–100;
  - start < end and end > now (D-11);
  - 1–10 distinct asset IDs that are live, not retired, not held, manual, and belong to that system;
  - each `hostname_description` ≤ 200.
- AC-4.1.2: if Bob can't see HRIS (another organisation) → 404. If he can see it but isn't in the
  ordinal-1 team → 403 `NOT_SUBMITTER`.
- AC-4.1.3: on success, workflow `bg-AUR-{uuid}` exists with search attributes `OrgCode=AUR`,
  `SystemCode=HRIS` and `RequestStatus=submitted`. The DB contains:
  - the request `BG-HRIS-{SGT yyyymmdd}-{NNN}` (sequence per system per SGT day, zero-padded to 3,
    widening past 999);
  - snapshot step 1 `submitted` decided by Bob and steps 2–3 `pending` with
    cluster/team_type/level copied;
  - the items;
  - holds;
  - audit `request.submitted`.
  
  Status is `submitted`, at ordinal 2.
- AC-4.1.4: if the system chain has exactly one step, the request starts at `under_review`, ordinal 0
  (owner stage).
- AC-4.1.5: the response is 201 within 5 s (p95, local). The UI navigates to the detail page and the
  graph shows Submit green and Step 2 active, with the approval-deadline timer listed in the Engine
  panel.

**US-4.2** As a submitter, I want retries and races handled.
- AC-4.2.1: double-submitting with the same token → one request, one workflow, the same 201 body.
- AC-4.2.2
  - G: Bob and Alice submit with different tokens, both selecting SP-01, within 50 ms.
  - T: exactly one succeeds. The other gets 409 `ASSET_IN_USE` (the unique hold index is violated
    inside `persist_submission`, a non-retryable error), and the losing workflow completes with
    result `submission_failed` and writes no request row.

**US-4.3** As a submitter, I want clear behaviour on infrastructure failure.
- AC-4.3.1
  - G: Temporal down.
  - W: submit.
  - T: 503 `ENGINE_UNAVAILABLE`, zero DB rows, and the form keeps its values.
- AC-4.3.2
  - G: the DB is down for 20 s during `persist_submission`.
  - T: the activity retries, the API returns 503 after a 10 s wait, the request eventually appears
    exactly once, and the UI "My requests" list shows it after refresh.

**US-4.4** As an attacker from another organisation, I can't create requests on foreign systems.
- AC-4.4.1: Gina (SEN) POSTs with HRIS `systemId` → 404. `temporal workflow list --query
  "SystemCode='HRIS'"` count is unchanged.

### ST-5 Approval chain

**US-5.1** As a step reviewer (Aaron, Alpha L2), I want to approve with optional remarks.
- AC-5.1.1
  - W: `step_decision{expectedOrdinal:2, decision:"approve", notes:"ok"}`.
  - T: step 2 is `approved` (decided_by Aaron, notes "ok"), the request is `under_review` at ordinal
    3, audit `request.step_approved`, and search attribute `RequestStatus=under_review`.
- AC-5.1.2: approving the last snapshot step sets ordinal 0. The graph highlights Owner, and the
  owners' "Needs my action" list now includes the request.

**US-5.2** As a reviewer, I want to return or reject with mandatory remarks.
- AC-5.2.1: return or reject with notes missing, blank or longer than 2000 chars → 422 (API, before
  the engine). A direct engine update with blank notes → the validator rejects it (defence in depth).
- AC-5.2.2: return → `returned_to_vendor`, ordinal kept, `return_count`+1 (owner returns count too), audit
  `request.step_returned`. The graph shows the Returned node amber, with a dashed edge back to
  Submit. The approval-deadline timer stays armed (D-10).
- AC-5.2.3: reject → `rejected` (terminal), holds released, audit `request.step_rejected`, timers
  cancelled, and the workflow completes.

**US-5.3** As a later-step reviewer, I want to override a stalled earlier step.
- AC-5.3.1
  - G: FINDB request at step 2 (Beta L2).
  - W: Frank (Ops L2, step 4) approves with `override:true`.
  - T: step 2 `approved` with `decided_via_override=true`, audit `request.step_override_approved`,
    ordinal 3.
- AC-5.3.2: Claire (step 3) overriding step 3 or Brian (step 2) using override on step 2 is treated
  as a normal decision. Betty (step 1 only) using override → 403 `NOT_OVERRIDE_ELIGIBLE`.

**US-5.4** As the system, I prevent self-approval at decision time (D-6).
- AC-5.4.1
  - G: Ivan (Ops L1, owner of OPSTOOL) submits on OPSTOOL, Frank approves step 2, and the request
    reaches the owner stage.
  - W: Ivan owner-approves.
  - T: 403 `SELF_APPROVAL_FORBIDDEN`. Carol (backup) can then approve.
- AC-5.4.2: the same check applies to override decisions and to `decide_resolution` (US-9.4).
- AC-5.4.3 (bite): disabling the validator check **and** the activity check makes AC-5.4.1 fail.
  Disabling only one still blocks the decision (proves both layers).

**US-5.5** As two reviewers acting simultaneously, exactly one decision wins.
- AC-5.5.1
  - G: Claire and Emma (both Ops L1).
  - W: both approve step 3 within 50 ms.
  - T: one gets 200 and the other 409 `STEP_ALREADY_DECIDED`. There is one step decision row, one
    audit row, and the ordinal advanced by exactly 1.
- AC-5.5.2
  - G: Claire's page shows ordinal 3, but the request is now at 0.
  - W: she approves with `expectedOrdinal:3`.
  - T: 409 `STALE_STATE`, and the UI shows "This request has moved on" and refetches.

**US-5.6** As the current-step reviewer, I want to correct the request content.
- AC-5.6.1: `approver_edit` with full validation (same as AC-4.1.1, no silent skipping) → content and
  items replaced, holds adjusted atomically, audit `request.edited_by_approver` with `old` and `new`
  values. Using override → 403.
- AC-5.6.2: changing `activity_end_at` cancels the old approval-deadline timer and arms a new one.
  The Engine panel shows the new fire time.
- AC-5.6.3: an edit that tries to change `systemId` or `submittedBy` → 422.

**US-5.7** As an ineligible user, I am blocked.
- AC-5.7.1: Betty (Beta L1, AUR) deciding an HRIS step → 403. Gina (SEN) → 404. A logged-out user →
  401.
- AC-5.7.2
  - G: Aaron removed from Alpha L2 after the API pre-check but before the activity (test hook delay).
  - T: the activity fails non-retryably with `NOT_STEP_REVIEWER`, the update returns 403, and there
    are no row changes.

### ST-6 Owner decision and PA visibility

**US-6.1** As the primary owner (Carol), I want to approve, return or reject at the owner stage.
- AC-6.1.1: the precondition is `under_review` with ordinal 0, enforced in the validator **and** in
  the SQL WHERE. In any other state → 409 `STALE_STATE`. Return and reject need notes (422).
- AC-6.1.2: approve → `approved` with owner_decision_by/at/notes, and audit `request.owner_approved`
  with details `{pa_team_id}`. Then `instantiate_custody` creates one case per item, and one child
  workflow per case exists, visible as linked children in the Temporal UI. The approval-deadline
  timer is cancelled.
- AC-6.1.3: Dan (backup) can do the same. Emma (delegate) can do the same inside her window, audited
  `acting_as='delegate'`; outside the window → 403.
- AC-6.1.4: before the owner stage, `GET /requests/:id` as Carol → 404. From the owner stage on → 200.
- AC-6.1.5: owner return → `returned_to_vendor` with `owner_decision_notes`. Owner reject →
  `rejected` and the workflow completes. Both are audited semantically.

**US-6.2** As PA (Priya), I want visibility of my principal's approved requests.
- AC-6.2.1: Priya's dashboard lists AUR requests at or past `approved` whose primary owner's PA
  mapping is Owner Support. Every POST by Priya → 403. Pre-approval requests → 404.

### ST-7 Return/resubmit, admin reset, tombstone auto-cancel, auto-expire

**US-7.1** As the submitter, I want to revise and resubmit.
- AC-7.1.1: only Bob, and only from `returned_to_vendor` (others → 403; other states → 409). Full
  validation. If the original window has elapsed, a new future window is required (422).
- AC-7.1.2: the snapshot is rebuilt from the **current** chain, the owner decision is cleared, status
  becomes `submitted` at ordinal 2, `resubmit_count`+1 (`return_count` unchanged), and audit `request.resubmitted`. The workflow
  **continues-as-new**: the Temporal UI shows a new run ID for the same workflow ID, and the Engine
  panel's run chain lists both. The new run's first activity is `load_projection`.
- AC-7.1.3
  - G: the current chain references a team that has since been deleted.
  - W: resubmit.
  - T: 409 `SYSTEM_WORKFLOW_INVALID` "This system's approval chain is invalid. Contact your organisation
    admin." The request stays `returned_to_vendor`.

**US-7.2** As an admin with `request_admin_reset`, I want to reset stuck requests.
- AC-7.2.1: from any status except `closed` (closed → 409): steps above 1 go back to `pending` with
  decided fields cleared, the owner decision is cleared, status becomes `returned_to_vendor`, audit
  `request.admin_reset{previous_status, notes}`, and continue-as-new.
- AC-7.2.2
  - G: approved, with SP-01 `in_requester_custody` and HT-01 `prepared`.
  - W: reset without force.
  - T: 409 `CUSTODY_IN_PROGRESS` listing SP-01.
  - W: force with notes.
  - T: HT-01's child is cancelled (case `cancelled`, hold released, audit `custody.case_cancelled`),
    SP-01's child keeps running, the request is `returned_to_vendor`, and the page shows "1 item
    still in custody from a previous approval".
- AC-7.2.3: Amanda resetting a SEN request → 404. A user without the capability → 403.

**US-7.3** As the system, I auto-cancel open requests when a submitter is tombstoned.
- AC-7.3.1
  - G: Bob has requests in `submitted`, `returned_to_vendor` and `approved`.
  - W: Greg tombstones Bob.
  - T: the API (after committing the tombstone) signals `submitter_tombstoned` to the first two
    workflows (found by DB query). Each runs `auto_cancel` → `rejected`, with
    `decision_reason='submitter_tombstoned'`, the System actor and audit `request.auto_cancelled`.
    The approved request is untouched.
- AC-7.3.2: signal delivery that fails because the engine is down is retried by the API's outbox
  (`pending_signals` table, drained every 5 s). A duplicate signal to an already-rejected workflow is
  a no-op.

**US-7.4** As the system, I auto-expire requests whose activity window passes before approval.
- AC-7.4.1
  - G: demo mode; a request at step 3 with `activity_end_at = now+60 s`.
  - W: 60 s elapse.
  - T: the request becomes `rejected` with `decision_reason='activity_window_elapsed'`, actor
    System, audit `request.auto_expired` and holds released. The graph shows the Expired node red
    and the workflow completes.
- AC-7.4.2
  - G: Claire's approval and the timer race.
  - T: the workflow processes them serially. If the timer is first, Claire gets 409 `STALE_STATE`.
    If Claire is first, the timer handler sees a non-pre-approval status (or the guarded UPDATE
    matches 0 rows) and does nothing.
- AC-7.4.3: after owner approval the deadline timer is no longer listed. After an approver edit
  extends the window, the timer shows the new time.

### ST-8 Fulfilment fork/join: custody happy path

**US-8.1** As the system, on owner approval I fork one custody child per item.
- AC-8.1.1: `instantiate_custody` creates one run, one case per live item (`awaiting_preparation`)
  and five stage-snapshot rows per case, audit `custody.run_started{case_count}`. Children start with
  `ParentClosePolicy=ABANDON`, and after parent continue-as-new they are re-attached by stored
  `child_workflow_id`.
- AC-8.1.2: the graph shows one lane per item with its asset label (for example "SP-01 · Secure
  phone").

**US-8.2** As the Asset Store team (Grace), I want to prepare and hand over.
- AC-8.2.1: on HRIS (the same team for prepare and initiate_handover) the action panel offers only
  "Prepare and initiate handover". Executing it writes two events in one transaction, the handoff is
  `pending` and the case is `handover_pending`.
- AC-8.2.2: on FINDB (different teams) Grace sees "Prepare" only. After it, Claire's team sees
  "Initiate handover".
- AC-8.2.3: acting on SP-01 never changes HT-01 (per-item isolation test).
- AC-8.2.4: the prepare SLA timer starts on fork and is cancelled on leaving `awaiting_preparation`.

**US-8.3** As the requester (Bob), I confirm receipt per item, and the join issues the request.
- AC-8.3.1: only Bob can `confirm_receipt` (others → 403). The handoff becomes `accepted` and the
  case `in_requester_custody`, and the child signals `crossed_receipt`.
- AC-8.3.2: with 3 live items, after the 1st and 2nd receipts the status is still `approved` and the
  Join node shows `live=3 crossed=2 … -> Wait`. After the 3rd it shows `-> Issue`, then
  `mark_issued` sets `credential_issued` with audit `request.credential_issued{route:'custody'}`.
- AC-8.3.3: two receipts within 10 ms of each other still produce exactly one `mark_issued` row and
  audit row (guarded UPDATE plus op_ref).

**US-8.4** As requester and agency, we complete the return.
- AC-8.4.1: Bob `initiate_return` → `return_pending`. Claire `verify_return` → `received_by_agency`.
  Grace `return_to_inventory` → `returned_to_inventory` (terminal), hold released, child completed.
- AC-8.4.2: when the verify and restock teams are the same, "Verify return and return to inventory"
  is offered and writes two events.
- AC-8.4.3: actions in the wrong state → 409 `INVALID_CUSTODY_STATE`. The wrong actor → 403.

**US-8.5** As requester or reviewer, I can close after issuance even with returns outstanding.
- AC-8.5.1: close from `credential_issued` with optional notes ≤2000 → `closed`, audit
  `request.closed`. Close in any other state → 409.
- AC-8.5.2: after close with 2 cases still out, the parent workflow is still **Running**, the list
  shows "Closed · 2 items outstanding", and the Engine panel shows "Waiting for custody (2)". When
  the last case is terminal, the parent completes.

**US-8.6** As an owner or auditor, I want to see what was issued and what came back, per item and over
time.
- AC-8.6.1
  - G: S1 with 3 items, after all 3 receipts.
  - T: the request's Issuance and returns panel lists each item with handed over by (Grace), issued to
    (Bob), issued at, and an empty return. The summary shows `issued=3 outstanding=3 returned=0`.
- AC-8.6.2
  - W: SP-01 is returned, verified by Claire and restocked by Grace.
  - T: SP-01 shows `returned_at`, `return_verified_by=Claire`, `restocked_by=Grace` and outcome
    `returned`. The summary shows `issued=3 outstanding=2 returned=1`. Every timestamp equals the
    `created_at` of its matching `custody_events` row.
- AC-8.6.3 (replacement and write-off)
  - G: S3 (HT-01 replaced by HT-02) and S4 (SP-01 written off after issuance).
  - T: in S3, HT-01 shows outcome `replaced` and was never issued, while HT-02 shows its own issuance,
    so the request has two issuance rows for the same credential slot. In S4, SP-01 shows outcome
    `written_off` with its issuance kept, and it counts in `items_written_off`, not in
    `items_outstanding`.
- AC-8.6.4: `/assets/HT-02` lists every issue and return of HT-02 across requests in time order.
  Issuing it on a second request adds a second row; the first row is unchanged.
- AC-8.6.5: `/reports/issuance-returns?outstanding=true` lists exactly the items issued and not yet
  returned, written off or replaced, overdue ones first. Amanda (org_admin @AUR) sees only AUR rows;
  the CSV export has the same rows as the table. A SEN user gets an empty AUR result (404 on an AUR
  `systemId`).
- AC-8.6.7: Bob (vendor) and Priya (PA) calling the report → 403.
- AC-8.6.6: the D-19 timestamp columns can't be changed once set. A direct UPDATE that changes or
  clears `issued_at` fails (trigger), and the API never writes these columns outside a custody
  activity (bite check: removing the trigger makes this test fail).

### ST-9 Custody unhappy paths

> **Amended 2026-11-08.** `resolve`, `propose` and `decide` below read as `act` on handler node
> instances (part 6 §27.9). Assertions are unchanged except where part 6 §27.13 says otherwise.

**US-9.1** As the Asset Store, I report a problem on one item without blocking the others.
- AC-9.1.1
  - G: SP-01, HT-01 and LT-01 all `awaiting_preparation`.
  - W: Grace reports HT-01 `missing` (severity high, description "Not in locker B3", 2 evidence
    files).
  - T: HT-01 becomes `exception_open` with `state_before_exception=awaiting_preparation`, the
    exception is `open` and assigned to Ops Leads, and the evidence is attached. SP-01 and LT-01
    still offer "Prepare and initiate handover". The graph shows an Exception node on HT-01's lane.
- AC-9.1.2: a second report on HT-01 while one is open → 409. Concurrent reports on HT-01 and LT-01
  are both accepted.
- AC-9.1.3: evidence that is >10 MiB, a disallowed type, or has mismatching magic bytes (a PNG
  renamed `.pdf`) → 422 at `POST /api/evidence`, and no row is stored. Evidence IDs owned by another
  user or already attached → 422 at report.
- AC-9.1.4: the join with HT-01 open behaves as follows: after SP-01 and LT-01 are received, the
  status stays `approved` (`live=3 crossed=2 -> Wait`).

**US-9.2** As the investigator (Frank), I work the exception.
- AC-9.2.1: open → investigating → awaiting_information → investigating, each with required notes,
  each producing an `custody_exception_updates` row and an audit row. Non-investigators → 403.
  Disallowed transitions (for example open → resolved directly) → 409.
- AC-9.2.2
  - G: investigate due 48 h, demo unit 10 s.
  - W: 480 s pass unresolved.
  - T: `overdue_at` is set, the event is `sla_breached`, and Frank's and Carol's dashboards show
    "Overdue".

**US-9.3** As the investigator, I resolve recoveries and false reports directly.
- AC-9.3.1: `resolve_direct{recovered, resumeState:"awaiting_preparation"}` → the exception is
  `resolved` (resolution `recovered`), the event is `asset_recovered`, and HT-01 is back in
  `awaiting_preparation` with its stage SLA re-armed.
- AC-9.3.2: for an item lost after receipt and then found at the agency, the allowed resume states
  are {`in_requester_custody`, `received_by_agency`}. Anything else → 422.
- AC-9.3.3: `false_report` → `cancelled_as_false_report`, and the case resumes at
  `state_before_exception`.

**US-9.4** As the investigator, I propose a replacement and the owner decides.
- AC-9.4.1: `propose{replacement, substituteAssetId: HT-02}` requires HT-02 to be live, not retired,
  not held, manual, the same system and the same `credential_type` (otherwise 422 or 409). The
  exception becomes `awaiting_resolution_approval`, and Carol sees "Approve replacement".
- AC-9.4.2: Carol approves. In one transaction:
  - the HT-01 case becomes `resolved_exception` (terminal) and the HT-01 item is excluded (reason
    `replaced`);
  - HT-01 is retired (`lost_replaced`);
  - a new item `HT-02` (`replaces_item_id` = HT-01's item), a hold and a case at
    `awaiting_preparation` are created, and a new child starts;
  - the event is `replacement_authorized`.
  
  The graph shows HT-01's lane struck through, with a "replaced by" edge to a new HT-02 lane.
- AC-9.4.3: the join after replacement counts HT-02, not HT-01. Issue happens only after HT-02 is
  received.
- AC-9.4.4: Frank (the proposer) deciding his own proposal → 403. Grace (the reporter), even if
  granted owner rights in a test fixture → 403. Carol rejecting with notes → the exception goes back
  to `investigating` and the proposal is `rejected`.

**US-9.5** As the investigator, I propose a write-off and the owner decides.
- AC-9.5.1 (lost before issuance, partial)
  - G: HT-01 written off (approved), SP-01 and LT-01 received.
  - T: HT-01 is excluded, the asset retired (`written_off`), and the join gives
    `live=2 crossed=2 excluded=1 -> Issue`, so `credential_issued`.
- AC-9.5.2 (all items lost)
  - G: a single-item request, written off before receipt.
  - T: the join gives `AllExcluded`, so `close_unfulfilled` sets `closed`, with
    `decision_reason='unfulfilled_all_items_written_off'`, audit `request.closed_unfulfilled`, and
    the workflow completes. The graph shows "Closed – unfulfilled".
- AC-9.5.3 (lost after issuance)
  - G: request `credential_issued`.
  - W: Bob reports SP-01 `missing` from `in_requester_custody`.
  - T: the status stays `credential_issued` (issuance is not revoked), Bob may still close, and the
    request shows "Attention required: 1 custody exception". After write-off approval, SP-01 is
    terminal (asset retired), and once all other cases are terminal the parent completes.
- AC-9.5.4 (lost after close): the same flow works when the request is already `closed`. Closure
  doesn't block reporting or resolving.

**US-9.6** As the investigator, I propose a waiver for a recorded deviation.
- AC-9.6.1
  - G: LT-01 exception `identity_mismatch` (serial label differs) at `handover_pending`.
  - W: a waiver is proposed and Carol approves.
  - T: the case resumes at `handover_pending`, the event is `waiver_granted` with the deviation
    notes, and the exception is `resolved` (resolution `waiver`).
- AC-9.6.2: waiver rejected → back to investigating, and the case stays `exception_open`.

**US-9.7** As requester and agency, we can dispute handoffs.
- AC-9.7.1: Bob at `handover_pending` chooses "Dispute" with notes. The handoff becomes `disputed`,
  a `handover_disputed` exception opens (`state_before_exception=handover_pending`), and the item
  doesn't count as received.
- AC-9.7.2: Claire at `return_pending` disputes ("wrong token returned"). A `return_disputed`
  exception opens, and a recovered resolution may resume at `return_pending` or
  `received_by_agency`.

**US-9.8** As the requester, I am reminded to return items.
- AC-9.8.1: at `activity_end_at`, each case still `in_requester_custody` gets the `return_due` event,
  and Bob's dashboard shows "Return due".
- AC-9.8.2: at end + `RETURN_GRACE_HOURS`, still-unreturned cases get `return_overdue` and appear on
  the owner and investigate-team dashboards. **No exception is opened automatically**: AC asserts
  that `custody_exceptions` count is unchanged.

**US-9.9** As an auditor, I want an immutable custody ledger.
- AC-9.9.1: every case transition has a `custody_events` row (from, to, actor, notes, op_ref).
  `UPDATE` or `DELETE` on `custody_events` and `custody_exception_updates` as the app DB role fails
  with "permission denied".

### ST-10 Timers, SLAs and the demo clock

**US-10.1** As a demo presenter, I want compressed time.
- AC-10.1.1: with `SLA_UNIT_SECONDS=10`, a 4 h prepare SLA fires after 40 s (±2 s). The new-request
  form offers window presets "+1 min" and "+3 min" only when `DEMO_MODE=1`.
- AC-10.1.2: the Engine panel lists each armed timer with kind, SGT fire time and a countdown. Fired
  timers appear in the timeline as System events.

**US-10.2** As an engineer, I want timer correctness.
- AC-10.2.1: a stage SLA firing after the case left the stage does nothing (the handler checks the
  current state and stage). There is a test for it.
- AC-10.2.2
  - G: the worker is stopped across a timer's due time.
  - W: it restarts.
  - T: the timer fires once and exactly one overdue event exists.
- AC-10.2.3: workflow unit tests use the Temporal test environment with time skipping. The 48 h
  exception-SLA test finishes in under 5 s of wall time.
- AC-10.2.4: timer settings are passed via workflow input. Changing `SLA_UNIT_SECONDS` and
  restarting the worker doesn't break replay of running workflows (replay test).

**US-10.3** As a reviewer, I see overdue work first.
- AC-10.3.1: "Needs my action" sorts overdue items first, with a red "Overdue {n}h" badge computed
  from DB `overdue_at` (the list works with Temporal down).

### ST-11 Read models, dashboards and visualisation

**US-11.1** As any user, I get a correctly scoped request list.
- AC-11.1.1: list results follow §8 visibility for each seeded persona. A table-driven test asserts
  the exact visible set per persona over a fixed seed.
- AC-11.1.2: for every persona and request in the seed, the "Needs my action" set equals the set of
  requests where `/actions` is non-empty (consistency test).

**US-11.2** As any viewer, I see a live, generated graph.
- AC-11.2.1: the request graph renders the actualised plan from `/plan` (part 4 R8). The FINDB v1
  graph shows its linear approvals plus Owner; HRIS v1 shows the §25.2 shape. After a new version is
  published, existing runs keep their pinned shape (AC-17.2.1).
- AC-11.2.2: node styles follow §12 for each state, and edge labels show the conditions. The Join
  node renders `join_trace`.
- AC-11.2.3: the graph updates within 3 s of an action by another persona (polling). A 3-item request
  plus 1 replacement lane renders in under 300 ms (performance mark).

**US-11.3** As an actor, I see only actions I may take.
- AC-11.3.1: the action panel is driven by `/actions`. Crafting a hidden action via curl still gets
  the correct 403 or 409 from the API.
- AC-11.3.2: return, reject, dispute and exception forms block submission without notes (client) and
  the server rejects blank notes (422).

**US-11.4** As any viewer, I see a merged timeline.
- AC-11.4.1: the timeline merges submission, step and owner decisions (with remarks), edits (diffs),
  custody events, exception updates, proposals, timers and System actions. Ordering is by
  `created_at`, displayed in SGT. Tombstoned actors show "(deactivated)".

**US-11.5** As an attacker, I can't enumerate requests.
- AC-11.5.1: for a non-viewer, a random UUID and a malformed ID, `/requests/:id`, `/timeline`,
  `/engine` and `/actions` return identical 404 bodies, with p95 latency difference under 50 ms.

### ST-12 API contract

**US-12.1** As a frontend developer, I want a stable, documented API.
- AC-12.1.1: every endpoint in §11 returns the envelope, and every error code used in this addendum
  appears in a single `error_codes.rs` enum, which is asserted by a test that greps the addendum.
- AC-12.1.2: a mutating call without `X-Idempotency-Token` → 422 `IDEMPOTENCY_TOKEN_REQUIRED`. A token
  reused for a *different* body → 409 `IDEMPOTENCY_TOKEN_USED`.
- AC-12.1.3: an OpenAPI 3.1 document is generated (utoipa) and served at `/api/openapi.json`. CI fails
  if it differs from the committed copy.

### ST-13 Engine resilience and evolution

**US-13.1** As an operator, I want crash safety.
- AC-13.1.1: a chaos test kills the worker at each of submit, step decision, owner approve (between
  `record_owner_decision` and `instantiate_custody`), fork, receipt and join. After each restart the
  scenario finishes and `/api/ops/reconcile` returns an empty list.
- AC-13.1.2: killing the API mid-update (after Temporal accepted it) → the client retries with the
  same token, gets the original result, and no duplicate is written.

**US-13.2** As an engineer, I want safe workflow code changes.
- AC-13.2.1: CI exports histories from the S1–S6 runs and replays them against the current worker
  code. Any non-determinism fails the build.
- AC-13.2.2: the addendum's worked example changes behaviour using `patched("auto-expire-v1")`, and a
  replay test proves workflows started before the patch still replay.

**US-13.3** As an engineer, I want bounded history.
- AC-13.3.1: a synthetic test with 2,100 exception updates triggers continue-as-new. The new run's
  projection equals the DB, and children remain attached.

### ST-14 Security and test discipline

**US-14.1** As a security reviewer, I want the OWASP-relevant tests in place.
- AC-14.1.1: for every mutating endpoint there are tests for positive, negative (validation), wrong
  state, wrong session (none or expired), IDOR (cross-organisation → 404, same-organisation ineligible → 403) and
  bounds/fuzz on every text field. They run against real Postgres and a Temporal test server, with no
  repository mocks.
- AC-14.1.2: bite checks — individually disabling each of these makes at least one named test fail:
  - the organisation predicate;
  - the self-approval check (both layers);
  - the activity membership re-check;
  - the hold unique index;
  - the guarded UPDATE WHERE clauses.
- AC-14.1.3: no secrets or evidence bytes are in workflow inputs or results. A test scans the
  exported histories for the seeded evidence SHA bytes and password hashes and finds none.
- AC-14.1.4: mutating routes reject a cross-origin `Origin` header (403). Security headers (CSP,
  `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`) are present on every
  response.
- AC-14.1.5: workflow and update payloads are size-limited (notes ≤ 2000 / 4000, items ≤ 10).
  Oversize requests are rejected by the API with 422 before reaching Temporal.

**US-14.2** As a presenter, I want recorded end-to-end demos.
- AC-14.2.1: a Playwright suite with video recording (repo standing rule) runs S1–S6 (§15), switching
  personas through the demo login. Each step asserts the graph node states, the timeline entries and
  DB rows (through a test-only read endpoint enabled only with `DEMO_MODE=1`).

---

