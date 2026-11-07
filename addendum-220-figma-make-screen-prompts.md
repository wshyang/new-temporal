# Addendum 220 (part 1) — Figma Make prompts for every screen of the Temporal break-glass MVP: how to use, foundation, shared states, login

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only; no code, no schema and no behaviour change.
**Tag:** `[CUSTOM: design handoff]`
**Relates to:** addendum 219, all parts (the specification of the Temporal break-glass MVP). 219 is normative.

**Parts of this addendum (each file is at most 800 lines):**
- Part 1 `addendum-220-figma-make-screen-prompts.md` (this file): header, how to use, Prompt 0 (foundation), Prompt 0B (shared states and dialogs), Prompt 1 (login).
- Part 2 `addendum-220-figma-make-screen-prompts-part-2-dashboard-new-request-and-request-shell.md`: Prompts 2 to 4 (dashboard, new request, request page shell and header).
- Part 3 `addendum-220-figma-make-screen-prompts-part-3-request-graph-actions-and-panels.md`: Prompts 5 to 9 (request graph, exception bands, action panel and dialogs, side panels, timeline).
- Part 4 `addendum-220-figma-make-screen-prompts-part-4-reports-assets-and-admin.md`: Prompts 10 to 19 (issuance and returns report, asset detail, admin shell and the seven admin pages).
- Part 5 `addendum-220-figma-make-screen-prompts-part-5-wizard-versions-and-editor-layout.md`: Prompts 20 to 24 (create-system wizard, workflow versions, editor layout and canvas with the embedded workflow node, Handlers tab, coverage grid).
- Part 6 `addendum-220-figma-make-screen-prompts-part-6-editor-tabs-simulation-and-dialogs.md`: Prompts 25 to 31 (handler mini-canvas, Validation and Variables tabs, Simulation and injector, Diff, publish and editor dialogs including the template picker, workflow template library and template editor mode, and the Expression (f(x)) and Function (JS) inspectors with the Test run drawer).

## What was decided

Write one paste-ready Figma Make prompt per app screen, and for the larger screens several prompts, so that
a designer or a Figma Make session can produce a high-fidelity prototype of **every screen of the
addendum 219 MVP** in the Singapore Government Design System (SGDS). Scope is the 219 MVP screens only, not
the current Firefighter app in `src/app/components`. The addendum contains prompts only: there are no
separate specification sections per screen, and the requirements live inside the prompt blocks. Short
framing text around each block (route, who sees it, when to paste, what it needs, which 219 section it
mirrors) sits outside the block.

## Why

The user asked for Figma Make prompts that describe the UX and UI of every screen in great clarity and
detail. Addendum 219 specifies the screens across seven files and about 7,200 lines. Figma Make cannot read
this repository, so each prompt copies the facts it needs (routes, roles, exact copy, states, mock data)
and never says "see section X".

## Relation to 219

- **219 is normative.** Where a prompt and 219 conflict, 219 wins and the prompt is corrected. The prompts add no behaviour.
- Copy strings from 219 §38.2 and §38.4 are reused verbatim. Strings that 219 does not define are written
  in SGDS style by this addendum and are listed as "authored" in the report that accompanied it, so they
  can be replaced by the server's text.
- Two kinds of 219 strings were adjusted for the SGDS writing rules (no em dash, no contractions): the
  banner "Workflow engine unavailable — actions are disabled" is written "Workflow engine unavailable.
  Actions are disabled." and the connection refusal "This port already has a connection — delete it
  first" is written "This port already has a connection. Delete it first." The node label "Closed –
  unfulfilled" is kept exactly as 219 writes it, because it names a graph terminal.
- Request numbers follow the normative format `BG-{SYSTEM}-{yyyymmdd}-{NNN}` (219 §14 AC-4.1.3). The sketch
  in 219 §38.2 uses the shortened form "HRIS-2026-0099" for illustration only.
- The create-system wizard has a step 4 "Confirm" and, inside step 3, a "Create draft" button (219 §38.3).
  The prompts resolve this as: for a cloned workflow, "Create draft" ends the wizard; for a blank or imported
  workflow, step 4 "Confirm" ends it with "Create system".

## Practical impact

- Nothing in the product changes. The prompts are an input to design work and to a clickable prototype.
- If 219 changes (copy, routes, states), update the matching prompt in the same change. The `data-testid`
  values in the prompts are the 219 §38.4 contract where 219 defines them, and follow the same pattern
  `{component-kebab}__{element-role}` elsewhere.
