# Engineering Starter Prompt — DEV-304238: Add Property Implementations Tab (SSP v2)

**Paste this into Cursor as your first message when implementing this epic.**

---

## Context

You are implementing DEV-304238 in the Entrata `core` PHP monolith. The feature adds a second navigation home for the existing Add New Contract & Property (SSP) workflow under Setup > Property Implementations, redesigns the SSP modal UX, adds a Migration Area section to Step 2, adds a Slack notification on property creation, and adds a View Property button post-create.

**Prototype (source of truth for UX):** https://brandonbayles-hub.github.io/implementation-workbench-preview/  
**Loom walkthrough (8 min):** https://www.loom.com/share/cf081d16544a4a3fb1de01aa7f5b0012  
**Jira:** DEV-304238

---

## Architecture Decisions

1. **Two nav homes, one backend.** The existing `CAddOnPropertyContractController` handles both the Properties tab entry and the new Property Implementations tab entry. Register `property_implementations_setupxxx` in `AssignApplicationModules.php` pointing to the same controller. Zero logic duplication.

2. **Migration fields follow the existing temp storage pattern.** The wizard stores form data via `CTemporaryDataStorageService` during the session, then persists on final submit via `CPropertyService::saveProperties()`. New fields (configuration_type, migration_start_date, close_date, priority_name, migration_completion_date) must flow through BOTH phases. See the existing `property_details[migration_type]` pattern in `handleAddPropertyToTemporaryStorage()` (line ~462) as the exact template.

3. **Slack notification is fire-and-forget.** Call `sendSlackProjectChannelNotification()` AFTER `saveProperties()` returns success. If Slack fails, log the error code and return normally — the property creation response must not be blocked. Channel name: `#cproject-{client-name-lowercase-hyphens}` where client name comes from `$this->getClient()->getName()` or equivalent.

4. **Feature flags are company-scoped, default OFF.** Three independent flags: `property_implementations_tab`, `ssp_migration_fields`, `ssp_slack_notification`. Check each flag at its own gate; do not bundle them.

5. **All new migration fields are nullable with no required validation in v1.** This is intentional per PM. Do not add `required` validation to the new fields. The existing `migration_type_id` has validation — do not remove it.

6. **Configuration Type dropdown is hardcoded in v1.** Options: `Takeover EDE`, `Takeover Third-Party`, `New Construction`, `NOTD`, `Accounting Only`, `Other`. Return these from a new `getConfigurationTypes()` method on the controller. DB-driven in v2.

---

## Implementation Sequence

Follow this order to avoid blocking yourself:

### Step 1: DB migration (do this first)
```sql
-- Run in dev, then staging, then prod
ALTER TABLE [confirm table name with DBA — likely setup_property_migration or similar]
  ADD COLUMN configuration_type VARCHAR(50) NULL AFTER migration_type,
  ADD COLUMN migration_start_date DATE NULL,
  ADD COLUMN close_date DATE NULL,
  ADD COLUMN priority_name VARCHAR(100) NULL,
  ADD COLUMN migration_completion_date DATE NULL;
```
*Confirm table name before running. All nullable — no data loss on rollback.*

### Step 2: Nav registration
In `Applications/Entrata/Application/AssignApplicationModules.php`, add alongside the existing `add_on_property_contractxxx` entry:
```php
'property_implementations_setupxxx' => './Setup/Properties/Property/AddOnPropertyContract/CAddOnPropertyContractController.class.php',
```

In `Applications/Entrata/Application/Setup/CSetupSystemController.class.php`, add the "Property Implementations" tab to the Setup nav, gated by `property_implementations_tab` feature flag.

### Step 3: New controller methods
In `CAddOnPropertyContractController.class.php`:

```php
private function getConfigurationTypes(): array {
    return [
        ['id' => 'takeover_ede',        'name' => 'Takeover EDE'],
        ['id' => 'takeover_third_party', 'name' => 'Takeover Third-Party'],
        ['id' => 'new_construction',     'name' => 'New Construction'],
        ['id' => 'notd',                 'name' => 'NOTD'],
        ['id' => 'accounting_only',      'name' => 'Accounting Only'],
        ['id' => 'other',                'name' => 'Other'],
    ];
}
```

Pass `$arrmixTemplateParameters['configuration_types'] = $this->getConfigurationTypes();` alongside the existing `migration_types` assignment at line ~796.

### Step 4: Template changes (view_add_property.tpl)
After the State/Postal Code row (line ~100), add the Migration Area section:
- Section heading: "Migration Area"
- Migration Type dropdown: existing pattern (already present — confirm the options match "Standard" and "Sim Self-Migration")
- Configuration Type dropdown: same `mrdn-form-select` pattern as Migration Type, id=`configuration_types`, hidden input id=`configuration_type_id`
- Conditional date fields (initially `display:none`, shown by JS):
  - `migration_start_date` (date picker, same pattern as existing `migration_date` field)
  - `close_date` (date picker)
  - `priority_name` (text input, maxlength=100)
  - `migration_completion_date` (date picker)

