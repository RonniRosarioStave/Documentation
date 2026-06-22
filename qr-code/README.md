# Stave QR Code Generator (`x_stave_qr_code`) — Technical Documentation

> ServiceNow scoped application that **generates and renders QR codes and barcodes** (CODE128, UPC, CODE39, EAN, ITF-14, MSI, Pharmacode, Codabar) tied to any ServiceNow record, table-filter result set, URL, free text, or Service Portal page — with customizable templates, batch generation, list-view printing, email embedding, and a Service Portal widget.

---

## 1. At-a-Glance

| Attribute              | Value                                                                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**               | Stave QR Code Generator                                                                                                                              |
| **Scope**              | `x_stave_qr_code`                                                                                                                                    |
| **Application sys_id** | `5761e35a0f3c0600c783cd8ce1050e37`                                                                                                                   |
| **Vendor**             | Stave (Stave Apps)                                                                                                                                   |
| **Version**            | 4.3.17                                                                                                                                               |
| **JS Level**           | `es_latest`                                                                                                                                          |
| **Created**            | 2015-06-24 by `martin.palacios`                                                                                                                      |
| **Last Updated**       | 2025-11-24 by `jonas@test.com`                                                                                                                       |
| **Store ID**           | `REPOAPP0002350529` (TPP appstore)                                                                                                                   |
| **Default user role**  | `x_stave_qr_code.user`                                                                                                                               |
| **Source manifest**    | sys_app_5761e35a0f3c0600c783cd8ce1050e37.xml |

**Short description (from manifest):** *Generate and maintain QR and Bar Codes.*

---

## 2. Capabilities Overview

The app is organized around **four pillars**:

| Pillar                | Trigger model           | Driven by                                                                       | Output                                                                                  |
| --------------------- | ----------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **QR Code Generator** | Per-record manual create | `x_stave_qr_code_stave_qr_code` rows                                            | One QR per record; rendered via `x_stave_qr_code_stave_qr_display.do` processor + SP widget. |
| **QR Code Batch**     | List-filter manual run  | `x_stave_qr_code_stave_qr_code_batch` rows + **Generate Batch Codes** UI action | Bulk QR generation against a filter on any table.                                       |
| **Barcode Generator** | Per-record manual create | `x_stave_qr_code_stave_bar_code` rows                                           | One barcode (CODE128/UPC/CODE39/etc.) per record; rendered via `x_stave_qr_code_stave_barcode_display.do`. |
| **Barcode Batch**     | List-filter manual run  | `x_stave_qr_code_stave_barcode_batch` rows + **Generate Batch Codes** UI action | Bulk barcode generation.                                                                |

**Supporting subsystem** — **QR Templates** (`x_stave_qr_code_qr_template` + `qr_template_content`) — define presentation: CSS, colors, logo scale, HTML wrappers, and field interpolation tokens like `field:assigned_to:field`.

---

## 3. Architecture

```
                ┌──────────────────────────────────────────────────────────────┐
                │                Stave QR Code Generator                      │
                └──────────────────────────────────────────────────────────────┘

  ┌──────── Definition tables ────────┐    ┌── Server engines ──┐   ┌─── Renderers / Outputs ───┐

  stave_qr_code       ─────────────────►   QRCodeHelper          ──► x_stave_qr_code_stave_qr_display.do
  stave_qr_code_batch ──► Generate ───►   QRCodeHelperAjax        │   (processor URL — server-rendered HTML/QR)
                          Batch Codes  ──► BatchGenerationAjax ──►
                          UI action                                ── SP widget "QR Code"
  stave_bar_code      ─────────────────►   BarcodeHelper         ──► x_stave_qr_code_stave_barcode_display.do
  stave_barcode_batch ──► Generate ───►   BatchGenerationAjax   ──►  (processor URL with JsBarcode rendering)
                          Batch Codes
                          UI action

  qr_template         ──► template style + custom_style + text_script (sandbox-evaluated)
  qr_template_content ──► HTML/CSS blocks per "type" (QR Code, HTML, Formatted HTML, Code Name)
                          with field:fieldname:field interpolation tokens
```

**Per-record QR flow:**
1. Create a `stave_qr_code` record pointing at a table + `document` (or a URL / free text), pick a `template`, pick a Service Portal `portal` + `page`.
2. The display processor URL `x_stave_qr_code_stave_qr_display.do?sysparm_code_id=<sys_id>` renders the QR (combining `QRCodeHelper` server-side templating with `AwesomeQR` / `qrcode.js` client-side drawing inside the SP widget).
3. Embed in emails via the shipped mail script `qr_code_in_email`, or list-print via the **View Codes** UI action.

