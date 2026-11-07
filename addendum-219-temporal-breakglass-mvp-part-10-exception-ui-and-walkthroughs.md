# Addendum 219 (part 10) — Exception and cloning UI, unhappy-path walkthroughs, demo state tables and stories

**Date:** 2026-11-08
**Status:** PROPOSED. Specification only; nothing is implemented.
**Parts:**
- Part 1 `addendum-219-temporal-breakglass-mvp.md` (§0–§12).
- Part 2 `…-part-2-stories.md` (§14, ST-0–ST-14).
- Part 3 `…-part-3-verification-and-prompt.md` (§13, §15–§19).
- Part 4 `…-part-4-workflow-definition-and-editor.md` (§20–§26).
- Part 5 `…-part-5-editor-stories.md` (§14 continued, ST-15–ST-24).
- Part 6 `…-part-6-fulfilment-exceptions-cloning-coverage.md` (§27–§29).
- Part 7 `…-part-7-exception-cloning-coverage-stories.md` (ST-25–ST-30).
- Part 8 `…-part-8-module-architecture-and-rust-conventions.md` (§30–§34).
- Part 9 `…-part-9-exception-runtime-and-cloning-detail.md` (§35–§37, ST-31: runtime sequences, payloads, error matrix, cloning algorithm, coverage CI).
- **This part (§38–§40, ST-32).**
- Appendix A `…-appendix-a-illustrative-example.md`.

**Why this part exists.** Parts 6 and 7 specify the fulfilment-exception handlers, the permutation
matrix PM-01…PM-31 and workflow cloning, but only in prose and tables. The user asked to "flesh out in
detail more" these unhappy paths (an item missing and impossible to issue, lost after issuance,
damaged after issuance, and their permutations) and the cloning of a workflow from an existing system's
version, so that the demo illustrates the engine. This part adds what an implementer and a presenter
still lack: the **screens** (§38, including the handler mini-canvas in §38.5), one **walkthrough per
permutation row** (§39), and the
**step-by-step state tables** of S10, S11 and S12 with their video checkpoints (§40). Part 6 and
part 9 are not repeated: runtime sequences, activities, payloads, races, the error matrix, the cloning
algorithm and the coverage CI are in part 9.

**Precedence.** This part elaborates parts 6 and 7 and adds no engine behaviour. Where it conflicts with
part 6, **part 6 wins**. Inconsistencies found while writing it are listed in §40.5, each resolved in
the part named there. Screens render server data as given (part 8 §30.12): no screen holds a custody rule,
an effect text or a routing rule.

---

## §38 UI specification for exceptions and cloning

### §38.1 Editor: the Handlers tab

Route `/admin/templates/[id]/workflow/edit` (the template editor of `standard_custody_lane`, a `subject=item` template), tab **Handlers** (part 6 §27.10).

```
┌ standard_custody_lane v1 · draft r12 · autosaved 14:02:11 ─────── [Validate] [Simulate] [Publish] ┐
│ Template "standard_custody_lane"  [ Lane ] [ Handlers ] [ Interface ] [ Eligibility ]             │
├ Handlers (drag the grip to reorder) ──────────┬ Mini-canvas · H3 Damaged in custody ───────────────┤
│ ⠿ 1  H1  Before issuance            ▸        │ (exception_start)─caught─▶(investigate)             │
│ ⠿ 2  H3  Damaged in custody   ◀ selected      │   Asset Store L1 · 8 h · escalate Ops Leads L2      │
│ ⠿ 3  H2  Lost in custody            ▸        │   ├ false_report ─▶ [resolution: cancel_false_report]│
│ ⠿ 4  H4  Problems at return         ▸        │   ├ replacement ─▶ (approval: Ops Leads L2 team)    │
│ [lock] 5  fallback (pinned last)             │   │                  └ approve ─▶ [resolution: replace]│
│ [+ Add handler]   [Delete] (off on fallback) │   ├ waiver ─▶ (approval: Owner class) ─▶ [waive]     │
│ ⚠ HANDLER_SHADOWED: none                     │   └ write_off ─▶ (approval: Owner class) ─▶ [write_off]│
├ Catch filter · H3 ────────────────────────────┴────────────────────────────────────────────────────┤
│ Exception types [damaged ×] [identity_mismatch ×] [+ add]      empty = all types                    │
│ Phases          [in_custody ×] [+ add]        Severities [any]                                      │
│ Reads as: type damaged ∈ {damaged, identity_mismatch}; phase in_custody ∈ {in_custody}; severity any│
├ Coverage grid · severity [any v] ── H1 caught · H3* shadowed · fb fallback-only · /// impossible · ~ varies
│                    pre_issuance   in_custody    return                                              │
│ missing                H1             H2           H4                                               │
│ unavailable            H1             H2           H4                                               │
│ damaged                H1             H3*          H4                                               │
│ wrong_item             H1             H2           H4                                               │
│ identity_mismatch      H1             H3*          H4                                               │
│ quantity_mismatch      H1             H2           H4                                               │
│ handover_disputed      H1             ///          ///                                              │
│ receipt_disputed       H1             ///          ///                                              │
│ return_disputed        ///            ///          H4                                               │
│ other                  H1             H2           H4                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Grid cell states** (colour is never the only cue: each state also has a text mark and a tooltip).

| State | Mark | Colour | Meaning and tooltip |
|---|---|---|---|
| caught | `H1` | green | exactly one non-fallback handler matches |
| shadowed | `H3*` | amber | the winner is a non-fallback handler and a later one also matches ("H2 also matches; H3 wins on order"). Informational |
| fallback-only | `fb` | blue-grey | only the fallback matches. Allowed, shown so nothing is silently uncovered |
| impossible | `///` | hatched grey | part 6 §27.2 pair. Not focusable by click, never raises V-32 |
| varies | `~` | purple outline | the winner depends on severity (selector on "any") |

The mini-canvas pane above is a **summary** of the handler's graph. The canvas itself (layout, node
visuals, drag rules, editing, Diff and the read-only request view) is specified in §38.5.

Clicking a cell focuses the winner's mini-canvas. Reordering re-evaluates the grid at once, client-side,
from the DTO `catalogue`; the server's V-32 result is shown in the Validation tab and is authoritative.

**Handler editing.**
- Tabs `H1…H4` and `fallback` also exist as a keyboard list (§38.4). Reordering by drag uses the grip;
  the keyboard alternative is "Move up" and "Move down" buttons. A drop below the fallback is refused.
- The catch-filter form writes `wf_exception_handler_filters` rows: one chip per value, none = all.
  The "Reads as" line is the same sentence format as `rendered_match` (part 6 §27.7).
- Inspector fields. `investigate`: assignee team picker, `sla_hours`, `escalate_to_team`, the six port
  check boxes, "Allow info hold". `resolution_approval`: assignee kind (**Owner class** or **Team**),
  team picker, optional `sla_hours`. `resolution`: kind, and for `resume` the mode (`state_before_exception`
  or `investigator_choice`). Team pickers list **live agency teams of the system's organisation** only
  (V-24); vendor and `pa` clusters do not appear. A node carrying `assignee_unresolved` shows the red chip
  "Choose team (was …)" in place of the picker value.
- The **Interrupts** tab shows three padlocked nodes (`expire`, `tombstone_cancel`, `admin_reset`) with no
  handles; selecting one shows a read-only sentence of what it cancels.

**Effect cards.** Selecting a `resolution` node shows one card per phase the handler can catch. The text
below is the **expected output of `GET /api/admin/platform/exception-effects`** (`describe_effects`, part 8
F-8). The web renders `cardText` verbatim and holds no copy.

| Kind | Phase · class | Card text |
|---|---|---|
| `resume` | any | Resume (any phase): the case resumes at the state you choose; the asset is unchanged; the stage SLA restarts. |
| `cancel_false_report` | any | False report: the case resumes where it was; nothing else changes. |
| `waive` | any | Waive: the case resumes where it was; the deviation is recorded; the expected return condition is unchanged. |
| `replace` | before issuance · any | Replace (before issuance): original lane ends and is excluded; asset retired 'lost_replaced' if lost or damaged, otherwise released; replacement lane starts on the substitute. |
| `replace` | after issuance · lost | Replace (after issuance, lost): original lane ends; asset retired 'lost_replaced'; issuance stands; supplementary lane starts on substitute. |
| `replace` | after issuance · damaged | Replace (after issuance, damaged): original resumes in custody expecting a damaged return; substitute gets a supplementary lane; issuance stands. |
| `replace` | after issuance · other | Replace (after issuance, other): original resumes in custody, return condition stays good; substitute gets a supplementary lane; issuance stands. |
| `replace` | return | Not permitted at return (V-35). |
| `write_off` | before issuance | Write off (before issuance): item excluded; asset retired 'written_off'; lane ends; if no lane is left the request closes unfulfilled. |
| `write_off` | after issuance or return | Write off (after issuance): asset retired 'written_off'; hold released; issuance stands; the item stays counted. |
| `quarantine_return` | after issuance · damaged or other | Quarantine on return: the item stays with the requester; the return is expected damaged; the asset will be quarantined, not restocked. |
| `quarantine_return` | return | Quarantine on return: the asset is quarantined for 'damaged_on_return', its hold released, and it is not restocked. |
| `quarantine_return` | before issuance | Not permitted before issuance (V-35). |

**Simulation injector** (Simulation tab, part 4 §24.3 and part 6 §27.10).

