# Stave QR Code Generator (`x_stave_qr_code`) — Documentation

> ServiceNow scoped application that **generates and renders QR codes and barcodes** (CODE128, UPC, CODE39, EAN, ITF-14, MSI, Pharmacode, Codabar) tied to any ServiceNow record, table-filter result set, URL, free text, or Service Portal page — with customizable templates, batch generation, list-view printing, email embedding, and a Service Portal widget.

---

## 1. At-a-Glance

| Attribute             | Value                    |
| --------------------- | ------------------------ |
| **Name**              | Stave QR Code Generator  |
| **Scope**             | `x_stave_qr_code`        |
| **Vendor**            | Stave (Stave Apps)       |
| **Version**           | 4.3.17                   |
| **JS Level**          | `es_latest`              |
| **Default user role** | `x_stave_qr_code.user`   |

**Short description:** *Generate and maintain QR and Bar Codes.*

---

## 2. Capabilities Overview

The app is organized around **four pillars**:

| Pillar                | Trigger model           | Output                                                                                  |
| --------------------- | ----------------------- | --------------------------------------------------------------------------------------- |
| **QR Code Generator** | Per-record manual create | One QR per record; rendered via a display processor + Service Portal widget.           |
| **QR Code Batch**     | List-filter manual run  | Bulk QR generation against a filter on any table.                                       |
| **Barcode Generator** | Per-record manual create | One barcode (CODE128/UPC/CODE39/etc.) per record; rendered via a display processor.    |
| **Barcode Batch**     | List-filter manual run  | Bulk barcode generation.                                                                |

**Supporting subsystem** — **QR Templates** (`x_stave_qr_code_qr_template` + `qr_template_content`) — define presentation: CSS, colors, logo scale, HTML wrappers, and field interpolation tokens like `field:assigned_to:field`.

**Per-record QR flow (conceptual):**
1. Create a `stave_qr_code` record pointing at a table + `document` (or a URL / free text), pick a `template`, pick a Service Portal `portal` + `page`.
2. The display processor URL `x_stave_qr_code_stave_qr_display.do?sysparm_code_id=<sys_id>` renders the QR.
3. Embed in emails via the shipped mail script, or list-print via the **View Codes** action.

**Batch flow (conceptual):**
1. Create a `stave_qr_code_batch` or `stave_barcode_batch` definition with `table_name` + condition-builder `filter` + `template`.
2. Click **Generate Batch Codes** on the form. The app previews the affected row count, then on confirm creates one definition row per matching record.

---

## 3. Data Model — Table Reference

The app ships **6 tables**, all prefixed `x_stave_qr_code_`.

### 3.1 `x_stave_qr_code_stave_qr_code`

Per-record QR definition.

| Field                    | Type                                    | Mand. | Notes                                                                                                  |
| ------------------------ | --------------------------------------- | :---: | ------------------------------------------------------------------------------------------------------ |
| `name`                   | string (100), table_reference           |   Y   | Display name of this code.                                                                             |
| `active`                 | boolean                                 |       | Default `true`.                                                                                        |
| `description`            | string (1000)                           |       |                                                                                                        |
| `type`                   | choice (40)                             |   Y   | `URL` (default), `Record`, `Service Portal Record`, `text`. Drives which other field is interpreted.   |
| `url`                    | url (1024)                              |       | Used when `type = URL`.                                                                                |
| `text`                   | string (384)                            |       | Used when `type = text`.                                                                               |
| `table`                  | string (40)                             |       | Source table when `type = Record` / `Service Portal Record`.                                           |
| `document`               | document_id (32), dependent on `table`  |       | The specific record the code points at.                                                                |
| `view`                   | string (40)                             |       | UI view name to honor when rendering / linking.                                                        |
| `template`               | reference → `x_stave_qr_code_qr_template` | Y   | Visual template (background, border, custom CSS, text-script).                                          |
| `portal`                 | reference → `sp_portal`                 |   Y   | Service Portal context used when `type = Service Portal Record`.                                       |
| `page`                   | reference → `sp_page`                   |   Y   | The Service Portal page the code links to.                                                             |
| `catalog`                | reference → `sc_catalog`                |       | Optional catalog scoping.                                                                              |
| `catalog_view`           | string (40)                             |       | Default `catalog_default`. View shown when the code lands on a catalog item.                           |
| `external_url`           | boolean                                 |       | If `true`, treat `url` as a full external URL rather than an instance-relative path.                   |
| `logo`                   | reference → `db_image`                  |       | Logo overlay.                                                                                          |
| `not_available_message`  | string (1000)                           |       | Shown when the QR cannot be rendered (e.g. record deleted, valid_to passed).                           |
| `valid_to`               | glide_date_time, default `2100-01-01 23:59:59` |  | Expiration cutoff.                                                                                     |

