# 6. UI Layer

## 6.1 UI Actions

| Name                          | Table                                  | Where           | Condition (role gate)                                              | Behavior                                                                  |
| ----------------------------- | -------------------------------------- | --------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| `Scramble Data`               | `x_stave_dt_data_scramble_definition`  | Form button     | `gs.hasRole('x_stave_dt.scramble_user')`                           | Confirms count via `DataScramble_Ajax.getTotalCount()`, then executes.    |
| `Generate Data`               | `x_stave_dt_data_generator`            | Form button     | `gs.hasRole('x_stave_dt.generator_user')`                          | Calls `generateDataConfirm()` client-side, which invokes `DataToolsHelper.generateData(def)`. |
| `Mask Data`                   | `x_stave_dt_data_mask_alert`           | Form button     | `gs.hasRole('x_stave_dt.user')`                                    | Applies the alert's resolution back to the source field. (action_name: `x_stave_dt_mask_data`) |
| `View Record` (×2)            | `x_stave_dt_data_mask_alert`, `x_stave_dt_data_mask_replaced` | Form button | `gs.hasRole('x_stave_dt.user')`                                | Navigates to the original record referenced by `table`+`id`.              |
| `Generate Regular Expression` | `x_stave_dt_data_mask_definition`      | Form link       | `current.dictionary_lookup == true && gs.hasRole('x_stave_dt.user')` | Builds a regex from the dictionary entries for the configured `dictionary_table`. |

## 6.2 Client Scripts

| Name                  | Table                              | Type     | Purpose                                                                                                                       |
| --------------------- | ---------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `Infinite Loop Checker` | `x_stave_dt_data_mask_definition` | onSubmit | Blocks save if `replace_text` contains `regular_expression` (which would cause the rule to re-trigger on its own output) or if the regex is too generic. |

## 6.3 UI Policies

The app ships ~27 UI policies that toggle field visibility based on related field values (e.g. on the data generator, hide the `Fields` selector when `table = sys_user`; on the mask definition, hide `dictionary_table` unless `dictionary_lookup = true`). See [update/sys_ui_policy_*.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/) for the full list.

## 6.4 App Modules (Navigation)

The application menu **Stave Data Tools** (`ec13bd5d0fe2e640c783cd8ce1050e72`) groups 39 modules into:

- **Data Management** — Data Mask Definitions (All / Create), Data Mask Alerts, Data Mask Replaced, Data Scramble Definitions, Data Scramble Logs, Data Generators.
- **Configuration** — Dictionary Entries, Business Rules, Script Includes.
- **Utilities** — Getting Started (external link), Data Quarantine, Replacement Log, Import Templates.
- **Support** — About Stave, More Stave Apps, Contact Stave Support.
