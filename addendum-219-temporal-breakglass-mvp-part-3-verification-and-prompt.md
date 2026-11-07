# Addendum 219 (part 3) — Demo scenarios, verification, scope, open points and the implementation prompt

**Part 1:** `addendum-219-temporal-breakglass-mvp.md` (§0–§12). This part also holds §13 (seed data). **Part 2:** `addendum-219-temporal-breakglass-mvp-part-2-stories.md` (§14 user stories and ACs). **Part 4:** `…-part-4-workflow-definition-and-editor.md` (§20–§26, normative where it conflicts). **Part 5:** `…-part-5-editor-stories.md` (ST-15–ST-24 and ST-33). **Part 6:** `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27–§29: exception handlers, cloning, 100% coverage gate). **Part 7:** `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30). **Part 8:** `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34: module map, coupling review, Rust conventions). **Part 9:** `…-part-9-exception-runtime-and-cloning-detail.md` (§35–§37, ST-31: runtime, races, effects, error matrix, cloning algorithm, coverage CI). **Part 10:** `…-part-10-exception-ui-and-walkthroughs.md` (§38–§40, ST-32: UI, PM walkthroughs, S10–S12 state tables). **Appendix A:** `…-appendix-a-illustrative-example.md`.

## §13 Seed data and personas

All names are synthetic. Passwords are `Password!2026Demo`; the admins use `ChangeMe!2026Demo`.

**Organisation AUR (Org Aurora)**

| Cluster (type) | Team | Members |
|---|---|---|
| Alpha Pte (vendor) | Alpha Engineers L1 | Alice Tan (team_admin), Bob Lim |
| | Alpha Leads L2 | Aaron Koh |
| Beta Pte (vendor) | Beta Engineers L1 | Betty Ong |
| | Beta Leads L2 | Brian Goh |
| AUR Ops (agency) | Ops Reviewers L1 | Claire Wong, Emma Yeo, Ivan Ng |
| | Ops Leads L2 | Frank Chua |
| AUR Logistics (agency) | Asset Store L1 | Grace Neo, Leo Tan |
| AUR PA (pa) | Owner Support L1 | Priya Nair |

Owners and other users: Carol Sim (owner), Dan Teo (owner), Ivan Ng (owner of OPSTOOL), Emma Yeo (delegate).

| System | Chain (ordinal 1 = submitter) | Owners | Delegate | PA |
|---|---|---|---|---|
| HRIS | Alpha L1 → Alpha L2 → Ops L1 → owner | Carol (primary), Dan (backup) | Emma (now → +2 days) | Owner Support |
| FINDB | Beta L1 → Beta L2 → Ops L1 → Ops L2 → owner | Dan (primary), Carol (backup) | — | Owner Support |
| OPSTOOL | Ops L1 → Ops L2 → owner | Ivan (primary), Carol (backup) | — | Owner Support |

HRIS custody stages (nodes of `standard_custody_lane` v1, below): prepare = Asset Store; initiate_handover = Asset Store (gives the combined
action); verify_return = Ops Reviewers; return_to_inventory = Asset Store; investigate = Ops Leads.
Due hours 4, 2, 24, 8, 48. FINDB and OPSTOOL (`basic_custody_lane` v1) use the same teams, with prepare and initiate split
across Grace and Leo's team versus Ops Reviewers, to show the non-combined path.

Assets (manual):
- HRIS: SP-01, SP-02, SP-03 (secure phone, device); HT-01, HT-02 (hardware token, token); LT-01, LT-02 (laptop, device).
- FINDB: HSM-01 (token).
- OPSTOOL: KB-01 (sealed password envelope, password).

**Exception handlers (part 6 §27.11) live in the lane templates, not in the root workflows.** `standard_custody_lane` v1 (HRIS) draws H1 before issuance, H3 damaged in custody, H2 lost in
custody, H4 problems at return, then a `fallback`; evaluated in that order (H3 before H2). They use
custom teams: in H3 Asset Store L1 investigates (ports include a direct `quarantine`) but the Ops Leads L2 team approves a replacement.
`basic_custody_lane` v1 (FINDB, OPSTOOL) and `sen_custody_lane` v1 (PORTAL) each hold the fallback handler only.

**System PAYROLL (S12 clone target):** created in S12 through the create-system wizard, with no seeded
definition. Owners Dan (primary) and Carol (backup); vendor cluster Beta Pte linked; PA Owner Support; one live asset per allowed credential type (PAY-D1 device, PAY-T1 token) so V-43 is not raised.

**Organisation SEN (Org Sentinel):** Gamma Services (vendor) L1 Gina Ho; SEN Ops (agency) L1 Hank Low; system
PORTAL (chain Gamma L1 → SEN Ops L1 → owner Sam Lee); asset HT-S1.

**Organisation template libraries (part 4 §25.1, D-40):** AUR holds `pin_reset` v1 (subject `item`),
`standard_custody_lane` v1 (subject `item`; embeds `pin_reset` v1 once per item), `dual_review` v1 and
v2 (subject `none`; v2 is a draft until S13) and `basic_custody_lane` v1 (subject `item`). SEN holds
`sen_custody_lane` v1. Pins: HRIS v1 pins `dual_review` v1 and `standard_custody_lane` v1; FINDB and
OPSTOOL pin `basic_custody_lane` v1; PORTAL pins `sen_custody_lane` v1 (D-42).

**Workflow definitions (part 4 §25):** HRIS v1 published (the §25.2 graph: root with a `review` single
embed of `dual_review` v1 and an `items` per-item embed of `standard_custody_lane` v1); HRIS v2 draft
(upgrades `review` to `dual_review` v2, 3-branch review, `k_of_n` k=2, with `required_score=5` as the
embed input, and `max_items=3`); FINDB v1 and OPSTOOL v1 published as linear chains (the table above)
with a per-item embed of `basic_custody_lane` v1; PORTAL v1 likewise with `sen_custody_lane` v1. The
chain and custody-stage tables above describe the content of those definitions.

**Admins:** Greg Global (`full_admin`, global), Amanda Scoped (`org_admin` @AUR) and Mona Multi
(`org_admin` @AUR and @SEN: two grant rows; agency user with live memberships of both AUR and SEN,
so D-38's membership rule holds). Amanda is a member of AUR only. Greg's grant has `organization_id NULL`.
**System actor:** `00000000-0000-7000-8000-000000000001` (is_system, is_protected).

---

## §15 Demo scenarios

| ID | Script (persona → action) | Shows |
|---|---|---|
| S1 Happy path, 3 items | Bob submits HRIS (SP-01, HT-01, LT-01) → Aaron approves → Claire approves → Carol approves (per-item embed: 3 lanes) → Grace prepare+handover ×3 → Bob confirms ×3 (join → issued) → Bob closes → returns and restocks ×3 → workflow completes | Sequential chain, embedded templates, per-item lanes and join, close independent of return; Issuance and returns panel goes from 3 outstanding to 0; archived history appears after completion |
| S2 Return loop | Bob submits FINDB-like on HRIS → Aaron returns "Wrong SR" → Bob resubmits (continue-as-new; new run ID) → approvals → owner approves | Remarks in DB, run chain, snapshot rebuilt |
| S3 Lost before issuance → replacement | S1 until fork → Grace reports HT-01 missing → others proceed and are received (join Wait) → Frank investigates and proposes replacement HT-02 → Frank tries to approve own proposal (403) → Carol approves → HT-02 lane → received → issued | Per-item isolation (lane template handlers), owner-gated resolution, join recomputation |
| S4 Lost after issuance → write-off | S1 until issued → Bob closes → Bob reports SP-01 missing → Frank proposes write-off → Bob completes the `loss_report` task (H2, part 6 §27.13) → Carol approves → remaining returns → workflow completes | Issuance not revoked, closed-but-running parent |
| S5 Auto-expire | Bob submits with "+1 min" window → no approval → timer fires → Expired | Durable timer, System actor |
| S7 Version pinning | Bob submits HRIS (v1) → Amanda publishes v2 → Bob's request still runs v1 → Claire returns → Bob resubmits → run #2 pinned to v2; run selector shows both | Per-run pin (R7) of the whole embed tree (D-42), definition change without deploy |
| S8 High-risk parallel review | Bob submits 4 items → Claire approves with `risk_level=high` → `review` embed (`dual_review` v1): split: Frank scores 3, Grace scores 2 (concurrently) → join → `score_ok` 5 ≥ 6 false → `failed` → Rejected; repeat with 3+3 → `passed` → Owner → per-item embed | Variables, field input, fork mutation, merge, configurable rule, approved/rejected |
| S9 Editor and simulation | Amanda opens HRIS v2 draft → breaks a connector (V-05 shown, Publish disabled) → fixes it → simulates 1 item then 5 items → toggles `score_ok` operator `>` vs `>=` and re-simulates → publishes | Connector rules, live validation, actualisation for N, quorum inside an embedded template, operator configurability |
| S6 Negatives | Claire + Emma approve concurrently (one 409) → Ivan approves own OPSTOOL request (403) → Gina opens HRIS request URL (404) → kill worker during owner approval, restart, reconcile empty | Concurrency, self-approval, IDOR, crash safety |
| S10 Mixed fulfilment exceptions | Four lanes on HRIS v1 hit four different handler paths (missing with substitute, missing without, lost after issuance, damaged after issuance with a supplementary lane); script and expected summary in part 6 §27.12.1, step tables in part 10 §40.1; includes **S10-T** (step 5a: 80 s idle shows the handler SLA breach and escalation) and **S10-X** (sidecar request R2: admin reset with and without force) | Drawn handlers, custom teams, refusals (`NO_SUBSTITUTE`), post-issuance replacement, quarantine, exception bands, final summary counts |
| S11 Cascade and exhaustion | (a) a replacement is disputed and hits `REPLACEMENT_LIMIT`, so both lanes are written off and the request closes unfulfilled; (b) a return verified as damaged is auto-quarantined via H4; scripts in part 6 §27.12.2 | Replacement bounds, `AllExcluded`, `closed_unfulfilled`, quarantine blocks reuse (`ASSET_QUARANTINED`) |
| S12 Clone a workflow | Amanda or Greg creates PAYROLL and clones HRIS v1 into it → maps Alpha Leads to Beta Leads (leaves the vendor team Alpha Engineers L1 unresolved) → V-42 blocks simulate and publish → Amanda chooses the team → publishes → Betty submits a PAYROLL request | One copy path, assignee remapping, unresolved placeholders, lineage (part 6 §28) |
| S13 Template reuse and upgrade | Amanda publishes `dual_review` v2 in the AUR library → the library row shows "Used by HRIS v1" → a running HRIS v1 request is unaffected (still on `dual_review` v1) → the HRIS v2 draft shows "v2 available · Upgrade" on the `review` embed → upgrade re-validates the bindings (V-52 warning on `required_score`) → simulate → the template picker greys an ineligible template (e.g. one containing `owner_approval`, V-45) with its reason | D-42 pinning, eligibility (E1–E6), reuse of one template by several roots, upgrade without touching running requests |
| S14 Expression and Function nodes | Amanda adds `risk_fx` (V-58 on a typo, then `risk_score` 4.0 and `needs_dual` false in simulation), branches on `needs_dual` with a rule labelled "Yes / No", and adds `window_fn` (Test run: `fuel_exhausted` stored in `window_error`, `thrown` on `eval`, then the SGT label) → with `FOUR_EYES_PUBLISH=code` her publish gets 409 `FOUR_EYES_REQUIRED` → Greg reviews the diff and publishes → Bob's v2 request shows `window_label ← function (JS)`; script in part 12 §42.16 | Opus-style f(x) and JS nodes, results as variables, custom rule labels, error ports with stored error types, sandbox failing closed, configurable four-eyes |

The seeded demo workflow and the scenarios must exercise **every row** of the permutation matrix (part 6
§27.12, AC-27.10.1), to illustrate the engine's capabilities.

---

## §16 Verification and Definition of Done

**Test layers:**
1. Pure domain unit tests: transition tables, join predicate, resolution effects, validators.
2. Workflow tests on the Temporal test environment with time skipping, with activities stubbed **only
   at the workflow boundary**. Activities have their own real-DB tests.
3. Activity integration tests against real Postgres 16: authorization re-checks, guarded updates,
   op_ref idempotency, audit-or-rollback.
4. API integration tests: the black-box harness pattern from `crates/ffs-test-support` (a real binary
   on a random port, a leased DB, a Temporal dev server).
5. Replay tests (AC-13.2.1).
6. Playwright e2e with video (S1–S14); the PM walkthrough fixtures (`walk_pm_NN` black-box, `ui_pm_NN`
   jsdom; part 10 §39, AC-32.12.1) and the S10–S12 checkpoint tables (part 10 §40) run as executable tests.
7. `mvp-engine` pure tests: validation fixtures (one per V-code), `settle` property tests,
   simulation-equals-execution (AC-22.1.2).
8. **Coverage gate** (part 6 §29): 100% lines, functions and regions on every Rust crate under
   `cargo-llvm-cov`; Vitest with jsdom at 100% lines, branches, functions and statements per file.
9. **Module fitness tests** (part 8 §30.3, ST-30): dependency direction, banned crates and
   `HashMap` bans.
10. **Expression and Function sandbox tests** (part 12 §42.9): the red-team corpus RT-01…RT-14, `cargo fuzz` on `fx::parse` and `fx::eval`, fuel determinism across Test run, simulation and execution per `runner_build`, and the container-hardening check (AC-36.1.3).

**Delivery order** (repo cross-layer rule, `CLAUDE-test-discipline.md`): UI DOM contract and scaffold
→ Playwright and component tests (RED) → domain, activity and workflow tests (RED) → schema →
activities → workflows → routes → GREEN → refactor.

**Definition of Done:**
- Every AC in §14, in part 5 (ST-15–ST-24 and ST-33), in part 7 (ST-25–ST-30), in part 9 (ST-31), in part 10 (ST-32), in part 11 (ST-34) and in part 12 (ST-35, ST-36) maps to at least one named test, through the mapping table
  `docs/ac-traceability.md` in the MVP directory.
- All tests pass.
- S1–S14 videos are recorded.
- The coverage gate is green (part 6 §29.7: `make coverage-gate`).
- Every row of the permutation matrix (PM-01 … PM-31, part 6 §27.12) is mapped to its tests and
  scenario or AC.
- The seeded HRIS v1 definition validates with zero errors, and every row of the coverage matrix
  (part 4 §25.3) is exercised by a passing test.
- `/api/ops/reconcile` is empty after the full suite.
- Bite checks pass (AC-14.1.2).
- No AWS, Slack or Entra code is present (a grep gate).
- The root repo build is unaffected (AC-0.3.1).
- Report the exact validation rung reached.

---

## §17 Out of scope / Future

- AWS credential exchange and secret delivery.
- Email, Slack and in-app push notifications. The UI dashboards stand in for them.
- Entra SSO and TOTP.
- Export/import of requests and other data. (Only the workflow-definition file export/import of
  part 4 §24.2 is in scope.)
- `step_kind='peer'` approval steps. (Quorum and all-of review are in scope via `split`/`join`,
  part 4 §26.)
- Multiple systems or multiple SRs in one request (D-24).
- More than one per-item embed in a system workflow; team-typed template parameters; sharing templates across organisations (cross-organisation reuse is by clone only).
- Chargeback or billing for lost items.
- An asset repair or release-from-quarantine workflow (a quarantined asset stays quarantined).
- Cloning a workflow across organisations by an `org_admin` (only `full_admin`, OP-8).
- Collaborative real-time co-editing of a draft (one author at a time, with `STALE_DRAFT`).
- Payload-codec encryption of Temporal payloads (D-16).
- A purge job that deletes records more than 7 years after request closure (§2.7). The MVP deletes
  nothing.
- Temporal's built-in archival to object storage (S3 or GCS) for production, which could replace the
  Postgres history archive (D-18).
- A namespace per organisation.
- System participant organisations (`system_organizations` participant edges, addendum-206) and
  cross-organisation participation by agency-affiliated users.
- Temporal Cloud.
- Using Temporal as the engine for the production app. That would need its own addendum after this
  MVP is evaluated.

---

## §18 Open points

All open points are resolved (user answers, 2026-11-07). OP-6 to OP-9 (2026-11-08) are recorded
with the recommended resolution and are open for the user to overturn.

| ID | Question | Resolution |
|---|---|---|
| OP-1 | What should the return counter count? | `return_count` counts every return to vendor (step and owner). `resubmit_count` counts resubmissions. **Issuance and returns are tracked per item and per asset** (D-19, US-8.6) |
| OP-2 | Is it acceptable that Temporal deletes its history after 7 days? | Yes, because Temporal is not the audit store. Every record and audit row is kept in Postgres for 7 years, append-only, and each closed Temporal history is archived into Postgres too (§2.7, D-16, D-18, US-1.5) |
| OP-3 | Should auto-expire reuse `rejected` with a `decision_reason`? | Yes, accepted (D-10) |
| OP-4 | Is the organisation tenant the app's `organizations` concept, made a hard boundary? | Yes, accepted (D-1) |
| OP-5 | Should `approved → closed` (all items written off) be allowed for the System actor only? | Yes, accepted (D-13) |
| OP-6 | May two live requests carry the same SR on the same system? | No (recommended): 409 `DUPLICATE_SR`; reuse after reject or close; other systems unaffected (D-24, AC-24.3.2) |
| OP-7 | On resubmit, does the new run use the request's old version or the system's current one? | Current published version (recommended), matching today's snapshot rebuild (R7, AC-17.2.2) |
| OP-8 | May an `org_admin` clone a workflow from another organisation? | No (recommended): cross-organisation cloning is `full_admin` only, even for an `org_admin` whose scope includes both organisations (Mona, D-38), and every assignee is remapped (no defaults); an `org_admin` gets 404 (D-32, part 6 §28.8) |
| OP-9 | May an item be replaced after issuance (lost or damaged in requester custody)? | Yes (recommended), bounded by `max_replacements_per_item` and the per-item embed's `allow_post_issuance_replacement` flag (D-30, part 6 §27.5.1) |
| OP-10 | Must Function-node code need a separate code-author role and a second person to publish it? | **Resolved 2026-11-10 (user):** no separate role; any definition editor may write code. Four-eyes publish is one app-wide environment variable, `FOUR_EYES_PUBLISH` = `off` (default), `code` or `all` (part 12 D-55, §42.10) |

---

## §19 The implementation prompt (replaces the draft "Advanced Implementation Guide")

> Copy this block verbatim into the implementing agent. It is self-contained, but it points to the
> sections above, which are normative.

```
ROLE
You are implementing a demonstration MVP: "Temporal Break-Glass MVP", specified normatively in
docs/specs/addenda/addendum-219-temporal-breakglass-mvp*.md (parts 1–11, §1–§41). Later parts win over
earlier ones where they conflict (part 4 §26, part 6 §27.13 and part 8 list every superseded clause). Where this prompt and the
addendum disagree, the addendum wins. Do not implement anything listed in §17.

