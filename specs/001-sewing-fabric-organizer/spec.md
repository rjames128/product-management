# Feature Specification: Sewing Fabric Organizer

**Feature Branch**: `001-sewing-fabric-organizer`  
**Created**: 2026-04-17  
**Status**: Draft  
**Input**: User description: "Build an application that can help me organize my sewing fabrics. Fabrics will be grouped by the store that they were purchased by. Each fabric should have a the quantity or count, name of the design and the store the fabric was bought from."

## Clarifications

### Session 2026-04-17

- Q: Should quantity store a unit of measure alongside the number? → A: Plain positive number only; no unit field in v1. Unit tracking is a planned future enhancement.
- Q: Should the application support exporting or backing up the fabric collection? → A: No export/import in v1; data lives in browser storage only. A backend service to persist data remotely is a planned future enhancement.
- Q: How should fabrics be ordered within each store group? → A: Alphabetical by design name (A → Z).
- Q: Should each fabric entry support additional optional attributes beyond design name, quantity, and store? → A: Yes — three optional fields: a free-text notes field, an image attachment, and a collection of user-defined key-value pairs.
- Q: Should users be able to search or filter fabrics within the collection? → A: Filter by store name only in v1. Full-text search across design names and notes is a future enhancement.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Add a Fabric to the Collection (Priority: P1)

A user purchases fabric from a store and wants to record it in the organizer. They open the application, select or enter the store name, type in the design name, and enter the quantity. The new fabric appears immediately in the collection grouped under the correct store.

**Why this priority**: This is the foundational action the entire application depends on. Without the ability to add fabrics, no other feature delivers value. It is the minimum viable product on its own.

**Independent Test**: Open the application, add one fabric entry with a store name, design name, and quantity. Verify the entry appears grouped under the correct store. The collection persists after closing and reopening the application.

**Acceptance Scenarios**:

1. **Given** the application is open and the fabric collection is empty, **When** the user adds a fabric with design name "Floral Print", quantity "2.5", and store "Fabric World", **Then** the fabric appears under a "Fabric World" group with the design name and quantity displayed.
2. **Given** an existing store "Fabric World" already has fabrics, **When** the user adds another fabric to the same store, **Then** the new fabric appears in the same "Fabric World" group alongside existing entries.
3. **Given** the user has added several fabrics, **When** the user closes and reopens the application, **Then** all previously added fabrics are still present and correctly grouped.

---

### User Story 2 - Browse Fabrics Grouped by Store (Priority: P2)

A user wants to see all their fabrics organised by the store they came from. The application displays each store as a group heading with its fabrics listed beneath it, including design name and quantity for each entry.

**Why this priority**: The grouped-by-store view is the core organisational goal stated in the feature description. Once fabrics can be added (P1), this view delivers the primary value: a clear, store-organised overview of the collection.

**Independent Test**: With at least two fabrics from different stores pre-loaded, open the application. Verify that fabrics are visually grouped under their respective store names, each showing design name and quantity.

**Acceptance Scenarios**:

1. **Given** the collection contains fabrics from multiple stores, **When** the user views the collection, **Then** fabrics are grouped under their store names, with design name and quantity visible for each fabric.
2. **Given** a store group contains many fabrics, **When** the user views that group, **Then** all fabrics for that store are listed and no fabrics appear under incorrect groups.
3. **Given** the collection is empty, **When** the user views the collection, **Then** a helpful empty-state message is displayed prompting them to add their first fabric.

---

### User Story 3 - Edit an Existing Fabric Entry (Priority: P3)

A user realises they entered the wrong quantity or misspelled the design name. They select the fabric entry, update the relevant fields, and save. The corrected entry is reflected immediately in the grouped view.

**Why this priority**: Editing is important for data accuracy but not required for the core organisational value. The collection is useful with add and browse capabilities alone.

**Independent Test**: Add a fabric entry, then edit its design name and quantity. Verify the updated values appear in the grouped view immediately, with no duplicate or stale entry remaining.

**Acceptance Scenarios**:

1. **Given** a fabric entry exists with design name "Stripes" and quantity "1", **When** the user edits the design name to "Wide Stripes" and saves, **Then** the entry shows "Wide Stripes" and the old value is gone.
2. **Given** a fabric entry exists, **When** the user changes its store to a different store, **Then** the entry moves to the new store group and is removed from the old group.
3. **Given** the user opens an edit form, **When** the user cancels without saving, **Then** no changes are applied to the fabric entry.

---

### User Story 4 - Delete a Fabric Entry (Priority: P4)

A user no longer has a fabric (used it up, sold it, etc.) and wants to remove it from the collection. They delete the entry and it is immediately removed from the grouped view.

**Why this priority**: Deletion keeps the collection accurate over time. Lower priority because a collection with too many entries is still usable; a collection missing key entries (P1–P3) is not.

**Independent Test**: Add a fabric entry, then delete it. Verify it no longer appears in any group. If it was the last fabric for a store, verify that store group is also removed.

**Acceptance Scenarios**:

1. **Given** a fabric entry exists, **When** the user deletes it, **Then** the entry is immediately removed from the collection view.
2. **Given** a store group contains exactly one fabric, **When** the user deletes that fabric, **Then** the store group heading is also removed from the view.
3. **Given** the user initiates a delete action, **When** a confirmation prompt is presented and the user cancels, **Then** the fabric entry is not deleted.

