# Stave WWEX Integration (`x_stave_wwex_sn`) — Technical Documentation

> ServiceNow scoped application that connects **Service Catalog shipment requests to the Worldwide Express (WWEX) SpeedShip API**. A requester fills in the **WWEX Shipment** catalog item (pickup and drop-off addresses with Google Places autocomplete, shipment type, optional insurance); a Flow Designer flow then builds the WWEX shipment envelope, validates addresses against the WWEX `addressValidationFlow` endpoint, and posts the request to the SpeedShip `shopFlow` endpoint using an OAuth 2.0 client-credentials token that the app caches and refreshes itself.

---

## 1. At-a-Glance

| Attribute              | Value                                                                                         |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| **Name**               | Stave WWEX Integration                                                                        |
| **App name (manifest)**| WWEX for ServiceNow                                                                           |
| **Scope**              | `x_stave_wwex_sn`                                                                             |
| **Application sys_id** | `db9848f1875bbe50b4fa426acebb35e1`                                                            |
| **Vendor**             | Stave (manifest `vendor` / `vendor_prefix` are empty)                                         |
| **Version**            | 1.0.0                                                                                         |
| **JS Level**           | `es_latest`                                                                                   |
| **Created**            | App record 2026-02-26 by `richard.castillo`; earliest artifacts 2025-04-21 by `jose.cornelio@stavecorp.com` |
| **Last Updated**       | 2026-03-11 by `richard.castillo` (source-control snapshot 2026-03-16)                         |
| **Revisions**          | App manifest `sys_mod_count = 1`                                                              |
| **Default user role**  | *(none)* — `user_role` is empty and the app ships no roles                                    |
| **Licensing**          | `licensable = true`, `enforce_license = none`, `license_model = none`                         |
| **Dependency**         | **Stave Buy It** (`x_stave_buy_it`) ≥ 1.5.8                                                   |
| **Source instance**    | `ven01088`                                                                                    |
| **Source manifest**    | sys_app_db9848f1875bbe50b4fa426acebb35e1.xml                                                  |

**Short description (from manifest):** *WWEX for Buy It in ServiceNow.*

> **Note on environment.** Every endpoint, token URL and OAuth `audience` in the app points at the **WWEX staging environment** (`*.staging-wwex.com`, audience `staging-wwex-apig`). See §9.3 and §13.

---

## 2. Capabilities Overview

The application is organized around **one catalog front end** and **one outbound integration pipeline**, plus a self-managed token cache.

| Pillar                      | Trigger model                         | Operates on                                         | Purpose                                                                                                     |
| --------------------------- | ------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Shipment request form**   | User submits catalog item             | `WWEX Shipment` (`sc_cat_item`) + **WWEX Variable Set** | Captures pickup (origin) and drop-off (destination) addresses, phone, shipment type, and optional insurance. |
| **Address autocomplete**    | User typing in the portal             | Two custom SP widgets (Google Places)               | Resolves a free-text address into street / city / region / postal code / country variables.                |
| **Shipment flow**           | Service Catalog trigger (on RITM)     | Flow **WWEX Shipment Request**                      | Reads catalog variables → builds sender + receiver addresses → builds envelope body → calls WWEX API.       |
| **Address validation**      | Inside the *Envelope Address* action  | WWEX `addressValidationFlow` endpoint               | Validates each address (`validationStatus = SUCCESS`) before it is added to the envelope.                  |
| **Shipment submission**     | Inside the *API Request* action       | WWEX `shopFlow` endpoint                            | Posts the `{request, correlationId}` envelope; the RITM number is the correlation ID.                      |
| **Token management**        | Every `wwex_api_helper` instantiation | `oauth_credential` (global)                         | Reuses a stored bearer token or fetches/refreshes one via client-credentials when < 1 hour remains.        |

---

## 3. Usage / How-To

### 3.1 Submit a shipment request

1. In the Service Portal, open **Service Catalog** (or **Stave Catalog**) → **Vendor Catalogs** → **WWEX Shipment** (*WWEX Domestic Shipment Req.*).
2. Complete the **Standard Employee Questions** variable set (provides `requested_for`, used as the sender contact).
3. In **Pickup Address**, start typing and pick a Google suggestion. The widget fills `street_origin`, `locality_origin`, `region_origin`, `postal_code_origin`, and `country_origin`.
4. Enter **Phone** (`phone_origin`).
5. In **Shipping Address**, pick the drop-off address. The widget fills the `*_destination` variables.
6. Confirm **Shipment Type** (`domestic` / `international`) — the *Shipment Type by Address Country* client script pre-selects it from `country_origin` (see §6.2 for a known mismatch).
7. (Optional) Set **Add Insureance = Yes**; **Package Value ($)** becomes visible and mandatory.
8. Submit. The **WWEX Shipment Request** flow runs against the new RITM.