### 3.2 `x_stave_qr_code_stave_qr_code_batch`

Bulk QR generator.

| Field                   | Type                                            | Mand. | Notes                                                                            |
| ----------------------- | ----------------------------------------------- | :---: | -------------------------------------------------------------------------------- |
| `number`                | string (40)                                     |       | Auto-generated.                                                                  |
| `name`                  | string (100)                                    |   Y   |                                                                                  |
| `active`                | boolean                                         |       | Default `true`.                                                                  |
| `description`           | string (1000)                                   |       |                                                                                  |
| `table_name`            | table_name (80)                                 |   Y   | Target table to walk.                                                            |
| `filter`                | conditions (4000), dependent on `table_name`    |       | Condition builder.                                                               |
| `type`                  | choice (40)                                     |   Y   | `Record` (default), `text`, `Service Portal Record`.                             |
| `template`              | reference → `x_stave_qr_code_qr_template`       |   Y   |                                                                                  |
| `portal` / `page`       | reference → `sp_portal` / `sp_page`             |       | SP context for generated children.                                               |
| `logo`                  | reference → `db_image`                          |       |                                                                                  |
| `view`                  | string (40)                                     |       |                                                                                  |
| `not_available_message` | string (1000)                                   |       |                                                                                  |
| `valid_to`              | glide_date_time, default `2100-01-01 16:59:59`  |       | Stamped onto every child row.                                                    |

### 3.3 `x_stave_qr_code_stave_bar_code`

Per-record barcode definition.

| Field            | Type                                    | Mand. | Notes                                                                              |
| ---------------- | --------------------------------------- | :---: | ---------------------------------------------------------------------------------- |
| `name`           | string (100)                            |   Y   |                                                                                    |
| `active`         | boolean                                 |       |                                                                                    |
| `description`    | string (1000)                           |       |                                                                                    |
| `barcode_types`  | string (40)                             |   Y   | Symbology — see codes below. Default `1` (CODE128).                                |
| `type`           | string (40)                             |   Y   | `1` Record (default). `2` Service Portal Record is shipped **inactive**.           |
| `table`          | table_name (40)                         |       |                                                                                    |
| `document`       | document_id (32), dependent on `table`  |   Y   |                                                                                    |
| `field`          | field_name (40), dependent on `table`   |   Y   | The field on `document` whose value becomes the barcode payload.                   |
| `template`       | reference → `x_stave_qr_code_qr_template` | Y   | Reuses the QR template table for styling wrappers.                                 |
| `portal` / `page`| reference → `sp_portal` / `sp_page`     |       |                                                                                    |
| `view`           | string (128)                            |       |                                                                                    |

**Barcode symbology codes** (`barcode_types`):

| Code | Symbology  | Code | Symbology  |
| ---- | ---------- | ---- | ---------- |
| `1`  | CODE128    | `5`  | MSI        |
| `2`  | UPC        | `6`  | Pharmacode |
| `3`  | CODE39     | `7`  | Codabar    |
| `4`  | ITF-14     | `8`  | EAN        |

### 3.4 `x_stave_qr_code_stave_barcode_batch`

Bulk barcode generator.

| Field        | Type                                            | Mand. | Notes                                                                            |
| ------------ | ----------------------------------------------- | :---: | -------------------------------------------------------------------------------- |
| `number`     | string (40)                                     |       | Auto-generated.                                                                  |
| `name`       | string (40)                                     |   Y   |                                                                                  |
| `description`| string (1000)                                   |       |                                                                                  |
| `barcode`    | string (40)                                     |   Y   | Symbology (same `1`-`8` codes as §3.3). Default `1` (CODE128).                   |
| `type`       | string (40)                                     |   Y   | `1` Record (only active option).                                                 |
| `table_name` | table_name (80)                                 |   Y   |                                                                                  |
| `filter`     | conditions (4000), dependent on `table_name`    |       | Condition builder.                                                               |
| `field`      | field_name (40), dependent on `table_name`      |   Y   | Field used as barcode payload for every matching record.                         |
| `document`   | document_id (32), dependent on `table_name`     |   Y   | (Present for legacy compatibility.)                                              |
| `page` / `portal`| reference → `sp_page` / `sp_portal`         |       |                                                                                  |
| `view`       | string (128)                                    |       |                                                                                  |

