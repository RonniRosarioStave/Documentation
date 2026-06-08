# 5. Business Logic — Server-Side

## 5.1 Script Includes

All located in [update/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/).

| Name                | API name                       | Access            | Client-callable | Responsibility                                                                                                                                                  |
| ------------------- | ------------------------------ | ----------------- | :-------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DataMaskerHelper`  | `x_stave_dt.DataMaskerHelper`  | public            |        No       | Core masking engine. `maskRecord(gr, isReturnValue)` iterates active definitions for `gr.getTableName()`, evaluates regex/dictionary, then either alerts or replaces. |
| `Masker`            | `x_stave_dt.Masker`            | public            |        No       | Lighter-weight alternative. `getBool(gr)` returns `true` if any active definition would match the record — useful for short-circuit checks.                       |
| `DataToolsHelper`   | `x_stave_dt.DataToolsHelper`   | public            |        No       | Data Generator engine. `generateData(def)` reads the generator record, types each target field via `getED().getInternalType()`, and inserts `quantity` rows. Special-cases `table == sys_user` (random-user API). |
| `DataScramble_Ajax` | `x_stave_dt.DataScramble_Ajax` | package_private   |       Yes       | Extends `global.AbstractAjaxProcessor`. `getTotalCount()` previews how many records a scramble definition would touch by running a `GlideAggregate(COUNT)` against its filter. |
| `RegexUtils`        | `x_stave_dt.RegexUtils`        | public            |        No       | Plain-object utility. `convertStringToRegex(str)` turns `/pattern/flags` strings into `RegExp`; `flags(rgx)` extracts the trailing flags; helpers for matches & reflag. |

> Each include carries the same copyright header: *Copyright (C) 2026 Stave, Inc. All Rights Reserved — Unauthorized copying ... Proprietary and confidential.*

## 5.2 Business Rules

All located in [update/](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/).

| Name                          | Table                                | When            | Active | Purpose                                                                              |
| ----------------------------- | ------------------------------------ | --------------- | :----: | ------------------------------------------------------------------------------------ |
| `DataMasker Check`            | `task`                               | before insert/update |   Y    | Calls `DataMaskerHelper.maskRecord(current)`. The hot path of the real-time masker.  |
| `Format Fields field in DG`   | `x_stave_dt_data_generator`          | before insert/update |   Y    | Strips all whitespace from the comma-separated `fields` column on save.              |
| `Format Fields field in DMD`  | `x_stave_dt_data_mask_definition`    | before insert/update |   Y    | Same whitespace normalization, on mask definitions.                                  |
| `Format Fields field in DSD`  | `x_stave_dt_data_scramble_definition`| before insert/update |   Y    | Same whitespace normalization, on scramble definitions.                              |
| `Run Mask Definitions`        | `live_message`                       | before insert/update |   N    | Disabled. Would mirror `DataMasker Check` for the Connect Chat / live-message table. |
| `Move comments` *(deleted)*   | `incident`                           | before insert/update |   —    | Tombstone for an old rule that moved comment content; deleted Jan 2026. See [author_elective_update/sys_script_0ac3a16d0f2ae640c783cd8ce1050eb3.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/author_elective_update/sys_script_0ac3a16d0f2ae640c783cd8ce1050eb3.xml). |