### 3.2 Inspect what was sent to WWEX

1. Open the RITM and go to its Flow Designer execution (**Flow Designer → Executions**, filter by the RITM).
2. Expand the **API Request** step. Outputs:
   - `request_body` — the exact envelope posted (`{ request, correlationId }`, with `correlationId` = RITM number).
   - `response` — raw response body from `shopFlow` (or the exception message on failure).
3. Expand each **Envelope Address** step to see `address` and `status` (address-validation result). An invalid address produces `address = "Address not valid"`.

### 3.3 Rotate or set WWEX credentials

1. Update **System Properties** `x_stave_wwex_sn.client_id` / `x_stave_wwex_sn.client_secret` (§9.1).
2. Update the **WWEX Basic Auth** profile (`sys_auth_profile_basic`) — used by *Get Token (Basic Auth)* and *AddressValidationFlow*.
3. Update the client ID/secret on the **WWEX OAuth v2** and **WWEX SpeedShip OAuth** application registries (`oauth_entity`).
4. Force a fresh token: delete the `oauth_credential` row with `scopes = wwex_client_credential`, or call `new x_stave_wwex_sn.wwex_api_helper().getValidToken(true)` from a scoped background script.

### 3.4 Point the app at WWEX production

1. On REST Message **WWEX API**, change every HTTP method endpoint from `speedship.staging-wwex.com` / `auth.staging-wwex.com` to the production hosts.
2. Change the `audience` parameter on *Get Token* / *Get Token (Basic Auth)*, **and** the hard-coded `"staging-wwex-apig"` in `wwex_api_helper.getAuthToken()` (§5.1).
3. Update the `token_url` on both OAuth registries.
4. Rotate credentials (§3.3) and clear the cached token.

### 3.5 Test the integration from a background script

```js
// Scope: x_stave_wwex_sn
var api = new x_stave_wwex_sn.wwex_api_helper();      // fetches / reuses a token
gs.info(api.addressValidation(['3050 Broad Ave'], 'Memphis', 'TN', '38112', 'US')); // true / false
```

---

## 4. Data Model — Table Reference

The app ships **no custom tables**. Its data lives in standard platform tables:

| Table                         | Owner  | How the app uses it                                                                                                |
| ----------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------ |
| `sc_req_item` + variables     | Global | One RITM per shipment request; catalog variables (§4.1) are the input to the flow.                                 |
| `oauth_credential`            | Global | Token cache. One row with `scopes = wwex_client_credential`; fields `token_received`, `expires`. Read/create/write granted via cross-scope privileges (§8.3). |
| `sys_flow_context` / executions | Global | Flow runs; holds `request_body` / `response` outputs of the API call (there is no custom log table).             |

### 4.1 Catalog variables — `WWEX Variable Set` (`wwex_variable_set`)

One-to-one variable set attached to **WWEX Shipment** (order 200), after **Standard Employee Questions** (order 100). Source: update/item_option_new_*.xml

