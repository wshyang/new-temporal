# Addendum 220 (part 4) — Figma Make prompts: issuance and returns report, asset detail, admin shell and the seven admin pages

**Date:** 2026-11-08
**Status:** PROPOSED. Documentation only. Tag `[CUSTOM: design handoff]`.
**Parts:** part 1 `addendum-220-figma-make-screen-prompts.md` (how to use, Prompts 0, 0B, 1); part 2 `…-part-2-dashboard-new-request-and-request-shell.md` (Prompts 2 to 4); part 3 `…-part-3-request-graph-actions-and-panels.md` (Prompts 5 to 9); **this part** (Prompts 10 to 19); part 5 `…-part-5-wizard-versions-and-editor-layout.md` (Prompts 20 to 24); part 6 `…-part-6-editor-tabs-simulation-and-dialogs.md` (Prompts 25 to 30).
**Normative source:** addendum 219 §12 (routes), ST-2, ST-3 (US-2.1 to US-2.6, US-3.1, US-3.4, US-3.5), ST-8 US-8.6 (report and asset history), part 6 §27.5.2 (quarantine), §38.4 (error copy). 219 wins on any conflict.
**Authored copy.** 219 specifies the admin pages in one table row (§12: "Organisations, clusters/teams (level up/down), memberships, users (tombstone), roles/grants, systems, assets") and in acceptance criteria. Column names, button labels and empty-state copy below are therefore authored in SGDS style from those criteria; replace them with server text when it exists. The validation rules and conflict outcomes are taken from the acceptance criteria.

---

## Prompt 10: issuance and returns report

**Route:** `/reports/issuance-returns`. **Mirrors:** 219 §12, US-8.6 (AC-8.6.4, AC-8.6.5, AC-8.6.7). **Needs:** Prompts 0, 0B.

````
Using the app shell, SGDS styling and mock data already in this file, build the page /reports/issuance-returns
("Issuance and returns report").

PURPOSE AND WHO. A cross-request table of every credential item that was issued, with its return status, so owners and
the investigating team can chase items that are still out. Seen by: owners and their backups and delegates (Carol Sim, Dan
Teo, Ivan Ng, Emma Yeo as delegate), members of the investigate team (Claire Wong, Emma Yeo, Ivan Ng, Frank Chua),
organisation admins (Amanda Scoped) and global admins (Greg Global). Bob Lim, Priya Nair and every other vendor or PA
persona get the 403 page of Prompt 0B (the nav item "Reports" is hidden for them too). Data is scoped to the person's
organisations: for an organisation admin, the organisations their grants name (Amanda: AUR; Mona Multi: AUR and SEN, 219 D-38); for owners and
investigators, the organisations of their systems; a global admin sees all organisations. A SEN persona asking for an AUR system sees the 404 page.

LAYOUT (desktop 1280 wide)
- sgds-container. Breadcrumb "Dashboard / Reports / Issuance and returns". Heading XL "Issuance and returns". Body MD
  under it: "Every item issued from a break-glass request, and whether it has come back."
- Filter bar (sgds-card, one row; wraps on tablet; becomes a collapsible "Filters" accordion on mobile):
  - "From" (sgds-datepicker, optional) and "To" (sgds-datepicker, optional). Error when To is before From: "End date must be on or after the start date."
  - "System" (sgds-select): "All systems" then the systems the person may see (for Carol Sim: HRIS, FINDB, OPSTOOL).
  - "Outstanding only" (sgds-switch, off by default). Helper: "Shows items issued and not yet returned, written off or replaced."
  - Buttons: "Apply filters" (primary), "Clear filters" (ghost; enabled only when a filter is set), and "Download CSV" (outline, icon).
  - The URL mirrors the filters (?from=&to=&systemId=&outstanding=true). Applying a filter shows a skeleton table for 400 ms.
- Summary tiles above the table (six sgds-card tiles; label above the number): "Items issued" 21, "Outstanding" 7, "Returned" 9,
  "Overdue" 3, "Written off" 3, "Replaced" 2. They follow the filters.
- Table (sgds-data-table, sortable headers, sticky header, caption "Issued items"). Columns in this order: Overdue (badge),
  Request (link to /requests/[id]), System, Asset (link to /assets/[id]), Lane origin (Initial, Replacement, Supplementary),
  Issued to, Handed over by, Issued at, Return due, Returned at, Verified by, Restocked at, Return condition (Good, Damaged, or "-"),
  Outcome. Overdue rows are sorted first, then by Return due ascending; the sort can be changed by the headers and the
  default is shown with an aria-sort. The Overdue cell shows a red badge "Overdue 2h" or "Overdue 3 d", else blank. Outcome
  values are sgds-badges with a word: "Outstanding" (blue), "Returned" (green), "Returned damaged" (red-brown), "Replaced"
  (teal), "Written off" (grey).
