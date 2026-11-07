# Addendum 220 (part 3) — Figma Make prompts: request page graph, exception bands, action panel, dialogs, side panels

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only. Tag `[CUSTOM: design handoff]`.
**Parts:** part 1 `addendum-220-figma-make-screen-prompts.md` (how to use, Prompts 0, 0B, 1); part 2 `…-part-2-dashboard-new-request-and-request-shell.md` (Prompts 2 to 4); **this part** (Prompts 5 to 9); part 4 `…-part-4-reports-assets-and-admin.md` (Prompts 10 to 19); part 5 `…-part-5-wizard-versions-and-editor-layout.md` (Prompts 20 to 24); part 6 `…-part-6-editor-tabs-simulation-and-dialogs.md` (Prompts 25 to 30).
**Normative source:** addendum 219 §12, §38.2, §38.4, §38.5.6, part 4 §21, §24.1, §25, ST-5, ST-7 to ST-9, ST-18 to ST-21, part 11 §41.13 (Variables panel groups and sources). 219 wins on any conflict.
**Route for all five prompts:** `/requests/[id]`. They fill the regions that Prompt 4 left as placeholders. Paste them in order, in the same file.

---

## Prompt 5: request page, workflow graph

**Mirrors:** 219 §12 "Graph generation" and node states, part 4 §24.1 (R8), §25.2 (HRIS v1 graph), ST-11 US-11.2, ST-20 US-20.5.
**Needs:** Prompts 0, 0B and 4 (RQ3 and its four demo states).

````
Using the app shell, SGDS styling and mock data already in this file, fill the "Workflow graph" card of the request
page at /requests/:requestNumber with the live workflow graph. Use a node-and-edge canvas (React Flow style) with
custom nodes and labelled edges, but read-only: no dragging of nodes, no connecting. Pan with drag on empty space,
zoom with the controls and the wheel.

PURPOSE AND WHO. Everyone who may open the request sees the graph. It shows how this request is moving through its
workflow, exactly as the workflow was drawn by the system's administrators. The graph is never hard-coded: it is built
from the request's data. A new request published under a newer workflow version keeps its old shape for its current run.

CARD ANATOMY (desktop: the card fills the main column, at least 560 px tall)
- Card header: Heading MD "Workflow", then a Body SM line "HRIS v1 · run 1" (workflow name, version, run), and on
  the right a toolbar: zoom out, a zoom label "100%", zoom in (icon buttons with aria-labels "Zoom out", "Zoom in"),
  "Fit" (sgds-button ghost), "Mini-map" (sgds-switch), "Full screen" (icon button). A legend button "Legend" opens a
  popover (sgds-dropdown) listing the states.
- Canvas: left to right flow. A mini-map in the bottom-right corner when switched on.
- Legend (also always printed under the canvas in Body SM): "Active: blue border · Done: green · Returned: amber ·
  Rejected or expired: red · Problem open: red dashed with a warning icon · Overdue: red badge · Excluded:
  struck through on grey · Supplementary: dashed border".

NODE STYLES (every node shows an icon, a text label and, when relevant, a state word, so colour is never the only cue)
- pending: grey border, label in default text, state word "Pending".
- active: blue (accent) border with a soft glow, state word "Active" and the assignee chip (for example "Ops Reviewers L1").
- done: green border and a check icon, state word "Done", with Body SM "{actor} {time}" (for example "Claire Wong 9:42am").
- returned: amber border, state word "Returned".
- rejected or expired: red border, state word "Rejected" or "Expired".
- exception open: red dashed border and a warning icon, state word "Problem open".
- overdue: a red badge "Overdue {n}h" at the top-right corner of the node.
- excluded (written off or replaced): grey fill, label struck through, state word "Excluded".
Selecting or hovering a node opens a details popover (or, on touch, a bottom sheet) with: kind, label, assignee, status,
who acted and when, the remark (if any, quoted), SLA due time, "Overdue {n}h" if late, the fields that were captured
(for example "Risk level: high"), and for rule nodes the rendered evaluation, for example "review_score (5) >= required_score (6) -> false".
Nodes are keyboard focusable (Tab in reading order); Enter opens the popover; Escape closes it. Each node's accessible
name follows "{Kind}: {assignee}, {status}" (for example "Approval: Ops Reviewers L1, active").

EDGES. Solid and coloured when taken; grey dashed when not taken. Labels sit on the edge: the action or condition,
for example "approve", "return (notes)", "reject (notes)", "owner approve", "all live items received",
"timer: window elapsed", "true", "false", "joined", "replaced by", "supplements". Hovering a rule edge shows the
evaluation text. Return edges are drawn as curved back-edges to the Submit or Amend node.

TOP-LEVEL SHAPE OF THE HRIS v1 WORKFLOW (for RQ3 and RQ1; draw exactly this order, left to right)
1. Submit (start). Label "Submit".
2. Approval "Alpha Leads L2" (SLA 4 hours, escalates to Alpha Engineers L1).
3. Approval "Ops Reviewers L1" (captures "Risk level").
4. Set variable "Assess risk" (sets high_risk when risk level is high, or items > 3, or window > 8 hours).
5. Rule "High risk?" with edges "true" and "false".
6. On true: the embedded workflow "Dual review" (cardinality Once), drawn as a collapsible group with the breadcrumb
   chip "Dual review · v1" in its header, the input chip "required_score" and the output chip "review_score". Its
   ports are "passed" and "failed". Expanded, the group shows its inner instances, left to right: Split "Review" with
   two branches labelled "Security" and "Logistics": Approval "Ops Leads L2" (captures "Security score") then Set
   variable "Add security score"; and Approval "Asset Store L1" (captures "Logistics score") then Set variable "Add
   logistics score"; then Join "All reviews" (policy: all) then Rule "Score ok?" (review score >= required score).
   "true" leaves the group by "passed"; "false" leaves by "failed". Port "passed" goes to Owner; port "failed" goes to
   the Rejected terminal. A reject or return from an inner approval leaves the group and acts on the whole request,
   so the amber and red back-edges are drawn from the group border to Submit or Rejected. Collapsed, the group shows
   only the header, its state word and the active inner step name (for example "Active: Ops Leads L2").