ABOUT APPENDIX A (READ BEFORE ANYTHING ELSE)
You will also be given an "Advanced Implementation Guide" about a procurement workflow (budget,
PROCURE_HARDWARE, PROCURE_SOFTWARE, total_spend). It is reproduced in Appendix A. It is ONLY AN
ILLUSTRATION of engine features. It is NOT our problem domain. Do NOT build procurement, do NOT seed
its variables or nodes, do NOT copy its code (its defects are listed in §19.1). DO demonstrate every
feature it illustrates — variable init, parallel fork with branches mutating a shared variable,
merge/join, configurable `left op right` rule evaluation, approved/rejected routing, completion,
and a React Flow live view (active blue, completed green, variables panel) — on OUR break-glass
domain, exactly as mapped in part 4 §25.3.

GOAL
Show a Temporal workflow engine running our break-glass request lifecycle end to end, driven by a
per-system workflow DEFINITION that admins draw in a GRAPHICAL WORKFLOW EDITOR (part 4), with the
request's ACTUALISED PLAN expanding a per-item embed of an organisation template into one lane per item (§20) —
configurable approvals (sequential, parallel, quorum), variables and rules, owner terminal approval, per-item physical-custody fork/join, durable
timers, and the custody unhappy paths — on top of our real identity model (organisation tenant, clusters,
teams L1–L5, team_admin, admin roles/grants where org_admin is scoped to one or more named
organisations the grantee belongs to (D-38), primary/backup owners, delegates, PA team).

