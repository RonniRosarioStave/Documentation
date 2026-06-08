# 7. Security Model

## 7.1 Roles (`sys_user_role`)

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

## 7.2 ACLs

The app ships roughly **130 ACLs** (`sys_security_acl_*.xml`) — broken down as ~56 base table ACLs (read/write/create/delete per table) plus ~74 role-specific overrides. They enforce that:

- Mask/scramble definition CRUD is gated by `x_stave_dt.admin` (or the corresponding `*_user` role for read).
- Alert and Replaced rows are readable by `x_stave_dt.user` and writable by `x_stave_dt.admin`.
- Dictionary entries are readable by `x_stave_dt.dictionary_entry_user`.
