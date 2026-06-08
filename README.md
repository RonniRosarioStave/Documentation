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
