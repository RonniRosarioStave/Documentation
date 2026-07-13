# Stave Maps (`x_stave_em`) — Documentation

> ServiceNow scoped application that plots **any ServiceNow table's records on an interactive map**. A map definition points at one or more source tables, resolves each record to latitude/longitude (from explicit coordinate fields or a location reference), and renders the results through either **Google Maps** (markers + heatmap + clustering) or **Esri ArcGIS** (web-map-based markers), with per-record tooltips and drill-through to the source record.

---

## 1. At-a-Glance

| Attribute      | Value                                            |
| -------------- | ------------------------------------------------ |
| **Name**       | Stave Maps                                       |
| **Scope**      | `x_stave_em`                                     |
| **Vendor**     | Stave Inc                                         |
| **Version**    | 3.2.14                                           |
| **JS Level**   | `es_latest`                                      |

**Short description:**

> Display ServiceNow data on Google Maps and ArcGIS Map

---

## 2. Capabilities Overview

The application is organized around **one configuration model** (map + data layers) and **two interchangeable rendering engines**, plus supporting search and drill-through subsystems.

| Pillar                | Trigger model                    | Purpose                                                                                     |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| **Map Definition**    | Configuration                    | Declares the map: provider, center, zoom, clustering, heatmap opacity/intensity, ArcGIS webmap ID. |
| **Data Layers**       | Configuration                    | Per-map layers: which table to plot, how to resolve location, filter, tooltip, icon, cluster/heatmap participation. |
| **Google renderer**   | On demand (View Map / portal)    | Markers with custom icons + **heatmap** + **clustering**, all toggleable per dataset.       |
| **ArcGIS renderer**   | On demand (View Map / portal)    | Web-map-based markers with picture symbols, info popups, legend, and optional type-ahead search. Markers only (no heatmap/cluster). |
| **On-screen search**  | User action (ArcGIS only)        | Full-text search across layer tables; centers/zooms the map on the picked result.           |
| **Drill-through**     | Marker/tooltip click             | Opens the underlying ServiceNow record in an iframe modal ("View Record").                  |

### 2.1 The "plot a table on a map" flow (conceptual)

1. An author creates an `x_stave_em_maps` record (provider, center, zoom, etc.) and one or more child `x_stave_em_map_data` layers (table, location field, filter, tooltip).
2. A user opens the map — via the **View Map** action on the map record, or by navigating to the portal page directly.
3. **View Map** saves the record and redirects by provider:
   - `map_provider = 0` (ArcGIS) → Service Portal page **`sp_map_esri?map_id=<sys_id>`**
   - `map_provider = 1` (Google) → Service Portal page **`sp_maps?map_id=<sys_id>`**
4. For each active data layer, the app queries the layer's `table` with its filter, resolves coordinates (explicit lat/long fields or a location reference), merges records at identical coordinates into a single weighted point, evaluates the per-record tooltip, and resolves the marker icon.
5. The widget draws the map with the appropriate provider SDK and wires up marker clicks, the options panel, and the "View Record" modal.

---

## 3. Data Model — Table Reference

The app ships **2 tables**, both prefixed `x_stave_em_`.

### 3.1 `x_stave_em_maps` — "Map Definitions"

The map configuration record.

| Field               | Type            | Mand. | Default       | Hint / Notes                                                                                   |
| ------------------- | --------------- | :---: | ------------- | --------------------------------------------------------------------------------------------- |
| `name`              | string (100)    |   Y   |               | Map name.                                                                                     |
| `description`       | string (1000)   |       |               | Free-text description.                                                                        |
| `map_provider`      | choice (40)     |       | `0`           | `0` = ArcGIS, `1` = Google. Drives which renderer/portal is used and which fields show.        |
| `center_latitude`   | float           |       | `36.008522`   | Initial map center latitude. Validated to [−90, 90] on submit.                                 |
| `center_longitude`  | float           |       | `-95.221764`  | Initial map center longitude. Validated to [−180, 180] on submit.                              |
| `zoom_level`        | integer         |       | `4`           | Starting zoom (0 = lowest).                                                                     |
| `webmap_id`         | string (32)     |   *   |               | ArcGIS.com web map ID. **Mandatory when `map_provider = 0`**. Ignored by Google.               |
| `show_search`       | boolean         |       |               | Show the on-screen type-ahead search. Only visible for ArcGIS.                                 |
| `enable_clustering` | boolean         |       |               | Marker clustering toggle (Google).                                                             |
| `opacity`           | decimal (15)    |       | `0.6`         | Heatmap opacity, 0–1. Defaults to 0.6.                                                          |
| `max_intensity`     | integer         |       |               | Heatmap max intensity. Blank = dynamic auto-scaling by point concentration.                     |