**Batch flow:**
1. Create a `stave_qr_code_batch` or `stave_barcode_batch` definition with `table_name` + condition-builder `filter` + `template`.
2. Click **Generate Batch Codes** on the form. The client-side `showBatchGenerationConfirm()` calls `BatchGenerationAjax.getTotalCount()` to preview the affected row count, then on confirm spawns one definition row per matching record.

---

## 4. Data Model — Table Reference

The app ships **6 tables**, all prefixed `x_stave_qr_code_`. None extend `sys_metadata` — these are plain `collection` tables (transactional data, not configuration metadata).

### 4.1 `x_stave_qr_code_stave_qr_code`

Per-record QR definition. Source: dictionary/x_stave_qr_code_stave_qr_code.xml

| Field                    | Type                                                            | Mand. | Notes                                                                                                  |
| ------------------------ | --------------------------------------------------------------- | :---: | ------------------------------------------------------------------------------------------------------ |
| `name`                   | string (100), table_reference                                   |   Y   | Display name of this code.                                                                             |
| `active`                 | boolean                                                         |       | Default `true`.                                                                                        |
| `description`            | string (1000)                                                   |       |                                                                                                        |
| `type`                   | choice (40)                                                     |   Y   | `URL` (default), `Record`, `Service Portal Record`, `text`. Drives which other field is interpreted.   |
| `url`                    | url (1024)                                                      |       | Used when `type = URL`.                                                                                |
| `text`                   | string (384), edge-encryption-enabled                           |       | Used when `type = text`.                                                                               |
| `table`                  | string (40)                                                     |       | Source table when `type = Record` / `Service Portal Record`.                                           |
| `document`               | document_id (32), dependent on `table`                          |       | The specific record the code points at.                                                                |
| `view`                   | string (40)                                                     |       | UI view name to honor when rendering / linking.                                                        |
| `template`               | reference → `x_stave_qr_code_qr_template`                       |   Y   | Visual template (background, border, custom CSS, text-script).                                          |
| `portal`                 | reference → `sp_portal`                                         |   Y   | Service Portal context used when `type = Service Portal Record`.                                       |
| `page`                   | reference → `sp_page`                                           |   Y   | The Service Portal page the code links to.                                                             |
| `catalog`                | reference → `sc_catalog`                                        |       | Optional catalog scoping.                                                                              |
| `catalog_view`           | string (40)                                                     |       | Default `catalog_default`. View shown when the code lands on a catalog item.                           |
| `external_url`           | boolean                                                         |       | If `true`, treat `url` as a full external URL rather than an instance-relative path.                   |
| `logo`                   | reference → `db_image`                                          |       | Logo overlay (no UTF-8 encoding of the sys_id).                                                        |
| `not_available_message`  | string (1000)                                                   |       | Shown when the QR cannot be rendered (e.g. record deleted, valid_to passed).                           |
| `valid_to`               | glide_date_time, dynamic default = `2100-01-01 23:59:59`        |       | Expiration cutoff.                                                                                     |

Indexes on `catalog`, `document`, `logo`, `page`, `portal`, `template`.

### 4.2 `x_stave_qr_code_stave_qr_code_batch`

Bulk QR generator. Source: dictionary/x_stave_qr_code_stave_qr_code_batch.xml

| Field                   | Type                                            | Mand. | Notes                                                                            |
| ----------------------- | ----------------------------------------------- | :---: | -------------------------------------------------------------------------------- |
| `number`                | string (40)                                     |       | Auto-generated via `getNextObjNumberPadded()`.                                   |
| `name`                  | string (100)                                    |   Y   |                                                                                  |
| `active`                | boolean                                         |       | Default `true`.                                                                  |
| `description`           | string (1000)                                   |       |                                                                                  |
| `table_name`            | table_name (80)                                 |   Y   | Target table to walk.                                                            |
| `filter`                | conditions (4000), dependent on `table_name`    |       | V2 condition builder, `show_condition_count=true`.                               |
| `type`                  | choice (40)                                     |   Y   | `Record` (default), `text`, `Service Portal Record`.                             |
| `template`              | reference → `x_stave_qr_code_qr_template`       |   Y   |                                                                                  |
| `portal` / `page`       | reference → `sp_portal` / `sp_page`             |       | SP context for generated children.                                               |
| `logo`                  | reference → `db_image`                          |       |                                                                                  |
| `view`                  | string (40)                                     |       |                                                                                  |
| `not_available_message` | string (1000)                                   |       |                                                                                  |
| `valid_to`              | glide_date_time, default `2100-01-01 16:59:59`  |       | Stamped onto every child row.                                                    |

Indexes on `logo`, `page`, `portal`, `template`.

### 4.3 `x_stave_qr_code_stave_bar_code`

Per-record barcode definition. Source: dictionary/x_stave_qr_code_stave_bar_code.xml

