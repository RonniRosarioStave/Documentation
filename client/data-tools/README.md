# Stave Data Tools (`x_stave_dt`) — Documentation

> ServiceNow scoped application that provides a unified toolbox for **data hygiene**: preventing unwanted data from entering the platform (Data Masker), scrubbing existing data in place (Data Scrambler), and generating synthetic records for testing (Data Generator).

---

## 1. At-a-Glance

| Attribute    | Value                  |
| ------------ | ---------------------- |
| **Name**     | Stave Data Tools       |
| **Scope**    | `x_stave_dt`           |
| **Vendor**   | Stave Inc              |
| **Version**  | 3.2.23                 |
| **JS Level** | `es_latest`            |

**Short description:**

> The Stave Data Masker provides the ability to define regular expressions that when matched will mask the data to prevent unwanted data from being entered in the system.
>
> The Stave Data Scrambler provides the ability to define regular expressions and when executed, the matched data will be altered to prevent unwanted data from being kept in the system. Each scrambler definition must be executed manually.

---

## 2. Capabilities Overview

The application is organized around **three pillars** plus one **supporting subsystem**:

| Pillar              | Trigger model        | Operates on             | Purpose                                                                                  |
| ------------------- | -------------------- | ----------------------- | ---------------------------------------------------------------------------------------- |
| **Data Masker**     | Real-time (on record save) | The record being saved  | Block or replace sensitive data **before** it is persisted. Optional alert-only mode.    |
| **Data Scrambler**  | Manual (UI action)   | Existing records in bulk | Walk a result set and overwrite matched characters/fields in place. Logged per execution. |
| **Data Generator**  | Manual (UI action)   | Target table (new rows) | Insert N synthetic records with randomized or evenly-distributed field values.            |
| **Dictionary**      | Configuration support | Mask Definitions       | Provide a curated word list as an alternative to regex pattern matching.                  |

**Real-time mask flow (conceptual):**

1. A user saves a record on a monitored table.
2. The Data Masker evaluates the active mask definitions configured for that table, in priority order.
3. For each match, the app either logs an alert (`mode = Alert`) or replaces the field value (`mode = Replace`) before the record is persisted, recording an audit row when a replacement occurs.

---

## 3. Data Model — Table Reference

The app ships **7 tables**, all prefixed `x_stave_dt_`.

### 3.1 `x_stave_dt_data_mask_definition`

Defines a real-time masking rule.

| Field                     | Type            | Mand. | Hint / Notes                                                                                                |
| ------------------------- | --------------- | :---: | ----------------------------------------------------------------------------------------------------------- |
| `name`                    | string (100)    |   Y   | Name of this data mask definition.                                                                          |
| `description`             | string (1000)   |       | The description of the data mask definition.                                                                |
| `active`                  | boolean         |       | Default `true`. Toggles the rule on/off.                                                                    |
| `mode`                    | string (40)     |   Y   | `Alert` (log only) or `Replace` (overwrite with `replace_text`). Default `Alert`.                          |
| `table`                   | table_name (80) |   Y   | Table that will be monitored by this rule.                                                                  |
| `fields`                  | string (1000)   |   Y   | Comma-separated list of fields on `table` to evaluate.                                                      |
| `filter`                  | conditions (4000) |     | Condition-builder filter scoping which records trigger the rule.                                            |
| `case_sensitive_filter`   | boolean         |       | Apply `filter` case-sensitively.                                                                            |
| `regular_expression`      | string (100000) |   Y   | The regex pattern matched against the specified fields.                                                     |
| `conditions_regex`        | string (2560)   |       | Additional regex applied to conditions.                                                                     |
| `case_sensitive`          | boolean         |       | Default `true`. Apply `regular_expression` case-sensitively.                                                |
| `dictionary_lookup`       | boolean         |       | If `true`, match against dictionary entries (table = `dictionary_table`) instead of regex.                  |
| `dictionary_table`        | table_name (80) |       | The table that holds the dictionary entries for lookup.                                                     |
| `replace_text`            | string (1000)   |       | Text inserted in place of the matched pattern when `mode = Replace`.                                        |
| `first`                   | boolean         |       | Replace **only the first** match in the field.                                                              |
| `last`                    | boolean         |       | Replace **only the last** match in the field.                                                               |
| `exclude_characters`      | integer         |       | Number of characters to skip when masking (e.g. last 4 of an SSN).                                          |
| `include_original_text`   | boolean         |       | Default `true`. If true, original value is stored on the `data_mask_replaced` audit row.                    |
| `order`                   | integer         |       | Execution priority when multiple definitions monitor the same table (lower runs first).                     |

### 3.2 `x_stave_dt_data_mask_alert`

Audit row written whenever a `mode=Alert` mask rule fires (and optionally when scramble rules with `alerts=true` fire).