| Order | Variable                       | Label                          | Type                         | Active | Notes                                                                                              |
| ----: | ------------------------------ | ------------------------------ | ---------------------------- | :----: | -------------------------------------------------------------------------------------------------- |
|   —   | `street_destination`           | Street Destination             | Single Line Text             |   Y    | Filled by the v2 widget for the drop-off address. No explicit order.                               |
|   —   | `locality_destination`         | Locality Destination           | Single Line Text             |   Y    | 〃                                                                                                  |
|   —   | `region_destination`           | Region Destination             | Single Line Text             |   Y    | 〃                                                                                                  |
|   —   | `postal_code_destination`      | Postal Code Destination        | Single Line Text             |   Y    | 〃                                                                                                  |
|   —   | `country_destination`          | Country Destination            | Single Line Text             |   Y    | 〃 (ISO short code, e.g. `US`).                                                                    |
|   —   | `formatter3`                   | —                              | Container Split              |   Y    | Layout only.                                                                                       |
|  150  | `phone_origin`                 | Phone                          | Single Line Text             |   Y    | Sender phone.                                                                                      |
|  210  | `dropoff_address_autocomplete` | Shipping Address               | Custom with Label            |   Y    | Widget **Google Address Auto Complete Custom v2**.                                                 |
|  250  | `pickup_address_autocomplete`  | Pickup Address                 | Custom with Label            |   Y    | Widget **Google Address Auto Complete Custom v2**.                                                 |
|  250  | `address_autocomplete_custom`  | Address Autocomplete Custom    | Custom with Label            | **N**  | Legacy widget **Google Address Auto Complete Custom** (writes `mapped_address`).                   |
|  300  | `shipment_type`                | Shipment Type                  | Select Box                   |   Y    | **Mandatory.** `domestic` (Domestic) / `international` (International).                            |
|  400  | `mapped_address`               | Mapped Address                 | Single Line Text             | **N**  | Legacy single-line address; parsed by *AddressOnChange*.                                           |
|  500  | `validated_address`            | Validated Address              | Container Start              |   Y    | Groups the origin fields below.                                                                    |
|  600  | `country_origin`               | Country                        | Single Line Text             |   Y    | Filled by widget (ISO short code).                                                                 |
|  700  | `region_origin`                | Region                         | Single Line Text             |   Y    | State / province short code.                                                                       |
|  800  | `street_origin`                | Street                         | Single Line Text             |   Y    | Street number + route.                                                                             |
| 1000  | `locality_origin`              | City                           | Single Line Text             |   Y    |                                                                                                    |
| 1100  | `postal_code_origin`           | Postal Code                    | Single Line Text             |   Y    |                                                                                                    |
| 1200  | `formatter2`                   | —                              | Container End                |   Y    | Closes *Validated Address*.                                                                        |
| 1220  | `formatter`                    | —                              | Container Split              |   Y    | Layout only.                                                                                       |
| 1225  | `add_insureance`               | Add Insureance                 | Yes / No                     |   Y    | Drives the UI policy in §6.3. (Label/name typo is in source.)                                      |
| 1230  | `package_value`                | Package Value ($)              | Single Line Text             |   Y    | Shown + mandatory when `add_insureance = Yes`. Free text — no numeric validation.                  |
| 1300  | `address_auto_complete_v4`     | Google Address Auto Complete   | Single Line Text (mandatory) | **N**  | Legacy experiment; target of the inactive *Google Address Auto-Complete* script.                   |

> **Not yet used by the flow:** `add_insureance`, `package_value`, `shipment_type`, and all `*_destination` variables. See §5.2.

### 4.2 Flow Designer complex objects

| Name                 | Defined on                         | Shape                                                                                                  |
| -------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `FD4b65e292…` (Address Line) | *Envelope Address* input `address_line` | Collection of strings → `addressLineList`.                                                    |
| **Shipping Envelope** | *Shipping Address* subflow output | Object mirroring the WWEX `address` block.                                                             |
| *API Request* schema | *API Request* action               | `request_body` / `response` outputs.                                                                   |

---

## 5. Business Logic — Server-Side

### 5.1 Script Includes

| Name               | API name                           | Access          | Client-callable | Responsibility |
| ------------------ | ---------------------------------- | --------------- | :-------------: | -------------- |
| `wwex_api_helper`  | `x_stave_wwex_sn.wwex_api_helper`  | package_private |       No        | All WWEX HTTP traffic and token handling. `initialize()` calls `getValidToken(false)` on every instantiation. Methods below. |
| `domesticShipment` | `x_stave_wwex_sn.domesticShipment` | package_private |       No        | Pure payload builders: `address(addressLineList, locality, region, postalCode, countryCode, companyName, phone, contactList)` returns the WWEX `address` object; `contactList(contactType, ...contacts)` upper-cases `contactType` (`SENDER` / `RECEIVER`) onto each contact; `test(message)` echoes. |

**`wwex_api_helper` methods**

| Method | Behavior |
| ------ | -------- |
| `getValidToken(forceNewToken)` | Queries `oauth_credential` where `scopes = wwex_client_credential`. If the stored token has **more than `tokenExpiryHours` (1 h)** left and `forceNewToken` is false, reuses `token_received`; otherwise calls `getAuthToken()` and updates the row. Inserts a new row if none exists. Debug logging when `enableLogging = true` (hard-coded on). |
| `storeValidToken(newToken, existing)` | Writes `token_received` and `expires = now + expires_in` (update or insert). |
| `getAuthToken()` | `RESTMessageV2("WWEX API", "Get Token (Basic Auth)")` with `grant_type = client_credentials`, `audience = staging-wwex-apig` (hard-coded). Returns the parsed JSON (`access_token`, `expires_in`, …). |
| `envelope(request, correlationId)` | Returns `{ request, correlationId }`. |
| `addressValidation(addressLineList, locality, region, postalCode, countryCode)` | POSTs to *AddressValidationFlow* with `productType = SMALLPACK`, `isInternational = false`, `geoLocationRequested = true`. Returns `true` only when `response.addressResponseList[0].validationStatus === "SUCCESS"`; `false` on any other status or exception. |
| `isValidAddress(valid_address)` / `correctedAddress(valid_address)` | Helpers over a validation response; `correctedAddress` returns `correctedAddressList[0]`. Not currently called. |
| `shopFlowDomestic(request_body)` | Stringifies the body if needed and POSTs to *shopFlow Domestic SAMPLE* with `Authorization: Bearer <token>`. Returns the raw response body, or the exception message. |