| Field            | Type                                                | Mand. | Notes                                                                              |
| ---------------- | --------------------------------------------------- | :---: | ---------------------------------------------------------------------------------- |
| `name`           | string (100), edge-encryption-enabled               |   Y   |                                                                                    |
| `active`         | boolean                                             |       |                                                                                    |
| `description`   | string (1000), edge-encryption-enabled              |       |                                                                                    |
| `barcode_types`  | string (40), edge-encryption-enabled                |   Y   | Symbology — see codes below. Default `1` (CODE128).                                |
| `type`           | string (40), edge-encryption-enabled                |   Y   | `1` Record (default). `2` Service Portal Record is shipped **inactive**.           |
| `table`          | table_name (40), edge-encryption-enabled            |       |                                                                                    |
| `document`       | document_id (32), dependent on `table`              |   Y   |                                                                                    |
| `field`          | field_name (40), dependent on `table`, `allow_null=false` | Y | The field on `document` whose value becomes the barcode payload.                   |
| `template`       | reference → `x_stave_qr_code_qr_template`           |   Y   | Reuses the QR template table for styling wrappers.                                 |
| `portal` / `page`| reference → `sp_portal` / `sp_page`                 |       |                                                                                    |
| `view`           | string (128), edge-encryption-enabled               |       |                                                                                    |

**Barcode symbology codes** (`barcode_types`):

| Code | Symbology  | Code | Symbology  |
| ---- | ---------- | ---- | ---------- |
| `1`  | CODE128    | `5`  | MSI        |
| `2`  | UPC        | `6`  | Pharmacode |
| `3`  | CODE39     | `7`  | Codabar    |
| `4`  | ITF-14     | `8`  | EAN        |

Indexes on `document`, `page`, `portal`, `template`.

### 4.4 `x_stave_qr_code_stave_barcode_batch`

Bulk barcode generator. Source: dictionary/x_stave_qr_code_stave_barcode_batch.xml

| Field        | Type                                            | Mand. | Notes                                                                            |
| ------------ | ----------------------------------------------- | :---: | -------------------------------------------------------------------------------- |
| `number`     | string (40)                                     |       | Auto-generated.                                                                  |
| `name`       | string (40), edge-encryption-enabled            |   Y   |                                                                                  |
| `description`| string (1000), edge-encryption-enabled          |       |                                                                                  |
| `barcode`    | string (40), edge-encryption-enabled            |   Y   | Symbology (same `1`-`8` codes as §4.3). Default `1` (CODE128).                   |
| `type`       | string (40), edge-encryption-enabled            |   Y   | `1` Record (only active option).                                                 |
| `table_name` | table_name (80), edge-encryption-enabled        |   Y   |                                                                                  |
| `filter`     | conditions (4000), dependent on `table_name`    |       | V2 condition builder.                                                            |
| `field`      | field_name (40), dependent on `table_name`      |   Y   | Field used as barcode payload for every matching record.                         |
| `document`   | document_id (32), dependent on `table_name`     |   Y   | (Marked `active="false"` in dictionary — present for legacy compatibility.)      |
| `page` / `portal`| reference → `sp_page` / `sp_portal`         |       |                                                                                  |
| `view`       | string (128), edge-encryption-enabled           |       |                                                                                  |

Indexes on `document`, `page`, `portal`.

### 4.5 `x_stave_qr_code_qr_template`

Visual template applied to QR codes and barcodes. Source: dictionary/x_stave_qr_code_qr_template.xml