| Field                       | Type                                      | Read-only | Notes                                                                  |
| --------------------------- | ----------------------------------------- | :-------: | ---------------------------------------------------------------------- |
| `number`                    | string (40)                               |           | Auto-generated.                                                        |
| `user`                      | reference → `sys_user`                    |   Yes     | User whose save triggered the alert.                                   |
| `table`                     | string (40)                               |   Yes     | Source table where the match occurred.                                 |
| `field`                     | string (40)                               |   Yes     | Field on `table` where the match occurred.                             |
| `id`                        | string (40)                               |   Yes     | `sys_id` of the source record.                                         |
| `matched_text`              | string (1000)                             |   Yes     | The substring that matched the pattern.                                |
| `data_mask_definition`      | reference → `x_stave_dt_data_mask_definition` | Yes   | Definition that fired (null if scramble-sourced).                      |
| `data_scramble_definition`  | reference → `x_stave_dt_data_scramble_definition` | Yes | Definition that fired (null if mask-sourced).                          |
| `state`                     | string (40)                               |           | `Pending Review` (default), `Masked`, or `Ignored`.                    |

### 3.3 `x_stave_dt_data_mask_replaced`

Audit row written whenever a `mode=Replace` rule actually overwrites a value.

| Field                  | Type                                                | Notes                                                                            |
| ---------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------- |
| `number`               | string (40)                                         | Auto-generated.                                                                  |
| `user`                 | reference → `sys_user`                              | User whose save triggered the replacement.                                       |
| `data_mask_definition` | reference → `x_stave_dt_data_mask_definition`       | Read-only. Rule that fired.                                                      |
| `table`                | table_name (80)                                     | Read-only. Source table.                                                         |
| `field`                | string (40)                                         | Field that was overwritten.                                                      |
| `id`                   | string (40)                                         | `sys_id` of the source record.                                                   |
| `mode`                 | string (40)                                         | The `mode` of the firing definition (typically `Replace`).                       |
| `original_text`        | string (100)                                        | Default `Not logged`. Populated only when the definition has `include_original_text = true`. |

### 3.4 `x_stave_dt_data_scramble_definition`

Defines a manually-executed bulk scramble.

| Field                  | Type                | Mand. | Notes                                                                                                          |
| ---------------------- | ------------------- | :---: | -------------------------------------------------------------------------------------------------------------- |
| `number`               | string (40)         |       | Auto-generated.                                                                                                |
| `description`          | string (100)        |       | Description of this scramble definition.                                                                       |
| `table`                | table_name (80)     |   Y   | Table whose data will be scrambled.                                                                            |
| `fields`               | string (40000)      |       | Comma-separated list of fields on `table` to scramble.                                                         |
| `filter`               | conditions (4000)   |       | Condition-builder filter.                                                                                      |
| `regular_expresssion`  | string (100000)     |       | Regex matching the substring within each field to scramble.                                                    |
| `case_sensitive`       | boolean             |       | Default `true`.                                                                                                |
| `mode`                 | choice (40)         |       | Scramble mode — see codes below. Default `10` (Xs).                                                            |
| `alerts`               | boolean             |       | If `true`, also write a `data_mask_alert` row for each scrambled value.                                        |

**Scramble mode codes** (stored as the integer string in the `mode` field):

| Code | Label       | Behavior                                                              |
| ---- | ----------- | --------------------------------------------------------------------- |
| `10` | Xs          | Replace every matched character with `x`.                             |
| `20` | Shuffle     | Re-order the characters of the match randomly.                        |
| `30` | Random User | Replace with user data drawn from `api.randomuser.me`.                |
| `40` | Nowlpsum    | Replace with lorem ipsum text.                                        |

### 3.5 `x_stave_dt_data_scramble_log`

Execution log for scramble runs.

| Field                 | Type                                                   | Notes                                                                                |
| --------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `number`              | string (40)                                            | Auto-generated.                                                                      |
| `scramble_definition` | reference → `x_stave_dt_data_scramble_definition`      | Definition that ran.                                                                 |
| `description`         | string (100)                                           | Free-text description.                                                               |
| `record_table`        | table_name (40)                                        | The table that was scrambled.                                                        |
| `record`              | document_id (32)                                       | Reference to a specific scrambled record.                                            |
| `start_date`          | glide_date_time                                        | Run start.                                                                           |
| `end_date`            | glide_date_time                                        | Run end.                                                                             |
| `matching_records`    | integer                                                | Records that matched the definition's filter + regex.                                |
| `modified_records`    | integer                                                | Records actually written.                                                            |

### 3.6 `x_stave_dt_data_generator`

Synthetic data generation definition.

| Field               | Type            | Mand. | Notes                                                                                       |
| ------------------- | --------------- | :---: | ------------------------------------------------------------------------------------------- |
| `number`            | string (40)     |       | Auto-generated.                                                                             |
| `description`       | string (40)     |       | Description of this generator.                                                              |
| `table`             | table_name (80) |   Y   | Table that will receive generated rows.                                                     |
| `quantity`          | integer         |   Y   | How many records to create per execution.                                                   |
| `fields`            | string (1000)   |   Y   | Comma-separated list of fields on `table` to populate.                                      |
| `reference_setting` | choice (40)     |       | `10` Random (default) / `20` Evenly Distributed — controls how reference field values are picked. |
| `choice_setting`    | choice (40)     |       | `10` Random (default) / `20` Evenly Distributed — same, for choice fields.                  |
| `set_values`        | string (4000)   |       | JSON map of `field → fixed_value`. Each key must appear in `fields`.                        |

