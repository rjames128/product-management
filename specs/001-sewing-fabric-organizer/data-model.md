# Data Model: Sewing Fabric Organizer

**Phase**: 1 — Design
**Branch**: `001-sewing-fabric-organizer`
**Source**: [spec.md](spec.md) entities + clarifications

---

## Entities

### Store

Represents a shop or supplier where fabrics are purchased.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | `TEXT` (UUID) | PRIMARY KEY | `crypto.randomUUID()` — Node built-in |
| `name` | `TEXT` | NOT NULL | Display name as entered by user |
| `name_normalised` | `TEXT` | NOT NULL, UNIQUE | `name.trim().toLowerCase()` — used for case-insensitive deduplication (FR-007) |
| `created_at` | `TEXT` | NOT NULL | ISO 8601 UTC timestamp |

**Validation rules**:
- `name` MUST be non-empty after trimming whitespace.
- `name_normalised` MUST be unique across all stores (enforced by UNIQUE constraint).
- A store record is created automatically when a fabric references a new store name.
- A store record is removed when its last fabric is deleted (cleanup at DELETE time).

---

### Fabric

An individual fabric entry in the collection. Belongs to exactly one store.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | `TEXT` (UUID) | PRIMARY KEY | `crypto.randomUUID()` |
| `store_id` | `TEXT` (UUID) | NOT NULL, FK → `stores.id` | Cascades on store delete |
| `design_name` | `TEXT` | NOT NULL | Display name of the fabric design |
| `quantity` | `REAL` | NOT NULL, CHECK > 0 | Positive decimal; no unit stored (v1 decision) |
| `notes` | `TEXT` | NULL | Optional free-text field |
| `image_path` | `TEXT` | NULL | Relative path from `api/data/` — e.g., `images/abc-123.jpg`; NULL if no image |
| `created_at` | `TEXT` | NOT NULL | ISO 8601 UTC timestamp |
| `updated_at` | `TEXT` | NOT NULL | ISO 8601 UTC timestamp; updated on every write |

**Validation rules**:
- `design_name` MUST be non-empty after trimming whitespace.
- `quantity` MUST be a positive finite number (> 0, not NaN, not Infinity).
- `store_id` MUST reference an existing `stores.id`.
- `image_path` is NULL until an image is attached; set to NULL again when image is deleted.

**Sort order**: Query results order by `stores.name_normalised ASC`, then `fabrics.design_name COLLATE NOCASE ASC` (FR-011).

---

### FabricAttribute

User-defined key-value pairs attached to a fabric entry (FR-014).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | `TEXT` (UUID) | PRIMARY KEY | `crypto.randomUUID()` |
| `fabric_id` | `TEXT` (UUID) | NOT NULL, FK → `fabrics.id` ON DELETE CASCADE | |
| `key` | `TEXT` | NOT NULL | Free-text attribute name (e.g., `colour`) |
| `value` | `TEXT` | NOT NULL | Free-text attribute value (e.g., `navy blue`) |
| `created_at` | `TEXT` | NOT NULL | ISO 8601 UTC timestamp |

**Constraints**:
- UNIQUE(`fabric_id`, `key`) — duplicate keys within a single fabric are prevented (FR-014).
- Both `key` and `value` MUST be non-empty after trimming.

---

## SQLite Schema

```sql
CREATE TABLE IF NOT EXISTS stores (
  id             TEXT    NOT NULL PRIMARY KEY,
  name           TEXT    NOT NULL,
  name_normalised TEXT   NOT NULL UNIQUE,
  created_at     TEXT    NOT NULL
);

CREATE TABLE IF NOT EXISTS fabrics (
  id          TEXT    NOT NULL PRIMARY KEY,
  store_id    TEXT    NOT NULL REFERENCES stores(id) ON DELETE CASCADE,
  design_name TEXT    NOT NULL,
  quantity    REAL    NOT NULL CHECK(quantity > 0),
  notes       TEXT,
  image_path  TEXT,
  created_at  TEXT    NOT NULL,
  updated_at  TEXT    NOT NULL
);

CREATE TABLE IF NOT EXISTS fabric_attributes (
  id         TEXT NOT NULL PRIMARY KEY,
  fabric_id  TEXT NOT NULL REFERENCES fabrics(id) ON DELETE CASCADE,
  key        TEXT NOT NULL,
  value      TEXT NOT NULL,
  created_at TEXT NOT NULL,
  UNIQUE(fabric_id, key)
);

-- Indexes for sort-order queries
CREATE INDEX IF NOT EXISTS idx_fabrics_store_design
  ON fabrics(store_id, design_name COLLATE NOCASE);

CREATE INDEX IF NOT EXISTS idx_stores_name_normalised
  ON stores(name_normalised);

CREATE INDEX IF NOT EXISTS idx_attrs_fabric
  ON fabric_attributes(fabric_id);
```

---

## Relationships

```
Store (1) ──────< Fabric (N)
                    │
                    └──< FabricAttribute (N)
```

- One store has zero or more fabrics.
- One fabric belongs to exactly one store.
- One fabric has zero or more attributes (key-value pairs).
- Deleting a fabric cascades to its attributes.
- Deleting a store cascades to its fabrics (and transitively to their attributes).

---

## TypeScript Interfaces (`api/src/models/types.ts`)

```typescript
export interface Store {
  id: string;
  name: string;
  nameNormalised: string;
  createdAt: string;
}

export interface Fabric {
  id: string;
  storeId: string;
  designName: string;
  quantity: number;
  notes: string | null;
  imagePath: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface FabricAttribute {
  id: string;
  fabricId: string;
  key: string;
  value: string;
  createdAt: string;
}

// API response shape — fabric with its store and attributes pre-joined
export interface FabricDetail extends Fabric {
  store: Store;
  attributes: FabricAttribute[];
}

// API response shape — store with its fabrics (grouped view)
export interface StoreGroup {
  store: Store;
  fabrics: FabricDetail[];
}
```

---

## State Transitions

### Fabric lifecycle

```
[not present] --add--> [active] --edit--> [active]
                              └---delete--> [not present]
                                             (orphan store cleanup triggered)
```

### Store lifecycle

```
[not present] --fabric-added-to-new-store--> [active]
                                                   └--last-fabric-deleted--> [not present]
```

Store records are created implicitly (when a fabric is added referencing a new store name) and deleted implicitly (when the last fabric for that store is deleted). No direct store create/delete API is exposed to the user.

---

## Image file conventions

- Stored at: `api/data/images/{uuid}{extension}` where extension is preserved from the original uploaded file (e.g., `.jpg`, `.png`, `.webp`).
- `image_path` in SQLite stores the path relative to `api/data/` (e.g., `images/abc-123.jpg`), making the data portable.
- When a fabric is deleted, its image file is also deleted from the filesystem.
- When a fabric's image is replaced, the old image file is deleted before the new one is written.
- Permitted MIME types: `image/jpeg`, `image/png`, `image/webp`, `image/gif`. Rejected otherwise with HTTP 415.