| Field          | Type                                          | Mand. | Notes                                                                                                                                                |
| -------------- | --------------------------------------------- | :---: | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`         | string (40), display=true                     |   Y   |                                                                                                                                                      |
| `background`   | string (100)                                  |       | CSS `background` value.                                                                                                                              |
| `border`       | string (100)                                  |       | CSS `border` value.                                                                                                                                  |
| `margin`       | string (100)                                  |       | CSS `margin`.                                                                                                                                        |
| `padding`      | string (100)                                  |       | CSS `padding`.                                                                                                                                       |
| `custom_style` | string (1000)                                 |       | Free-form additional CSS appended to the wrapper.                                                                                                    |
| `is_text`      | boolean, edge-encryption-enabled              |       | If `true`, the template is used with `type = text` QR codes — enables the "text replace" business rule.                                              |
| `text`         | string (384), edge-encryption-enabled         |       | Static text payload.                                                                                                                                 |
| `text_script`  | script (8000), edge-encryption-enabled        |       | Server-side script executed with `current` as the code record. Returns a `variables` object that `current.text` placeholders can reference. Default scaffolding: `(function(current){ var variables = { example: function() { return 'Hello world'; } }; return variables; })(current)` |

### 4.6 `x_stave_qr_code_qr_template_content`

Layout fragments composed under a template. Source: dictionary/x_stave_qr_code_qr_template_content.xml

| Field            | Type                              | Mand. | Notes                                                                                                            |
| ---------------- | --------------------------------- | :---: | ---------------------------------------------------------------------------------------------------------------- |
| `qr_template`    | reference → `x_stave_qr_code_qr_template` |  Y  | Parent template.                                                                                                 |
| `type`           | choice (40)                       |   Y   | `x_stave_qr_code_template_code` (QR Code), `_html` (HTML), `_formatted_html` (Formatted HTML), `_name` (Code Name). |
| `order`          | integer                           |       | Composition order within the template.                                                                           |
| `width`          | integer (default `100`)           |       | QR / image pixel width.                                                                                          |
| `height`         | integer (default `100`)           |       | QR / image pixel height.                                                                                         |
| `color_dark`     | string (40), default `#000000`    |       | QR foreground color.                                                                                             |
| `color_light`    | string (40), default `white`      |       | QR background color.                                                                                             |
| `logo_scale`     | float, default `0.2`              |       | Logo overlay size (0.1 - 0.9 — enforced by client script, see §6.2).                                             |
| `html`           | string (10000)                    |       | Raw HTML block. Supports `field:field_name:field` tokens that interpolate values from the linked record (e.g. `field:assigned_to:field`). |
| `formatted_html` | html (8000)                       |       | WYSIWYG-rendered HTML block.                                                                                     |
| `custom_style`   | string (1000)                     |       | Per-content CSS overrides.                                                                                       |

Index on `qr_template`.

---

## 5. Business Logic — Server-Side

### 5.1 Script Includes

All located in update/.

| Name                  | API name                              | Access            | Client-callable | Responsibility                                                                                                                                                       |
| --------------------- | ------------------------------------- | ----------------- | :-------------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `QRCodeHelper`        | `x_stave_qr_code.QRCodeHelper`        | package_private   |        No       | Core QR rendering helper. `getTemplateStyle(template)` composes inline CSS from template fields; `getContentByType(gr, type)` returns the encoded payload (URL / record-link / text); `getContentAndFields(code, fields)` returns interpolated field values for the widget. |
| `QRCodeHelperAjax`    | `x_stave_qr_code.QRCodeHelperAjax`    | package_private   |       Yes       | `AbstractAjaxProcessor` wrapper around `QRCodeHelper`. Methods: `getType(sysparm_qrcode)` and `getValues(code, fields)`. Used by the SP widget.                       |
| `BarcodeHelper`       | `x_stave_qr_code.BarcodeHelper`       | package_private   |        No       | Barcode equivalent of `QRCodeHelper`. `getTemplateStyle(template)` builds CSS; symbology rendering happens client-side via `JsBarcode` in the display processor page. |
| `BatchGenerationAjax` | `x_stave_qr_code.BatchGenerationAjax` | package_private   |       Yes       | `AbstractAjaxProcessor` for the **Generate Batch Codes** UI action. `getTotalCount()` previews the affected row count by running `GlideAggregate(COUNT)` against the batch's `filter`. Includes a security check that rejects calls where `gs.getUserID()` ≠ the requesting user. |

All four carry the same proprietary header: *Copyright (C) 2017–2021 Stave, Inc. All Rights Reserved — Unauthorized copying ... Proprietary and confidential.*

### 5.2 Business Rules

| Name                          | Table                                | When                  | Active | Purpose                                                                                                                                  |
| ----------------------------- | ------------------------------------ | --------------------- | :----: | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `Stave QR Code Text Replace`  | `x_stave_qr_code_stave_qr_code`      | before insert/update  |   Y    | Runs when `type=text` and `template.is_text=true`. Resolves `text_script` variables and substitutes placeholders in `current.text`.       |
| `freemium`                    | `x_stave_qr_code_qr_template`        | before insert/update  |   N    | Shipped disabled. Aborts the insert/update and displays a "demo version — please contact us" message linking to staveapps.com.            |

### 5.3 Mail Script

| Name              | Source                                                                                                                                  |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `qr_code_in_email`| sys_script_email_ab66eab90f5ea240c783cd8ce1050ed4.xml — embeds a QR into outbound email as an `<img>` tag pointing at `https://<instance_name>.service-now.com/x_stave_qr_code_stave_qr_display.do?sysparm_code_id=<current.sys_id>`. Use in notification templates with `${mail_script:qr_code_in_email}`. |

---

## 6. UI Layer

### 6.1 UI Actions