7. On false (not high risk): straight to Owner approval.
8. Owner approval ("Owner: Carol Sim, Dan Teo or a delegate"), then Notify "Notify PA", then Wait "Until 30 min before
   start" (timer), then the per-item step (below).
9. After the per-item step: its "issued" port goes to Task "Close request" (submitter or any reviewer), then Join "Done".
   Its "completed" port goes to the same Join "Done" then Terminal "Completed". Its "unfulfilled" port goes to
   Terminal "Closed – unfulfilled".
10. Side terminals drawn at the bottom or right: "Returned" (dashed edge back to Submit through "Amend: Bob Lim"),
    "Rejected", "Expired". Return and reject edges leave every approval node.
For FINDB or OPSTOOL requests draw the linear chain only: Submit, one approval node per team in the chain, Owner, the
per-item step (Items · Basic custody lane v1), Close request, Completed, with the same side terminals. The graph for RQ1, RQ6, RQ7, RQ10 uses these.

THE PER-ITEM STEP (the "items" group; an embedded workflow with cardinality Per item)
- A large rounded group node with the header "Items · Standard custody lane v1 · ×6 per item (1..5)" for RQ3 (use the
  real lane count), a breadcrumb chip "Standard custody lane · v1" and an expand and collapse chevron. Inside it, one LANE per item, stacked vertically, each lane a card with header "L1 SP-01
  (device)" and a state word.
- Lane steps, left to right, each a small node: "Reset PIN" (only for token items; a task for Asset Store L1), "Prepare",
  "Hand over", "Receipt" (marked with a flag icon "Issuance milestone"), "Return", "Verify", "Restock". After the
  receipt step a tiny node "Count received". Done steps show a green check, the active step is blue, pending grey.
  Combined steps are drawn as one node when the same team does both ("Prepare and hand over").
- A lane can end early: a lane that was replaced or written off before issuance is shown excluded (struck through,
  grey). A replacement lane is a new lane with an edge labelled "replaced by" from the old lane. A supplementary lane
  (a substitute added after issuance) has a dashed border, an edge labelled "supplements" from its original lane, and
  the caption "supplementary · depth 1".
