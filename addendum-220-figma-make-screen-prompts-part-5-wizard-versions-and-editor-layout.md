# Addendum 220 (part 5) — Figma Make prompts: create-system wizard, workflow versions, workflow editor layout and handlers

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only. Tag `[CUSTOM: design handoff]`.
**Parts:** part 1 `addendum-220-figma-make-screen-prompts.md` (how to use, Prompts 0, 0B, 1); part 2 `…-part-2-dashboard-new-request-and-request-shell.md` (Prompts 2 to 4); part 3 `…-part-3-request-graph-actions-and-panels.md` (Prompts 5 to 9); part 4 `…-part-4-reports-assets-and-admin.md` (Prompts 10 to 19); **this part** (Prompts 20 to 24); part 6 `…-part-6-editor-tabs-simulation-and-dialogs.md` (Prompts 25 to 30).
**Normative source:** addendum 219 §38.3 (wizard and clone flow), part 6 §28.2 to §28.8 (cloning), part 4 §24.1 (routes and editor layout), §21 (connectors, node kinds incl. the embed node, expressions), §22 (validation, V-44 to V-52), part 11 §41.3 and §41.13 (scope-filtered pickers, binding sentences), §25.2 (HRIS v1 graph and the template seeds), part 6 §27.3 and §27.10, part 10 §38.1 (Handlers tab), §38.4 (copy). 219 wins on any conflict.
**Authored copy.** Strings not found in 219 (for example the import errors and the retire refusal) are authored in SGDS style and marked "authored" in the report that accompanied this addendum.
**Ambiguity resolved here.** 219 sketches the editor title bar as "HRIS v1 · draft r12". A draft has no version number until it is published (part 4 §23.1: `version` is set at publish), so the prompts write "HRIS · draft r12 · from v1".

---

## Prompt 20: create-system wizard, details and owners

**Route:** `/admin/systems/new`, steps 1 and 2 of 4. **Mirrors:** 219 §38.3, US-3.1, US-3.5, part 9 §37 (atomic create). **Needs:** Prompts 12 and 18.

````
Using the app shell, SGDS styling and mock data already in this file, build the "Create system" wizard at /admin/systems/new inside the Admin layout, steps 1 and 2. Step 3 (Workflow) and step 4 (Confirm) are built by the next prompt on the same page.

PURPOSE AND WHO. Create a new break-glass system and its first workflow in one go, so a system is never left without a workflow. Greg Global (any organisation), Amanda Scoped (Org Aurora only) and Mona Multi (Org Aurora or Org Sentinel) can open the wizard. Everyone else sees the 403 page. An organisation admin can create a system only in an organisation their grants name (219 D-38): not SEN for Amanda, and not an organisation added later for Mona.

LAYOUT (desktop)
- Admin layout of Prompt 12. Breadcrumb "Dashboard / Admin / Systems / New". Heading XL "Create system". Body MD "Register a system, its owners and its workflow."
- sgds-stepper (horizontal on desktop, vertical on mobile) with four steps exactly: "1 Details", "2 Owners and PA", "3 Workflow", "4 Confirm". Completed steps show a check and are clickable to go back; later steps are not clickable until reached. The current step has aria-current="step".
- The step panel is an sgds-card, 8 of 12 columns, with a Body SM line "Step 1 of 4" above the step heading. A right rail (4 of 12) shows a live "Summary" card: Organisation, Code, Name, Primary owner, Backup owner, Workflow choice. Unset values show "Not set yet". On tablet and mobile the Summary card moves below the step.
- Footer row of the panel: left "Cancel" (ghost; opens the leave dialog), right "Back" (outline, hidden on step 1) and "Next" (primary). When the person moves to another step, the step heading takes focus and an aria-live="polite" region announces "Step 2 of 4: Owners and PA".

STEP 1: DETAILS (form, validation on blur and on Next)
- "Organisation" (sgds-select, required): Greg picks Org Aurora or Org Sentinel; Mona picks Org Aurora or Org Sentinel (only the organisations her grants name); Amanda sees read-only text "Org Aurora".
- "Short code" (sgds-input, required, uppercased as typed, 2 to 10 characters; hint "Letters and numbers only, for example PAYROLL."). Errors: "Enter a short code." / "Use 2 to 10 letters or numbers." / "That code is already used in this organisation."
- "Name" (sgds-input, required, up to 200). Error "Enter a name."
- "Description" (sgds-textarea, optional, up to 500, character counter).
- "Agency cluster" (sgds-select, required; agency clusters of the organisation: AUR Ops, AUR Logistics). Error "Choose an agency cluster." Hint "The agency that runs the system."
- "Vendor clusters" (sgds-combo-box, multiple, ordered; vendor clusters of the organisation; for AUR: Alpha Pte, Beta Pte). Chips can be reordered with Move left and Move right buttons and the first chip is labelled "1". Hint "The order is the order of vendor review." Optional.

STEP 2: OWNERS AND PA
- "Primary owner" and "Backup owner" (sgds-combo-box each; active agency users of the organisation: Carol Sim, Dan Teo, Ivan Ng, Claire Wong, and so on). Errors: "Choose a primary owner." / "Choose a backup owner." / "The primary and backup owners must be different people." / "Choose an active agency user of this organisation."
- "Approval PA team for the primary owner" and "for the backup owner" (sgds-select each, teams of PA clusters of the organisation; for AUR: Owner Support L1). Error "Choose a team from a PA cluster of this organisation." Helper "People in this team can see the request but cannot act on it."
- Optional "Delegate" (sgds-combo-box) with "From" and "Until" (date-time pickers); the section is a collapsed sgds-accordion "Add a delegate". Errors "The delegate must be an active agency user of this organisation." and "The end must be after the start."

LEAVE DIALOG. "Leave the wizard?" body "Nothing has been saved. Your answers will be lost." buttons "Stay" and "Leave without saving" (danger).

SUBMIT ERRORS (shown as an sgds-alert danger at the top of the step where they arise, with the field values kept): DUPLICATE code at creation, "That code is already used in this organisation."; any 404 shows the 404 page.

MOCK VALUES THE PROTOTYPE SHOULD PRE-FILL WITH A "Fill sample data" demo button: Organisation Org Aurora; code PAYROLL; name Payroll System; agency cluster AUR Ops; vendor cluster Beta Pte; primary Dan Teo; backup Carol Sim; PA teams Owner Support L1.