### 3.2 `x_stave_em_map_data` — "Map Data"

A data layer belonging to a map.

| Field                | Type                              | Mand. | Default | Hint / Notes                                                                              |
| -------------------- | --------------------------------- | :---: | ------- | ---------------------------------------------------------------------------------------- |
| `name`               | string (40)                       |   Y   |         | Layer name.                                                                               |
| `map`                | reference → `x_stave_em_maps`      |       |         | Parent map.                                                                               |
| `table`              | table_name (80)                   |   Y   |         | Source table whose records are plotted.                                                  |
| `filter`             | conditions (4000)                 |       |         | Condition-builder filter (dependent on `table`) scoping which records are plotted.        |
| `specific_location`  | boolean                           |       |         | If `true`, use explicit `latitudes`/`longitude` fields; if `false`, use the `location` reference. |
| `location`           | field_name (80)                   |   *   | `null`  | Field on `table` that holds the location reference. Mandatory unless `specific_location`. |
| `latitudes`          | field_name (80)                   |       |         | Field on `table` holding latitude. Shown only when `specific_location = true`.            |
| `longitude`          | field_name (80)                   |       |         | Field on `table` holding longitude. Shown only when `specific_location = true`.           |
| `tooltip`            | script (8000)                     |       | *(see below)* | Server script producing the marker popup HTML. Vars: `content`, `gr`.                |
| `show_icons`         | boolean                           |       | `true`  | Whether to draw marker icons for this layer.                                             |
| `icon`               | user_image                        |       |         | Uploaded marker icon.                                                                    |
| `icon_db`            | reference → `db_image`            |       |         | DB image marker icon (alternative to `icon`).                                            |
| `icon_width`         | integer                           |       | `25`    | Icon width in px.                                                                        |
| `icon_height`        | integer                           |       | `25`    | Icon height in px.                                                                       |
| `include_in_cluster` | boolean                           |       |         | Participate in marker clustering (Google).                                               |
| `include_in_heatmap` | boolean                           |       |         | Participate in the heatmap layer (Google).                                               |

**Default `tooltip` script (as shipped):**

```javascript
/**
* You have access to these variables: "content" and "gr".
* content : The content of this variable will be shown in the pop-up menu if user clicks on its icon
* gr:  GlideRecord to the table with the filter applied.
*
* Usage: content = gr.short_description;
*
* Please note: The records without location information, and locations without latitude/longitude information
* are omitted from the glide record.
**/

content = gr.getDisplayValue();
```

---

## 4. UI Layer

Map rendering is implemented as **Service Portal AngularJS widgets**. Two portals, two widgets, one per provider.

### 4.1 Service Portal — Portals, Pages, Widgets

**Portals:**

| Portal            | URL suffix   | Purpose                     |
| ----------------- | ------------ | --------------------------- |
| Stave Maps        | `sp_maps`    | The **Google Maps** portal. |
| Stave Map Esri    | `sp_map_esri`| The **ArcGIS/Esri** portal. |

**Widgets:**

| Widget          | Renderer | Features                                                                              |
| --------------- | -------- | ------------------------------------------------------------------------------------ |
| S Maps Widget   | **Google** | Markers + heatmap + clustering. Per-dataset Markers/Heatmap/Cluster toggles, auto-centering, live refresh. |
| S_Map_Widget_Esri | **ArcGIS** | ArcGIS JS API 3.x web map. Picture-marker symbols + info popups + legend. No heatmap/clustering. |

Both widgets source data two ways: from a saved definition or an ad-hoc query. Both templates embed a "View Record" iframe modal.

### 4.2 Navigation

Application menu **Stave Maps**:

- **Core data** — Map Definitions (`x_stave_em_maps`), Map Data (`x_stave_em_map_data`).
- **External links** — Contact Stave Support, Privacy Policy, About Stave.

---

## 5. Configuration

### 5.1 System Properties