- When there are more than 3 lanes, lanes collapse into a one-row summary (for example "L2 SP-02 ▸ written off before
  issuance") that expands on click; the collapsed row still shows the lane state word and any exception badge. Add the
  buttons "Expand all lanes" and "Collapse all lanes" to the toolbar.
- Exception bands, quarantined markers and the Interrupts band are added by the next prompt; leave room under each lane
  and at the bottom of the card.

JOIN NODE
The Join (issue) node at the end of the per-item step shows its trace in Body SM monospace-free text, exactly in this form:
"live=4 crossed=2 excluded=1 pending_replacements=0 -> Wait" where the last word is Wait, Issue or AllExcluded; when
supplementary lanes exist append " (1 supplementary ignored)". Examples for the four RQ3 states are in Prompt 4.

STATES AND INTERACTION
- Loading: a skeleton block with the shape of two columns of nodes. Error: sgds-alert danger inside the card with
  "Try again". Empty: not applicable.
- Old run selected: read-only; the graph shows that run's shape and a neutral note "Run 1 of 2".
- Live update: when a node changes state in the mock (a timer button in the demo panel "Advance demo by 1 step"),
  update colours without moving the viewport and announce "Step {label} is now {state}" to a polite live region.
- Pinning: show a Body SM line under the toolbar "This request runs HRIS v1. Newer versions apply only to new runs."
  when the system has a newer published version.
- Full screen opens the same canvas in a modal-like overlay with focus trapped, closing with Escape.
- Mobile: the canvas is 420 px tall; the toolbar shows only Fit and Full screen; nodes keep their text.

MOCK. Draw RQ3 in all four demo states (A to D) as defined in Prompt 4, and RQ1, RQ6, RQ7, RQ10 as simple chains.
RQ7: Alpha Leads L2 is active, Overdue 2h, with the escalation chip "Escalates to Alpha Engineers L1". RQ8: the Expired
terminal is red and the other nodes after the last done step are pending. RQ9: Score ok? shows false and the Rejected
terminal is red. RQ4: all lanes excluded; the Join reads "live=0 -> AllExcluded"; the terminal "Closed – unfulfilled" is lit.

ACCESSIBILITY. The canvas has role="application" with an aria-label "Workflow graph for BG-HRIS-20261108-003"; provide
an equivalent "List view" toggle in the toolbar that shows the same nodes as an outline list (a nested list with state
words), for screen reader and keyboard users. Do not rely on colour alone.

TEST IDS. workflow-graph__canvas, workflow-graph__node (data-node-key, data-state), workflow-graph__edge (data-from-port,
data-taken), workflow-graph__legend, workflow-graph__fit, workflow-graph__list-view, lane-card__header (data-lane-no),
lane-card__supplementary, lane-card__quarantined, join-trace__text.
````

---

## Prompt 6: request page, exception bands, supplementary lanes, quarantine, Interrupts band, handler graph

**Mirrors:** 219 §38.2 "Rendering rules", §38.5.6, §38.4 band ARIA, ST-32 US-32.5 and US-32.18. **Needs:** Prompt 5.

````
Using the app shell, SGDS styling and mock data already in this file, extend the workflow graph of the request page
(Prompt 5) with exception bands, resolution badges, the quarantined marker, supplementary lanes, the join trace
suffix, the Interrupts band and a read-only handler graph. Use RQ3 in demo states A to D.

PURPOSE. When something goes wrong with one item (missing, damaged, wrong item, a disputed handover or return) a
"problem" is reported on that item's lane and routed along a handler that the administrators drew. The request page
shows every problem as a numbered band under its lane, so any viewer can see what happened and why.

EXCEPTION BAND (a card inside the lane, under the lane steps; role="group")
- Header row, left to right: "Exception #1" (numbered per lane, in order; Heading SM); a phase chip that always
  contains the phase word: "Before issuance" (amber), "In custody" (red-orange), "Return" (purple); a route chip
  "H3 · Damaged in custody"; a status word ("Open", "Investigating", "Awaiting information", "Awaiting approval",
  "Resolved", "False report"); and at the right the resolution badge when resolved. A chevron expands and collapses
  the band; collapsed bands keep the header.
- Open exception: the band gets the red dashed border and a warning icon, status word "Open" or "Investigating".
- aria-label on the band: "Exception 1, in custody, route H3, resolved, waived" (number, phase, handler key, status,
  resolution), role group.
- Body (expanded): a row "route: H3 (ordinal 2 of 5)" with a chevron "Show route reasons"; expanding it lists the
  skipped handlers with the reason, for example "skipped: H1 (phase pre_issuance ≠ in_custody)". Then a mini chain of
  the handler steps that were visited, each a small pill: "exc_start ✓", "investigate ✓ Grace 12m",
  "approval ✓ Carol", "resolution: waive". A step that is active shows a blue dot; pending steps are grey. An
  investigate step with an SLA shows a badge "8 h"; if overdue show "Overdue 2h" red; the escalation shows the chip
  "⤴ Ops Leads L2". A task step (the loss report) shows "task loss_report (Bob Lim) ○" and, when done, "✓". A step
  visited twice (after a rejected proposal) shows "×2" and lists both visits on hover.
- Toggle "Show handler graph" at the bottom of the band (a text button; test id exception-band__toggle-graph): it opens
  the handler's own graph inline (see below).
- Examples to draw (RQ3):
  - L1 SP-01, Exception #1, "In custody", "H3 Damaged in custody", RESOLVED · Replaced (state C) or Investigating,
    Overdue 2h, escalated (state B). Chain: exc_start ✓ → investigate ● Grace Neo → approval ○ → resolution.
    In state C the chain reads: exc_start ✓ ▸ investigate ✓ Grace 12m ▸ approval ✓ Frank Chua ▸ resolution: replace.
  - L1 SP-01 also has Exception #2 in state C if you show the return variant: "Return" phase, "H4 Problems at return",
    RESOLVED · Quarantined, with the line "reported by verifier (automatic)" and chain investigate ✓ Claire Wong ▸
    resolution: quarantine_return. (Use this for a second RQ5-like card: RQ5 shows SP-01 return verified damaged with
    this band open.)
  - L3 HT-01, Exception #1, "Before issuance", "H1 Before issuance", RESOLVED · Replaced, an edge "replaced by" to L5.
  - L2 SP-02, Exception #1, "Before issuance", H1, RESOLVED · Written off.
  - L4 LT-01, Exception #1, "In custody", "H2 Lost in custody", RESOLVED · Written off, chain includes the task
    "loss_report (Bob Lim)" done before the owner approval.
- Resolution terminal badges (sgds-badge with icon and text): "Resumed" green; "False report" grey; "Waived" teal
  (cyan); "Replaced" green with a right arrow; "Written off" dark grey (neutral emphasis); "Quarantined" red-brown (a
  danger-muted badge); "Cancelled by reset" grey with strike-through and a "cancelled by" edge from the Interrupts
  band.
- Effect card: when a resolution is shown, an inline Body SM card under the chain in a neutral surface with the
  server-provided sentence. Use these exact sentences:
  - "Replace (before issuance): original lane ends and is excluded; asset retired 'lost_replaced' if lost or damaged, otherwise released; replacement lane starts on the substitute."
  - "Replace (after issuance, lost): original lane ends; asset retired 'lost_replaced'; issuance stands; supplementary lane starts on substitute."
  - "Replace (after issuance, damaged): original resumes in custody expecting a damaged return; substitute gets a supplementary lane; issuance stands."
  - "Write off (before issuance): item excluded; asset retired 'written_off'; lane ends; if no lane is left the request closes unfulfilled."
  - "Write off (after issuance): asset retired 'written_off'; hold released; issuance stands; the item stays counted."
  - "Waive: the case resumes where it was; the deviation is recorded; the expected return condition is unchanged."
  - "False report: the case resumes where it was; nothing else changes."
  - "Quarantine on return: the asset is quarantined for 'damaged_on_return', its hold released, and it is not restocked."
  Render the sentence exactly as given; do not rewrite it.

QUARANTINED MARKER. On the last node of a lane whose asset is quarantined show a red-brown tag "[Q] Quarantined
(damaged_on_return)" (icon plus text) that links to /assets/SP-01. A lane that is expected to come back damaged shows
a chip "Expected back damaged" on its Return step. Test id lane-card__quarantined.

SUPPLEMENTARY LANE. L6 SP-03: dashed border, caption "supplementary · depth 1", a dashed edge from L1 labelled
"supplements". The Join text ends "(1 supplementary ignored)". A replacement lane (L5 HT-02) uses the existing
"replaced by" edge from L3 and has a solid border.

INTERRUPTS BAND (always drawn at the bottom of the graph card, even when nothing fired)
- A thin band titled "Interrupts" with three locked nodes in a row: "expire", "tombstone_cancel", "admin_reset", each
  with a padlock icon and no connection handles. Tooltips: "Cancels the request when the activity window passes before
  owner approval", "Cancels the request when the requester is deactivated", "Resets the request to the requester".
- A fired node is highlighted with the text "fired 14:07 by Greg Global" and a "cancelled by" edge to each instance it
  cancelled. Unfired nodes show no highlight. Provide a mock for RQ3-X (a sidecar request "BG-HRIS-20261108-011") where
  admin_reset fired and the lane's H1 exception shows "Cancelled by reset".

READ-ONLY HANDLER GRAPH (opened by "Show handler graph")
- Inside the band, a smaller canvas, no palette, no handles, no editing. Nodes: Start (flag), Investigate (magnifier),
  Approval (shield with check), Resolution (square, thick border, "■ Replace"), each with an icon and a text label;
  ports are labelled text next to the handle: recovered, false_report, replacement, write_off, waiver, quarantine,
  approve, reject.
- Visited nodes carry the actor and time ("✓ Grace Neo 14:07"); a node visited twice shows "×2". Taken edges are
  highlighted; untaken edges are grey. The resolution that was reached shows its effect card.
- It renders from the handler of the lane template pinned to this request's run (Standard custody lane v1), so it still shows version 1 after version 2 is published. Under the canvas a Body SM line reads "Handler of Standard custody lane v1".
- A close text button "Hide handler graph".

JOIN, SUMMARY AND STATES. The Join trace follows Prompt 5 and must update for each of the four demo states. Loading: the
band shows a skeleton row. Hidden request: nothing is rendered because the whole page is 404. Mobile: bands stack
full-width; the chain wraps to two lines.

ACCESSIBILITY. Phase and resolution are text, not colour alone; the band has the aria-label above; the toggle is a
button with aria-expanded; focus order is header, route reasons, chain, toggle.

TEST IDS. exception-band__header (data-exception-no, data-phase), exception-band__route-chip, exception-band__skipped,
exception-band__resolution-badge, exception-band__toggle-graph, lane-card__supplementary, lane-card__quarantined,
join-trace__text, interrupts-band__node (data-kind, data-fired), interrupts-band__cancelled-by.
````

---

## Prompt 7: request page, action panel

**Mirrors:** 219 §38.2 "Action panel" table, §8 (who may act), part 4 §21.2 (node kinds), §9.1 (custody actions), ST-5, ST-6, ST-7, ST-8,
ST-9, ST-19 US-19.3, ST-24. **Needs:** Prompts 4 to 6. Buttons open the dialogs of Prompt 8.

````
Using the app shell, SGDS styling and mock data already in this file, fill the "Action panel" card in the right rail of
the request page. The panel is driven by data: it lists only the actions the signed-in person may take now, and for
anything blocked it shows the button disabled with the reason. It never shows a button the person could never use.

PURPOSE AND WHO. Every person who can open the request sees the card. Eligible people see action groups; other people
see one read-only line "Waiting for {assignee}" per active step. PA personas and old runs see "You can view this
request. You cannot act on it." The card heading is Heading MD "What you can do".

CARD ANATOMY
- Heading, then one group per active step the person can act on, each group an sgds-card nested: title (the step name
  and, for item steps, the asset code, for example "Prepare and hand over · HT-02"), a Body SM line "Due 4 Nov, 6:00pm"
  or "Overdue 2h", then a row of buttons, then (if any) a field area. The first group is open; others are
  collapsible (sgds-accordion).
- Buttons: primary for the main action, outline for the others, danger tone for destructive ones (reject, write off).
  All disabled buttons use aria-disabled and show the reason as Body SM text directly under the row with the id used
  in aria-describedby (never only a tooltip).
- If the person has nothing to do: Body MD "Nothing for you to do on this request right now." and the waiting lines
  below it.
- Waiting lines: "Waiting for Ops Reviewers L1" (team), "Waiting for Carol Sim, Dan Teo or a delegate" (owner), "Waiting
  for the loss report from Bob Lim" (task), "Waiting for Frank Chua" (single person), "Waiting for the timer" (wait
  node, with the fire time "fires at 9:30am").

