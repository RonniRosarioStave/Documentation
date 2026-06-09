# Stave Data Tools (`x_stave_dt`) — Technical Documentation

> ServiceNow scoped application that provides a unified toolbox for **data hygiene**: preventing unwanted data from entering the platform (Data Masker), scrubbing existing data in place (Data Scrambler), and generating synthetic records for testing (Data Generator).

---

## 1. At-a-Glance

| Attribute            | Value                                                                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Name**             | Stave Data Tools                                                                                                                     |
| **Scope**            | `x_stave_dt`                                                                                                                         |
| **Application sys_id** | `d0aee0190f62e640c783cd8ce1050e34`                                                                                                 |
| **Vendor**           | Stave Inc                                                                                                                            |
| **Version**          | 3.2.23                                                                                                                               |
| **JS Level**         | `es_latest`                                                                                                                          |
| **Created**          | 2023-11-23 by `jonas.perusquia`                                                                                                      |
| **Last Updated**     | 2025-11-21 by `Jonas.Perusquia@stavecorp.com`                                                                                        |
| **Store ID**         | `REPOAPP0002349846` (TPP appstore)                                                                                                   |
| **Source manifest**  | [sys_app_d0aee0190f62e640c783cd8ce1050e34.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/sys_app_d0aee0190f62e640c783cd8ce1050e34.xml) |

**Short description (from manifest):**

> The Stave Data Masker provides the ability to define regular expressions that when matched will mask the data to prevent unwanted data from being entered in the system.
>
> The Stave Data Scrambler provides the ability to define regular expressions and when executed, the matched data will be altered to prevent unwanted data from being kept in the system. Each scrambler definition must be executed manually.

---

## 2. Capabilities Overview

The application is organized around **three pillars** plus one **supporting subsystem**:

| Pillar              | Trigger model        | Operates on             | Purpose                                                                                  |
| ------------------- | -------------------- | ----------------------- | ---------------------------------------------------------------------------------------- |
| **Data Masker**     | Real-time (business rule on `task` insert/update) | The record being saved  | Block or replace sensitive data **before** it is persisted. Optional alert-only mode.    |
| **Data Scrambler**  | Manual (UI action)   | Existing records in bulk | Walk a result set and overwrite matched characters/fields in place. Logged per execution. |
| **Data Generator**  | Manual (UI action)   | Target table (new rows) | Insert N synthetic records with randomized or evenly-distributed field values.            |
| **Dictionary**      | Configuration support | Mask Definitions       | Provide a curated word list as an alternative to regex pattern matching.                  |

---

## 3. Architecture

```
                  ┌──────────────────────────────────────────────────────┐
                  │                  Stave Data Tools                    │
                  └──────────────────────────────────────────────────────┘

  ┌─── Definition tables ───┐    ┌── Engines (Script Includes) ──┐    ┌── Outputs / Logs ──┐

  data_mask_definition  ───────► DataMaskerHelper.maskRecord()  ─────► data_mask_alert
                                 (or Masker.getBool())                  data_mask_replaced

  data_scramble_definition ────► (UI Action "Scramble Data")  ────────► data_scramble_log
                                  + DataScramble_Ajax.getTotalCount()    data_mask_alert
                                                                         (when alerts=true)

  data_generator ──────────────► DataToolsHelper.generateData()  ─────► target table (new rows)

  dictionary_entry  ────────────► consulted by Mask Definitions
                                  when dictionary_lookup = true
```

**Real-time mask trigger flow:**

1. A user saves any record on `task` (or a child class).
2. The `DataMasker Check` business rule (before, insert/update) fires.
3. It calls `new x_stave_dt.DataMaskerHelper().maskRecord(current)`.
4. The helper queries `x_stave_dt_data_mask_definition` records where `active = true` and `table = current.getTableName()`, ordered by `order`.
5. For each match, the helper either creates an alert record (`mode = Alert`) or replaces the field value in `current` (`mode = Replace`), and writes a row to `data_mask_replaced` when an actual replacement occurred.

---

## 4. Data Model — Table Reference

The app ships **7 tables**, all prefixed `x_stave_dt_`. The first three extend `sys_metadata` (i.e. they participate in the Source Control / update-set lifecycle); the rest are plain collections.

### 4.1 `x_stave_dt_data_mask_definition`