```
┌ Inject an exception ───────────────────────────────────────────────────────────────────────────┐
│ Items 4 (token, device, device, device)   Lane [L3 HT-01 v]   Type [missing v]  Severity [high v]│
│ Phase  (•) force: [pre_issuance v] state [awaiting_preparation v]   ( ) derive from a script step│
│ Substitutes available [x] on (off = NO_SUBSTITUTE)    Replacement count already on lane [0 v]    │
│ [Inject]   Route: H1 Before issuance (ordinal 1)  · candidates tested: H1 matched                │
│ Then act on: [investigate v] outcome [replacement v] substitute [HT-02 v]  [Add step] [Run]      │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The injector sends `{kind:'exception', lane_no, type, severity, at_state, substitutes_available,
replacement_count}` and then `act` steps. "Phase forcing" picks `at_state` from the states of §27.2 for the
chosen phase, so the pair (type, phase) can always be built; an impossible pair is disabled. A refused
outcome shows its error copy (§38.4) in the trace. The panel shows `RouteDecision` with every candidate.
Nothing is written (AC-22.1.1 unchanged).

### §38.2 Request page: exception bands, supplementary lanes, quarantine and action panel

Route `/requests/[id]`, graph from `GET /api/requests/:id/plan` (part 6 §27.10). Illustrative request
with four original lanes and one supplementary lane (PM-20 waiver then PM-23 on SP-01; PM-10; PM-11).

```
HRIS-2026-0099 · HRIS · [credential_issued]      Run 1 v        ⚠ Attention required: 2 custody exceptions
┌ Graph ──────────────────────────────────────────────────────────────────────────────────────────┐
│ … submit ▸ approvals ▸ owner ▸ wait ▸ ┌ items (per-item embed) ────────────────────────────────┐│
│ ┌ L1 SP-01 (device) · returned damaged ───────────────────────────────────────────────────────┐ ││
│ │ prepare✓ ▸ handover✓ ▸ receipt✓ ▸ return✓ ▸ verify✓(damaged) ▸ restock ✗  [Q] Quarantined    │ ││
│ │ ┌ Exception #1 · in custody · H3 Damaged in custody ── RESOLVED · Waived ───────────────────┐│ ││
│ │ │ route: H3 (ordinal 2 of 5) ▾  skipped: H1 (phase pre_issuance ≠ in_custody)              ││ ││
│ │ │ exc_start✓ ▸ investigate✓ Grace 12m ▸ approval✓ Carol ▸ resolution: waive  [Waived]      ││ ││
│ │ └───────────────────────────────────────────────────────────────────────────────────────────┘│ ││
│ │ ┌ Exception #2 · return · H4 Problems at return ── RESOLVED · Quarantined ──────────────────┐│ ││
│ │ │ reported by verifier (automatic) ▸ investigate✓ Claire ▸ resolution: quarantine_return    ││ ││
│ │ └───────────────────────────────────────────────────────────────────────────────────────────┘│ ││
│ └──────────────────────────────────────────────────────────────────────────────────────────────┘ ││
│ ┌ L2 SP-02 ▸ collapsed: delivered ┐ ┌ L3 HT-01 · in custody ──────────────────────────────────┐ ││
│ │                                 │ │ ┌ Exception #1 · in custody · H2 ── RESOLVED · Replaced ┐│ ││
│ └─────────────────────────────────┘ │ │ … approval✓ Carol ▸ resolution: replace              ││ ││
│ ┌ L4 LT-01 · EXCEPTION OPEN ⚠ ───────┐ │ └──────────────────────────────────────────────────────┘│ ││
│ │ ┌ Exception #1 · in custody · H2 ─ AWAITING INFO ┐         ┆ supplements                       ││
│ │ │ investigate● Frank ▸ task loss_report (Bob) ○  │  ┌┄ L5 HT-02 (supplementary) ┄┄┄┄┄┄┄┄┄┐ ││
│ │ └────────────────────────────────────────────────┘  ┆ prepare● ▸ handover ○ …            ┆ ││
│ └─────────────────────────────────────────────────────┘ └┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘ ││
│ Join (issue): live=4 crossed=4 excluded=0 pending_replacements=0 -> Issue (1 supplementary ignored)││
│ Interrupts: [lock] expire   [lock] tombstone_cancel   [lock] admin_reset       (none fired)       ││
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
Legend: ● active (blue)  ✓ done (green)  ○ pending (grey)  ⚠ exception open (red dashed)  [Q] quarantined
        dashed border = supplementary lane   ▾ expand route reasons   struck-through = excluded
```

**Rendering rules.**
- Bands are numbered by `case_exception_no` ("Exception #1", "#2"), in order, per lane. Collapsed
  bands keep their header (phase, route chip, status, resolution badge).
- The band header colour follows the **phase** and always carries the phase word: pre_issuance amber,
  in_custody red-orange, return purple. A band whose exception is open adds the red dashed ⚠ style.
- **Resolution terminal badges:** `Resumed` (green), `False report` (grey), `Waived` (teal),
  `Replaced` (green, arrow), `Written off` (dark grey), `Quarantined` (red-brown), `Cancelled by reset`
  (grey, struck-through, with a "cancelled by" edge from `admin_reset`).
- **Quarantined marker** `[Q] Quarantined (damaged_on_return)` sits on the lane's last node and links to
  `/assets/[id]`. **Supplementary lane:** dashed border, "supplements" edge from the original lane,
  caption "supplementary · depth 1". A replacement lane uses the existing "replaced by" edge.
- **Join node** trace text: `live=L crossed=C excluded=X pending_replacements=P -> Wait|Issue|AllExcluded`
  followed by "(N supplementary ignored)" when N>0. **Interrupts band** is always drawn; a fired node is
  highlighted with the text "fired 14:07 by Greg Global" and a "cancelled by" edge to each instance.

**Action panel** (server-driven from `GET /api/requests/:id/actions` and `GET /api/exceptions/:id`;
`offeredPorts[{port, enabled, disabledReason}]`). Eligible actors see the buttons; others see one
read-only line "Waiting for {assignee}".

| Node (status) | Who sees buttons | Buttons and forms (required fields **bold**) | Disabled reasons (exact copy) |
|---|---|---|---|
| `investigate` (`open`) | the node's team | "Start investigating" (**notes**) | none |
| `investigate` (`investigating`) | the node's team | "Mark as recovered" (**resume state** select, notes); "Report as false alarm" (**notes**); "Propose replacement" (**substitute** select, **notes**); "Propose write-off" (**notes**); "Propose waiver" (**deviation notes**); "Quarantine on return" (**notes**); "Ask for information" (**notes**) | Replacement: "Replacement is not allowed after issuance for this system" · "Replacement limit reached (1/1)" · "No substitute available". Write-off is absent when the per-item embed disallows it |
| `investigate` (`awaiting_information`) | the node's team | "Continue investigating" (**notes**) | All port buttons: "On hold for information. Continue investigating first" |
| `resolution_approval` | owner class, or the named team | "Approve" ; "Reject" (**notes**) | "You investigated this exception, so you cannot approve it" · "You reported this exception" · "You are the requester" |
| `task` (`loss_report`) | the submitter (or `submitter_team`) | "Complete loss report" (**done** tick, notes) | non-assignees: "Waiting for the loss report from Bob Lim" |
| report (any live case) | submitter, stage team | "Report a problem": **type**, **severity**, **description**, evidence | "A problem is already open for this item" |

The replacement select lists only candidate substitutes from `GET /api/exceptions/:id`; a stale choice
returns `SUBSTITUTE_INVALID` (§38.4).

**Issuance and returns panel** gains per-item columns **Lane origin** (initial, replacement,
supplementary), **Replaces / replaced by** (asset codes), **Return condition** (good, damaged, —),
**Quarantined at** and outcome `returned_damaged`. The summary adds **Returned damaged** and
**Replaced after issuance**. The CSV has the same columns (AC-27.9.1).

**Timeline entries** (text templates; actors and assets substituted).

| Event | Text |
|---|---|
| `custody.exception_reported` | "Bob Lim reported SP-01 as damaged (medium): {description}" |
| `custody.exception_routed` | "Routed to H3 Damaged in custody (rule 2 of 5). Skipped: H1, phase pre_issuance ≠ in_custody" |
| investigator starts | "Grace Neo started investigating SP-01" |
| info hold / resume | "Grace Neo asked for information: {notes}" · "Grace Neo continued investigating" |
| `sla_breached` | "Investigation of SP-01 is overdue (8 h). Escalated to Ops Leads L2" |
| proposal | "Grace Neo proposed a replacement of SP-01 with SP-03" (write-off, waiver likewise) |
| approve / reject | "Frank Chua approved the replacement" · "Carol Sim rejected the proposal: {notes}. Investigation continues" |
| `replacement_authorized` | "Replacement applied: SP-01 stays with the requester (expected back damaged); SP-03 held for a supplementary lane" |
| `asset_written_off` · `asset_recovered` · `waiver_granted` · `exception_cancelled` | "LT-01 written off; issuance stands" · "HT-01 recovered; resumed at preparation" · "Waiver granted: {deviation}" · "Report judged a false alarm" |
| `return_condition_recorded` · `asset_quarantined` | "Claire Wong recorded SP-01 as damaged on return" · "SP-01 quarantined (damaged on return); not restocked" |
| `custody.supplementary_lane_started` · `workflow.interrupt_fired` | "Supplementary lane started for SP-03" · "Admin reset fired; 1 exception cancelled" |

**Dashboards** ("Needs my action", part 1 §12). Rows sort overdue first; `Overdue {n}h` is red;
`Escalated` is amber and shown only to the escalation team with "from Asset Store L1".

| Row text | Shown to | Node |
|---|---|---|
| "Investigate SP-01 (damaged) · HRIS-2026-0099 · H3" | the investigate team (Grace, Leo for H3) | `investigate` |
| "Approve replacement of SP-01 with SP-03" | the approver (Frank for H3 team; Carol, Dan or Emma for owner class) | `resolution_approval` |
| "Complete loss report for LT-01" | Bob (or the `submitter_team`) | `task loss_report` |
| "Quarantine or write off SP-01 (damaged return)" | Ops Reviewers L1 (H4) | `investigate` with the `quarantine` port offered |

### §38.3 Create-system wizard and clone flow

`/admin/systems/new` steps: **1 Details · 2 Owners and PA · 3 Workflow · 4 Confirm**. Step 3 has four
sub-steps. The same sub-steps 2–4 open from "New draft from another system's version" on
`/admin/systems/[id]/workflow` (the target is then fixed and step W1 is skipped).

```
Workflow 1 of 4 · Choose a start
 ( ) Start blank (starter template)   start ▸ one approval ▸ owner ▸ per-item embed ▸ end; teams to be chosen
 (•) Clone from system version        copy the whole graph, handlers included
 ( ) Import file                      upload a workflow file (max 1 MiB)
                                                                   [Back]  [Next]

Workflow 2 of 4 · Pick the source
 Organisation [Org Aurora v]   System [HRIS v]
 ┌ Version ─ Status ──── Published ─────────── By ───────── Fingerprint ┐
 │ (•) v1    published   2026-11-08 09:30      Greg Global   3fa9c1e2   │
 │ ( ) v2    draft (not available)                                       │
 └───────────────────────────────────────────────────────────────────────┘
 Only published or retired versions are listed. An organisation admin sees only this system's organisation.

Workflow 3 of 4 · Map the teams        Source: HRIS v1 (Org Aurora) → PAYROLL    5 teams to map
 Source team            Used by                 Suggested target        Match          Your choice
 Alpha Leads L2         alpha_l2                Beta Leads L2           Position match [Beta Leads L2 v]
 Alpha Engineers L1     alpha_l2 (escalation)   none                    Needs a choice [Choose a team v] [Leave unresolved]
 Ops Reviewers L1       ops_l1, verify_return…  Ops Reviewers L1        Same team      [Ops Reviewers L1 v]
 Ops Leads L2           sec_review, H1, H2 …    Ops Leads L2            Same team      [Ops Leads L2 v]
 Asset Store L1         log_review, prepare …   Asset Store L1          Same team      [Asset Store L1 v]
 ⚠ 1 team will be left unresolved. You can choose it in the editor, but you cannot simulate or publish until you do.

Workflow 4 of 4 · Review                                         [Back]  [Create draft]
 Copies 41 nodes, 52 connections, 5 handlers, 12 variables, 2 embeds. Not copied: owners, assets, requests, history, templates (unless "Also clone these templates" is ticked).
 Platform nodes (interrupts, expired) are re-created from the current template.
 ⚠ PAYROLL has no live asset of type "token" (V-43). You can still publish.
 ⚠ 1 team unresolved (V-42). The draft opens with a red "Choose team (was Alpha Engineers L1 …)" chip.