ACTIONS BY STEP KIND (use the exact button labels in quotes)
1. Approval step (a reviewer in the node's team): buttons "Approve", "Return" (opens the notes dialog, notes required),
   "Reject" (notes required, danger). Where the node allows override and the person is in a later team: the extra button
   "Approve as override" with helper "You are approving for a team that has not acted." Required fields of the node
   appear above the buttons: Ops Reviewers L1 asks "Risk level" (select low, medium, high; required); Ops Leads L2 asks
   "Security score" (number 0 to 5; required); Asset Store L1 asks "Logistics score" (number 0 to 5; required). Field
   errors: "Choose a risk level." and "Enter a score from 0 to 5."
   Disabled reasons (authored in the same style):
   - the requester is also a reviewer: "You submitted this request, so you cannot approve it."
   - already decided in a parallel review: "You already decided another branch of this review."
   - another team member already decided: "Someone else already decided this step."
   - not eligible to override: "You can only approve steps for your own team."
   - delegation window over: "Your delegation is not active."
   - rule input locked after an approver edit: "Item changes would alter a decision already taken. Return the request instead."
2. Approver edit (the person in the current approval team, before the owner stage): a link-style button "Edit
   request details" (opens the edit drawer of Prompt 8). Helper: "Correct the justification, SR number, window or items.
   Every change is recorded."
3. Owner approval (primary owner, backup owner or an active delegate): "Approve", "Return", "Reject", same rules. If the
   person is also the requester (Ivan Ng on RQ10): "Approve" is aria-disabled with the reason "You submitted this request,
   so you cannot approve it." and the line "Another owner can approve: Carol Sim."
4. Amend step (the original submitter only, status returned to vendor): the prominent primary button "Edit and resubmit"
   (opens the resubmit drawer). Others see "Waiting for Bob Lim to revise".
5. Custody item steps (the stage team named in the node; all combined labels are exact):
   - awaiting preparation, one team does both: "Prepare and initiate handover"; two teams: "Prepare" for the first
     team and "Initiate handover" for the second.
   - a token's first step: "Complete reset PIN" with a tick "PIN has been reset" (required) for Asset Store L1.
   - handover pending, the requester: "Confirm receipt" and "Dispute handover" (notes required).
   - with the requester: "Initiate return" (the requester); a "Report a problem" button for the requester and stage teams.
   - return pending: "Verify return" for the verifying team with a required select "Condition" (Good, Damaged); if the
     same team also restocks: "Verify return and return to inventory"; the team can also "Dispute return" (notes).
   - received by agency: "Return to inventory".
   Each item is its own group with the asset code, so acting on SP-01 never changes HT-01.
6. Wait step or timer: read-only "Waiting for the timer · fires at 9:30am".
7. Close request task (the submitter or any reviewer, once the request is credential issued): "Close request" (opens a
   notes dialog with optional notes). Helper: "You can close the request while items are still out. Returns continue."
8. Platform: "Reset request…" is not here; it lives in the page header menu.
9. Report a problem (any live item for the requester and the stage teams): button "Report a problem" (opens the report
   dialog). Disabled reason when one is already open on that item: "A problem is already open for this item".

EXCEPTION ACTIONS (from the data for the open problem; an eligible actor sees the node's buttons)
- Investigate step in status "Open", the node's team: "Start investigating" (notes required).
- Investigate step in status "Investigating", the node's team: "Mark as recovered" (select "Resume state", notes);
  "Report as false alarm" (notes required); "Propose replacement" (select "Substitute", notes required); "Propose
  write-off" (notes required); "Propose waiver" (deviation notes required); "Quarantine on return" (notes required);
  "Ask for information" (notes required). Offer only the ports the handler has; if the per-item step disallows write-off,
  there is no write-off button at all.
  - Replacement disabled reasons, first match wins, shown as text: "Replacement is not allowed after issuance for this
    system" · "Replacement limit reached (1/1)" · "No substitute available".
  - The substitute select lists only candidate substitutes of the same credential type that are free and not quarantined;
    if the chosen one is taken before submit the dialog shows SUBSTITUTE_INVALID copy.
- Investigate step in status "Awaiting information": every port button is aria-disabled with the reason "On hold for
  information. Continue investigating first", and one enabled button "Continue investigating" (notes required).
- Resolution approval step (the owner class, or the named team): "Approve" and "Reject" (notes required). Disabled
  reasons: "You investigated this exception, so you cannot approve it" · "You reported this exception" · "You are the
  requester". A person outside the assignee set sees only "Waiting for {assignee}".
- Task "loss report" (the submitter or the named team): "Complete loss report" with a required tick "I have completed the
  loss report" and optional notes. Non-assignees see "Waiting for the loss report from Bob Lim".

MOCK AND PERSONA MATRIX (build all, switch persona to check)
- RQ3 state B: Grace Neo sees the investigate group for SP-01 with "Mark as recovered", "Report as false alarm", "Propose
  replacement" (disabled "Replacement limit reached (1/1)" is NOT used here; replacement is allowed), "Propose
  write-off", "Propose waiver", "Quarantine on return", "Ask for information". Claire Wong sees "Waiting for Grace Neo".
  Carol Sim sees "Waiting for Grace Neo" and, after the proposal in the demo, "Approve" and "Reject" on the approval step
  (Frank Chua sees the same for the H3 team approval and Carol is blocked because the H3 approval is a team step: "Waiting
  for Ops Leads L2"). Bob Lim sees "Initiate return" for SP-01, HT-02 and "Report a problem", and "Close request".
- RQ3 state A: Grace Neo sees "Prepare and initiate handover" for SP-01, SP-02 (excluded; no group), LT-01 and HT-02 with
  the extra "Complete reset PIN" first on HT-02 (a token). Bob Lim sees "Waiting for Asset Store L1".
- RQ7: Aaron Koh sees "Approve", "Return", "Reject" for step 2; Alice Tan sees the same card plus the amber chip
  "Escalated · from Alpha Leads L2"; Bob Lim sees "Waiting for Alpha Leads L2".
- RQ2: Bob Lim sees "Edit and resubmit"; Aaron Koh sees "Waiting for Bob Lim to revise".
- RQ10: Ivan Ng sees the disabled "Approve" (submitted it) and the line about Carol Sim; Carol Sim sees "Approve",
  "Return", "Reject".
- RQ6: Brian Goh sees the approval buttons; Frank Chua (a later team in the chain, override allowed) sees "Approve as
  override".
- Priya Nair on any approved request: "You can view this request. You cannot act on it."

STATES. Loading: skeleton group. Error: sgds-alert danger inline with the code. After pressing an action, the button
shows a spinner and "Working…", the card keeps its form values on failure, shows the server message (copy table of
Prompt 0B) in an sgds-alert at the top of the card with focus moved to it, and refreshes the offer; no optimistic change
is shown. NODE_NOT_ACTIVE reads "This step has already moved on. Refresh to see the latest." with a "Refresh" button.
Success: a toast ("Request approved.", "Returned to vendor.", "Request rejected.", "Receipt confirmed.", "Request
closed.", "Problem reported.") and the live indicator updates. Engine down: all buttons aria-disabled with the reason
"Workflow engine unavailable. Actions are disabled."

ACCESSIBILITY. The card is a region "Actions for this request". Groups are headings. Disabled buttons expose their reason
through aria-describedby. Do not move focus on live refresh except to the alert after a failure.

TEST IDS. exception-actions__port (data-port, aria-disabled), exception-actions__reason, exception-actions__approve,
exception-actions__reject, exception-actions__notes, action-panel__group (data-node-key), action-panel__waiting,
action-panel__approve, action-panel__return, action-panel__reject, action-panel__close.
````

---

## Prompt 8: request page, action dialogs and drawers

**Mirrors:** 219 §38.2 action table forms, ST-4, ST-5 US-5.2 and US-5.6, ST-7 US-7.1, ST-9 (report, resolve), ST-18 US-18.2, §38.4 error copy.
**Needs:** Prompts 0B, 4, 7.

````
Using the app shell, SGDS styling and mock data already in this file, build the dialogs and drawers that the action
panel (Prompt 7) opens. Use sgds-modal (size md) for short forms and sgds-drawer (right side, 480 px; full width on
mobile) for long forms. All dialogs trap focus, close on Escape, return focus to the button that opened them, and show
the shared error component of Prompt 0B at the top on failure while keeping every field value. Primary buttons keep an
aria-disabled state with a visible reason until the form is valid. Footers: left "Cancel" (outline), right the verb.

1. APPROVE WITH FIELDS (modal). Title "Approve {request number}". Body shows the node's required fields: "Risk level"
   (sgds-select low, medium, high; error "Choose a risk level."), "Security score" and "Logistics score" (sgds-input
   number, 0 to 5; error "Enter a score from 0 to 5."), and an optional "Notes" (up to 2,000, counter). Primary
   "Approve". Success toast "Request approved."
2. RETURN OR REJECT (the notes dialog of Prompt 0B). Titles "Return {request number} to vendor" and "Reject
   {request number}". Helper for return: "The requester can revise and resubmit." Helper for reject: "Rejecting ends the
   request. Held assets are released." Buttons "Return to vendor" and "Reject request" (danger). Notes required. Toasts:
   "Returned to vendor." and "Request rejected."
3. OVERRIDE APPROVAL (modal). Title "Approve as override". Text "You are approving a step for a team that has not acted.
   Your name is recorded as the approver." Optional notes. Primary "Approve as override".
4. EDIT REQUEST DETAILS BY APPROVER (drawer). Title "Edit request details". Same fields and validation as the new
   request form (justification, SR number, window, items with the disabled-with-reason rows) but pre-filled; the system
   cannot be changed and is shown as read-only text with the helper "The system and the requester cannot be changed."
   A diff summary at the top after changes: "Changes: window end 5:00pm to 6:00pm; item LT-01 removed." Primary "Save
   changes". Error RULE_INPUT_LOCKED: "Item changes would alter a decision already taken. Return the request instead."
   Success toast "Changes saved." and the approval-deadline timer in the Engine tab shows the new time.
5. EDIT AND RESUBMIT BY REQUESTER (drawer). Title "Edit and resubmit". A neutral alert at the top quotes the return
   remark: "Returned by Aaron Koh: Wrong SR". Same form as above with a required new window if the old one has elapsed:
   "Enter a new end time in the future." Primary "Resubmit request". Success toast "Request resubmitted." and the page
   shows run 2 pinned to the current workflow version; the run selector appears. Error SYSTEM_WORKFLOW_INVALID: "This
   system's approval chain is invalid. Contact your organisation admin." The request stays returned.
6. CONFIRM RECEIPT / DISPUTE HANDOVER. Confirm receipt is a one-click button with a small confirm popover "Confirm that
   you hold SP-01?" (buttons "Not yet" and "Confirm receipt"). Dispute opens the notes dialog titled "Dispute handover of
   SP-01" with helper "Tell the store what is wrong. A problem is opened for this item." and primary "Dispute handover".
7. VERIFY RETURN (modal). Title "Verify return of SP-01". A required sgds-radio group "Condition": "Good" and "Damaged". If
   "Damaged" and the item was expected good, show an sgds-alert warning "Recording this as damaged opens a problem for the
   store to resolve." Optional notes. Primary "Verify return" (or "Verify return and return to inventory" for the combined
   action). Disputing a return opens the notes dialog "Dispute return of SP-01".
8. CLOSE REQUEST (modal). Title "Close {request number}?", text "You can close the request while items are still out.
   Returns continue after you close.", optional notes (up to 2,000). Primary "Close request". Toast "Request closed."
9. REPORT A PROBLEM (drawer). Title "Report a problem with {asset code}". Fields: "Type" (required sgds-select: Missing,
   Unavailable, Damaged, Wrong item, Identity mismatch, Quantity mismatch, Other; the handover, receipt and return dispute
   types appear only through the dispute buttons), "Severity" (required radio: Low, Medium, High, Critical), "Description"
   (required textarea, up to 4,000, counter; error "Describe the problem in up to 4,000 characters."), "Evidence"
   (sgds-file-upload, up to 10 MiB per file, images and PDF only, multiple; errors "That file is too large. Upload a file
   under 10 MiB." and "That file type is not allowed. Upload a PNG, JPEG or PDF."; a file whose content does not match its
   type shows "That file does not match its type."). Each uploaded file shows name, size and a remove button. Primary
   "Report problem". Disabled reason when one is already open: "A problem is already open for this item". Success toast
   "Problem reported." The lane shows the new band (Prompt 6).
10. INVESTIGATE FORMS (modal each; title names the action and asset)
    - "Start investigating SP-01": notes required.
    - "Mark SP-01 as recovered": select "Resume state" (required; for an item lost before issuance the options are the
      states before the problem; for an item lost after receipt "With requester" and "Received by agency"); error
      RESUME_STATE_INVALID copy; notes. Primary "Mark as recovered".
    - "Report as false alarm": notes required. Primary "Report as false alarm".
    - "Propose replacement of SP-01": select "Substitute" (required, only free assets of the same type; shows code,
      description and "Available"), notes (required). Errors NO_SUBSTITUTE, SUBSTITUTE_INVALID, REPLACEMENT_LIMIT,
      POST_ISSUANCE_REPLACEMENT_DISABLED, ASSET_IN_USE with the copy of Prompt 0B. Primary "Propose replacement".
    - "Propose write-off of SP-01": notes required. Primary "Propose write-off" (danger).
    - "Propose waiver for SP-01": textarea "Deviation notes" (required, hint "Describe what differs and why it is acceptable.").
      Primary "Propose waiver".
    - "Quarantine SP-01 on return": notes required; helper "The item stays with the requester. It will be quarantined, not
      restocked, when it comes back." Primary "Quarantine on return".
    - "Ask for information": notes required; helper "The problem is put on hold until you continue." Primary "Ask for information".
    - "Continue investigating": notes required. Primary "Continue investigating".
11. DECIDE A PROPOSAL (modal). Title "Approve replacement of SP-01 with SP-03?" (or "Approve write-off of LT-01?", "Approve
    waiver for SP-01?"). Shows the proposer, notes and the effect sentence from Prompt 6 in a neutral box. Buttons "Approve" and
    "Reject" (reject opens notes; rejection returns the problem to investigation: toast "Proposal rejected. Investigation
    continues."). Errors SOD_VIOLATION, ASSET_IN_USE.
12. COMPLETE LOSS REPORT (modal). Title "Complete loss report for LT-01". Required tick "I have completed the loss report";
    optional notes. Primary "Complete loss report". Without the tick the primary button is aria-disabled with the reason
    "Tick the box to confirm".
13. ADMIN RESET: use the dialog of Prompt 0B (c). Triggered from the header menu; blocked while closed.
14. STALE AND CONFLICT. Any dialog may receive STALE_STATE, NODE_NOT_ACTIVE or IDEMPOTENCY_TOKEN_USED; show the copy and a
    "Refresh" button; the form keeps its values.

DEMO PANEL. A collapsed "Demo controls" drawer opener at the bottom of the page lists a button per error code above to
force that error on the next submit, so the error states can be reviewed.

ACCESSIBILITY. Every field has a visible label; errors are linked with aria-describedby; the drawer is a dialog with a
heading; the file upload has a keyboard path and announces added and removed files politely.

TEST IDS. exception-actions__notes, exception-actions__approve, exception-actions__reject, action-dialog__submit,
action-dialog__cancel, action-dialog__error, report-problem__type, report-problem__severity, report-problem__description,
report-problem__evidence, edit-request__submit.
````

---

## Prompt 9: request page, side panels, issuance and returns, timeline

**Mirrors:** 219 §12 panels, ST-1 US-1.5, ST-8 US-8.6, ST-10 US-10.1, ST-11 US-11.4, ST-18 US-18.1 and US-20.5, §38.2 (columns, timeline).
**Needs:** Prompts 4 to 8.

````
Using the app shell, SGDS styling and mock data already in this file, fill the right-rail tabs (Record, Engine, Variables)
and the two cards under the graph (Issuance and returns, Timeline) on the request page.

TABS (sgds-tab-group, underlined variant, in the right rail; tab labels "Record", "Engine", "Variables")

1. RECORD (the facts in the database)
- sgds-description-list: Request number, System, Organisation, Submitted by (name, with "(deactivated)" after the name if
  the person was deactivated), SR number, Justification (full text, wrapping), Window start, Window end, Status, Return
  count, Resubmit count, Credential issued at (when set), Closed by, Closed at, Closing notes.
- Section "Items": an sgds-table with columns Asset, Type, Description (hostname or description), Status ("Live",
  "Replaced", "Written off", "Supplementary"). Excluded items are struck through with the reason in text.
- Section "Remarks and decisions": a list of every decision with remarks, newest first, each card showing actor, role,
  decision (Approved, Returned, Rejected), time and the remark in a quote block. Example: "Aaron Koh · Returned · 8 Nov 2026,
  9:15am: Wrong SR".
- Section "Audit": a compact table (Time, Actor, Action, Target) showing the last 10 entries with a link "Open full timeline" that
  scrolls to the Timeline card. Action examples: "request.submitted", "request.step_approved", "request.owner_approved",
  "custody.run_started", "request.credential_issued", "request.closed".

2. ENGINE (what the workflow service is doing). Read-only. Shows "Source of truth: database" as Body SM above.
- Current wait: one line, for example "Waiting for approval at Ops Reviewers L1" or "Waiting for custody (2)" for a closed
  request with returns outstanding, "Completed" for a finished workflow.
- Armed timers: an sgds-table with columns Timer (Approval deadline, Stage SLA, Exception SLA, Return due, Return overdue,
  Wait until), Fires at (date and time in SGT) and a Countdown that ticks every second in the prototype ("in 3 min 12 s",
  "fired 2 h ago"). Note under it for demo mode: "Demo mode shortens hour-based timers to seconds."
- Child workflows: a list of one row per item "Lane L1 SP-01 · Running" with the state word.
- History length: "1,284 events" (Body SM) and an sgds-progress-bar to the 2,000-event limit labelled "Run history 64% of limit".
- Run ID chain: a vertical list "Run 2 (current) → Run 1" with run IDs in small print.
- Link "Open in Temporal UI" (sgds-link, external icon, opens a new tab): http://localhost:8233/namespaces/breakglass-mvp/workflows/{id}.
- When the history is archived: "History archived" with the SHA-256 prefix and the date, and the panels still show data.
- Consistency: a Body SM line "Record and engine agree" with a check icon, or a warning alert listing the difference if the demo
  flag "drift" is on ("The record and the engine differ: 1 item").
- Engine down: the tab shows "Engine unavailable" and keeps the last data with a "Last updated" time.

3. VARIABLES (typed values of the workflow)
- Intro Body SM: "These values drive the rules in this workflow."
- An sgds-table with columns Name, Value, Type, and an expandable history. Rows for RQ3: item_count 4 (int), window_hours 7 (decimal),
  activity_start_at 10:00am (timestamp), risk_level high (enum), high_risk true (bool), security_score 3 (int 0..5),
  logistics_score 3 (int 0..5), review_score 6 (int), required_score 6 (int, constant, with a lock icon and the note
  "Constant. Changed only in a new version."), items_received 3 (int), pins_reset 1 (int). Then one group per embedded call and lane, for example "Dual review v1 · call 1" (review_score, security_score, logistics_score, required_score) and "Lane 2 · HT-01" (pin_reset_done); then "Platform" (item_count, window_hours, and per lane item.credential_type).
- Expanding a row shows its change history: Time (SGT), Actor, Node, New value, Source (init, system, field, set_variable, binding, reduction, lane delta). Example for pins_reset: "10:05am · reduction · Items · Count true of pin_reset_done · 1". Example for
  review_score: "9:30am · system · Add security score · 3" and "9:31am · system · Add logistics score · 6".
- Rule evaluations: a section "Rule checks" listing each rule with its rendered evaluation, for example
  "Score ok?: review_score (6) >= required_score (6) -> true". For RQ9 show "review_score (5) >= required_score (6) -> false".

ISSUANCE AND RETURNS (card under the graph; sgds-card)
- Heading MD "Issuance and returns". Beside it the button "Download CSV" (sgds-button outline, icon) that downloads the same rows as the
  table (toast "CSV downloaded."), and a link "See asset history" per row.
- Summary counts row, as six statistic tiles (label above the number, never colour alone): "Items issued", "Outstanding" (issued and not
  returned), "Returned", "Overdue" (return overdue), "Replaced", "Written off", plus "Returned damaged" and "Replaced after issuance".
  Examples: state B shows issued 3, outstanding 3, returned 0, overdue 0, replaced 1, written off 2, returned damaged 0, replaced after
  issuance 0. State D (final) shows total 6, issued 4, outstanding 0, returned 3, returned damaged 1, replaced 1, replaced after issuance 1,
  written off 2.
- Per-item table (sgds-table, caption "Issued and returned items"; columns exactly: Asset, Lane origin (Initial, Replacement, Supplementary),
  Replaces or replaced by (asset codes), Issued to, Handed over by, Issued at, Return due, Returned at, Verified by, Restocked at, Return
  condition (Good, Damaged, or a dash), Quarantined at, Outcome (Outstanding, Returned, Returned damaged, Replaced, Written off). The Asset
  cell is a link to /assets/SP-01. Outstanding rows with an overdue return show the red badge "Return overdue 3h". Mobile: stacked cards.
- Mock rows RQ3 state D: SP-01 Initial, issued to Bob Lim, handed over by Grace Neo, returned damaged, verified by Claire Wong, quarantined at 4:12pm, outcome
  Returned damaged; HT-02 Replacement of HT-01 (shown "Replaces HT-01"), returned, restocked by Grace Neo; SP-03 Supplementary (supplements
  SP-01), returned; LT-01 Initial, issued, outcome Written off; SP-02 and HT-01 never issued, outcomes Written off and Replaced. State B and C show
  matching partial rows.
- Empty: "Nothing has been issued yet. Items appear here after the requester confirms receipt."
- Access: the report link (to /reports/issuance-returns) appears only to owners, investigate-team members, organisation admins (for the organisations
  their grants name, 219 D-38) and global admins.

TIMELINE (card under the issuance card; sgds-card)
- Heading MD "Timeline". A filter chip row: All, Approvals, Custody, Problems, System (single choice; sgds-radio styled as chips).
- A vertical list, newest first, times in SGT (for example "8 Nov 2026, 2:07pm"), each entry with an icon, the sentence, actor chip and
  time. Tombstoned actors show "(deactivated)" after the name. System events are labelled "System".
- Use these sentences exactly (substitute actors and assets):
  - "Bob Lim reported SP-01 as damaged (medium): {description}"
  - "Routed to H3 Damaged in custody (rule 2 of 5). Skipped: H1, phase pre_issuance ≠ in_custody"
  - "Grace Neo started investigating SP-01"
  - "Grace Neo asked for information: {notes}" and "Grace Neo continued investigating"
  - "Investigation of SP-01 is overdue (8 h). Escalated to Ops Leads L2"
  - "Grace Neo proposed a replacement of SP-01 with SP-03" (write-off and waiver in the same form)
  - "Frank Chua approved the replacement" and "Carol Sim rejected the proposal: {notes}. Investigation continues"
  - "Replacement applied: SP-01 stays with the requester (expected back damaged); SP-03 held for a supplementary lane"
  - "LT-01 written off; issuance stands" · "HT-01 recovered; resumed at preparation" · "Waiver granted: {deviation}" ·
    "Report judged a false alarm"
  - "Claire Wong recorded SP-01 as damaged on return" and "SP-01 quarantined (damaged on return); not restocked"
  - "Supplementary lane started for SP-03" and "Admin reset fired; 1 exception cancelled"
  - Authored lines in the same style for ordinary events: "Bob Lim submitted the request", "Aaron Koh approved step 2 (Alpha Leads L2)",
    "Aaron Koh returned the request: {notes}", "Carol Sim approved as owner", "Grace Neo prepared and handed over HT-02", "Bob Lim
    confirmed receipt of HT-02", "Credential issued", "Bob Lim closed the request", "Return due for 3 items", "Return overdue for SP-01",
    "The request expired because the activity window passed", "Bob Lim initiated the return of SP-01", "Claire Wong verified the return of SP-01".
- Each entry that carries a remark shows it in a quote block. "Show more" loads 20 more entries.
- Empty: not applicable (a request always has a first entry).

POLLING AND STATES. All panels refresh every 2 seconds while the page is visible (the "Live" indicator of the header) and stop when the
workflow has completed. Loading: skeleton per card. Error: sgds-alert with the code and "Try again". Old run selected: all panels show that run's
data and a read-only note. Mobile: tabs become an sgds-accordion; tables stack.

ACCESSIBILITY. Tab group follows the SGDS tab pattern with arrow-key navigation; countdowns are not read out every second (aria-hidden on the ticking
part, with a static text alternative "in about 3 minutes"). Tables have captions. The CSV button states the file name in its accessible name.

TEST IDS. record-panel__items, engine-panel__timer, engine-panel__run-chain, engine-panel__temporal-link, variables-panel__row (data-variable),
issuance-panel__summary (data-metric), issuance-panel__row (data-asset), issuance-panel__csv, timeline__entry (data-event), timeline__filter.
````
