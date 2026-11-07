# Addendum 220 (part 2) — Figma Make prompts: dashboard, new request, request page shell

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only. Tag `[CUSTOM: design handoff]`.
**Parts:** part 1 `addendum-220-figma-make-screen-prompts.md` (how to use, Prompts 0, 0B, 1); **this part** (Prompts 2 to 4); part 3 `…-part-3-request-graph-actions-and-panels.md` (Prompts 5 to 9); part 4 `…-part-4-reports-assets-and-admin.md` (Prompts 10 to 19); part 5 `…-part-5-wizard-versions-and-editor-layout.md` (Prompts 20 to 24); part 6 `…-part-6-editor-tabs-simulation-and-dialogs.md` (Prompts 25 to 30).
**Normative source:** addendum 219 (219 wins on any conflict). Paste the prompts in order, in the same Figma Make file that already holds Prompts 0 and 0B.

---

## Prompt 2: dashboard

**Route:** `/`. **Who sees it:** every signed-in person; the content depends on the role (219 §8, §12, §38.2).
**Mirrors:** 219 §12 dashboard, ST-10 US-10.3, ST-11 US-11.1, §38.2 "Dashboards", ST-9 US-9.8, ST-6 US-6.2, ST-21 US-21.2.
**Needs:** Prompts 0 and 0B.

````
Using the app shell, SGDS styling and mock data already in this file, build the dashboard at "/".

PURPOSE AND WHO. It answers one question: "What do I need to do now?" Everyone who is signed in sees it. It has
three sections, shown only when they apply to the person: "Needs my action", "My requests", and (PA personas
only) "Requests for the owners I support". Global admins also see an operations banner when the mock flag
"archiveLagging" is on.

LAYOUT (desktop, 1024 px and above). Fluid SGDS container. Page heading Heading XL "Dashboard"; under it Body SM
"Signed in as {name} · {organisation} · {role}". Then, top to bottom, full-width sections separated by the SGDS
layout gap. "Needs my action" comes first and is visually strongest (sgds-card with a primary top border).
Tablet: same, rows wrap their badges below the title. Mobile: rows become stacked cards; the action button goes
full width under the text.

  Dashboard
  Signed in as Grace Neo · AUR · Agency, Asset Store L1
  +- Needs my action (2) -----------------------------------------------+
  | [!] Investigate SP-01 (damaged) · BG-HRIS-20261108-003 · H3         |
  |     HRIS   [Overdue 2h]                                [Open]       |
  | [ ] Prepare and hand over HT-02 · BG-HRIS-20261108-001   [Open]     |
  +---------------------------------------------------------------------+
  +- My requests (0) ---------------------------------------------------+
  | No requests yet. ... (hidden for people who cannot submit)          |
  +---------------------------------------------------------------------+

SECTION 1. "NEEDS MY ACTION" (count in brackets)
- A list of rows (not a data table). Sort: overdue first (largest overdue first), then oldest first.
- Row anatomy: a leading status icon with a text label for screen readers; the row text (Body MD semibold); under it
  Body SM meta: the request number as a link, a neutral sgds-badge with the system short code (HRIS), and an optional
  suffix such as "H3". Right side: badges, then the button "Open" (sgds-button outline, size sm) that goes to the
  request page. The whole row is also a link.
- Badges: "Overdue {n}h" = sgds-badge danger (filled), with a clock icon, shown when the item is overdue. "Escalated"
  = sgds-badge warning with the text "from {team}" in the same badge (for example "Escalated · from Asset Store L1");
  shown ONLY to the escalation team, never to the original team. "Return due" = sgds-badge warning outlined, shown
  to the requester when the activity window has ended and items are still with them.