| Property                            | Type    | Value | Purpose                                                                        |
| ----------------------------------- | ------- | ----- | ----------------------------------------------------------------------------- |
| `x_stave_em.searchResult.zoomLevel` | integer | `6`   | Zoom level applied when the user clicks a search result to center the map.      |
| `x_stave_em.google_map_scope`       | string  | *(widget scope reference)* | Reference to the Google map widget scope.                          |
| `x_stave_em.esri_map_scope`         | string  | *(widget scope reference)* | Reference to the Esri map widget scope.                            |

The renderers also read platform properties `google.maps.key` (Google Maps API key) and `glide.servlet.uri` (icon base path).

### 5.2 Choice Lists

| Field / choice set              | Values                 | Status  |
| ------------------------------- | ---------------------- | ------- |
| `x_stave_em_maps.map_provider`  | `0` ArcGIS, `1` Google | Active  |

### 5.3 Cross-Scope Table Access

Read-only table-access grants let the app plot records from other scopes/tables on maps. Out of the box, common tables such as `task`, `incident`, `cmdb_ci` (and its child classes), `alm_asset`, `cmn_location`, `sys_user`, and `wm_order` are granted. Adding a new source table requires granting read access for that table (see §7.5).

### 5.4 Image Assets

The app ships marker/cluster/logo image binaries used as map icons, including a default marker `stave_map_default_marker.png`.

---

## 6. Usage / How-To

### 6.1 Plot a table using a location reference (e.g. Assets by location)

1. Create an `x_stave_em_maps` record: set `name`, `map_provider` (Google or ArcGIS), `center_latitude`/`center_longitude`, `zoom_level`. For ArcGIS, also set `webmap_id` (mandatory).
2. Add an `x_stave_em_map_data` layer:
   - `table = alm_asset` (or any granted table);
   - leave `specific_location = false`;
   - `location = <reference field that points at cmn_location>` (mandatory in this mode);
   - `filter = <optional condition>`;
   - `tooltip = content = gr.getDisplayValue();` (or a richer script).
3. Save, then click **View Map**. You're redirected to `sp_maps` (Google) or `sp_map_esri` (ArcGIS) and the records render as markers.

### 6.2 Plot a table using explicit latitude/longitude columns

1. On the `x_stave_em_map_data` layer, set `specific_location = true`.
2. The `location` field disappears; set `latitudes` and `longitude` to the two coordinate field names on the source table.
3. Configure the icon (`icon` upload or `icon_db`), `icon_width`/`icon_height`, and tooltip as needed.

### 6.3 Enable heatmap / clustering (Google only)

1. On the map, set `opacity` (0–1) and optionally `max_intensity` (blank = auto-scale).
2. On each data layer, tick `include_in_heatmap` and/or `include_in_cluster`.
3. In the rendered Google widget, use the **Map Options** panel to toggle Markers / Heatmap / Cluster per dataset at runtime.

### 6.4 Use an existing ArcGIS.com web map

1. Set `map_provider = 0` (ArcGIS) on the map record.
2. Set `webmap_id` to the ID from the ArcGIS.com item URL.
3. Optionally tick `show_search` to enable the on-screen type-ahead search (ArcGIS only).

### 6.5 Add a new source table

Because the app enforces cross-scope table access, plotting a table outside those granted by default requires adding a new read grant for that table before it can be queried by the map engines.

---

## 7. Extension Points

- **Ad-hoc maps without a saved definition.** The app can build a map from parameters alone — embed the map widget on any portal page and pass table, encoded query, coordinate fields, icon, and target page to render a map of an arbitrary query.
- **Custom marker icons.** Per layer via uploaded `icon`, a `db_image` reference (`icon_db`), and `icon_width`/`icon_height`; falls back to `stave_map_default_marker.png`.
- **Rich tooltips.** The `tooltip` script field runs server-side with `content` and `gr` in scope — build HTML popups, links, or computed values.
- **New source tables.** Grant read access for the table (§5.3, §6.5).
- **Provider swap.** A map can be flipped between Google and ArcGIS by changing `map_provider`; the View Map action routes to the matching portal automatically.

---

## 8. Version Notes

- **3.2.14** — Current version.
- This app is delivered as a ServiceNow scoped application. The modern path (View Map → `sp_maps`/`sp_map_esri` portal widgets) supersedes older map-rendering pages.