### 5.2 Flows & Subflows (Flow Designer)

| Name | Type | Trigger | Status | Active | Purpose |
| ---- | ---- | ------- | ------ | :----: | ------- |
| **WWEX Shipment Request** (`wwex_shipment_request`) | Flow | Service Catalog (`WWEX Shipment`), run as user | **draft** (published snapshot exists, see below) | Y | Main pipeline. |
| **Shop Flow Domestic** (`shop_flow_domestic`) | Flow | Service Catalog, run as user | draft | **N** | Earlier copy of the same pipeline with hard-coded test values. |
| **Shipping Address** (`shipping_envelope`) | Subflow | — | draft | **N** | Inputs `address_line`, `locality`, `region`, `postal_code`, `country_code`, `company_name`, `phone`, `contact_list`, `reciber_sender` → output `address` (*Shipping Envelope* object). Not referenced. |

**WWEX Shipment Request — current draft steps**

| # | Action | Comment | Key inputs |
| - | ------ | ------- | ---------- |
| 1 | Get Catalog Variables (OOB) | — | `requested_item = trigger.request_item`, template item = WWEX Shipment. |
| 2 | Envelope Address | *Sender Address* | `sender_receiver = sender`; `street_origin`, `locality_origin`, `region_origin`, `postal_code_origin`, `country_origin`, `phone_origin`; contact = `requested_for`; company via inline script `fd_data._1__get_catalog_variables.company`. |
| 3 | Envelope Address | *Receiver Address* | `sender_receiver = receiver`; **hard-coded** Dallas, TX 75219, US; fixed company and contact sys_ids. |
| 4 | Envelope Body | *API Request Body* | `origin_address = step2.address`, `destination_address = step3.address`. |
| 5 | API Request | — | `request_data = step4.request_data`, `correlation_id = trigger.request_item.number`. |

**Published snapshot** (`07140634fbe2a610842efe557eefdcc7`, 2025-07-02) contains only *Get Catalog Variables* → *Log* (the log message script calls `wwex_api_helper().infoMessage()`, a method that no longer exists). The draft above has **not been published**.

### 5.3 Flow Designer Actions

| Name | Internal name | State | Steps | Inputs → Outputs |
| ---- | ------------- | ----- | ----- | ---------------- |
| **Envelope Address** | `wwex_envelope_address` | **draft** (snapshot 2025-07-07 published) | 1 Script step | `sender_receiver`, `address_line` (string collection), `locality`, `region`, `postal_code`, `country_code`, `company`, `phone`, `contact` → `address`, `address_status`. Draft script builds the contact list, calls `wwex_api_helper.addressValidation(...)`, and returns the `domesticShipment.address(...)` object when valid or the string `"Address not valid"` otherwise. The published snapshot has **no** validation and only outputs `address`. |
| **Envelope Body** | `envelope_body` | published | *Get Current Time* script (`YYYY-MM-DD HH:mm:ss`) + *Envelope Body* script | `origin_address`, `destination_address` → `request_data`. Builds a `SMALLPACK` shipment with `shipmentDate = current_time` and **fixed sample values**: one 24×24×24 in custom package (`packagingType 02`), 15 LB, 50 % handling charge, `insuredValue 5000 USD`, `isCOD = true` / `codAmount 1700 USD`, sample references, `insuranceRequestFlag = true`. |
| **API Request** | `api_request` | published | 1 Script step | `request_data`, `correlation_id` → `request_body`, `response`. Calls `wwex.envelope(...)` then `wwex.shopFlowDomestic(...)`. |
| **WWEX API** | `wwex_api` | draft | *(none)* | Empty shell, unused. |

---

## 6. UI Layer

### 6.1 Catalog Item