- Mock data in the prompts uses only the synthetic personas, organisations, teams, systems and assets of
  219 §13. Email addresses are synthetic and end in `.example`.
- SGDS token values (hex colours, exact sizes of the focus ring) are not written in the prompts, because
  they are not in the SGDS skill files available to the author. The prompts name SGDS tokens and components
  and let Figma Make take the values from the SGDS library.

## How to use

1. Open a **new Figma Make file**. Paste **Prompt 0** (foundation) and wait for it to finish. It builds the
   SGDS app shell, the shared mock-data module and the persona switcher.
2. In the **same file**, paste **Prompt 0B** (shared states and dialogs), then the screen prompts **in order**
   (Prompt 1, 2, 3 and so on). Every screen prompt begins "Using the app shell, SGDS styling and mock data
   already in this file…" so that it composes with the earlier ones, and names the mock records it needs.
3. Large screens are split into several prompts (the request page has six, the editor has eight). Paste them
   in order; each says which earlier prompt it extends.
4. After each prompt, switch the persona in the demo persona switcher and check the roles named in the
   prompt. If Figma Make drops a detail, paste the single sentence again as a follow-up.
5. Do not paste the framing text outside the blocks; paste only what is inside the block.

**Order of prompts**

| # | Prompt | Route | Part |
|---|---|---|---|
| 0 | Foundation: SGDS look, app shell, mock data, accessibility | all | 1 |
| 0B | Shared states, error copy, toasts, confirmation dialogs, 403 and 404 | all | 1 |
| 1 | Login | `/login` | 1 |
| 2 | Dashboard | `/` | 2 |
| 3 | New request | `/requests/new` | 2 |
| 4 | Request page: shell, header, banners, layout | `/requests/[id]` | 2 |
| 5 | Request page: workflow graph | `/requests/[id]` | 3 |
| 6 | Request page: exception bands, supplementary lanes, handler graph | `/requests/[id]` | 3 |
| 7 | Request page: action panel | `/requests/[id]` | 3 |
| 8 | Request page: action dialogs | `/requests/[id]` | 3 |
| 9 | Request page: side panels, issuance and returns, timeline | `/requests/[id]` | 3 |
| 10 | Issuance and returns report | `/reports/issuance-returns` | 4 |
| 11 | Asset detail | `/assets/[id]` | 4 |
| 12 | Admin shell and sidebar | `/admin/*` | 4 |
| 13 | Admin: organisations | `/admin/organisations` | 4 |
| 14 | Admin: clusters and teams | `/admin/clusters` | 4 |
| 15 | Admin: memberships | `/admin/memberships` | 4 |
| 16 | Admin: users | `/admin/users` | 4 |
| 17 | Admin: roles and grants | `/admin/roles` | 4 |
| 18 | Admin: systems (owners, delegates, PA) | `/admin/systems` | 4 |
| 19 | Admin: assets | `/admin/assets` | 4 |
| 20 | Create-system wizard, details and owners | `/admin/systems/new` | 5 |
| 21 | Create-system wizard, workflow start, clone and import | `/admin/systems/new` | 5 |
| 22 | Workflow versions | `/admin/systems/[id]/workflow` | 5 |
| 23 | Editor: layout, canvas, palette, inspector | `/admin/systems/[id]/workflow/edit` | 5 |
| 24 | Editor: handlers tab, coverage grid, effect cards (template editor for the custody lane) | same | 5 |
| 25 | Editor: handler mini-canvas | same | 6 |
| 26 | Editor: bottom tabs Validation and Variables | same | 6 |
| 27 | Editor: Simulation with the exception injector | same | 6 |
| 28 | Editor: Diff | same | 6 |
| 29 | Editor: publish dialog and editor dialogs (template picker, upgrade) | same | 6 |
| 30 | Workflow template library and template editor mode | `/admin/templates`, `/admin/templates/[id]/workflow/edit` | 6 |

---

## Prompt 0: foundation (paste first)

**Mirrors:** 219 §12 (routes), §13 (personas), §8 (roles), §38.4 (accessibility and test ids). **Needs:** nothing.

````
You are building a clickable, high-fidelity prototype of a government web application called "Break-glass".
It manages the custody of physical emergency credentials (secure phones, hardware tokens, laptops, sealed
password envelopes): people request them, reviewers approve, an asset store hands them over, the requester
returns them. Build it as a multi-page React app with client-side routing and mock data only. There is no
backend. Everything in this file must follow the Singapore Government Design System (SGDS).