- The four exception row types, with exactly this text:
  1. "Investigate SP-01 (damaged) · BG-HRIS-20261108-003 · H3" is shown to the investigate team (Grace Neo and Leo Tan
     for H3).
  2. "Approve replacement of SP-01 with SP-03" is shown to the approver (Frank Chua for the H3 team approval; Carol Sim,
     Dan Teo or Emma Yeo for an owner-class approval).
  3. "Complete loss report for LT-01" is shown to Bob Lim (or the submitter's team).
  4. "Quarantine or write off SP-01 (damaged return)" is shown to Ops Reviewers L1 (Claire Wong; for H4).
- Other row types, authored in the same style: "Approve BG-HRIS-20261108-007 · Step 2 · Alpha Leads L2";
  "Approve as owner BG-OPSTOOL-20261108-001"; "Revise and resubmit BG-HRIS-20261108-002 · returned by Aaron Koh";
  "Return due: 3 items · BG-HRIS-20261108-001"; "Prepare and hand over HT-02".
- Empty state: sgds card text "Nothing needs your action." with Body SM "New work appears here when it reaches you."

MOCK ROWS PER PERSONA (build all; the persona switcher changes the content)
- Bob Lim: "Return due: 3 items · BG-HRIS-20261108-001" (Return due badge); "Revise and resubmit
  BG-HRIS-20261108-002 · returned by Aaron Koh" (secondary line: remark "Wrong SR").
- Aaron Koh: "Approve BG-HRIS-20261108-007 · Step 2 · Alpha Leads L2" with "Overdue 2h".
- Alice Tan: the same request with "Escalated · from Alpha Leads L2" and "Overdue 2h". (Aaron does not see "Escalated".)
- Brian Goh: "Approve BG-FINDB-20261108-001 · Step 2 · Beta Leads L2".
- Claire Wong: "Quarantine or write off SP-01 (damaged return) · BG-HRIS-20261108-005 · H4".
- Grace Neo and Leo Tan: "Investigate SP-01 (damaged) · BG-HRIS-20261108-003 · H3" with "Overdue 2h".
- Frank Chua: the same investigate row with "Escalated · from Asset Store L1" and "Overdue 2h".
- Carol Sim: "Approve as owner BG-OPSTOOL-20261108-001".
- Dan Teo, Emma Yeo, Ivan Ng, Betty Ong, Hank Low, Sam Lee, Gina Ho, Amanda Scoped, Greg Global, Priya Nair: empty
  state (Sam Lee sees "Approve as owner BG-PORTAL-20261108-001").
- A "Row gallery" link in the footer of the page (design review only) opens /design/dashboard-rows that shows all
  four exception row types and the other types stacked, for any persona.

SECTION 2. "MY REQUESTS" (only for personas who can submit: Alice, Bob, Betty, Claire, Emma, Ivan, Gina)
- Heading with count and, on the right, the button "New request" (sgds-button primary) to /requests/new.
- An sgds-data-table (caption "My requests") with columns: Request number (link), System, Status (status pill from
  the shared language), Items (count), Updated (date and time), and a trailing sgds-overflow-menu "Open". A filter
  row above: sgds-select "Status" (All, Submitted, Under review, Returned to vendor, Rejected, Approved, Credential
  issued, Closed). Sortable by Updated (default, newest first). 10 rows per page with sgds-pagination.
- Special status text in the table: a closed request with returns outstanding reads "Closed · 2 items outstanding";
  a request with an open problem has a danger sgds-badge "1 problem open" next to the status pill.
- Bob's rows: RQ1 credential issued, 3 items; RQ2 returned to vendor; RQ3 credential issued (problem open); RQ4
  closed (the "Closed – unfulfilled" reason in Body SM under the pill); RQ5 credential issued (problem open); RQ7
  under review (Overdue 2h shown as a badge); RQ8 rejected (Body SM "Expired: the activity window passed before
  owner approval"); RQ9 rejected (Body SM "Stopped by a rule: the review score was below the required score").
  Betty: RQ6. Gina: RQ11. Ivan: RQ10.
- Empty state: "No requests yet. Create your first request." with a primary "New request" button.
- Loading: five skeleton rows. Error: sgds-alert danger with the error code.

SECTION 3. "REQUESTS FOR THE OWNERS I SUPPORT" (PA personas only: Priya Nair)
- Read-only. Body SM under the heading: "You can view these requests. You cannot act on them."
- sgds-table of Approved-or-later AUR requests whose primary owner is supported by Owner Support: RQ1, RQ3, RQ4, RQ5
  (columns: request number link, system, owner "Carol Sim", status pill, last update). No action buttons. Requests
  before approval are not listed. Opening a request before approval shows the 404 page.

OPERATIONS BANNER (Greg Global and Amanda Scoped is NOT shown it; global admins only)
- When "archiveLagging" is on: sgds-alert warning, not dismissible, above all sections: "The history archive is
  lagging. 3 closed workflows are not archived yet." with the code "archive: lagging" in small print.

STATES. Loading (skeleton for each section); empty (per section, above); error (sgds-alert danger with the code
and a "Try again" button); engine down (global banner; the lists still show because they come from the database).
Live behaviour: the lists refresh every 2 seconds in the real product; show the text "Updated 2 s ago" in Body SM
at the right of the heading row, with aria-live polite, and do not animate rows.

ACCESSIBILITY. One h1. Each section is a landmark region with an aria-label. Overdue is conveyed by the words
"Overdue 2h" and a clock icon, not by colour alone. Rows are keyboard focusable in reading order; Enter opens.

TEST IDS. dashboard__needs-action, dashboard__my-requests, dashboard__pa-panel, dashboard__ops-banner,
action-row__title, action-row__overdue, action-row__escalated, action-row__open, request-table__row
(data-request-number), request-table__status.
````

---

## Prompt 3: new request

**Route:** `/requests/new`. **Who sees it:** people in the live team at step 1 of at least one system's workflow, in the
system's organisation (219 §8 "Submit"; ST-4). Everyone else gets the empty "no eligible system" state; a person
from another organisation never sees a foreign system. **Mirrors:** 219 §12, ST-4, ST-10 US-10.1, ST-24 US-24.3, ST-20
US-20.1, §38.4 `ASSET_QUARANTINED`. **Needs:** Prompts 0, 0B, 2.

````
Using the app shell, SGDS styling and mock data already in this file, build the new request form at /requests/new.

PURPOSE AND WHO. A requester asks for one or more physical emergency credentials on ONE system, under ONE service
request (SR) number and ONE activity window. Only people who sit in the first team of a system's approval chain can
submit (Alice Tan and Bob Lim on HRIS; Betty Ong on FINDB; Claire Wong, Emma Yeo and Ivan Ng on OPSTOOL; Gina Ho on
PORTAL). Others who open this URL see the empty state below.

LAYOUT (desktop). Standard SGDS container, breadcrumb "Dashboard / New request". Heading XL "New request". A single
column form 640 px wide on the left (a form card), and on the right (360 px) a sticky summary card "Your request"
that updates live: system, number of items, window, and a checklist of what is still missing. Tablet: the summary card
moves above the form and becomes a collapsed accordion. Mobile: one column; summary at the bottom above the buttons.

  New request
  +- Request details ------------------------+  +- Your request -----------+
  | System (required)      [HRIS        v]   |  | System   HRIS            |
  | Justification (required) [..........]    |  | Items    2 of 5          |
  | SR number (required)   [CHG-2041     ]   |  | Window   not set         |
  | Activity window                           |  | To do: SR number,        |
  |  Start [date][time]  End [date][time]    |  |        window            |
  |  Quick window: [+1 min] [+3 min]          |  +--------------------------+
  | Items (required) 2 selected               |
  |  [x] SP-02 Secure phone  device           |
  |  [ ] SP-01 Quarantined (damaged on return)|
  |  ...                                      |
  | [Cancel]                 [Submit request] |
  +-------------------------------------------+

FIELDS (all use SGDS form components; labels above; hint text under the label)
1. System: sgds-select, label "System", required. Lists ONLY systems the person may submit to, each as
   "HRIS · Org Aurora". Bob sees only HRIS. If exactly one, it is preselected. Changing the system clears the asset
   selection (with a toast-free inline note "Items cleared because the system changed.").
2. Justification: sgds-textarea, label "Justification", hint "Why you need the credentials. Up to 2,000 characters.",
   live counter "0 of 2,000", required. Trimmed. Errors: blank "Enter a justification." ; over 2,000 "Shorten the
   justification to 2,000 characters or fewer."
3. SR number: sgds-input, label "SR number", hint "The service request or change reference, for example CHG-2041.",
   max 100 characters, required. Errors: blank "Enter the SR number."; over 100 "Shorten the SR number to 100
   characters or fewer."; duplicate live request on the same system: "Another live request on this system already
   uses this SR number. See BG-HRIS-20261108-007." (the request number appears only when the person can see that
   request; otherwise the message reads "Another live request on this system already uses this SR number."). Code
   DUPLICATE_SR in small print. The same SR is allowed on a different system and again after the first request is
   rejected or closed (helper text under the field when the error shows: "You can reuse an SR number after the
   earlier request is rejected or closed.").
4. Activity window: two rows, Start and End; each is an sgds-datepicker plus a time sgds-input (type time).
   Labels "Start date", "Start time", "End date", "End time", all required. Rules and errors: end must be after
   start: "Enter an end time after the start time."; end must be in the future: "Enter an end time in the future."
   The mock clock is 8 Nov 2026, 2:07pm SGT; show it in Body SM under the section ("Now: 8 Nov 2026, 2:07pm").
   Demo presets (only when demo mode is on): a row "Quick window" with two sgds-button outline size sm buttons
   "+1 min" and "+3 min". Pressing one sets Start to now and End to now plus 1 or 3 minutes. Hide the row entirely
   when demo mode is off.
5. Items: a group titled "Items" with hint "Choose 1 to 5 items. All items must be for this system." (5 is the
   HRIS maximum; show the system's own maximum). Render a selectable list (sgds-checkbox rows inside a bordered
   list; a search sgds-input "Filter items" above when there are more than 6). Each row: checkbox, asset code in
   semibold (SP-02), description (Secure phone), a neutral sgds-badge with the credential type (device, token,
   password), and a status text at the right.
   - Available: no status text.
   - In use: the row is disabled (aria-disabled, not selectable) with the visible reason "In use by BG-HRIS-20261108-003"
     (the request number is a link only if the person can see that request; otherwise "In use by another request").
   - Quarantined: disabled with the visible reason "Quarantined (damaged on return)" and a danger outlined sgds-badge
     "Quarantined". Tooltip: "This asset is quarantined and cannot be requested."
   - Retired assets are not listed.
   - A credential type not allowed on this system is not listed (HRIS allows device and token, not password).
   Under each SELECTED item show an optional sgds-input "Hostname or description" (max 200 characters, hint "Where
   the credential will be used.", error "Shorten this to 200 characters or fewer.").
   Item bounds errors (shown under the group, code ITEM_BOUNDS): fewer than 1 "Choose at least 1 item."; more than the
   maximum "Choose no more than 5 items for this system."
   (The bounds come from the per-item step of the system workflow, the embed "Items" that runs Standard custody lane v1 per item; HRIS allows 1 to 5, FINDB and OPSTOOL use the same bounds.)
   Mock stock: a demo select "Stock" (hidden in a small "Demo data" panel) with two values: "Fresh stock" (all HRIS
   assets available) and "After scenario S10" (default): SP-01 Quarantined (damaged on return); SP-03 In use by
   BG-HRIS-20261108-003; HT-01 In use by BG-HRIS-20261108-001; LT-01 In use by BG-HRIS-20261108-001; SP-02, HT-02 and
   LT-02 available. FINDB shows HSM-01; OPSTOOL shows KB-01 (type password); PORTAL shows HT-S1.

SUMMARY CARD (right). Rows: "System", "Items" (count of selected and maximum), "Window" (formatted "8 Nov 2026,
2:07pm to 2:10pm (3 min)"), "Approvals" (the number of steps after yours, from the workflow: "3 approvals then the
owner" for HRIS). A checklist "Still to do" lists the fields that are blank or invalid, each a link that focuses the
field.

BUTTONS
- "Submit request" (sgds-button primary). Disabled with aria-disabled, and the reason visible under the button, until
  required fields are valid: "Complete the fields in the checklist first." While submitting: spinner and "Submitting…".
- "Cancel" (outline) returns to the dashboard after a confirm if the form has changes: title "Discard this request?",
  text "Your changes will be lost.", buttons "Keep editing" and "Discard request".

SUBMIT RESULT AND ERRORS (mock)
- Success: navigate to /requests/BG-HRIS-20261108-010 with the toast "Request submitted." The new request page shows
  the Submit step green and the next approval step active (Prompts 4 and 5).
- 422 validation: an sgds-alert danger "Fix the errors below, then try again." with a link per field; focus moves to
  it; field messages from the copy above.
- 409 ASSET_IN_USE (simulate by selecting an asset that another persona "just" took): alert "One of these assets was
  taken by another request. Choose other assets." with code ASSET_IN_USE; the taken row becomes disabled "In use by
  another request"; the form keeps all values.
- 409 ASSET_QUARANTINED: "This asset is quarantined and cannot be requested."
- 403 NOT_SUBMITTER: "You are not in the first team of this system's approval chain." code NOT_SUBMITTER.
- 404: the neutral 404 page (the system belongs to another organisation).
- 503 ENGINE_UNAVAILABLE: "The workflow service is not available. Try again shortly." The form keeps its values.
- Double click: the second submit is ignored (the same idempotency token), no second request.

EMPTY / NOT ELIGIBLE STATE. If the person has no eligible system: a card with Heading MD "You cannot create requests
yet", Body MD "You need to be in the first team of a system's approval chain. Ask your organisation admin to add you."
and a link "Back to dashboard".

STATES. Loading: skeleton for the form while systems and assets load. Per-field validation runs on blur and on submit.

ACCESSIBILITY. The items group is a fieldset with a legend; each disabled row keeps its reason in aria-describedby. The
live summary is aria-live polite and does not steal focus. Do not use colour alone for disabled states.

TEST IDS. new-request__system, new-request__justification, new-request__sr-number, new-request__window-start,
new-request__window-end, new-request__preset (data-minutes), asset-picker__item (data-asset, aria-disabled),
asset-picker__reason, new-request__submit, new-request__error.
````

---

## Prompt 4: request page shell and header

**Route:** `/requests/[id]` (for example `/requests/BG-HRIS-20261108-003`). **Who sees it:** the submitter; reviewer teams
(any snapshot step) before the owner stage; owners and delegates from the owner stage; stage teams from approval; the
primary owner's PA from approval (read-only); an organisation admin whose grants name the request's organisation (Amanda and Mona Multi for AUR; Mona for SEN; 219
D-38); global admins. Everyone else,
including every person of another organisation, gets the neutral 404 (219 §8). **Mirrors:** 219 §12, §38.2, ST-7, ST-8
US-8.5, ST-9 US-9.5, ST-11, ST-20 US-20.5, ST-17 US-17.2. **Needs:** Prompts 0, 0B, 2. **Extended by:** Prompts 5 to 9.

````
Using the app shell, SGDS styling and mock data already in this file, build the shell of the request page at
/requests/:requestNumber. This prompt builds the page frame, the header, the banners, the layout regions, the run
selector, the "live" indicator and the showcase mock request RQ3 with a four-state switcher. The workflow graph, the
action panel and the side panels are separate regions that later prompts fill in; render clearly labelled placeholders
for them now (each in an sgds-card with its heading) so the layout can be reviewed.

WHO SEES WHAT. Allowed to open: the submitter; people in the reviewer teams of the request's approval steps (before the
owner stage); owners, backup owner and active delegate (from the owner stage); stage teams (from approval); the PA team
of the primary owner (from approval, read-only: no action buttons anywhere); an organisation admin whose grants name the
request's organisation (219 D-38); global admins. Everyone else, including every AUR persona opening an SEN request such as
BG-PORTAL-20261108-001, and every SEN persona opening an AUR request, sees the neutral 404 page ("We cannot find that
page."), identical for a missing and a hidden request.

LAYOUT (desktop, 1280 px and above; fluid container with SGDS layout padding)
- Row 1: breadcrumb "Dashboard / Requests / BG-HRIS-20261108-003".
- Row 2, the header card (full width).
- Row 3, banners (only when they apply).
- Row 4, two columns: main column (8 of 12) and right rail (4 of 12, sticky below the nav, scrolls inside itself).
  Main column, top to bottom: Workflow graph card (tall, at least 560 px), Issuance and returns card, Timeline card.
  Right rail, top to bottom: Action panel card, then an sgds-tab-group with tabs "Record", "Engine", "Variables".
- 1024 to 1279: same, with the right rail 5 of 12. 768 to 1023: one column in this order: header, banners, Action
  panel, Workflow graph, tabs (Record / Engine / Variables), Issuance and returns, Timeline. Under 768: same order,
  the graph card is 420 px tall with a "Open full screen" button, tables become stacked cards.

  BG-HRIS-20261108-003  [Credential issued]  HRIS · Org Aurora        Run 1 v   Live · updated 2 s ago
  Submitted by Bob Lim · SR CHG-2041 · Window 8 Nov 2026, 10:00am to 5:00pm
  [!] Attention required: 1 custody exception
  +- Workflow graph ---------------------------------+ +- Action panel ------+
  |                                                   | |                     |
  |                                                   | +- Record|Engine|Vars-+
  +---------------------------------------------------+ |                     |
  +- Issuance and returns ----------------------------+ +---------------------+
  +- Timeline ---------------------------------------+

HEADER CARD (sgds-card)
- Left: Heading LG with the request number (BG-HRIS-20261108-003) and a copy-to-clipboard icon button (aria-label
  "Copy request number"; toast "Request number copied."). Next to it the status pill from the shared language.
  Under it Body MD: "{System name} · {Organisation name}" ("HRIS · Org Aurora"). Under that Body SM: "Submitted by
  Bob Lim · SR CHG-2041 · Window 8 Nov 2026, 10:00am to 5:00pm". If the status is rejected or closed unfulfilled, show
  the reason line from the shared language under the pill.
- Right: the run selector, an sgds-select labelled "Run" with options "Run 2 · HRIS v2 · current" and
  "Run 1 · HRIS v1". It is shown only when the request has more than one run (RQ2 and RQ3 variants have two; RQ3 has
  one by default). Choosing an older run shows a neutral sgds-alert "You are viewing run 1. It is read-only." and
  switches the graph, the action panel and the variables to that run; actions are hidden for an old run.
  Beside it the live indicator: a dot and the text "Live · updated 2 s ago" (Body SM) with aria-live polite; the
  indicator pauses with the text "Paused" when the tab is hidden, and shows "Completed" and stops when the workflow
  has completed. When the engine is down it reads "Engine unavailable".
- Under the title row, a line of header facts as a sgds-description-list in two columns on desktop: Items (count),
  Updated (date and time), Return count (number), Resubmit count (number). On mobile it is one column.
- Overflow menu (sgds-overflow-menu "More actions", only for people who may use them): "Reset request…" (opens the admin
  reset dialog from Prompt 0B; shown to people with the reset permission: Amanda Scoped for AUR requests and Greg
  Global for all; hidden for a closed request; for a closed request show the item disabled with the reason "Closed
  requests cannot be reset"); "Download history archive" (global admins, and organisation admins whose grants name the request's organisation; text "Download archived
  workflow history (SHA-256 shown after download)").

BANNERS (stack under the header; each is an sgds-alert, not dismissible)
- Attention: danger, title "Attention required: N custody exceptions" (singular "Attention required: 1 custody
  exception"), counting the open exceptions of THIS request only. Body SM link "Go to the problem" scrolls to and
  focuses the first open exception band in the graph (Prompt 6).
- Closed with returns outstanding: neutral, "Closed · 2 items outstanding" (the number is the live outstanding count).
  When the request is closed and the workflow is still running the Engine tab says "Waiting for custody (2)".
- Previous approval: warning, "1 item still in custody from a previous approval" (shown after a forced admin reset).
- Returned: warning, shows the remark: "Returned by Aaron Koh: Wrong SR" with the button "Edit and resubmit" for the
  submitter only (Prompt 7).
- Expired: danger, "Expired: the activity window passed before owner approval" (RQ8).
- Rule stop: danger, "Stopped by a rule: the review score was below the required score" (RQ9).
- Closed unfulfilled: neutral, "Closed – unfulfilled: every item was written off" (RQ4).
- Engine unavailable: handled by the global system banner (Prompt 0); the action buttons become aria-disabled.
- Stale: if an action returns STALE_STATE, show danger alert "Someone else changed this item. Refresh and try again."
  code STALE_STATE with a "Refresh" button; for the requester's stale view of an approval show the plain sentence
  "This request has moved on" and refetch.

THE SHOWCASE REQUEST RQ3 AND ITS STATE SWITCHER (also used by Prompts 5 to 9; keep the data in the shared module)
RQ3 is BG-HRIS-20261108-003: system HRIS, organisation Org Aurora, submitted by Bob Lim, SR number CHG-2041,
justification "Replace failed gateway credentials during the 8 Nov maintenance.", activity window 8 Nov 2026 10:00am
to 5:00pm, workflow definition HRIS v1, one run. The mock clock for RQ3 is 8 Nov 2026, 2:07pm. Six lanes (items):
L1 SP-01 Secure phone (device), L2 SP-02 Secure phone (device), L3 HT-01 Hardware token (token), L4 LT-01 Laptop
(device), L5 HT-02 Hardware token (replacement for HT-01), L6 SP-03 Secure phone (supplementary lane for SP-01).
Approval path already done in every state: Bob submitted; Aaron Koh approved (Alpha Leads L2); Claire Wong approved
(Ops Reviewers L1, risk level high because 4 items is more than 3); review split: Frank Chua scored security 3, Grace
Neo scored logistics 3 (total 6, required 6); Carol Sim approved as owner.
A demo-only segmented control "Demo state" at the top of the page (outside the product chrome, labelled "Demo state:
RQ3") switches between four states:
- State A "Before issuance" (status Approved). L1, L2 and L4 awaiting preparation. L3 HT-01 was reported missing and
  replaced: its lane is excluded, with a "replaced by" link to L5 HT-02 (awaiting preparation). Exception #1 on L3:
  phase "Before issuance", handler "H1 Before issuance", RESOLVED · Replaced. Join: live=4 crossed=0 excluded=1
  pending_replacements=0 -> Wait. No Attention banner.
- State B "Issued, one problem open" (status Credential issued). L1 SP-01 with the requester and a damage report
  open (exception #1, phase "In custody", handler "H3 Damaged in custody", status Investigating, Overdue 2h,
  escalated to Ops Leads L2). L2 SP-02 written off before issuance (excluded; exception #1 H1 Written off). L3 HT-01
  replaced (excluded). L4 LT-01 reported lost after issuance and written off (exception #1, "In custody", H2 Lost in
  custody, with a loss report task done by Bob Lim; resolved Written off; the lane stays counted as issued). L5 HT-02
  with the requester. Join: live=3 crossed=3 excluded=2 pending_replacements=0 -> Issue. Banner "Attention required:
  1 custody exception". Header shows no closed text.
- State C "Closed, two items outstanding" (status Closed). L1 SP-01 returned damaged and quarantined (marker
  "Quarantined (damaged_on_return)"); its exception #1 H3 resolved Replaced with SP-03 (L6, supplementary, with the
  requester). L5 HT-02 with the requester. L2, L3, L4 as in B (resolved). Header banner "Closed · 2 items outstanding".
  Join: live=3 crossed=3 excluded=2 pending_replacements=0 -> Issue (1 supplementary ignored).
- State D "Completed" (status Closed, workflow completed). All lanes finished. Final counts: items 6, issued 4,
  outstanding 0, returned 3, returned damaged 1, replaced 1, replaced after issuance 1, written off 2. Engine tab says
  "Completed" and "history archived". Live indicator reads "Completed".
All other requests (RQ1, RQ2, RQ4 to RQ11) use the same page with simple data: RQ1 three lanes SP-01, HT-01, LT-01 all
with the requester; RQ2 returned with the remark; RQ4 closed unfulfilled; RQ5 SP-01 return verified damaged, problem
open; RQ6 submitted at Beta Leads L2; RQ7 under review at Alpha Leads L2 overdue 2h; RQ8 expired; RQ9 stopped by a rule
(review score 5, required 6); RQ10 at the owner stage with Ivan Ng as both requester and owner; RQ11 is SEN-only.

STATES FOR THE WHOLE PAGE. Loading: skeleton header, skeleton graph block, skeleton rail. Not found or hidden: the 404
page. Read-only (PA, old run, closed request with no actions): the action panel says "You can view this request. You
cannot act on it." Engine unavailable: global banner and disabled buttons. Offline update failure: the live indicator
reads "Update failed. Retrying" and keeps the last data.

ACCESSIBILITY. One h1 (the request number). Each card is a region with an aria-label. The live indicator is polite.
After navigating from the dashboard, focus moves to the h1. The Demo state control is a radiogroup.

TEST IDS. request-header__number, request-header__status, request-header__run-select, request-header__live,
request-banner__attention, request-banner__closed-outstanding, request-banner__previous-approval,
request-page__graph, request-page__actions, request-page__tabs, request-page__issuance, request-page__timeline.
````
