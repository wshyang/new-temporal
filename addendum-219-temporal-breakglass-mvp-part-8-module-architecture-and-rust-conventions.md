# Addendum 219 (part 8) — Module architecture, coupling assessment and Rust conventions

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:** 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12); 2 `…-part-2-stories.md` (ST-0–ST-14);
3 `…-part-3-verification-and-prompt.md` (§13, §15–§19); 4 `…-part-4-workflow-definition-and-editor.md`
(§20–§26); 5 `…-part-5-editor-stories.md` (ST-15–ST-24); 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md`
(§27–§29); 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30); **8 this part (§30–§34)**; 9 `…-part-9-exception-runtime-and-cloning-detail.md` (§35–§37, ST-31); 10 `…-part-10-exception-ui-and-walkthroughs.md` (§38–§40, ST-32);
Appendix A `…-appendix-a-illustrative-example.md`.

**Why this part exists.** The user asked (2026-11-08) to apply `ecc:rust-patterns`, `modularity:design`
and `modularity:review` to the MVP design, and approved the subdomain classification in §30.1. This
part changes **structure and conventions only**: no behaviour in parts 1–7 is altered.

**Method.** The Balanced Coupling model (Vlad Khononov) rates each integration on strength (intrusive,
functional, model, contract; strongest first), distance (cost of changing both sides together) and
volatility (from the DDD subdomain). **BALANCE = (STRENGTH XOR DISTANCE) OR NOT VOLATILITY**: strong
coupling is acceptable only over a short distance or to a stable module. Strong coupling over a long
distance to a volatile module makes one change cascade through every module sharing the knowledge.

**Precedence.** Normative for module structure, dependency direction and Rust conventions. The **§30
module map supersedes the crate list in part 1 §5.1 and in the part 3 §19 prompt**; all superseded
clauses are listed at the end.

---

## §30 Module design

### §30.1 Subdomains (user-approved)

| Type | Subdomain | Volatility | Why | Lives in |
|---|---|---|---|---|
| **Core** | Workflow engine and interpreter | High | The differentiator; every new node kind, V-rule or per-item join policy lands here | `mvp-engine` |
| **Core** | Custody fulfilment and unhappy paths | High | Exceptions, resolutions, quarantine and replacement rules change with every operational finding | `mvp-custody` |
| Supporting | Identity and authorization | Medium | Needed and non-trivial, but not differentiating; changes with assignee classes | `mvp-authz`, `mvp-db::identity` |
| Supporting | Editor UI | Medium | Follows the engine's catalogue; owns no business rule | `web/features/editor` |
| Supporting | Issuance reporting | Low | Derived views and CSV over the custody ledger | `mvp-db::reports`, `web/features/admin` |
| Supporting | Workflow cloning | Low | One copy path with an assignee mapping (part 6 §28) | `mvp-db::copy`, `mvp-api::workflow_admin` |
| Generic | Audit, retention, history archive | Low | Append-only tables and a 60 s archiver loop; no business rule | `mvp-db::{audit,archive}`, worker archiver |
| Generic | Evidence storage | Low | Pending rows, `bytea`, SHA-256 | `mvp-db::evidence` |
| Generic | Temporal host adapter | Low | Workflow shells, activity shells, `EngineGateway` adapter | `mvp-worker`, `mvp-api::gateway` |

Rule of thumb: complexity lives only where volatility is high, and there it is pure.

### §30.2 Module map

| Module | Kind | Subdomain | Owns | Public surface |
|---|---|---|---|---|
| `mvp-kernel` | crate | Generic | Newtype IDs, value objects, `AssigneeSpec`, `RequestStatus` and its reasons | types and parsers only; no business decisions |
| `mvp-contract` | crate | Supporting | Wire DTOs, `ErrorCode`, Temporal names and payloads, `GraphDto`/`PlanDto`, OpenAPI schema | types and constants only |
| `mvp-engine` | crate | **Core** | `compile`, `validate`, `settle`, `simulate`, handler routing, `DefinitionRows`, `CompiledPlan` | four pure functions plus their types |
| `mvp-custody` | crate | **Core** | Transitions, phases, resolution effects, `LaneMilestone`, `available_actions`; module `vocab` | `vocab` (shared language), `transition`, `resolve` |
| `mvp-authz` | crate | Supporting | `ActorFacts`, `Rule`, `authorize`, `separation_of_duties` | pure functions |
| `mvp-db` | crate | mixed | Repositories by domain, migrations, `load_actor_facts`, `definition_copy` | one module per domain |
| `mvp-worker` | binary | Generic | Workflow hosts, activities, history archiver | none (binary) |
| `mvp-api` | binary | Supporting | axum routes, `EngineGateway` trait and Temporal adapter, workflow-admin service | none (binary) |
| `mvp-fn-runner` | binary | Generic | Function-node sandbox: `wasmtime` plus pinned QuickJS WASM, socket server (part 12 §42.8, §42.15) | none (binary; reached only through its Unix socket) |
| `web` | app | Supporting | `features/{editor,request,admin}`, `lib/api` generated from OpenAPI | none |

`mvp-domain` is **retired** (F-7).

### §30.3 Dependency rules

| From | May depend on (workspace) |
|---|---|
| `mvp-kernel` | nothing |
| `mvp-custody`, `mvp-authz` | `mvp-kernel` |
| `mvp-engine`, `mvp-contract` | `mvp-kernel`, `mvp-custody::vocab` (the module only) |
| `mvp-db` | `mvp-kernel`, `mvp-custody`, `mvp-authz`, `mvp-engine` |
| `mvp-worker`, `mvp-api` | `mvp-kernel`, `mvp-contract`, `mvp-engine`, `mvp-custody`, `mvp-authz`, `mvp-db` |
| `mvp-fn-runner` | `mvp-contract` only |

Rules beyond the edge list:
1. **No crate depends on `mvp-worker`, `mvp-api` or `mvp-fn-runner`.** `mvp-api` never depends on `mvp-worker` (F-4): they meet only through `mvp-contract` and Temporal.
2. `mvp-kernel`, `mvp-custody`, `mvp-authz` and `mvp-engine` have **no I/O dependencies**: no `sqlx`, `tokio`, `axum`, `reqwest`, `temporalio-*`, and no `rand`. `mvp-custody` and `mvp-authz` have no DB access (F-3). Only `mvp-fn-runner` may link a WASM or JS engine (RC-20).
3. **D-3 in the type system.** Every function that writes a lifecycle column lives in a submodule
   named `lifecycle` (for example `mvp_db::plans::lifecycle`). `mvp-api` must not import any
   `lifecycle` module.
4. ID minting in `mvp-kernel` is behind the cargo feature `mint`, enabled only by `mvp-db`, so pure
   and workflow code cannot create a UUID (RC-17).
5. `web/features/*` import only `lib/api` and `lib/ui`, never each other.

**Fitness test (ST-30).** A test-only workspace member `mvp-fitness` (no library code) runs
`cargo metadata` and fails when:
- a workspace edge is not in the table above;
- a banned external crate appears in the dependency closure of a pure crate;
- the `mint` feature is enabled by any crate but `mvp-db`.

Cargo cannot restrict an edge to one module, so the test also scans `mvp-engine` and `mvp-contract` for any `mvp_custody::` path outside `mvp_custody::vocab` (risk R-1) and `mvp-api` for `::lifecycle`
imports. `dependency-cruiser` enforces rule 5.

### §30.4 `mvp-kernel`

**Functional responsibilities.**
- Newtype IDs from one macro (RC-01); value objects with fallible constructors (RC-02): `Notes`,
  `SrNumber`, `ShortCode`, `SlaHours`, `RequestNumber`; `AssigneeSpec` and `RequestStatus`.

**Encapsulated knowledge.** Every length, character-set and range rule for a primitive input. No
other crate repeats "1 to 2000 characters, non-blank".

**Subdomain.** Generic; near-zero volatility.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| out (consumed by all) | every crate | Model | ID and value-object types | `define_id!`, `Notes::parse`, `AssigneeSpec`, `RequestStatus` |

**Change vectors.** A new input rule changes one constructor. A new request status adds a variant, and
the exhaustive matches elsewhere fail to compile (intended).

**Boundary contract checks.**
- Boundary table for every constructor (0, 1, max, max+1, whitespace-only, control characters).
- `serde` round trip, and rejection of invalid strings, for each ID and value object.
- A size guard fails above 1,500 lines (risk R-2).

### §30.5 `mvp-contract`

**Functional responsibilities.**
- Request and response DTOs, `IssueDto`, `SimTraceDto`, `GraphDto`/`PlanDto` and the OpenAPI schema
  (`utoipa`, served at `/api/openapi.json`).
- `ErrorCode`: the one table of §11 and §24.2 codes with `http_status()` and a `NON_RETRYABLE` subset
  (the activity list of §7.3).
- Everything that crosses between `api` and `worker` (F-4): `temporal::{NAMESPACE, TASK_QUEUE,
  workflow, update, signal, query}` name constants, the id builders `request_workflow_id`,
  `lane_workflow_id`, and the payload structs `StartInput`, `LaneInput`, `RefValues`, `ActInput`, `LaneEvent`,
  `ProjectionDto` (`LaneInput`, `RefValues`, `LaneEvent`: part 11 §41.12).

**Encapsulated knowledge.** Wire spelling: every name, status code and JSON shape that two processes
must agree on.

**Subdomain.** Supporting; volatility follows additions to the API (additive by rule).

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-kernel`, `mvp-custody::vocab` | Model | IDs; vocabulary enums (same `snake_case` spelling as the DB enums) | `vocab` derives `ToSchema` under the `schema` feature |
| out | `mvp-api`, `mvp-worker` | Contract | Temporal names and payloads | constants and structs above |
| out | `web` | Contract | OpenAPI schema → generated TypeScript client | committed `openapi.json` snapshot |

**Change vectors.** Renaming a Temporal update edits one constant. A new error code adds a variant and
the exhaustive `ApiError` mapping (§33.2) fails to compile until handled.

**Boundary contract checks.**
- Golden JSON for every Temporal payload, deserialised by the current code (additive-only rule, RC-14).
- A committed `openapi.json` snapshot; CI regenerates and diffs it.
- Every `ErrorCode` has a unique wire string and an HTTP status.

### §30.6 `mvp-engine` (Core)

**Functional responsibilities.**
- `compile(&DefinitionRows) -> Result<CompiledPlan, CompileError>`: typed node configuration (RC-03), adjacency and dominance, the pinned embed tree (D-42) into `templates`.
- `validate(&CompiledPlan) -> Vec<Issue>` (§22, V-01…V-69, incl. template interface, binding, pin, nesting, data flow).
- `settle(&CompiledPlan, &State, Event) -> (State, Vec<Effect>)` (§23.3), including exception-handler
  routing (part 6 §27) and `single` embeds run inline as call frames (`EnterEmbed`/`ExitEmbed`).
- `simulate(&CompiledPlan, SimInput) -> SimTrace` (§24.3); `catalogue()`: node kinds, ports and platform variables; `scope_table`, `dataflow` (part 11 §41.12). Module `fx`: FX `parse`, `check`, `cost`, `eval`; V-57…V-69 (part 12 §42.4, §42.15).

**Encapsulated knowledge.** The graph semantics: connector rules, tokens, joins, per-item join policies,
embed call frames, template interfaces, expression evaluation and typing, handler selection by catch filter. Nothing about how custody
works: the engine reads a lane only as a `LaneMilestone` (F-2).

**Subdomain.** Core; high volatility.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-custody::vocab` | Contract | `LaneMilestone`, `ExceptionPhase`, `ResolutionKind::allowed_in` | the `vocab` module only |
| in | `mvp-kernel` | Model | IDs, `AssigneeSpec`, `RequestStatus` | types |
| out | `mvp-db` | Model | `DefinitionRows` (flat structs mirroring the `wf_*` tables) | owned by the engine; the repository maps SQL to it |
| out | `mvp-worker`, `mvp-api` | Contract | the four functions; `CompiledPlan` (serialised in history) | `PLAN_FORMAT` version (RC-15) |

**Change vectors.** New node kind: engine (`NodeKind`, compile, validate, settle), one editor renderer,
and a `wf_nodes` column with its `mvp-db` mapping. New V-rule or per-item join policy: engine only.

**Boundary contract checks.**
- Dependency fitness (§30.3); `HashMap`/`HashSet` banned (`disallowed-types`).
- Determinism: `settle` applied twice to equal inputs gives equal outputs.
- Replay corpus (F-5): golden `CompiledPlan` files for every `PLAN_FORMAT`, plus recorded
  event-to-effect traces, run against current code.
- `validate` and `compile` never panic on fuzzed row sets (a property test).

### §30.7 `mvp-custody` (Core)

**Functional responsibilities.**
- `transition(state, action, &ctx) -> Result<Transition, CustodyError>`: the **only** custody
  transition table (F-1), including `crossed_receipt` and `LaneMilestone::of`.
- `available_actions(state, &ctx) -> Vec<CustodyAction>`: what is legal in a state. `/actions` combines
  it with `mvp-authz`.
- `resolve(resolution, &ResolutionContext) -> Result<ResolutionPlan, CustodyError>`: the platform
  effects of each resolution (part 6 §27 matrix): case patch, item and asset patch, milestone,
  optional supplementary lane. Pure; the repository applies the plan.
- `describe_effects(kind, phase)`: the "effect card" that `/plan` serves to the inspector.
- Module `vocab`: the small shared language (§33.2): `LiveState`, `CustodyState`, `ExceptionPhase`,
  `ExceptionType`, `Resolution`, `ResolutionKind`, `LaneMilestone`, `LaneOutcome`.

**Encapsulated knowledge.** What each custody state allows, what each resolution does to custody,
items and assets, and which resolutions fit which phase.

**Subdomain.** Core; high volatility.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| out (vocab) | `mvp-engine`, `mvp-contract` | Contract | `LaneMilestone`, `ExceptionPhase`, `ResolutionKind::allowed_in` | `vocab` module; rule enforced by §30.3 |
| out | `mvp-worker` | Contract | `transition`, `resolve` results | pure functions |
| out | `mvp-api` | Contract | `available_actions`, `describe_effects` | pure functions |
| out | `mvp-db` | Model | `CustodyState`, `Resolution` to map columns | `FromRow` mapping lives in `mvp-db` |

**Change vectors.** New custody state or outcome (for example `quarantined`): this crate, the `mvp-db`
mapping, migration and report classification; the UI changes nothing. New resolution kind:
`ResolutionKind`, `resolve`, `allowed_in`, engine catalogue.

**Boundary contract checks.**
- Exhaustive table test: every `(LiveState × CustodyAction)` cell is `Ok` or a named `Err`.
- Truth tables for `crossed_receipt`, `LaneMilestone::of` and `ExceptionPhase::of`.
- Resolution matrix: every `(ResolutionKind × ExceptionPhase)` cell agrees with V-35 through
  `allowed_in`.

### §30.8 `mvp-authz`

**Functional responsibilities.**
- `authorize(&ActorFacts, &Rule, &Scope, now) -> Result<ActingAs, Denied>` for §8: team at a
  snapshot step, owner or delegate in window, submitter, any reviewer, admin capability and scope,
  organisation isolation. `Denied` distinguishes `NotVisible` (404) from `NotEligible` (403).
- `separation_of_duties(actor, &SodContext) -> Result<(), Denied>` for D-6.

**Encapsulated knowledge.** Who may do what: membership liveness, delegate windows, cluster-type
matching, the self-approval set.

**Subdomain.** Supporting; medium volatility.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-db::actor_facts` | Model | `ActorFacts` (built by `load_actor_facts`) | struct defined here; the loader is in `mvp-db` |
| out | `mvp-api` | Contract | `authorize` before an engine call | pure function |
| out | `mvp-worker` | Contract | `authorize` inside the activity transaction; `separation_of_duties` in validators | pure functions |

**Change vectors.** Delegate windows or a new actor class: this crate and `load_actor_facts` only (F-3).

**Boundary contract checks.**
- A decision-table test that walks every row of §8 with an allowed and a denied `ActorFacts`.
- Window edges (`active_from`, `active_until` inclusive and exclusive) and cross-organisation facts
  (`NotVisible`).
- Property: removing a membership never turns a denial into an allow.

### §30.9 `mvp-db`

**Functional responsibilities.** Repositories grouped by domain; each module splits into reads and a
`lifecycle` submodule.

Modules: `identity`, `actor_facts` (`load_actor_facts(&mut Tx, actor, org)`) and `definitions` (system workflows, `wf_templates`, `wf_embed_pins`), `copy`,
`reports` (Supporting); `plans`, `custody` (Core persistence, including `persist_settlement`);
`audit`, `archive`, `evidence` (Generic).

`definition_copy` (F-6) is the one copy path:

```rust
pub enum CopySource { Version(DefinitionId), Rows(DefinitionRows) } // rows come from an import file
pub struct AssigneeMapping(BTreeMap<AssigneeRef, Mapped>);
pub enum AssigneeRef { VendorCluster { ordinal: u8, level: Level }, AgencyTeam { name: TeamName, level: Level } }
pub enum Mapped { Team(TeamId), Unresolved } // Unresolved becomes a V-42 placeholder

pub async fn definition_copy(
    tx: &mut Transaction<'_, Postgres>,
    source: CopySource,
    mapping: &AssigneeMapping,
    target: DraftTarget,
) -> Result<DefinitionId, DbError>;
```

Draft-from-version uses `AssigneeMapping::identity()`; import passes `CopySource::Rows`; clone passes
the wizard's mapping. One function knows the `wf_*` insert order and key remapping.

**Encapsulated knowledge.** SQL, the schema, constraints, grants, append-only enforcement, FK order,
the mapping of nullable column bags to engine and custody enums.

**Subdomain.** Mixed (see table). One crate, by decision (minor finding M-1).

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-engine` | Model | `DefinitionRows` | engine-owned structs |
| in | `mvp-custody`, `mvp-authz` | Model | enums to map; `ActorFacts` | types |
| out | `mvp-worker` | Model | `lifecycle` functions taking `&mut Tx` | function signatures |
| out | `mvp-api` | Model | read functions and read models | function signatures |

**Change vectors.** A new `wf_*` table changes `definitions` and `copy` only. A new ledger event type
changes the enum migration, the mapping and the view classification.

**Boundary contract checks.** Real-database tests (repository mocks are not allowed):
- Organisation predicate on every read (IDOR negative per module).
- `UPDATE` on `audit_log`, `custody_events`, `request_variable_changes` fails (append-only grants).
- `definition_copy` round trip equal modulo IDs; a failed copy leaves no rows.
- View classification: every ledger event type and `LaneOutcome` lands in exactly one summary bucket.

### §30.10 `mvp-worker` (binary)

**Functional responsibilities.** `BreakglassRequestWorkflow` and `LaneWorkflow` shells (validator,
`settle`, activity call); activity shells (transaction, `op_ref`, `authorize`, one `lifecycle` call,
audit row); the history archiver loop (D-18).

**Encapsulated knowledge.** The Temporal SDK surface (macros, deterministic `select!`, timers,
continue-as-new, retry policy). No business rule.

**Subdomain.** Generic; low volatility.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-engine`, `mvp-custody`, `mvp-authz` | Contract | pure functions | signatures |
| in | `mvp-db` | Model | `lifecycle` functions | signatures |
| in | `mvp-contract` | Contract | names and payloads | constants and structs |
| out | `mvp-api` (through Temporal) | Contract | updates, signals, queries | `mvp-contract` only |

**Change vectors.** SDK upgrade: this crate only. A new update: `mvp-contract` constant and payload,
then the shell here.

**Boundary contract checks.**
- The names registered with the SDK equal the `mvp-contract` constants (needed when a macro requires
  a literal; risk R-3).
- Every activity returns the stored result on a repeated `op_ref`; the replay corpus runs through the
  real workflow shells; no `lifecycle` function is reachable from workflow code.

### §30.11 `mvp-api` (binary)

**Functional responsibilities.**
- axum routes, session, Origin and idempotency middleware, the `ApiError` mapping.
- `EngineGateway` (the port to the engine) and `TemporalGateway` (the adapter): `submit`, `act`,
  `lane_update`, `admin_reset`, `projection`, `health`. Routes receive a generic `AppState<G:
  EngineGateway>`.
- `workflow_admin` service: definition CRUD, validate, simulate (in-process `mvp-engine`, no Temporal),
  publish, diff, export, import, clone preview and clone.
- `plan_view`: assembles `PlanDto` from plan rows, including exception bands, interrupts and effect
  cards (F-8).

**Encapsulated knowledge.** HTTP: status codes, envelopes, cookies, multipart, OpenAPI registration,
and how a Temporal rejection becomes an `ErrorCode`.

**Subdomain.** Supporting (`workflow_admin`, routes); Generic (`gateway`).

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| out | Temporal (to `mvp-worker`) | Contract | updates, signals, queries | `EngineGateway` + `mvp-contract` |
| in | `mvp-authz` | Contract | pre-checks (404/403) | `authorize` |
| in | `mvp-custody` | Contract | `/actions`, effect cards | `available_actions`, `describe_effects` |
| in | `mvp-engine` | Contract | validate and simulate endpoints | `validate`, `simulate` |
| in | `mvp-db` | Model | read models | read functions only |
| out | `web` | Contract | HTTP + OpenAPI | `mvp-contract` |

**Change vectors.** New route: handler, DTO in `mvp-contract`, OpenAPI entry. New engine call: one
trait method plus its adapter body.

**Boundary contract checks.**
- A fake `EngineGateway` drives a test per route: each `ErrorCode` maps to its documented status and
  envelope.
- The Temporal adapter against the dev server: a rejected update becomes its `ErrorCode` (409), an
  unreachable engine becomes 503.
- Every route is in the OpenAPI schema; every mutating route has a missing-`X-Idempotency-Token`
  negative; no `lifecycle` import (§30.3).

### §30.12 `web`

**Functional responsibilities.**
- `features/editor`: canvas, palette, inspector, rule builder, tabs (§24.1). Palette and ports come
  from the DTO `catalogue`; it renders validation issues and never computes them.
- `features/request`: renders `PlanDto` as given (nodes, lanes, exception bands, Interrupts band,
  effect cards, `display_label`). `features/admin`: administration, reports, asset history.
- `lib/api`: client generated from `/api/openapi.json`, committed so jsdom tests run offline.

**Encapsulated knowledge.** Layout, interaction and accessibility. **No custody state, transition or
exception rule.**

**Subdomain.** Supporting.

**Integration contracts.**

| Direction | With | Strength | Shared | Contract |
|---|---|---|---|---|
| in | `mvp-api` | Contract | HTTP and the generated client | OpenAPI |
| between features | none | — | — | features share only `lib/api` and `lib/ui` |

**Change vectors.** A new custody state or exception kind needs **no web change**: the server sends
labels, available actions and effect cards. A new node kind needs one inspector renderer, registered
by kind.

**Boundary contract checks.**
- Generated-client drift: regenerate, then `git diff --exit-code`.
- Fixture `PlanDto` tests render every handler and interrupt shape.
- ESLint `no-restricted-syntax` forbids custody state and exception type literals in `features/**`.

---

## §31 Coupling assessment

### §31.1 Reading the table

High strength = intrusive or functional; low = model or contract; "(was …)" is the first design.
Distance: Low = one crate; High = across crates; Highest = across processes or time (the rule treats
High and Highest alike). Volatility is from §30.1. **Balanced?** rates the "after" design.

### §31.2 Integrations

| # | Integration | Strength | Distance | Volatility | Balanced? | Action |
|---|---|---|---|---|---|---|
| I-1 | engine ↔ custody (`LaneMilestone`, `vocab`) | Low: contract (was functional: the join read `CaseView.state`) | High | High (Core, Core) | Yes | F-2. The engine reads lanes only through `LaneMilestone` |
| I-2 | engine internals (`compile`, `validate`, `settle`, `simulate`) | High: functional (one `CompiledPlan`) | Low (one crate) | High | Yes | Keep one crate; never split `validate` from `settle` |
| I-3 | engine ↔ worker host (`settle` calls; `CompiledPlan` in history) | Low: contract (was model, unversioned) | Highest (replay across deploys) | High | Yes | F-5. `PLAN_FORMAT`, additive serde, replay corpus. Balanced only because of the version; fragile if skipped |
| I-4 | worker ↔ db (activities → `lifecycle`) | Low: model | High | Medium | Yes | RC-12. `persist_settlement` breadth accepted (M-3) |
| I-5 | api ↔ worker through Temporal | Low: contract (was functional: string names by convention) | Highest (processes) | High | Yes | F-4. Names and payloads in `mvp-contract` |
| I-6 | api ↔ db | Low: model | High | Medium | Yes | api uses reads only; `lifecycle` import forbidden (D-3) |
| I-7 | api ↔ custody (`/actions`, effect cards) | Low: contract (was functional: SQL and UI re-derived) | High | High | Yes | F-1, F-8 |
| I-8 | custody ↔ db (columns ↔ enums) | Low: model (was functional: SQL guards held the transition table) | High | High | Yes | SQL guards check only `version` and `state = from`; mapping at the repository edge |
| I-9 | authz ↔ api/worker (`ActorFacts`) | Low: model | High | Medium | Yes | F-3. One loader, one `authorize` |
| I-10 | web ↔ api | Low: contract (was functional: UI re-derived states) | Highest | Medium | Yes | F-1, F-8. Lint bans state literals |
| I-11 | editor ↔ engine `validate`/`simulate` | Low: contract (`IssueDto`, `SimTraceDto`) | Highest | High | Yes | The UI renders issues and never reimplements rules (part 4 §22) |
| I-12 | cloning, import, draft ↔ definition tables | High: functional (knows every `wf_*` table) | Low (`db::copy`) | Medium | Yes | F-6. Part 6 adds three tables; one place to extend |
| I-13 | reporting views ↔ custody ledger | High: functional (view SQL reads event meaning) | Low (same crate, same migrations) | Medium | Yes | View classification test (§30.9) |
| I-14 | archive ↔ Temporal history | Low: contract (SDK history API, opaque bytes) | Highest | Low | Yes | None; replay tests read the archive |
| I-15 | workflow projection ↔ DB lifecycle columns | Low: model (same `ProjectStatus` effect writes both) | Highest | Medium | Yes | M-2. `reconcile` detects drift |

### §31.3 Commentary

- **Every strong coupling is short-distance** (I-2, I-12, I-13); every long-distance coupling is weak.
  The first design violated this six times (the "was" notes in I-1, I-3, I-5, I-7, I-8, I-10); each is
  now a finding in §32.
- **The Core pair is deliberately decoupled** (I-1). Both are volatile, so a model or functional link
  would make every custody change an engine change. `vocab` is the only shared place and holds types
  and a few total functions.
- **The long-lived couplings are the dangerous ones** (I-3, I-14): a process boundary costs a redeploy;
  a persisted format costs a migration plus replay. They carry explicit versions.
- **I-9 and I-15** duplicate *data*, not *logic*, which is why they are accepted.
- **Change cost fell.** A new custody state touches `mvp-custody` and the `mvp-db` mapping (was: validator,
  SQL guards, join, UI, views, projection); a renamed Temporal update touches `mvp-contract` only.

---

## §32 Modularity review findings

**Result: no Critical issue remains.** Eight Significant findings were found in the design as written
in parts 1–5; each is resolved by §30 and §33. Three Minor findings are accepted with a revisit
condition. Verification is by the ST-30 fitness tests (part 7).

### F-1 Custody transition knowledge is duplicated (Significant)
- **Knowledge leaked:** which actions are legal in which custody state, and what counts as having
  crossed receipt. It existed in the child validator (`plan_for_state`), the SQL guards, the
  `crossed_receipt` predicate used by the join and projection, and the UI action buttons.
- **Dimensions:** functional strength; Highest distance (Rust, SQL, TypeScript); High volatility
  (Core).
- **Consequence:** adding `returned_damaged` or `quarantined` meant editing four places that could
  silently disagree (a button for an action the validator rejects).
- **Resolution:** one `mvp_custody::transition` and one `crossed_receipt`. SQL guards check only
  optimistic `version` and `state = from`. `/actions` is server-driven from `available_actions` plus
  `authorize`; the UI renders the list and infers nothing.

### F-2 The engine's per-item join reads custody states (Significant)
- **Knowledge leaked:** the meaning of custody states, because `CaseView.state` was an input to the
  join predicate (D-13).
- **Dimensions:** functional strength; High distance; High volatility on both sides.
- **Consequence:** quarantine, supplementary lanes and handler outcomes would each force an engine
  change, and a custody refactor could silently change when a request becomes `credential_issued`.
- **Resolution:** the `LaneMilestone` contract (§33.2): the engine's `per_item_verdict` is a function of
  milestones, lane origins and the join policy. Supplementary lanes never affect `issued`; they only
  hold `completed`.

### F-3 Authorization is coded three times (Significant)
- **Knowledge leaked:** team liveness, delegate windows, cluster-type matching and the self-approval
  set, in API SQL, the workflow validator and the activity re-check.
- **Dimensions:** functional strength; Highest distance; Medium volatility.
- **Consequence:** a delegate-window fix applied to two of three leaves a silent privilege gap.
- **Resolution:** `mvp-authz::authorize` over `ActorFacts`, and one `load_actor_facts`. The API
  calls it before the update (404/403); the activity calls the same function with facts loaded inside
  its transaction (TOCTOU, D-5). Validators keep only state rules and call `separation_of_duties`.

### F-4 Temporal names and payloads are plain strings (Significant)
- **Knowledge leaked:** update, signal and query names, workflow ID formats and payload shapes, agreed
  only by convention between `api` and `worker`.
- **Dimensions:** functional strength (implicit contract); Highest distance (two processes); High
  volatility (every new action adds a name).
- **Consequence:** a rename compiles, then fails at runtime as an unknown update or an ignored signal.
- **Resolution:** constants, ID builders and payload structs in `mvp-contract`; `mvp-api` never
  depends on `mvp-worker`; golden payload tests and the registered-name test (§30.5, §30.10).

### F-5 The serialised `CompiledPlan` couples history to future engine versions (Significant)
- **Knowledge leaked:** the exact shape of `CompiledPlan`, because it is loaded by activity and so
  recorded in Temporal history and the archive.
- **Dimensions:** unversioned model strength (effectively functional); Highest distance (time); High volatility.
- **Consequence:** a harmless engine refactor makes old histories fail to replay, blocking every open request.
- **Resolution:** `PLAN_FORMAT: u16`, additive-only serde, readers kept for earlier formats, and a
  replay corpus (RC-15). Engine behaviour changes still use `patched()` (§23.3).

### F-6 Three definition-copy paths (Significant)
- **Knowledge leaked:** the `wf_*` table graph: insert order, key remapping, expression trees, ports,
  positions. Draft-from-version, import and clone (part 6 §28) each needed it.
- **Dimensions:** functional strength over three call sites; Medium volatility, raised by part 6's new tables.
- **Consequence:** each new table is added three times; a clone drops handlers while a draft keeps them.
- **Resolution:** one `definition_copy(source, AssigneeMapping, target)` in `mvp-db::copy` (§30.9).

### F-7 `mvp-domain` has low cohesion (Significant)
- **Knowledge leaked:** none specific; IDs, custody rules, authorization rules and wire types shared one
  crate that every crate depended on, with the volatility of its worst part (High).
- **Consequence:** every Core change rebuilt and retested everything, and "just put it in domain"
  became the default home for rules that belong to custody or authz.
- **Resolution:** retired and split by change reason into `mvp-kernel`, `mvp-custody`, `mvp-authz`,
  `mvp-engine` and `mvp-contract` (§30.2). Risk R-2 watches for kernel drifting back.

### F-8 Exception and interrupt semantics are hidden (Significant)
- **Knowledge leaked:** how exceptions route, what resolutions do, what interrupts exist. Only the lane
  workflow knew; the UI hard-coded how to draw them.
- **Dimensions:** functional strength; Highest distance (server, workflow, browser); High
  volatility.
- **Consequence:** users could not see an exception's flow; each new resolution needed a UI change that could drift.
- **Resolution:** drawn handlers and locked interrupts (part 6 §27) plus a `PlanDto` that carries
  exception bands, the Interrupts band and server-built effect cards (`describe_effects`). The
  request page renders what it is given.

### Minor findings (accepted)

| ID | Finding | Why accepted | Revisit when |
|---|---|---|---|
| M-1 | A single `mvp-db` crate holds all repositories (mixed subdomains, one compile unit) | Splitting adds crates with no change-isolation gain while one team owns it; modules are already by domain | The crate passes about 15,000 lines, or incremental test builds exceed the team's budget, or a second team owns a domain: split into `mvp-db-core` plus per-domain crates |
| M-2 | Status and ordinal are held in workflow state and in DB columns (§2.6) | Both are written from the same `ProjectStatus` effect; `reconcile` reports drift; D-3 keeps one writer | `reconcile` finds any divergence in CI, or a second projection field is added |
| M-3 | `persist_settlement` writes instances, edges, variables, rule evaluations, notifications, status and audit in one activity | The one-transaction rule (§2.6) requires it; each effect has its own writer function under 50 lines | `Effect` passes about 20 variants, or a requirement appears for partial settlement |

---

## §33 Rust conventions (normative for the implementer)

### §33.1 Convention table

| ID | Convention | Enforced by |
|---|---|---|
| RC-01 | **Newtypes for every ID**, generated by one macro. No bare `Uuid`, `String` or `i64` identifying an entity in any signature | Review; `clippy::disallowed-types` for `uuid::Uuid` outside `mvp-kernel` and `mvp-db` row structs |
| RC-02 | **Parse, don't validate.** `Notes` (1..2000 characters, non-blank, stored trimmed), `SrNumber` (1..100), `ShortCode` (`^[A-Z0-9]{1,10}$`, upper-cased), `SlaHours` (1..720), `RequestNumber`. Fields private; the constructor is fallible; `serde` goes through `try_from` | Kernel boundary tests (§30.4) |
| RC-03 | **Illegal states unrepresentable.** Enums carry their data: `CustodyState`, `Resolution`, `RequestStatus`, `Effect`, `NodeKind`. The nullable column bag of `wf_nodes` is mapped to `NodeKind` configuration enums at `compile()`; `(state, state_before_exception)` is mapped to `CustodyState` at the repository edge. A row that fits no variant is a `CompileError`/`DbError`, not a panic | Review; `compile` property test |
| RC-04 | **Exhaustive `match`; no `_` arm** on business enums (`RequestStatus`, `CustodyState`, `LiveState`, `ExceptionPhase`, `Resolution`, `ResolutionKind`, `Effect`, `NodeKind`, `LaneMilestone`, `ErrorCode`). Adding a variant must break the build | `clippy::wildcard_enum_match_arm = deny`; `allow_attributes_without_reason = deny` |
| RC-05 | **Errors.** `thiserror` enum per library crate (`KernelError`, `CompileError`, `CustodyError`, `Denied`, `DbError`, `GatewayError`). `anyhow` only in the `main` of the two binaries. Exactly one `From<DomainError> for ApiError`, exhaustive, mapping to the §11 codes | Review; grep gate: `anyhow` appears only in `main.rs` |
| RC-06 | **`#[must_use]`** on `settle`, `validate`, `route`, `authorize`, `transition`, `resolve`, `Effect` and every `Result`-bearing pure function | `clippy::must_use_candidate = warn` |
| RC-07 | **Decimals** use `rust_decimal` (`numeric(18,4)`); no `f32`/`f64` for a business value | `clippy::disallowed-types` |
| RC-08 | **Deterministic collections.** `BTreeMap`/`BTreeSet` only in `mvp-engine`, `mvp-custody` and workflow code | `disallowed-types` for `HashMap`/`HashSet` in those crates |
| RC-09 | **`pub(crate)` by default.** Each crate's public surface is listed in its `lib.rs` documentation | `unreachable_pub = warn` |
| RC-10 | **`#![forbid(unsafe_code)]`** in every crate root and `unsafe_code = "forbid"` in workspace lints | compiler |
| RC-11 | **No `unwrap`, `expect`, `panic!`, `todo!`, `unimplemented!`, `dbg!`** outside `cfg(test)`. Tests may use them (`allow-*-in-tests` in `clippy.toml`) | `clippy` deny set (below) |
| RC-12 | **Repositories take `&mut Transaction<'_, Postgres>`**; one transaction per activity; no function in `lifecycle` takes a pool | Review; signature grep |
| RC-13 | **Nothing blocks in async code.** `zstd` compression, SHA-256 over evidence or history bytes, and password hashing run in `spawn_blocking`. Workflow code does none of this | Review; `clippy::disallowed-methods` for `zstd::encode_all` and `Sha256::digest` in async modules |
| RC-14 | **serde.** `#[serde(deny_unknown_fields)]` on every inbound **HTTP** DTO. Temporal payloads and `CompiledPlan` do **not** deny unknown fields: new fields are `#[serde(default)]` (additive only), nothing is renamed, removed or retyped. `rename_all = "snake_case"` on enums | Golden payload tests (§30.5) |
| RC-15 | **`CompiledPlan` format.** `PLAN_FORMAT: u16` is written into the plan. Additive changes keep the number. A breaking change bumps it, keeps a reader for the earlier format, and adds a golden file to the replay corpus. Adding `CompiledPlan.templates` and the embed nodes bumps `PLAN_FORMAT` once | Replay corpus test (§30.6) |
| RC-16 | **Temporal names are `const`s in `mvp-contract`**; the worker and the gateway use them and no string literal repeats one | Registered-name test (§30.10); literal grep |
| RC-17 | **Time and identity are inputs.** Pure crates and workflow code take `now` as an argument and never call `Utc::now`, `SystemTime::now` or `Uuid::now_v7`. Instance identity inside `State` is a deterministic `InstanceSeq`; `mvp-db` mints the UUID at insert | `disallowed-methods`; `mint` feature check (§30.3) |
| RC-18 | **Immutability and ownership.** Pure functions take borrowed inputs and return new values (`settle` returns a new `State`); no `Rc<RefCell<…>>`; a loaded `CompiledPlan` is shared as `Arc<CompiledPlan>` | Review |
| RC-19 | **Size.** Functions under 50 lines, files under 800 | `clippy::too_many_lines` with threshold 50; the file-length check in ST-30 |
| RC-20 | **Sandboxed code only in `mvp-fn-runner`.** `wasmtime*`, `rquickjs*`, `boa*`, `quickjs*`, `rhai`, `evalexpr`, `cel*`, `zen-*` are banned in every other crate; FX lives in `mvp-engine::fx` with no new crate (part 12 §42.15) | `mvp-fitness` banned list; `cargo deny check advisories` |

Workspace lints (every crate sets `[lints] workspace = true`): `unsafe_code = forbid`,
`unreachable_pub = warn`; clippy deny set `unwrap_used expect_used panic todo unimplemented dbg_macro
wildcard_enum_match_arm allow_attributes_without_reason`; warn set `must_use_candidate too_many_lines`.

### §33.2 Type sketches

These fix shapes, not final code. Part 6 §27 owns handler-related variants.

**IDs and a value object (`mvp-kernel`).**

```rust
macro_rules! define_id {
    ($(#[$meta:meta])* $name:ident) => {
        $(#[$meta])*
        #[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash, Serialize, Deserialize)]
        #[serde(transparent)]
        pub struct $name(uuid::Uuid);
        impl $name {
            #[must_use] pub const fn from_uuid(raw: uuid::Uuid) -> Self { Self(raw) }
            #[cfg(feature = "mint")]
            #[must_use] pub fn mint() -> Self { Self(uuid::Uuid::now_v7()) }
        }
    };
}
define_id!(RequestId); define_id!(LaneId); define_id!(ExceptionId); define_id!(AssetId);

#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(try_from = "String")]
pub struct Notes(String);

impl Notes {
    pub const MAX_CHARS: usize = 2000;
    pub fn parse(raw: &str) -> Result<Self, KernelError> {
        let text = raw.trim();
        match text.chars().count() {
            0 => Err(KernelError::Blank { field: "notes" }),
            n if n > Self::MAX_CHARS => Err(KernelError::TooLong { field: "notes", max: Self::MAX_CHARS, actual: n }),
            _ => Ok(Self(text.to_owned())),
        }
    }
}
// TryFrom<String> delegates to Notes::parse (serde try_from).
```

**Custody vocabulary (`mvp-custody::vocab`).**

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LiveState {
    AwaitingPreparation, Prepared, HandoverPending,
    InRequesterCustody, ReturnPending, ReceivedByAgency,
}

/// `before` is data, so "exception without a previous state" cannot exist.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum CustodyState {
    Live(LiveState),
    ExceptionOpen { before: LiveState },
    ResolvedException { before: LiveState },
    ReturnedToInventory,
    Cancelled,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ExceptionPhase { PreIssuance, InCustody, Return }

impl ExceptionPhase {
    #[must_use]
    pub const fn of(state: LiveState) -> Self {
        match state {
            LiveState::AwaitingPreparation | LiveState::Prepared | LiveState::HandoverPending => Self::PreIssuance,
            LiveState::InRequesterCustody => Self::InCustody,
            LiveState::ReturnPending | LiveState::ReceivedByAgency => Self::Return,
        }
    }
}
```

**Resolution (`vocab`).** The designer picks the kind; runtime data (substitute, resume state) is added on apply.

```rust
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum Resolution {
    Resume { to: LiveState },
    CancelFalseReport,
    Replace { substitute: AssetId },
    WriteOff,
    Waive,
    QuarantineReturn,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum ResolutionKind { Resume, CancelFalseReport, Replace, WriteOff, Waive, QuarantineReturn }

// Resolution::kind(&self) -> ResolutionKind is a total match (omitted).
impl ResolutionKind {
    /// Static part of V-35, shared by `validate` (design time) and `resolve` (run time).
    #[must_use]
    pub const fn allowed_in(self, phase: ExceptionPhase) -> bool {
        match self {
            Self::Replace => !matches!(phase, ExceptionPhase::Return),
            Self::QuarantineReturn => !matches!(phase, ExceptionPhase::PreIssuance),
            Self::Resume | Self::CancelFalseReport | Self::WriteOff | Self::Waive => true,
        }
    }
}
```

**Lane milestone (`vocab`): the only lane fact the engine reads (F-2).**

```rust
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum LaneMilestone {
    InProgress,
    CrossedReceipt,
    Excluded { reason: ExclusionReason },   // written off or cancelled before receipt
    Replaced { successor: LaneId },
    Terminal { outcome: LaneOutcome },      // reached only after receipt
}

impl LaneMilestone {
    #[must_use]
    pub const fn is_live(&self) -> bool {
        !matches!(self, Self::Excluded { .. } | Self::Replaced { .. })
    }
    // counts_as_issued(): CrossedReceipt | Terminal => true, all other variants false.
}
```

**Engine effect (`mvp-engine`).**

```rust
#[must_use]
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum Effect {
    EnterNode { node: NodeKey, lane: Option<LaneId> },
    LeaveNode { instance: InstanceSeq, port: PortKey },
    TakeEdge { edge: EdgeKey, lane: Option<LaneId> },
    SetVariable { var: VarName, lane: Option<LaneId>, value: Value, source: ChangeSource },
    StartLanes { items: Vec<ItemId>, origin: LaneOrigin },
    EnterEmbed { call: EmbedCallId, node: NodeKey, lane: Option<LaneId> },  ExitEmbed { call: EmbedCallId, outcome: PortKey },
    StartHandler { lane: LaneId, exception: ExceptionId, handler: HandlerKey },
    ApplyResolution { lane: LaneId, exception: ExceptionId, resolution: Resolution },
    ArmTimer { timer: TimerKey, after: Duration },
    ProjectStatus(RequestStatus),
    EndRun { kind: RunEnd },
    // plus RecordRule, Notify, CancelTimer, CancelInstance (part 4 §23.3)
}

#[must_use] pub fn settle(plan: &CompiledPlan, state: &State, event: Event) -> (State, Vec<Effect>);
#[must_use] pub fn validate(plan: &CompiledPlan) -> Vec<Issue>;
pub enum NodeKind { Embed { template: TemplateId, definition: DefinitionId, cardinality: Cardinality } /* , … */ }
pub enum Cardinality { Single, PerItem(PerItemPolicy) } // PerItemPolicy: join policy, bounds, flags
pub struct CompiledPlan { pub root: CompiledTemplate, pub templates: BTreeMap<DefinitionId, CompiledTemplate> }
```

**Engine gateway (`mvp-api`).** The API's only view of the engine.

```rust
pub trait EngineGateway: Send + Sync + 'static {
    fn submit(&self, cmd: SubmitCommand)
        -> impl Future<Output = Result<SubmitAck, GatewayError>> + Send;
    fn act(&self, request: RequestId, cmd: ActCommand)
        -> impl Future<Output = Result<ActAck, GatewayError>> + Send;
    fn lane_update(&self, lane: LaneId, cmd: LaneCommand)
        -> impl Future<Output = Result<LaneAck, GatewayError>> + Send;
    // admin_reset, projection and health follow the same shape.
}