Defines a real-time masking rule. Source: [dictionary/x_stave_dt_data_mask_definition.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_definition.xml)

| Field                     | Type            | Mand. | Hint / Notes                                                                                                |
| ------------------------- | --------------- | :---: | ----------------------------------------------------------------------------------------------------------- |
| `name`                    | string (100)    |   Y   | Name of this data mask definition.                                                                          |
| `description`             | string (1000)   |       | The description of the data mask definition.                                                                |
| `active`                  | boolean         |       | Default `true`. Toggles the rule on/off.                                                                    |
| `mode`                    | string (40)     |   Y   | `Alert` (log only) or `Replace` (overwrite with `replace_text`). Default `Alert`.                          |
| `table`                   | table_name (80) |   Y   | Table that will be monitored by this rule. (`allow_public=true`)                                            |
| `fields`                  | string (1000)   |   Y   | Comma-separated list of fields on `table` to evaluate.                                                      |
| `filter`                  | conditions (4000) |     | Condition-builder filter scoping which records trigger the rule (v2 condition builder, dependent on `table`). |
| `case_sensitive_filter`   | boolean         |       | Apply `filter` case-sensitively.                                                                            |
| `regular_expression`      | string (100000) |   Y   | The regex pattern matched against the specified fields. Edge-encryption-enabled.                            |
| `conditions_regex`        | string (2560)   |       | Additional regex applied to conditions.                                                                     |
| `case_sensitive`          | boolean         |       | Default `true`. Apply `regular_expression` case-sensitively.                                                |
| `dictionary_lookup`       | boolean         |       | If `true`, match against `x_stave_dt_dictionary_entry` records (table = `dictionary_table`) instead of regex. |
| `dictionary_table`        | table_name (80) |       | The table that holds the dictionary entries for lookup.                                                     |
| `replace_text`            | string (1000)   |       | Text inserted in place of the matched pattern when `mode = Replace`.                                        |
| `first`                   | boolean         |       | Replace **only the first** match in the field.                                                              |
| `last`                    | boolean         |       | Replace **only the last** match in the field.                                                               |
| `exclude_characters`      | integer         |       | Number of characters to skip when masking (e.g. last 4 of an SSN).                                          |
| `include_original_text`   | boolean         |       | Default `true`. If true, original value is stored on the `data_mask_replaced` audit row.                    |
| `order`                   | integer         |       | Execution priority when multiple definitions monitor the same table (lower runs first).                     |

Extends `sys_metadata`. The application also ships a `sys_dictionary` override per field to set tooltips and column widths (see [update/sys_dictionary_x_stave_dt_data_mask_definition_*.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/)).

### 4.2 `x_stave_dt_data_mask_alert`

Audit row written whenever a `mode=Alert` mask rule fires (and optionally when scramble rules with `alerts=true` fire). Source: [dictionary/x_stave_dt_data_mask_alert.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_alert.xml)

| Field                       | Type                                      | Read-only | Notes                                                                  |
| --------------------------- | ----------------------------------------- | :-------: | ---------------------------------------------------------------------- |
| `number`                    | string (40)                               |           | Auto-generated via `getNextObjNumberPadded()`.                         |
| `user`                      | reference → `sys_user` (32)               |   Yes     | User whose save triggered the alert.                                   |
| `table`                     | string (40)                               |   Yes     | Source table where the match occurred.                                 |
| `field`                     | string (40)                               |   Yes     | Field on `table` where the match occurred.                             |
| `id`                        | string (40)                               |   Yes     | `sys_id` of the source record.                                         |
| `matched_text`              | string (1000)                             |   Yes     | The substring that matched the pattern.                                |
| `data_mask_definition`      | reference → `x_stave_dt_data_mask_definition` (32) | Yes  | Definition that fired (null if scramble-sourced).                      |
| `data_scramble_definition`  | reference → `x_stave_dt_data_scramble_definition` (32) | Yes | Definition that fired (null if mask-sourced).                          |
| `state`                     | string (40)                               |           | `Pending Review` (default), `Masked`, or `Ignored`.                    |

Indexes: `data_mask_definition`, `data_scramble_definition`, `user`.

### 4.3 `x_stave_dt_data_mask_replaced`

Audit row written whenever a `mode=Replace` rule actually overwrites a value. Source: [dictionary/x_stave_dt_data_mask_replaced.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_replaced.xml)