| Attribute            | Value                                                                                   |
| -------------------- | --------------------------------------------------------------------------------------- |
| **Name**             | WWEX Shipment (`756eb9b8fba2a610842efe557eefdc6e`)                                      |
| **Short description**| WWEX Domestic Shipment Req.                                                             |
| **Catalogs**         | Service Catalog, Stave Catalog                                                          |
| **Category**         | Vendor Catalogs                                                                         |
| **Variable sets**    | Standard Employee Questions (100), WWEX Variable Set (200)                              |
| **Request method**   | `request`; cart + quantity enabled; availability `on_desktop`; not hidden on portal     |
| **Fulfillment**      | Flow **WWEX Shipment Request** (catalog trigger)                                        |

### 6.2 Catalog Client Scripts

All on `WWEX Shipment`. Source: update/catalog_script_client_*.xml

| Name | Type | Variable | UI type | Active | Purpose |
| ---- | ---- | -------- | ------- | :----: | ------- |
| **Shipment Type by Address Country** | onChange | `country_origin` | All | Y | Sets `shipment_type = domestic` when the value is `United States`, otherwise `international`. ⚠ The v2 widget writes the ISO short code (`US`), so this currently always selects **international**. |
| **AddressOnChange** | onChange | `mapped_address` | All | Y | Splits a `street, city, ST ZIP, country` string into the `*_origin` variables (country defaults to `USA`). Only fires for the inactive legacy widget path. |
| **TestingGlideAjax** | onChange | *(none)* | Desktop | Y | Debug script calling `GooglePlacesAjax.getPlaceSuggestions` (not in this app) to fill `address_suggestions` (no such variable). Has no variable, so it never fires. |
| Prevent submit without Mapped Address | onSubmit | — | All | N | Blocks submit when `mapped_address` is empty. |
| Google Address Validation | onChange | — | All | N | Calls `GoogleAddressValidation.validateAddress` (not in this app). |
| Google Address Auto-Complete | onLoad | — | All | N | DOM-injected suggestion dropdown on `address_auto_complete_v4`. |

### 6.3 Catalog UI Policies

| Short description | Applies to | Condition | Action |
| ----------------- | ---------- | --------- | ------ |
| **Add Insurance = Yes, show Package Value** | WWEX Variable Set (on load, reverse if false) | `add_insureance = Yes` | `package_value` → visible + mandatory. |

### 6.4 App Modules (Navigation)

None. The app ships no application menu or modules — users reach it only through the catalog item.

---

## 7. Service Portal Integration

### 7.1 Widgets

| Attribute | **Google Address Auto Complete Custom v2** | **Google Address Auto Complete Custom** (legacy) |
| --------- | ------------------------------------------ | ------------------------------------------------ |
| **ID / sys_id** | `google_address_auto_complete_custom_v2` / `5e5491a11baf3a10509b20e4604bcbe1` | `google_address_autocomplete_custom` / `e6fbbb77879f3210b4fa426acebb3555` |
| **Used by** | `pickup_address_autocomplete`, `dropoff_address_autocomplete` | `address_autocomplete_custom` (inactive) |
| **Server** | Empty IIFE. | Sets a test `data.message`. |
| **Client** | Lazy-loads the Google Maps JS API (`libraries=places`) once, attaches `google.maps.places.Autocomplete` (`types: ['address']`) to `#autocomplete_<field>`. On `place_changed` builds `street, city, ST ZIP, CC`, writes it to its own variable, and writes the parts to the `*_origin` variables when the field is `pickup_address_autocomplete`, otherwise to the `*_destination` variables. | Attaches Autocomplete to `#custom-address-input`, validates with a US-format regex (`ST 12345[-6789]`), writes the result to `mapped_address`, shows an inline error on incomplete addresses. Google script loaded from the template. |
| **Template** | Label + text input bound to `field.value`; `.pac-container` z-index fix. | Labelled input + error message; styled `.pac-*` dropdown. |

**Dependency:** both widgets need a **Google Maps JavaScript API key with Places enabled**. The key is hard-coded in the widget source (§8.4).

---

## 8. Security Model

### 8.1 Roles

None. The app ships no `sys_user_role` records and the manifest has no `user_role`. The catalog item has no role restriction, so anyone who can see the two catalogs can submit it.

### 8.2 ACLs

None. With no custom tables there are no `sys_security_acl` records. Script Includes are `package_private`, so they can only be called from within the `x_stave_wwex_sn` scope (flows/actions in this app).

### 8.3 Cross-Scope Privileges

**12** `sys_scope_privilege` records, all `allowed`:

| Target | Scope | Type | Operation |
| ------ | ----- | ---- | --------- |
| `oauth_credential` | Global | table | read, create, write |
| `GlideRecord.setValue` / `.insert` / `.update` | Global | scriptable | execute |
| `ScriptableRESTMessageClient.execute` / `.setStringParameterNoEscape` | Global | scriptable | execute |
| `ScriptableRESTResponse.getBody` / `.getStatusCode` | Global | scriptable | execute |
| `Glide API: properties` | Global | scriptable | execute |
| `x_stave_sc_fleet` | Stave Supply Core | table | read — *not used by any current script* |

### 8.4 Credentials & Secrets in Source

> ⚠ **Action needed.** The update set XML contains live-looking secrets in plain text. Values are intentionally not reproduced here.

| Where | What |
| ----- | ---- |
| `sys_properties` `x_stave_wwex_sn.client_id` / `.client_secret` | WWEX client ID and secret as plain `string` properties (`is_private = true` keeps them out of update sets on export, but they are present in this repo). |
| REST method *Get Token* parameters `client_id` / `client_secret` | Same values as default parameter values. |
| REST methods *AddressValidationFlow* and *shopFlow Domestic SAMPLE*, parameter `token` | A previously issued JWT bearer token stored as the default value. |
| `sys_auth_profile_basic` **WWEX Basic Auth**, `oauth_entity` ×2 | Client ID stored as username / client_id. |
| Both SP widgets | Google Maps API key embedded in client script / template. |

Recommended: rotate the WWEX client secret and Google key, change the properties to type `password2`, read them with `gs.getProperty()` in `getAuthToken()`, clear the default `token` / `client_secret` parameter values, and restrict the Google key by HTTP referrer.

---

## 9. Configuration

### 9.1 System Properties

| Property | Type | Private | Purpose |
| -------- | ---- | :-----: | ------- |
| `x_stave_wwex_sn.client_id` | string | Y | WWEX OAuth client ID. |
| `x_stave_wwex_sn.client_secret` | string | Y | WWEX OAuth client secret. |

Both were added on 2026-03-11 and are **not yet read by any script** — token calls still use the auth profile / REST parameters.

### 9.2 Choice Lists

| Field / choice set | Values |
| ------------------ | ------ |
| `shipment_type` (question_choice) | `domestic` Domestic (100), `international` International (200) |
| *Envelope Address* input `sender_receiver` | `sender` Sender, `receiver` Receiver |

### 9.3 REST Messages

**WWEX API** (`a9f5dd8087c1aa10ed1f4048dabb359d`) — *"shopFlow Domestic - SpeedShip"*, base endpoint `https://speedship.staging-wwex.com/svc/shopFlow`, parent OAuth profile *WWEX OAuth Profile*.

| HTTP method | Verb | Endpoint | Auth | Headers | Parameters |
| ----------- | ---- | -------- | ---- | ------- | ---------- |
| **Get Token (Basic Auth)** | POST | `https://auth.staging-wwex.com/oauth/token` | Basic — *WWEX Basic Auth* | `content-type: application/json` | `grant_type`, `audience` |
| **Get Token** | POST | same | none | `content-type: application/json` | `grant_type`, `client_id`, `client_secret`, `audience` |
| **AddressValidationFlow** | POST | `https://speedship.staging-wwex.com/svc/addressValidationFlow` | Basic + OAuth profile set, but the request uses the explicit header | `Content-Type: application/json`, `Authorization: Bearer ${token}` | `token`, `body` |
| **shopFlow Domestic SAMPLE** | POST | `https://speedship.staging-wwex.com/svc/shopFlow` | none (header) | `Content-Type: application/json`, `Authorization: Bearer ${token}` | `token`, `body` (sample WWEX payload as default) |
| Default GET | GET | `…/svc/shopFlow` | inherit | — | — |

A second, older **WWEX API** record (`293157691bab7a10509b20e4604bcbec`) and its *Default GET* method are shipped as **deletes** under author_elective_update/.

### 9.4 OAuth & Authentication Profiles

| Record | Type | Details |
| ------ | ---- | ------- |
| **WWEX OAuth v2** (`oauth_entity`) | OAuth provider | Token URL `auth.staging-wwex.com/oauth/token`, default grant `client_credentials`, access token lifespan 1800 s. Profile **WWEX OAuth v2 default_profile** (client_credentials). |
| **WWEX SpeedShip OAuth** (`oauth_entity`) | OAuth provider | Same token URL / grant; redirect URL is malformed (instance URL duplicated). Profile **WWEX OAuth Profile** (*authorization_code* grant) — attached to the REST message but not used by scripts. |
| **WWEX Basic Auth** (`sys_auth_profile_basic`) | Basic auth | Client ID / secret pair for the token and address-validation calls. |