- Pagination (sgds-pagination) at the bottom with a page-size select (10, 25, 50; default 25) and the text "Showing 1 to 10 of 21 items".
- Mobile (320 to 767): the table becomes stacked cards, each with Asset as the card title, Request and Outcome in the header
  row, and the remaining fields as a description list. Filters collapse as above.

MOCK ROWS (use at least these 12, from the RQ fixtures; times in SGT)
1. BG-HRIS-20261108-001 · HRIS · HT-01 · Initial · Bob Lim · handed over by Grace Neo · issued 8 Nov 9:10am · due 8 Nov 5:00pm · not returned · Outstanding · Overdue 3h
2. BG-HRIS-20261108-001 · HRIS · SP-01 · Initial · Bob Lim · Grace Neo · 8 Nov 9:10am · due 8 Nov 5:00pm · Outstanding · Overdue 3h
3. BG-HRIS-20261108-001 · HRIS · LT-01 · Initial · Bob Lim · Grace Neo · 8 Nov 9:10am · due 9 Nov 5:00pm · Outstanding
4. BG-HRIS-20261108-003 · HRIS · SP-01 · Initial · Bob Lim · Grace Neo · returned 8 Nov 4:05pm · verified by Claire Wong · condition Damaged · Returned damaged
5. BG-HRIS-20261108-003 · HRIS · HT-02 · Replacement (replaces HT-01) · Bob Lim · Grace Neo · returned 8 Nov 4:10pm · verified by Claire Wong · restocked by Grace Neo 4:15pm · Good · Returned
6. BG-HRIS-20261108-003 · HRIS · SP-03 · Supplementary (supplements SP-01) · Bob Lim · Leo Tan · returned 8 Nov 4:12pm · Good · Returned
7. BG-HRIS-20261108-003 · HRIS · LT-01 · Initial · issued · Written off
8. BG-HRIS-20261108-003 · HRIS · HT-01 · Initial · never issued (issued at "-") · Replaced
9. BG-HRIS-20261108-005 · HRIS · SP-01 · Initial · Bob Lim · Grace Neo · returned 8 Nov 4:05pm · Damaged · Returned damaged (problem open)
10. BG-FINDB-20261108-002 · FINDB · HSM-01 · Initial · Betty Ong · Grace Neo · issued 7 Nov 2:30pm · due 7 Nov 6:00pm · Outstanding · Overdue 1 d
11. BG-OPSTOOL-20261107-001 · OPSTOOL · KB-01 · Initial · Ivan Ng · Leo Tan · returned 7 Nov 3:00pm · Good · Returned
12. BG-HRIS-20261107-004 · HRIS · LT-02 · Initial · Alice Tan · Grace Neo · returned 7 Nov 6:00pm · Good · Returned
With "Outstanding only" on, show only rows 1, 2, 3, 10 (sorted overdue first) and the tiles recompute.

PERSONA CHECKS. Carol Sim: sees HRIS, FINDB and OPSTOOL rows (AUR). Amanda Scoped: AUR only. Greg Global: AUR and SEN rows (add 2 SEN
rows for PORTAL: HT-S1 Initial, Gina Ho, Outstanding). Hank Low (SEN agency, not an investigator): 403 page. Gina Ho: 403 page.
A SEN persona with ?systemId=HRIS shows the 404 page.

STATES
- Loading: skeleton tiles and ten skeleton rows. Error: sgds-alert danger "Could not load the report. Try again." with a "Try again" button.
- Empty (no rows after filters): illustration-free empty state with the text "No items match these filters." and the button "Clear filters".
- Empty (nothing ever issued): "Nothing has been issued yet. Items appear here after a requester confirms receipt."
- CSV: clicking "Download CSV" shows a button spinner for 600 ms then the toast "CSV downloaded." The CSV has the same rows and columns as the
  table for the full filtered set (not only the page); say so in a Body SM helper under the button: "The file contains every row that matches the filters."
- Engine banner does not affect this page (read-only).

ACCESSIBILITY. Landmark main with the h1; filters in a form with a visible label per field; the table has a caption and sortable headers expose
aria-sort; overdue is shown by the word as well as the colour; pagination announces the page change politely.

TEST IDS. report-filter__from, report-filter__to, report-filter__system, report-filter__outstanding, report-filter__apply, report-filter__csv,
report-table__row (data-asset, data-request), report-table__overdue, report-summary__tile (data-metric).
````

---

## Prompt 11: asset detail

**Route:** `/assets/[id]`. **Mirrors:** 219 §12, AC-8.6.4, part 6 §27.5.2 (quarantine). **Needs:** Prompts 0, 0B, 10.

````
Using the app shell, SGDS styling and mock data already in this file, build the page /assets/[id] ("Asset detail") for the assets
SP-01, HT-02, LT-01, HT-01 and KB-01.