| Field                  | Type                                                | Notes                                                                            |
| ---------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------- |
| `number`               | string (40)                                         | Auto-generated.                                                                  |
| `user`                 | reference → `sys_user` (32)                         | User whose save triggered the replacement.                                       |
| `data_mask_definition` | reference → `x_stave_dt_data_mask_definition` (32)  | Read-only. Rule that fired.                                                      |
| `table`                | table_name (80)                                     | Read-only. Source table.                                                         |
| `field`                | string (40)                                         | Field that was overwritten.                                                      |
| `id`                   | string (40)                                         | `sys_id` of the source record.                                                   |
| `mode`                 | string (40)                                         | The `mode` of the firing definition (typically `Replace`).                       |
| `original_text`        | string (100)                                        | Default `Not logged`. Populated only when the definition has `include_original_text = true`. |

Indexes: `data_mask_definition`, `user`.

### 4.4 `x_stave_dt_data_scramble_definition`

Defines a manually-executed bulk scramble. Source: [dictionary/x_stave_dt_data_scramble_definition.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_scramble_definition.xml)

| Field                  | Type                | Mand. | Notes                                                                                                          |
| ---------------------- | ------------------- | :---: | -------------------------------------------------------------------------------------------------------------- |
| `number`               | string (40)         |       | Auto-generated.                                                                                                |
| `description`          | string (100)        |       | Description of this scramble definition.                                                                       |
| `table`                | table_name (80)     |   Y   | Table whose data will be scrambled (`allow_public=true`).                                                      |
| `fields`               | string (40000)      |       | Comma-separated list of fields on `table` to scramble.                                                         |
| `filter`               | conditions (4000)   |       | Condition-builder filter (v2, dependent on `table`).                                                            |
| `regular_expresssion`  | string (100000)     |       | Regex matching the substring within each field to scramble. (*Note: source dictionary spells this with three `s`s.*) |
| `case_sensitive`       | boolean             |       | Default `true`. Edge-encryption-enabled.                                                                       |
| `mode`                 | choice (40)         |       | Scramble mode — see codes below. Default `10` (Xs).                                                            |
| `alerts`               | boolean             |       | If `true`, also write a `data_mask_alert` row for each scrambled value.                                        |

**Scramble mode codes** (stored as the integer string in the `mode` field):

| Code | Label       | Behavior                                                              |
| ---- | ----------- | --------------------------------------------------------------------- |
| `10` | Xs          | Replace every matched character with `x`.                             |
| `20` | Shuffle     | Re-order the characters of the match randomly.                        |
| `30` | Random User | Replace with user data drawn from `api.randomuser.me`.                |
| `40` | Nowlpsum    | Replace with ServiceNow-flavored lorem ipsum text. (Label as shipped.) |

### 4.5 `x_stave_dt_data_scramble_log`

Execution log for scramble runs. Source: [dictionary/x_stave_dt_data_scramble_log.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_scramble_log.xml)

| Field                 | Type                                                   | Notes                                                                                |
| --------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `number`              | string (40)                                            | Auto-generated.                                                                      |
| `scramble_definition` | reference → `x_stave_dt_data_scramble_definition` (32) | Definition that ran. (Field label is "Number" in the dictionary, but the column is `scramble_definition`.) |
| `description`         | string (100)                                           | Free-text description.                                                               |
| `record_table`        | table_name (40)                                        | The table that was scrambled (mirrors `scramble_definition.table`).                  |
| `record`              | document_id (32)                                       | Reference to a specific scrambled record (dependent on `record_table`).              |
| `start_date`          | glide_date_time                                        | Run start.                                                                           |
| `end_date`            | glide_date_time                                        | Run end.                                                                             |
| `matching_records`    | integer                                                | Records that matched the definition's filter + regex.                                |
| `modified_records`    | integer                                                | Records actually written (may differ from matching when ACLs block writes).          |

Indexes: `record`, `scramble_definition`.

### 4.6 `x_stave_dt_data_generator`

Synthetic data generation definition. Source: [dictionary/x_stave_dt_data_generator.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_generator.xml)