ACCESSIBILITY. Each step is a real form reachable by Tab; labels are visible; errors link by aria-describedby; the stepper uses aria-current="step".

TEST IDS. clone-wizard__step (data-step), clone-wizard__next, clone-wizard__back, clone-wizard__error, system-form__code, system-form__name, system-form__owner-primary, system-form__owner-backup.
````

---

## Prompt 21: create-system wizard, workflow start, clone and import

**Route:** `/admin/systems/new`, steps 3 and 4. Also the same sub-steps opened from `/admin/systems/[id]/workflow` by "New draft from another system's version". **Mirrors:** 219 §38.3, part 6 §28.2 to §28.8, V-42, V-43. **Needs:** Prompt 20.

````
Using the app shell, SGDS styling and mock data already in this file, build step 3 ("Workflow") and step 4 ("Confirm") of the create-system wizard of the previous prompt, plus the same clone sub-steps as a stand-alone page opened by "New draft from another system's version" from the workflow versions page.

PURPOSE AND WHO. Choose how the new system's first workflow starts: from a blank starter, by copying a published workflow from any system the person can see (with its handlers), or by importing a file. Greg Global sees every organisation as a clone source; Amanda Scoped sees Org Aurora only. Anyone else: the 403 page; a source that does not exist or is not visible is the 404 page.

STEP 3 HAS FOUR SUB-STEPS, shown as a second small progress line "Workflow 1 of 4" ... "Workflow 4 of 4". Buttons per sub-step "Back" and "Next"; the final sub-step's primary is "Create draft".

W1: Workflow 1 of 4, Choose a start (sgds-radio card group, one required)
- "Start blank (starter template)": helper "start, one approval, owner, a per-item step that uses the default lane template (Standard custody lane v1), end. The teams are chosen later." Selecting it sets the sub-steps to 1 only and the next button reads "Next" which goes to step 4 Confirm.
- "Clone from system version": helper "Copy the whole graph, handlers included." (default selected in the mock).
- "Import file": helper "Upload a workflow file (max 1 MiB)." Selecting it shows an sgds-file-upload below the choice (accept .json, one file, up to 1 MiB). Errors (authored): "That file is larger than 1 MiB. Choose a smaller file." and "That file is not a workflow file. Export one from the workflow versions page." and, if teams in the file cannot be found by cluster name and level, an sgds-alert listing the paths: "These teams could not be found: nodes.alpha_l2.assignee (Alpha Leads L2). Create the teams first, or choose another file."
- Footer: "Back" and "Next".

W2: Workflow 2 of 4, Pick the source (only for clone)
- "Organisation" (sgds-select) and "System" (sgds-select). The panel shows a version table (sgds-table with radio in the first column) with columns Version, Status, Published, By, Fingerprint; rows in the mock for AUR then HRIS: "(o) v1 published 2026-11-08 09:30 Greg Global 3fa9c1e2" and "( ) v2 draft (not available)" with the v2 row greyed and the radio disabled, and the text under the table: "Only published or retired versions are listed. An organisation admin sees only this system's organisation." (the draft row is shown only to explain the rule). Fingerprint is the first 8 characters of the SHA-256.
- Error states shown in an inline alert at this step: DRAFT_EXISTS (when the target already has a draft; only for the stand-alone page): "This system already has a draft. Open it, or discard it first." with the link "Open the existing draft". CLONE_SOURCE_NOT_PUBLISHED: "Only published or retired versions can be copied." CLONE_SAME_SYSTEM: "This version belongs to this system. Use "New draft from a version"." with a button "New draft from a version of this system".

W3: Workflow 3 of 4, Map the teams
- Header line: "Source: HRIS v1 (Org Aurora) to PAYROLL" and the count "5 teams to map".
- A real table (sgds-table, caption "Team mapping"). Columns: Source team, Used by, Suggested target, Match, Your choice. One select per row labelled "Target team for {source team}". Rows for the mock (target PAYROLL in Org Aurora with vendor cluster Beta Pte):
  1. Alpha Leads L2 · alpha_l2 · Beta Leads L2 · Position match (amber) · select = Beta Leads L2.
  2. Alpha Engineers L1 · alpha_l2 (escalation) · none · Needs a choice (red) · select = "Choose a team" with the button "Leave unresolved".
  3. Ops Reviewers L1 · ops_l1, verify_return, ... · Ops Reviewers L1 · Same team (green) · select = Ops Reviewers L1.
  4. Ops Leads L2 · sec_review, H1, H2 ... · Ops Leads L2 · Same team (green) · select = Ops Leads L2.
  5. Asset Store L1 · log_review, prepare ... · Asset Store L1 · Same team (green) · select = Asset Store L1.
- Confidence badges: "Same team" green, "Name match" blue, "Position match" amber, "Needs a choice" red; each badge has the word and an icon, and a tooltip explaining the basis (same team in the same organisation; same cluster name and level; same position in the vendor cluster list; no suggestion).
- Unresolved rows are highlighted red with an icon and the text "Unresolved". "Leave unresolved" appears only on rows with no default suggestion (a row with a default is remapped, not left unresolved). After pressing it the select shows "Unresolved" and the button becomes "Choose a team".
- The select lists only live teams of the right type in the target organisation (vendor for vendor references, agency for agency references, never a PA team); for another organisation it starts empty.
- Warning under the table: "1 team will be left unresolved. You can choose it in the editor, but you cannot simulate or publish until you do."
- Error CLONE_MAPPING_INVALID: "Pick a live team of this organisation, of the right kind." The offending rows are outlined red and each row shows its own message.
- Cross-organisation case (Greg picks Org Sentinel as source; the template copy list of W4 appears): an info alert "This workflow belongs to another organisation. Every team needs a choice." and every default is empty.

