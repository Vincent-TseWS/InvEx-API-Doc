### WMS Core — Public SAP/ERP API Guide

This document describes the **public** HTTP APIs exposed by WMS Core for SAP/ERP and other external systems.

It focuses on:

- **Authentication & headers**
- **Item Master** (SKU) synchronization
- **Inbound Orders** (receiving)
- **Outbound Orders** (shipping)

Other internal or admin APIs are intentionally omitted.

**Base URL:** `{host}/api/`  
**Content-Type:** `application/json`

---

### 1. Authentication

All endpoints in this guide require authentication via **Token** in the `Authorization` header.

- **Headers**
  - `Authorization: Token <your-token>` (required)
  - `Content-Type: application/json` (required for POST/PUT)

Tokens are provisioned by WMS Core and may represent:

- A **user token** — for staff or integration users.
- A **client token** — for client/tenant-scoped integrations such as SAP.

If the token is invalid or missing, the API returns `401 Unauthorized`.

**Example (invalid token)**

```json
{
  "result": "Failed",
  "data": null,
  "error": "Unauthorized"
}
```

---

### 2. Item Master APIs

These APIs keep item master (SKU) data in sync between WMS Core and your ERP/SAP.

All endpoints below are relative to `{host}/api/`.

#### 2.1 List Item Master

- **Endpoint:** `GET /api/item_master/`
- **Description:** List item master (SKU) records.

**Query Parameters**

| Parameter         | Type    | Required | Description                       |
| ----------------- | ------- | -------- | --------------------------------- |
| `skip`            | integer | No       | Pagination offset. Default: `0`.  |
| `take`            | integer | No       | Page size. Default: `20`, max 100 |
| `client_reference`| string  | No       | Filter by client reference.       |

**Example Request**

```http
GET /api/item_master/?skip=0&take=20 HTTP/1.1
Host: wms-core.example.com
Authorization: Token abc123
```

**Example Success Response (200)**

```json
{
  "result": "Success",
  "data": [
    {
      "client": "DEMOCLIENT",
      "item_type": "STANDARD",
      "product_name": "Sample Product",
      "item_code": "SKU001",
      "description": "Product description",
      "active": true,
      "group_name": "RAW_MATERIAL",
      "unit": "EACH",
      "stock": 1000
    }
  ],
  "error": null
}
```

---

#### 2.2 SAP Item Master Create / List

- **Endpoint:** `GET /api/sap/item_master/`
- **Endpoint:** `POST /api/sap/item_master/`
- **Description:** Create and list item master (SKU) records for SAP integrations.

##### 2.2.1 Request Fields (POST)

**Body (JSON)**