| Field               | Type            | Mand. | Notes                                                                                       |
| ------------------- | --------------- | :---: | ------------------------------------------------------------------------------------------- |
| `number`            | string (40)     |       | Auto-generated.                                                                             |
| `description`      | string (40)     |       | Description of this generator.                                                              |
| `table`             | table_name (80) |   Y   | Table that will receive generated rows (`allow_public`).                                    |
| `quantity`          | integer         |   Y   | How many records to create per execution.                                                   |
| `fields`            | string (1000)   |   Y   | Comma-separated list of fields on `table` to populate.                                      |
| `reference_setting` | choice (40)     |       | `10` Random (default) / `20` Evenly Distributed — controls how reference field values are picked. |
| `choice_setting`    | choice (40)     |       | `10` Random (default) / `20` Evenly Distributed — same, for choice fields.                  |
| `set_values`        | string (4000)   |       | JSON map of `field → fixed_value`. Each key must appear in `fields`.                        |

### 4.7 `x_stave_dt_dictionary_entry`

Custom word/phrase list used by mask definitions when `dictionary_lookup = true`. Source: [dictionary/x_stave_dt_dictionary_entry.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_dictionary_entry.xml)

| Field            | Type             | Notes                                                       |
| ---------------- | ---------------- | ----------------------------------------------------------- |
| `name`           | string (40)      | Unique. The word/phrase to match.                           |
| `sys_class_name` | sys_class_name (80) | Default `current.getTableName()`. Supports table extension. |

Indexes: `name` (and a `unique` index on `name`), `sys_class_name`.

The app also ships an import template — [u_imp_tmpl_x_stave_dt_dictionary_entry.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/u_imp_tmpl_x_stave_dt_dictionary_entry.xml) — for bulk-loading dictionary entries.

---

## 5. Business Logic — Server-Side

### 5.1 Script Includes

All located in [update/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/).