```

(The counts are read from the preview response; the numbers above are illustrative.)

**Confidence badges** map the `basis` of part 6 §28.5: default (same organisation) = "Same team" green;
`name_level` = "Name match" blue; `ordinal_level` = "Position match" amber; none = "Needs a choice" red.
Unresolved rows are highlighted red with an icon and the text "Unresolved"; they are never colour-only.
**"Leave unresolved" is offered only for a team with no default** (§28.4: a defaulted reference is
remapped, not left unresolved). The target dropdown lists only live teams of the right type
(vendor or agency, never `pa`) in the target organisation; for another organisation it starts empty.

**Result.** `201` opens `/admin/systems/[id]/workflow/edit` on the new draft with the lineage chip
"Cloned from HRIS v1, Org Aurora", the Validation tab listing V-42 (one error per node and slot) and
V-43, and **Simulate** and **Publish** disabled with the tooltip "Choose the unresolved teams first".
**Error states** show the copy of §38.4 in an inline alert at the step where it arises:
`DRAFT_EXISTS` (Next on W2, with the link "Open the existing draft"), `CLONE_SOURCE_NOT_PUBLISHED`,
`CLONE_SAME_SYSTEM` (offers "New draft from a version of this system"), `CLONE_CROSS_ORGANISATION` (API only: an
`org_admin`'s W2 Organisation select offers just the target's organisation, D-38), `CLONE_MAPPING_INVALID` (the
offending rows are outlined and each row's message is its `fields` path), and `404` (neutral page).

### §38.4 Accessibility, copy and testid contract

**Keyboard paths.**
- Handlers list: `Tab` enters the list; `Up`/`Down` move; `Alt+Up`/`Alt+Down` reorder (or the Move buttons);
  `Enter` opens the mini-canvas; `Delete` on a selected non-fallback handler asks for confirmation.
- Coverage grid is a `role="grid"` with roving tabindex; `Enter` on a cell focuses its handler.
- Handler nodes use the part 4 §24.1 ARIA label form, for example "Investigate: Asset Store L1, 4 ports,
  1 unconnected"; "Connect *port* to…" is the pointer-free alternative to dragging.
- Clone wizard: every step is a form reachable by `Tab`; the mapping table is a real `<table>` with
  one `<select>` per row labelled "Target team for {source team}". The step heading takes focus on change
  and an `aria-live="polite"` region announces "Step 3 of 4: Map the teams".
- Bands are `role="group"` with `aria-label` "Exception 1, in custody, route H3, resolved, waived".
  Resolution and phase are text, not colour alone. Disabled action buttons use `aria-disabled` and keep
  their reason in `aria-describedby`, so a screen reader reads the reason.

**Error copy** (plain language, sentence case, says what to do; SGDS writing style). The code is shown in
small print under the message. Unknown codes fall back to "Something went wrong. Try again, or contact
support with the code below".

| Code | Message |
|---|---|
| `NO_SUBSTITUTE` | No substitute is available. Write the item off, or wait until one is free. |
| `SUBSTITUTE_INVALID` | That substitute can no longer be used. Pick another from the list. |
| `REPLACEMENT_LIMIT` | The replacement limit for this item has been reached. |
| `POST_ISSUANCE_REPLACEMENT_DISABLED` | This system does not allow replacement after issuance. |
| `RESUME_STATE_INVALID` | The case cannot resume at that state. Choose one of the states listed. |
| `INVALID_EXCEPTION_STATUS` | This problem is on hold or not yet started. Continue investigating first. |
| `OUTCOME_NOT_ALLOWED` | That outcome is not available for this problem. |
| `NODE_NOT_ACTIVE` | This step has already moved on. Refresh to see the latest. |
| `STALE_STATE` | Someone else changed this item. Refresh and try again. |
| `ASSET_IN_USE` | The substitute was taken by another request. Pick another. |
| `ASSET_QUARANTINED` | This asset is quarantined and cannot be requested. |
| `SOD_VIOLATION` | You cannot approve a problem you reported or investigated. |
| `CUSTODY_IN_PROGRESS` | Some items have already been handed over. Use "Force" with notes to reset the rest. |
| `PLATFORM_NODE_LOCKED` / `PLATFORM_TERMINAL_LOCKED` | This platform step is fixed and cannot be changed. |
| `HANDLER_SHADOWED` (warning) | Earlier handlers already catch everything this one catches. |
| V-32 (error) | Some combinations of problem type and phase are not caught. Add a handler or fix the fallback. |
| `ASSIGNEE_UNRESOLVED` (V-42) | Choose a team (was {source team}). |
| `CLONE_MAPPING_INVALID` | Pick a live team of this organisation, of the right kind. |
| `CLONE_SOURCE_NOT_PUBLISHED` | Only published or retired versions can be copied. |
| `CLONE_SAME_SYSTEM` | This version belongs to this system. Use "New draft from a version". |
| `CLONE_CROSS_ORGANISATION` | Only a global admin can copy a workflow from another organisation. |
| `DRAFT_EXISTS` | This system already has a draft. Open it, or discard it first. |
| `DEFINITION_INVALID` | Fix the errors in the Validation tab, then try again. |
| `STALE_DRAFT` | Someone else changed this draft. Reload to continue. |
| `IDEMPOTENCY_TOKEN_USED` | That action was already submitted. Refresh to see the result. |
| `ENGINE_UNAVAILABLE` | The workflow service is not available. Try again shortly. |
| 403 other | You do not have permission to do this. |
| 404 | We cannot find that page. (Identical for a missing and a hidden object.) |

**`data-testid` contract** (`{component-kebab}__{element-role}`, addendum-005; repeated elements share one id
and carry `data-*` attributes for scoping).

| Component | Test ids |
|---|---|
| `handler-list` | `__item` (`data-handler-key`), `__grip`, `__move-up`, `__move-down`, `__add`, `__delete` |
| `catch-filter` | `__types`, `__phases`, `__severities`, `__reads-as` |
| `coverage-grid` | `__cell` (`data-type`, `data-phase`, `data-state`), `__severity` |
| `effect-card` | `__text` (`data-kind`, `data-phase`) |
| `sim-injector` | `__lane`, `__type`, `__phase`, `__substitutes`, `__replacement-count`, `__inject`, `__route` |
| `exception-band` | `__header` (`data-exception-no`, `data-phase`), `__route-chip`, `__skipped`, `__resolution-badge` |
| `lane-card` | `__supplementary`, `__quarantined` |
| `join-trace`, `interrupts-band` | `__text`; `__node` (`data-kind`, `data-fired`), `__cancelled-by` |
| `exception-actions` | `__port` (`data-port`, `aria-disabled`), `__reason`, `__approve`, `__reject`, `__notes` |
| `embed-node`, `embed-inspector`, `template-library` | `embed-node__open`; `embed-inspector__template`, `embed-inspector__upgrade`; `template-library__row` (`data-template-key`) |
| `clone-wizard` | `__start-choice`, `__source-system`, `__source-version`, `__mapping-row` (`data-ref`), `__badge`, `__target`, `__unresolved`, `__next`, `__back`, `__create`, `__error` |
| `handler-canvas` (§38.5) | `__node` (`data-kind`, `data-node-key`, `data-diff`), `__port` (`data-port`), `__edge` (`data-from-port`, `data-taken`, `data-diff`), `__refused`, `__zoom-in`, `__zoom-out`, `__fit`, `__auto-layout`, `__minimap`, `__expand`, `__close-expand` |
| `handler-palette` (§38.5) | `__item` (`data-kind`) |
| `handler-add-dialog` (§38.5) | `__label`, `__key`, `__filter`, `__create`, `__cancel`, `__error` |
| `handler-list` (additions, §38.5) | `__duplicate`, `__diff-badge` (`data-diff`) |
| `exception-band` (addition, §38.5) | `__toggle-graph` |

### §38.5 Handler mini-canvas: viewing and editing

This section extends part 4 §24.1 (the top-level editor canvas) to the handlers of part 6 §27.3. Where
§24.1 already states a behaviour (autosave, `STALE_DRAFT`, undo depth, inspector, "Connect *port* to…"),
it applies unchanged and is not repeated. It adds no node kind, port, validation code or engine
behaviour. Every rule below is a **client convenience**: the server's `validate` (part 4 §22, part 6
§27.6) remains the control.

#### §38.5.1 Canvas model

- **One React Flow instance per handler.** The right pane of the Handlers tab (§38.1) holds a separate
  `@xyflow/react` 12 instance in its own `ReactFlowProvider`, showing only the nodes whose `handler_id`
  is the selected handler. It is **not** a sub-flow of the embed node, so no edge can be drawn to a
  node outside the handler (V-39 holds by construction, and the `wf_edges` trigger of part 6 §27.7 is
  the backstop). The custom node-type registry is shared with the main canvas.
- **Coordinates.** Handler nodes use `wf_nodes.pos_x/pos_y` (part 4 §23.1) in the handler's own
  coordinate space, with `exception_start` conventionally at the origin. Positions are part of the draft,
  are autosaved, and are copied by cloning (part 6 §28). Seeded HRIS v1 handlers ship laid out.
- **Size and controls.** The pane is at least 640 × 360 px and resizes with the window. Controls: zoom in
  and out (25 %–200 %), **Fit** (fits every node of the handler), **Auto-layout** (dagre, left to right,
  preserving pinned positions, as §24.1), a **Mini-map** toggle, and **Expand**, which opens the same
  canvas in a full-width overlay with a focus trap; `Esc` or "Back to handlers" closes it and returns focus
  to the handler's row in the list.
- **Selection.** Selecting a node or edge opens the shared inspector (§38.1 fields). The palette and the
  inspector follow the canvas into the Expand overlay.

```
┌ Mini-canvas · H3 Damaged in custody ───────────────── [−] 100% [+] [Fit] [Auto-layout] [Mini-map] [Expand] ┐
│                                                                                                            │
│ ╭ [flag] Start [lock] ─╮    ┌ [search] Investigate ────┐                                                   │
│ │ damaged +1           │    │ Asset Store L1 · 8 h     │                                                   │
│ │ in_custody · any sev │    │ ⤴ Ops Leads L2 · [hold]  │                                                   │
│ ╰────────── caught ○───┼───▶● in                       │                                                   │
│                             │         false_report ○───┼─────────────────────────────────▶ ■ False report  │
│                             │          replacement ○───┼─▶● Approval: team    approve ○─▶ ■ Replace        │
│                             │                          │                      reject  ○─↺ Investigate      │
│                             │               waiver ○───┼─▶● Approval: owner   approve ○─▶ ■ Waive          │
│                             │                          │                      reject  ○─↺ Investigate      │
│                             │            write_off ○───┼─▶● Approval: owner   approve ○─▶ ■ Write off      │
│                             │                          │                      reject  ○─↺ Investigate      │
│                             │           quarantine ○───┼─────────────────────────────────▶ ■ Quarantine    │
│                             └──────────────────────────┘                                                   │
│                                                                                                            │
└ ○ out-port  ● in-connector  ■ resolution  ↺ reject goes back to Investigate (V-37)  [lock] fixed ──────────┘
```

The `reject` edges are drawn as curved back-edges to the `investigate` node's in-connector; the wireframe
abbreviates them as `↺`. "Approval: team" is Ops Leads L2; "owner" is the owner class.

#### §38.5.2 Node visuals

Every node shows its kind as an icon **and** a text label; every port shows its key as text next to the
handle. Colour is never the only cue (§38.4).

| Kind | Shape and header | Body | Ports (label, colour) | Badges |
|---|---|---|---|---|
| `exception_start` | rounded pill, flag icon, "Start" | the catch filter in short form ("damaged +1 · in_custody · any sev"); the fallback shows "Everything else" | `caught` (grey) | lock: cannot be deleted or copied alone |
| `investigate` | card, magnifier icon, "Investigate" | team chip, SLA badge "8 h", escalation chip "⤴ {team}", hold icon when `allow_info_hold` | `recovered` green · `false_report` grey · `replacement` green with arrow · `write_off` dark grey · `waiver` teal · `quarantine` red-brown (only checked ports are drawn) | port count "4 ports, 0 unconnected" |
| `resolution_approval` | card, check-shield icon, "Approval" | "Owner class" or team chip; SLA badge if set | `approve` green · `reject` red | — |
| `resolution` | terminal (square corners, thick border), "■ {kind}" | the first clause of its effect card from the server (§38.1); for `resume`, "at: state before exception" or "at: investigator's choice" | none | colour of the matching §38.2 resolution badge |
| `rule`, `switch`, `set_variable`, `task`, `wait`, `notify` | as part 4 §24.1 | as part 4 §24.1; a `task` assigned to the submitter shows "Submitter" | as part 4 §24.1 | — |

An unresolved assignee (part 6 §28.4) shows the red chip "Choose team (was {source team})" in place of the
team chip, on the node itself as well as in the inspector. A node with a validation error has a red
outline and an error icon whose tooltip lists the issue codes; a warning uses amber and a warning icon.

#### §38.5.3 Connection rules while dragging

`isValidConnection` on a handler canvas applies the part 4 §24.1 rules that still make sense (one edge per
out-port, no self-loop) and refuses the following. A refused drop draws nothing, sends no save request,
and shows the tooltip `handler-canvas__refused` with the text below. The "Connect *port* to…" dropdown
lists only targets that pass these rules.

| # | Refused connection | Mirrors | Tooltip text |
|---|---|---|---|
| 1 | Any edge into `exception_start` | V-33 | The start of a handler cannot have incoming connections. |
| 2 | Any edge out of a `resolution` | V-33 | A resolution ends the handler. |
| 3 | `reject` to anything but an `investigate` node of this handler | V-37 | A rejected proposal goes back to an Investigate step. |
| 4 | An `investigate` port **directly** to a `resolution` of a different kind (pairing table, part 6 §27.3) | V-33 | This outcome can only end in "{kind}". |
| 5 | `replacement`, `write_off` or `waiver` **directly** to a `resolution` | V-34 | This outcome needs an approval first. |
| 6 | Any edge that closes a cycle other than `reject` → `investigate` | V-37 | Only a rejected proposal may loop back. |
| 7 | A `replace` resolution in a handler whose filter can catch `return`, or `quarantine_return` where it can catch `pre_issuance` (on drop and when the filter changes) | V-35 | Not permitted in the phases this handler catches. |

Rules 4 and 5 look only at the direct target. A path through generic nodes (for example
`write_off` → `task loss_report` → approval → `write_off`, H2) is checked by the server alone. Rule 7 is
shown as a node error (§38.5.2), not as a refused drop, when it is caused by a filter change.

#### §38.5.4 Editing actions

- **Palette.** In the Handlers tab the palette offers `investigate`, `resolution_approval`, `resolution`,
  `rule`, `switch`, `set_variable`, `task`, `wait` and `notify`. `exception_start`, `custody_stage` (V-41)
  and every top-level kind are absent. Dropping a `resolution` asks for its kind first.
- **Add handler** (`handler-list__add`) opens `handler-add-dialog`: label (1..80), handler key (pre-filled
  from the label, `^[a-z][a-z0-9_]{0,39}$`, unique in the definition) and **at least one** filter chip, so
  that a new handler cannot silently take over every exception ahead of later handlers. It is inserted with
  the ordinal just above the fallback. It opens with a template laid out automatically:
  `exception_start` → `investigate` (team "Choose team" chip, V-42; ports `recovered` and `false_report`) →
  `resolution: resume` (`state_before_exception`) and `resolution: cancel_false_report`. The draft is
  therefore valid apart from V-42 until a team is chosen.
- **Duplicate handler** (`handler-list__duplicate`, not offered on the fallback) copies the nodes, edges,
  positions and filter rows with new keys (`{key}_copy`, then `_copy2`…), label "Copy of {label}", inserted
  directly after the source. Since the copy has the same filter as its source, the copy is shadowed
  (`HANDLER_SHADOWED`) until its filter or order is changed.
- **Delete.** `Delete` (or the inspector button) removes the selected node with its edges, or the
  selected edge. `exception_start` and platform-locked nodes cannot be deleted (`PLATFORM_NODE_LOCKED` for
  interrupts). Deleting a handler asks for confirmation and names the number of nodes it removes; the
  fallback cannot be deleted (§38.1).
- **Undo and redo** use the single 50-step stack of part 4 §24.1, shared by the main canvas and all
  handler canvases. They cover add, delete, move, connect and disconnect, inspector edits, filter chips,
  reordering, and adding, duplicating and deleting handlers. Undo after switching handler first selects
  the handler it applies to.

#### §38.5.5 Validation focus and Diff

- **Focus.** Clicking an issue in the Validation tab whose node or edge belongs to a handler switches the
  editor to the Handlers tab of the lane template, selects that handler, pans the mini-canvas to the node or edge and selects it.
  V-32 issues select the handler list and the uncovered grid cells instead.
- **Diff** (part 4 §24.1 Diff tab), against any published version:
  - Handler list rows carry a badge `added`, `removed` (struck through, read-only), `changed` or
    `moved n → m` (old and new ordinal), as text plus colour (`handler-list__diff-badge`).
  - The catch-filter form shows added chips underlined and removed chips struck through.
  - The mini-canvas highlights added nodes and edges (green outline, "+"), removed ones as ghosts (grey,
    dashed, "−"), and changed nodes (amber outline, "~"; the inspector lists the changed fields).
  - Handlers are matched across versions by `handler_key`, nodes by `node_key`. The server computes the
    match: `GET /api/admin/workflow-definitions/:defId/diff` (part 4 §24.2) also covers
    `wf_exception_handlers` and their filter rows.

#### §38.5.6 Read-only handler graph on the request page

Each exception band (§38.2) has a toggle **Show handler graph** (`exception-band__toggle-graph`). It
renders the handler's graph from the **pinned** definition with the node components of §38.5.2, read-only
(no palette, no handles, no inspector edits):
- visited nodes carry their instance status, actor and time ("✓ Grace Neo 14:07"); a node visited twice
  (after a `reject`) shows "×2" and lists both instances on hover;
- taken edges are highlighted and untaken edges are greyed (R8, part 4 §20.2);
- the `resolution` that was reached shows the effect card the server sent (AC-32.3.3).

The data comes from `GET /api/requests/:id/plan`: each entry of `exceptionBands[]` (part 6 §27.9) gains
`handlerGraph{nodes[], edges[]}`, taken from the pinned definition, with instance statuses and taken edges
from Postgres. It renders after Temporal history is archived (AC-32.5.6). This adds a field to an existing
response and no engine behaviour.

---

## §39 Unhappy-path walkthroughs (PM-01 … PM-31)

One walkthrough per row of part 6 §27.12, in two tables. Table A gives the preconditions, the persona
actions in order, the handler matched and the node instances created. Table B gives the resulting state.
Every row is an executable test (AC-32.12.1).

**Legend.** States: AP `awaiting_preparation`, PR `prepared`, HP `handover_pending`, IC `in_requester_custody`,
RP `return_pending`, RA `received_by_agency`, EO `exception_open`, RE `resolved_exception`.
Instances: S `exception_start`, I `investigate`, A `resolution_approval`, T `task`, R `resolution(kind)`.
Personas: Bo Bob, Gr Grace, Fr Frank, Ca Carol, Cl Claire, Em Emma, Gg Greg. "takes" = `open → investigating`.
Route: `Hn ord k` is `rendered_match` abbreviated; the stored text is
`Hn <label> (ordinal k): type … ∈ {…}; phase … ∈ {…}; severity …` and the candidates list shows earlier
handlers as skipped with a reason (for example "H1 skipped: phase pre_issuance ≠ in_custody"). Ordinals:
H1=1, H3=2, H2=3, H4=4, fallback=5. "Offer" = `GET /api/exceptions/:id`.

### §39.1 Table A: trigger, route and instances

| PM | Preconditions | Persona actions in order | Route · instances |
|---|---|---|---|
| 01 | HT-01 AP; HT-02 free; depth 0 of 1 | Gr reports HT-01 missing (high) · Fr takes · Fr `replacement` {HT-02} · Ca approves | H1 ord 1 · S, I#1, A, R(replace) |
| 02 | SP-02 AP; no free SP device (R0 holds LT-02, SP-03) | Gr reports SP-02 missing · Fr takes · Fr sees `replacement` disabled · Fr attempts it: 422 `NO_SUBSTITUTE` | H1 ord 1 · S, I#1 (stays active) |
| 03 | HT-02 replaced HT-01 (depth 1 of 1); spare token HT-03 exists (fixture); HT-02 AP | Gr reports HT-02 missing · Fr takes · offer shows `replacement` disabled · attempt: 409 `REPLACEMENT_LIMIT` | H1 ord 1 · S, I#1 |
| 04 | HT-02 HP, depth 1 of 1, no spare (S11a) | Bo disputes handover ("token cracked") · Fr takes · attempt `replacement`: 409 `REPLACEMENT_LIMIT` (reported before `NO_SUBSTITUTE`) | H1 ord 1 (type `handover_disputed`) · S, I#1 |
| 05 | HT-01 PR; HT-02 free | Gr reports HT-01 damaged · Fr takes · `replacement` {HT-02} · Ca approves | H1 ord 1 · S, I#1, A, R(replace) |
| 06 | (a) SP-02 AP missing (S10 step 2); (b) HT-01 PR damaged; (c) HT-02 HP disputed (S11 step 3) | reporter reports · Fr takes · Fr `write_off` · Ca approves | H1 ord 1 · S, I#1, A, R(write_off) |
| 07 | (a) HT-01 AP missing; (b) LT-01 HP `wrong_item` | reporter reports · Fr takes · (a) `recovered` {state_before_exception}; (b) `false_report` | H1 ord 1 · S, I#1, R(resume) or R(cancel_false_report) |
| 08 | LT-01 HP `identity_mismatch` (serial label differs) | Gr reports · Fr takes · Fr `waiver` (deviation notes) · Ca approves; variant: Ca rejects with notes, Fr re-proposes | H1 ord 1 · S, I#1, A, R(waive); reject adds I#2 |
| 09 | LT-01 HP | Bo disputes handover ("not the laptop I asked for") · Fr takes · Fr `recovered` | H1 ord 1 · S, I#1, R(resume) |
| 10 | Request `credential_issued`; LT-01 IC | Bo reports LT-01 missing · Fr takes · Fr `write_off` · Bo completes `loss_report` · Ca approves | H2 ord 3 (H1, H3 skipped) · S, I#1, T(loss_report), A, R(write_off) |
| 11 | HT-01 IC; HT-02 free; depth 0 | Bo reports missing · Fr takes · `replacement` {HT-02} · Ca approves | H2 ord 3 · S, I#1, A, R(replace) |
| 12 | HT-01 IC; no spare token | Bo reports missing · Fr takes · offer disabled · attempt: 422 `NO_SUBSTITUTE` | H2 ord 3 · S, I#1 |
| 13 | HT-02 IC on a supplementary lane (depth 1 of 1) | Bo reports HT-02 missing · Fr takes · attempt `replacement`: 409 `REPLACEMENT_LIMIT` · offer lists recovered, false report, write-off | H2 ord 3 · S, I#1 |
| 14 | `allow_post_issuance_replacement=false`; HT-02 free; HT-01 IC | Bo reports missing · Fr takes · attempt: 409 `POST_ISSUANCE_REPLACEMENT_DISABLED` (checked first) | H2 ord 3 · S, I#1 |
| 15 | HT-01 IC reported missing | Fr `recovered` {received_by_agency}; variants: {awaiting_preparation} 422 `RESUME_STATE_INVALID`; `false_report` | H2 ord 3 · S, I#1, R(resume) or R(cancel_false_report) |
| 16 | SP-01 IC damaged; SP-03 free; depth 0 | Bo reports · (80 s idle: Overdue, escalated, S10-T) · Gr takes · Gr `replacement` {SP-03} · Gr approves: 403 (not in team) · Ca approves: 403 (team assignee) · Fr approves | H3 ord 2 (H1 skipped) · S, I#1, A, R(replace) |
| 17 | SP-01 IC damaged; no spare SP | Bo reports · Gr takes · attempt `replacement`: 422 `NO_SUBSTITUTE` · Gr then proposes waiver or write-off (PM-20) | H3 ord 2 · S, I#1 |
| 18 | SP-03 IC on a supplementary lane (depth 1 of 1), damaged | Bo reports · Gr takes · attempt: 409 `REPLACEMENT_LIMIT` · offer lists false report, waiver, write-off | H3 ord 2 · S, I#1 |
| 19 | flag false; SP-03 free; SP-01 IC damaged | Bo reports · Gr takes · attempt: 409 `POST_ISSUANCE_REPLACEMENT_DISABLED` | H3 ord 2 · S, I#1 |
| 20 | SP-01 IC damaged | (a) Gr `write_off` · Ca approves (Fr: 403, not owner class); (b) Gr `waiver` · Ca approves | H3 ord 2 · S, I#1, A(owner), R(write_off) or R(waive) |
| 21 | SP-01 IC damaged | Bo reports · Gr takes · Gr `quarantine` (direct, no approval) | H3 ord 2 · S, I#1, R(quarantine_return) |
| 22 | LT-01 IC `identity_mismatch`; LT-02 free (fixture) | Bo reports · Gr takes · (a) `replacement` {LT-02} · Fr approves; (b) `waiver` · Ca approves | H3 ord 2 · S, I#1, A, R(replace) or R(waive) |
| 23 | SP-01 RP, expected `good` (S11b) | Cl verifies `condition=damaged` (auto-report, verifier) · Cl takes · Cl `quarantine` | H4 ord 4 (H1, H3, H2 skipped) · S, I#1, R(quarantine_return) |
| 24 | SP-01 IC, `expected_return_condition=damaged` (after PM-16 or PM-21) | Bo initiates return · Cl verifies `condition=damaged` · Gr `return_to_inventory` | none: no exception, no route row |
| 25 | SP-01 RP: (a) lost at return; (b) unexpected damage; (c) false alarm | (a) Cl reports `missing` · Em takes · `write_off` · Ca approves; (b) as PM-23 then Em `write_off` · Ca approves; (c) Em `false_report` | H4 ord 4 · S, I#1, A, R(write_off) or R(cancel_false_report) |
| 26 | SP-01 RP | Cl disputes return ("wrong token returned") · Em takes · Em `recovered` {RP or RA} | H4 ord 4 · S, I#1, R(resume) |
| 27 | any return-phase exception (H4 or fallback) | Em takes · offer has no `replacement` · crafted `act` `replacement`: 409 `OUTCOME_NOT_ALLOWED` · publish test: a handler catching `return` that reaches `replace`: V-35 | H4 ord 4 · S, I#1 |
| 28 | fixture limit 3; tokens HT-02…HT-04 free | HT-01 missing at AP → H1 replace HT-02 (depth 1) → HT-02 lost in custody → H2 replace HT-03 (depth 2) → HT-03 lost → H2 replace HT-04 (depth 3) → HT-04 lost → attempt: 409 `REPLACEMENT_LIMIT` | H1, then H2, H2, H2 · four chains S, I, A, R(replace) |
| 29 | S10 preconditions | S10 steps 1–6 (§40.1) | H1 ×2, H2, H3 |
| 30 | (a) LT-01 exception open, request `credential_issued`; (b) post-approval; (c) SP-01 AP with open H1 exception, SP-02 HP | (a) Bo closes · Fr and Ca continue; (b) auto-expire and tombstone Bo: nothing; `loss_report` done by an Alpha Engineers L1 member; (c) Gg resets without force, then with force and notes | (c) H1 instances `cancelled` by `admin_reset` |
| 31 | H3, and a fixture handler whose approval team equals its investigate team (V-36) | investigator proposes then approves; reporter (owner-class fixture) approves; submitter (delegate window) approves; unrelated approver approves | any · A |

### §39.2 Table B: resulting state

| PM | Case · item · asset | Lane · join | Request status · summary Δ | Graph |
|---|---|---|---|---|
| 01 | HT-01 case RE; item excluded `replaced`; asset retired `lost_replaced`; new item (`replaces_item_id`), hold on HT-02, case AP | HT-01 lane excluded; new lane `origin=replacement`, depth 1; join counts HT-02 | `approved` · total+1, replaced+1 | HT-01 struck through; "replaced by" edge; band #1 chip "H1 · Before issuance" |
| 02 | nothing written: case EO, exception `investigating`, no proposal, no history event | unchanged; join Wait | `approved` · none | I#1 blue; chip "Replacement · No substitute available" |
| 03 | nothing written | unchanged | `approved` · none | chip "Replacement limit reached (1/1)"; offer: recovered, false alarm, write-off, waiver |
| 04 | nothing written; case EO with `state_before_exception=HP` | unchanged | `approved` · none | as PM-03 |
| 05 | as PM-01; asset retired `lost_replaced` (class damaged) | as PM-01 | as PM-01 | as PM-01 |
| 06 | case RE; item excluded `written_off`; asset retired `written_off`; hold released | lane excluded; join recomputed: (a) live=3; (c) live=0 `AllExcluded` → `close_unfulfilled` | (a) `approved`; (c) `closed`, reason `unfulfilled_all_items_written_off` · written off+1 | struck lane; card "Write off · before issuance"; (c) terminal "Closed – unfulfilled" |
| 07 | (a) case AP; (b) case HP; exception `resolved` or `cancelled_as_false_report`; events `asset_recovered` or `exception_cancelled` | unchanged; (a) stage SLA re-armed | none · none | band closed with badge "Resumed" or "False report" |
| 08 | case HP; event `waiver_granted`; `expected_return_condition` unchanged; reject: proposal `rejected`, exception `investigating` | unchanged | none · none | badge "Waived"; reject shows A(reject) edge back to I#2 |
| 09 | case HP; event `asset_recovered`; item not counted as received meanwhile | stage SLA re-armed | none · none | badge "Resumed" |
| 10 | case RE; asset retired `written_off`; hold released; item **not** excluded | lane not excluded; issuance stands | stays `credential_issued` · written off+1, outstanding−1 | band #1 with task node (Bo); card "Write off (after issuance)" |
| 11 | HT-01 case RE; item not excluded; asset retired `lost_replaced`; HT-02 item, hold, case AP | issuance stands; new lane `origin=supplementary`, depth 1 | `credential_issued` · total+1, replaced after issuance+1 | dashed HT-02 lane; "supplements" edge |
| 12 | nothing written | unchanged | none · none | chip "No substitute available" |
| 13 | nothing written | unchanged | none · none | chip "Replacement limit reached (1/1)" |
| 14 | nothing written | unchanged | none · none | chip "Replacement is not allowed after issuance for this system" |
| 15 | (a) case RA; (b) case IC; events `asset_recovered`, `exception_cancelled`; invalid state: nothing written | unchanged | none · none | badge "Resumed" or "False report" |
| 16 | SP-01 case IC with `expected_return_condition=damaged`; item not excluded; hold kept; SP-03 item, hold, case AP | issuance stands; new lane `supplementary`, depth 1 | `credential_issued` · total+1, replaced after issuance+1 | dashed SP-03 lane; card "Replace (after issuance, damaged)"; Overdue then Escalated badges on I#1 |
| 17 | nothing written | unchanged | none · none | chip "No substitute available" |
| 18 | nothing written | unchanged | none · none | chip "Replacement limit reached (1/1)" |
| 19 | nothing written | unchanged | none · none | chip "Replacement is not allowed after issuance …" |
| 20 | (a) case RE; asset retired `written_off`; hold released; item not excluded; (b) case IC, `expected_return_condition` stays `good` | (a) issuance stands | (a) written off+1; (b) none; a later damaged return opens a new return-phase exception (PM-23) | badge "Written off" or "Waived" |
| 21 | case IC; `expected_return_condition=damaged`; asset unchanged until return; exception `resolved` | unchanged | none · none | badge "Quarantine on return"; leads into PM-24 |
| 22 | (a) LT-01 case IC, expected `good`, not quarantined unless found damaged; LT-02 item, hold, case AP; (b) case IC | (a) new `supplementary` lane, depth 1 | (a) replaced after issuance+1 | (a) dashed LT-02 lane |
| 23 | one transaction: `return_received`, `return_condition_recorded`, exception opened (`reported_by_role=verifier`); then asset `quarantined_at/by/reason='damaged_on_return'`, hold released, not restocked; case RE; outcome `returned_damaged` | lane terminal | returned+1, returned damaged+1 (I-7) | band #1 "reported by verifier (automatic)"; [Q] marker |
| 24 | `return_condition=damaged` recorded; asset quarantined instead of restocked, `restocked_*` null; outcome `returned_damaged` | lane terminal | returned+1, returned damaged+1 | no band; [Q] marker and condition chip "damaged" |
| 25 | (a) asset retired `written_off`; hold released; item not excluded; (b) as (a) after PM-23's opening; (c) case resumes at `state_before_exception` | join unchanged | written off+1; (b) returned+1 as well | badge "Written off" or "False report" |
| 26 | case RP or RA; event `asset_recovered` | unchanged | none · none | badge "Resumed" |
| 27 | nothing written for the refused `act` | unchanged | none · none | no "Propose replacement" button; V-35 listed on publish |
| 28 | four cases RE (the last EO); assets HT-01…HT-03 retired `lost_replaced`; no fourth substitute | lanes: replacement, supplementary, supplementary; depths 1, 2, 3 | replaced+1, replaced after issuance+2 | chain of four lanes; four bands; chip "Replacement limit reached (3/3)" |
| 29 | see §40.1 | see §40.1 | see §40.1 | four numbered bands |
| 30 | (a) exception continues to resolution; (b) no change; `acting_as=submitter_team`; (c) without force 409 `CUSTODY_IN_PROGRESS` and nothing written; with force SP-01 case `cancelled`, exception `resolved` with `resolution_notes='cancelled_by_admin_reset'`, no resolution; SP-02 continues | (c) SP-01 lane cancelled | (a) status `closed`, parent Running until lanes terminal | (c) `admin_reset` highlighted, "cancelled by" edge, badge "Cancelled by reset" |
| 31 | refused attempts: nothing written, instance stays active; the unrelated approver succeeds | as the resolution | none · none | disabled "Approve" with reason "You investigated this exception, so you cannot approve it" |

---

## §40 Demo scenario state tables

Columns: **Lanes** use the state codes of §39 (RE with `xl` = excluded). **Open exc.** counts open
exceptions of the request. **Summary** = `tot` items_total, `iss` issued, `out` outstanding, `ret`
returned, `retD` returned damaged, `rep` replaced, `repAI` replaced after issuance, `wo` written off.
A **video checkpoint** is a Playwright assertion made while the video records (§16 layer 6); the video
is the proof for the presenter and for review.

### §40.1 S10 Mixed fulfilment exceptions (PM-01, 02, 06, 10, 16, 20, 24, 29, 31)

Lanes: L1 SP-01, L2 SP-02, L3 HT-01, L4 LT-01, L5 HT-02 (replacement), L6 SP-03 (supplementary).
Setup: R0 by Bob is issued and holds LT-02 and SP-03.

| Step | Persona · action | Request | Lanes | Open exc. | Summary | Graph highlights | Video checkpoint |
|---|---|---|---|---|---|---|---|
| 0 | Bo submits R1 (SP-01, SP-02, HT-01, LT-01). `item_count 4 > 3` sets `high_risk`: Aaron, Claire (risk high) approve; Fr scores 3, Gr 3 (6 ≥ 6); Ca approves | `approved` | L1–L4 AP | 0 | tot 4 | split/join review; Join "live=4 crossed=0 -> Wait"; Interrupts band unfired | Variables panel `review_score`=6; four lane cards; no band |
| 1 | Gr reports HT-01 missing; Fr takes, proposes replacement {HT-02}; Ca approves | `approved` | L1 AP · L2 AP · L3 RE xl · L4 AP · L5 AP | 1 → 0 | tot 5, rep 1 | band #1 on L3 "H1 · Before issuance"; "replaced by" L5 | route chip text; `rendered_match`; struck lane; card text "Replace (before issuance)"; Join "live=4" |
| 2 | Gr reports SP-02 missing; Fr takes; offer shows replacement disabled; attempt 422; Fr proposes write-off; Ca approves | `approved` | L2 RE xl; others as 1 | 1 → 0 | tot 5, rep 1, wo 1 | disabled chip; Join "live=3" | button disabled with "No substitute available"; 422 toast text; API row counts unchanged for the refusal |
| 3 | Gr does `reset_pin` (L5), prepares and hands over L1, L4, L5; Bo confirms ×3 | `credential_issued` | L1 IC · L4 IC · L5 IC | 0 | tot 5, iss 3, out 3, rep 1, wo 1 | Join "live=3 crossed=3 -> Issue" | status pill; "Close" offered (not pressed); Issuance panel 3 issued rows |
| 4 | Bo reports LT-01 missing; Fr takes, proposes write-off; Bo completes `loss_report`; Ca approves | `credential_issued` | L4 RE (not xl) | 1 → 0 | iss 3, out 2, wo 2 | band #1 on L4 "H2 · Lost in custody" with `loss_report` node; "Attention required: 1 custody exception" | Ca's line "Waiting for the loss report from Bob Lim" before the task; LT-01 outcome `written_off`; status unchanged |
| 5a | R0 returns SP-03. Bo reports SP-01 damaged; 80 s pass (S10-T) | `credential_issued` | L1 EO | 1 | unchanged | I#1 "Overdue"; Fr's dashboard "Escalated" | Overdue badge on the node; Fr's row "Escalated"; timeline `sla_breached` text |
| 5b | Gr takes, proposes replacement {SP-03}; Gr and Ca cannot approve; Fr approves | `credential_issued` | L1 IC (expected damaged) · L6 AP | 1 → 0 | tot 6, repAI 1 | dashed L6 with "supplements"; Join "(1 supplementary ignored)" | class `lane--supplementary`; edge label; Gr and Ca see no "Approve"; API 403 asserted for both |
| 6a | Gr prepares and hands over SP-03; Bo confirms | `credential_issued` | L6 IC | 0 | iss 4, out 3 | status stays | SP-03 issued row |
| 6b | Bo closes the request | `closed` | unchanged | 0 | unchanged | header "Closed · 3 items outstanding" | text asserted |
| 6c | Bo initiates return of SP-01; Cl verifies `damaged` (no exception); Gr `return_to_inventory` | `closed` | L1 RI, [Q] | 0 | ret 1, retD 1, out 2 | [Q] on L1; no new band | marker; outcome `returned_damaged`; `/requests/new` SP-01 disabled |
| 6d | Bo returns HT-02 and SP-03; Cl verifies; Gr restocks both | `closed`, workflow Completed | all terminal | 0 | tot 6, iss 4, out 0, ret 3, retD 1, rep 1, repAI 1, wo 2 | four numbered bands (L3, L2, L4, L1) | final summary equals part 6 §27.12.1; CSV equal; Engine panel "Completed" |

**Additions to S10, made by this part** (mirrored in part 3 §15 and part 6 §27.12.1): **S10-T**
(step 5a): the 80 s idle period that shows the handler SLA breach and escalation (AC-26.4.2), and
**S10-X** (sidecar request R2, after 6d): R2 has SP-01 at AP with an open H1 exception and SP-02 at HP.
Greg resets without force (409 `CUSTODY_IN_PROGRESS` lists SP-02), then with force and notes: SP-01's case is
cancelled, its H1 instances show "Cancelled by reset", `admin_reset` is highlighted with a "cancelled by"
edge, SP-02 continues. The checkpoint asserts the band, the highlight and the edge (PM-30).

### §40.2 S11 Cascade and exhaustion (PM-01, 04, 06, 23, 25)

**Variant a** (lanes L1 HT-01, L2 HT-02).

| Step | Persona · action | Request | Lanes | Open exc. | Summary | Graph highlights | Video checkpoint |
|---|---|---|---|---|---|---|---|
| a0 | Bo submits one token (HT-01); approvals complete | `approved` | L1 AP | 0 | tot 1 | not high risk: straight to owner | one lane card |
| a1 | Gr reports HT-01 missing; Fr proposes replacement {HT-02}; Ca approves | `approved` | L1 RE xl · L2 AP | 1 → 0 | tot 2, rep 1 | "replaced by" edge | chip "H1 · Before issuance"; L2 depth 1 |
| a2 | Gr prepares and hands over HT-02; Bo disputes ("token cracked"); Fr takes; offer opened | `approved` | L2 EO (before: HP) | 1 | unchanged | chip "Replacement limit reached (1/1)" | disabled reason text; offer lists write-off, recovered, false alarm, waiver; API 409 writes nothing |
| a3 | Fr proposes write-off; Ca approves | `closed` (unfulfilled) | L1 xl · L2 RE xl | 0 | tot 2, iss 0, rep 1, wo 1 | Join "live=0 -> AllExcluded"; terminal "Closed – unfulfilled" | status pill; reason text; Engine panel "Completed" |

**Variant b** (lane L1 SP-01).

| Step | Persona · action | Request | Lanes | Open exc. | Summary | Graph highlights | Video checkpoint |
|---|---|---|---|---|---|---|---|
| b0 | Bo submits SP-01; issued; Bo initiates return | `credential_issued` | L1 RP | 0 | tot 1, iss 1, out 1 | return in progress | Issuance row "return pending" |
| b1 | Cl verifies with `condition=damaged` | `credential_issued` | L1 EO (before: RA) | 1 | unchanged until b2 (I-7) | band #1 "reported by verifier (automatic)", route H4 | band header text; timeline `return_condition_recorded` |
| b2 | Cl takes; acts `quarantine` | `credential_issued` | L1 RE | 0 | ret 1, retD 1, out 0 | [Q] on L1; card "Quarantine on return" | marker; `/requests/new` SP-01 disabled; submit 409 `ASSET_QUARANTINED` copy |
| b3 | Bo closes | `closed`, Completed | L1 terminal | 0 | unchanged | Engine "Completed" | final summary row |

### §40.3 S12 Clone a workflow (HRIS v1 → PAYROLL)

PAYROLL: owners Dan and Carol, vendor cluster Beta linked, PA Owner Support, and (I-5, resolved) two live
assets PAY-D1 (device) and PAY-T1 (token). Lanes and exceptions do not apply until step 8.

| Step | Persona · action | Draft or request | Issues | Graph or editor highlights | Video checkpoint |
|---|---|---|---|---|---|
| 1 | Amanda opens `/admin/systems/new`; steps 1–2 (details, owners, PA); step 3 W1 "Clone from system version" | none | none | wizard W1 | start choice radio; three options text |
| 2 | W2: AUR → HRIS → v1 (fingerprint shown); Next runs preview | none (nothing written) | none | version list excludes the v2 draft | v2 row disabled; DB row counts unchanged |
| 3 | W3: Alpha Leads L2 → Beta Leads L2 (position match); Alpha Engineers L1 left unresolved; other teams keep defaults | none | preview: 5 teams | badges "Position match", "Needs a choice", "Same team" | badge texts; red "Unresolved" row |
| 4 | W4 review, then Create draft | draft r1, lineage set | V-42 ×1 (`alpha_l2` escalation); V-43 none | editor opens; red chip "Choose team (was Alpha Engineers L1 …)"; lineage chip | chip text; Simulate and Publish disabled with tooltip |
| 5 | Amanda presses Simulate and Publish (disabled); crafted publish | unchanged | 409 `DEFINITION_INVALID` | Validation tab lists the error | tooltip text; API 409; no definition row added |
| 6 | Amanda chooses Beta Engineers L1 in the inspector | draft r2 | V-42 clears | chip disappears; Validation clean | Validation tab empty of errors; Simulate enabled; simulate 1 item runs |
| 7 | Amanda publishes PAYROLL v1 with change notes | PAYROLL v1 published | none | version list "cloned from HRIS v1, AUR" | lineage text; HRIS fingerprint unchanged |
| 8 | Betty submits a PAYROLL request; Brian (Beta Leads L2) is the first approver | `submitted` | none | graph shows H1–H4 in the plan and Beta teams in the vendor steps; Interrupts band unfired | run pinned to PAYROLL v1; assignee chips read Beta; Interrupts band present |

### §40.4 Capability-to-step map and presenter script

| Capability | Where shown | What the presenter says and shows |
|---|---|---|
| Drawn handlers | editor Handlers tab; S10 step 1; every band | "The path an exception takes is drawn, not hidden. Here is H3." Open the mini-canvas, then the same nodes on the request graph |
| Custom team assignment | S10 5b (H3); editor H3 | "Asset Store investigates; Ops Leads approve." Grace cannot approve; Carol cannot approve a team step |
| First-match routing and fallback | S10 steps 1, 4, 5b; coverage grid; FINDB fallback | Expand the route reasons: H1 skipped. Show the grid cell and the single-handler FINDB |
| Separation of duties | S10 5b; PM-31 | Investigator and reporter cannot approve; the copy "You investigated this exception…" |
| Pre- and post-issuance replacement | S10 step 1 versus 5b; S11 a1 | Same action, two effects: replacement lane versus supplementary lane |
| Supplementary lane | S10 5b, 6a | Dashed lane; request stays issued; cannot complete until SP-03 is back |
| Refusal when no substitute | S10 step 2 | Disabled button, 422, nothing written |
| Write-off before and after issuance | S10 steps 2 and 4 | Join "live=3"; issuance stands for LT-01; loss report task before the decision |
| Quarantine and reuse block | S10 6c; S11 b2 | [Q] marker; `/requests/new` disabled; 409 copy |
| Replacement limit | S11 a2 | "Replacement limit reached (1/1)" reported before "No substitute" |
| Unfulfilled closure | S11 a3 | Join `AllExcluded`; "Closed – unfulfilled" |
| Timers and escalation in handlers | S10-T (5a) | 80 s idle: Overdue badge, Frank's "Escalated" row, timeline text |
| Interrupts band | every request; S10-X | Locked nodes always drawn; fired `admin_reset` with "cancelled by" edge |
| Cloning with mapping | S12 steps 2–8 | Preview writes nothing; red chip blocks publish; lineage; HRIS unchanged |

### §40.5 Inconsistencies found in parts 6 and 7 (all resolved)

| ID | Where | Finding | Status |
|---|---|---|---|
| I-1 | part 6 §27.11 H3 versus PM-21 and the coverage rule | Seeded H3 has no `quarantine` port, so PM-21 (in custody, `quarantine_return`) is not reachable in HRIS v1, although the demo must cover every row | Resolved in part 6 §27.11 (H3 gains the direct `quarantine` port; part 3 §13 and part 4 §25.3 follow); V-38 is unaffected |
| I-2 | part 6 §27.12.1 step 3 versus AC-27.2.1 and AC-27.6.1 | Step 3 says Bob closes the request, yet step 4 and the ACs expect `credential_issued` and "Bob may still close" | Resolved in part 6 §27.12.1 (step 3 no longer closes; step 6 closes, as 6b here) |
| I-3 | part 6 §27.12.1 step 6 | "HT-02 and SP-03 are received by Bob" but HT-02 was received in step 3 | Resolved in part 6 §27.12.1 step 6 (receives only SP-03) |
| I-4 | AC-28.1.1 versus §28.4 | "Leaves the `sec_review` team unresolved": that node's team is an agency team (Ops Leads L2) with a same-organisation default, so it cannot be left unresolved; and a mapping row applies to every use of a team, so one choice would affect several nodes | Resolved in part 7 AC-28.1.1 and part 3 §15 S12 (a vendor team, Alpha Engineers L1, is left unresolved) |
| I-5 | AC-28.1.1 | "One live asset of an allowed type" still raises V-43 for the other allowed type | Resolved in part 7 AC-28.1.1 and part 3 §13 (one asset per allowed type) |
| I-6 | S10 step 5 versus AC-27.8.1 | Grace's 403 is written "`SOD_VIOLATION` or not in team"; Grace is not in the approval team, so it is a plain 403 | Resolved in part 6 §27.12.1 step 5 and part 7 AC-27.8.1 (plain 403 for non-team actors; `SOD_VIOLATION` for the §27.3 rules) |
| I-7 | part 6 §27.5.2 `quarantine_return` / return | `items_returned` for an unexpected-damage return is stated at resolution, but `returned_at` is set at `return_received` | Resolved in part 6 §27.5.2 and part 7 AC-27.4.1 (counts update at resolution; part 9 §35.7 E-12) |
| I-8 | part 6 header | lists part 8 as §30–§33; parts 3, 4 and 7 say §30–§34 | Resolved in part 6 header (§30–§34, and parts 9 and 10 listed) |

---

## Stories

**Conventions:** these continue §14 (part 2) and parts 5 and 7, with the same AC format. **Every negative
AC also asserts no side effect** (no row changed, no workflow history event, no audit row, no
notification). Personas as in part 7. Component tests are jsdom (Vitest, React Testing Library); e2e tests
are Playwright with video (§16 layer 6). Test-id names are those of §38.4.

### ST-32 Exception and cloning UI, walkthroughs and demo state tables

**US-32.1** As an author, I edit handlers in the Handlers tab (§38.1).
- AC-32.1.1
  - G: Amanda opens the `standard_custody_lane` v1 draft in the template editor and the Handlers tab.
  - T: `handler-list__item` rows read H1, H3, H2, H4 and the fallback in ordinal order. The fallback shows
    the lock, no `handler-list__grip` and a disabled `handler-list__delete`. Selecting H3 shows its
    mini-canvas and the filter "Reads as" sentence equal to the stored `rendered_match` filter text.
- AC-32.1.2: dragging H2 above H3, or clicking "Move up" on H2, saves the ordinals; the grid updates at
  once and `HANDLER_SHADOWED` appears on H3. A drop below the fallback is refused with no save request.
- AC-32.1.3: adding `damaged` to the types chip list sends one filter row; removing every chip shows
  "All types"; a crafted `PUT` with a value outside the domain, a filter on the fallback, or two fallbacks
  returns 422 with a path, and the revision is unchanged.
- AC-32.1.4: the `investigate` inspector keeps at least one port checked; unchecking the last is blocked.
  Unchecking a wired port asks for confirmation and removes the edge with it; declining changes nothing.
- AC-32.1.5: team pickers for `investigate`, `resolution_approval` and `escalate_to_team` list only live
  agency teams of the system's organisation; `resolution_approval` also offers "Owner class". A crafted
  vendor, `pa` or other-organisation team is V-24 and nothing changes.
- AC-32.1.6: the Interrupts tab shows three padlocked nodes with no handles and a read-only description
  each; a crafted edit returns `PLATFORM_NODE_LOCKED` with the copy of §38.4.

**US-32.2** As an author, I trust the coverage grid (§38.1).
- AC-32.2.1: for HRIS v1 the grid has the 30 cells of §38.1: `data-state` is `caught` for every H1, H2 and
  H4 cell, `shadowed` for in-custody damaged and identity_mismatch (H3), `impossible` for the six
  impossible pairs, and none is `fallback-only`.
- AC-32.2.2: deleting H4 turns every `return` cell to `fallback-only` (`fb`) without an error; removing
  the fallback marker by a crafted `PUT` shows the uncovered pairs and V-32 (error) names each pair.
- AC-32.2.3: with H3 limited to severity {high, critical}, the in-custody damaged cell reads `~` (varies)
  on "any", H2 on `low` and H3 on `high`.
- AC-32.2.4: clicking a non-impossible cell focuses the winning handler; clicking an impossible cell does
  nothing. Every cell state has its text mark and tooltip (not colour alone).

**US-32.3** As an author, I read effect cards that cannot drift from the platform (§38.1).
- AC-32.3.1: for every row of the §38.1 table, `effect-card__text` for that `data-kind` and `data-phase`
  equals the string returned by `GET /api/admin/platform/exception-effects`, and the string equals the
  table. The web bundle contains none of the card strings (a grep test over `web/src`).
- AC-32.3.2: with the endpoint mocked to return a changed `cardText`, the card shows the change without a
  code change; a phase the handler cannot catch has no card; `replace` at `return` shows "Not permitted".
- AC-32.3.3: the request-page resolution terminal shows the card the server sent in `/plan`, and its text
  equals the editor card for the same (kind, phase, class).

**US-32.4** As an author, I inject exceptions into the simulator (§38.1).
- AC-32.4.1: choosing lane, type, severity, a forced phase and `substitutes_available` sends the
  `{kind:'exception', …}` payload of part 6 §27.10; the panel shows `RouteDecision` with every candidate
  and its reason. An impossible (type, phase) pair has the Inject button disabled.
- AC-32.4.2: with substitutes off, `act replacement` shows the `NO_SUBSTITUTE` copy in the trace and no
  state change; with replacement count 1 of 1 it shows the `REPLACEMENT_LIMIT` copy (reported first).
- AC-32.4.3: the injector can build all 31 PM rows of §39; the simulation writes no rows and starts no
  workflow (row counts and Temporal list unchanged).

**US-32.5** As a requester or reviewer, I see exceptions on the request page (§38.2).
- AC-32.5.1: a lane with two exceptions renders bands whose `data-exception-no` are 1 and 2 in order, each
  with phase text, route chip, status and resolution badge; expanding the route shows the skipped handlers
  with their reasons.
- AC-32.5.2: a supplementary lane has the class `lane--supplementary`, a dashed border and a "supplements"
  edge; a replacement lane has the "replaced by" edge; the Join trace ends "(1 supplementary ignored)".
- AC-32.5.3: a quarantined asset shows the `[Q] Quarantined` marker linking to `/assets/[id]`; the lane
  with `expected_return_condition=damaged` shows the condition chip.
- AC-32.5.4: the Interrupts band is always drawn with three locked nodes; a fired node shows "fired {time}
  by {actor}" and a "cancelled by" edge to each cancelled instance; unfired nodes show no highlight.
- AC-32.5.5: band header colours follow the phase and every band header carries the phase word; a
  screenshot test with colour removed still distinguishes the three phases by text.
- AC-32.5.6: after Temporal history is archived, the bands, routes and Interrupts band render from the
  database rows alone (AC-26.3.5) and the Engine panel shows "history archived".

**US-32.6** As an actor, I see exactly the actions I may take, with the reason when I may not (§38.2).
- AC-32.6.1: on an `investigate` instance in `investigating`, the replacement button is `aria-disabled` with
  the exact reason copy for each of the three causes in the order of part 6 §27.4; with
  `allow_write_off=false` no write-off button exists.
- AC-32.6.2: in `awaiting_information` every port button is disabled with "On hold for information.
  Continue investigating first"; "Continue investigating" is enabled and keeps the same instance.
- AC-32.6.3: on `resolution_approval`, the investigator and the reporter see "Approve" disabled with the
  `SOD_VIOLATION` reason; a user outside the assignee set sees only "Waiting for …" and no button; a
  forced API call by either is 403 with no side effect.
- AC-32.6.4: while the `loss_report` task is open, Carol sees "Waiting for the loss report from Bob Lim" and
  no approve button; Bob sees "Complete loss report" and cannot submit it without the tick.
- AC-32.6.5: an `act` refused by the server (`STALE_STATE`, `ASSET_IN_USE`, `NODE_NOT_ACTIVE`) shows its
  copy, keeps the form values and refreshes the offer; no optimistic change is shown.

**US-32.7** As a participant, I see the exception in the timeline, panel and dashboards (§38.2).
- AC-32.7.1: for S10 step 5 the timeline shows the templates of §38.2 in order (reported, routed,
  started, `sla_breached`, proposed, approved, `replacement_authorized`, `supplementary_lane_started`)
  with actor and asset names substituted.
- AC-32.7.2: the Issuance and returns panel and the CSV show the columns Lane origin, Replaces/replaced by,
  Return condition and Quarantined at, and the counts Returned damaged and Replaced after issuance; the panel,
  CSV and `/assets/:id/history` agree on S10's final summary.
- AC-32.7.3: "Needs my action" shows the four rows of §38.2 to the stated personas only (Grace for H3
  `investigate`, Frank for H3 approval, Bob for `loss_report`, Claire for the H4 quarantine row); Emma sees
  none; overdue rows sort first with the red badge, and the escalation team sees "Escalated".
- AC-32.7.4: the header shows "Attention required: N custody exceptions" counting open exceptions of this
  request only.

**US-32.8** As a user, I read an error I can act on (§38.4).
- AC-32.8.1: a test enumerates every code of part 6 §27.9, §28.8 and part 9's matrix and asserts the
  copy table of §38.4 has a message for it; a missing entry fails the build. Unknown codes use the fallback
  copy, which carries the code.
- AC-32.8.2: no message contains an identifier of another organisation; a 404 for a hidden and for a
  missing object renders the same text and DOM.

**US-32.9** As an administrator, I clone through the wizard (§38.3).
- AC-32.9.1
  - G: Amanda is on `/admin/systems/new`, step 3.
  - T: W1 offers Start blank, Clone from system version and Import file; choosing Clone enables Next;
    W2 lists only published and retired versions with publisher, date and fingerprint prefix; an
    `org_admin` sees only the target system's organisation (D-38), Greg sees all.
- AC-32.9.2: Next on W2 calls `clone/preview` and writes nothing (row counts unchanged). W3 shows one row
  per distinct `(cluster, level)` with source, used-by, suggestion, badge and a target select; badges
  follow §38.3 and an unresolved row is red with the text "Unresolved".
- AC-32.9.3: "Leave unresolved" exists only for rows with no default; a defaulted row offers only live
  teams. Cross-organisation previews start every row empty.
- AC-32.9.4: Create draft sends the mappings; on 201 the editor opens with the lineage chip, V-42 errors
  and "Choose team (was …)" chips, with Simulate and Publish disabled and the stated tooltip. Choosing a
  team clears its chip and error.
- AC-32.9.5: each of `DRAFT_EXISTS`, `CLONE_SOURCE_NOT_PUBLISHED`, `CLONE_SAME_SYSTEM`,
  `CLONE_MAPPING_INVALID` and a 404 shows its §38.4 copy at the step; mapping errors outline the rows and
  show each `fields` path; no draft exists afterwards.
- AC-32.9.6: "New draft from another system's version" on `/admin/systems/[id]/workflow` opens W2 with the
  target fixed; with a live draft it shows `DRAFT_EXISTS` and offers "Open the existing draft".

**US-32.10** As a keyboard or screen-reader user, I can do the same work (§38.4).
- AC-32.10.1: a Playwright test adds a handler, sets its filter, reorders it with `Alt+Up` and connects two
  nodes with the dropdown, using no pointer; the coverage grid is reachable by roving tabindex.
- AC-32.10.2: a Playwright test completes the clone wizard (S12 steps 1–4) by keyboard alone; each step
  heading takes focus and the live region announces "Step n of 4".
- AC-32.10.3: bands, grid cells and disabled buttons expose the ARIA labels and `aria-describedby` reasons
  of §38.4; an automated axe check on the editor, request page and wizard reports no serious violation.

**US-32.11** As an implementer, I get a stable component contract (§38.4).
- AC-32.11.1: every component of the §38.4 table renders the listed `data-testid` values and attributes in a
  jsdom test; a missing id fails. No component imports another feature (AC-30.1.6).
- AC-32.11.2: each component renders solely from DTO fixtures (`/plan`, exception-effects, preview); a
  fixture where the server offers an unknown port or state renders it without a code change; the
  components contain no custody state, transition or exception-type literal.

**US-32.12** As the team, I want each walkthrough to be a test (§39).
- AC-32.12.1 (executable fixture)
  - G: `fixtures/pm-walkthrough.csv`, generated from §39 Tables A and B, and `fixtures/permutation-matrix.csv`.
  - W: the meta-test runs.
  - T: it fails if the ids differ, if any of PM-01…PM-31 lacks `walk_pm_NN` (black-box harness, layer 4:
    runs Table A and asserts Table B state) or `ui_pm_NN` (jsdom: renders the recorded `/plan` DTO and
    asserts the Graph column), or if a named test fails. Deleting `ui_pm_16` turns it red.
- AC-32.12.2: the refusal rows (PM-02, 03, 04, 12, 13, 14, 17, 18, 19, 27) assert zero changed rows, zero
  history events and zero audit rows after the attempt, and the copy of §38.4 in the UI test.
- AC-32.12.3: PM-21 runs on the HRIS v1 handler set, whose H3 exposes `quarantine` (I-1, resolved in part 6 §27.11); if the port is
  missing the meta-test fails with "PM-21 not reachable in the seed", so the gap cannot pass silently.

**US-32.13** As a presenter and reviewer, I want S10–S12 asserted at every checkpoint (§40).
- AC-32.13.1: the Playwright run of S10 asserts, at each row of §40.1, the request status, each lane state,
  the open-exception count, the summary counts and the checkpoint text; the video is recorded and stored
  as an artefact; a changed count fails the step that follows the change.
- AC-32.13.2: the Playwright runs of S11 a and b assert every row of §40.2, including the `AllExcluded`
  closure (`decision_reason='unfulfilled_all_items_written_off'`) and the `ASSET_QUARANTINED` copy on
  `/requests/new`.
- AC-32.13.3: the Playwright run of S12 asserts every row of §40.3, including: no definition row after
  W2; V-42 count 1; disabled Simulate and Publish with the tooltip; the crafted publish returns 409; the
  HRIS fingerprint before and after is identical; Betty's run is pinned to PAYROLL v1.
- AC-32.13.4 (capability proof): a test reads the table of §40.4 and asserts that every capability row
  names a step that exists in §40.1–§40.3, and that S10-T and S10-X run and produce a checkpoint; deleting
  the S10-T step makes the "Timers and escalation in handlers" row fail.

**US-32.14** As an author, I view and arrange a handler's graph on its own canvas (§38.5.1).
- AC-32.14.1
  - G: Amanda opens the `standard_custody_lane` v1 draft, the Handlers tab, and selects H3.
  - T: `handler-canvas__node` elements are exactly the nodes whose `handler_id` is H3, laid out from the
    stored `pos_x/pos_y`; no custody-stage node of the lane template or node of another handler is rendered.
- AC-32.14.2: moving a node autosaves its position; reloading restores it. Auto-layout moves only
  unpinned nodes. Fit shows every node. Zoom stops at 25 % and 200 %.
- AC-32.14.3: Expand opens the overlay with the palette and inspector; `Tab` stays inside it; `Esc` closes
  it and returns focus to H3's `handler-list__item`.
- AC-32.14.4: a crafted `PUT` with an edge between a node of H3 and a node of H2 or of the lane template
  returns 422 (V-39, or the `wf_edges` trigger) with a path; the revision is unchanged.

**US-32.15** As an author, I cannot draw an edge the server would refuse (§38.5.3).
- AC-32.15.1: for each of rules 1–6 of §38.5.3, a drag that breaks the rule draws no edge, sends no save
  request (network log empty), and shows `handler-canvas__refused` with the exact tooltip text.
- AC-32.15.2: "Connect *port* to…" on H3's `replacement` port lists the approval nodes and the generic nodes
  of H3, and no `resolution`, no `exception_start` and no node of another handler.
- AC-32.15.3: adding `return` to the phase filter of a handler that contains a `replace` resolution marks
  that node with the V-35 error and its tooltip (rule 7); removing the phase clears it.
- AC-32.15.4 (the server is the control): for each rule, a crafted `PUT` containing the refused edge returns
  the matching V-code in `issues[]` (V-33, V-34, V-35 or V-37) and Publish stays disabled; nothing else in
  the draft changes.
- AC-32.15.5: the H2 path `write_off` → `task loss_report` → approval → `write_off` can be drawn edge by edge
  with no refusal (rules 4–5 look only at the direct target).

**US-32.16** As an author, I add, copy and remove handlers and nodes (§38.5.4).
- AC-32.16.1: the palette in the Handlers tab lists exactly the nine kinds of §38.5.4 (`handler-palette__item`
  `data-kind`); `custody_stage`, `exception_start` and top-level kinds are absent; in the Lane tab, handler
  kinds are absent.
- AC-32.16.2: Add handler with no filter chip has `handler-add-dialog__create` disabled; with a duplicate key
  it shows "This key is already used" (unique `(definition_id, handler_key)`, part 6 §27.7) and creates
  nothing. With a chip it creates the handler at the ordinal just above
  the fallback, with the four-node template of §38.5.4, and the only issue for it is V-42.
- AC-32.16.3: Duplicate on H3 creates `h3_copy` after H3 with the same nodes, edges, positions and filter rows
  under new keys; the coverage grid is unchanged and `HANDLER_SHADOWED` names the copy. Duplicate is absent on
  the fallback.
- AC-32.16.4: `Delete` on `exception_start` does nothing and announces "The start of a handler cannot be
  deleted"; on an `investigate` node it removes the node and its edges, and V-04/V-05 issues appear for the
  nodes left unconnected.
- AC-32.16.5: after reordering H2 above H3, editing H2's SLA, and deleting a node in H4, three undos restore
  the original draft exactly (ordinals, SLA, node and edges) and three redos re-apply them; each undo selects
  the handler it applies to.

**US-32.17** As any author, I can read the canvas without relying on colour or a pointer (§38.5.2).
- AC-32.17.1: each node kind renders its icon, its text label and one labelled `handler-canvas__port` per
  drawn port; an `investigate` node renders only its checked ports.
- AC-32.17.2: a screenshot test of H3 in greyscale still distinguishes every port and resolution kind by its
  text; an axe check of the Handlers tab and the Expand overlay reports no serious violation.
- AC-32.17.3: a node with a V-42 placeholder shows "Choose team (was {source team})" on the node; a node with
  an error has the error icon and the issue codes in its tooltip.
- AC-32.17.4 (extends AC-32.10.1): a Playwright test, with no pointer, adds a handler, drops an `investigate`
  and a `resolution` from the palette, connects them with the dropdown, and moves a node with the arrow keys;
  the resulting draft equals the expected fixture.

**US-32.18** As an author and a reviewer, I find issues, compare versions and read a past handler run (§38.5.5, §38.5.6).
- AC-32.18.1: clicking a V-37 issue on H1 switches to the Handlers tab, selects H1 and selects the offending
  edge on its canvas; clicking a V-32 issue highlights the uncovered grid cells.
- AC-32.18.2: diffing a draft against HRIS v1 in which H3 was moved, H4 changed and a handler added shows the
  badges `moved 2 → 3`, `changed` and `added` with their text; filter chip and node changes are highlighted
  as §38.5.5; the badges are read from the server diff, with no client re-computation of the match.
- AC-32.18.3: on the request page, "Show handler graph" on S10 step 5's band renders H3 from the `/plan`
  fixture alone (jsdom): visited nodes carry actor and time, taken edges have `data-taken="true"`, untaken
  edges are greyed, and the reached resolution shows the server's effect card.
- AC-32.18.4: a request pinned to HRIS v1 still renders v1's H3 after v2 changes H3 and is published; a
  user who cannot see the request gets 404 and no graph data.