| Field                               | Type    | Required | Description                                                                                          |
| ----------------------------------- | ------- | -------- | ---------------------------------------------------------------------------------------------------- |
| `client` / `client_code`           | string  | **Yes**  | Client code. Must exist under the token's organization.                                             |
| `item_type`                        | string  | **Yes**  | Item type code. Must exist in system (e.g., STANDARD).                                              |
| `product_name`                     | string  | **Yes**  | Product display name.                                                                               |
| `item_code` / `code`               | string  | **Yes**  | SKU/item code. Must be unique per client.                                                           |
| `description`                      | string  | No       | Product description.                                                                                |
| `upc`                              | string  | No       | UPC/barcode.                                                                                        |
| `foreign_name`                     | string  | No       | Foreign language name.                                                                              |
| `group_name`                       | string  | No       | Product grouping name (category).                                                                   |
| `remark`                           | string  | No       | Internal remark.                                                                                    |
| `auom_control`                     | string  | No       | Alternative UOM control. Allowed: `DISABLE`, `IN`, `OUT`, `BOTH`. Default: `DISABLE`.               |
| `expiry_date_control`             | string  | No       | Expiry date control. Allowed: `DISABLE`, `OUT`, `BOTH`. Default: `DISABLE`.                         |
| `lot_control`                      | string  | No       | Lot number control. Allowed: `DISABLE`, `OUT`, `BOTH`. Default: `DISABLE`.                          |
| `serial_number_control`           | string  | No       | Serial number control. Allowed: `DISABLE`, `OUT`, `BOTH`. Default: `DISABLE`.                       |
| `required_condition`              | string  | No       | Zone condition code (e.g., ambient, chilled). Must exist if provided.                               |
| `force_required_condition`        | boolean | No       | Enforce required condition. Default: `false`.                                                       |
| `picking_method`                  | string  | No       | `FIFO`, `FEFO`, or `LIFO`. Default: `FEFO` if expiry control enabled, else `FIFO`.                  |
| `hs_code`                          | string  | No       | Harmonized System code.                                                                            |
| `custom_code`                      | string  | No       | Custom/customer reference code.                                                                     |
| `pick_size`                        | integer | No       | Pick size. Default: `0`.                                                                            |
| `pack_size`                        | integer | No       | Pack size. Default: `0`.                                                                            |
| `stock`                            | number  | No       | Stock quantity value. Default: `0`.                                                                 |
| `shelf_life`                      | integer | No       | Shelf life value (days). Default: `0`.                                                              |
| `pack_description`                | string  | No       | Pack description text.                                                                              |
| `team`                             | string  | No       | Team name/value.                                                                                    |
| `brand`                            | string  | No       | Brand name.                                                                                         |
| `business_partner_code`           | string  | No       | Business partner code.                                                                              |
| `business_partner_name`           | string  | No       | Business partner name.                                                                              |
| `place_of_origin`                 | string  | No       | Place of origin.                                                                                    |
| `container_type`                  | string  | No       | Container type.                                                                                     |
| `wight_per_each`                  | number  | No       | Weight per each item. Default: `0`.                                                                 |
| `weight_unit`                     | string  | No       | Weight unit (e.g., `kg`, `g`).                                                                      |
| `cost`                             | number  | No       | Cost price. Default: `0`.                                                                           |
| `selling`                          | number  | No       | Selling price. Default: `0`.                                                                        |
| `msrp`                             | number  | No       | MSRP. Default: `0`.                                                                                 |
| `active`                           | boolean | No       | Whether the item is active. Default: `true`. Accepts `true`/`false`, `"true"`/`"false"`, `1`/`0`.   |
| `uom`                              | array   | No       | Array of unit-of-measurement objects (see below).                                                   |
| `custom_field`                    | array   | No       | Array of custom field objects (see below).                                                          |

**UOM Object (each item in `uom`)**

| Field         | Type    | Required | Description                                              |
| ------------- | ------- | -------- | -------------------------------------------------------- |
| `unit`        | string  | **Yes**  | Unit code (e.g., `EACH`, `CTN`). Must exist in system.  |
| `qty`         | number  | No       | Conversion factor to base unit. Default: `0` (ignored if < 1). |
| `is_base`     | boolean | No       | Whether this is the base UOM. Default: `false`.         |
| `length`      | number  | No       | Length (cm). Default: `0`.                              |
| `width`       | number  | No       | Width (cm). Default: `0`.                               |
| `height`      | number  | No       | Height (cm). Default: `0`.                              |
| `gross_weight`| number  | No       | Gross weight (kg). Default: `0`.                        |
| `net_weight`  | number  | No       | Net weight (kg). Default: `0`.                          |

**Custom Field Object (each item in `custom_field`)**

| Field  | Type   | Required | Description        |
| ------ | ------ | -------- | ------------------ |
| `name` | string | **Yes**  | Custom field name. |
| `value`| any    | **Yes**  | Custom field value.|

##### 2.2.2 Example POST Request

```http
POST /api/sap/item_master/ HTTP/1.1
Host: wms-core.example.com
Authorization: Token abc123
Content-Type: application/json

{
  "client": "DEMOCLIENT",
  "item_type": "STANDARD",
  "product_name": "Sample Product",
  "item_code": "SKU001",
  "description": "Product description",
  "upc": "123456789012",
  "group_name": "RAW_MATERIAL",
  "expiry_date_control": "BOTH",
  "picking_method": "FEFO",
  "active": true,
  "uom": [
    {
      "unit": "EACH",
      "qty": 1,
      "is_base": true
    }
  ],
  "custom_field": []
}
```