| Name                | API name                       | Access            | Client-callable | Responsibility                                                                                                                                                  |
| ------------------- | ------------------------------ | ----------------- | :-------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DataMaskerHelper`  | `x_stave_dt.DataMaskerHelper`  | public            |        No       | Core masking engine. `maskRecord(gr, isReturnValue)` iterates active definitions for `gr.getTableName()`, evaluates regex/dictionary, then either alerts or replaces. |
| `Masker`            | `x_stave_dt.Masker`            | public            |        No       | Lighter-weight alternative. `getBool(gr)` returns `true` if any active definition would match the record — useful for short-circuit checks.                       |
| `DataToolsHelper`   | `x_stave_dt.DataToolsHelper`   | public            |        No       | Data Generator engine. `generateData(def)` reads the generator record, types each target field via `getED().getInternalType()`, and inserts `quantity` rows. Special-cases `table == sys_user` (random-user API). |
| `DataScramble_Ajax` | `x_stave_dt.DataScramble_Ajax` | package_private   |       Yes       | Extends `global.AbstractAjaxProcessor`. `getTotalCount()` previews how many records a scramble definition would touch by running a `GlideAggregate(COUNT)` against its filter. |
| `RegexUtils`        | `x_stave_dt.RegexUtils`        | public            |        No       | Plain-object utility. `convertStringToRegex(str)` turns `/pattern/flags` strings into `RegExp`; `flags(rgx)` extracts the trailing flags; helpers for matches & reflag. |

> Each include carries the same copyright header: *Copyright (C) 2026 Stave, Inc. All Rights Reserved — Unauthorized copying ... Proprietary and confidential.*

### 5.2 Business Rules

All located in [update/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/).

| Name                          | Table                                | When            | Active | Purpose                                                                              |
| ----------------------------- | ------------------------------------ | --------------- | :----: | ------------------------------------------------------------------------------------ |
| `DataMasker Check`            | `task`                               | before insert/update |   Y    | Calls `DataMaskerHelper.maskRecord(current)`. The hot path of the real-time masker.  |
| `Format Fields field in DG`   | `x_stave_dt_data_generator`          | before insert/update |   Y    | Strips all whitespace from the comma-separated `fields` column on save.              |
| `Format Fields field in DMD`  | `x_stave_dt_data_mask_definition`    | before insert/update |   Y    | Same whitespace normalization, on mask definitions.                                  |
| `Format Fields field in DSD`  | `x_stave_dt_data_scramble_definition`| before insert/update |   Y    | Same whitespace normalization, on scramble definitions.                              |
| `Run Mask Definitions`        | `live_message`                       | before insert/update |   N    | Disabled. Would mirror `DataMasker Check` for the Connect Chat / live-message table. |
| `Move comments` *(deleted)*   | `incident`                           | before insert/update |   —    | Tombstone for an old rule that moved comment content; deleted Jan 2026. See [author_elective_update/sys_script_0ac3a16d0f2ae640c783cd8ce1050eb3.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/author_elective_update/sys_script_0ac3a16d0f2ae640c783cd8ce1050eb3.xml). |

---

## 6. UI Layer

### 6.1 UI Actions

| Name                          | Table                                  | Where           | Condition (role gate)                                              | Behavior                                                                  |
| ----------------------------- | -------------------------------------- | --------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| `Scramble Data`               | `x_stave_dt_data_scramble_definition`  | Form button     | `gs.hasRole('x_stave_dt.scramble_user')`                           | Confirms count via `DataScramble_Ajax.getTotalCount()`, then executes.    |
| `Generate Data`               | `x_stave_dt_data_generator`            | Form button     | `gs.hasRole('x_stave_dt.generator_user')`                          | Calls `generateDataConfirm()` client-side, which invokes `DataToolsHelper.generateData(def)`. |
| `Mask Data`                   | `x_stave_dt_data_mask_alert`           | Form button     | `gs.hasRole('x_stave_dt.user')`                                    | Applies the alert's resolution back to the source field. (action_name: `x_stave_dt_mask_data`) |
| `View Record` (×2)            | `x_stave_dt_data_mask_alert`, `x_stave_dt_data_mask_replaced` | Form button | `gs.hasRole('x_stave_dt.user')`                                | Navigates to the original record referenced by `table`+`id`.              |
| `Generate Regular Expression` | `x_stave_dt_data_mask_definition`      | Form link       | `current.dictionary_lookup == true && gs.hasRole('x_stave_dt.user')` | Builds a regex from the dictionary entries for the configured `dictionary_table`. |

### 6.2 Client Scripts

| Name                  | Table                              | Type     | Purpose                                                                                                                       |
| --------------------- | ---------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `Infinite Loop Checker` | `x_stave_dt_data_mask_definition` | onSubmit | Blocks save if `replace_text` contains `regular_expression` (which would cause the rule to re-trigger on its own output) or if the regex is too generic. |

### 6.3 UI Policies

The app ships ~27 UI policies that toggle field visibility based on related field values (e.g. on the data generator, hide the `Fields` selector when `table = sys_user`; on the mask definition, hide `dictionary_table` unless `dictionary_lookup = true`). See [update/sys_ui_policy_*.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/) for the full list.

### 6.4 App Modules (Navigation)

The application menu **Stave Data Tools** (`ec13bd5d0fe2e640c783cd8ce1050e72`) groups 39 modules into:

- **Data Management** — Data Mask Definitions (All / Create), Data Mask Alerts, Data Mask Replaced, Data Scramble Definitions, Data Scramble Logs, Data Generators.
- **Configuration** — Dictionary Entries, Business Rules, Script Includes.
- **Utilities** — Getting Started (external link), Data Quarantine, Replacement Log, Import Templates.
- **Support** — About Stave, More Stave Apps, Contact Stave Support.

---

## 7. Security Model

### 7.1 Roles (`sys_user_role`)

All located in [update/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/).

| Role                              | Description (verbatim)                                                                                                              |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `x_stave_dt.admin`                | This user can view Getting Started, Data Masker, Data Quarantine, Replacement Log, Scramble Logs and All Data Scrambler modules and create data mask definitions. |
| `x_stave_dt.user`                 | This user can view Data Mask Definitions, Data Mask Replaced, Getting Started, Stave Data Tools, Create new Monitored Phrase, Data Quarantine, About Stave, More Stave Apps and Contact Stave Support modules. |
| `x_stave_dt.scramble_user`        | This user can only view Create new Scramble Definition and All Scramble Definitions modules.                                        |
| `x_stave_dt.generator_user`       | This user can only view the Data Generator module.                                                                                  |
| `x_stave_dt.dictionary_entry_user`| This user can only view the Dictionary Entries module.                                                                              |

All roles have `can_delegate = true` and `grantable = true`. None set `elevated_privilege`.

Several `sys_user_role_contains_*.xml` records define role-inheritance edges (e.g. `admin` contains `user`).

### 7.2 ACLs

The app ships roughly **130 ACLs** (`sys_security_acl_*.xml`) — broken down as ~56 base table ACLs (read/write/create/delete per table) plus ~74 role-specific overrides. They enforce that:

- Mask/scramble definition CRUD is gated by `x_stave_dt.admin` (or the corresponding `*_user` role for read).
- Alert and Replaced rows are readable by `x_stave_dt.user` and writable by `x_stave_dt.admin`.
- Dictionary entries are readable by `x_stave_dt.dictionary_entry_user`.

---

## 8. Configuration

### 8.1 System Properties

| Property                          | Type        | Default                       | Purpose                                                                                  |
| --------------------------------- | ----------- | ----------------------------- | ---------------------------------------------------------------------------------------- |
| `x_stave_dt.name_generator_api`   | string      | `http://api.randomuser.me/`   | Endpoint used by `DataToolsHelper` when generating `sys_user` rows or by Random-User scramble mode. |
| `x_stave_dt.logging.verbosity`    | choicelist  | `debug`                       | Logging level for all script includes. Choices: `debug`, `info`, `warn`, `error`.       |