PART A. DESIGN SYSTEM (SGDS)

1. Use the SGDS web component library @govtechsg/sgds-web-component wherever it has a component, and tag
   components by name: sgds-masthead, sgds-mainnav (with sgds-mainnav-item and sgds-mainnav-dropdown),
   sgds-footer, sgds-breadcrumb, sgds-sidenav (admin pages), sgds-subnav, sgds-button, sgds-icon-button,
   sgds-close-button, sgds-link, sgds-dropdown, sgds-overflow-menu, sgds-input, sgds-textarea, sgds-select,
   sgds-combo-box, sgds-datepicker, sgds-checkbox, sgds-radio, sgds-switch, sgds-file-upload, sgds-badge,
   sgds-alert, sgds-system-banner, sgds-toast, sgds-modal, sgds-drawer, sgds-tab-group with sgds-tab and
   sgds-tab-panel, sgds-table, sgds-data-table, sgds-pagination, sgds-card, sgds-accordion,
   sgds-description-list, sgds-stepper, sgds-progress-bar, sgds-skeleton, sgds-spinner, sgds-tooltip,
   sgds-divider, sgds-icon. If a component cannot render in the preview, reproduce its look and behaviour
   with plain React and CSS. Never set explicit widths on an sgds-* element; control width with the parent
   layout (flex, grid, wrapper).
2. Typeface: Inter, weights 300, 400, 600 and 700. Type scale (SGDS names): Heading XL (h1, bold), Heading LG
   (h2, bold), Heading MD (h3, semibold, the default), Heading SM (h4, semibold), Body LG, Body MD (the
   default body text), Body SM (captions, small print, error codes). Do not use the Display sizes.
3. Colour: use SGDS semantic colour families by name and never invent hex values: primary (brand), accent
   (informational, links), success, danger, warning, purple, cyan, neutral, each with muted, default and
   emphasis steps. Use the SGDS default product primary scale (do not retheme). Surfaces and borders use the
   SGDS semantic surface, border and text tokens. Day theme only.
4. Radius: SGDS general scale for cards, buttons, badges and panels (default medium, 8 px); SGDS form radius
   for native form controls. Spacing: SGDS 4 px scale; use the semantic text-gap, layout-gap and
   layout-padding steps rather than raw numbers.
5. Breakpoints (SGDS): 320, 512 (sm), 768 (md), 1024 (lg), 1280 (xl), 1440 (2xl). Content container: the SGDS
   container (centred, 1168 px wide at 1280 and above, 888 px at 1024). Wide screens (request page, editor)
   use a fluid full-width container with the SGDS layout padding.
6. Focus: every interactive element shows the SGDS focus ring on keyboard focus. Never remove the outline.
7. Writing style (SGDS): UK English; sentence case for headings, labels and buttons; no em dash; no
   contractions; no "please"; active voice; short sentences; dates as "8 Nov 2026" and times as "2:07pm" in
   tables and prose, all in Singapore time (SGT). Where this prompt gives an exact string, use it exactly,
   including strings that contain a 24-hour time such as "fired 14:07 by Greg Global".

PART B. STATUS LANGUAGE (use everywhere; colour is never the only cue, always add the word or an icon)

Request status pill (sgds-badge) and text:
- submitted: "Submitted", accent outlined
- under_review: "Under review", accent
- returned_to_vendor: "Returned to vendor", warning
- rejected: "Rejected", danger
- approved: "Approved", success outlined
- credential_issued: "Credential issued", success
- closed: "Closed", neutral
A rejected request may carry a reason line under the pill: "Expired: the activity window passed before owner
approval", "Cancelled: the requester was deactivated", "Stopped by a rule: the review score was below the
required score". A closed request with no live items carries "Closed – unfulfilled: every item was written off".

Workflow node and lane states (used in the graph, also in lists): pending = grey border; active = blue (accent)
border with a soft glow; done = green; returned = amber; rejected or expired = red; exception open = red dashed
border with a warning icon; overdue = red badge "Overdue {n}h"; excluded (written off or replaced) = strike-through
text on a grey fill; supplementary = dashed border.

Custody states for an item (sgds-badge, outlined): awaiting_preparation "Awaiting preparation", prepared
"Prepared", handover_pending "Handover pending", in_requester_custody "With requester", return_pending "Return
pending", received_by_agency "Received by agency", returned_to_inventory "Returned to inventory", exception_open
"Problem open", resolved_exception "Resolved", cancelled "Cancelled".