### 9.5 Scheduled Jobs

None. Everything runs on catalog submission. The token is refreshed lazily whenever `wwex_api_helper` is instantiated.

### 9.6 Other

- **Images** — three `db_image` records (2020 Chevrolet Spark photos) are leftovers, not referenced anywhere.
- **Dependency** — `sys_package_dependency_m2m` on **Stave Buy It** ≥ 1.5.8 (the manifest short description ties the app to Buy It).

---

## 10. Architecture

```
                 ┌──────────────────────────────────────────────────────────────────┐
                 │                     Stave WWEX Integration                       │
                 └──────────────────────────────────────────────────────────────────┘

  ── Service Portal ──                ── Flow Designer ──                         ── WWEX (staging) ──

  WWEX Shipment (catalog item)
   ├─ Pickup Address  ─┐ Google Places
   ├─ Shipping Address ┘ widget v2 ──► *_origin / *_destination variables
   ├─ Shipment Type / Insurance
   └─ submit ──► RITM ──► WWEX Shipment Request flow
                            1 Get Catalog Variables
                            2 Envelope Address (sender) ──► wwex_api_helper.addressValidation ──► /svc/addressValidationFlow
                            3 Envelope Address (receiver, hard-coded)  ──► (same)
                            4 Envelope Body  (SMALLPACK request, sample package data)
                            5 API Request    ──► wwex_api_helper.shopFlowDomestic ─────────────► /svc/shopFlow
                                                        │
                                  getValidToken() ◄─────┘
                                  oauth_credential (scopes = wwex_client_credential)
                                  └─ refresh via Get Token (Basic Auth) ──────────────────────► auth…/oauth/token
```

**Request flow:**
1. The requester submits **WWEX Shipment**; the v2 widget has already split each Google address into separate variables.
2. The catalog trigger starts **WWEX Shipment Request** on the RITM.
3. *Get Catalog Variables* exposes the variables to the flow.
4. *Envelope Address* (×2) instantiates `wwex_api_helper` (which ensures a valid bearer token), validates the address with WWEX, and returns a WWEX `address` object with a `SENDER` / `RECEIVER` contact list.
5. *Envelope Body* stamps the current time and wraps both addresses in a `SMALLPACK` shipment request.
6. *API Request* wraps it as `{ request, correlationId: <RITM number> }` and POSTs to `shopFlow`. The response is kept only as a flow output.

---

## 11. Extension Points

- **Drive the receiver from the form.** Map step 3 of *WWEX Shipment Request* to the `*_destination` variables instead of the hard-coded Dallas address.
- **Package details.** Replace the fixed dimensions / weight / insured value / COD in *Envelope Body* with inputs (e.g. `package_value` when `add_insureance = Yes`).
- **International shipments.** `addressValidation` always sends `isInternational = false`; pass `shipment_type` through to the validation and shopFlow bodies.
- **Use corrected addresses.** `correctedAddress()` already extracts WWEX's corrected address — feed it into `domesticShipment.address()` instead of the raw input.
- **Persist results.** Write the `shopFlow` response (quotes / IDs) back to the RITM (work notes or variables) instead of leaving it only in the flow execution.
- **New endpoints.** Add HTTP methods to the **WWEX API** REST message and a matching method on `wwex_api_helper` that sets the `token` parameter from `this.token`.

---

## 12. Appendix — Key Identifiers & File Map