| Name (display)         | action_name                    | Table                                                                | Where                  | Behavior                                                                                                                                                                                       |
| ---------------------- | ------------------------------ | -------------------------------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Generate Batch Codes** | `qr_generate_batch_codes`     | `x_stave_qr_code_stave_qr_code_batch`                                | Form button            | Calls `showBatchGenerationConfirm()` → `BatchGenerationAjax.getTotalCount()` → confirms → executes batch creation.                                                                              |
| **Generate Batch Codes** | `barcode_generate_batch_codes`| `x_stave_qr_code_stave_barcode_batch`                                | Form button            | Same flow as above, for barcodes.                                                                                                                                                              |
| **Test Link**          | `qr-test-link`                | `x_stave_qr_code_stave_qr_code`                                      | Form button            | Opens the QR's resolved target URL in a new window (for QA / validation).                                                                                                                       |
| **View Codes**         | `qr_view_codes`               | `x_stave_qr_code_stave_qr_code`                                      | List action + List choice | Opens `x_stave_qr_code_stave_qr_display.do?sysparm_print=true&sysparm_code_id=<selected sys_ids>&sysparm_query=<list query>` — printable multi-QR page for the selected rows.                  |
| **View Codes**         | `barcode_view_codes`          | `x_stave_qr_code_stave_bar_code`                                     | List action + List choice | Same as above for barcodes — opens `x_stave_qr_code_stave_barcode_display.do?sysparm_print=true&sysparm_code_id=<selected sys_ids>`.                                                            |

UI action roles are wired via separate `sys_ui_action_role_*` records (~5) — all gate to `x_stave_qr_code.user`.

### 6.2 Client Scripts

| Name                     | Table                                  | Type     | Active | Purpose                                                                                                              |
| ------------------------ | -------------------------------------- | -------- | :----: | -------------------------------------------------------------------------------------------------------------------- |
| `Logo Scale validation`  | `x_stave_qr_code_qr_template_content`  | onSubmit |   Y    | Rejects submission if `logo_scale` ≤ 0 or ≥ 1. Field message: "Value must be between 0.1 and 0.9 inclusive".          |
| *(unnamed)*              | `x_stave_qr_code_qr_template_content`  | onChange (`logo_scale`) |   Y    | Live validation of `logo_scale` with the same range; clears the field message on valid values.                       |
| `freemium`               | `x_stave_qr_code_qr_template`          | onSubmit |   N    | Shipped disabled. Pop-up "limited version" alert that blocks new template creation in demo builds.                    |

### 6.3 UI Policies

The app ships **~18 UI policies** (plus ~40 corresponding action records) that toggle field visibility based on related fields. Examples:

- On `stave_qr_code`: hide `url` unless `type=URL`; hide `text` unless `type=text`; show `page`/`portal` when `type=Service Portal Record`.
- On `stave_bar_code` / `stave_barcode_batch`: hide `document` until `table_name` is set.
- On `qr_template_content`: hide `html` / `formatted_html` based on `type`.

See update/sys_ui_policy_*.xml for the full list.

### 6.4 App Modules (Navigation)

The application menu **Stave QR Code Generator** (`2f616f5a0f3c0600c783cd8ce1050ea5`) groups ~21 modules into:

- **QR Codes** — All QR Codes, Create new QR Code, All QR Code Batches, Create new QR Code Batch.
- **Barcodes** — All Barcodes, Create new Barcode, All Barcode Batches, Create new Barcode Batch.
- **Configure** — All QR Templates, New QR Template.
- **Administration** — Script Includes, UI Macros.
- **Support** — Getting Started, About Stave, More Stave Apps, Contact Stave Support, Privacy Policy.

---

## 7. Service Portal Integration

The app ships a complete Service Portal stack for rendering QR codes inside portal pages.

### 7.1 Widget

| Attribute        | Value                                                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name / ID**    | `QR Code` / `bc9c2730db8650101645146139961958`                                                                                                                                              |
| **Roles**        | `x_stave_qr_code.user`                                                                                                                                                                      |
| **Option schema**| `qr_size` (integer, default `150`, "QR code size") and `show_text` (boolean, default `false`, "Show Text") — both in the *Presentation* section.                                            |
| **Server**       | Pulls QR rows by either `qr_codes` (comma-separated sys_ids) or `table + sys_id`. Builds `{src, uniqueValue, type, content, name}` objects via `QRCodeHelper.getContentByType()`.            |
| **Client**       | AngularJS controller. For each code, instantiates `new AwesomeQR.AwesomeQR({text, size, colorDark, colorLight, margin, components: {...}}).draw()` and injects the resulting data-URL `<img>` into the container `#{uniqueValue}`. Watches `x_stave_qr_code_stave_qr_code` via `spUtil.recordWatch` and re-renders on change. |
| **Template**     | Grid/List toggle (`QRshowGrid()` / `QRshowList()`) plus a Print button (`window.print()`).                                                                                                  |
| **Source**       | update/sp_widget_bc9c2730db8650101645146139961958.xml |