#[derive(Debug, thiserror::Error)]
pub enum GatewayError {
    #[error("update rejected: {0:?}")] Rejected(ErrorCode), // the validator's code travels with it
    #[error("engine unreachable")] Unavailable,
    #[error("engine call timed out")] TimedOut,
}
```

All command and ack types are in `mvp-contract`. `TemporalGateway` is the only implementation.

**One error mapping (`mvp-api`).**

```rust
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error(transparent)] Kernel(#[from] KernelError),
    #[error(transparent)] Custody(#[from] CustodyError),
    #[error(transparent)] Authz(#[from] Denied),
    #[error(transparent)] Db(#[from] DbError),
    #[error(transparent)] Gateway(#[from] GatewayError),
}

impl From<DomainError> for ApiError {
    fn from(err: DomainError) -> Self {
        match &err {
            DomainError::Kernel(e) => ApiError::validation(e.field(), e.to_string()),
            DomainError::Custody(e) => ApiError::new(e.code(), e.to_string()), // exhaustive in CustodyError::code
            DomainError::Authz(Denied::NotVisible) => ApiError::new(ErrorCode::NotFound, "not found"),
            DomainError::Authz(Denied::NotEligible) => ApiError::new(ErrorCode::NotAuthorized, "not eligible"),
            DomainError::Db(e) => ApiError::from_db(e), // exhaustive over DbError
            DomainError::Gateway(GatewayError::Rejected(code)) => ApiError::new(*code, err.to_string()),
            DomainError::Gateway(GatewayError::Unavailable | GatewayError::TimedOut) =>
                ApiError::new(ErrorCode::EngineUnavailable, "engine unavailable"),
        }
    }
}
```

No `_` arm appears in any of these matches (RC-04).

---

## §34 Unresolved risks

| ID | Risk | Why it remains | Revisit when |
|---|---|---|---|
| R-1 | The `vocab`-only rule for `mvp-engine` and `mvp-contract` is enforced by a source scan, not by the compiler | Cargo restricts edges per crate, not per module | The scan is bypassed or relaxed once: extract `vocab` as the crate `mvp-vocab` (a mechanical change) |
| R-2 | `mvp-kernel` could regrow into `mvp-domain` (types plus rules) | It holds `AssigneeSpec` and `RequestStatus`, which are shared by three or more crates | It passes 1,500 lines, or any function there makes a business decision: move the shared enums to `mvp-vocab` and keep only IDs and value objects |
| R-3 | `temporalio-sdk` macros may require string literals for names, or may expand code that conflicts with `forbid(unsafe_code)` | The SDK is 1.1.x and the macros are unverified (part 1 §7 SDK note) | First compile of the worker: keep the registered-name test; if a macro needs `unsafe_code` relaxed, downgrade to `deny` for `mvp-worker` only and record the reason |
| R-4 | The replay corpus starts synthetic | No real histories exist yet | After the first full demo run: add archived histories from `workflow_history_archive` |
| R-5 | The distance model assumes one team owns every crate | True for the MVP | A second team owns a crate: re-rate §31 with that boundary as Highest distance and split `mvp-db` (M-1) |

---

## Supersession by this part

| Clause | Replaced by |
|---|---|
| Part 1 §5.1, last paragraph: crates `mvp-api`, `mvp-worker`, `mvp-domain`, `mvp-db` "share the domain types and DB access" | The §30.2 module map and §30.3 dependency rules. `mvp-domain` is retired |
| Part 3 §19 prompt, STACK line "crates: mvp-domain … mvp-db … mvp-worker … mvp-api" | The §30.2 crate list. Prompt rules 16–19 (part 3 edits) point here for module boundaries and the §33 conventions |
| Part 1 §7.1 validator "actor ≠ submitter" and part 1 §8 eligibility evaluated per call site | `mvp_authz::authorize` and `separation_of_duties` (F-3, §30.8) |
| Part 1 §7.2 `plan_for_state`, §9.1 `crossed_receipt`, §11 `GET /api/requests/:id/actions` | One `mvp_custody::transition`, `available_actions` and `LaneMilestone::of` (F-1, §30.7) |
| Part 4 §23.3 `State` keyed by `NodeInstanceId` | Keyed by deterministic `InstanceSeq`; `persist_settlement` maps it to the `NodeInstanceId` UUID at insert (RC-17) |
| Part 4 §23.3 `compile(def_rows)` | `compile(&DefinitionRows)`, where `DefinitionRows` is owned by `mvp-engine` (§30.6) |
| Part 4 §23.3 join "reads lane states" and §6/§7 `CaseView.crossed_receipt` | Per-item verdict over `LaneMilestone` (F-2, §33.2) |

Unchanged: decisions D-1…D-27, the §22 semantics, the §8 matrix, all timers, and every AC in parts 2, 5 and 7.