A handful of `sys_properties_category_*.xml` records group these under a "Stave Data Tools" properties category for the System Properties UI.

### 8.2 Choice Lists

Choice values for the dictionary fields are defined inline (see §4), plus the `author_elective_update/` folder ships these author-customized choice lists:

- `x_stave_dt_data_generator.choice_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_generator.reference_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_mask_alert.state` — Pending Review, Masked, Ignored
- `x_stave_dt_data_mask_definition.mode` — Alert, Replace
- `x_stave_dt_data_scramble_definition.mode` — Xs (`10`), Shuffle (`20`), Random User (`30`), Nowlpsum (`40`)

### 8.3 Scheduled Jobs

The app ships exactly one schedule record — [cmn_schedule_5056b9f70ffbf28055d1cd8ce1050ec7.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/cmn_schedule_5056b9f70ffbf28055d1cd8ce1050ec7.xml). It is a configuration record (no active recurring scramble or mask job ships out of the box).

### 8.4 Data Source

One `sys_data_source` record ([sys_data_source_39fd71800f1e5b4055d1cd8ce1050ec0.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_data_source_39fd71800f1e5b4055d1cd8ce1050ec0.xml)) backs the dictionary-entry import template.

---

## 9. Usage / How-To

### 9.1 Defining a real-time mask rule (regex)

Example shipped in the app — IP-address masker on the incident table — [x_stave_dt_data_mask_definition_1fcc7a690feae640c783cd8ce1050e4a.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/x_stave_dt_data_mask_definition_1fcc7a690feae640c783cd8ce1050e4a.xml):

```text
Name:               IP Address Monitoring and Mask
Table:              incident
Fields:             short_description
Mode:               Replace
Regular Expression: \b(?:\d{1,3}\.){3}\d{1,3}\b
Replace Text:       #Masked IP Address#
Case sensitive:     true
Order:              5
Active:             false       (shipped disabled — enable per environment)
```

When a user saves an incident with `short_description = "Down at 10.0.5.42"`, the `DataMasker Check` business rule rewrites the field to `Down at #Masked IP Address#` **before** the row is persisted, and writes an audit row to `x_stave_dt_data_mask_replaced` with `original_text = "Down at 10.0.5.42"` (because `include_original_text` defaults to `true`).

### 9.2 Defining a mask rule (dictionary lookup)

1. Add words/phrases to `x_stave_dt_dictionary_entry` (one row per term).
2. Create a `x_stave_dt_data_mask_definition` with:
   - `dictionary_lookup = true`
   - `dictionary_table = <your dictionary table>` (defaults to `x_stave_dt_dictionary_entry`)
   - Leave `regular_expression` blank — the UI action **Generate Regular Expression** will build one from the dictionary on demand.

### 9.3 Running a scramble job

1. Create a `x_stave_dt_data_scramble_definition` with `table`, `fields`, `filter`, `regular_expresssion` (yes, three `s`s), and a `mode` (10/20/30/40).
2. From the form, click **Scramble Data** (requires `x_stave_dt.scramble_user`).
3. The UI action calls `DataScramble_Ajax.getTotalCount()` to preview the affected row count and asks for confirmation.
4. On confirm, the scramble runs and writes one row per execution to `x_stave_dt_data_scramble_log` with `matching_records` / `modified_records`.
5. If `alerts = true`, a `data_mask_alert` row is also produced for each scrambled value, with `data_scramble_definition` set instead of `data_mask_definition`.