W4: Workflow 4 of 4, Review (primary "Create draft")
- A summary card: "Copies 16 nodes, 23 connections, 2 embedded workflows, 7 variables." and "Not copied: owners, assets, requests, history." and "Platform nodes (interrupts, expired) are re-created from the current template." and, in Body SM, "Custody stages and handlers live in the embedded templates. Embedded templates are referenced, not copied, unless you tick them below."
- A checkbox list "Also copy these templates into the target library" (shown only when the target system is in another organisation than the source): "Dual review v1", "Standard custody lane v1", "Pin reset v1 (needed by Standard custody lane v1)". Ticking a template ticks and locks the templates it embeds (helper "Needed by Standard custody lane v1."). Each ticked template is copied as a draft of the target library. Untick all: every embedded workflow becomes an unresolved placeholder.
- Warnings (sgds-alert warning each): "PAYROLL has no live asset of type "token" (V-43). You can still publish." and "1 team unresolved (V-42). The draft opens with a red "Choose team (was Alpha Engineers L1 ...)" chip." and, in the cross-organisation case with nothing ticked, "2 embedded workflows unresolved (V-42). The draft opens with the red chips "Choose template (was standard_custody_lane v1, AUR)" and "Choose template (was dual_review v1, AUR)""
- On submit: the button shows a spinner and "Creating draft…", all fields disabled. Success: the system is created together with the draft (all or nothing), a toast "Draft created." and the page goes to /admin/systems/[id]/workflow/edit (Prompt 23), where the lineage chip reads "Cloned from HRIS v1, Org Aurora", the Validation tab lists the unresolved-team error and the token warning, and the Simulate and Publish buttons are disabled with the tooltip "Choose the unresolved teams first".
- For the stand-alone page (from the versions page) the target system is fixed, W1 is skipped and the heading is "New draft from another system's version".

STEP 4: CONFIRM (blank or imported start only; a cloned start ends at W4 with "Create draft")
- Review card with all earlier answers in an sgds-description-list (Organisation, Code, Name, Agency cluster, Vendor clusters, Primary owner, Backup owner, PA teams, Delegate, Workflow start). Primary "Create system". Success toast "System created." and go to the editor on the new draft.

SUBMIT FAILURE AND IDEMPOTENCY. If the server fails after the user pressed Create, show the error at this step and keep every answer; nothing is created (nothing is half-made). Pressing the primary twice creates one system. IDEMPOTENCY_TOKEN_USED: "That action was already submitted. Refresh to see the result."

STATES. Loading versions or preview: skeleton table. Error: sgds-alert danger with the server message and "Try again". Mobile: the mapping table stacks into one card per source team with the select under its label.

ACCESSIBILITY. Every sub-step is a form; the heading takes focus on change and a polite live region announces "Step 3 of 4: Map the teams"; the mapping table is a real table with a select per row; unresolved is text as well as colour.

TEST IDS. clone-wizard__start-choice (data-mode), clone-wizard__source-system, clone-wizard__source-version, clone-wizard__mapping-row (data-ref), clone-wizard__badge, clone-wizard__target, clone-wizard__unresolved, clone-wizard__next, clone-wizard__back, clone-wizard__create, clone-wizard__error, clone-wizard__template-copy (data-template-key).
````

---

## Prompt 22: workflow versions

**Route:** `/admin/systems/[id]/workflow`. **Mirrors:** part 4 §24.1, §24.2, ST-17 (US-17.1), part 6 §28.2, part 10 §38.3. **Needs:** Prompts 12, 18, 21.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/systems/[id]/workflow ("Workflow versions") for the system HRIS, inside the Admin layout.

PURPOSE AND WHO. Shows every version of a system's workflow and lets administrators start, edit, simulate, compare, export, import and retire them. A published version never changes. Greg Global and Amanda Scoped (own organisation) can do everything. Carol Sim and Dan Teo (owners of HRIS) can view the page read only: the action buttons are not rendered and the page shows the banner "You can view this workflow. You cannot change it." Everyone else (for example Bob Lim) and every SEN persona get the 404 page.

LAYOUT (desktop)
- Breadcrumb "Dashboard / Admin / Systems / HRIS / Workflow". Heading XL "HRIS workflow". Body MD "Org Aurora · HR Information System".
- Top action row: "Edit draft" (primary; when a draft exists) or "New draft from v1" (primary; when no draft exists, "vN" is the current published version), "New draft from another system's version" (outline; opens Prompt 21 in its stand-alone form), "Import" (outline; opens the import dialog of the last editor prompt), "Simulate" (outline; opens the editor on the Simulation tab), "Diff" (outline; opens the editor on the Diff tab), "Export" (outline; downloads the current published version).
  Disabled reasons (shown as text under the row, never only a tooltip): "New draft from v1" disabled while a draft exists with "This system already has a draft. Open it, or discard it first."; "Simulate" and "Diff" disabled without a draft with "Create a draft first."
- A "Current version" card: "v1 published" (green badge), "Published 8 Nov 2026, 9:30am by Greg Global", "Fingerprint 3fa9c1e2", change notes in a quote block ("Initial HRIS workflow"), and a Body SM line "Requests that started on v1 keep running v1." Under it a line "Uses templates:" followed by one chip per pinned template, "Dual review v1" with the badge "v2 available" (accent, with an info icon; tooltip "Open the draft to upgrade.") and "Standard custody lane v1"; each chip links to that template's versions page (/admin/templates/[id]/workflow). Without embedded templates the line reads "Uses templates: none".
- A "Draft" card (when a draft exists): "Draft r12", "Based on v1", "Last autosaved 14:02:11 by Amanda Scoped", a Body SM line "The draft has 2 errors and 1 warning." (from the last validation), a line "Uses templates: Dual review v1 (v2 available), Standard custody lane v1" (the draft pins versions explicitly; V-52 warns when a newer one exists), and buttons "Edit draft" and "Discard draft" (danger outline).
- A "Version history" table (sgds-table, caption "Workflow versions"): columns Version, Status (Draft, Published, Retired with word and colour), Published (date and time, SGT), By, Fingerprint (8 characters, monospace-free), Change notes (truncated, full text on hover), Requests (count of runs pinned), Actions (overflow: Open (read only), Diff against current, Export, Retire).
  Mock rows for HRIS: Draft r12 (no version number; "from v1") · Amanda Scoped · "-"; v1 · Published (Current) · 8 Nov 2026, 9:30am · Greg Global · 3fa9c1e2 · "Initial HRIS workflow" · 5 requests. For a second system show how history grows: OPSTOOL with v2 Published (Current) 9 Nov 2026, 10:00am, Amanda Scoped, b71c04aa, "Add a second review" and v1 Retired, Greg Global, 51d9e07c.
- Mobile: the table becomes stacked cards; the action row wraps and "More" opens the secondary actions in an overflow menu.