### 7.2 Dependencies

| Artifact       | Name                            | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| sp_dependency  | **Stave QR Code dependencies**  | Aggregates the JS includes below. Bound to the QR Code widget. sp_dependency_91362bc7db8e18101645146139961935.xml |
| sp_js_include  | `x_stave_qr_code.qrcode`        | The local `AwesomeQR.qrcode` library used by the widget's client script. Backed by `sys_ui_script fffea87a0fbc0600c783cd8ce1050ec5`.                                                                                                                                                                                                                                                                                                 |
| sp_js_include  | `x_stave_qr_code.jquery.print`  | Local jQuery print plugin. Backed by `sys_ui_script f9d16e760f700200c783cd8ce1050e12`.                                                                                                                                                                                                                                                                                                                                               |
| sp_css         | `qr_code_style`                 | Print-friendly stylesheet (`@media print { img { display: block !important; visibility: visible !important; } }` + `@page { margin: 0 }`). sp_css_07e0c537dbecb45094ea6693ca961962.xml |
| sp_css_include | `qr_code_style`                 | CSS include linkage to the widget.                                                                                                                                                                                                                                                                                                                                                                                                   |
| content_css    | `qr_code_style`                 | Same stylesheet exposed for native ServiceNow content rendering (non-portal display processor pages).                                                                                                                                                                                                                                                                                                                                |

M2M binding records that join widget → dependency → JS includes:
- m2m_sp_dependency_js_include_3cd62b4bdb8e181016451461399619b2.xml
- m2m_sp_dependency_js_include_fee66b4bdb8e181016451461399619f7.xml
- m2m_sp_widget_dependency_13f6af0bdb8e18101645146139961921.xml

---

## 8. Security Model

### 8.1 Roles

| Role                  | Description                                                                            |
| --------------------- | -------------------------------------------------------------------------------------- |
| `x_stave_qr_code.user`| Single role shipped with the app. Description is empty in source; effectively the "everything" role for the application — referenced by every UI action role binding, the widget's `roles`, and the `sys_app_application.roles` field. Defined in sys_user_role_a3616f5a0f3c0600c783cd8ce1050ea6.xml. |

### 8.2 ACLs

The app ships **~88 ACL records** (`sys_security_acl_*.xml` = ~47 base ACLs + `sys_security_acl_role_*.xml` = ~41 role bindings). They enforce that:

- Read/Write/Create/Delete on the 6 application tables is gated by `x_stave_qr_code.user`.
- The display-processor pages and SP widget pull data via `GlideRecordSecure`, so client-rendered QRs respect the same ACL surface.
- One ACL pattern (`sys_security_acl_bc91c2a5f334621048dff*`) is a fan-out set generated for fine-grained field permissions on `qr_template_content`.

---

## 9. Configuration

### 9.1 System Properties

| Property                              | Type        | Default | Choices                                          | Purpose                                                                       |
| ------------------------------------- | ----------- | ------- | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| `x_stave_qr_code.logging.verbosity`   | choicelist  | `debug` | `Debug=debug`, `Info=info`, `Warn=warn`, `Error=error` | Logging level honored by `gs.debug` / `gs.info` calls throughout the helpers. |

This is the *only* property the app defines.

### 9.2 Choice Lists

Inline choices in dictionaries (§4) plus standalone choice records in author_elective_update/:

- `x_stave_qr_code_qr_template_content.correct_level` — QR error-correction level choices.
- `x_stave_qr_code_qr_template_content.type` — QR Code / HTML / Formatted HTML / Code Name.
- `x_stave_qr_code_stave_bar_code.barcode_types` — the 8 barcode symbologies listed in §4.3.
- `x_stave_qr_code_stave_bar_code.type` — Record (1) / Service Portal Record (2, inactive).
- `x_stave_qr_code_stave_barcode_batch.barcode` — same 8 symbologies as bar_code.
- `x_stave_qr_code_stave_barcode_batch.type` — Record (1) only.
- `x_stave_qr_code_stave_qr_code_batch.type` — Record (default), text, Service Portal Record.
- `x_stave_qr_code_stave_qr_code.type` — URL (default), Record, Service Portal Record, text.

### 9.3 Demo Data

The unload.demo/ folder ships **~28 `sys_metadata_link` records** that wire demo QR codes / barcodes / templates / portal pages to the application. These are imported only when the "Install demo data" flag is set during app installation and exist mainly to give first-time users working examples to inspect. They reference Service Portal pages, sample portal records, and demo `stave_qr_code` entries.

### 9.4 Scheduled Jobs

None. The app has no `sysauto_script` or `cmn_schedule` records — all code/barcode generation is **on-demand** (per-record save or list-action UI button).