### 3.7 `x_stave_dt_dictionary_entry`

Custom word/phrase list used by mask definitions when `dictionary_lookup = true`.

| Field            | Type             | Notes                                                       |
| ---------------- | ---------------- | ----------------------------------------------------------- |
| `name`           | string (40)      | Unique. The word/phrase to match.                           |
| `sys_class_name` | sys_class_name (80) | Supports table extension.                                |

An import template is provided for bulk-loading dictionary entries.

---

## 4. Configuration

### 4.1 System Properties

| Property                          | Type        | Default                       | Purpose                                                                                  |
| --------------------------------- | ----------- | ----------------------------- | ---------------------------------------------------------------------------------------- |
| `x_stave_dt.name_generator_api`   | string      | `http://api.randomuser.me/`   | Endpoint used when generating `sys_user` rows or by Random-User scramble mode.           |
| `x_stave_dt.logging.verbosity`    | choicelist  | `debug`                       | Logging level. Choices: `debug`, `info`, `warn`, `error`.                                |

### 4.2 Choice Lists

- `x_stave_dt_data_generator.choice_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_generator.reference_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_mask_alert.state` — Pending Review, Masked, Ignored
- `x_stave_dt_data_mask_definition.mode` — Alert, Replace
- `x_stave_dt_data_scramble_definition.mode` — Xs (`10`), Shuffle (`20`), Random User (`30`), Nowlpsum (`40`)

### 4.3 Navigation

The application menu **Stave Data Tools** groups modules into:

- **Data Management** — Data Mask Definitions (All / Create), Data Mask Alerts, Data Mask Replaced, Data Scramble Definitions, Data Scramble Logs, Data Generators.
- **Configuration** — Dictionary Entries.
- **Utilities** — Getting Started, Data Quarantine, Replacement Log, Import Templates.
- **Support** — About Stave, More Stave Apps, Contact Stave Support.

---

## 5. Usage / How-To

### 5.1 Defining a real-time mask rule (regex)

Example — IP-address masker on the incident table:

```text
Name:               IP Address Monitoring and Mask
Table:              incident
Fields:             short_description
Mode:               Replace
Regular Expression: \b(?:\d{1,3}\.){3}\d{1,3}\b
Replace Text:       #Masked IP Address#
Case sensitive:     true
Order:              5
Active:             false       (enable per environment)
```

When a user saves an incident with `short_description = "Down at 10.0.5.42"`, the field is rewritten to `Down at #Masked IP Address#` **before** the row is persisted, and an audit row is written to `x_stave_dt_data_mask_replaced` with `original_text = "Down at 10.0.5.42"` (because `include_original_text` defaults to `true`).

### 5.2 Defining a mask rule (dictionary lookup)

1. Add words/phrases to `x_stave_dt_dictionary_entry` (one row per term).
2. Create a `x_stave_dt_data_mask_definition` with:
   - `dictionary_lookup = true`
   - `dictionary_table = <your dictionary table>` (defaults to `x_stave_dt_dictionary_entry`)
   - Leave `regular_expression` blank — the **Generate Regular Expression** action will build one from the dictionary on demand.

### 5.3 Running a scramble job

1. Create a `x_stave_dt_data_scramble_definition` with `table`, `fields`, `filter`, `regular_expresssion`, and a `mode` (10/20/30/40).
2. From the form, click **Scramble Data** (requires the Scramble user role).
3. The app previews the affected row count and asks for confirmation.
4. On confirm, the scramble runs and writes one row per execution to `x_stave_dt_data_scramble_log` with `matching_records` / `modified_records`.
5. If `alerts = true`, a `data_mask_alert` row is also produced for each scrambled value.

### 5.4 Generating synthetic records

1. Create a `x_stave_dt_data_generator` with `table`, `quantity`, `fields`, and (optionally) `set_values`.
2. `set_values` is a JSON object mapping fields to fixed values, e.g.

   ```json
   {"category": "software", "priority": "3"}
   ```
3. Tune distribution: `reference_setting` and `choice_setting` each take `10` (Random) or `20` (Evenly Distributed).
4. Click **Generate Data** (requires the Generator user role).

---

## 6. Extension Points

- **External name service.** The Random-User scramble mode and the synthetic-user generator both use the URL in `x_stave_dt.name_generator_api`. Override the property to point at an internal mirror if egress is restricted.
- **Dictionary-based masking.** `dictionary_table` on a mask definition can point at any table that has a `name` column — useful for reusing existing taxonomy/CMDB lists as match sources.

---

## 7. Version Notes

- **3.2.23** — Current version.
- This app is delivered as a ServiceNow scoped application.