### 3.5 `x_stave_qr_code_qr_template`

Visual template applied to QR codes and barcodes.

| Field          | Type                      | Mand. | Notes                                                                                                                                                |
| -------------- | ------------------------- | :---: | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`         | string (40), display=true |   Y   |                                                                                                                                                      |
| `background`   | string (100)              |       | CSS `background` value.                                                                                                                              |
| `border`       | string (100)              |       | CSS `border` value.                                                                                                                                  |
| `margin`       | string (100)              |       | CSS `margin`.                                                                                                                                        |
| `padding`      | string (100)              |       | CSS `padding`.                                                                                                                                       |
| `custom_style` | string (1000)             |       | Free-form additional CSS appended to the wrapper.                                                                                                    |
| `is_text`      | boolean                   |       | If `true`, the template is used with `type = text` QR codes.                                                                                         |
| `text`         | string (384)              |       | Static text payload.                                                                                                                                 |
| `text_script`  | script (8000)             |       | Server-side script executed with `current` as the code record. Returns a `variables` object that `current.text` placeholders can reference.          |

### 3.6 `x_stave_qr_code_qr_template_content`

Layout fragments composed under a template.

| Field            | Type                              | Mand. | Notes                                                                                                            |
| ---------------- | --------------------------------- | :---: | ---------------------------------------------------------------------------------------------------------------- |
| `qr_template`    | reference → `x_stave_qr_code_qr_template` | Y | Parent template.                                                                                                 |
| `type`           | choice (40)                       |   Y   | QR Code, HTML, Formatted HTML, Code Name.                                                                        |
| `order`          | integer                           |       | Composition order within the template.                                                                           |
| `width`          | integer (default `100`)           |       | QR / image pixel width.                                                                                          |
| `height`         | integer (default `100`)           |       | QR / image pixel height.                                                                                         |
| `color_dark`     | string (40), default `#000000`    |       | QR foreground color.                                                                                             |
| `color_light`    | string (40), default `white`      |       | QR background color.                                                                                             |
| `logo_scale`     | float, default `0.2`              |       | Logo overlay size (0.1 - 0.9).                                                                                    |
| `html`           | string (10000)                    |       | Raw HTML block. Supports `field:field_name:field` tokens that interpolate values from the linked record (e.g. `field:assigned_to:field`). |
| `formatted_html` | html (8000)                       |       | WYSIWYG-rendered HTML block.                                                                                     |
| `custom_style`   | string (1000)                     |       | Per-content CSS overrides.                                                                                       |

---

## 4. Service Portal Integration

The app ships a Service Portal stack for rendering QR codes inside portal pages.

### 4.1 Widget

| Attribute        | Value                                                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**         | `QR Code`                                                                                                                                                                                   |
| **Roles**        | `x_stave_qr_code.user`                                                                                                                                                                      |
| **Option schema**| `qr_size` (integer, default `150`, "QR code size") and `show_text` (boolean, default `false`, "Show Text").                                                                                 |
| **Behavior**     | Pulls QR rows by either `qr_codes` (comma-separated sys_ids) or `table + sys_id`, renders each QR client-side, and re-renders live when the underlying record changes.                       |
| **Template**     | Grid/List toggle plus a Print button.                                                                                                                                                       |

---

## 5. Configuration

### 5.1 System Properties

| Property                              | Type        | Default | Choices                                          | Purpose                                                                       |
| ------------------------------------- | ----------- | ------- | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| `x_stave_qr_code.logging.verbosity`   | choicelist  | `debug` | `Debug=debug`, `Info=info`, `Warn=warn`, `Error=error` | Logging level honored throughout the app.                               |

This is the *only* property the app defines.

### 5.2 Choice Lists

- `x_stave_qr_code_qr_template_content.correct_level` — QR error-correction level choices.
- `x_stave_qr_code_qr_template_content.type` — QR Code / HTML / Formatted HTML / Code Name.
- `x_stave_qr_code_stave_bar_code.barcode_types` — the 8 barcode symbologies listed in §3.3.
- `x_stave_qr_code_stave_bar_code.type` — Record (1) / Service Portal Record (2, inactive).
- `x_stave_qr_code_stave_barcode_batch.barcode` — same 8 symbologies as bar_code.
- `x_stave_qr_code_stave_barcode_batch.type` — Record (1) only.
- `x_stave_qr_code_stave_qr_code_batch.type` — Record (default), text, Service Portal Record.
- `x_stave_qr_code_stave_qr_code.type` — URL (default), Record, Service Portal Record, text.

