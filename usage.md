# 9. Usage / How-To

## 9.1 Defining a real-time mask rule (regex)

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

## 9.2 Defining a mask rule (dictionary lookup)

1. Add words/phrases to `x_stave_dt_dictionary_entry` (one row per term).
2. Create a `x_stave_dt_data_mask_definition` with:
   - `dictionary_lookup = true`
   - `dictionary_table = <your dictionary table>` (defaults to `x_stave_dt_dictionary_entry`)
   - Leave `regular_expression` blank — the UI action **Generate Regular Expression** will build one from the dictionary on demand.

## 9.3 Running a scramble job

1. Create a `x_stave_dt_data_scramble_definition` with `table`, `fields`, `filter`, `regular_expresssion` (yes, three `s`s), and a `mode` (10/20/30/40).
2. From the form, click **Scramble Data** (requires `x_stave_dt.scramble_user`).
3. The UI action calls `DataScramble_Ajax.getTotalCount()` to preview the affected row count and asks for confirmation.
4. On confirm, the scramble runs and writes one row per execution to `x_stave_dt_data_scramble_log` with `matching_records` / `modified_records`.
5. If `alerts = true`, a `data_mask_alert` row is also produced for each scrambled value, with `data_scramble_definition` set instead of `data_mask_definition`.

## 9.4 Generating synthetic records

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