| Artifact | sys_id | Source file |
| -------- | ------ | ----------- |
| Application | `db9848f1875bbe50b4fa426acebb35e1` | sys_app_db9848f1875bbe50b4fa426acebb35e1.xml |
| Catalog item `WWEX Shipment` | `756eb9b8fba2a610842efe557eefdc6e` | update/sc_cat_item_756eb9b8fba2a610842efe557eefdc6e.xml |
| Variable set `WWEX Variable Set` | `49f417f3fbdf7ad00030f9837befdcfd` | update/item_option_new_set_49f417f3fbdf7ad00030f9837befdcfd.xml |
| Script Include `wwex_api_helper` | `8c2022a187452650ed1f4048dabb352a` | update/sys_script_include_8c2022a187452650ed1f4048dabb352a.xml |
| Script Include `domesticShipment` | `41d5f9c1fb6ae610842efe557eefdc3a` | update/sys_script_include_41d5f9c1fb6ae610842efe557eefdc3a.xml |
| Flow `WWEX Shipment Request` | `300be1f8fb22a610842efe557eefdcb9` | update/sys_hub_flow_300be1f8fb22a610842efe557eefdcb9.xml |
| Flow `Shop Flow Domestic` (inactive) | `e51c7362fbaea2100030f9837befdcfb` | update/sys_hub_flow_e51c7362fbaea2100030f9837befdcfb.xml |
| Subflow `Shipping Address` (inactive) | `92781ac5fb2ee610842efe557eefdc07` | update/sys_hub_flow_92781ac5fb2ee610842efe557eefdc07.xml |
| Action `Envelope Address` | `38a4f281fbeee610842efe557eefdcd7` | update/sys_hub_action_type_definition_38a4f281fbeee610842efe557eefdcd7.xml |
| Action `Envelope Body` | `600dae9efbea22100030f9837befdc49` | update/sys_hub_action_type_definition_600dae9efbea22100030f9837befdc49.xml |
| Action `API Request` | `9c970b56fbae22100030f9837befdca1` | update/sys_hub_action_type_definition_9c970b56fbae22100030f9837befdca1.xml |
| Action `WWEX API` (empty) | `71dc21bcfb22a610842efe557eefdc50` | update/sys_hub_action_type_definition_71dc21bcfb22a610842efe557eefdc50.xml |
| REST Message `WWEX API` | `a9f5dd8087c1aa10ed1f4048dabb359d` | update/sys_rest_message_a9f5dd8087c1aa10ed1f4048dabb359d.xml |
| OAuth `WWEX OAuth v2` | `19b37421fbc7e2100030f9837befdcc0` | update/oauth_entity_19b37421fbc7e2100030f9837befdcc0.xml |
| OAuth `WWEX SpeedShip OAuth` | `eec51d8087c1aa10ed1f4048dabb3540` | update/oauth_entity_eec51d8087c1aa10ed1f4048dabb3540.xml |
| Basic Auth `WWEX Basic Auth` | `062c5eb9fb9ea2d00030f9837befdc32` | update/sys_auth_profile_basic_062c5eb9fb9ea2d00030f9837befdc32.xml |
| Widget `Google Address Auto Complete Custom v2` | `5e5491a11baf3a10509b20e4604bcbe1` | update/sp_widget_5e5491a11baf3a10509b20e4604bcbe1.xml |
| Widget `Google Address Auto Complete Custom` | `e6fbbb77879f3210b4fa426acebb3555` | update/sp_widget_e6fbbb77879f3210b4fa426acebb3555.xml |
| Property `client_id` | `1e9c9fa91beb7a10509b20e4604bcbbd` | update/sys_properties_1e9c9fa91beb7a10509b20e4604bcbbd.xml |
| Property `client_secret` | `6d4c572d1beb7a10509b20e4604bcb58` | update/sys_properties_6d4c572d1beb7a10509b20e4604bcb58.xml |
| Dependency on Stave Buy It | `428c0831879bbe50b4fa426acebb35f9` | update/sys_package_dependency_m2m_428c0831879bbe50b4fa426acebb35f9.xml |

---

## 13. Version Notes

- **1.0.0** — Current version. App record created 2026-02-26; the catalog item, flows, actions and script includes date from April–October 2025 (`jose.cornelio@stavecorp.com`, `nrivera`, `Edgard.Ordonez@stavecorp.com`); the shipment type, pickup/drop-off widgets, destination variables, insurance fields and credential properties were added in March 2026 (`richard.castillo`).
- **Work in progress.** The main flow and the *Envelope Address* action have unpublished drafts; the published flow snapshot only logs. The receiver address and all package data are hard-coded test values, and every endpoint targets WWEX **staging**.
- **Known issues** — `Shipment Type by Address Country` compares against `United States` while the widget writes `US`; the published flow calls a removed `infoMessage()` method; `getAuthToken()` hard-codes the audience; the new `client_id` / `client_secret` properties aren't read yet; secrets are committed in source (§8.4).
- **Vestigial artifacts** — inactive `Shop Flow Domestic` flow and `Shipping Address` subflow, empty `WWEX API` action, three legacy autocomplete variables and their client scripts, the unused `x_stave_sc_fleet` privilege, Chevrolet Spark `db_image` records, and a deleted duplicate REST message.
- **Repository convention** — Delivered as ServiceNow-generated XML in x_stave_wwex_sn/db9848f1875bbe50b4fa426acebb35e1/. The README.md inside is the generic ServiceNow recovery guide, not app-specific.

---

*Document generated from source XML on 2026-10-12 against scope `x_stave_wwex_sn` v1.0.0.*