### 5.3 Demo Data

Demo QR codes / barcodes / templates / portal pages are imported only when the "Install demo data" flag is set during app installation, and exist mainly to give first-time users working examples to inspect.

### 5.4 Scheduled Jobs

None. All code/barcode generation is **on-demand** (per-record save or list-action button).

### 5.5 Display Processors

The display URLs `x_stave_qr_code_stave_qr_display.do` and `x_stave_qr_code_stave_barcode_display.do` are ServiceNow display processors (server-side endpoints) referenced by the mail script, UI actions, and widget.

### 5.6 Navigation

The application menu **Stave QR Code Generator** groups modules into:

- **QR Codes** — All QR Codes, Create new QR Code, All QR Code Batches, Create new QR Code Batch.
- **Barcodes** — All Barcodes, Create new Barcode, All Barcode Batches, Create new Barcode Batch.
- **Configure** — All QR Templates, New QR Template.
- **Support** — Getting Started, About Stave, More Stave Apps, Contact Stave Support, Privacy Policy.

---

## 6. Usage / How-To

### 6.1 Generate a QR for one record

1. Navigate to **Stave QR Code Generator → Create new QR Code**.
2. Set:
   - `Name` — display label.
   - `Type` — `Record`, `Service Portal Record`, `URL`, or `text`.
   - For `Record`: set `Table` then pick `Document`.
   - For `Service Portal Record`: set `Table`, `Document`, plus `Portal` + `Page`.
   - For `URL`: set `URL` and optionally `External URL = true`.
   - For `text`: set `Text`.
3. Pick a `Template` (or create one — see §6.4).
4. Save. The display URL is now `x_stave_qr_code_stave_qr_display.do?sysparm_code_id=<sys_id>`.
5. Click **Test Link** to verify the target opens correctly in a new window.

### 6.2 Generate QR codes in bulk

1. Navigate to **Stave QR Code Generator → Create new QR Code Batch**.
2. Set `Table Name`, build a `Filter`, and pick `Type` + `Template`.
3. Click **Generate Batch Codes**. The app previews affected rows and asks for confirmation.
4. On confirm, one `stave_qr_code` row is created per matching record, each inheriting `template`, `portal`, `page`, `valid_to`, and `not_available_message` from the batch.

### 6.3 Print or view multiple codes

From a list view of QR codes or barcodes, select rows and click **View Codes** in the list actions menu. A new tab opens with a printable page rendering the selected codes.

### 6.4 Define a template with dynamic content

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

### 6.5 Embed a QR in an outbound email

Use the shipped mail script in a notification template: `${mail_script:qr_code_in_email}`. It expands to an `<img>` tag pointing at the display processor URL for `current.sys_id`.

### 6.6 Add the widget to a Service Portal page

1. In Service Portal Designer, drop the **QR Code** widget onto a page.
2. Configure widget options:
   - `qr_size` — pixel size of each QR (default 150).
   - `show_text` — render the code's `name` as a caption.
3. Pass parameters via the page URL or instance config:
   - `qr_codes` — comma-separated `sys_id`s to render.
   - `table` + `sys_id` — render all QR codes linked to a given record.

---

## 7. Extension Points

- **Template scripting.** `qr_template.text_script` is evaluated server-side with `current` set to the QR code row. Use it for record-driven text payloads. Return a `variables` object whose keys can be referenced as placeholders.
- **Field interpolation.** Any `qr_template_content.html` block supports `field:<field_name>:field` — handy for `Owner: field:assigned_to:field`, `Asset: field:asset_tag:field`, etc.
- **Display processors.** The shipped processor URLs accept `sysparm_code_id`, `sysparm_print`, and `sysparm_query` — any external system can request a rendered QR/barcode image by hitting these URLs with an authenticated session.
- **Widget reuse.** The `QR Code` widget can be added to any portal page and updates live when the underlying record changes.
- **Barcode symbology.** Adding a new symbology requires updating the `barcode_types` / `barcode` choice list and the barcode rendering path.

---

## 8. Version Notes

- **4.3.17** — Current version.
- This app is delivered as a ServiceNow scoped application. The Service Portal stack was added after Service Portal was introduced.
