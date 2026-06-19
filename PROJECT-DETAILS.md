# SC-E001 — Add Property Implementations Tab (SSP v2)

**Jira:** [DEV-304238](https://entrata.atlassian.net/browse/DEV-304238)  
**Prototype:** https://brandonbayles-hub.github.io/implementation-workbench-preview/  
**Loom walkthrough:** https://www.loom.com/share/cf081d16544a4a3fb1de01aa7f5b0012  
**Submitted by:** Brandon Bayles  
**Domain validated by:** Not yet validated — pending EM review  
**Release:** Rapid July 7, 2026 / Standard GA October 6, 2026  
**Target repo:** `core` | Retrofit  

---

## 1. Summary

This epic adds a dedicated **Setup > Property Implementations** tab to the Entrata nav, surfaces the existing Add New Contract & Property (SSP) workflow there with a redesigned dark-blue modal, extends Step 2 with a **Migration Area** (Migration Type + Configuration Type dropdowns + conditional date/name fields), sends an automated Slack notification to the client's `#cproject-{name}` channel on property creation, and adds a **View Property** button post-create.

The codebase footprint spans the `Setup/Properties/Property/AddOnPropertyContract/` subtree (controller, Smarty templates, JS, LESS), the `AssignApplicationModules.php` nav registration, and a new `CPropertyService::sendSlackProjectChannelNotification()` method.

---

## 2. Why

Implementation consultants navigate to Setup > Properties > [Property] > Add On Property Contract every time they add a property — a buried path in a high-frequency workflow. Surfacing the same entry point under Setup > Property Implementations reduces navigation overhead. The Migration Area fields (Configuration Type, conditional dates) capture structured implementation metadata at the moment of property creation, eliminating a manual data-collection step during kick-off calls. The Slack notification closes the loop between property creation and ProServ team awareness without a manual post — a step that currently takes 5 minutes per property and is easy to miss.

---

## 3. What Changed in the Codebase

**New files**
- `Applications/Entrata/Application/Setup/Properties/Property/AddOnPropertyContract/` — no new files; existing controller extended
- DB migration script (to be created): adds `configuration_type` (varchar), `migration_start_date` (date), `close_date` (date), `priority_name` (varchar), `migration_completion_date` (date) to the property setup migration table — all nullable

**Modified files**
- `Applications/Entrata/Application/AssignApplicationModules.php` — add `property_implementations_setupxxx` module key pointing to `CAddOnPropertyContractController`
- `Applications/Entrata/Application/Setup/CSetupSystemController.class.php` — add "Property Implementations" nav tab, feature-flagged
- `Applications/Entrata/Application/Setup/Properties/Property/AddOnPropertyContract/CAddOnPropertyContractController.class.php` — add `getConfigurationTypes()` method, extend `handleAddProperties()` with Slack notification, extend `handleAddPropertyToTemporaryStorage()` with new migration fields, update `getMigrationTypes()` if dropdown values change
- `Interfaces/Templates/Entrata/setup/properties/property/add_on_property_contract/view_add_property.tpl` — add Migration Area section (Configuration Type dropdown + conditional date/name fields) below State/Postal Code
- `Interfaces/Templates/Entrata/setup/properties/property/add_on_property_contract/header.tpl` — dark-blue header styling per prototype
- `Interfaces/Templates/Entrata/setup/properties/property/add_on_property_contract/success_page.tpl` — add "View Property" button
- `www/Entrata/Entrata/js/module/setup/properties/property/add_on_property_contract/add_on_property_contract.js` — add Configuration Type conditional show/hide logic, extend event handler at line ~850
- `www/Entrata/Entrata/css/modules/setup/properties/property/add_on_property_contract/add_on_property_contract.less` — modal fixed-width/height, dark-blue header, step indicator color update
- `Applications/Entrata/Library/AddOnPropertyContract/Services/CPropertyService.php` — add `sendSlackProjectChannelNotification()` method

**Config/migration**
- Feature flags: `property_implementations_tab`, `ssp_migration_fields`, `ssp_slack_notification` — all company-scoped, default OFF
- DB migration: new nullable columns on migration/property setup table

---

## 4. Components and Services Reused

| Type | Location in repo | Reused for | Why this fits |
|------|-----------------|------------|--------------|
| Controller | `Applications/Entrata/Application/Setup/Properties/Property/AddOnPropertyContract/CAddOnPropertyContractController.class.php` | Both nav entry points (Properties tab + new Property Implementations tab) | Same backend, second nav home — zero logic duplication |
| Service | `Applications/Entrata/Library/AddOnPropertyContract/Services/CPropertyService.php` | Property creation pipeline; Slack notification hooks into `saveProperties()` success path | Already owns property creation; extending it is correct |
| Temp storage service | `Applications/Entrata/Library/AddOnPropertyContract/Services/CTemporaryDataStorageService.php` | Persisting new migration fields through the wizard flow | Existing pattern for all wizard field persistence |
| State service | `Applications/Entrata/Library/AddOnPropertyContract/Services/CStateService.php` | State dropdown in Step 2 (unchanged) | Already wired |
| Module registration | `Applications/Entrata/Application/AssignApplicationModules.php` | Registering `property_implementations_setupxxx` | Standard pattern for all module registrations |
| Date picker | `mrdn-ui-pattern-datepicker` pattern (existing in view_add_property.tpl line ~130) | New conditional date fields | Same pattern; copy-paste the existing Migration Date field markup |

---

## 5. New Components and Services Added

| Type | Location in repo | Purpose | Open question |
|------|-----------------|---------|--------------|
| Method | `CAddOnPropertyContractController.class.php::getConfigurationTypes()` | Returns Configuration Type dropdown options array | Should options be hardcoded or DB-driven? Hardcoded for v1 per PM; revisit for v2 |
| Method | `CPropertyService.php::sendSlackProjectChannelNotification()` | Fire-and-forget Slack API call to `#cproject-{name}` after successful property creation | Does an existing Slack API helper exist in `Libraries/` or `Psi/`? Engineering to check before implementing |
| Nav entry | `property_implementations_setupxxx` in `AssignApplicationModules.php` | Second nav home for the SSP workflow under Setup | Should this share the same permission group as `add_on_property_contractxxx`? Default assumption: yes |
| DB columns | Migration table (name TBD by DBA) | Persist Configuration Type + conditional date/name fields | Which table owns migration metadata? Engineering to confirm with DBA |

---

## 6. Migrations and Shared-System Changes

**Database migration required:**
```sql
ALTER TABLE [migration_property_setup_table] 
  ADD COLUMN configuration_type VARCHAR(50) NULL,
  ADD COLUMN migration_start_date DATE NULL,
  ADD COLUMN close_date DATE NULL,
  ADD COLUMN priority_name VARCHAR(100) NULL,
  ADD COLUMN migration_completion_date DATE NULL;
```
*Table name TBD — engineering to confirm with DBA. All columns nullable (v1: no required validation).*

**Feature flag plumbing:**
| Flag | Where read | Default | Who toggles |
|------|-----------|---------|------------|
| `property_implementations_tab` | `CSetupSystemController.class.php` | OFF | ProServ admin |
| `ssp_migration_fields` | `CAddOnPropertyContractController.class.php::handleViewAddProperties()` | OFF | ProServ admin |
| `ssp_slack_notification` | `CPropertyService.php::sendSlackProjectChannelNotification()` | OFF | ProServ admin |

**Slack integration:**
- New Slack API call from `CPropertyService`. No existing Slack helper confirmed in codebase — engineering must check `Libraries/` before implementing from scratch.
- Channel name resolved from Client Admin client name (spaces → hyphens, lowercase).
- Fire-and-forget: property creation must succeed regardless of Slack result.
- Error codes: `SSP_SLACK_CHANNEL_NOT_FOUND`, `SSP_SLACK_SEND_FAILED` — log only, no user-facing block.

**No cross-repo dependencies** for v1. `core-config` does not need updates.

---

## 7. Test Gaps

**Existing test surfaces that touch the affected subtree:**
- Unit tests for `CAddOnPropertyContractController` (if any exist in `TestSuite/`) — engineering to locate
- Integration tests for `CPropertyService::saveProperties()` — extend to cover Slack call path

**New test surfaces this change should add:**
- `TC-1`: Property Implementations nav tab visible with flag ON, hidden with flag OFF
- `TC-2`: SSP modal launches from both nav entry points and reaches same Step 1
- `TC-3`: Configuration Type dropdown shows correct options per Migration Type selection
- `TC-4`: Conditional fields show/hide correctly for each Migration Type × Configuration Type combination (Takeover EDE, Takeover Third-Party — see spec)
- `TC-5`: New migration fields persist through temp storage → final save correctly
- `TC-6`: Slack notification fires after successful property creation (mock Slack API)
- `TC-7`: Property creation succeeds even when Slack channel not found
- `TC-8`: "View Property" button navigates to Property General Details for newly created property

**Test surfaces intentionally skipped:**
- Full conditional matrix for all 12 Migration Type × Configuration Type combinations — prototype is source of truth; only Takeover EDE and Standard + Takeover Third-Party are spec'd explicitly. Other combinations are lower priority for v1 testing.

---

## 8. How to Run the Prototype

**GitHub Pages prototype (no auth required):**
```
https://brandonbayles-hub.github.io/implementation-workbench-preview/
```
Open the URL → click "Add New Contract & Property" (top right, dark blue button) → walk through the SSP modal Steps 1-3.

**Loom walkthrough (8 min):**
```
https://www.loom.com/share/cf081d16544a4a3fb1de01aa7f5b0012
```

**Local Entrata (seeded env — engineering):**
- Module: `?module=property_implementations_setupxxx` (after nav registration is implemented)
- Feature flags must be ON for the company
- The prototype branch `prototype/ssp-v2-property-implementations` on `core--product-copy` contains handoff artifacts but not yet production code

**Dev environment caveats:**
- The GitHub Pages prototype does NOT connect to the Entrata backend — it demonstrates UX only
- Slack notification in the prototype is simulated (toast message); production will use real Slack API
- All migration fields are optional in v1 — no required validation to implement

---

## 9. Out of Scope

- Requiring any migration fields (all optional in v1; may be required in v2 after pilot)
- Bulk property import via CSV (separate epic)
- Dashboard/reporting on Configuration Type data (separate epic)
- Slack notification for property edits or deletions (v1: only on property creation)
- Full localization/i18n of new dropdown values
- Mobile-responsive SSP modal (existing modal is desktop-only; not addressed in v1)

---

## 10. Links

| Resource | Link |
|----------|------|
| Jira Epic | [DEV-304238](https://entrata.atlassian.net/browse/DEV-304238) |
| Prototype (GitHub Pages) | https://brandonbayles-hub.github.io/implementation-workbench-preview/ |
| Loom walkthrough | https://www.loom.com/share/cf081d16544a4a3fb1de01aa7f5b0012 |
| Spec | `specs/setup/sc-e001-ssp-v2-property-implementations.md` |
| Pipeline state | `specs/setup/_pipeline-state.json` |
| Anchoring audit | `audit/SC-E001/anchoring/anchoring-audit.md` |
| Signal intake | `audit/SC-E001/signal-intake.md` |
| ROI analysis | `runs/ssp-v2-property-implementations/roi-analysis.md` |
| Engineering Prompt | `core--product-copy/_handoff/DEV-304238/ENGINEERING-PROMPT.md` |

---

## 11. Vertical Impact

This feature ships in the Entrata `core` product under Setup > Property Implementations. Vertical impact assessment:

| Vertical | Required? | Type | Summary | Notes |
|----------|-----------|------|---------|-------|
| Residential | Yes | Full | Primary target — all conventional multifamily implementations use SSP workflow | No deviations from standard implementation |
| Commercial | Yes | Full | Commercial properties use SSP workflow for contract adds | Configuration Type options include commercial-relevant types (e.g., New Construction) |
| Affordable | Yes | Full | Affordable properties use SSP workflow | No deviations; migration fields apply equally |
| Student | Yes | Full | Student housing implementations use SSP | No deviations; same workflow |
| Military | Yes | Full | Military housing uses SSP | No deviations |
| Senior | Yes | Full | Senior housing uses SSP | No deviations |
| HOA | Partial | Limited | HOA properties may use SSP depending on contract structure | Engineering to confirm HOA property type visibility in dropdown |

**Scope guard:** No vertical-specific logic is introduced in v1 — this is a nav and form enhancement that applies uniformly across all verticals that use the SSP add-property workflow.