ACTIONS AND DIALOGS
- "New draft from v1": creates the draft and goes to the editor. Toast "Draft created." 409 DRAFT_EXISTS inline alert "This system already has a draft. Open it, or discard it first." with the link "Open the existing draft".
- "Discard draft": generic confirmation "Discard the draft?" body "The draft and its changes will be removed. Published versions are not affected." primary "Discard draft" (danger). Toast "Draft discarded." Only drafts can be discarded.
- "Retire" on a version (confirmation "Retire v1?" body "New requests will not use a retired version. Requests already running keep it." primary "Retire version"). Refused when it is the current version and no newer one exists: alert in the dialog "The only published version cannot be retired. Publish a newer version first." (authored). A retired version's overflow shows no Retire. A system whose only version is retired shows a danger banner "This system has no published workflow. Requests cannot be submitted for it." (the system is hidden from the new request form).
- "Export": downloads a file named "hris-v1.json" and shows the toast "Workflow exported." The file shows its fingerprint in the toast Body SM line "Fingerprint 3fa9c1e2".
- "Open (read only)" opens the editor of that version in view mode.

STATES. Loading: skeleton cards and table. Error: sgds-alert danger with "Try again". No published version yet: the Current card reads "No published version yet. Publish the draft to let requests use this system." Engine banner does not apply (admin CRUD stays enabled).

ACCESSIBILITY. The status is a word and a colour; the table has a caption; disabled buttons use aria-disabled with the reason in aria-describedby.

TEST IDS. workflow-versions__new-draft, workflow-versions__edit-draft, workflow-versions__discard, workflow-versions__row (data-version), workflow-versions__export, workflow-versions__retire, workflow-versions__clone-other, workflow-versions__templates (data-template-key, data-upgrade-available).
````

---

## Prompt 23: editor, layout, canvas, palette, inspector

**Route:** `/admin/systems/[id]/workflow/edit`. **Mirrors:** part 4 §21.1 to §21.4, §22, §24.1, §25.2, ST-15 to ST-16, ST-18 US-18.3, part 10 §38.4, addendum 219 part 12 §42.14. **Needs:** Prompts 12, 22.

````
Using the app shell, SGDS styling and mock data already in this file, build the workflow editor at /admin/systems/[id]/workflow/edit for the HRIS draft, using a node-and-edge canvas (React Flow style, custom node types). Build the editor shell, the left palette, the centre canvas with the HRIS v1 graph and the right inspector. The Handlers tab and the coverage grid (which belong to the template editor) are built by the next prompt; the bottom tabs (Validation, Variables, Simulation, Diff) by the ones after it.

PURPOSE AND WHO. An administrator draws a system's workflow as a graph of steps. Greg Global and Amanda Scoped can edit and publish. Carol Sim (owner of HRIS) sees the same page read only: the palette is hidden, the inspector is disabled, the Publish button and every edit control are absent, and a banner shows "You can view this workflow. You cannot change it." Others get the 404 page.

PAGE FRAME (full viewport height; the editor uses the whole width, no sgds-container; the masthead and main nav stay)
- Top bar (sticky): left, breadcrumb-style "HRIS" link and the text "Workflow draft r12 · from v1" with a save indicator ("Saved 14:02:11", "Saving…", "Could not save. Retrying…"); a lineage chip when cloned: "Cloned from HRIS v1, Org Aurora". Right: "Undo" and "Redo" icon buttons (aria-labels "Undo", "Redo"; 50 steps), "Validate" (outline), "Simulate" (outline), "Publish" (primary).
  - Publish is disabled while any error issue exists; the reason shows as Body SM text next to it: "Fix the errors in the Validation tab first." When team placeholders remain: "Choose the unresolved teams first".
  - Simulate is disabled when errors exist, with the same reasons.
- Three columns under the bar: left palette (width 240), centre canvas (flex), right inspector (width 360). Below the canvas, a bottom tab strip (Validation, Variables, Simulation, Diff) 280 px tall and collapsible (built later).
- Tablet: the palette collapses to a floating "Add node" button with a popover; the inspector opens as a right drawer on node selection. Mobile: the editor shows a notice "Use a larger screen to edit workflows" with the "List view" of nodes (read only) so the prototype still renders.

LEFT PALETTE (sgds-card, scrollable, each kind a button with an icon and a one-line description)
- Top-level kinds, in this order: "Start" (flag), "Approval" (check shield), "Owner approval" (crown), "Amend" (pencil), "Rule" (diamond), "Switch" (branch), "Set variable" (equals), "Expression · f(x)" (function icon; "Work out values with a formula"), "Function · JS" (code icon; "Run a short JavaScript function"), "Split" (fork), "Join" (merge), "Wait" (clock), "Notify" (bell), "Task" (checklist), "Embedded workflow" (stack icon; opens the template picker of the last editor prompt), "Terminal" (square).
- "Start", "Amend" and "Owner approval" are aria-disabled once present, with the reason "Already in the workflow" under the item. Neither Expression nor Function appears in the template editor's handler palette. "Embedded workflow" can be placed several times; choosing it opens the "Choose a template" picker, and the new node asks "Runs" (Once or Per item). A system workflow holds exactly one Per item embed that uses a template with custody stages (error V-51).
- There is no group of custody kinds in this editor. "Custody stage", "Entry" and "Exit" exist only in the template editor (the last editor prompt); custody stages and exception handlers live inside the lane template "Standard custody lane", which is edited there.
- Drag a kind onto the canvas, or press Enter on it to place it at the centre. Keyboard: each item is a button.

