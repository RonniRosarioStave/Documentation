# 8. Configuration

## 8.1 System Properties

| Property                          | Type        | Default                       | Purpose                                                                                  |
| --------------------------------- | ----------- | ----------------------------- | ---------------------------------------------------------------------------------------- |
| `x_stave_dt.name_generator_api`   | string      | `http://api.randomuser.me/`   | Endpoint used by `DataToolsHelper` when generating `sys_user` rows or by Random-User scramble mode. |
| `x_stave_dt.logging.verbosity`    | choicelist  | `debug`                       | Logging level for all script includes. Choices: `debug`, `info`, `warn`, `error`.       |

A handful of `sys_properties_category_*.xml` records group these under a "Stave Data Tools" properties category for the System Properties UI.

## 8.2 Choice Lists

Choice values for the dictionary fields are defined inline (see [Data Model](data-model.md)), plus the `author_elective_update/` folder ships these author-customized choice lists:

- `x_stave_dt_data_generator.choice_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_generator.reference_setting` — Random (`10`), Evenly Distributed (`20`)
- `x_stave_dt_data_mask_alert.state` — Pending Review, Masked, Ignored
- `x_stave_dt_data_mask_definition.mode` — Alert, Replace
- `x_stave_dt_data_scramble_definition.mode` — Xs (`10`), Shuffle (`20`), Random User (`30`), Nowlpsum (`40`)

## 8.3 Scheduled Jobs

The app ships exactly one schedule record — [cmn_schedule_5056b9f70ffbf28055d1cd8ce1050ec7.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/cmn_schedule_5056b9f70ffbf28055d1cd8ce1050ec7.xml). It is a configuration record (no active recurring scramble or mask job ships out of the box).

## 8.4 Data Source

One `sys_data_source` record ([sys_data_source_39fd71800f1e5b4055d1cd8ce1050ec0.xml](../Stave%20Doc/stave_ai_query/x_stave_dt/d0aee0190f62e640c783cd8ce1050e34/update/sys_data_source_39fd71800f1e5b4055d1cd8ce1050ec0.xml)) backs the dictionary-entry import template.
