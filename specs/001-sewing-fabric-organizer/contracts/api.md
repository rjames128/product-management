# API Contract: Sewing Fabric Organizer

**Phase**: 1 — Design
**Branch**: `001-sewing-fabric-organizer`
**Base URL**: `http://localhost:{PORT}/api` (PORT injected by Aspire at runtime)
**Content-Type**: `application/json` for all requests and responses unless noted

All error responses use the shape:
```json
{ "error": "<human-readable message>" }
```

---

## Stores

### `GET /api/stores`

Returns all stores sorted alphabetically by normalised name.

**Response `200 OK`**
```json
[
  {
    "id": "uuid",
    "name": "Fabric World",
    "nameNormalised": "fabric world",
    "createdAt": "2026-04-17T10:00:00.000Z"
  }
]
```

Returns `[]` when no stores exist.

---

## Fabrics

### `GET /api/fabrics`

Returns all fabrics with their store and attributes, sorted alphabetically by store name then by design name.

**Query parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `storeId` | `string` (UUID) | No | Filter to fabrics belonging to this store |

**Response `200 OK`**
```json
[
  {
    "id": "uuid",
    "storeId": "uuid",
    "designName": "Floral Print",
    "quantity": 2.5,
    "notes": "Purchased on sale",
    "imagePath": "images/abc-123.jpg",
    "createdAt": "2026-04-17T10:00:00.000Z",
    "updatedAt": "2026-04-17T10:00:00.000Z",
    "store": {
      "id": "uuid",
      "name": "Fabric World",
      "nameNormalised": "fabric world",
      "createdAt": "2026-04-17T10:00:00.000Z"
    },
    "attributes": [
      { "id": "uuid", "fabricId": "uuid", "key": "colour", "value": "blue", "createdAt": "2026-04-17T10:00:00.000Z" }
    ]
  }
]
```

Returns `[]` when no fabrics match.

---

### `GET /api/fabrics/:id`

Returns a single fabric with store and attributes.

**Path parameters**: `id` — fabric UUID

**Response `200 OK`** — same shape as a single element from `GET /api/fabrics`

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```

---

### `POST /api/fabrics`

Creates a new fabric entry. If the store name does not already exist it is created automatically (case-insensitive deduplication applied).

**Request body**
```json
{
  "designName": "Floral Print",
  "quantity": 2.5,
  "storeName": "Fabric World",
  "notes": "Optional free text",
  "attributes": [
    { "key": "colour", "value": "blue" }
  ]
}
```

| Field | Type | Required | Validation |
|-------|------|----------|-----------|
| `designName` | `string` | Yes | Non-empty after trim |
| `quantity` | `number` | Yes | Positive finite number |
| `storeName` | `string` | Yes | Non-empty after trim |
| `notes` | `string` | No | Any string; `null` or omit to leave blank |
| `attributes` | `array` | No | Each entry must have non-empty `key` and `value`; duplicate keys rejected |

**Response `201 Created`** — full `FabricDetail` object (same shape as `GET /api/fabrics/:id`)

**Response `400 Bad Request`** — validation failure
```json
{ "error": "designName is required" }
```

**Response `409 Conflict`** — duplicate attribute key
```json
{ "error": "Duplicate attribute key: colour" }
```

---

### `PUT /api/fabrics/:id`

Updates an existing fabric. All fields are optional; only provided fields are updated. If `storeName` is provided the fabric is re-assigned to that store (creating the store if needed; orphaned store cleaned up if applicable).

**Path parameters**: `id` — fabric UUID

**Request body** (all fields optional)
```json
{
  "designName": "Wide Stripes",
  "quantity": 1,
  "storeName": "Another Store",
  "notes": "Updated note",
  "attributes": [
    { "key": "colour", "value": "red" }
  ]
}
```

When `attributes` is provided, it **replaces** the full set of attributes for this fabric (not a merge). Send an empty array `[]` to remove all attributes.

**Response `200 OK`** — full updated `FabricDetail` object

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```

**Response `400 Bad Request`** — validation failure

---

### `DELETE /api/fabrics/:id`

Deletes a fabric entry and its image file (if any). If this was the last fabric for its store, the store record is also deleted.

**Path parameters**: `id` — fabric UUID

**Response `204 No Content`** — success; no body

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```

---

## Fabric Images

### `GET /api/fabrics/:id/image`

Streams the image file for a fabric. Sets `Content-Type` to the detected MIME type of the stored file.

**Path parameters**: `id` — fabric UUID

**Response `200 OK`** — binary image stream with appropriate `Content-Type`

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```
```json
{ "error": "No image for this fabric" }
```

---

### `POST /api/fabrics/:id/image`

Uploads or replaces the image for a fabric. If an image already exists for this fabric, it is deleted and replaced.

**Content-Type**: `multipart/form-data`

**Form field**: `image` — the image file

**Permitted MIME types**: `image/jpeg`, `image/png`, `image/webp`, `image/gif`

**Response `200 OK`** — full updated `FabricDetail` object (with `imagePath` set)

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```

**Response `415 Unsupported Media Type`**
```json
{ "error": "Unsupported image type. Use JPEG, PNG, WebP, or GIF." }
```

---

### `DELETE /api/fabrics/:id/image`

Removes the image from a fabric. Deletes the file from the filesystem and sets `image_path` to `NULL` in the database.

**Path parameters**: `id` — fabric UUID

**Response `204 No Content`** — success; no body

**Response `404 Not Found`**
```json
{ "error": "Fabric not found" }
```
```json
{ "error": "No image for this fabric" }
```

---

## Error Reference

| HTTP Status | Meaning |
|-------------|---------|
| `200 OK` | Successful read or update |
| `201 Created` | Resource created successfully |
| `204 No Content` | Successful delete |
| `400 Bad Request` | Invalid or missing request fields |
| `404 Not Found` | Resource with given ID does not exist |
| `409 Conflict` | Uniqueness constraint violation (e.g. duplicate attribute key) |
| `415 Unsupported Media Type` | Uploaded file MIME type not permitted |
| `500 Internal Server Error` | Unexpected server error |

---

## OpenTelemetry headers

All responses include W3C trace context headers when a trace is active:

```
traceparent: 00-{traceId}-{spanId}-01
tracestate: (optional vendor extensions)
```

These are injected by Hono middleware and consumed by the Aspire dashboard automatically.