### 9.5 REST / Processors

The display URLs `x_stave_qr_code_stave_qr_display.do` and `x_stave_qr_code_stave_barcode_display.do` are referenced throughout (mail script, UI actions, widget). They are ServiceNow display processors (server-side JSP-style endpoints) shipped via the UI Pages / Processor framework. No `sys_ws_*` / `sys_processor_*` records are bundled with the app — these processor URLs resolve to UI pages registered under the same scope.

---

## 10. Usage / How-To

### 10.1 Generate a QR for one record

1. Navigate to **Stave QR Code Generator → Create new QR Code**.
2. Set:
   - `Name` — display label.
   - `Type` — `Record` (link to a ServiceNow record), `Service Portal Record` (link to a portal page rendering the record), `URL` (free URL), or `text`.
   - For `Record`: set `Table` then pick `Document`.
   - For `Service Portal Record`: set `Table`, `Document`, plus `Portal` + `Page`.
   - For `URL`: set `URL` and optionally `External URL = true`.
   - For `text`: set `Text`.
3. Pick a `Template` (or create one — see §10.4).
4. Save. The display URL is now `x_stave_qr_code_stave_qr_display.do?sysparm_code_id=<sys_id>`.
5. Click **Test Link** to verify the target opens correctly in a new window.

### 10.2 Generate QR codes in bulk

1. Navigate to **Stave QR Code Generator → Create new QR Code Batch**.
2. Set `Table Name`, build a `Filter` (condition builder), and pick `Type` + `Template`.
3. Click **Generate Batch Codes**. The client UI action calls `BatchGenerationAjax.getTotalCount()` to preview affected rows and asks for confirmation.
4. On confirm, one `stave_qr_code` row is created per matching record, each inheriting `template`, `portal`, `page`, `valid_to`, and `not_available_message` from the batch.

### 10.3 Print or view multiple codes

From a list view of QR codes or barcodes, select rows and click **View Codes** in the list actions menu. A new tab opens with `?sysparm_print=true` and renders the selected codes for printing (grid or list toggle in the widget; `window.print()` button on the page).

### 10.4 Define a template with dynamic content

1. Create an `x_stave_qr_code_qr_template` row with `Name`, optional CSS (`background`, `border`, `margin`, `padding`, `custom_style`).
2. (Optional) Set `Is text = true` and write a `Text Script` like:
   ```js
   (function(current){
     var variables = {
       assignee: function() {
         return current.document.assigned_to.getDisplayValue();
       }
     };
     return variables;
   })(current)
   ```
3. Add one or more `qr_template_content` rows (related list on the template) to compose the layout:
   - `Type = QR Code` to inject the QR itself (`width`, `height`, `color_dark`, `color_light`, `logo_scale`).
   - `Type = HTML` to inject arbitrary HTML supporting `field:<field_name>:field` interpolation tokens — e.g. `Owner: field:assigned_to:field` resolves to the assigned_to display value of the linked record.
   - `Type = Formatted HTML` for WYSIWYG content.
   - `Type = Code Name` to print the code's name as a caption.

### 10.5 Embed a QR in an outbound email

Use the shipped mail script in a notification template: `${mail_script:qr_code_in_email}`. It expands to an `<img>` tag pointing at the display processor URL for `current.sys_id`. The receiving mail client just renders an inline image.

### 10.6 Add the widget to a Service Portal page

1. In Service Portal Designer, drop the **QR Code** widget onto a page.
2. Configure widget options:
   - `qr_size` — pixel size of each QR (default 150).
   - `show_text` — render the code's `name` as a caption.
3. Pass parameters via the page URL or instance config:
   - `qr_codes` — comma-separated `sys_id`s to render.
   - `table` + `sys_id` — render all QR codes linked to a given record.

---

## 11. Extension Points

- **Template scripting.** `qr_template.text_script` is sandbox-evaluated server-side with `current` set to the QR code row. Use it for record-driven text payloads (e.g. embedding a signed token, joining multiple fields). Return a `variables` object whose keys can be referenced as placeholders.
- **Field interpolation.** Any `qr_template_content.html` block supports `field:<field_name>:field` — handy for `Owner: field:assigned_to:field`, `Asset: field:asset_tag:field`, etc.
- **Display processors.** The shipped processor URLs (`x_stave_qr_code_stave_qr_display.do`, `x_stave_qr_code_stave_barcode_display.do`) accept `sysparm_code_id`, `sysparm_print`, and `sysparm_query` — any external system can request a rendered QR/barcode image by hitting these URLs with an authenticated session.
- **Widget reuse.** The `QR Code` widget can be added to any portal page; combine with `recordWatch` for live updates when the underlying `stave_qr_code` row changes.
- **Barcode symbology.** Adding a new symbology code requires updating the `barcode_types` / `barcode` choice list AND the `JsBarcode` rendering path inside the barcode display processor UI page.