### 9.4 Generating synthetic records

1. Create a `x_stave_dt_data_generator` with `table`, `quantity`, `fields`, and (optionally) `set_values`.
2. `set_values` is a JSON object mapping fields to fixed values, e.g.

   ```json
   {"category": "software", "priority": "3"}
   ```
3. Tune distribution: `reference_setting` and `choice_setting` each take `10` (Random) or `20` (Evenly Distributed).
4. Click **Generate Data** (requires `x_stave_dt.generator_user`).
5. `DataToolsHelper.generateData(def)` inspects each target field via `getElement(fields[j]).getED().getInternalType()` and routes to the appropriate population strategy:
   - `string` → from candidate list (`_setupChoices`) or random string (`_getString`)
   - `integer` → from candidate list or random integer (`_getInt`)
   - `reference` → `_setUpReferences` (honors `reference_setting`)
   - `sys_user` target table → branches to `_generateUser`, which calls `api.randomuser.me`.

---

## 10. Extension Points

- **External name service.** The Random-User scramble mode and the synthetic-user generator both hit the URL in `x_stave_dt.name_generator_api`. Override the property to point at an internal mirror if egress is restricted.
- **Reusable utilities.** `x_stave_dt.RegexUtils` and `x_stave_dt.Masker` are `public` script includes — other scoped apps can `new x_stave_dt.Masker().getBool(gr)` as a quick "would any mask rule fire here?" probe without taking on the full alert-creation path of `DataMaskerHelper.maskRecord`.
- **Dictionary-based masking.** `dictionary_table` on a mask definition can point at any table that has a `name` column (not strictly the shipped `x_stave_dt_dictionary_entry`) — useful for reusing existing taxonomy/CMDB lists as match sources.
- **Disabled scaffolding.** `Run Mask Definitions` on `live_message` ships **inactive** — re-enable to extend masking into Connect Chat / live messages.

---

## 11. Appendix — Key Identifiers & File Map