Exception status: open "Open", investigating "Investigating", awaiting_information "Awaiting information",
awaiting_resolution_approval "Awaiting approval", resolved "Resolved", cancelled_as_false_report "False report".

PART C. APP SHELL (every page except /login)

Page structure, top to bottom: sgds-masthead (fluid, "A Singapore Government Agency Website", with the "How to
identify" toggle), then an optional sgds-system-banner row for global messages, then sgds-mainnav (fluid), then the
page content, then sgds-footer.
- Main nav brand: "Break-glass". Items depend on the signed-in persona:
  Dashboard (everyone); New request (only personas who can submit: Alice Tan, Bob Lim, Betty Ong, Claire Wong,
  Emma Yeo, Ivan Ng, Gina Ho); Reports (owners, members of an investigate team, organisation admins, global admins;
  hidden for Bob Lim and Priya Nair); Admin (sgds-mainnav-dropdown: Organisations, Clusters and teams, Memberships,
  Users, Roles and grants, Systems, Workflow templates, Assets; shown to global admin and organisation admin (an organisation admin only ever sees the organisations their grants
  name, 219 D-38); a team admin such as Alice
  Tan sees only Admin > Memberships).
- Right end of the nav: a user menu (sgds-mainnav-dropdown with an aria-label) showing the person's name, a small
  chip with their organisation code (for example AUR) and their role (for example "Vendor L1"), then the items
  "Switch persona" (demo mode only) and "Sign out". If the persona is acting for someone else, show the line
  "Acting as delegate for Carol Sim until 10 Nov 2026" under the name, and in the nav a neutral badge "Acting as
  delegate".
- Demo mode strip (demo only): directly under the nav, a slim neutral bar "Demo mode. Signed in as {name}
  ({organisation} · {role})" with a button "Switch persona" that opens a drawer listing every persona grouped
  by organisation then role (same list as the login picker). Selecting a persona signs in at once, shows the
  toast "Signed in as {name}" and returns to the dashboard.
- Footer: sgds-footer with title "Break-glass", description "Custody of emergency credentials for government
  systems.", links Contact, Feedback, Privacy statement, Terms of use, copyright "Government of Singapore".
- Breadcrumbs (sgds-breadcrumb) on every page below the dashboard, for example "Dashboard / Requests /
  BG-HRIS-20261108-003".
- Global engine banner: when the mock flag "engineDown" is on, show on every page an sgds-system-banner (danger)
  "Workflow engine unavailable. Actions are disabled." and set every action button to aria-disabled. Add a
  demo-only toggle for the flag inside the persona drawer.
- Polling indicator (request page only, defined in a later prompt): a small text "Live" with a dot and "updated
  {n} s ago".

PART D. SHARED MOCK DATA (one module, used by every page; synthetic data only)

Password for every demo persona is not shown anywhere. Emails are fake and end in .example.

Organisations: AUR "Org Aurora"; SEN "Org Sentinel".

People (name · email · organisation · role):
- Alice Tan · alice.tan@aur.example · AUR · Vendor L1, Alpha Engineers L1, team admin
- Bob Lim · bob.lim@aur.example · AUR · Vendor L1, Alpha Engineers L1
- Aaron Koh · aaron.koh@aur.example · AUR · Vendor L2, Alpha Leads L2
- Betty Ong · betty.ong@aur.example · AUR · Vendor L1, Beta Engineers L1
- Brian Goh · brian.goh@aur.example · AUR · Vendor L2, Beta Leads L2
- Claire Wong · claire.wong@aur.example · AUR · Agency, Ops Reviewers L1
- Emma Yeo · emma.yeo@aur.example · AUR · Agency, Ops Reviewers L1; delegate for Carol Sim (now to +2 days)
- Ivan Ng · ivan.ng@aur.example · AUR · Agency, Ops Reviewers L1; primary owner of OPSTOOL
- Frank Chua · frank.chua@aur.example · AUR · Agency, Ops Leads L2
- Grace Neo · grace.neo@aur.example · AUR · Agency, Asset Store L1
- Leo Tan · leo.tan@aur.example · AUR · Agency, Asset Store L1
- Priya Nair · priya.nair@aur.example · AUR · PA, Owner Support L1
- Carol Sim · carol.sim@aur.example · AUR · Agency, owner (primary of HRIS, backup of FINDB and OPSTOOL)
- Dan Teo · dan.teo@aur.example · AUR · Agency, owner (primary of FINDB, backup of HRIS)
- Amanda Scoped · amanda.scoped@aur.example · AUR · organisation admin (org_admin) for AUR
- Mona Multi · mona.multi@aur.example · AUR and SEN (member of both) · organisation admin (org_admin) for AUR and SEN
- Greg Global · greg.global@aur.example · global admin (full_admin)
- Gina Ho · gina.ho@sen.example · SEN · Vendor L1, Gamma Services L1
- Hank Low · hank.low@sen.example · SEN · Agency, SEN Ops L1
- Sam Lee · sam.lee@sen.example · SEN · Agency, owner of PORTAL

Clusters and teams, organisation AUR: Alpha Pte (vendor): Alpha Engineers L1 (Alice team admin, Bob), Alpha Leads
L2 (Aaron). Beta Pte (vendor): Beta Engineers L1 (Betty), Beta Leads L2 (Brian). AUR Ops (agency): Ops Reviewers
L1 (Claire, Emma, Ivan), Ops Leads L2 (Frank). AUR Logistics (agency): Asset Store L1 (Grace, Leo). AUR PA (pa):
Owner Support L1 (Priya). Organisation SEN: Gamma Services (vendor) L1 Gina; SEN Ops (agency) L1 Hank.

Systems: HRIS (AUR, owners Carol primary and Dan backup, delegate Emma now to +2 days, PA Owner Support, chain
Alpha L1 then Alpha L2 then Ops L1 then owner). FINDB (AUR, Dan primary and Carol backup, chain Beta L1, Beta L2,
Ops L1, Ops L2, owner). OPSTOOL (AUR, Ivan primary and Carol backup, chain Ops L1, Ops L2, owner). PORTAL (SEN,
Sam Lee, chain Gamma L1, SEN Ops L1, owner). Each system has one owning organisation.

Workflow template library (published, not requests): AUR: Pin reset v1 (item), Standard custody lane v1 (item,
embeds Pin reset), Dual review v1 (no subject; input required_score, output review_score, exits passed and failed)
with Dual review v2 as a draft, Basic custody lane v1 (item). SEN: Sen custody lane v1 (item). HRIS embeds Dual review
v1 once and Standard custody lane v1 per item; FINDB and OPSTOOL embed Basic custody lane v1; PORTAL embeds Sen
custody lane v1.

Assets (all manual, physical): HRIS: SP-01, SP-02, SP-03 (secure phone, type device); HT-01, HT-02 (hardware
token, type token); LT-01, LT-02 (laptop, type device). FINDB: HSM-01 (token). OPSTOOL: KB-01 (sealed password
envelope, type password). PORTAL: HT-S1 (token). Credential types: device, token, password.

Request fixtures (independent snapshots; do not try to keep asset holds consistent across them). All are
organisation AUR, dated 8 Nov 2026:
- RQ1 BG-HRIS-20261108-001: Bob Lim; HRIS; credential_issued; items SP-01, HT-01, LT-01 all with the requester;
  summary issued 3, outstanding 3, returned 0; return due.
- RQ2 BG-HRIS-20261108-002: Bob Lim; HRIS; returned_to_vendor; returned by Aaron Koh with the remark "Wrong SR";
  items SP-02, HT-02.
- RQ3 BG-HRIS-20261108-003: Bob Lim; HRIS; credential_issued; the showcase request with six lanes and exceptions
  (defined in the request-page prompts). It has a demo state switcher with four states A, B, C, D.
- RQ4 BG-HRIS-20261108-004: Bob Lim; HRIS; closed, unfulfilled (one token, replaced, then both written off).
- RQ5 BG-HRIS-20261108-005: Bob Lim; HRIS; credential_issued; SP-01 return verified damaged, problem open.
- RQ6 BG-FINDB-20261108-001: Betty Ong; FINDB; submitted; waiting for Beta Leads L2 (Brian Goh).
- RQ7 BG-HRIS-20261108-007: Bob Lim; HRIS; under_review at Alpha Leads L2; 2 hours overdue; escalated to Alpha
  Engineers L1.
- RQ8 BG-HRIS-20261108-008: Bob Lim; HRIS; rejected, expired (the activity window passed).
- RQ9 BG-HRIS-20261108-009: Bob Lim; HRIS; rejected by a rule (review score 5, required 6).
- RQ10 BG-OPSTOOL-20261108-001: Ivan Ng; OPSTOOL; under_review at the owner stage (Ivan is both owner and requester).
- RQ11 BG-PORTAL-20261108-001: Gina Ho; PORTAL; submitted. Organisation SEN, so any AUR persona gets the 404 page.

Persona switcher: all personas above, grouped as Organisation, then role group (Vendor L1, Vendor L2, Agency Ops,
Agency Logistics, PA, Owners and delegates, Admins). Each item reads "AUR · Vendor L1 · Bob Lim". Global admin Greg
Global sits in a group called "Global". Mona Multi sits in a group called "Multiple organisations" and her item reads
"AUR, SEN · Org admin · Mona Multi".

PART E. GLOBAL STATE PATTERNS

- Loading: sgds-skeleton shaped like the final content (table rows, cards), never a blank page. Buttons that
  save show sgds-spinner and the label changes to the present progressive ("Saving…").
- Empty: say why it is empty and offer the next step, for example "No requests yet. Create your first request."
- Error: an sgds-alert (danger, dismissible) with the message in sentence case and the error code in Body SM
  small print under it, for example "STALE_STATE". Unknown errors read "Something went wrong. Try again, or contact
  support with the code below".
- Permission: 403 is shown inline as "You do not have permission to do this." 404 is a full neutral page "We cannot
  find that page." and is identical for a missing and a hidden object.
- Read-only: controls are replaced by text, or disabled with aria-disabled and a visible reason.
- Every table has a caption, sortable headers where useful, and sgds-pagination at 10 rows.
- Mobile (under 768 px): tables become stacked cards with the label above each value; multi-column forms become
  one column; the main nav collapses into the SGDS hamburger.

PART F. ACCESSIBILITY (WCAG 2.2 AA) AND TEST IDS

- Colour is never the only cue; every coloured state also has a word or icon.
- A disabled action uses aria-disabled="true" (not the disabled attribute) and links to its visible reason with
  aria-describedby, so a screen reader reads the reason. The reason is visible text beside or under the button.
- Every flow is reachable by keyboard. Dialogs trap focus, close on Escape and return focus to the opener. After
  a route change, move focus to the page heading. Live updates use aria-live="polite".
- Touch targets are at least 24 by 24 px.
- Add data-testid attributes using the pattern {component-kebab}__{element-role}, for example login-form__submit.
  Where a later prompt gives a test id, use it exactly. Repeated elements share one id and carry data-* attributes
  (for example data-handler-key).

Build the shell, the mock-data module, the persona switcher and an empty dashboard route now. Do not build the
other pages yet.
````

---

## Prompt 0B: shared states, error copy, toasts and confirmation dialogs

**Mirrors:** 219 §11 (status codes), §38.4 (error copy), §14 ST-7 (admin reset), ST-5 (return, reject).
**Needs:** Prompt 0 (shell, mock data). **Paste once;** later prompts reuse these components by name.

````
Using the app shell, SGDS styling and mock data already in this file, add the shared states, error copy, toasts
and confirmation dialogs that every later page will reuse. Build them as reusable components and show each one in a
hidden "/design/shared" page with a button per variant, so they can be reviewed.

1. NEUTRAL PAGES
- 404 page: sgds-alert not used; a centred card, Heading LG "We cannot find that page.", Body MD "Check the address,
  or go back to your dashboard." and an sgds-button primary "Go to dashboard". The text is identical for a missing
  object and for an object the person is not allowed to see. It never names an organisation, a system or a request.
- 403 inline panel (used inside a page when the person can see the page but not the action): sgds-alert danger,
  title "You do not have permission to do this."
- 401: after 15 minutes of inactivity every call fails; redirect to /login?next=… and on the login page show an
  sgds-alert warning "Your session has ended. Sign in again."
- Engine unavailable: the system banner of the app shell plus, on a form that failed to submit, an inline alert
  "The workflow service is not available. Try again shortly." with the form values kept.

2. ERROR COPY TABLE (use these messages exactly; show the code in Body SM small print under the message)
- NO_SUBSTITUTE: "No substitute is available. Write the item off, or wait until one is free."
- SUBSTITUTE_INVALID: "That substitute can no longer be used. Pick another from the list."
- REPLACEMENT_LIMIT: "The replacement limit for this item has been reached."
- POST_ISSUANCE_REPLACEMENT_DISABLED: "This system does not allow replacement after issuance."
- RESUME_STATE_INVALID: "The case cannot resume at that state. Choose one of the states listed."
- INVALID_EXCEPTION_STATUS: "This problem is on hold or not yet started. Continue investigating first."
- OUTCOME_NOT_ALLOWED: "That outcome is not available for this problem."
- NODE_NOT_ACTIVE: "This step has already moved on. Refresh to see the latest."
- STALE_STATE: "Someone else changed this item. Refresh and try again."
- ASSET_IN_USE: "The substitute was taken by another request. Pick another."
- ASSET_QUARANTINED: "This asset is quarantined and cannot be requested."
- SOD_VIOLATION: "You cannot approve a problem you reported or investigated."
- CUSTODY_IN_PROGRESS: "Some items have already been handed over. Use "Force" with notes to reset the rest."
- PLATFORM_NODE_LOCKED and PLATFORM_TERMINAL_LOCKED: "This platform step is fixed and cannot be changed."
- HANDLER_SHADOWED (warning): "Earlier handlers already catch everything this one catches."
- V-32 (error): "Some combinations of problem type and phase are not caught. Add a handler or fix the fallback."
- ASSIGNEE_UNRESOLVED (V-42): "Choose a team (was {source team})."
- CLONE_MAPPING_INVALID: "Pick a live team of this organisation, of the right kind."
- CLONE_SOURCE_NOT_PUBLISHED: "Only published or retired versions can be copied."
- CLONE_SAME_SYSTEM: "This version belongs to this system. Use "New draft from a version"."
- CLONE_CROSS_ORGANISATION: "Only a global admin can copy a workflow from another organisation."
- DRAFT_EXISTS: "This system already has a draft. Open it, or discard it first."
- DEFINITION_INVALID: "Fix the errors in the Validation tab, then try again."
- STALE_DRAFT: "Someone else changed this draft. Reload to continue."
- IDEMPOTENCY_TOKEN_USED: "That action was already submitted. Refresh to see the result."
- ENGINE_UNAVAILABLE: "The workflow service is not available. Try again shortly."
- any other 403: "You do not have permission to do this."
- 404: "We cannot find that page."
- unknown code: "Something went wrong. Try again, or contact support with the code below"
The error component takes a code and optional fields (a map of field path to message) and renders the message, the
code and, for field errors, links that move focus to the field.

3. TOASTS (sgds-toast, top right, autohide after 5 seconds, dismissible, variant by meaning, title always set)
- success: "Changes saved." and "Request submitted." and "Signed in as {name}"
- danger: used only when no inline error is possible.
- info: not used for live updates; the "Live" indicator on the request page covers those.
Toasts are announced with aria-live polite. An error that needs a decision stays inline, not in a toast.

4. CONFIRMATION DIALOGS (sgds-modal, size sm, focus trapped, Escape closes, focus returns to the opener)
a. Generic confirm: title as a question in sentence case (for example "Discard this draft?"), one sentence of
   consequence, a secondary button "Cancel" and a primary button with the verb ("Discard draft"). Destructive
   confirms use the danger tone.
b. Notes dialog (used for return, reject, dispute and similar): title names the action and the request number, one
   textarea "Notes" (required, up to 2,000 characters, with a live counter "0 of 2,000"), helper text "Explain what
   the person needs to know.", error "Enter notes of up to 2,000 characters." when blank or too long. Buttons
   "Cancel" and the action verb; the verb button stays aria-disabled with the reason "Notes are required" until
   the notes are valid.
c. Admin reset dialog (request page, for users with the reset permission): title "Reset request
   BG-HRIS-20261108-003?", text "The request goes back to the requester and a new run starts. Approvals after step 1
   are cleared.", textarea "Notes" (required only when Force is on), switch "Force" with helper "Cancel items that
   are not handed over yet. Items already handed over keep going.". Primary button "Reset request" (danger). If the
   mock returns CUSTODY_IN_PROGRESS, show the error copy above plus the list of blocking items, for example
   "SP-02 is with the requester", and keep the dialog open. After a forced reset, the request page shows the line
   "1 item still in custody from a previous approval".
d. Session dialog: none; use the 401 redirect.

5. FORM CONVENTIONS
Labels above inputs (sgds-input, sgds-textarea, sgds-select, sgds-combo-box, sgds-datepicker); required fields end
the label with " (required)" in text, not only an asterisk; hint text under the label; an error replaces the hint
under the field with the field message in danger colour plus an icon; on submit failure focus moves to the first
invalid field and an sgds-alert summarises "Fix the errors below, then try again." with links to each field.
````

---

## Prompt 1: login

**Route:** `/login`. **Who sees it:** everyone who is signed out. **Mirrors:** 219 §12, ST-2 US-2.5.
**Needs:** Prompt 0 (shell, personas), Prompt 0B (alerts).

````
Using the app shell, SGDS styling and mock data already in this file, build the sign-in page at /login.

PURPOSE AND WHO. Everyone who is signed out lands here. Signed-in users who open /login are sent to the dashboard.
In demo mode (a mock flag, on by default in this prototype) the page also offers one-click persona sign-in. With
demo mode off, only email and password are shown and the persona picker does not exist.

LAYOUT (desktop, 1024 px and above). The shell shows the masthead, the main nav with only the brand (no menu items,
no user menu) and the footer. Content: a centred SGDS container, two columns of equal width with a 24 px gap.
Left card: "Sign in". Right card (demo mode only): "Demo personas". Tablet (768 to 1023): same two columns narrower.
Mobile: one column, the persona card below the sign-in card, collapsed into an sgds-accordion that is closed by default.

  +--------------------------+  +----------------------------------+
  | Sign in                  |  | Demo personas                    |
  | Email (required)         |  | Org Aurora (AUR)                 |
  | [__________________]     |  |   Vendor L1                      |
  | Password (required)      |  |     Alice Tan   Bob Lim  ...     |
  | [__________________]     |  |   Vendor L2 ...                  |
  | [Sign in]                |  | Org Sentinel (SEN) ...           |
  +--------------------------+  +----------------------------------+

LEFT CARD (sgds-card)
- Heading MD "Sign in". Body MD intro "Use your work account."
- Field Email: sgds-input type email, label "Email", autocomplete "username", required. Errors: empty "Enter your
  email address."; not an email "Enter an email address in the format name@example.gov".
- Field Password: sgds-input type password, label "Password", autocomplete "off" (sensitive), required. Error empty
  "Enter your password." Include a "Show password" toggle button inside the field (icon button with aria-label).
- Button "Sign in" (sgds-button primary, full width on mobile). While the mock call runs: spinner and "Signing in…",
  button aria-disabled.
- Failure states, shown in an sgds-alert danger above the form with focus moved to it:
  - wrong email or password: "Email or password is incorrect." with code small print "401". The message is the same
    whether the email exists or not.
  - a deactivated (tombstoned) user: the same message as wrong password (no hint).
  - five failed attempts in a row: "Too many sign-in attempts. Try again in 30 minutes." code "429"; the button stays
    aria-disabled with the reason visible under it, even if the next password is correct.
  - session ended: "Your session has ended. Sign in again." (sgds-alert warning) when arriving from a 401 redirect.
- Mock behaviour: any listed person's email with the password "demo-password" signs in; five wrong attempts lock the
  form for the rest of the session; successful sign-in goes to the page in ?next= or to the dashboard.

RIGHT CARD, DEMO PERSONAS (only in demo mode)
- Heading MD "Demo personas". Body SM "Demo mode is on. Pick a persona to sign in at once. These are synthetic people."
- Group by organisation (Heading SM, with the code in a neutral sgds-badge): "Org Aurora (AUR)", "Org Sentinel (SEN)",
  then "Multiple organisations" (Mona Multi), then "Global". Inside each, sub-groups by role: Vendor L1, Vendor L2, Agency Ops, Agency Logistics, PA, Owners and
  delegates, Admins (only those that have members). Each person is a button (sgds-button, outline, size sm) whose label
  reads "AUR · Vendor L1 · Bob Lim" (organisation code · role · name). Where a person has two roles they appear in both
  groups (Ivan Ng under Agency Ops and under Owners and delegates; Emma Yeo under Agency Ops and Owners and delegates
  as "delegate for Carol Sim").
- Clicking a persona signs in immediately, shows the toast "Signed in as {name}", and goes to the dashboard.
- Keyboard: the groups are a list of headings followed by buttons in reading order; arrow keys are not required; Tab
  moves button to button.
- If demo mode is off, this card is not rendered at all and the left card is centred.

STATES. Loading: the persona list is static, no skeleton. Empty: not applicable. Offline or engine down: sign-in
still works; the engine banner may show.

ACCESSIBILITY. One h1 "Sign in to Break-glass". The error alert has role alert. Persona buttons have the visible label
as their accessible name. Do not use colour alone for the lock state.

TEST IDS. login-form__email, login-form__password, login-form__submit, login-form__error, demo-personas__group
(data-organisation), demo-personas__item (data-user-id).
````