PURPOSE AND WHO. Shows one physical credential asset and every time it has been issued and returned, across all requests, in time order.
Anyone who can see at least one request that touched the asset sees the page (requesters, reviewers, owners, investigators, admins) scoped to
their organisations (for an organisation admin, the organisations their grants name, 219 D-38); an asset of another organisation, or an unknown id, shows the 404 page of Prompt 0B (same body for both).

LAYOUT (desktop)
- Breadcrumb "Dashboard / Assets / SP-01". Heading XL "SP-01" with the status badge next to it. Body MD "Secure phone · HRIS · Org Aurora".
- Status badge values (icon plus word): "In stock" (green), "In use" (blue; Body SM "Held by BG-HRIS-20261108-001" with a link), "Quarantined"
  (red-brown; icon and word), "Retired" (grey; word "Retired" with the reason "written_off" or "lost_replaced").
- Left column, 8 of 12: "Issue and return history" card. Right column, 4 of 12: "Details" card and, when quarantined, an sgds-alert.
- Details card (sgds-description-list): Asset code, Type (device, token or password), Hostname or description, Credential name, System, Organisation,
  Created (date), Status, Held by (request link or "-"), Quarantined at (date and time, or "-"), Quarantine reason ("damaged_on_return" or "-"),
  Retired at, Retired reason.
- Quarantine alert (sgds-alert warning, shown only when quarantined): "This asset is quarantined because it came back damaged. It cannot be requested, used as a substitute or put back in stock." Body SM: "There is no release path yet."
- Retired alert (sgds-alert neutral): "This asset is retired and does not appear in pickers."
- History card: sgds-table, caption "Issues and returns of SP-01", columns exactly: Request (link), Lane origin, Issued to, Handed over by, Issued at, Return due,
  Returned at, Verified by, Restocked at, Return condition, Quarantined at, Outcome (badge as in Prompt 10). Time order is oldest first with the
  newest row highlighted by a left accent bar; a toggle "Newest first" flips the order. Issuing the asset again adds a new row and never edits the old one.
- Tablet: columns collapse to one column; the Details card goes above the history. Mobile: the history becomes stacked cards.

MOCK
- SP-01 (secure phone, HRIS): status Quarantined (damaged_on_return, quarantined 8 Nov 2026 4:12pm). History: 6 Nov BG-HRIS-20261106-002 issued to Alice Tan, returned
  good, restocked (Returned); 8 Nov BG-HRIS-20261108-003 issued to Bob Lim, returned damaged, verified by Claire Wong, quarantined at 4:12pm (Returned damaged).
- HT-02 (hardware token, HRIS): In use, held by BG-HRIS-20261108-003 (replacement lane); history has two rows (an earlier return on 5 Nov and the current issuance, Outstanding).
- LT-01 (laptop, HRIS): Retired (written_off, 8 Nov); history shows one row, Written off, issuance kept.
- HT-01 (hardware token, HRIS): Retired (lost_replaced); history shows one row, never issued ("-" in Issued at), Replaced.
- KB-01 (sealed password envelope, OPSTOOL): In stock; history one row, Returned.

STATES. Loading: skeleton heading and table. Error: sgds-alert danger with "Try again". Empty history: "This asset has not been issued yet." Not found: the 404 page.

ACCESSIBILITY. The status is a word plus an icon, not colour alone; the history table has a caption; links say where they go ("Open BG-HRIS-20261108-003").

TEST IDS. asset-detail__status, asset-detail__quarantine-alert, asset-history__row (data-request), asset-history__order-toggle.
````

---

## Prompt 12: admin shell and sidebar

**Route:** `/admin/*`. **Mirrors:** 219 §12, US-2.1 to US-2.6, §14 role table. **Needs:** Prompts 0, 0B. Prompts 13 to 19 fill the content region (and Prompt 30 fills "Workflow templates").

````
Using the app shell, SGDS styling and mock data already in this file, build the shared Admin layout used by every /admin page.

PURPOSE AND WHO. Administration of the organisation structure and system registry. Who sees what (extend the persona drawer so these roles can be tried):
- Greg Global (global admin, full_admin): every page, all organisations. The page header shows an organisation scope select "All organisations".
- Amanda Scoped (org_admin for AUR only): every page, AUR data only. With one organisation in scope, the scope shows the fixed chip "Org Aurora" and cannot be changed.
- Mona Multi (org_admin for AUR and SEN): every page, AUR and SEN data only. The scope is a select (admin-scope__select) with "All my organisations" (default),
  "Org Aurora" and "Org Sentinel"; it never lists an organisation she was not granted. Organisation and global powers she lacks (create or delete an organisation,
  Roles and grants changes) are not rendered.