| Artifact                                | sys_id                                  | Source file                                                                                                                                          |
| --------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Application                             | `d0aee0190f62e640c783cd8ce1050e34`      | [sys_app_d0aee0190f62e640c783cd8ce1050e34.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/sys_app_d0aee0190f62e640c783cd8ce1050e34.xml) |
| Application menu                        | `ec13bd5d0fe2e640c783cd8ce1050e72`      | [update/sys_app_application_ec13bd5d0fe2e640c783cd8ce1050e72.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_app_application_ec13bd5d0fe2e640c783cd8ce1050e72.xml) |
| Logo (sys_attachment)                   | `843a35591b69e61020bbeaca234bcb3e`      | (embedded in app manifest)                                                                                                                           |
| Table `x_stave_dt_data_mask_definition` (db_object_id) | `8e6653b31ba635502f41cbf3604bcba8`      | [dictionary/x_stave_dt_data_mask_definition.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_definition.xml) |
| Table `x_stave_dt_data_mask_alert`      | `a666d3b31ba635502f41cbf3604bcb72`      | [dictionary/x_stave_dt_data_mask_alert.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_alert.xml) |
| Table `x_stave_dt_data_mask_replaced`   | `4e6693b31ba635502f41cbf3604bcb5f`      | [dictionary/x_stave_dt_data_mask_replaced.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_mask_replaced.xml) |
| Table `x_stave_dt_data_scramble_definition` | `726617b31ba635502f41cbf3604bcb38`  | [dictionary/x_stave_dt_data_scramble_definition.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_scramble_definition.xml) |
| Table `x_stave_dt_data_scramble_log`    | `166693b31ba635502f41cbf3604bcbb7`      | [dictionary/x_stave_dt_data_scramble_log.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_scramble_log.xml) |
| Table `x_stave_dt_data_generator`       | `9e66d3b31ba635502f41cbf3604bcb10`      | [dictionary/x_stave_dt_data_generator.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_data_generator.xml) |
| Table `x_stave_dt_dictionary_entry`     | `e666d3b31ba635502f41cbf3604bcbd8`      | [dictionary/x_stave_dt_dictionary_entry.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_dictionary_entry.xml) |
| Role `x_stave_dt.admin`                 | `4197e1d50fe2e640c783cd8ce1050eb9`      | [update/sys_user_role_4197e1d50fe2e640c783cd8ce1050eb9.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_user_role_4197e1d50fe2e640c783cd8ce1050eb9.xml) |
| Role `x_stave_dt.user`                  | `0946f85d0f62e640c783cd8ce1050ee8`      | [update/sys_user_role_0946f85d0f62e640c783cd8ce1050ee8.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_user_role_0946f85d0f62e640c783cd8ce1050ee8.xml) |
| Role `x_stave_dt.scramble_user`         | `191609590fa2e640c783cd8ce1050eee`      | [update/sys_user_role_191609590fa2e640c783cd8ce1050eee.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_user_role_191609590fa2e640c783cd8ce1050eee.xml) |
| Role `x_stave_dt.generator_user`        | `9842fc990f62e640c783cd8ce1050eef`      | [update/sys_user_role_9842fc990f62e640c783cd8ce1050eef.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_user_role_9842fc990f62e640c783cd8ce1050eef.xml) |
| Role `x_stave_dt.dictionary_entry_user` | `034d39400f1e5b4055d1cd8ce1050eec`      | [update/sys_user_role_034d39400f1e5b4055d1cd8ce1050eec.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_user_role_034d39400f1e5b4055d1cd8ce1050eec.xml) |
| Script Include `DataMaskerHelper`       | `d43ca9a50f6ae640c783cd8ce1050e59`      | [update/sys_script_include_d43ca9a50f6ae640c783cd8ce1050e59.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_include_d43ca9a50f6ae640c783cd8ce1050e59.xml) |
| Script Include `Masker`                 | `fc125aec1b38981017a6b9dcdd4bcb61`      | [update/sys_script_include_fc125aec1b38981017a6b9dcdd4bcb61.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_include_fc125aec1b38981017a6b9dcdd4bcb61.xml) |
| Script Include `DataToolsHelper`        | `5b5ee9e50f6ae640c783cd8ce1050e08`      | [update/sys_script_include_5b5ee9e50f6ae640c783cd8ce1050e08.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_include_5b5ee9e50f6ae640c783cd8ce1050e08.xml) |
| Script Include `DataScramble_Ajax`      | `f1ada5e50f6ae640c783cd8ce1050e5d`      | [update/sys_script_include_f1ada5e50f6ae640c783cd8ce1050e5d.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_include_f1ada5e50f6ae640c783cd8ce1050e5d.xml) |
| Script Include `RegexUtils`             | `76c4b4104709d1103c8b8a57436d4365`      | [update/sys_script_include_76c4b4104709d1103c8b8a57436d4365.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_include_76c4b4104709d1103c8b8a57436d4365.xml) |
| BR `DataMasker Check`                   | `63cf825d0f26e640c783cd8ce1050e1b`      | [update/sys_script_63cf825d0f26e640c783cd8ce1050e1b.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_script_63cf825d0f26e640c783cd8ce1050e1b.xml) |
| Property `name_generator_api`           | `b50631e90f6ae640c783cd8ce1050e1d`      | [update/sys_properties_b50631e90f6ae640c783cd8ce1050e1d.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_properties_b50631e90f6ae640c783cd8ce1050e1d.xml) |
| Property `logging.verbosity`            | `c075f5e50f6ae640c783cd8ce1050efc`      | [update/sys_properties_c075f5e50f6ae640c783cd8ce1050efc.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_properties_c075f5e50f6ae640c783cd8ce1050efc.xml) |

---

## 12. Version Notes

- **3.2.23** — Current version, packaged 2025-11-21. App manifest `sys_mod_count = 47` cumulative revisions since creation.
- **Created** — 2023-11-23 by `jonas.perusquia`.
- **Code copyright header in script includes** — bears "Copyright (C) 2026 Stave, Inc.", aligned with the file's last modification rather than the original authorship dates (some included scripts go back to 2016).
- **Repository convention** — This app is delivered as ServiceNow-generated XML in [x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/). The repository's [README.md](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/README.md) is a generic ServiceNow recovery guide and is not app-specific.

---

*Document generated from source XML on 2026-05-21 against scope `x_stave_dt` v3.2.23.*
