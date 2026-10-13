# Stave WWEX Integration (`x_stave_wwex_sn`) — Documentation

> ServiceNow scoped application that connects **Service Catalog shipment requests to the Worldwide Express (WWEX) SpeedShip API**. A requester fills in the **WWEX Shipment** catalog item (pickup and drop-off addresses with Google Places autocomplete, shipment type, optional insurance); a Flow Designer flow then builds the WWEX shipment envelope, validates addresses with WWEX, and submits the request to SpeedShip using an OAuth 2.0 client-credentials token that the app manages automatically.

---

## 1. At-a-Glance

| Attribute             | Value                                  |
| --------------------- | -------------------------------------- |
| **Name**              | Stave WWEX Integration                 |
| **App name (instance)** | WWEX for ServiceNow                  |
| **Scope**             | `x_stave_wwex_sn`                      |
| **Vendor**            | Stave                                  |
| **Version**           | 1.0.0                                  |
| **JS Level**          | `es_latest`                            |
| **Requires**          | Stave Buy It (`x_stave_buy_it`) 1.5.8+ |

**Short description:** *WWEX for Buy It in ServiceNow.*

---

## 2. Capabilities Overview

The application is organized around **one catalog front end** and **one outbound integration pipeline**, plus automatic token management.

| Pillar                    | Trigger model                         | Output                                                                                              |
| ------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Shipment request form** | User submits catalog item             | A requested item (RITM) holding pickup / drop-off addresses, phone, shipment type, and insurance.   |
| **Address autocomplete**  | User typing in the portal             | Google Places suggestions split into street, city, region, postal code, and country fields.         |
| **Address validation**    | Automatic, during the flow            | Each address is checked with WWEX before it is added to the shipment.                               |
| **Shipment submission**   | Automatic, during the flow            | The shipment request is posted to WWEX SpeedShip with the RITM number as the correlation ID.        |
| **Token management**      | Automatic, on every WWEX call         | A WWEX bearer token is stored and reused, and refreshed when less than one hour of validity is left. |

**Shipment request flow (conceptual):**
1. The requester submits **WWEX Shipment** from the Service Portal.
2. The **WWEX Shipment Request** flow starts on the new RITM and reads the catalog variables.
3. The sender (pickup) and receiver addresses are validated with WWEX and converted into WWEX address objects with sender / receiver contacts.
4. The flow builds a `SMALLPACK` shipment request time-stamped with the current date/time.
5. The request is sent to WWEX SpeedShip (`shopFlow`). The submitted body and WWEX's response are available on the flow execution.

---

## 3. Usage / How-To

### 3.1 Submit a shipment request

1. In the Service Portal, open **Service Catalog** (or **Stave Catalog**) → **Vendor Catalogs** → **WWEX Shipment**.
2. Fill in the standard requester questions (the *Requested for* user becomes the sender contact).
3. In **Pickup Address**, start typing and pick a suggestion. Street, City, Region, Postal Code, and Country fill in automatically.
4. Enter the sender **Phone**.
5. In **Shipping Address**, pick the drop-off address the same way.
6. Check **Shipment Type** (`Domestic` / `International`). It is pre-selected from the pickup country — change it if needed.
7. (Optional) Set **Add Insureance = Yes** and enter **Package Value ($)** (required once insurance is selected).
8. Submit. The shipment is processed automatically.

### 3.2 Review what was sent to WWEX

1. Open the requested item and go to its flow execution (**Flow Designer → Executions**, filtered by the RITM).
2. Open the **API Request** step:
   - `request_body` — the shipment envelope sent to WWEX (`correlationId` = RITM number).
   - `response` — the response returned by WWEX.
3. Open the **Envelope Address** steps to see each address and its validation status. An address WWEX can't validate shows as `Address not valid`.

### 3.3 Configure your WWEX credentials

1. Get an API client ID and secret from WWEX.
2. Set them in **System Properties** `x_stave_wwex_sn.client_id` and `x_stave_wwex_sn.client_secret`.
3. Enter the same pair on the **WWEX Basic Auth** profile and the **WWEX OAuth v2** application registry.
4. Remove any previously stored WWEX token (see §5.3) so a new one is requested with the new credentials.

### 3.4 Move from WWEX staging to production

The app ships configured for the **WWEX staging environment**. To go live:

1. On REST Message **WWEX API**, replace the staging hosts (`speedship.staging-wwex.com`, `auth.staging-wwex.com`) with the production hosts provided by WWEX.
2. Update the `audience` value used to request tokens to the production audience.
3. Update the token URL on the OAuth application registries.
4. Enter production credentials (§3.3).

---

## 4. Service Portal Integration

The catalog item uses custom Service Portal widgets for address entry.

### 4.1 Widgets

| Attribute     | Value                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**      | `Google Address Auto Complete Custom v2`                                                                                                    |
| **Used by**   | **Pickup Address** and **Shipping Address** variables.                                                                                      |
| **Behavior**  | Loads Google Places autocomplete on the field. When the user picks an address, it fills the related origin (pickup) or destination (drop-off) fields. |
| **Requires**  | A Google Maps JavaScript API key with the **Places** API enabled.                                                                           |

A previous version, `Google Address Auto Complete Custom`, is still shipped but is attached only to an inactive variable.

---

## 5. Configuration