Reference line ~108-160 of `view_add_property.tpl` for the exact `mrdn-form-select` and date picker markup pattern.

### Step 5: JS — conditional show/hide (add_on_property_contract.js)
Extend the existing `change` event handler at line ~850 (currently handles `property_type_id`, `state_id`, `migration_type_id`):

```javascript
// Add configuration_type_id to the existing event selector
$( document ).on( 'change', '#property_type_id, #state_id, #migration_type_id, #configuration_type_id', function() {
    updateConditionalMigrationFields();
});

function updateConditionalMigrationFields() {
    var strMigrationType = $( '#migration_type_id' ).val();
    var strConfigType = $( '#configuration_type_id' ).val();
    // Show/hide logic per the spec:
    // Takeover EDE → show start_date, close_date, priority_name, completion_date
    // Standard + Takeover Third-Party → show start_date, close_date, completion_date
    // All other combinations → hide all conditional fields
    // Reference prototype for full matrix
}
```

Also add `configuration_type_id` to the `syncAddPropertyMrdnSelectHidden` call (line ~456 pattern).

### Step 6: Temp storage + persistence
In `handleAddPropertyToTemporaryStorage()`: extract `property_details[configuration_type]`, `property_details[migration_start_date]`, `property_details[close_date]`, `property_details[priority_name]`, `property_details[migration_completion_date]` from request data, following the existing `migration_type` pattern.

In `handleAddProperties()` / `CPropertyService::saveProperties()`: persist the new fields to the DB columns added in Step 1.

### Step 7: Slack notification
In `CPropertyService.php`, add:

```php
public function sendSlackProjectChannelNotification(
    array $arrmixPropertyDetails,
    string $strClientName
): void {
    // Fire-and-forget — never throw; log errors only
    try {
        $strChannel = '#cproject-' . strtolower(str_replace(' ', '-', $strClientName));
        // Build message with all migration fields
        // Call Slack API (check Libraries/ for existing helper first)
        // Log success/failure to application log
    } catch (\Exception $e) {
        error_log('SSP_SLACK_SEND_FAILED: ' . $e->getMessage());
    }
}
```

Call this from `handleAddProperties()` after successful `saveProperties()` call, gated by `ssp_slack_notification` flag.

### Step 8: UX changes (LESS + templates)
- `add_on_property_contract.less`: fixed modal width/height per prototype, dark-blue header color, step indicator color update. Reference prototype URL for exact values.
- `header.tpl`: apply dark-blue background class to modal header
- `success_page.tpl`: add "View Property" button linking to `property_details_generalxxx` with the new property ID

---

## Integration Points

| System | Interaction | File |
|--------|-------------|------|
| Client Admin nav | Tab registration | `AssignApplicationModules.php`, `CSetupSystemController.class.php` |
| Wizard temp storage | Field persistence mid-session | `CTemporaryDataStorageService.php` |
| Property creation | Final save + DB write | `CPropertyService::saveProperties()` |
| Slack API | Post-create notification | `CPropertyService::sendSlackProjectChannelNotification()` — NEW |
| Property General Details | "View Property" button target | `property_details_generalxxx` module (already registered) |
| Feature flags | 3 independent company flags | Entrata feature flag system |

---

## Definition of Done

- [ ] DB migration ran cleanly in dev; all 5 columns present and nullable
- [ ] `property_implementations_setupxxx` module registered; nav tab visible with flag ON, hidden with flag OFF
- [ ] SSP modal opens from both nav locations (Properties tab + Property Implementations tab)
- [ ] Modal header is dark-blue per prototype; step indicator colors updated
- [ ] Migration Area section appears in Step 2 with both dropdowns
- [ ] Conditional fields show/hide correctly for Takeover EDE and Takeover Third-Party combinations (minimum — full matrix per prototype)
- [ ] All new fields persist through temp storage → final save → confirmed in DB
- [ ] Slack notification fires after successful property creation; property creation succeeds when Slack fails
- [ ] "View Property" button appears post-create and navigates to Property General Details
- [ ] All 8 SDET test cases from spec pass
- [ ] PHP lint clean (`php vendor/bin/phpcs --standard=phpcs.xml {changed-files}`)
- [ ] No JS console errors on Step 1 or Step 2 of the SSP modal
- [ ] Feature flags all default to OFF; can be toggled per company

---

## Key Files to Read First

Before writing any code, read these files to understand existing patterns:

```
Applications/Entrata/Application/Setup/Properties/Property/AddOnPropertyContract/CAddOnPropertyContractController.class.php
Applications/Entrata/Library/AddOnPropertyContract/Services/CPropertyService.php
Applications/Entrata/Library/AddOnPropertyContract/Services/CTemporaryDataStorageService.php
Interfaces/Templates/Entrata/setup/properties/property/add_on_property_contract/view_add_property.tpl
www/Entrata/Entrata/js/module/setup/properties/property/add_on_property_contract/add_on_property_contract.js
Applications/Entrata/Application/AssignApplicationModules.php (lines 1217-1230)
```