STACK (fixed)
- Rust 1.99.1 from github.com/wshyang/psd-rust-artifacts (concat parts, sha256 -c, extract; pin in
  rust-toolchain.toml). Own Cargo workspace under sidequest/temporal-breakglass-mvp/ (NOT a member of
  the root workspace), own Cargo.lock and vendor/ for offline builds.
- crates (part 8 §30.2): mvp-kernel, mvp-contract, mvp-engine, mvp-custody, mvp-authz, mvp-db
  (sqlx 0.8, compile-checked queries, migrations), mvp-worker (temporalio-sdk 1.1.x: workflows +
  activities, task queue "breakglass"), mvp-api (axum 0.7 + temporalio-client; port 3001),
  mvp-fn-runner (wasmtime + pinned QuickJS WASM; Unix socket, own hardened container; part 12 §42.8);
  mvp-domain is retired. Dependency direction per §30.3.
- web: Next.js 15 App Router + TypeScript + @xyflow/react 12; /api/* rewritten to :3001 (same-origin).
- Temporal: `temporal server start-dev --namespace breakglass-mvp --db-filename .temporal/dev.db`.
- PostgreSQL 16 on :5433.

NON-NEGOTIABLE RULES
1. Temporal owns progress; Postgres owns facts (§2). Every remark, decision, custody event,
   exception, evidence file and audit row is written to Postgres by an ACTIVITY in ONE transaction
   together with a workflow_ops(op_ref) row and the semantic audit row. Audit failure ⇒ rollback.
2. Lifecycle columns are written only by activities (D-3). The API never updates them.
3. Human actions are Temporal UPDATES with validators; update id = X-Idempotency-Token (D-4).
   Signals only for system events (submitter_tombstoned, child→parent case_milestone).
4. Three-layer authorization (D-5): API visibility (404) + eligibility (403) from DB; workflow
   validator for state/position/lock/self-approval; activity re-checks membership in-transaction.
5. Runtime self-approval prevention (D-6) for step, override, owner and resolution decisions.
6. Organisation isolation (D-1): every query on organisation-owned data has an organisation predicate; cross-organisation ⇒ 404.
7. Workflow determinism: NO tokio::select!, tokio::time, SystemTime::now, randomness, I/O, Rc<RefCell>
   or HashMap-ordered decisions inside workflow code. Use the SDK's wait_condition, timers,
   temporalio_sdk::workflows::select!, child workflows, continue-as-new, patched(). State lives on the
   #[workflow] struct. VERIFY every SDK signature on docs.rs before use.
8. Repo invariants: UUIDv7 ids, soft delete via expires_after, seeded System actor row
   00000000-0000-7000-8000-000000000001, no JSONB for business data, real-DB tests (no repository
   mocks), IDOR negative tests, bite checks.
9. Evidence bytes never enter Temporal (uploaded first as pending rows; only ids travel).
9a. Long-term data lives in Postgres for 7 years (§2.7): audit_log, custody ledger and history archive
    are append-only; nothing is purged. Archive every closed workflow history into
    workflow_history_archive (D-18). Track issuance and returns per item and per asset (D-19).
10. No permissive CORS; HttpOnly SameSite=Lax cookie + Origin check.
11. Exclude AWS, Slack, Entra, email, TOTP. All items are manual (physical custody). The only
    export/import is the workflow-definition file (§24.2).
12. Definition vs request (§20, D-21, D-23): a system's definition is cardinality-free; lanes are created when
    the token ENTERS the per-item embed (R1), never at submit; joins range over the lane collection
    (R4); lanes change only by approved replacement/write-off (R5); consumed rule inputs lock (R6);
    each plan run pins one published version, resubmit re-pins (R7); the request graph renders from
    Postgres /plan (R8).
13. Connector rules (§21, §22): exactly ONE start (no in-connector), ≥1 terminal (no out-connector);
    every other node's in-connector has ≥1 edge; every out-port has exactly one edge; fan-out only
    via split. Every approval/owner_approval has approve, return and reject wired; reject → a
    rejected terminal; return → the single amend node (D-25). ONE validator (mvp_engine::validate)
    used by editor, validate, simulate and publish; the UI never reimplements it.
14. One request = one system, one SR/reference, one activity window; 409 DUPLICATE_SR (D-24).
15. One interpreter (mvp_engine: compile/validate/settle/simulate, pure, no I/O, BTreeMap only),
    shared by the workflow, the API simulation and replay tests (D-22). The CompiledPlan is loaded by
    activity so it lives in history. Request variables are typed rows, written only by activities;
    lane writes to request variables go through the parent serially (§23.3). No JSONB anywhere in
    wf_* or request_plan_* tables.
16. Exception handlers are DRAWN (part 6 §27): the designer draws investigate, approval and resolution
    nodes; the platform owns the effects (§27.5.2) and cannot be bypassed. Route first-match by
    ordinal, with a fallback. Supplementary lanes never gate `issued`.
17. Cloning (part 6 §28) goes through the single `definition_copy` function, shared with
    draft-from-version and import. Unresolved assignees block simulate and publish (V-42).
18. Coverage gate (part 6 §29): 100% lines, functions and regions on EVERY Rust crate under
    cargo-llvm-cov; the frontend at 100% per file under Vitest with jsdom. No ignore pragmas or
    lowered thresholds; only generated files listed in coverage-exclusions.md are excluded. You choose
    the tests; coverage never replaces real-DB tests, bite checks or replay tests.
19. Module boundaries and Rust conventions per part 8 §30 and §33; the dependency fitness test (ST-30)
    must pass.
20. Exception runtime per part 9: race and TOCTOU rules (§35.4–§35.5), effects in data (§35.7), error
    matrix (§35.8) and cloning algorithm (§36); UI (§38), PM walkthroughs (§39) and S10–S12 state tables
    (§40) per part 10. Part 9 and 10 refine part 6; ST-31 and ST-32 ACs are tests too.
21. Embeddable templates (part 4 §20.4 D-39…D-43, §21.2, ST-33): a published org template (`wf_templates`,
    versioned like a system workflow) is embedded by the `embed` node, `single` (inline call frame) or
    `per_item` (lane template, one lane per item). Eligibility E1–E6 (entry inputs, normal and
    propagating exits, no root-only kinds, subject rules, private locals) is checked by the one validator
    (V-44…V-52). Bindings are typed; per-item outputs need a reducer. Publishing a root freezes its pin
    tree (`wf_embed_pins`); publishing a template never changes a root. Nesting depth ≤ 3, no cycles,
    per_item at most once per path. Custody stages and exception handlers live in the lane template.
22. Variable scope and data flow (part 11 §41, D-44…D-48, ST-34): scope is static per node context
    (`scope_table`); platform facts come from `catalogue()`, not rows. `in` is copied at call or lane
    start and read-only; `out` is copied only on a normal exit; per-item outputs are reduced once at
    `completed`; a lane writes a root variable only through a `ref`, additively, applied by the parent
    in `seq` order. V-53…V-56 are enforced by the same validator.
23. Expression and Function nodes (part 12 §42, D-49…D-61, ST-35, ST-36): `expression` evaluates FX, a
    ZEN-compatible subset parsed and typed by `mvp-engine::fx` (no eval, no host calls, statically bounded
    cost); `function` runs JavaScript `handler(input)` only as the `run_function` activity in the separate
    `mvp-fn-runner` (QuickJS on WASM in wasmtime, fresh instance per call, fuel, 32 MiB, no network, no
    secrets). FX reads its scope; a function reads only declared inputs; both read one entry snapshot and
    write ordinary variables, all or nothing. Ports are `next` and `error`; a fault never fails the request
    and may be stored in author-chosen error-type and error-message variables. Branch with `rule` (custom
    port labels) or `switch`. Variables are created only at edit time, never by a run-time write. Publishing follows `FOUR_EYES_PUBLISH` (off, code, all). RC-20 keeps every
    engine crate out of other crates.

WHAT TO BUILD
- Schema per §6 as amended by §23 (definition, plan and variable tables); seed per §13 including the
  HRIS v1 / v2-draft, FINDB, OPSTOOL and PORTAL definitions (§25).
- mvp-engine crate (§23.3) with the validation catalogue §22 (V-01…V-31, V-44…V-52 for embeds, V-53…V-56 of part 11 for data flow, V-57…V-69 of part 12 for expression and function nodes, plus V-32…V-43 of part 6
  §27.6) and simulation §24.3. Exception handlers, effects and the permutation matrix per part 6
  §27; cloning per §28; the coverage gate per §29; module layout per part 8.
- The GRAPHICAL WORKFLOW EDITOR (§24.1): palette, canvas with typed labelled port handles,
  expandable per-item embed, template library and template editor mode, inspector with visual rule builder, live validation tab, variables
  tab, simulation tab with animated trace, diff tab, autosave with STALE_DRAFT, undo/redo,
  keyboard-accessible connect, publish with notes, version list, export/import file.
- BreakglassRequestWorkflow as the interpreter host and LaneWorkflow (§23.3, replacing §7.1/§7.2
  updates with `act`); activities per §7.3 plus persist_settlement / persist_lane_settlement /
  load_plan; per-item join policy all_live = D-13; timers per §10 with SLA_UNIT_SECONDS / RETURN_GRACE_HOURS passed
  as workflow input.
- Custody transitions §9.1, exception transitions §9.2, resolution effects §9.3.
- Authorization exactly per §8.
- API per §11 (envelope, error codes, idempotency, OpenAPI at /api/openapi.json).
- UI per §12 as amended by §24.1: persona login (DEMO_MODE), dashboards, request form (one system,
  one SR, items within bounds), request page whose React Flow graph renders the ACTUALISED PLAN from
  /plan (lanes per item, taken edges, active blue, done green, run selector) plus a Variables panel,
  action panel driven by /actions, Record vs Engine panels, timeline; admin screens incl. the
  workflow editor; Issuance and returns
  panel, /reports/issuance-returns (with CSV) and /assets/[id] history (D-19).
- `make demo-up`, `make demo-reset`, `make demo-backup` (pg_dump of the app DB), scripts/bootstrap-toolchain.sh.

ILLUSTRATIVE WORKFLOW SHAPE (verify against temporalio-sdk docs; this is NOT copy-paste code)
  #[workflow]
  pub struct BreakglassRequestWorkflow { p: Projection, decision_lock: Option<String>,
                                         seen: BTreeSet<String>, cfg: TimerCfg }
  #[workflow_methods]
  impl BreakglassRequestWorkflow {
      #[init] fn new(_ctx: &WorkflowContextView, input: StartInput) -> Self { … }
      #[run]  async fn run(ctx: &mut WorkflowContext<Self>) -> WorkflowResult<Outcome> {
          // Resumed ⇒ p = load_projection activity
          // loop: race approval-deadline timer vs wait_condition(status changed) while pre-approval;
          //       after approval: wait_condition(join changed | all cases terminal) → mark_issued /
          //       close_unfulfilled / complete; continue_as_new when history is large
      }
      #[update(name = "step_decision")]
      async fn step_decision(&mut self, ctx: &mut WorkflowContext<Self>, a: StepDecision)
          -> Result<StatusView, UpdateError> { /* lock → activity record_step_decision → apply */ }
      // validator for step_decision: pure checks on &self (status, expected ordinal, lock,
      //   actor != submitter, notes rule)
      #[signal(name = "case_milestone")] fn case_milestone(&mut self, _c: &mut WorkflowContext<Self>, m: Milestone) { … }
      #[query] fn projection(&self, _c: &WorkflowContextView) -> Projection { self.p.clone() }
  }
  In the amended design (§23.3) the human-action updates collapse into one
  `#[update(name = "act")]` whose validator calls mvp_engine checks and whose handler calls
  `settle` then the persist_settlement activity. LaneWorkflow follows the same pattern with the
  custody updates in §7.2.