### 5.1 System Properties

| Property                        | Type   | Purpose                    |
| ------------------------------- | ------ | -------------------------- |
| `x_stave_wwex_sn.client_id`     | string | WWEX OAuth client ID.      |
| `x_stave_wwex_sn.client_secret` | string | WWEX OAuth client secret.  |

### 5.2 Choice Lists

- `shipment_type` — Domestic (`domestic`), International (`international`).
- *Envelope Address* action `sender_receiver` — Sender (`sender`), Receiver (`receiver`).

### 5.3 Integration Endpoints & Authentication

| Record | Purpose |
| ------ | ------- |
| REST Message **WWEX API** | Holds every WWEX call: **Get Token** / **Get Token (Basic Auth)** (`/oauth/token`), **AddressValidationFlow** (`/svc/addressValidationFlow`), and **shopFlow Domestic SAMPLE** (`/svc/shopFlow`). |
| Basic Auth profile **WWEX Basic Auth** | Client credentials used when requesting tokens and validating addresses. |
| OAuth registries **WWEX OAuth v2** / **WWEX SpeedShip OAuth** | WWEX token endpoint with the `client_credentials` grant. |
| Token store | The current bearer token is kept in the platform OAuth credential table (`oauth_credential`) under scope `wwex_client_credential`. Delete that row to force a new token. |

### 5.4 Scheduled Jobs

None. Shipments are processed **on demand** when the catalog item is submitted, and tokens are refreshed only when needed.

### 5.5 Navigation

The app has no application menu. Users reach it through the **WWEX Shipment** catalog item in **Service Catalog** and **Stave Catalog** (category **Vendor Catalogs**).

---

## 6. Data Model — Table Reference

The app ships **no custom tables**. Each shipment is a standard requested item (`sc_req_item`) whose data is held in the catalog variables below.

### 6.1 `WWEX Variable Set` — catalog variables

| Variable                       | Label                        | Type              | Notes                                                                 |
| ------------------------------ | ---------------------------- | ----------------- | --------------------------------------------------------------------- |
| `pickup_address_autocomplete`  | Pickup Address               | Custom (widget)   | Google autocomplete; fills the origin fields.                         |
| `dropoff_address_autocomplete` | Shipping Address             | Custom (widget)   | Google autocomplete; fills the destination fields.                    |
| `phone_origin`                 | Phone                        | Single Line Text  | Sender phone.                                                         |
| `shipment_type`                | Shipment Type                | Select Box        | **Mandatory.** Domestic / International.                              |
| `street_origin`                | Street                       | Single Line Text  | Pickup street (number + route).                                       |
| `locality_origin`              | City                         | Single Line Text  |                                                                       |
| `region_origin`                | Region                       | Single Line Text  | State / province code.                                                |
| `postal_code_origin`           | Postal Code                  | Single Line Text  |                                                                       |
| `country_origin`               | Country                      | Single Line Text  | ISO country code (e.g. `US`).                                         |
| `street_destination`           | Street Destination           | Single Line Text  | Drop-off street.                                                      |
| `locality_destination`         | Locality Destination         | Single Line Text  |                                                                       |
| `region_destination`           | Region Destination           | Single Line Text  |                                                                       |
| `postal_code_destination`      | Postal Code Destination      | Single Line Text  |                                                                       |
| `country_destination`          | Country Destination          | Single Line Text  |                                                                       |
| `add_insureance`               | Add Insureance               | Yes / No          | Shows **Package Value** when `Yes`. (Label spelled this way on the form.) |
| `package_value`                | Package Value ($)            | Single Line Text  | Required when insurance is requested.                                 |

The origin fields are grouped in a **Validated Address** section on the form. The item also includes the **Standard Employee Questions** variable set (requested for, etc.).

### 6.2 WWEX request envelope

What the flow sends to `shopFlow`:

| Element | Source |
| ------- | ------ |
| `correlationId` | RITM number. |
| `request.productType` | `SMALLPACK`. |
| `request.shipment.shipmentDate` | Current date/time (`YYYY-MM-DD HH:mm:ss`). |
| `request.shipment.originAddress.address` | Pickup address + `SENDER` contact (requested for, phone). |
| `request.shipment.destinationAddress.address` | Receiver address + `RECEIVER` contact. |
| `request.shipment.handlingUnitList`, weights, insurance, COD flags | Package and service options. |

---

## 7. Extension Points

- **Package details.** Package dimensions, weight, insured value, and COD options are set in the **Envelope Body** Flow Designer action. Map them to catalog variables (e.g. **Package Value**) to collect them from the requester.
- **Receiver address.** The receiver step of **WWEX Shipment Request** can be mapped to the drop-off (destination) variables.
- **International shipments.** Pass **Shipment Type** into the WWEX requests to enable international handling.
- **Store results.** Add a flow step that writes the WWEX response back to the requested item (work notes or variables).
- **More WWEX services.** Add HTTP methods to the **WWEX API** REST message and call them from the app's API helper.

---

## 8. Version Notes

- **1.0.0** — Current version.
- This app is delivered as a ServiceNow scoped application and requires **Stave Buy It**. It ships configured for the WWEX **staging** environment, and parts of the shipment payload (receiver address, package details) currently use fixed sample values — see §3.4 and §7 before going to production.