##### 2.2.3 Example Success Response (201)

```json
{
  "data": {
    "client": "DEMOCLIENT",
    "item_type": "STANDARD",
    "product_name": "Sample Product",
    "item_code": "SKU001",
    "description": "Product description",
    "active": true,
    "uom": [
      {
        "unit": "EACH",
        "qty": 1,
        "is_base": true
      }
    ],
    "custom_field": []
  },
  "error": null
}
```

##### 2.2.4 Example Error Responses

- **Duplicate SKU / client combination (400)**

```json
{
  "data": null,
  "error": "Item code 'SKU001' already exists for client 'DEMOCLIENT'"
}
```

- **Invalid control code (400)**

```json
{
  "data": null,
  "error": "Invalid expiry_date_control. Allowed values: DISABLE, OUT, BOTH"
}
```

---

#### 2.3 SAP Item Master Update / Detail

- **Endpoint:** `GET /api/sap/item_master/<code>/`
- **Endpoint:** `PUT /api/sap/item_master/<code>/`
- **Description:** Get and update an existing item master by SKU code.

**Update Rules**

- Only fields provided in the payload are updated; omitted fields keep current values.
- `client`, `item_type`, and `item_code` **cannot** be changed via `PUT`.
- Control fields (`auom_control`, `expiry_date_control`, `lot_control`, `serial_number_control`) may be restricted when the SKU has inventory; updates may be rejected if stock exists.
- UOM updates may be restricted once UOMs exist; in many cases `uom` is only applied when the SKU has **no existing UOMs**.

**Example PUT Request**

```http
PUT /api/sap/item_master/SKU001/ HTTP/1.1
Host: wms-core.example.com
Authorization: Token abc123
Content-Type: application/json

{
  "product_name": "Updated Product Name",
  "description": "Updated description",
  "active": true,
  "remark": "Updated remark",
  "brand": "BrandY"
}
```

**Example Success Response (200)**

```json
{
  "data": {
    "client": "DEMOCLIENT",
    "item_type": "STANDARD",
    "product_name": "Updated Product Name",
    "item_code": "SKU001",
    "description": "Updated description",
    "active": true,
    "remark": "Updated remark",
    "brand": "BrandY"
  },
  "error": null
}
```

---

### 3. Inbound Order APIs (Receiving)

Inbound order APIs manage stock **receiving** from vendors.

All endpoints are relative to `{host}/api/`.

#### 3.1 Create Inbound Order (Batch, SAP-style)

- **Endpoint:** `POST /api/sap/inbound/`
- **Description:** Create one or more inbound (receiving) orders in a single request.

##### 3.1.1 Request Body

| Field        | Type   | Required | Description                                                                                                                                              |
| ------------ | ------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `batch_code` | string | No       | Upload batch identifier for tracking (e.g., Excel batch).                                                                                               |
| `submit`     | bool   | No       | If `true`, orders are auto-submitted after creation (status `DRAF → SUBM`). Default: `false`.                                                           |
| `orders`     | array  | **Yes**  | Array of inbound order objects. At least one order required.                                                                                            |

**Order Object (each item in `orders`)**

