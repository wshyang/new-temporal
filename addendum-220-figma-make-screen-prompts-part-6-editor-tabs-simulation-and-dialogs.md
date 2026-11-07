# Addendum 220 (part 6) — Figma Make prompts: handler mini-canvas, editor bottom tabs, simulation, diff, publish and editor dialogs

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only. Tag `[CUSTOM: design handoff]`.
**Parts:** part 1 `addendum-220-figma-make-screen-prompts.md` (how to use, Prompts 0, 0B, 1); part 2 `…-part-2-dashboard-new-request-and-request-shell.md` (Prompts 2 to 4); part 3 `…-part-3-request-graph-actions-and-panels.md` (Prompts 5 to 9); part 4 `…-part-4-reports-assets-and-admin.md` (Prompts 10 to 19); part 5 `…-part-5-wizard-versions-and-editor-layout.md` (Prompts 20 to 24); **this part** (Prompts 25 to 30).
**Normative source:** addendum 219 part 10 §38.5 (handler mini-canvas), §38.1 (simulation injector), §38.4 (copy and test ids), part 4 §22 (validation catalogue), §24.1 (bottom tabs, saving, publish), §24.3 (simulation), §25.1 (variables), ST-16, ST-17, ST-22, part 6 §27.6, §27.7 (V-32 to V-43), part 4 §20 to §26 (embeddable templates, V-44 to V-52), ST-33, part 11 §41 (variable scope and data flow, V-53 to V-56), ST-34. 219 wins on any conflict.
**Authored copy.** The dialogs for add handler, delete handler, import, publish and retire are specified in 219 by fields and rules, not by words; their titles, helper texts and button labels are authored in SGDS style.

---

## Prompt 25: editor, handler mini-canvas

**Route:** `/admin/templates/[id]/workflow/edit` (template editor of Standard custody lane), Handlers tab, right pane. **Mirrors:** part 10 §38.5.1 to §38.5.4. **Needs:** Prompt 24.

````
Using the app shell, SGDS styling and mock data already in this file, replace the placeholder card "Mini-canvas · H3 Damaged in custody" of the Handlers tab with the real handler mini-canvas, using a node-and-edge canvas (React Flow style) with its own separate instance per handler.

PURPOSE AND WHO. Shows and edits the graph of the selected handler. Greg Global and Amanda Scoped edit; owners see it read only (no palette, no handles, no inspector edits). The handlers belong to the lane template Standard custody lane v1 and are edited in the template editor, not in the system workflow editor. It is not a sub-flow of the lane: no edge can be drawn to a node outside the handler.

