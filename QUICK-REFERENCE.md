# Quick Reference — DEV-304238: SSP v2

## What this is
New Setup > Property Implementations tab + SSP modal redesign + Migration Area fields + Slack notification + View Property button.

## Prototype
https://brandonbayles-hub.github.io/implementation-workbench-preview/ → click "Add New Contract & Property"

## Jira
https://entrata.atlassian.net/browse/DEV-304238 | Rapid: July 7 | Standard GA: October 6

## Key files to touch
| File | Change |
|------|--------|
| `AssignApplicationModules.php` | Add `property_implementations_setupxxx` module |
| `CSetupSystemController.class.php` | Add Property Implementations nav tab |
| `CAddOnPropertyContractController.class.php` | Add `getConfigurationTypes()`, extend temp storage + Slack |
| `view_add_property.tpl` | Add Migration Area section |
| `add_on_property_contract.js` | Add Configuration Type conditional show/hide |
| `add_on_property_contract.less` | Fixed modal dimensions, dark-blue header |
| `success_page.tpl` | Add View Property button |
| `CPropertyService.php` | Add `sendSlackProjectChannelNotification()` |

## Feature flags (all OFF by default, company-scoped)
- `property_implementations_tab`
- `ssp_migration_fields`
- `ssp_slack_notification`

## DB migration needed
5 nullable columns on the property migration table: `configuration_type`, `migration_start_date`, `close_date`, `priority_name`, `migration_completion_date`

## Slack channel format
`#cproject-{client-name-lowercase-hyphens}` — matches Client Admin display name

## All new migration fields are OPTIONAL in v1. Do not add required validation.