---

### Edge Cases

- What happens when the user attempts to add a fabric with no design name? The system must prevent submission and display a clear validation message.
- What happens when the user enters a quantity of zero or a negative number? The system must reject non-positive quantities and prompt the user to enter a valid amount.
- What happens when two fabrics from different stores share the same design name? Both entries must be stored and displayed independently under their respective stores.
- What happens if the user types a store name with slight variations (e.g., "Fabric World" vs "fabric world")? The system treats them as the same store (case-insensitive matching).
- What happens when the store field is left empty? The system must prevent submission and prompt the user to provide a store name.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Users MUST be able to add a fabric entry specifying: design name (text), quantity (positive decimal number, no unit field), and store name (text). Unit tracking is explicitly out of scope for v1 and designated as a future enhancement.
- **FR-002**: Users MUST be able to select an existing store from a list when adding or editing a fabric, to avoid duplicate store names caused by typos.
- **FR-003**: The application MUST display all fabric entries grouped by store, with design name and quantity visible for each entry within a group.
- **FR-004**: Users MUST be able to edit the design name, quantity, and store of any existing fabric entry.
- **FR-005**: Users MUST be able to delete any fabric entry, with a confirmation step before permanent removal.
- **FR-006**: The application MUST validate all required fields (design name, quantity, store) before saving a new or edited entry; invalid submissions must surface a clear error message.
- **FR-007**: Store name matching MUST be case-insensitive so that "fabric world" and "Fabric World" refer to the same store.
- **FR-008**: The application MUST persist all fabric entries and store names across sessions so the collection is available the next time the application is opened.
- **FR-009**: When the last fabric is removed from a store group, that store group MUST also be removed from the display.
- **FR-010**: The application MUST display an empty-state prompt when the collection contains no fabrics.
- **FR-011**: Within each store group, fabrics MUST be displayed in alphabetical order by design name (A → Z). Store groups themselves MUST also be ordered alphabetically.
- **FR-012**: Each fabric entry MAY include an optional free-text notes field for any informal information (e.g., colour, material, intended project). Notes are not required and may be left blank.
- **FR-013**: Each fabric entry MAY include an optional image attachment (e.g., a photo of the fabric). Only one image per fabric entry is supported in v1. The image MUST be stored locally alongside the other entry data.
- **FR-014**: Each fabric entry MAY include an optional collection of user-defined key-value pairs (e.g., `colour: blue`, `material: cotton`). Both key and value are free-text strings. There is no fixed schema; users define their own attributes. Duplicate keys within a single fabric entry MUST be prevented.
- **FR-015**: The application MUST provide a filter control that narrows the displayed collection to fabrics belonging to a selected store. Selecting no filter (clearing the filter) restores the full grouped view.

### Key Entities

- **Store**: Represents a shop or supplier where fabrics are purchased. Identified by a unique, case-insensitive name. Has zero or more associated fabrics.
- **Fabric**: An individual fabric entry in the collection. Has a design name (text, required), a quantity (positive decimal number, required; no unit of measure stored in v1), and belongs to exactly one store. Optionally has: a notes field (free text), an image attachment (single image, stored locally), and a collection of user-defined key-value pairs (both key and value are free-text strings; keys must be unique within an entry).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can add a new fabric entry — including selecting or entering a store, entering a design name, and entering a quantity — in under 30 seconds.
- **SC-002**: All fabrics in the collection are visible and correctly grouped by store on a single screen without requiring pagination for collections up to 200 fabric entries.
- **SC-003**: Changes to a fabric entry (edit or delete) are reflected in the grouped view immediately, with no page reload required.
- **SC-004**: Fabric data added in one session is fully available in a subsequent session without any manual export or save action by the user.
- **SC-005**: Validation errors for missing or invalid fields are presented within the same form, clearly identifying which field requires correction, before any data is saved.

## Assumptions

- The application is intended for a single user; no authentication, user accounts, or data sharing between users is required.
- Data will be persisted in a local SQLite database (`api/data/fabric.db`) served by a Node.js API running on the local machine via Aspire. The earlier clarification noting "browser storage" has been superseded by this plan decision. A remote backend service for persistence remains a planned future enhancement.
- "Quantity" is a plain positive decimal number (e.g., `2.5`). No unit of measure field is stored in v1; the user is expected to interpret the number contextually. Unit tracking is a designated future enhancement.
- The application targets modern desktop and tablet browsers; full mobile optimisation is a future enhancement.
- Store names are created by the user as freeform text; there is no external store directory or lookup service.
- Fabrics are displayed alphabetically by design name within each store group; store groups are also ordered alphabetically. Manual reordering is out of scope for v1.
- Each fabric entry supports three optional fields: a free-text notes field, a single image attachment stored locally, and a collection of user-defined key-value pairs. None of these fields are required.
- Filtering is limited to selecting a single store in v1. Full-text search across design names, notes, or key-value pairs is a future enhancement.
- The total number of fabric entries a single user is expected to manage is in the hundreds, not tens of thousands; no pagination or lazy-loading is required for this version.