| Field                    | Type   | Required | Description                                                                                              |
| ------------------------ | ------ | -------- | -------------------------------------------------------------------------------------------------------- |
| `row_no`                 | string | No       | Row identifier for error reporting.                                                                      |
| `organization_code`      | string | **Yes**  | Organization/warehouse code. Must exist in system.                                                       |
| `client_code`            | string | **Yes**  | Client code. Must exist under the organization.                                                          |
| `vendor`                 | object | **Yes**  | Vendor (supplier) information (see below).                                                               |
| `courier_code`           | string | No       | Courier code.                                                                                            |
| `courier_name`           | string | No       | Courier display name (informational).                                                                    |
| `client_reference`       | string | **Yes**  | Unique order reference from your system. Must not duplicate other active orders.                         |
| `purchase_order`         | string | No       | Purchase order number.                                                                                   |
| `estimated_arrival_date`| string | **Yes**  | Expected arrival date. Format: `DD-MM-YYYY` or `DD/MM/YYYY`.                                             |
| `air_waybill`           | string | No       | Air waybill number.                                                                                      |
| `waybill`               | string | No       | Waybill/tracking number.                                                                                 |
| `shipping`               | object | No       | Shipping address (see Address Object). Uses vendor default if empty.                                     |
| `billing`                | object | No       | Billing address (see Address Object). Uses vendor default if empty.                                      |
| `remark`                 | string | No       | Order remark.                                                                                            |
| `note_on_document`       | string | No       | Note to appear on documents.                                                                             |
| `GR_number`             | string | No       | Goods Receipt number.                                                                                    |
| `add_by`                 | string | No       | Added-by user/name from source system.                                                                   |
| `sap_ref_no`            | string | No       | SAP reference number.                                                                                   |
| `source`                 | string | No       | Source system identifier.                                                                                |
| `external_id`           | string | No       | External system ID for correlation.                                                                      |
| `dockey`                | string | No       | SAP document key.                                                                                        |
| `order_line`            | array  | **Yes**  | Order line items. At least one line required.                                                            |

**Vendor Object (`vendor`)**

| Field       | Type   | Required | Description                                                                        |
| ----------- | ------ | -------- | ---------------------------------------------------------------------------------- |
| `code`      | string | **Yes**  | Vendor code — key for external system. Auto-created if not exists.                |
| `name`      | string | No       | Vendor display name.                                                              |
| `account`   | string | No       | Account code in external system.                                                 |
| `email`     | string | No       | Vendor email.                                                                     |
| `phone`     | string | No       | Vendor phone (e.g., `+(852)91234567`).                                            |

**Address Object (`shipping` / `billing`)**

| Field         | Type   | Required | Description                                      |
| ------------- | ------ | -------- | ------------------------------------------------ |
| `contact_name`| string | No       | Contact person name.                             |
| `phone`       | string | No       | Phone number.                                    |
| `email`       | string | No       | Email address.                                   |
| `address`     | string | No       | Street address.                                  |
| `district`    | string | No       | District/region.                                 |
| `city`        | string | No       | City.                                            |
| `country`     | string | No       | Country.                                         |
| `postal_code` | string | No       | Postal/ZIP code.                                 |
| `address_type`| string | No       | `B` (Business), `R` (Resident), `S` (Shop). Default: `B`. |

**Order Line Object (each item in `order_line`)**

| Field                     | Type          | Required | Description                                                                              |
| ------------------------- | ------------- | -------- | ---------------------------------------------------------------------------------------- |
| `row_no`                  | string        | No       | Line row identifier for error reporting.                                                |
| `sku`                     | string        | **Yes**  | SKU/item code. Must exist in item master for the client.                                |
| `qty`                     | string/number | **Yes**  | Quantity. Must be a positive integer.                                                   |
| `unit`                    | string        | No       | Unit of measure (e.g., `EACH`, `CTN`). Must match SKU's UOM. Default: base UOM.         |
| `exp_date`               | string        | No       | Expiry date. Format: `DD-MM-YYYY`. Required if SKU has expiry control.                  |
| `lot_no`                 | string        | No       | Lot number. Required if SKU has lot control.                                            |
| `serial_no`              | string        | No       | Serial number. Required if SKU has serial control.                                      |
| `batch`                   | string        | No       | Batch number.                                                                           |
| `warehouse_location_code`| string        | No       | Warehouse location code. Must exist if provided.                                        |
| `line_remark`           | string        | No       | Line-level remark.                                                                      |

##### 3.1.2 Example Request