- An organisation admin's scope is exactly the organisations their grants name (219 D-38). It is never "all organisations"; an organisation added later is not in it.
- Alice Tan (team admin of Alpha Engineers L1): sees only "Memberships", and within it only her own team.
- Everyone else: the Admin menu is hidden and /admin/* shows the 403 page of Prompt 0B.
An object of another organisation, requested by id, shows the 404 page (the same page as an unknown id).

LAYOUT (desktop 1280)
- App shell of Prompt 0, then a two-column area with .sgds-container-sidebar: left sgds-sidenav (width 256), right the content region.
- Sidenav title "Administration". Items in this order, each with an icon: "Organisations", "Clusters and teams", "Memberships", "Users", "Roles and grants",
  "Systems", "Workflow templates" (route /admin/templates), "Assets". The current item is highlighted with aria-current="page". Items the persona may not use are not rendered.
- Content region: breadcrumb "Dashboard / Admin / {page}", Heading XL page title, Body MD one-line purpose, then the page body.
- Tablet (768 to 1023): the sidenav collapses into an icon rail with tooltips; Mobile: a "Menu" button above the content opens the sidenav in an sgds-drawer.
- Every admin page uses the same pattern: toolbar (search, filters, primary create button on the right), sgds-data-table, row actions in an sgds-overflow-menu,
  create and edit in an sgds-modal (md) or sgds-drawer (480 px right) , and confirmation dialogs from Prompt 0B.

SHARED RULES FOR ALL ADMIN PAGES
- Soft delete: "Delete" actions say "Delete" in the menu and "Delete {name}?" in the dialog, with the body "This hides {name} from lists. History keeps showing it." Success toast "{name} deleted."
- Conflicts show the server message in an sgds-alert at the top of the dialog (never only a toast), with all field values kept.
- Field-level 422 errors appear under the field, in the form "fields.shortCode" style mapped to the label.
- Empty state per page: a short sentence plus the create button.
- Loading: skeleton table of 8 rows. Error: sgds-alert danger "Could not load {thing}. Try again." with "Try again".
- Admin CRUD does not depend on the workflow engine, so admin buttons stay enabled when the engine banner is shown.
- Organisation scope appears as a required select "Organisation" in every create form: for Greg it lists every organisation; for Mona it lists only Org Aurora and
  Org Sentinel (preselected from the page scope when that is a single organisation); for Amanda, whose scope has one organisation, it is fixed and shown as read-only text.
- Row counts show "Showing 1 to 10 of 24 {things}". Page size 10, 25, 50.

STATES FOR THE SHELL. 403 page for personas without access; 404 page for foreign objects; a skeleton sidenav while loading. The user can bookmark any page.

ACCESSIBILITY. The sidenav is a nav with aria-label "Administration"; the content region is main; skip link "Skip to admin content"; dialogs trap focus and return it.

TEST IDS. admin-nav__item (data-key), admin-scope__select, admin-page__create, admin-table__row (data-id), admin-row__menu, admin-dialog__submit, admin-dialog__cancel, admin-dialog__error.
````

---

## Prompt 13: admin, organisations

**Route:** `/admin/organisations`. **Mirrors:** US-2.1 (AC-2.1.1 to AC-2.1.3). **Needs:** Prompt 12.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/organisations inside the Admin layout of Prompt 12.

WHO. Greg Global manages all organisations (create, edit, delete). An organisation admin sees only the organisations their grants name (219 D-38): Amanda Scoped sees
Org Aurora; Mona Multi sees Org Aurora and Org Sentinel. They can read but not create or delete (the "Create organisation" button is not rendered for them, and "Delete" is absent
from their row menu; "Edit" is allowed for the name). An organisation that Greg adds later does not appear for them until Greg grants it.

TABLE (sgds-data-table, caption "Organisations"): columns Short code, Name, Systems (count, link to /admin/systems filtered), Users (count), Created, Actions (overflow menu: Edit, Delete).
Mock rows: AUR · Org Aurora · 3 systems · 16 users · 1 Aug 2026; SEN · Org Sentinel · 1 system · 4 users · 1 Aug 2026 (Mona Multi counts in both). Search by name or code ("Search organisations").

CREATE / EDIT (modal, md). Title "Create organisation" / "Edit Org Aurora".
- "Short code" (sgds-input, required, uppercased as you type, up to 10 characters; hint "Letters and numbers only, up to 10 characters, for example AUR."; read-only on edit).
  Errors: "Enter a short code." (empty); "Use 1 to 10 letters or numbers." (pattern fails); "That short code is already used." (duplicate).
- "Name" (sgds-input, required, up to 200). Errors: "Enter a name." and "That name is already used." (the check is case-insensitive).
- Buttons: "Cancel", "Create organisation" or "Save changes". Primary is aria-disabled until valid, with the reason under the form "Enter a short code and a name".
- Success toasts: "Organisation created." and "Changes saved."

DELETE. Dialog "Delete Org Aurora?" with the generic body. Blocked with an sgds-alert danger in the dialog when it still has live systems:
"Org Aurora still has 3 live systems. Delete or move them first." The primary "Delete organisation" is shown disabled once the alert is shown.

STATES. Empty (only for a fresh install): "No organisations yet. Create the first one." Loading, error as in Prompt 12.

TEST IDS. org-form__short-code, org-form__name, org-form__submit, admin-table__row (data-id), admin-row__menu.
````

---

## Prompt 14: admin, clusters and teams

**Route:** `/admin/clusters`. **Mirrors:** US-2.2 (AC-2.2.1 to AC-2.2.3), §12 "level up/down". **Needs:** Prompt 12.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/clusters ("Clusters and teams") inside the Admin layout.

WHO. Greg Global and the organisation admins, for the organisations their grants name (Amanda Scoped: AUR; Mona Multi: AUR and SEN). Team admins do not see this page.

LAYOUT. A master-detail view. Left (5 of 12): an sgds-card list of clusters grouped by type with an sgds-tab-group "All, Vendor, Agency, PA"; each cluster row shows name, a type badge
(Vendor, Agency, PA, with the word), and "{n} teams". Right (7 of 12): the selected cluster's teams as a vertical stack ordered by level ascending, each team a card:
"L1 Alpha Engineers" with member count "2 members" and a team admin chip, and the buttons "Level up" (arrow up icon), "Level down" (arrow down icon), "Rename", "Delete".
Under the stack: "Add team".

Mock (organisation AUR): Alpha Pte (vendor): L1 Alpha Engineers (Alice Tan as team admin, Bob Lim), L2 Alpha Leads (Aaron Koh). Beta Pte (vendor): L1 Beta Engineers, L2 Beta Leads.
AUR Ops (agency): L1 Ops Reviewers (Claire Wong, Emma Yeo, Ivan Ng), L2 Ops Leads (Frank Chua). AUR Logistics (agency): L1 Asset Store (Grace Neo, Leo Tan). AUR PA (pa): L1 Owner Support (Priya Nair).
SEN (visible to Greg only): Gamma Services (vendor) L1 (Gina Ho); SEN Ops (agency) L1 (Hank Low).

CREATE CLUSTER (modal). Fields: "Name" (required, up to 200; error "Enter a name."), "Type" (sgds-radio: Vendor, Agency, PA; required; error "Choose a type."), "Organisation" (Greg only).
Buttons "Cancel" and "Create cluster". Changing the type of a cluster that already has teams is not offered: the Type field is read-only on edit with the helper
"The type cannot change while the cluster has teams." (server 409 shown as the same text).

ADD OR EDIT TEAM (modal). "Name" (required), "Level" (sgds-select 1 to 5; required). Errors: "Choose a level from 1 to 5." (for 0 or 6) and, on conflict, the sgds-alert
"A team already exists at level 2 in Alpha Pte." The level select marks levels in use as "(in use)" and disables them.

LEVEL UP / DOWN. "Level up" moves the team to a higher level number (L1 to L2); "Level down" the other way. If the next level is taken the button is aria-disabled with the reason
"Level 2 is already used by Alpha Leads." and no swap happens. Level 1 has no "Level down" and level 5 has no "Level up", shown disabled with "Already at the lowest level." and "Already at the highest level."
Moving a team shows the toast "Alpha Engineers moved to level 2."

DELETE TEAM. Generic dialog; blocked when members exist: "Remove the 2 members first." (alert in the dialog). DELETE CLUSTER blocked while it has teams: "Delete its teams first."

STATES. Empty cluster: "No teams yet. Add the first team." Empty list: "No clusters yet. Create one." Loading and error as in Prompt 12.

TEST IDS. cluster-list__item (data-id), team-card__level (data-level), team-card__up, team-card__down, team-form__level, team-form__submit.
````

---

## Prompt 15: admin, memberships

**Route:** `/admin/memberships`. **Mirrors:** US-2.2 AC-2.2.2 and AC-2.2.3, US-2.3 (AC-2.3.1 to AC-2.3.3). **Needs:** Prompt 12.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/memberships inside the Admin layout.

WHO AND SCOPE. Greg Global manages every team; Amanda Scoped and Mona Multi manage every team of the organisations their grants name (AUR; AUR and SEN). Alice Tan (team admin) manages only Alpha Engineers L1: she sees a single team, with no team picker and no
rename, delete or level actions; the page heading becomes "Memberships: Alpha Engineers L1". Aaron Koh (member of Alpha Leads L2, not a team admin) and all others get the 403 page.

LAYOUT. Toolbar: "Team" (sgds-combo-box listing "Alpha Pte / Alpha Engineers L1" and so on; for Alice it is fixed), "Search members", and the primary button "Add member".
Table (caption "Members of Alpha Engineers L1"): Name, Email, Affiliation (Vendor or Agency badge), Team role ("Member" or "Team admin"), Added, Actions (Promote to team admin / Demote to member / Remove).
Mock: Alice Tan · alice.tan@aur.example · Vendor · Team admin; Bob Lim · bob.lim@aur.example · Vendor · Member.

ADD MEMBER (modal). "User" (sgds-combo-box, required; lists active users whose affiliation matches the cluster type: vendor for vendor clusters, agency for agency and PA), "Team role" (sgds-radio: Member, Team admin; default Member).
Rules: the first member added to an empty team becomes team admin automatically (a Body SM line: "The first member of an empty team becomes team admin.").
Errors as sgds-alert in the dialog: "Bob Lim is already at level 1 in Alpha Pte." (user already in another team of the same cluster, 409); "A vendor cannot join an agency team." and
"An agency user cannot join a vendor team." (422). Success toast "Bob Lim added to Alpha Engineers L1."

REMOVE / DEMOTE. Remove opens the generic dialog "Remove Bob Lim from Alpha Engineers L1?" with the body "Bob will no longer be able to review steps for this team." Success toast "Bob Lim removed."
The sole team admin cannot demote or remove themselves: for Alice the actions are aria-disabled with the reason "You are the only team admin. Promote someone else first." and the server message is the same.
Alice attempting to touch Alpha Leads L2 (demo control "Try another team" in the persona drawer) shows the 403 page.

STATES. Empty team: "This team has no members yet. Add the first member." Loading, error as in Prompt 12.

TEST IDS. membership-form__user, membership-form__role, membership-table__row (data-user), membership-row__promote, membership-row__demote, membership-row__remove.
````

---

## Prompt 16: admin, users

**Route:** `/admin/users`. **Mirrors:** US-2.4 (tombstone), US-2.5, D-1 identity. **Needs:** Prompt 12.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/users inside the Admin layout.

WHO. Greg Global (all users), Amanda Scoped (users of Org Aurora only) and Mona Multi (users of Org Aurora or Org Sentinel). All three may use "Tombstone"; an
organisation admin may tombstone a person only when every organisation that person belongs to is in the admin's scope (219 D-38): Amanda can tombstone Bob Lim but
not Mona Multi; Mona can tombstone Bob Lim and Gina Ho.

TABLE (caption "Users"): Name, Email, Organisation, Affiliation (Vendor or Agency badge), Teams (chips such as "Alpha Engineers L1"), Admin role (chip: "Global admin", "Org admin · AUR", "Org admin · AUR, SEN", or "-"), Status
(badges: Active, Pending, Deactivated), Actions (overflow: Edit, Tombstone, Reset demo session).
Search ("Search by name or email"), filters Organisation, Affiliation, Status. Mock: the 20 people of the shared data, plus one deactivated person "Former Staff (deactivated)" with email removed.

CREATE / EDIT USER (drawer). Fields "Full name" (required), "Email" (required, valid, unique among live users; error "That email is already used by an active user."), "Affiliation" (radio Vendor, Agency), "Organisations" (Greg
only; multi-select of memberships). Removing an organisation from a person who is an organisation admin for it shows the warning "Mona Multi is an organisation admin for
Org Sentinel. Removing the membership also revokes that admin role." (219 D-38). Password is not set here; text "The user signs in through the identity provider. In demo mode, personas are listed on the login page."

TOMBSTONE (confirmation with notes, Prompt 0B generic notes dialog variant). Title "Deactivate Bob Lim?". Body: "Bob Lim will not be able to sign in. His past activity keeps showing his name with (deactivated).
Open requests that he submitted and that are not yet approved will be rejected automatically." An info list shows "2 open requests would be rejected" with links. Primary "Deactivate user" (danger).
Blocked cases show an sgds-alert danger and the primary disabled: "You cannot deactivate yourself." / "You cannot deactivate the last global admin." / "System and protected users cannot be deactivated." /
"This person also belongs to an organisation you do not administer." (for example Amanda opening the dialog for Mona Multi)
Success toast "Bob Lim deactivated." After this the row shows "Deactivated" and an email that can be reused.

STATES. Empty search: "No users match." Loading and error as in Prompt 12.

TEST IDS. user-table__row (data-user), user-row__tombstone, user-form__email, user-tombstone__confirm.
````

---

## Prompt 17: admin, roles and grants

**Route:** `/admin/roles`. **Mirrors:** US-2.6 (AC-2.6.1 to AC-2.6.7), D-38, §12 data model `admin_roles`, `user_admin_role_grants`. **Needs:** Prompt 12.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/roles ("Roles and grants") inside the Admin layout.

WHO. Greg Global only grants and revokes. Amanda Scoped and Mona Multi can open the page read-only (their own grants and the grants of the organisations in their
scope) with no grant or revoke buttons.

LAYOUT. Two sections.
1. "Admin roles" (sgds-card, read-only table, caption "Admin roles"): Key, Name, Capabilities. Rows: full_admin · Full administrator · "Every capability";
   org_admin · Organisation administrator · "org_structure, systems_assets, users_org, request_admin_reset". Each capability is an sgds-badge with its plain meaning in a tooltip.
2. "Grants" (sgds-data-table, caption "Admin grants"): one row per user, role and organisation. Columns User, Role, Organisation ("All organisations" only for
   full_admin), Granted by, Granted at, Status (Active, or Revoked with the date and reason), Actions (Revoke). A filter "User" groups a person's rows.
   Mock: Greg Global · full_admin · All organisations · system · 1 Aug 2026 · Active; Amanda Scoped · org_admin · AUR · Greg Global · 2 Aug 2026 · Active;
   Mona Multi · org_admin · AUR · Greg Global · 3 Aug 2026 · Active; Mona Multi · org_admin · SEN · Greg Global · 3 Aug 2026 · Active.

GRANT (modal). Title "Grant admin role". Fields: "User" (sgds-combo-box; only active agency users appear), "Role" (select full_admin, org_admin), "Organisations" (sgds-combo-box with multi-select;
required for org_admin, at least one; hidden for full_admin). The options are only the organisations the chosen user is a member of, with hint "An organisation admin can
only administer organisations they belong to. Add a membership first in Users." Organisations the user already administers are shown checked and disabled with "(already granted)".
The primary "Grant role" is aria-disabled until a user, a role and, for org_admin, at least one organisation are chosen.
Errors in an sgds-alert: "Admin roles can only be granted to active agency users." (a vendor, pending, system or protected user); "Org admin can only be granted for
organisations the user belongs to." (GRANT_NOT_MEMBER); "Choose at least one organisation for an org admin." (GRANT_SCOPE_INVALID). Nothing is granted when any
organisation fails. Success toast "Role granted for 2 organisations." (or "Role granted." for full_admin).

REVOKE. Per row, so one organisation at a time. Generic dialog "Revoke org_admin for Org Sentinel from Mona Multi?" with the body "Mona keeps her other admin
organisations. Her next admin action in Org Sentinel will be refused." (for a person's last org_admin row: "Amanda's next admin action will be refused.").
Success toast "Role revoked." The revoked row shows "Revoked 8 Nov 2026"; a row revoked by a membership removal shows "Revoked 8 Nov 2026 · membership removed".

STATES. Empty grants: "No grants yet." Loading and error as in Prompt 12.

TEST IDS. grant-form__user, grant-form__role, grant-form__organisations, grant-form__submit, grant-table__row (data-user, data-organisation), grant-row__revoke.
````

---

## Prompt 18: admin, systems (list, owners, delegates, PA)

**Route:** `/admin/systems`. **Mirrors:** US-3.1, US-3.5, §12 systems row, part 4 §24.1 (links to the workflow screens). **Needs:** Prompt 12. Prompts 20 to 22 build the workflow screens that this page links to.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/systems ("Systems") and the system detail drawer inside the Admin layout.

WHO. Greg Global (all), Amanda Scoped (Org Aurora) and Mona Multi (Org Aurora and Org Sentinel). Others see the 403 page.

LIST. Toolbar: search, "Organisation" filter (Greg, and Mona over her two organisations), and the primary button "Create system" which opens the wizard at /admin/systems/new (Prompt 20). Table (caption "Systems"): Code, Name, Organisation, Primary owner,
Backup owner, Vendor clusters (chips in order), Workflow (chip "v1 published" or "v2 draft" plus "v1 published"), Assets (count), Actions (overflow: Open, Edit details, Open workflow, Delete).
Mock: HRIS · HR Information System · AUR · Carol Sim · Dan Teo · Alpha Pte · v1 published · 7 assets; FINDB · Finance Database · AUR · Dan Teo · Carol Sim · Beta Pte · v1 published · 1; OPSTOOL · Operations Tool ·
AUR · Ivan Ng · Carol Sim · (none) · v1 published · 1; PORTAL · Citizen Portal · SEN · Sam Lee · (none) · Gamma Services · v1 published · 1 (visible to Greg and Mona, not Amanda).
Row click opens the system drawer; "Open workflow" goes to /admin/systems/[id]/workflow (Prompt 22).

SYSTEM DRAWER (right, 560 px; sgds-tab-group "Details", "Owners and delegates", "Approval PA", "Workflow"). Header: code, name, organisation chip.
1. Details (form): "Short code" (read-only after creation; creation rule: 2 to 10 letters or numbers, unique in the organisation; error "Use 2 to 10 letters or numbers." and "That code is already used in this organisation."),
   "Name" (required), "Description" (up to 500, counter), "Agency cluster" (sgds-select of agency clusters of the organisation; required). Buttons "Cancel" and "Save changes".
2. Owners and delegates:
   - "Primary owner" and "Backup owner" (sgds-combo-box each; active agency users of the organisation). Errors: "The primary and backup owners must be different people." and "Choose an active agency user of this organisation."
   - Delegates table: Delegate, For owner, Active from, Active until, Status ("Active now", "Scheduled", "Ended"), Actions (End now, Edit). Button "Add delegate" opens a modal with "Owner" (select primary or backup),
     "Delegate" (combo-box), "From" and "Until" (date-time pickers). Errors: "The delegate must be an active agency user of this organisation." and "The end must be after the start."
     Mock: Emma Yeo for Carol Sim, now to +2 days, "Active now". Success toast "Delegate added." The Body SM note: "A delegate can make the owner decision only inside this window."
3. Approval PA: a table of owner to PA team mapping: Owner, PA team. Mock: Carol Sim to Owner Support L1; Dan Teo to Owner Support L1. Edit via select of PA-type teams of the organisation.
   Error: "Choose a team from a PA cluster of this organisation." The note "These people see the request but cannot act on it."
4. Workflow: a read-only summary "Current: v1 published 8 Nov 2026 by Amanda Scoped. Draft: none." and the buttons "Open versions" and "Open editor", both going to the Prompt 22 and 23 screens. Chip
   "Chain: Alpha Engineers L1, Alpha Leads L2, Ops Reviewers L1, owner" rendered from the published workflow (read-only).

DELETE SYSTEM. Generic dialog; blocked while requests are in progress: "HRIS has requests in progress. Wait for them to finish." Success toast "HRIS deleted."

STATES. Empty: "No systems yet. Create the first one." Loading, error as in Prompt 12.

TEST IDS. system-table__row (data-code), system-drawer__tab (data-tab), system-form__code, system-form__owner-primary, system-form__owner-backup, delegate-form__submit, pa-mapping__row (data-owner).
````

---

## Prompt 19: admin, assets

**Route:** `/admin/assets`. **Mirrors:** US-3.4 (AC-3.4.1, AC-3.4.2), part 6 §27.5.2 (quarantine). **Needs:** Prompts 12, 11.

````
Using the app shell, SGDS styling and mock data already in this file, build /admin/assets ("Assets") inside the Admin layout. Assets are manual physical credentials.

WHO. Greg Global and the organisation admins, for the organisations their grants name (Amanda Scoped: AUR; Mona Multi: AUR and SEN). Others see the 403 page.

TABLE (caption "Assets"): Code, Hostname or description, Credential name, Type (Device, Token, Password badges), System, Status (In stock, In use, Quarantined, Retired; with the word), Held by (request link), Actions
(overflow: Open, Edit, Retire, Delete). Toolbar: search, filters System, Type, Status; primary button "Add asset".
Mock rows: SP-01 Quarantined; SP-02 In stock; SP-03 In stock; HT-01 Retired (lost_replaced); HT-02 In use (BG-HRIS-20261108-003); LT-01 Retired (written_off); LT-02 In stock; HSM-01 In stock; KB-01 In stock; HT-S1 In stock (Greg and Mona, not Amanda).

ADD OR EDIT ASSET (drawer). Fields: "System" (select; required; read-only on edit), "Asset code" (required, up to 20), "Hostname or description" (required, up to 200), "Credential name" (required, up to 100),
"Credential type" (radio Device, Token, Password; required; on edit it is read-only with the helper "The type cannot change after the asset is created."). Errors:
"That hostname and credential name already exist in this system." (409 duplicate among live assets) and "Choose a credential type."
Buttons "Cancel" and "Add asset" / "Save changes". Toasts "Asset added." and "Changes saved."

RETIRE. Dialog "Retire SP-02?" with a required "Reason" select (Written off, Lost and replaced, Other) and optional notes; body "A retired asset does not appear in pickers. Submitting a request with it is refused." Toast "SP-02 retired."
Blocked when the asset is held: "SP-02 is in use by BG-HRIS-20261108-003. It cannot be retired until the request releases it." A quarantined asset can still be retired, so keep "Retire" enabled for it.
Quarantined rows show a red-brown badge "Quarantined" and a Body SM line "Damaged on return. Not restocked." There is no release action; do not draw one.

DELETE. Generic dialog; blocked while held. Success toast "Asset deleted."

STATES. Empty: "No assets yet. Add the first asset." Loading and error as in Prompt 12.

TEST IDS. asset-table__row (data-code), asset-form__code, asset-form__hostname, asset-form__credential-name, asset-form__type, asset-row__retire.
````