---

## 12. Appendix — Key Identifiers & File Map

| Artifact                                     | sys_id                                  | Source file                                                                                                                                                                |
| -------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Application                                  | `5761e35a0f3c0600c783cd8ce1050e37`      | sys_app_5761e35a0f3c0600c783cd8ce1050e37.xml |
| Application menu                             | `2f616f5a0f3c0600c783cd8ce1050ea5`      | update/sys_app_application_2f616f5a0f3c0600c783cd8ce1050ea5.xml |
| Role `x_stave_qr_code.user`                  | `a3616f5a0f3c0600c783cd8ce1050ea6`      | update/sys_user_role_a3616f5a0f3c0600c783cd8ce1050ea6.xml |
| Table `x_stave_qr_code_stave_qr_code`        | db_object_id `9761e35a0f3c0600c783cd8ce1050e37` | dictionary/x_stave_qr_code_stave_qr_code.xml |
| Table `x_stave_qr_code_stave_qr_code_batch`  | db_object_id `c4efb1990f95ea40c783cd8ce1050ef1` | dictionary/x_stave_qr_code_stave_qr_code_batch.xml |
| Table `x_stave_qr_code_stave_bar_code`       | db_object_id `295588dc0f4c7a00c783cd8ce1050e6b` | dictionary/x_stave_qr_code_stave_bar_code.xml |
| Table `x_stave_qr_code_stave_barcode_batch`  | db_object_id `aa6732650fd8b60055d1cd8ce1050e97` | dictionary/x_stave_qr_code_stave_barcode_batch.xml |
| Table `x_stave_qr_code_qr_template`          | db_object_id `b56a2bda0f3c0600c783cd8ce1050ede` | dictionary/x_stave_qr_code_qr_template.xml |
| Table `x_stave_qr_code_qr_template_content`  | db_object_id `a31bebda0f3c0600c783cd8ce1050ec1` | dictionary/x_stave_qr_code_qr_template_content.xml |
| Script Include `QRCodeHelper`                | `5a0b068b0fb00200c783cd8ce1050ed5`      | update/sys_script_include_5a0b068b0fb00200c783cd8ce1050ed5.xml |
| Script Include `QRCodeHelperAjax`            | `432edca4dba3ec10f24c5e25ca961987`      | update/sys_script_include_432edca4dba3ec10f24c5e25ca961987.xml |
| Script Include `BarcodeHelper`               | `39250ebe0fc47e00c783cd8ce1050eac`      | update/sys_script_include_39250ebe0fc47e00c783cd8ce1050eac.xml |
| Script Include `BatchGenerationAjax`         | `5abaaabb0f1d2200c783cd8ce1050ec1`      | update/sys_script_include_5abaaabb0f1d2200c783cd8ce1050ec1.xml |
| BR `Stave QR Code Text Replace`              | `4cbb3f8bdbce18101645146139961954`      | update/sys_script_4cbb3f8bdbce18101645146139961954.xml |
| BR `freemium` (disabled)                     | `24bf37670f592200c783cd8ce1050e12`      | update/sys_script_24bf37670f592200c783cd8ce1050e12.xml |
| Mail Script `qr_code_in_email`               | `ab66eab90f5ea240c783cd8ce1050ed4`      | update/sys_script_email_ab66eab90f5ea240c783cd8ce1050ed4.xml |
| Widget `QR Code`                             | `bc9c2730db8650101645146139961958`      | update/sp_widget_bc9c2730db8650101645146139961958.xml |
| Property `logging.verbosity`                 | `fac76c3a0fbc0600c783cd8ce1050e04`      | update/sys_properties_fac76c3a0fbc0600c783cd8ce1050e04.xml |

---

## 13. Version Notes

- **4.3.17** — Current version, packaged 2025-11-24. App manifest `sys_mod_count = 161` cumulative revisions.
- **Created** — 2015-06-24 by `martin.palacios`. One of the older apps in the Stave catalog (the Service Portal stack was added later when SP was introduced; the AwesomeQR library was bundled circa 2020).
- **Demo data** — A "freemium" business rule + client script (both shipped disabled) hint that the app has historically had a demo build distributed separately from the full version; both contain "contact us at staveapps.com" messages.
- **Repository convention** — Delivered as ServiceNow-generated XML in x_stave_qr_code/5761e35a0f3c0600c783cd8ce1050e37/. The README.md inside is the generic ServiceNow recovery guide, not app-specific.

---

*Document generated from source XML on 2026-06-22 against scope `x_stave_qr_code` v4.3.17.*