```http
POST /api/sap/inbound/ HTTP/1.1
Host: wms-core.example.com
Authorization: Token abc123
Content-Type: application/json

{
  "batch_code": "BATCH-2025-001",
  "submit": false,
  "orders": [
    {
      "row_no": "1",
      "organization_code": "DEMO",
      "client_code": "DEMOCLIENT",
      "vendor": {
        "code": "VEND001",
        "name": "Vendor Name",
        "email": "vendor@example.com",
        "phone": "+(852)91234567"
      },
      "client_reference": "INB-2025-001",
      "purchase_order": "PO-12345",
      "estimated_arrival_date": "15-03-2025",
      "shipping": {
        "contact_name": "John Doe",
        "phone": "91234567",
        "address": "123 Warehouse St",
        "city": "Hong Kong",
        "country": "HK",
        "address_type": "B"
      },
      "GR_number": "GR-000123",
      "order_line": [
        {
          "row_no": "1",
          "sku": "SKU001",
          "qty": "100",
          "unit": "EACH",
          "exp_date": "31-12-2025",
          "line_remark": ""
        }
      ]
    }
  ]
}
```

##### 3.1.3 Example Success Response (200)

```json
{
  "result": "Success",
  "data": {
    "created": 1,
    "failed": 0,
    "orders": [
      {
        "client_reference": "INB-2025-001",
        "status": "DRAF",
        "dockey": "SAP_DOC_KEY_123"
      }
    ]
  },
  "error": null
}
```

##### 3.1.4 Example Error Response (400)

```json
{
  "result": "Failed",
  "data": null,
  "error": [
    "row 1: Organization[DEMO] does not exist in system",
    "row 1: SKU[SKU999] not found in system"
  ]
}
```

---

### 4. Outbound Order APIs (Shipping)

Outbound order APIs manage **shipping** to customers.

#### 4.1 Create Outbound Order (Batch, SAP-style)

- **Endpoint:** `POST /api/sap/outbound/`
- **Description:** Create one or more outbound (shipping) orders in a single request.

##### 4.1.1 Request Body

| Field        | Type   | Required | Description                                                                                                                                            |
| ------------ | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `batch_code` | string | No       | Upload batch identifier.                                                                                                                              |
| `submit`     | bool   | No       | Auto-submit after creation. Default: `false`. May be overridden by client settings or batch settings.                                                |
| `orders`     | array  | **Yes**  | Array of outbound order objects.                                                                                                                      |

**Order Object**

| Field                     | Type   | Required | Description                                                                                 |
| ------------------------- | ------ | -------- | ------------------------------------------------------------------------------------------- |
| `row_no`                  | string | No       | Row identifier for error messages.                                                          |
| `organization_code`       | string | **Yes**  | Organization code.                                                                          |
| `client_code`             | string | **Yes**  | Client code.                                                                                |
| `customer`                | object | **Yes**  | Customer (relation) information (see below).                                                |
| `courier_code`            | string | No       | Courier code.                                                                               |
| `courier_name`            | string | No       | Courier display name (informational).                                                       |
| `client_reference`        | string | **Yes**  | Unique order reference. Must not duplicate other active orders.                             |
| `sales_order`             | string | No       | Sales order number.                                                                         |
| `estimated_shipping_date` | string | **Yes**  | Expected shipping date. Format: `DD-MM-YYYY`.                                               |
| `air_waybill`            | string | No       | Air waybill.                                                                                |
| `waybill`                | string | No       | Waybill/tracking number.                                                                    |
| `shipping`                | object | No       | Shipping address (see Address Object).                                                      |
| `billing`                 | object | No       | Billing address (see Address Object).                                                       |
| `remark`                  | string | No       | Order remark.                                                                               |
| `note_on_document`        | string | No       | Document note.                                                                              |
| `dockey`                  | string | No       | SAP document key.                                                                           |
| `order_line`              | array  | **Yes**  | Order line items.                                                                           |

**Customer Object (`customer`)**