LAYOUT. The pane is at least 640 by 360 px and resizes with the window. Header: "Mini-canvas · H3 Damaged in custody" and a toolbar: zoom out, "100%", zoom in (25% to 200%), "Fit", "Auto-layout", "Mini-map" (switch), "Expand" (opens the same canvas in a full-width overlay with a focus trap; "Esc" or the button "Back to handlers" closes it and returns focus to the handler's row; the palette and inspector come with it). Footer legend: "out-port ○ · in-connector ● · resolution ■ · reject goes back to Investigate · fixed (padlock)".
A handler-only palette sits above the canvas as a row of buttons: "Investigate", "Approval", "Resolution", "Rule", "Switch", "Set variable", "Task", "Wait", "Notify". The handler start node, custody stages and every top-level kind are absent. Dropping "Resolution" first asks for its kind (a small popover: resume, false report, replace, write off, waive, quarantine on return).

NODE VISUALS (every node shows an icon and a text label; every port shows its key as text beside the handle; colour is never the only cue)
- Start (pill with a flag icon, "Start", padlock): body shows the catch filter in short form, for example "damaged +1 · in_custody · any sev"; the fallback shows "Everything else". One port "caught" (grey). Cannot be deleted or copied alone.
- Investigate (card, magnifier icon, "Investigate"): a team chip ("Asset Store L1"), an SLA badge "8 h", an escalation chip "⤴ Ops Leads L2", a hold icon when information hold is allowed, and a count "4 ports, 0 unconnected". Only the ticked ports are drawn: recovered (green), false_report (grey), replacement (green with an arrow), write_off (dark grey), waiver (teal), quarantine (red-brown).
- Approval (card, check-shield icon, "Approval"): "Owner class" or a team chip, SLA badge if set. Ports approve (green) and reject (red).
- Resolution (terminal: square corners, thick border, "■ {kind}"): the first clause of its effect text (for example "Replace (after issuance, lost)") and, for resume, "at: state before exception" or "at: investigator's choice". No ports. Its border colour matches the resolution badges of the request page.
- Rule, Switch, Set variable, Task, Wait, Notify: as in the main editor. A Task assigned to the requester shows "Submitter".
- An unresolved team shows the red chip "Choose team (was Alpha Engineers L1)" on the node as well as in the inspector. A node with an error has a red outline and an error icon (tooltip with the issue codes); a warning uses amber and a warning icon.

THE H3 GRAPH TO DRAW (default, selected) laid out left to right:
Start (damaged +1, in_custody, any severity) caught to Investigate (Asset Store L1, 8 h, escalates to Ops Leads L2, hold on). From Investigate:
  false_report to Resolution "False report";
  replacement to Approval "Ops Leads L2 (team)"; approve to Resolution "Replace"; reject back to Investigate;
  waiver to Approval "Owner class"; approve to Resolution "Waive"; reject back to Investigate;
  write_off to Approval "Owner class"; approve to Resolution "Write off"; reject back to Investigate;
  quarantine to Resolution "Quarantine on return".
Reject edges are curved back-edges to the Investigate in-connector. Also provide the other four handlers when the user selects them in the list: H1 (Start, Investigate Ops Leads L2 48 h with recovered to Resume "at: state before exception", false_report to False report, replacement, write_off and waiver each through Approval "Owner class" to Replace, Write off, Waive); H2 (Investigate Ops Leads L2 24 h, recovered to Resume "at: investigator's choice", false_report, write_off to Task "Loss report" (Submitter) then Approval "Owner class" then Write off, replacement via Approval "Owner class" to Replace); H4 (Investigate Ops Reviewers L1 24 h, recovered to Resume "investigator's choice", false_report, quarantine to Quarantine on return, write_off via Approval "Owner class" to Write off); fallback (Start "Everything else", Investigate Ops Leads L2 48 h, recovered, false_report, write_off and waiver via owner).

CONNECTION RULES WHILE DRAGGING. A refused drop draws nothing, sends no save, and shows a tooltip (test id handler-canvas__refused) with exactly this text:
  1. Any edge into the Start: "The start of a handler cannot have incoming connections."
  2. Any edge out of a Resolution: "A resolution ends the handler."
  3. Reject to anything but an Investigate step of this handler: "A rejected proposal goes back to an Investigate step."
  4. An Investigate port directly to a Resolution of a different kind: "This outcome can only end in "{kind}"." (for example "This outcome can only end in "Replace".")
  5. Replacement, write-off or waiver directly to a Resolution: "This outcome needs an approval first."
  6. Any edge that closes a cycle other than reject to Investigate: "Only a rejected proposal may loop back."
  7. A Replace resolution in a handler whose filter can catch the return phase, or Quarantine on return where it can catch the pre-issuance phase (on drop and when the filter changes): "Not permitted in the phases this handler catches." (when caused by a filter change this is a node error, not a refused drop).
Also: one edge per out-port ("This port already has a connection. Delete it first."), no self-loop. The "Connect port to…" select in the inspector lists only targets that pass these rules.

EDITING
- Select a node or edge to open the shared inspector (fields as in Prompt 24). Delete removes the selected node with its edges, or the selected edge. The Start and platform-locked nodes cannot be deleted ("This platform step is fixed and cannot be changed.").
- Undo and redo use the single 50-step stack of the editor, shared with the main canvas and all handler canvases; undo after switching handler first selects that handler.
- Keyboard: nodes are focusable; ARIA labels "Investigate: Asset Store L1, 4 ports, 1 unconnected"; "Connect port to…" is the pointer-free alternative to dragging; a "List view" toggle shows the handler as a nested outline.
- Draft state: positions are saved with the draft; seeded handlers ship laid out; autosave and stale-draft behaviour are as in Prompt 23.

READ-ONLY MODE. For owners, and in the Diff tab for "removed" handlers: no palette, no handles, no inspector edits; nodes are selectable and the inspector shows the fields disabled.

STATES. Loading: skeleton canvas. Handler with errors: the node outlines described above and a count in the pane header "2 issues". Empty selection: "Select a handler to see its graph." Mobile: the canvas is 320 px tall, controls reduced to Fit and Expand.

ACCESSIBILITY. Role="application" with an aria-label "Handler graph for H3 Damaged in custody"; focus order is toolbar, palette, canvas nodes, inspector; Esc closes the Expand overlay.

TEST IDS. handler-canvas__node (data-kind, data-node-key, data-diff), handler-canvas__port (data-port), handler-canvas__edge (data-from-port, data-taken, data-diff), handler-canvas__refused, handler-canvas__zoom-in, handler-canvas__zoom-out, handler-canvas__fit, handler-canvas__auto-layout, handler-canvas__minimap, handler-canvas__expand, handler-canvas__close-expand, handler-palette__item (data-kind).
````

---

## Prompt 26: editor, bottom tabs Validation and Variables

**Route:** `/admin/systems/[id]/workflow/edit`, bottom tab strip. **Mirrors:** part 4 §22 and §24.1, §25.1, ST-16 (US-16.1), ST-18, part 6 §27.6 (V-32 to V-43), part 10 §38.4 and §38.5.5, addendum 219 part 12 §42.11 (V-57 to V-69). **Needs:** Prompt 23.

````
Using the app shell, SGDS styling and mock data already in this file, build the bottom tab strip of the workflow editor with its first two tabs, "Validation" and "Variables". The strip sits under the canvas, is 280 px tall by default, can be collapsed to its tab row, and can be resized by dragging its top edge (keyboard: a "Resize" button with arrow keys). In a system workflow the strip has four tabs in this order: "Validation", "Variables", "Simulation", "Diff" (the last two are built by the next prompts; draw them as tabs with an empty placeholder). In the template editor (Prompt 30) it also has "Interface" and "Eligibility", and "Handlers" when the template runs Per item. The Validation tab shows a count badge: red "2" for errors, amber "1" for warnings.

VALIDATION TAB
- Purpose: live list of every problem in the draft. The page never invents rules; it only shows the issues the server returns, and updates within 500 ms of each change (show a thin progress line while checking).
- Header row: a segmented filter "All 3", "Errors 2", "Warnings 1" and a Body SM line "Checked 14:02:11".
- Table (sgds-table, caption "Validation issues"): columns Severity (icon and word: "Error" red, "Warning" amber), Code (V-04), Where (node label or connection), Message. Clicking a row, or pressing Enter on it, selects the node or edge and centres the canvas on it. When the issue belongs to a handler, the template editor switches to the Handlers tab, selects that handler, pans its mini-canvas to the node and selects it; for the coverage code V-32 it selects the handler list and the uncovered grid cells instead.
- The three mock issues for the draft "HRIS · draft r12 · from v1" in the clone demo: 
  1. Error · V-42 · "alpha_l2 (assignee)" · "Choose a team (was Alpha Leads L2)."
  2. Error · V-42 · "alpha_l2 (escalation)" · "Choose a team (was Alpha Engineers L1)."
  3. Warning · V-43 · "Items" (per-item embed) · "No live asset of type "token" in this system. You can still publish."
  (V-42 gives one error per node and slot.)
- Further sample messages the list must be able to show (use them for a "Demo issues" switcher in the demo panel; each has a code and a place):
  V-01 "There must be exactly one start node." · V-02 "The start node cannot have an incoming connection." · V-03 "Add at least one terminal." · V-04 "This node is never entered." · V-05 "ops_l1.reject is not connected." · V-06 "A terminal has no outgoing connections." · V-07 "A connection points to a node that does not exist, or crosses into an embedded workflow." · V-08 "This node cannot be reached from the start." · V-09 "This node cannot reach an end." · V-10 "Only the amend step may form a loop." · V-11 "There must be exactly one amend step, entered only from return ports, leading to an approval." · V-12 "Every approval needs approve, return and reject connected." · V-13 "Reject can only lead to a rejected terminal, and return only to amend." · V-14 "There must be exactly one owner approval on every path to the per-item embed." · V-15 "There must be exactly one per-item embed of a template with custody stages, with all three outputs connected and 1 to 10 items." · V-16 "Issuance cannot be undone: no rejected terminal may follow the per-item embed." · V-17 "Each custody stage must appear once, in order, on every path of the lane template." · V-18 "Splits and joins must be nested properly." · V-19 "A close task must lie on the issued path." · V-20 "A rule refers to a variable that is not available here." · V-21 "Compare values of the same type." · V-22 "That variable cannot be changed here." · V-23 "That value is not one of the options." · V-24 "Choose a live team of this organisation, of a valid type." · V-25 "Enter 1 to 720 hours." · V-26 "Use lowercase letters, digits and underscores, starting with a letter." · V-27 "A required field must be a variable this step can write." · V-28 (warning) "This step has the same team as the step before it." · V-29 (warning) "This rule is always true or always false." · V-30 (warning) "The first approval has no SLA." · V-31 "The Expired terminal is required and fixed." · V-32 (error) "Some combinations of problem type and phase are not caught. Add a handler or fix the fallback." · HANDLER_SHADOWED (warning) "Earlier handlers already catch everything this one catches." · V-33 "A handler must start once, reach a resolution on every path and stay inside itself." · V-34 "Replace, write off and waive need an approval first." · V-35 "That resolution is not permitted in the phases this handler catches." · V-36 (warning) "The approval team is the same as the investigating team." · V-37 "Only a rejected proposal may loop back, and it goes to Investigate." · V-38 "A handler with a replacement port also needs write off or recovered." · V-39 "A handler cannot lead out of itself." · V-40 "Enter 1 to 720 hours." · V-41 "A handler cannot contain a custody stage." · V-44 "A template needs exactly one entry and one to eight distinct exits, and every output must have a value on every exit." · V-45 "This kind of step belongs to a system workflow only: Start, Owner approval, Amend, platform interrupts and Expired cannot be used in a template." · V-46 "A reject or return exit cannot be placed here: it cannot be embedded per item or after an owner approval." · V-47 "Item attributes, custody stages and handlers need a template whose subject is Per item." · V-48 "Bind every input and output of the embedded workflow to a variable of the same type. A Per item output needs a way to combine." · V-49 "Pick a published version of a template of this organisation that can be used this way." · V-50 "A template cannot contain itself, and embedded workflows can be nested at most 3 deep." · V-51 "There must be exactly one per-item embed with custody stages, and no more than one per-item embed on any path." · V-52 (warning) "A newer version of {template} is available. Upgrade to check the bindings." · V-53 "Inside an item, a shared variable can only be added to or subtracted from." · V-54 "Two parallel branches set {variable}. Use an addition on both branches, or set it after the join." · V-55 "{variable} is only combined when all items finish. Read it after Completed." · V-56 (warning) "{variable} is read before anything sets it, so its initial value is used." · V-57 "Line 2 has a syntax error near ')'." · V-58 "risk_levle is not a variable this step can use." · V-59 "This formula is too long or too deeply nested. Split it into more lines." · V-60 "window_label cannot be changed here. A function cannot write a shared variable." · V-61 "Give every function input a value of the same type, and every output a variable." · V-62 "This function does not export a handler." · V-62 (warning) "Not yet checked. Publishing will check it." · V-63 (warning) "This function uses eval. It will be blocked when it runs." · V-64 "Expression and Function steps cannot be used inside a handler." · V-65 "This step saves nothing. Choose a variable for at least one line or output." · V-66 "Store the error type and message in text variables of this workflow, not in a shared variable or a value this step also writes." · V-67 (warning) "per_item is never used by a later line." · V-68 "Give both ports a label of up to 40 characters, and make them different." · V-69 (warning) "risk_band is set here but never used."
  Authored wording note: the codes and rules are normative; apart from V-32, V-42 and HANDLER_SHADOWED the sentences are authored. The server returns V-44 to V-56 for embedded workflows, templates and data flow (addendum 219 part 11 §41.11), and V-57 to V-69 for Expression and Function steps and rule port labels (part 12 §42.11; V-57 and V-58 also carry a line number and a character range, which Prompt 31 underlines); V-44 to V-47 appear mostly in the template editor.
- Each row's code is also shown in small print under the message. A "Copy" button copies the list as text (toast "Issues copied.").
- Publish and Simulate rely on this tab: errors disable them with the reason "Fix the errors in the Validation tab first."; V-42 gives "Choose the unresolved teams first".
- Empty state: a green check, "No problems found." and Body SM "You can simulate and publish."
- Error state (the check failed): sgds-alert warning "Could not check the workflow. Try again." with "Try again".

VARIABLES TAB
- Purpose: declare typed variables the rules can use.
- Toolbar: "Add variable" (primary small), "Declared here" and "Platform (read only)" (segmented). In the template editor (Prompt 30) the Variables tab has two sections, "Interface" and "Local variables"; Interface lists the parameters that cross the template boundary: Name, Direction (In, Out, By reference), Type, Required, Default, Used by. Local variables are private to each use of the template. Direction is chosen in the drawer ("Direction" select, only in template mode); a By reference parameter shows the helper "Reads and writes a variable of the request that uses this template."
- Table (sgds-table, caption "Variables"): Name, Scope, Type, Initial value, Source, Used by, Actions (Edit, Delete). Columns also include Read by and Written by. Declared rows for the seeded HRIS v1 (request scope): risk_level · enum {low, medium, high} · low · field on Ops Reviewers L1; high_risk · bool · false · Assess risk; review_score · int · 0 · written by the Dual review embed (output); required_score · int · 6 · constant, with a padlock and the note "Constant. Changed only in a new version."; items_received · int · 0 · shared with the Items embed (received_counter, by reference; items add to it); pins_reset · int · 0 · written by the Items embed (pin_reset_done, combined by Count true, at Completed). The variables security_score and logistics_score (int 0..5, fields on Security review and Logistics review) and the sum into review_score are local to Dual review v1 and appear only in the template editor; received_counter, pin_reset_done (output, set by Reset PIN) and loss_report_done (field on Loss report, handler H2) are local to Standard custody lane v1. Platform rows (read only, no Edit or Delete): item_count · int · live item count; window_hours · decimal; activity_start_at · timestamp; in a Per item template item.credential_type and item.asset_tag; inside a handler the exception values (exc.type, exc.severity, exc.phase, lane.issued, lane.replacement_count, substitute_available).
- ADD OR EDIT VARIABLE (drawer, 480 px): "Name" (required; lowercase letters, digits and underscores, starting with a letter, up to 40; error "Use lowercase letters, digits and underscores, starting with a letter." and "That name is already used."), Scope is not chosen: it is Request in a system workflow and Local in a template, "Type" (Int, Decimal, Text, Yes or no, Timestamp, Option list), "Initial value" (typed input; for Decimal at most 4 decimal places, error "Use up to 4 decimal places."; for an option list: "Options" chips and a default), "Minimum" and "Maximum" (int only), "Constant" switch (helper "A constant is set here and can only change in a new version."). Wrong-typed initial value: "Enter a value of the chosen type." Buttons "Cancel" and "Save variable".
- Deleting a variable that a rule or a field uses is refused: "This variable is used by Score ok?. Remove those uses first."
- Read-only mode (owner): no add, edit or delete.

STATES. Loading: skeleton rows. Collapsed strip: only the tab row; a badge shows the counts. Mobile: the strip is a full-width bottom sheet opened by a "Problems and variables" button.

ACCESSIBILITY. The tab group follows the SGDS tab pattern with arrow-key navigation; the issue list rows are buttons; severity is a word and an icon; a polite live region announces "2 errors, 1 warning" after each check.

TEST IDS. validation-tab__issue (data-code, data-severity, data-node), validation-tab__filter, validation-tab__copy, variables-tab__row (data-variable), variables-tab__add, variable-form__name, variable-form__type, variable-form__submit.
````

---

## Prompt 27: editor, Simulation tab and the exception injector

**Route:** `/admin/systems/[id]/workflow/edit`, bottom tab "Simulation". **Mirrors:** part 4 §24.3, ST-22 (US-22.1), part 10 §38.1 (simulation injector), §38.4 (error copy), ST-19 and ST-18 (rule renderings), addendum 219 part 12 §42.6 and §42.14. **Needs:** Prompts 23, 24, 26.

````
Using the app shell, SGDS styling and mock data already in this file, build the Simulation tab of the workflow editor's bottom strip, including the exception injector and the trace. Simulation runs the draft in memory: it writes nothing and starts nothing. It is meant for administrators to preview how a request with N items would run before publishing.

TEMPLATE MODE. In the template editor (Prompt 30) the Simulation tab takes the template's inputs instead of the request: "Setup" lists each In parameter of the Interface (for example required_score 6 for Dual review) and, for a template that runs Per item, one row for the single item ("Credential type"); the result shows the exit reached ("passed", "failed", "returned") and the output values (for example review_score 6). Embedded workflows inside the template run inline and appear in the trace as one step "Dual review started" then its inner steps indented.

WHO. Greg Global and Amanda Scoped. Owners do not see the tab's run controls (read only). Simulate is available only when the draft has no errors; otherwise the tab shows the sgds-alert warning "Fix the errors in the Validation tab first." with a link to the Validation tab, and for unresolved teams "Choose the unresolved teams first".

LAYOUT (two columns inside the bottom strip, which can be maximised with an "Expand" button to the full editor height)
- Left (5 of 12): "Setup" and "Script".
- Right (7 of 12): "Trace".
- The canvas above animates the trace: the active node is blue, completed nodes green, taken edges highlighted; a "Step n of N" chip at the canvas corner.

SETUP
- "Number of items" (stepper 1 to the maximum of the per-item embed, for example 5 for HRIS; default 4). For each item a row: "Item 1" and "Credential type" (select Device, Token, Password; only allowed types). Mock: 4 items (token, device, device, device).
- "Initial variables" (collapsed accordion with the writable variables and their values; for example required_score 6, risk_level low).
- Buttons "Run script" (primary), "Step" (outline), "Reset" (ghost). The primary is aria-disabled with the reason when there are errors.

SCRIPT (an ordered list of steps; each step a row with a drag grip, a number, and fields)
- Step row: "Node" (select of nodes that can act, for example Alpha Leads L2, Ops Reviewers L1), "Lane" (only for lane nodes), "Outcome" (Approve, Return, Reject, or a lane outcome), "Fields" (inline inputs for the node's required fields, for example "Risk level" and "Security score"), and a remove button. "Add step" appends one. The mock script for the high-risk path: 1 Alpha Leads L2 approve; 2 Ops Reviewers L1 approve with Risk level high; 3 Ops Leads L2 approve with Security score 3; 4 Asset Store L1 approve with Logistics score 3; 5 Owner approve.
- A step that names a node that is not active makes the trace stop at that step with the message "Step 4: log_review is not active".

THE EXCEPTION INJECTOR (an sgds-card titled "Inject an exception" with this exact structure; the first line shows the items summary)
- Row 1: "Items 4 (token, device, device, device)", "Lane" (select, for example "L3 HT-01"), "Type" (select: missing, unavailable, damaged, wrong_item, identity_mismatch, quantity_mismatch, handover_disputed, receipt_disputed, return_disputed, other), "Severity" (select low, medium, high, critical).
- Row 2: "Phase" with two radios: "Force" (select pre_issuance, in_custody, return, and a state select for that phase such as awaiting_preparation) and "Derive from a script step". Forcing a phase picks the state from the allowed states of that phase, so the pair (type, phase) can always be built; an impossible pair (for example handover_disputed in the return phase) is disabled in the select with the reason "This problem cannot happen in that phase."
- Row 3: "Substitutes available" (checkbox, ticked; when unticked the replacement path fails with "No substitute is available. Write the item off, or wait until one is free."), "Replacement count already on the lane" (select 0, 1, 2).
- Row 4: button "Inject" and, after it, a result line "Route: H1 Before issuance (ordinal 1) · candidates tested: H1 matched". Clicking the line expands the route reasons: every candidate handler with matched or skipped and the reason, for example "H1: matched" and "H3: skipped (phase pre_issuance not in {in_custody})".
- Row 5: "Then act on:" with selects "Step" (investigate), "Outcome" (recovered, false report, replacement, write-off, waiver, quarantine), "Substitute" (for example HT-02, disabled unless the outcome is replacement), then the buttons "Add step" and "Run".
- The injector adds the steps to the script (an "exception" step followed by the act steps). A refused outcome shows its error copy in the trace, exactly these messages:
  NO_SUBSTITUTE "No substitute is available. Write the item off, or wait until one is free."
  SUBSTITUTE_INVALID "That substitute can no longer be used. Pick another from the list."
  REPLACEMENT_LIMIT "The replacement limit for this item has been reached."
  POST_ISSUANCE_REPLACEMENT_DISABLED "This system does not allow replacement after issuance."
  RESUME_STATE_INVALID "The case cannot resume at that state. Choose one of the states listed."
  INVALID_EXCEPTION_STATUS "This problem is on hold or not yet started. Continue investigating first."
  OUTCOME_NOT_ALLOWED "That outcome is not available for this problem."
  The error code is shown in small print under the message.

TRACE (right column)
- Header: the terminal reached ("Completed", "Rejected", "Closed – unfulfilled", "Waiting at Ops Leads L2") as a badge with a word.
- A vertical list of steps, each with an effect (for example "Alpha Leads L2 approved", "Rule Is high risk? true", "Split Review started 2 branches"), and a "State digest" in small print. Selecting a step highlights it on the canvas and shows the variables after it.
- A "Variables after this step" table (Name, Value) with changed values highlighted and a text mark "changed". Example after step 4: review_score 6, security_score 3, logistics_score 3.
- Rule evaluations: each rule with its rendered expression, for example "Is high risk?: high_risk (true) == true -> true" and "Score ok?: review_score (6) >= required_score (6) -> true". For the demo toggle "Use required score 7" show "review_score (6) >= required_score (7) -> false" and the terminal "Rejected".
- Expression and Function steps: an Expression step lists each line, for example "Risk score: risk_score := round((window_hours (6) / item_count (3)) * 2, 1) -> 4.0". A Function step shows "Format window (JS): ok, fuel 3.1 M, 14 ms -> window_label", or on a fault "Format window (JS): fuel_exhausted -> error port" with the badge "Error port" and the line "window_error := fuel_exhausted". Every Expression and Function step starts with a "Reads" line ("reads window_hours 6, item_count 3, risk_level high"); rules show their labels, for example "Many tokens?: needs_dual (false) == true -> No". Under a Function step, a collapsible "Console (simulation only)" lists up to 64 lines in a monospace text block; a note says "Console output is never kept for real requests." In the script, a Function step can be stubbed: "Use stub result" with typed output fields or an error class select.
- Final actualised graph: a button "Show final graph" opens the same graph shape as the request page with the final lanes (1 lane for 1 item, 5 lanes for 5 items).
- Exception results: an exception band under the lane exactly like the request page (phase chip, route chip, resolution badge and effect sentence), so authors can see the effect of each resolution.
- Constraint note under the trace in Body SM: "Simulation writes nothing and does not start a real workflow."
- Errors: the draft has errors, 422 shows the issues list; a stale step shows the message above.

STATES. Idle: "Set up the items and the script, then run." Running: a progress bar and "Running…" for 600 ms. Done: the trace as above. Empty script: the Run button is aria-disabled with the reason "Add at least one step."

ACCESSIBILITY. The injector is a form with visible labels; the trace is an ordered list with a polite live region announcing "Step 4 of 6: Logistics review approved"; animations respect reduced motion; colour on the canvas is always paired with a state word in the node.

TEST IDS. sim-injector__lane, sim-injector__type, sim-injector__phase, sim-injector__substitutes, sim-injector__replacement-count, sim-injector__inject, sim-injector__route, sim-trace__step (data-index), sim-trace__terminal, sim-setup__items, sim-setup__run, sim-trace__fx-line, sim-trace__fn-console, sim-trace__error-target, sim-script__fn-stub.
````

---

## Prompt 28: editor, Diff tab

**Route:** `/admin/systems/[id]/workflow/edit`, bottom tab "Diff". **Mirrors:** part 4 §24.1 (Diff), ST-17 (AC-17.1.1, AC-17.1.2), part 10 §38.5.5. **Needs:** Prompts 23 to 26.

````
Using the app shell, SGDS styling and mock data already in this file, build the Diff tab of the workflow editor's bottom strip. It compares the open draft against any published or retired version of the same system and colours the changes on the canvas.

WHO. Greg Global, Amanda Scoped and (read only) the owners of the system.

LAYOUT
- Toolbar: "Compare against" (sgds-select listing the published and retired versions, for example "v1 published 8 Nov 2026" and "v2 retired"; default the current published version), a summary line "Draft r12 against v1: 5 added, 2 removed, 3 changed", and the toggle "Show on canvas" (switch, on by default).
- Tabs inside the Diff tab: "Nodes", "Connections", "Variables", "Handlers". Each is an sgds-table with a Change column (Added, Removed, Changed, Moved; word, icon and colour: green with "+", red ghost with "-", amber with "~", blue with an arrow) and a "Details" column that lists the fields that changed.
- Mock for the HRIS draft against v1 (from the v2 demo): Nodes: Added "Ops lead review" [ops_lead_review] (Approval, Ops Leads L2); Changed "All reviews" [join_review] (policy All to K of N, k 2); Changed "Items" [items] (maximum items 5 to 3). Connections: Added split_review.branch_3 to ops_lead_review; Added ops_lead_review.approve to join_review; Removed none. Variables: Changed required_score (initial value 6 to 5). Handlers: Changed H3 (investigate SLA 8 h to 6 h); Moved H2 "moved 3 to 2"; Added H5 "Copy of H3"; Removed H4 (struck through, read only).
- Handler rows in the list of the Handlers tab (Prompt 24) carry the same diff badge: "added", "removed" (struck through, read only), "changed" or "moved n to m", always as text beside the colour. The catch-filter form shows added chips underlined and removed chips struck through. The handler mini-canvas highlights added nodes and edges (green outline, "+"), removed ones as ghosts (grey, dashed, "-"), and changed nodes (amber outline, "~"; the inspector lists the changed fields). Handlers match across versions by their key and nodes by their node key.
- Clicking a row selects the node, edge, variable or handler on the canvas or in the editor.

CANVAS OVERLAY. With "Show on canvas" on, added nodes and connections get a green outline and a "+", removed nodes appear as grey dashed ghosts with a "-", changed nodes get an amber outline and a "~". A legend chip row over the canvas: "+ Added · - Removed · ~ Changed".

STATES. No differences: "This draft is the same as v1." with a success tick (a fresh draft from v1 has an empty diff; the fingerprint of the draft's content equals the version's). Loading: skeleton table. Error: sgds-alert danger "Could not compare the versions. Try again." Only one version exists and the draft has no base: "There is nothing to compare against yet."

ACCESSIBILITY. The change is a word as well as a colour; the tables have captions; the select is labelled "Compare against".

TEST IDS. diff-tab__against, diff-tab__summary, diff-tab__row (data-change, data-key), diff-tab__canvas-toggle, handler-list__diff-badge (data-diff).
````

---

## Prompt 29: editor, publish dialog and the other editor dialogs

**Route:** `/admin/systems/[id]/workflow/edit`, `/admin/systems/[id]/workflow` and, for the handler dialogs, `/admin/templates/[id]/workflow/edit`. **Mirrors:** ST-16 (US-16.2), ST-17, ST-33, part 10 §38.4 (error copy), §38.5.4, addendum 219 part 12 §42.10. **Needs:** Prompts 22 to 28.

````
Using the app shell, SGDS styling and mock data already in this file, build the dialogs of the workflow editor and the versions page. All dialogs are sgds-modal (md unless stated), trap focus, close on Escape, return focus to the button that opened them, keep every field value on failure, and show the server error in an sgds-alert at the top. Footers: left "Cancel" (outline), right the main verb.

1. PUBLISH (opened by the editor's "Publish" button, md)
- Title "Publish this workflow". Body MD "Publishing creates a new version. Requests already running keep the version they started with."
- A summary card: "System HRIS", "Based on v1", "Next version v2", "Changes: 3 nodes, 2 connections, 1 variable, 1 embedded workflow upgraded" (the same counts as the Diff tab summary) and any warnings from the Validation tab as a list ("1 warning: No live asset of type "token" in this system.").
- Second-person review: follow the app setting FOUR_EYES_PUBLISH from /api/auth/me (demo panel switch "Four-eyes: Off / Code / All", default Code for the demo). Off: no review section. Code: shown only when function code changed since the base version: a "Function changes" sgds-table (caption "Function changes"), columns Step, Change, Changed by, with the row "Format window · Added · Amanda Scoped" and a "View code changes" link to the Diff tab. All: shown for every publish, listing "Changed by: Amanda Scoped". Above the table, an info sgds-alert: "This app needs a second person to publish these changes. Someone who did not make them must publish." When the current user made the changes (Amanda), "Publish version" is aria-disabled with the reason "Ask another editor to review and publish." When another editor publishes (Greg), a line shows "You are publishing as reviewer."
- "Change notes" (sgds-textarea, required, 1 to 2,000 characters, counter). Errors: "Enter change notes." and "Change notes can be at most 2,000 characters."
- Primary "Publish version" disabled with the visible reason while notes are blank ("Enter change notes"), and the dialog cannot be opened while errors exist (the editor's Publish button is aria-disabled with "Fix the errors in the Validation tab first.").
- Server errors: DEFINITION_INVALID "Fix the errors in the Validation tab, then try again." with a "Show problems" link; STALE_DRAFT "Someone else changed this draft. Reload to continue." with "Reload"; IDEMPOTENCY_TOKEN_USED "That action was already submitted. Refresh to see the result."; FOUR_EYES_REQUIRED "Someone who did not make these changes must publish this version."; FN_RUNNER_UNAVAILABLE "Functions cannot be checked right now. Try again later."; ENGINE_UNAVAILABLE does not apply (publishing needs no engine) but the generic fallback does: "Something went wrong. Try again, or contact support with the code below".
- Success: toast "Version 2 published." and the editor goes read only for that version and the page returns to /admin/systems/[id]/workflow with the new version highlighted. A published version never changes; opening it later shows the banner "This version is published and cannot be changed."

2. ADD HANDLER (md; opened by "Add handler" in the Handlers tab of the template editor)
- Title "Add handler". Fields: "Label" (required, 1 to 80 characters; error "Enter a label up to 80 characters."), "Key" (pre-filled from the label in lowercase with underscores, editable; pattern lowercase letters, digits and underscores starting with a letter, up to 40 characters; errors "Use lowercase letters, digits and underscores, starting with a letter." and "That key is already used in this workflow."), and "Catch filter" with at least one chip (types, phases, severities; same chips as Prompt 24). Error when no chip is chosen: "Choose at least one filter so this handler does not catch everything." Helper "A new handler is placed just above the fallback."
- Primary "Add handler". Result: the handler is inserted with the ordinal just above the fallback and opens with an automatically laid-out template: Start, Investigate (team chip "Choose team", ports recovered and false report), Resolution "Resume" (state before the problem) and Resolution "False report". The Validation tab shows the unresolved-team error until a team is chosen.

3. DELETE HANDLER (generic confirmation, sm). Title "Delete H3?" body "This removes the handler and its 9 nodes. This can be undone with Undo." primary "Delete handler" (danger). The fallback has no delete.

4. DUPLICATE HANDLER (no dialog): copies nodes, edges, positions and filter rows with new keys ("h3_copy", then "h3_copy2"), label "Copy of Damaged in custody", inserted directly after the source. Toast "Handler duplicated." The copy has the same filter as its source, so the list shows the warning "Earlier handlers already catch everything this one catches." on the copy until its filter or order is changed.

5. IMPORT FILE (md; opened by "Import" on the versions page)
- Title "Import a workflow". Body "The file becomes a new draft for HRIS. Teams are matched by cluster name and level in Org Aurora."
- sgds-file-upload (.json, one file, up to 1 MiB). Errors (authored): "That file is larger than 1 MiB. Choose a smaller file." and "That file is not a workflow file. Export one from the workflow versions page."
- Result: success toast "Draft created from the file." and the editor opens. Unresolved teams show as an sgds-alert listing paths before closing: "These teams could not be found: nodes.alpha_l2.assignee (Alpha Leads L2). Create the teams first, or choose another file." and the dialog stays open, with no draft created. DRAFT_EXISTS: "This system already has a draft. Open it, or discard it first."

6. DISCARD DRAFT (sm) and RETIRE VERSION (sm): as Prompt 22.

7. UNSAVED-CHANGE AND CONFLICT BARS (not dialogs). STALE_DRAFT bar across the top of the editor: "Someone else changed this draft. Reload to continue." with "Reload". Offline: "You are offline. Changes will be saved when you reconnect." Autosave failure: "Could not save. Retrying…".

8. KEYBOARD HELP (sm; opened by "?" in the editor). A table of shortcuts: "Tab" move between nodes; "Enter" open the selected node's settings; "Delete" remove the selected node or connection; "Ctrl+Z" undo; "Ctrl+Shift+Z" redo; "Alt+Up" and "Alt+Down" reorder handlers; "Esc" close overlays. Title "Keyboard shortcuts"; primary "Close".

9. CHOOSE A TEMPLATE (lg; opened by the palette item "Embedded workflow" or by "Choose template" on a red placeholder chip)
- Title "Choose a template". Body MD "The template runs as one step. Its inputs and outputs become the step's inputs and outputs." Above the list: a search field "Search templates" and a segmented filter "All", "Usable here". Beside it, a radio "Runs" with "Once" and "Per item" (default Once; it decides which templates are usable).
- List of options (radio cards, test id template-picker__option, data-template-key, data-eligible), one per published template of the organisation, each with the name, key chip, "Subject: None" or "Subject: Per item", "v1" and a one-line interface summary ("Input: required_score · Output: review_score · Exits: passed, failed"). Mock for AUR: Dual review v1 (usable with Once), Standard custody lane v1 (usable with Per item), Basic custody lane v1 (usable with Per item), Pin reset v1 (usable only inside another Per item template), and for contrast one greyed option "Draft: Dual review v2" (not published).
- An ineligible option is greyed, its radio is aria-disabled, and a reason line sits under it in Body SM (never only a tooltip), for example: "Contains an owner approval, which only a system workflow may have." (code V-45), "Runs Per item, but its subject is None." (V-47), "Its exits include a reject, so it cannot be embedded per item." (V-46), "Not published yet. Publish a version first." (V-49). Choose Once and Standard custody lane v1 is greyed with "Runs Per item only. Choose Per item to use it."
- Buttons: "Cancel", primary "Use template" (aria-disabled with the reason "Choose a template" until one is selected). Result: the embed node is placed with its derived ports and an empty bindings state; the inspector opens on the section "Embedded workflow" and V-48 lists the unbound inputs. If a placeholder chip opened the dialog, the choice replaces the placeholder and keeps the existing bindings that still match.
- Errors: TEMPLATE_NOT_ELIGIBLE "This template cannot be used here. Choose another." (422, inline alert, selection kept); EMBED_PIN_INVALID "Pick a published version of a template of this organisation." (422).

10. UPGRADE AN EMBEDDED WORKFLOW (md; opened by "Upgrade to v2" in the inspector of the embed node)
- Title "Upgrade Dual review to v2?" Body MD "This draft will use v2 of Dual review. Requests already running keep the version they started with."
- A "Binding changes" list (sgds-table, caption "Binding changes"): columns Parameter, Change, Result. Mock for Dual review v1 to v2: "required_score · Kept · Still bound to required_score"; "review_score · Kept · Still written to review_score"; "Exit passed · Kept"; "Exit failed · Kept"; "Branch 3 (Ops lead review) · Added inside the template · No action needed". A removed or retyped parameter shows "Removed" or "Type changed" in amber and "Bind it again" with a link that selects the binding in the inspector.
- A line "v2 changes how the review is scored: 3 branches, 2 of 3 must approve." (from the template's change notes).
- Buttons: "Cancel", primary "Upgrade to v2". Result: the pin changes in the draft, the toast "Dual review upgraded to v2." appears, V-52 clears and V-48 re-checks the bindings. EMBED_BINDING_INVALID "Bind every input and output of the embedded workflow, then try again." keeps the dialog open.

TOASTS (sgds-toast, auto-hide after 5 seconds; success tone unless noted): "Version 2 published." · "Draft created." · "Draft created from the file." · "Draft discarded." · "Workflow exported." · "Handler duplicated." · "Issues copied." · "Version retired." · "Dual review upgraded to v2." · "Template created." · "Template version published."
INFO toast: "Changes saved." after an autosave only when the user presses Save by keyboard.

ACCESSIBILITY. Each dialog has a heading, a first-focus field, labelled controls and error text linked with aria-describedby; destructive buttons use the danger tone and a verb ("Delete handler"); toasts are announced politely.

TEST IDS. publish-dialog__notes, publish-dialog__submit, publish-dialog__error, publish-dialog__fn-changes, handler-add-dialog__label, handler-add-dialog__key, handler-add-dialog__filter, handler-add-dialog__create, handler-add-dialog__cancel, handler-add-dialog__error, handler-list__duplicate, handler-list__delete, import-dialog__file, import-dialog__submit, template-picker__dialog, template-picker__option (data-template-key, data-eligible), template-picker__reason, template-picker__confirm, embed-upgrade-dialog__changes, embed-upgrade-dialog__confirm.
````

---

## Prompt 30: workflow template library and template editor mode

**Routes:** `/admin/templates`, `/admin/templates/[id]/workflow`, `/admin/templates/[id]/workflow/edit`. **Mirrors:** part 4 §20 to §24 (templates, interface, eligibility, library, editor), part 5 ST-33, part 6 §28 (cloning templates). **Needs:** Prompts 12, 22, 23, 24, 26, 29.

````
Using the app shell, SGDS styling and mock data already in this file, build the workflow template library, the template versions page and the template mode of the workflow editor, inside the Admin layout of Prompt 12. A template is a workflow that other workflows embed as one node. A template is not a system: it has no owners, no assets and no requests.

PURPOSE AND WHO. Administrators keep reusable workflows (for example the custody lane every HRIS item follows, or a two-branch review) in a library per organisation. A system workflow embeds a published version of a template. Greg Global and Amanda Scoped (own organisation, AUR) can create, edit, publish and clone templates. Carol Sim and Dan Teo (owners) see a template only through the read-only drill-in of an embed node in their own system's workflow ("Open" on the embed node, Prompt 23); they do not see the library. Mona Multi edits templates of AUR and SEN. Bob Lim and every other persona get the 404 page; an SEN template requested by an AUR persona (Amanda) is the 404 page.

1. LIBRARY PAGE (/admin/templates)
- Breadcrumb "Dashboard / Admin / Workflow templates". Heading XL "Workflow templates". Body MD "Reusable workflows that system workflows can embed." Organisation scope as in Prompt 12.
- Toolbar: search "Search templates", filter "Subject" (All, None, Per item), primary "New template" (test id template-library__new).
- Table (sgds-data-table, caption "Workflow templates", test id template-library__row, data-template-key): columns Name, Key (key chip), Subject (None or Per item, as a word with an icon), Interface (summary, for example "In: required_score · Out: review_score · Exits: passed, failed"), Current version ("v1 published" green, or "No published version" neutral), Used by (count, link opens a popover listing "HRIS v1, HRIS v2 (draft)"), Actions (overflow menu: Open versions, Edit draft, Clone, Delete). "Clone" is also a visible outline button per row (data-action clone) that opens the clone dialog (name, key, "Organisation" for a global admin); the key error is "Use lowercase letters, digits and underscores, starting with a letter." and "That key is already used in this library."
- Mock rows for AUR: Pin reset · pin_reset · Per item · "In: none · Out: pin_reset_done · Exits: done" · v1 · used by Standard custody lane v1 (1); Standard custody lane · standard_custody_lane · Per item · "In: ref received_counter · Out: none · Exits: returned" · v1 · used by HRIS v1 (1); Dual review · dual_review · None · "In: required_score · Out: review_score · Exits: passed, failed" · v1 (v2 draft) · used by HRIS v1; Basic custody lane · basic_custody_lane · Per item · "In: none · Out: none · Exits: returned" · v1 · used by FINDB v1, OPSTOOL v1. For Org Sentinel: Sen custody lane · sen_custody_lane · Per item · v1 · used by PORTAL v1.
- "New template" opens a drawer (480 px): "Name" (required, up to 80), "Key" (prefilled from the name, pattern as above, cannot change later), "Subject" radio "None" and "Per item" with helpers "None: the template can run once in any workflow." and "Per item: the template runs once for each item and may hold custody stages and handlers.", "Description" (up to 500). Primary "Create template"; success toast "Template created." and the template editor opens on a new draft.
- Delete is refused while the template is used: "This template is used by HRIS v1. Remove it from those workflows first." (authored). Deleting an unused template asks "Delete {name}?" "This hides {name} from the library. History keeps showing it."
- Empty: "No templates yet. Create your first template." with "New template". Loading: skeleton rows. Error: sgds-alert danger "Could not load templates. Try again.".

2. TEMPLATE VERSIONS PAGE (/admin/templates/[id]/workflow)
- The same pattern as Prompt 22 for a template: heading "Dual review workflow", a Current version card, a Draft card, a Version history table (Version, Status, Published, By, Fingerprint, Change notes, Used by) and the actions "Edit draft" or "New draft from v1", "Export", "Retire". Mock for Dual review: v1 Published 8 Nov 2026, 9:30am, Greg Global, 7c20d4ab, "Initial dual review", used by 1 workflow; Draft r3 "from v1" (the 3-branch version 2). Publishing a template never changes any workflow that uses an earlier version; a Body SM line says so: "Workflows that use v1 keep using v1 until they upgrade."
- A "Used by" card under the history: a table of Workflow (HRIS v1, HRIS v2 draft), Pinned version (v1, v1), Status ("v2 available" badge when a newer published version exists).
- Publish refused with the eligibility errors: the Publish button is aria-disabled with the visible reason "Fix the errors in the Validation tab first." and, when the Eligibility tab shows failures, "This template cannot be embedded yet. See the Eligibility tab." Publish dialog as Prompt 29 with the title "Publish this template"; success toast "Template version published."
- Owners and other personas: 404, except the drill-in described above.

3. TEMPLATE EDITOR MODE (/admin/templates/[id]/workflow/edit)
Same editor frame as Prompt 23 (top bar, left palette, centre canvas, right inspector, bottom tab strip, autosave, undo and redo, STALE_DRAFT bar). The title reads "Dual review · draft r3 · from v1" with the chip "Template" and, beside it, "Subject: None". Differences:
- PALETTE. "Entry" (a flag with an arrow in) and "Exit" (a square with an arrow out) replace "Start", "Owner approval", "Amend" and "Terminal". The kinds Approval, Rule, Switch, Set variable, Split, Join, Wait, Notify, Task and "Embedded workflow" stay. "Custody stage" appears only when the subject is Per item. The platform interrupt nodes and the locked Expired terminal do not exist in a template. Disabled reason for a missing kind: "Only a system workflow has this step." (code V-45).
- ENTRY AND EXIT NODES. Entry: exactly one, label "Entry", one port "next", no in-connector, shows the input chips from the Interface. Exit: one or more (1 to 8), each with an "Exit outcome" key (lowercase letters, digits and underscores) and a "Kind" select Normal, Reject, Return. A Normal exit becomes an out-port of the embed node (for Dual review: passed, failed). A Reject or Return exit is drawn with a red or amber border and the tag "Propagates to the request"; the inspector helper says "Ends the whole request as rejected (or returned to the requester) wherever this template is used." and, when the subject is Per item or the template follows an owner approval, the error "A reject or return exit cannot be used here." (V-46).
- BOTTOM TABS. Validation, Variables, Interface, Eligibility, Handlers (only when Subject is Per item), Simulation, Diff.
- INTERFACE TAB (test id template-editor__interface). Three blocks. (a) "Subject": radio None / Per item. Changing from Per item to None when custody stages or handlers exist is refused: "Remove the custody stages and handlers first." (b) "Inputs": a table Name, Type, Required, Default, By reference (checkbox); add and remove rows; Dual review: required_score · Number · Yes · none; Standard custody lane: received_counter · Number · Yes · By reference (helper "Reads and writes a variable of the request that uses this template."). (c) "Outputs and exits": an Outputs table (Name, Type, Initial value; helper "Every output needs a value on every exit.") with review_score · Number · 0, and an Exits table (Outcome, Kind, Label) with passed · Normal and failed · Normal, plus the propagating reject and return exits drawn by the approvals. Buttons "Add input", "Add output", "Add exit". Errors: "Enter a name with lowercase letters, digits and underscores." and "That name is already used."
- ELIGIBILITY TAB (test id template-editor__eligibility). A checklist of the criteria with a status word and icon each (Pass, Fail, Not applicable): "Entry: exactly one entry with a next port" (V-44); "Exits: one to eight distinct exits, every output has a value on every exit" (V-44); "No system-only steps (Start, Owner approval, Amend, interrupts, Expired)" (V-45); "Reject and return exits are placed correctly" (V-46); "Subject rules: item attributes, custody stages and handlers only in a Per item template" (V-47); "Embedded workflows inside are bound, published and not cyclic, at most 3 deep" (V-48, V-49, V-50). Under the list, a summary line "Can be embedded Once: Yes. Can be embedded Per item: No (the subject is None)." and a derived "Ports preview" (for Dual review "passed, failed"; for Standard custody lane "issued, completed, unfulfilled" when embedded Per item). Failures link to the node. Publishing a template with a failing criterion is refused.
- HANDLERS TAB. Exactly Prompt 24 (handler list, catch filter, coverage grid, effect cards) and Prompt 25 (handler mini-canvas), for the template Standard custody lane v1 only. Replacement limits are not here; they are set on the embed node (Prompt 23).
- INSPECTOR for an embed node inside a template: as Prompt 23 with Runs limited to Once, except that a Per item template may contain a Once embed (for example Standard custody lane embeds Pin reset). Nesting deeper than 3 shows V-50.
- SIMULATION: as Prompt 27, template mode.
- Read-only view (the drill-in from an embed node; anyone who can view the embedding workflow): no palette, no inspector edits, the banner "You are viewing an embedded workflow. Edit it in the template editor." with a link only for those who may edit templates. Breadcrumbs "HRIS v1 › Standard custody lane v1 › PIN reset v1" above the canvas; the last crumb is the current view.

STATES. Loading: skeleton table or canvas. Error: sgds-alert danger with "Try again". Engine banner does not apply (admin CRUD stays enabled). Mobile: the library table becomes stacked cards; the template editor shows the notice "Use a larger screen to edit workflows" and the List view, as Prompt 23.

ACCESSIBILITY. The library table has a caption and the Subject is a word; the Eligibility checklist uses a word and an icon for every status, never colour alone; the Interface tables are real tables with labelled inputs; the exit kind is announced ("Exit passed, normal"); the drill-in breadcrumb is a nav with aria-label "Embedded workflow path" and the current crumb has aria-current="page".

TEST IDS. template-library__row (data-template-key), template-library__new, template-library__clone, template-editor__interface, template-editor__eligibility, template-editor__subject, template-editor__input-row (data-param), template-editor__output-row (data-param), template-editor__exit-row (data-outcome, data-kind), eligibility__criterion (data-code, data-status), embed-node__open, embed-inspector__template, embed-inspector__upgrade, template-picker__option (data-template-key, data-eligible), template-versions__used-by.
````

---

## Prompt 31: editor, Expression (f(x)) and Function (JS) inspectors and the Test run drawer

**Route:** `/admin/systems/[id]/workflow/edit` (and the template editor), right inspector for the node kinds "Expression · f(x)" and "Function · JS". **Mirrors:** addendum 219 part 12 §42.3, §42.4, §42.7, §42.10, §42.14, ST-35, ST-36. **Needs:** Prompts 23, 26, 27.

````
Using the app shell, SGDS styling and mock data already in this file, build the inspector sections for the two formula and code steps of the workflow editor, and the "Test run" drawer for functions. Use the HRIS v2 draft mock: the Expression step "Risk score" (key risk_fx) after Ops Reviewers L1 approve, and the Function step "Format window" (key window_fn) after the owner approve.

PURPOSE AND WHO. Any editor of the workflow (Greg Global, Amanda Scoped, Frank in the demo) can edit both steps, including function code, and can Test run. Carol Sim (owner) sees both read only. Code and formulas are shown as text, never run in the browser.

EXPRESSION · f(x) INSPECTOR (section test id fx-inspector__section)
- Common fields as Prompt 23 (Label, Key, Connect port to… with ports "next" and "error"). Helper under "error": "Taken when a formula cannot be worked out, for example a division by zero. No value is changed."
- "Lines" (ordered list, 1 to 16, "Add line" button, drag handle and Alt+Up/Alt+Down to reorder, "Remove" per line). Each line has:
  - "Name" (lowercase letters, digits and underscores, up to 40; error "Use lowercase letters, digits and underscores, starting with a letter."; "That name is already used in this step.").
  - "Write to" (select: "Keep in this step only" or a writable variable of the same workflow, for example risk_score · Number with decimals).
  - "Formula": a one-line code field (CodeMirror style, monospace, up to 1,000 characters, counter), with syntax colours only. Mock lines:
    1. per_item · Keep in this step only · window_hours / item_count
    2. risk_score · risk_score · round($.per_item * (risk_level == "high" ? 2 : 1), 1)
    3. needs_dual · Keep in this step only · count(items, #.credential_type == "token") >= 2 or $.per_item > 4
  - Chips above the field insert text at the cursor: "Variables" (a popover listing only what this step can use, with type), "Earlier lines" ($.per_item), "Functions" (round, min, max, abs, floor, ceil, len, upper, lower, trim, contains, starts_with, ends_with, now, hours_between, add_hours; in a system workflow also count, some, all). Each function shows a one-line description.
  - Under the field: the type of the result ("Number with decimals") and, after a simulation, "Last simulation: 4.0" in Body SM.
  - Errors V-57 and V-58 underline the exact characters in red (with a wavy underline and an error icon, not colour alone) and show the message under the field, for example "risk_levle is not a variable this step can use." Demo switcher "Show typo" swaps line 2 to contain risk_levle.
- "Reads" (read-only chips above the lines, test id fx-inspector__reads): the variables the formulas use, worked out from the text, for example "window_hours · item_count · risk_level · items". Each chip links to the variable in the Variables tab.
- Per line, beside "Write to": "+ New variable" (opens a small form: name, type pre-filled from the formula's result, starting value pre-filled by type: 0, 0.0000, No or empty text; an option list or a date and time must be chosen). When the line's name matches no variable, a chip under the field offers "Create variable risk_band (Text)" (test id fx-inspector__create-variable); for a whole number it asks "Whole number or decimal?". Until it is chosen the line shows "Choose or create a variable to write to." Nothing is created by running the step. In a template the new variable is private to the template and never added to its inputs or outputs. Deleting the step asks "Also delete risk_band, which only this step sets?" with "Delete step and variable" and "Delete step only" and, for a line that writes a variable, the link button "Branch on this result" (test id fx-inspector__branch). It inserts after "next" a Rule (yes or no result) or a Switch (one case per option, or one threshold case) already filled in, and selects it.
- ON ERROR section (test id node-error__section): "Store error type in" and "Store error message in", each a select of text variables of this workflow with "None" and "+ New variable". Helper: "If the step fails, the type (for example divide_by_zero) and a short message are saved here. Nothing else is changed." Mock: error type → risk_error, message → None.
- A "How this works" disclosure: "Lines run in order. Each name reads the value from when the step started; use $.name to read a line's new value. If any line fails, none of the values are changed, only the error type and message are saved, and the step goes to error."

FUNCTION · JS INSPECTOR (section test id fn-inspector__section)
- Common fields and ports "next", "error" as above. Helper under "error": "Taken when the function fails, runs too long, uses too much memory or returns the wrong shape."
- "Inputs" sgds-table (caption "Function inputs"): columns Name, Type, Value from. Rows: "start · Date and time · activity_start_at", "end · Date and time · activity_end_at", "risk · Number with decimals · risk_score". Value from is a "Variable", "Platform value" or "Value" toggle (start and end use Platform value). "Add input" (up to 8). Under the table: "The function sees only these inputs. Numbers with decimals arrive as text, for example "4.0"."
- "Outputs" sgds-table (caption "Function outputs"): columns Name, Type, Write to variable. Row "window_label · Text · window_label". "Add output" (up to 8). A shared variable is not offered as a target; helper "A function cannot write a shared variable." Each row has "+ New variable" and "Branch on this result" as above.
- ON ERROR section as above. Mock: error type → window_error, message → None.
- "Code": a full-height code editor (monospace, line numbers, syntax colours, up to 20 KiB, counter "1.2 KiB of 20 KiB"), pre-filled with:
  export const handler = async (input) => {
    const start = dayjs(input.start).utcOffset(480);
    const end   = dayjs(input.end).utcOffset(480);
    const risk  = new Big(input.risk).toFixed(1);
    return { window_label: `${start.format('ddd D MMM HH:mm')}–${end.format('HH:mm')} SGT · risk ${risk}` };
  };
  Above it, Body SM: "JavaScript (ES2020). dayjs and Big are available. Imports, network, files and random numbers are not."
  A "Last check" chip: "Checked with runner r2026.11.1" (success) or "Not yet checked" (neutral) or the V-62 message (danger). V-63 warnings show as amber gutter marks with the message on hover and in a list under the editor (never hover only).
  Read-only mode (Carol): the editor is read only, Inputs and Outputs show as plain tables, and "Test run" is hidden.
- Button "Test run" (outline; test id fn-inspector__test-run) opens the drawer.

TEST RUN DRAWER (sgds-drawer, right, md; test id fn-test__drawer)
- Title "Test run · Format window". Body SM "Runs the function in the sandbox with the values below. Nothing is saved."
- One typed field per input (date-time pickers for start and end, a decimal field for risk), pre-filled with sample values; "Time now" (date-time, default the current time).
- Primary "Run". Results area:
  - Success: badge "Ran" and the outputs table ("window_label · Tue 10 Nov 14:00–18:00 SGT · risk 4.0"), a fuel bar ("3.1 M of 500 M units", aria-valuenow), "14 ms", "Runner r2026.11.1".
  - Failure: badge "Failed" and the class in words with the code in small print: "Ran out of steps (fuel_exhausted)", "Used too much memory (memory_exceeded)", "The function threw an error (thrown): eval is not available", "Returned the wrong shape (invalid_output): unexpected key extra". A sentence: "In a request, this step would go to error."
  - Console: up to 64 lines, monospace, as plain text; "Console output is shown only here and in simulation."
- Demo switcher in the demo panel: "Infinite loop", "Uses eval", "Wrong shape", "Good code" swap the code and the result.
- Errors: RATE_LIMITED "You have run too many tests. Wait a minute and try again."; FN_RUNNER_UNAVAILABLE "The function runner is not available. Try again later."; 422 per-field messages ("Enter a number with up to 4 decimal places.").

STATES. Expression: empty ("Add a line to work out a value."), editing, error underlines, simulated values. Function: empty code ("Write a handler function, or paste one."), checking ("Checking…" chip), checked, check failed, read only, runner unhealthy (banner "Functions run without the sandbox hardening in this environment." when health reports unhardened). Test run: idle, running (spinner "Running…"), success, each failure class, rate limited.

ACCESSIBILITY. Code fields have visible labels, announce their line count and errors through aria-describedby; underlined errors are also listed as text; the drawer traps focus and returns it to "Test run"; the fuel bar has a text value; the console is a labelled region, not a live region.

TEST IDS. fx-inspector__section, fx-inspector__line (data-index), fx-inspector__line-name, fx-inspector__line-target, fx-inspector__line-source, fx-inspector__add-line, fx-inspector__issue, fn-inspector__section, fn-inspector__inputs, fn-inspector__outputs, fn-inspector__code, fn-inspector__check-chip, fn-inspector__test-run, fn-test__drawer, fn-test__input (data-name), fn-test__run, fn-test__result, fn-test__error-class, fn-test__fuel, fn-test__console, fx-inspector__reads, fx-inspector__branch, fx-inspector__new-variable, fx-inspector__create-variable, fn-inspector__branch, node-error__section, node-error__class-target, node-error__message-target.
````
