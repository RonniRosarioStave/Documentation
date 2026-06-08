# 4. Data Model — Table Reference

The app ships **7 tables**, all prefixed `x_stave_dt_`. The first three extend `sys_metadata` (i.e. they participate in the Source Control / update-set lifecycle); the rest are plain collections.

## 4.1 `x_stave_dt_data_mask_definition`

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

## 4.2 `x_stave_dt_data_mask_alert`

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

## 4.3 `x_stave_dt_data_mask_replaced`

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

## 4.4 `x_stave_dt_data_scramble_definition`

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

## 4.5 `x_stave_dt_data_scramble_log`

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

## 4.6 `x_stave_dt_data_generator`

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

## 4.7 `x_stave_dt_dictionary_entry`

Custom word/phrase list used by mask definitions when `dictionary_lookup = true`. Source: [dictionary/x_stave_dt_dictionary_entry.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/x_stave_dt_dictionary_entry.xml)

| Field            | Type             | Notes                                                       |
| ---------------- | ---------------- | ----------------------------------------------------------- |
| `name`           | string (40)      | Unique. The word/phrase to match.                           |
| `sys_class_name` | sys_class_name (80) | Default `current.getTableName()`. Supports table extension. |

Indexes: `name` (and a `unique` index on `name`), `sys_class_name`.

The app also ships an import template — [u_imp_tmpl_x_stave_dt_dictionary_entry.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/dictionary/u_imp_tmpl_x_stave_dt_dictionary_entry.xml) — for bulk-loading dictionary entries.