| Field   | Type   | Required | Description                                                              |
| ------- | ------ | -------- | ------------------------------------------------------------------------ |
| `code`  | string | **Yes**  | Customer code — key for external system. Auto-created if not exists.    |
| `name`  | string | No       | Customer display name.                                                  |
| `account`| string| No       | Account code.                                                            |
| `email` | string | No       | Customer email.                                                         |
| `phone` | string | No       | Customer phone.                                                         |

**Address Object (`shipping` / `billing`)**

Same structure as in **Inbound Orders** (see section 3.1).

**Order Line Object**

| Field                     | Type          | Required | Description                                                |
| ------------------------- | ------------- | -------- | ---------------------------------------------------------- |
| `row_no`                  | string        | No       | Line row identifier.                                       |
| `sku`                     | string        | **Yes**  | SKU code.                                                  |
| `qty`                     | string/number | **Yes**  | Quantity. Positive integer.                                |
| `bin_location`            | string        | No       | Preferred bin for pick (if known).                         |
| `warehouse_location_code` | string        | No       | Warehouse location code. Must exist if provided.           |
| `unit`                    | string        | No       | Unit of measure. Default: base UOM when omitted.           |
| `exp_date`               | string        | No       | Expiry date. Format: `DD-MM-YYYY`.                         |
| `lot_no`                 | string        | No       | Lot number.                                                |
| `serial_no`              | string        | No       | Serial number.                                             |
| `batch`                   | string        | No       | Batch number.                                              |
| `line_remark`           | string        | No       | Line remark.                                               |

##### 4.1.2 Example Request

```http
POST /api/sap/outbound/ HTTP/1.1
Host: wms-core.example.com
Authorization: Token abc123
Content-Type: application/json

{
  "batch_code": "BATCH-2025-002",
  "submit": true,
  "orders": [
    {
      "row_no": "1",
      "organization_code": "DEMO",
      "client_code": "DEMOCLIENT",
      "customer": {
        "code": "CUST001",
        "name": "Customer Name",
        "email": "customer@example.com",
        "phone": "+(852)91234567"
      },
      "courier_code": "DHL",
      "courier_name": "DHL Express",
      "client_reference": "OUT-2025-001",
      "sales_order": "SO-12345",
      "estimated_shipping_date": "20-03-2025",
      "shipping": {
        "contact_name": "Jane Doe",
        "phone": "91234567",
        "address": "456 Delivery Ave",
        "city": "Hong Kong",
        "country": "HK",
        "address_type": "B"
      },
      "order_line": [
        {
          "row_no": "1",
          "sku": "SKU001",
          "qty": "50",
          "unit": "EACH",
          "line_remark": ""
        }
      ]
    }
  ]
}
```

##### 4.1.3 Example Success Response (200)

```json
{
  "result": "Success",
  "data": {
    "created": 1,
    "failed": 0,
    "orders": [
      {
        "client_reference": "OUT-2025-001",
        "status": "SUBM",
        "dockey": "SAP_DOC_KEY_456"
      }
    ]
  },
  "error": null
}
```

##### 4.1.4 Example Error Response (400)

```json
{
  "result": "Failed",
  "data": null,
  "error": [
    "row 1: Organization[XYZ] does not exist in system",
    "row 1: SKU[SKU999] not found in system",
    "row 1: Client Reference[OUT-2025-001] already exist in system"
  ]
}
```

---

### 5. Common Conventions

- **Idempotency**
  - Use unique `client_reference` per order. Duplicate references are rejected while the order is active.
- **Date Format**
  - Unless stated otherwise, dates use `DD-MM-YYYY` or `DD/MM/YYYY`.
- **Standard Envelope**
  - Many endpoints use:

```json
{
  "result": "Success" | "Failed",
  "data": ...,
  "error": null | "message" | [ "messages" ]
}
```

- **HTTP Status Codes**
  - `200` — Success (may still contain validation errors in batch flows).
  - `400` — Invalid request or payload.
  - `401` — Unauthorized (invalid/missing token).
  - `404` — Resource not found.