DELIVERY ORDER
UI DOM contract → Playwright + component tests (RED) → domain/activity/workflow tests (RED) →
migrations → activities → workflows → routes → GREEN → refactor. Record every non-spec choice as a
new addendum.

DEFINITION OF DONE (§16)
Every AC-x.y.z in §14 and parts 5, 7, 9 and 10 (except the retired AC-3.2.x / AC-3.3.x) is mapped to ≥1 named
passing test in docs/ac-traceability.md (including part 7, ST-25–ST-30, part 9 ST-31 and part 10 ST-32); every row of the §25.3
coverage matrix and of the part 6 §27.12 permutation matrix (PM-01…PM-31) passes; the coverage gate
(part 6 §29) is green; S1–S12 Playwright videos recorded; replay tests pass; /api/ops/reconcile empty after the full suite; bite checks
(AC-14.1.2) pass; grep gate shows no AWS/Slack/Entra code; root repo build unaffected; report the exact
validation rung reached and anything not executed.
```

### §19.1 What changed from the draft prompt, and why

| Draft | Problem | This addendum |
|---|---|---|
| Procurement `budget`/`total_spend` variables, `total_spend <= budget` rule | Not our process (Appendix A is illustrative only) | Same features on our domain: typed variables, `review_score` mutated by parallel reviews, `score_ok` rule with an editor-chosen operator, region join over lanes (part 4 §25.3) |
| Fixed graph INIT → FORK → MERGE → CONDITION hard-coded in workflow code | Every system needs its own shape; item count unknown at design time | Per-system versioned definition drawn in the editor, interpreted generically; per-item embed expanded at entry (§20, §23.3) |
| `make_signal_channel` + `tokio::select!` in workflow | Not the 1.1.x API; `tokio::select!` is non-deterministic | Updates with validators, `wait_condition`, SDK `select!`, timers |
| `Rc<RefCell<AppState>>` shared with query closure | Not how the SDK holds state | State on the `#[workflow]` struct; `#[query]` reads `&self` |
| Fixed workflow ID `poc-complex-1` | One request only; collisions | `bg-{ORG}-{uuid}` + child IDs |
| Signals for user actions, `unwrap()` everywhere | No result for the UI, panics on failure | Updates returning typed results; error envelope; 503 on engine down |
| All state in workflow memory | Lost to retention; not queryable | Postgres is the system of record for 7 years; closed histories archived (§2, §2.7) |
| `CorsLayer::permissive()` | CSRF and data exposure | Same-origin rewrite + cookie + Origin check |
| Hard-coded React Flow nodes | Can't show per-system workflows or N items | Editor for definitions; request graph rendered from the actualised plan (R8) |
| No users or tenancy | Required by the MVP | Organisation tenant + full identity model (§6, §8, §13) |