CENTRE CANVAS
- Pan by dragging the empty canvas; zoom 25% to 200% with the wheel and the controls bottom-left (zoom in, zoom out, "Fit", "Auto-layout", "Mini-map" switch, lock). The mini-map sits bottom-right. Auto-layout arranges the graph left to right and keeps pinned positions.
- Nodes: every node shows its label, a kind icon, an assignee chip, and one labelled handle per port (the port key is text next to the handle; a Rule shows its author's labels, for example "Yes" and "No", with the key in small print). An Expression or Function node shows the subtitle "Writes risk_score, needs_dual" or "In 3 · Out 1" and the ports next and error (error in the danger tone). Port colours: approve and true green, return amber, reject and false red, others grey; the text always names the port, so colour is never the only cue. A node with an error has a red outline and an error icon (tooltip lists the issue codes); a warning uses amber and a warning icon. A node with no inbound edge shows a red outlined in-handle.
- The node ARIA label has the form "Approval: Ops Reviewers L1, 3 ports, 1 unconnected".
- Edges are labelled with the port key; selecting an edge opens the inspector; Delete removes the selected node (with its edges) or the selected edge.
- Connection rules while dragging: an invalid drop draws nothing and shows a tooltip with the reason. Refusals and their tooltips (the first one is exact; the others are authored):
  - a second edge from a port: "This port already has a connection. Delete it first."
  - an edge into Start: "The start node cannot have incoming connections."
  - an edge out of a terminal: "A terminal ends the workflow."
  - reject to anything but a rejected terminal: "Reject can only lead to a rejected terminal."
  - return to anything but Amend: "Return can only lead to the amend step."
  - an edge into or out of the inside of an embedded workflow: "Connections cannot cross into an embedded workflow."
- The server re-checks everything; the UI is a convenience.
- The locked platform nodes (the "Expired" terminal and the three interrupt nodes) show a padlock and cannot be moved, edited or deleted ("This platform step is fixed and cannot be changed.").
- Read-only (owner view): nodes are selectable and show the inspector in read-only form; no handles are draggable.

THE HRIS V1 GRAPH TO DRAW (16 top-level nodes, plus the locked Expired terminal; draw left to right in this order; node keys in brackets)
1. Start "Submit" [submit] port submitted.
2. Approval "Alpha Leads L2" [alpha_l2] team Alpha Leads L2, SLA 4 h, escalates to Alpha Engineers L1; ports approve, return, reject.
3. Approval "Ops Reviewers L1" [ops_l1] team Ops Reviewers L1; required field "Risk level" on approve.
4. Set variable "Assess risk" [assess]: high_risk := risk_level == high OR item_count > 3 OR window_hours > 8.
5. Rule "High risk?" [is_high_risk]: high_risk == true; ports true, false.
6. Embedded workflow "Dual review" [review] of template Dual review v1, Runs "Once". Input required_score (bound to the request variable required_score); output review_score (written to the request variable review_score). Ports passed and failed. Reached from the rule's true port. passed goes to Owner, failed goes to the Rejected terminal. A reject or return inside it acts on the whole request, so its propagated reject and return edges also go to the Rejected terminal and to Amend.
7. Owner approval "Owner" [owner] (primary, backup or delegate); is_high_risk false also goes here.
8. Notify "Notify PA" [notify_pa] (the PA of the primary owner), then Wait "Until 30 min before start" [wait_window] (until activity_start_at minus 30 min).
9. Embedded workflow "Items" [items] of template Standard custody lane v1, Runs "Per item" (1..5 items), input none, output items_received bound to the request variable received_counter (combine: sum), ports issued, completed, unfulfilled.
10. Task "Close request" [close_req] (kind close; submitter or any reviewer) on the issued port, then Join "Done" [join_done] policy all, also fed by completed, then Terminal "Completed" [completed]. The unfulfilled port goes to Terminal "Closed – unfulfilled" [closed_unfulfilled].
11. Terminal "Rejected" [rejected] receives every reject edge and the failed port of Dual review. Amend "Amend" [amend] receives every return edge and sends resubmitted back to alpha_l2 as a back-edge. Locked terminal "Expired" [expired].
EMBED NODE VISUALS. An embedded workflow is one node, not a group. It shows: the template name and version ("Dual review · v1"), a cardinality badge "Once" or "Per item", input chips on the left (for example "required_score") and output chips on the right (for example "review_score"), one labelled handle per derived port, a stack icon, and a button "Open". "Open" shows the template read only in place, with breadcrumbs above the canvas "HRIS v1 › Standard custody lane v1 › PIN reset v1" (each crumb is a link; the last crumb is the current view; a "Back to HRIS v1" button returns). Inside the drill-in nothing is editable and the banner reads "You are viewing an embedded workflow. Edit it in the template editor." with a link for those who may edit templates. When a newer template version exists the node shows the badge "v2 available" (accent). An embed whose template is missing (after a cross-organisation clone) shows the red chip "Choose template (was standard_custody_lane v1, AUR)" and no ports.
The read-only drill-in of Standard custody lane v1 shows its 11 nodes: Entry [entry] to Rule "Is token?" [is_token] (item.credential_type == token); true to Embedded workflow "PIN reset" [pin_reset] (Pin reset v1, Once; output pin_reset_done, port done) then Prepare; false to Prepare; custody stages Prepare [prepare] (Asset Store, 4 h), Initiate handover [initiate_handover] (Asset Store, 2 h), Confirm receipt [confirm_receipt] (submitter, issuance milestone flag), Set variable "Count received" [count_received] (items_received := items_received + 1, written to the referenced request variable received_counter), Initiate return [initiate_return] (submitter), Verify return [verify_return] (Ops Reviewers, 24 h), Return to inventory [return_to_inventory] (Asset Store, 8 h), Exit "returned" [exit_returned]. Pin reset v1 is one Task "Reset PIN" (Asset Store L1) between Entry and Exit "done".
FINDB and OPSTOOL drafts are linear chains whose per-item step embeds Basic custody lane v1; PORTAL embeds Sen custody lane v1. Draw them when the system is switched in the demo panel.

RIGHT INSPECTOR (sgds-card; empty state "Select a node or a connection to see its settings.")
- Common fields for every node: "Label" (1 to 80 characters; error "Enter a label up to 80 characters."), "Key" (read-only after creation; must match lowercase letters, digits and underscores, starting with a letter, up to 40 characters; error "Use lowercase letters, digits and underscores, starting with a letter."), and a "Connect port to…" section: for each port a select of valid targets (only targets that pass the connection rules), the pointer-free alternative to dragging. The current selection also shows "Delete node" (danger ghost; disabled for locked nodes with "This platform step is fixed and cannot be changed.").
- Approval: "Assignee team" (picker listing live teams of the system's organisation of a valid type; vendor or agency; never PA; an unresolved team shows the red chip "Choose team (was Alpha Leads L2)"), "Required fields on approve" (a list of fields: label, variable, type, min, max, required tick), "SLA (hours)" 1 to 720 (error "Enter 1 to 720 hours."), "Escalate to team" (picker), "Allow override" (switch; helper "A reviewer from a later team may approve for this team.").
- Owner approval: assignee shown fixed "Primary or backup owner, or an active delegate"; fields list.
- Amend: no settings; helper "The requester revises and resubmits. Re-entry goes to an approval step."
- Rule: the visual rule builder (below), and "Port labels" (test id rule-inspector__labels): a select "True / False" (default), "Yes / No", "Pass / Fail", "Met / Not met", "Custom…". "Custom…" shows two text fields "Label for true" and "Label for false" (1 to 40 characters; errors "Enter a label up to 40 characters." and "Use different labels for the two ports." (V-68)). Helper "Labels are shown on the canvas and the request page. The workflow still uses true and false." The node handles show the labels instead of the keys. Switch: an ordered list of cases, each with a label and a rule builder, plus "Default".
- Set variable: "Target variable" (writable variables only), "Expression" (builder: variable or literal, operators add, subtract, multiply, no division). Errors: "That variable cannot be changed here." and a type mismatch message. Helper under the builder: "Need division or rounding? Use an Expression · f(x) step."
- Expression · f(x) and Function · JS: the inspectors are built by Prompt 31 (addendum 220 part 6). Here, show only the common fields and the two ports "next" and "error" in "Connect port to…".
- Split: "Branches" (2 to 5; add and remove; each has a label). Join: "Policy" (All, Any, K of N) and "K" when K of N (1 to N; error "K must be between 1 and the number of branches.").
- Wait: "Wait for" radio "A number of hours" (1 to 720) or "Until a time" (timestamp variable plus "Offset" in minutes, negative allowed).
- Notify: "Notify" radio "PA of the primary owner", "The requester", "A team" (picker). Helper "A record is kept. No message is sent in this version."
- Task: "Assigned to" radio "The requester", "Any reviewer on the request", "A team"; "Kind" select (Close request, Other); fields list.
- Embedded workflow, section "Embedded workflow" (test id embed-inspector__section):
  - "Template" (select of eligible published templates of the organisation; test id embed-inspector__template) and "Version" (read-only "v1") with the button "Upgrade to v2" (test id embed-inspector__upgrade) when a newer one exists; it opens the upgrade confirmation of the last editor prompt.
  - "Runs" (radio "Once" or "Per item"; helper "Per item starts one copy of the template for each item.").
  - Per item settings, shown only for Per item (these are the call-site settings; the template does not carry them): "Minimum items" and "Maximum items" (1 to 10, min not above max; error "Minimum must not be more than maximum."), "Allowed credential types" (checkboxes Device, Token, Password), "Issuance policy" (All live, All, K of N), "Allow replacement" and "Allow write-off" (switches), "Max replacements per item" (select 0 to 3; default 1; helper "How many times an item may be replaced."), "Allow replacement after issuance" (checkbox; helper "Lets a replacement be added after the requester has received the item."). The handlers are part of the template and are edited in the template editor.
  - "Inputs" table (sgds-table, caption "Embedded workflow inputs"): columns Parameter, Type, Required, Bound to. Bound to is a select "Variable" or "Value" toggle: for Dual review, row "required_score · Number · Yes · Value 6". Under each row a Body SM sentence: "Copied in when the embedded workflow starts. Read only inside it."
  - "Outputs" table (caption "Embedded workflow outputs"): columns Parameter, Type, Write to variable, Combine. For Dual review, row "review_score · Number · review_score · none", with the sentence "Copied out when it ends normally (passed or failed). Not copied if it is rejected or returned." For the Per item embed: "pin_reset_done · Yes or no · pins_reset · Count true", with the sentence "Combined once over all finished items, when Completed fires." Combine (shown only for Per item) is one of Sum, Count true, Any, All, Min, Max, filtered by the type.
  - "Shared variables" table (caption "Shared by reference"; shown only when the template has By reference parameters): columns Parameter, Type, Shared with. Row "received_counter · Number · items_received", with the sentence "Shared with this workflow. Items can only add to it; additions are applied one at a time." A missing or mistyped binding shows the inline error "Bind this parameter to a variable of type {type}." (code V-48).
  - "Ports preview" (read only): Dual review "passed, failed"; Standard custody lane per item "issued, completed, unfulfilled". Helper "Ports come from the template. Change the template to change them."
- Terminal: "Kind" (Completed, Rejected, Closed – unfulfilled); the locked Expired terminal is read-only.
- Custody stage (template editor only): "Team" (agency teams, or "The requester" for Confirm receipt and Initiate return), "Due in hours" (1 to 720, optional); the stage name is fixed; Confirm receipt shows the tag "Issuance milestone".
- VISUAL RULE BUILDER (no free text; the Expression · f(x) step of Prompt 31 is the only place formulas are typed). Conditions are rows: "Variable" select, "Operator" select (>, <, >=, <=, ==, !=), then "Variable" or "Value" toggle and its input (typed literal: number, true or false, enum value, date-time). "Add condition" and group toggles "AND" / "OR"; "Add group"; "NOT" per group. Aggregates for rules in a system workflow (a root): "Item count", "Any item where…", "All items where…", "Count of items where…". Built-ins "Now" and "Hours between". A live sentence under the builder renders the expression, for example "high_risk == true". Type errors appear inline: "Compare values of the same type." and "A yes or no value cannot be ordered."
  Item attributes (item.credential_type, item.asset_tag) can be used only in a template whose subject is Per item. The variable list shows only what the selected node can use: its own workflow's variables and the platform values for where it sits (exception values only inside a handler). The rendered text of the rule shows on edge hover in the request page.
- Edge selection: shows the from and to node, the port, and "Delete connection".

ISSUE MARKERS. Each issue from the Validation tab (built later) shows on its node or edge. Clicking an issue selects the node and centres the canvas on it. Examples of the messages the server returns: "This node is never entered" (a node with no inbound edge), "ops_l1.reject is not connected" (an unconnected port), "Choose a team (was Alpha Engineers L1)".

SAVING. Autosave 1 second after the last change, sending the whole draft with its revision. STALE_DRAFT: a danger alert bar across the top "Someone else changed this draft. Reload to continue." with a "Reload" button; local changes stay in the undo history. DEFINITION_IMMUTABLE (a published version opened by mistake): "This version is published and cannot be changed."

ACCESSIBILITY. Every action has a keyboard path; palette items are buttons; nodes are focusable with ARIA labels as above; the inspector's "Connect port to…" select replaces dragging; the canvas has a "List view" toggle that shows the graph as a nested outline; focus rings are visible.

TEST IDS. editor__canvas, editor__palette-item (data-kind), editor__inspector, editor__publish, editor__undo, editor__redo, editor__save-status, editor__node (data-node-key, data-kind), editor__edge (data-from-port), editor__connect-select (data-port), embed-node__open, embed-node__badge (data-cardinality), embed-node__upgrade-badge, embed-inspector__section, embed-inspector__template, embed-inspector__upgrade, embed-inspector__runs, embed-inspector__inputs, embed-inspector__outputs, embed-inspector__ports, editor__breadcrumb (data-depth), rule-inspector__labels, rule-inspector__label-true, rule-inspector__label-false.
````

---

## Prompt 24: editor, handlers, catch filter, coverage grid, effect cards (template editor for the custody lane)

**Route:** `/admin/templates/[id]/workflow/edit` for the template Standard custody lane (the Handlers tab and the coverage grid belong to the template editor of Prompt 30; the root editor `/admin/systems/[id]/workflow/edit` keeps only the Interrupts tab, reached from the bottom tab strip). **Mirrors:** part 10 §38.1, §38.4 handler tests ids, part 6 §27.2 to §27.7, §27.11, V-32 to V-41. **Needs:** Prompt 23.

````
Using the app shell, SGDS styling and mock data already in this file, extend the workflow editor, opened in template mode on the template "Standard custody lane" v1 draft (route /admin/templates/[id]/workflow/edit, the same editor shell; the last editor prompt describes template mode), with the tab area "Lane", "Handlers" and "Interrupts". The Lane graph and the Handlers tab belong to the lane template, so they appear here and not in the system workflow editor; the Interrupts tab stays in the system workflow editor (it is read only and also shown here as a reference). Build the Handlers tab completely except for the handler mini-canvas, which the next prompt adds (draw a placeholder card for it with the title "Mini-canvas · H3 Damaged in custody").

PURPOSE. A "problem" (an exception) can be reported on an item at any stage. Handlers are the drawn rules that say what happens to each kind of problem. Administrators order the handlers, say which problems each one catches, draw what it does, and see at a glance that every kind of problem is caught by something.

WHERE IT APPEARS. In the template editor of Standard custody lane (subject Per item), the Handlers tab sits among the bottom tabs (Validation, Variables, Interface, Eligibility, Handlers, Simulation, Diff). The tab area has the title "Handlers of Standard custody lane v1" and the tabs [ Lane ] [ Handlers ] [ Interrupts ]. The replacement settings "Max replacements per item" and "Allow replacement after issuance" are NOT here: they are call-site settings of the per-item embed in the system workflow and are edited in that embed's inspector (previous prompt). A Body SM note under the title says "Replacement limits are set where this template is used." Only administrators can edit; others see a read-only view.

TAB LANE. The lane template graph is the canvas of the template editor (Entry, Is token?, PIN reset embed, the custody stages, Count received, Exit "returned"); this tab shows a text list of the 11 lane nodes with their keys, the milestone tag, and the note "Problems and disputes during custody are handled by the handlers, not drawn in the lane."

TAB HANDLERS (layout in two columns over a catch-filter band and the coverage grid)
Left column, "Handlers (drag the grip to reorder)": a vertical list of handler rows. Each row has a grip handle, the order number, the key chip, the label, a chevron, and a diff badge slot (empty here). The selected row is highlighted ("selected"). The rows for HRIS (in this order, which is the evaluation order):
  1 · H1 · Before issuance
  2 · H3 · Damaged in custody (selected)
  3 · H2 · Lost in custody
  4 · H4 · Problems at return
  5 · fallback (pinned last, with a padlock and the word "fallback"; its row cannot be dragged or deleted)
Below the list: "Add handler" (outline; test id handler-list__add; opens the Add handler dialog of the last prompt), "Duplicate" (not offered for the fallback), "Delete" (danger outline; disabled on the fallback with the reason "The fallback cannot be deleted.").
Under the list a status line: "HANDLER_SHADOWED: none" when there is none, or a warning "Earlier handlers already catch everything this one catches." beside the shadowed handler row.
Reordering: drag by the grip; keyboard Up and Down arrows move focus; Alt+Up and Alt+Down reorder; the buttons "Move up" and "Move down" are the pointer-free alternative. A drop below the fallback is refused with the tooltip "The fallback stays last." Reordering re-evaluates the grid immediately. Delete on a selected non-fallback handler asks for confirmation (dialog in the last prompt).
Right column: the mini-canvas placeholder card (next prompt) with a toolbar line "[-] 100% [+] Fit Auto-layout Mini-map Expand".

CATCH FILTER BAND ("Catch filter · H3")
- Row 1 "Exception types": chips [damaged x] [identity_mismatch x] and "+ add" (sgds-combo-box popover of types: missing, unavailable, damaged, wrong_item, identity_mismatch, quantity_mismatch, handover_disputed, receipt_disputed, return_disputed, other). Helper "Empty means all types."
- Row 2 "Phases": chip [in_custody x] and "+ add" (pre_issuance, in_custody, return). "Severities": "any" (select: any, low, medium, high, critical, with multi-select chips).
- Row 3 "Reads as": a sentence in Body SM, exactly in this style: "type damaged ∈ {damaged, identity_mismatch}; phase in_custody ∈ {in_custody}; severity any". It updates with every chip change. For the fallback the band shows "Everything else" and the filters are not editable.
- A new handler must keep at least one chip; removing the last chip shows the inline error "Keep at least one filter so this handler does not catch everything."

COVERAGE GRID (a table with role="grid", roving tabindex; heading "Coverage grid" and a selector "severity [any v]")
Legend in one line: "H1 caught · H3* shadowed · fb fallback-only · /// impossible · ~ varies". Columns: pre_issuance, in_custody, return. Rows and exact cell content for the HRIS mock:
  missing: H1, H2, H4
  unavailable: H1, H2, H4
  damaged: H1, H3*, H4
  wrong_item: H1, H2, H4
  identity_mismatch: H1, H3*, H4
  quantity_mismatch: H1, H2, H4
  handover_disputed: H1, ///, ///
  receipt_disputed: H1, ///, ///
  return_disputed: ///, ///, H4
  other: H1, H2, H4
Cell states (each has a text mark, a colour, a tooltip, and is never colour only):
  caught: the key such as "H1" (green): exactly one non-fallback handler matches.
  shadowed: "H3*" (amber): the winner is a non-fallback handler and a later one also matches; tooltip "H2 also matches; H3 wins on order". Informational.
  fallback-only: "fb" (blue-grey): only the fallback matches; allowed, shown so nothing is silently uncovered.
  impossible: "///" (hatched grey): the combination cannot happen; not focusable by click; never raises the coverage error.
  varies: "~" (purple outline): the winner depends on severity (when the selector is "any").
Pressing Enter or clicking a cell focuses the winner's mini-canvas. Changing the severity selector re-evaluates. Uncovered cells (no handler and no fallback) are red with the text "Not caught" and block publishing with the error V-32 shown in the Validation tab: "Some combinations of problem type and phase are not caught. Add a handler or fix the fallback."
Reorder and edits re-evaluate the grid on the client; the server's validation result is the authority.

HANDLER INSPECTOR FIELDS (appear in the right inspector when a handler node is selected)
- Investigate: "Assignee team" (live agency teams only; vendor and PA teams do not appear), "SLA (hours)", "Escalate to team", six port tick boxes (recovered, false report, replacement, write-off, waiver, quarantine), "Allow info hold" (switch; helper "Lets the investigator ask for information and pause the problem.").
- Resolution approval: "Approver" radio "Owner class" or "Team" with a team picker, optional "SLA (hours)".
- Resolution: "Kind" (resume, false report, replace, write off, waive, quarantine on return) and, for resume, "Mode" radio "State before the problem" or "State chosen by the investigator".
- An unresolved team shows the red chip "Choose team (was Alpha Engineers L1)" in place of the picker value.

EFFECT CARDS. Selecting a resolution node shows one card per phase the handler can catch. The server supplies the text; render these exact sentences:
  Resume (any phase): "Resume (any phase): the case resumes at the state you choose; the asset is unchanged; the stage SLA restarts."
  False report (any): "False report: the case resumes where it was; nothing else changes."
  Waive (any): "Waive: the case resumes where it was; the deviation is recorded; the expected return condition is unchanged."
  Replace before issuance: "Replace (before issuance): original lane ends and is excluded; asset retired 'lost_replaced' if lost or damaged, otherwise released; replacement lane starts on the substitute."
  Replace after issuance, lost: "Replace (after issuance, lost): original lane ends; asset retired 'lost_replaced'; issuance stands; supplementary lane starts on substitute."
  Replace after issuance, damaged: "Replace (after issuance, damaged): original resumes in custody expecting a damaged return; substitute gets a supplementary lane; issuance stands."
  Replace after issuance, other: "Replace (after issuance, other): original resumes in custody, return condition stays good; substitute gets a supplementary lane; issuance stands."
  Replace at return: "Not permitted at return (V-35)."
  Write off before issuance: "Write off (before issuance): item excluded; asset retired 'written_off'; lane ends; if no lane is left the request closes unfulfilled."
  Write off after issuance or return: "Write off (after issuance): asset retired 'written_off'; hold released; issuance stands; the item stays counted."
  Quarantine on return, after issuance (damaged or other): "Quarantine on return: the item stays with the requester; the return is expected damaged; the asset will be quarantined, not restocked."
  Quarantine on return, at return: "Quarantine on return: the asset is quarantined for 'damaged_on_return', its hold released, and it is not restocked."
  Quarantine before issuance: "Not permitted before issuance (V-35)."
An effect card is a neutral sgds-card with a heading (the resolution and the phase) and the sentence; a "Not permitted" card is shown in the danger tone with an error icon.

TAB INTERRUPTS. Three padlocked read-only nodes in a row with no handles: "expire", "tombstone_cancel", "admin_reset". Selecting one shows a sentence (read only): expire "Cancels the request when the activity window passes before owner approval."; tombstone_cancel "Cancels the request when the requester is deactivated."; admin_reset "Resets the request to the requester." Deleting or moving them is refused with "This platform step is fixed and cannot be changed."

THE HRIS v1 HANDLER SET (draw the lists, filters and the grid from this; the mini-canvas is next)
  H1 Before issuance: catches pre_issuance, all types and severities; investigate Ops Leads L2, 48 h; recovered resumes at the state before the problem; false report; replacement, write-off and waiver each go through the owner approval; allow info hold on.
  H3 Damaged in custody: catches in_custody with types damaged and identity_mismatch; investigate Asset Store L1, 8 h, escalates to Ops Leads L2; false report; replacement via approval by the Ops Leads L2 team; waiver via owner approval; write-off via owner approval; quarantine goes straight to Quarantine on return.
  H2 Lost in custody: catches in_custody, everything else; investigate Ops Leads L2, 24 h; recovered resumes at a state the investigator chooses; false report; write-off goes through a "Loss report" task for the requester then the owner approval; replacement via the owner.
  H4 Problems at return: catches return, all; investigate Ops Reviewers L1, 24 h; recovered (investigator chooses); false report; quarantine goes straight to Quarantine on return; write-off via the owner approval.
  fallback: everything, Ops Leads L2, 48 h; recovered, false report, write-off via owner, waiver via owner; no replace (it also catches the return phase).

STATES. Loading: skeleton list and grid. Error: sgds-alert danger. Read-only (owner view): no grip, no add, duplicate or delete, no editable chips; the grid and filters stay visible. Mobile: the panel stacks list, filter band, grid and a card for the mini-canvas.

ACCESSIBILITY. The handler list is a keyboard list: Tab enters it, Up and Down move, Alt+Up and Alt+Down reorder (or the Move buttons), Enter opens the mini-canvas, Delete on a selected non-fallback handler asks for confirmation. The grid is role="grid" with roving tabindex; Enter on a cell focuses its handler. Phase and resolution are text, not colour alone.

TEST IDS. handler-list__item (data-handler-key), handler-list__grip, handler-list__move-up, handler-list__move-down, handler-list__add, handler-list__delete, catch-filter__types, catch-filter__phases, catch-filter__severities, catch-filter__reads-as, coverage-grid__cell (data-type, data-phase, data-state), coverage-grid__severity, effect-card__text (data-kind, data-phase).
````
