---
description: "Task list for Sewing Fabric Organizer implementation"
---

# Tasks: Sewing Fabric Organizer

**Input**: Design documents from `/specs/001-sewing-fabric-organizer/`

**Prerequisites consulted**:
- [plan.md](plan.md) — implementation phases, project structure, constitution check
- [spec.md](spec.md) — user stories, acceptance scenarios, functional requirements
- [data-model.md](data-model.md) — SQLite schema, TypeScript interfaces, state transitions
- [contracts/api.md](contracts/api.md) — all REST endpoints, request/response shapes
- [research.md](research.md) — Aspire 13.3 corrections (Decisions 7–13), OTel setup (Decision 10)
- [quickstart.md](quickstart.md) — environment variables, test commands

**TDD convention**: Per Constitution Principle VII (NON-NEGOTIABLE) — within every phase, test tasks appear first and MUST fail before the corresponding implementation task begins. The Red-Green-Refactor cycle applies throughout.

## Format: `[ID] [P?] [Story?] Description — file path`

- **[P]**: Can run in parallel with other [P] tasks in the same phase (different files, no incomplete dependencies)
- **[US1]–[US4]**: User story label (User Story phases only — omit in Setup, Foundational, Polish)

---

## Phase 1: Setup

**Purpose**: Repository skeleton and tooling. No feature code — just structure, configs, and verified toolchain.

- [ ] T001 Create repository root files: `aspire.config.json` stub, `.gitignore` (add `.modules/`, `api/data/`, `node_modules/`), `docs/` directory — repo root
- [ ] T002 [P] Initialize `apphost/` package: `package.json` (`"type":"module"`, `"engines": {"node": "^20.19.0 || ^22.13.0 || >=24"}`), `tsconfig.json` (`module: NodeNext, strict: true, target: ES2022`) — `apphost/`
- [ ] T003 [P] Initialize `api/` package: `package.json` (all deps: hono, @hono/node-server, better-sqlite3, @opentelemetry/sdk-node, @opentelemetry/api, @opentelemetry/exporter-trace-otlp-http, @opentelemetry/auto-instrumentations-node, @opentelemetry/semantic-conventions; vitest, @types/better-sqlite3 as devDeps), `tsconfig.json` (`strict: true, target: ES2022`), `vitest.config.ts` (Node environment, 80% coverage threshold) — `api/`
- [ ] T004 [P] Initialize `frontend/` package: `package.json` (deps: vite 6, typescript 6; devDeps: vitest, @vitest/coverage-v8, happy-dom), `tsconfig.json` (`strict: true, target: ES2022`), `vite.config.ts` (VITE_API_URL env var exposure via `define`), `vitest.config.ts` (`environment: 'happy-dom'`, 80% coverage threshold) — `frontend/`
- [ ] T005 Configure `aspire.config.json` with `Aspire.Hosting.JavaScript: 13.3.0` package entry and AppHost path pointing to `apphost/src/index.ts`; run `aspire restore` to generate `.modules/` SDK; confirm `apphost/src/index.ts` can import from `.modules/aspire.js` without TypeScript errors — repo root
- [ ] T006 Create `docs/dependencies.md` stub with section headers for each package (`apphost/`, `api/`, `frontend/`) and a table template for: package name, version, purpose, justification — `docs/dependencies.md`

**Checkpoint**: `npm install` succeeds in all three packages. `tsc --noEmit` passes in all three. `.modules/` exists and is gitignored.

---

## Phase 2: Foundational

**Purpose**: All infrastructure that MUST exist before any user story can be implemented or tested independently.

**⚠️ CRITICAL**: No user story phase work can begin until this phase is complete.

### Data layer (TDD — tests first)

- [ ] T007 Create TypeScript interfaces: `Store`, `Fabric`, `FabricAttribute`, `FabricDetail`, `StoreGroup` — `api/src/models/types.ts`
- [ ] T008 [P] Write `schema.test.ts` (RED): assert all three tables exist after `runMigrations()`, assert all indexes present, assert `runMigrations()` is idempotent (safe to call twice) — `api/src/db/schema.test.ts`
- [ ] T009 [P] Write `client.test.ts` (RED): assert `getDb()` returns a Database instance, assert repeated calls return the same singleton, assert in-memory DB is used when `DATABASE_PATH=':memory:'` — `api/src/db/client.test.ts`
- [ ] T010 [P] Implement `schema.ts` (GREEN): `runMigrations()` executing the full DDL from [data-model.md](data-model.md) (`CREATE TABLE IF NOT EXISTS` for stores, fabrics, fabric_attributes + all three indexes) — `api/src/db/schema.ts`
- [ ] T011 [P] Implement `client.ts` (GREEN): `getDb()` singleton using `DATABASE_PATH` env var; creates `IMAGES_DIR` directory on first call (`fs.mkdirSync(..., { recursive: true })`) — `api/src/db/client.ts`

### API server infrastructure

- [ ] T012 Implement `telemetry.ts`: `NodeSDK` with `OTLPTraceExporter` (reads `OTEL_EXPORTER_OTLP_ENDPOINT` automatically), `getNodeAutoInstrumentations` (disable `instrumentation-fs`); call `sdk.start()` synchronously — **this file must be imported first in `server.ts`** — `api/src/telemetry.ts`
- [ ] T013 Implement `server.ts`: import `./telemetry` as first line; create Hono app; add W3C `traceparent`/`tracestate` middleware (reads incoming header, propagates to response); bind `app.listen` to `process.env.PORT`; call `runMigrations()` on startup — `api/src/server.ts`

### Aspire AppHost wiring

- [ ] T014 Implement `apphost/src/index.ts`: import `createBuilder` from `.modules/aspire.js`; `addJavaScriptApp("api", "../api")` with `.withHttpEndpoint({ env: "PORT" })` and `.withEnvironment()` for `DATABASE_PATH` + `IMAGES_DIR`; `addViteApp("frontend", "../frontend")` with `.withEnvironment("VITE_API_URL", apiEndpoint)`; use unified `withEnvironment(name, value)` (see [research.md](research.md) Decision 11 — per-kind helpers deprecated) — `apphost/src/index.ts`
- [ ] T015 Verify full stack boots: `aspire run` from repo root → both `api` and `frontend` resources appear healthy in Aspire dashboard; OTel endpoint is reachable

### Frontend base infrastructure (TDD — tests first)

- [ ] T016 [P] Write `base-component.test.ts` (RED): assert shadow root is attached in `open` mode after `connectedCallback`, assert `emit<T>()` fires a `CustomEvent` with the correct `detail` payload, assert `attributeChangedCallback` is invoked when a tracked attribute changes — `frontend/src/components/base/base-component.test.ts`
- [ ] T017 [P] Write `api-client.test.ts` (RED): mock `globalThis.fetch`; assert each method (`getStores`, `getFabrics`, `getFabric`, `createFabric`, `updateFabric`, `deleteFabric`) calls the correct URL + method + body; assert 4xx/5xx responses are thrown as typed errors — `frontend/src/services/api-client.test.ts`
- [ ] T018 [P] Implement `base-component.ts` (GREEN): `HTMLElement` base class; `attachShadow({ mode: 'open' })` in constructor; abstract `render()` called from `connectedCallback`; typed `emit<T>(eventName: string, detail: T): void` helper using `new CustomEvent(eventName, { detail, bubbles: true, composed: true })` — `frontend/src/components/base/base-component.ts`
- [ ] T019 [P] Implement `api-client.ts` (GREEN): typed Fetch wrapper using `import.meta.env.VITE_API_URL` as base URL; implement `getStores()`, `getFabrics(storeId?: string)`, `getFabric(id)`, `createFabric(data)`, `updateFabric(id, data)`, `deleteFabric(id)` — see [contracts/api.md](contracts/api.md) for all request/response shapes — `frontend/src/services/api-client.ts`
- [ ] T020 Create `index.html`: single HTML page with `<store-filter>`, `<fabric-form>`, and `<fabric-list>` mount points; import `src/main.ts` as module — `frontend/index.html`
- [ ] T021 Create `main.ts` skeleton: `customElements.define()` call stubs for all three components; `DOMContentLoaded` handler stub — `frontend/src/main.ts`

**Checkpoint**: All Phase 2 tests pass. `aspire run` boots cleanly. `tsc --noEmit` passes in all packages.

---

## Phase 3: User Story 1 — Add a Fabric to the Collection (Priority: P1) 🎯 MVP

**Goal**: A user can open the app, fill in a store name, design name, and quantity, submit the form, and immediately see the new fabric entry grouped under the correct store. Data persists across sessions.

**Independent Test**: Open the app → add "Floral Print", qty 2.5, store "Fabric World" → entry appears under a "Fabric World" store group showing design name and quantity → close and reopen app → entry is still present.

**Acceptance scenarios**: [spec.md](spec.md) User Story 1 scenarios 1–3
**Functional requirements**: FR-001, FR-002, FR-003, FR-006, FR-007, FR-008, FR-011

### Tests for User Story 1 (TDD — write and confirm FAIL before implementation)

- [ ] T022 [P] [US1] Write `stores.test.ts` (RED): assert `GET /api/stores` returns `[]` when no stores exist; returns stores sorted alphabetically by `nameNormalised`; returns correct JSON shape per [contracts/api.md](contracts/api.md) — `api/src/routes/stores.test.ts`
- [ ] T023 [P] [US1] Write `fabrics.test.ts` for POST + GET (RED): assert `POST /api/fabrics` with valid body returns 201 + `FabricDetail`; assert store is created implicitly; assert duplicate store names (case-insensitive) are deduplicated (FR-007); assert missing `designName` returns 400; assert non-positive `quantity` returns 400; assert `GET /api/fabrics` returns fabrics sorted by `stores.name_normalised ASC, fabrics.design_name COLLATE NOCASE ASC` per [data-model.md](data-model.md) — `api/src/routes/fabrics.test.ts`
- [ ] T024 [P] [US1] Write `fabric-form.test.ts` add-mode (RED): assert required fields (storeName, designName, quantity) render; assert form dispatches `fabric-saved` CustomEvent with correct payload on valid submit; assert validation error messages appear for empty designName, zero/negative quantity, empty storeName (FR-006); assert storeName input exposes a `<datalist>` populated from `GET /api/stores` — `frontend/src/components/fabric-form.test.ts`
- [ ] T025 [P] [US1] Write `fabric-list.test.ts` (RED): assert empty-state message renders when passed an empty fabrics array (FR-010); assert fabrics are rendered grouped under store headings; assert each entry shows design name and quantity; assert store headings are in alphabetical order — `frontend/src/components/fabric-list.test.ts`

### Implementation for User Story 1

- [ ] T026 [P] [US1] Implement `stores.ts` (GREEN): `GET /api/stores` — query `SELECT * FROM stores ORDER BY name_normalised ASC`; return `Store[]` per [contracts/api.md](contracts/api.md) — `api/src/routes/stores.ts`
- [ ] T027 [P] [US1] Implement POST + GET in `fabrics.ts` (GREEN): `POST /api/fabrics` — validate fields, `name_normalised = storeName.trim().toLowerCase()`, upsert store on conflict, insert fabric + attributes, return `FabricDetail`; `GET /api/fabrics` — join stores + fabric_attributes, support optional `?storeId=` filter, sort by `stores.name_normalised ASC, fabrics.design_name COLLATE NOCASE ASC` — see [data-model.md](data-model.md) sort order and [contracts/api.md](contracts/api.md) shapes — `api/src/routes/fabrics.ts`
- [ ] T028 [US1] Register `/api/stores` and `/api/fabrics` route handlers in `server.ts` — `api/src/server.ts`
- [ ] T029 [P] [US1] Implement `fabric-form.ts` (GREEN) add mode: shadow DOM form with `storeName` text input + `<datalist>` (populated from `getStores()` on connect), `designName` text input, `quantity` number input, `notes` textarea; client-side validation before emit; emits `fabric-saved` CustomEvent with form data payload — `frontend/src/components/fabric-form.ts`
- [ ] T030 [P] [US1] Implement `fabric-list.ts` (GREEN): receives `FabricDetail[]` via attribute or property; groups by `storeId` and renders store heading + fabric entries in alphabetical order per [data-model.md](data-model.md); renders empty-state prompt (FR-010) when array is empty — `frontend/src/components/fabric-list.ts`
- [ ] T031 [US1] Wire US1 in `main.ts`: register `fabric-form` + `fabric-list` via `customElements.define()`; on `DOMContentLoaded` call `getFabrics()` and pass result to `<fabric-list>`; listen for `fabric-saved` event → call `createFabric(data)` → refresh `<fabric-list>` — `frontend/src/main.ts`

**Checkpoint**: US1 is fully functional and independently testable. `aspire run` → add a fabric → it appears in the grouped view → refresh page → entry persists.

---

## Phase 4: User Story 2 — Browse Fabrics Grouped by Store (Priority: P2)

**Goal**: With multiple fabrics from different stores, the user sees clear store group headings with their fabrics listed beneath. A store filter narrows the view. Clearing the filter restores the full collection.

**Independent Test**: Pre-load two fabrics from different stores → open app → store groups appear with correct fabrics; select a store in the filter → only that store's fabrics visible; clear filter → all fabrics return; empty collection → empty-state message shown.

**Acceptance scenarios**: [spec.md](spec.md) User Story 2 scenarios 1–3
**Functional requirements**: FR-003, FR-009, FR-010, FR-011, FR-015

### Tests for User Story 2 (TDD — write and confirm FAIL before implementation)

- [ ] T032 [US2] Write `store-filter.test.ts` (RED): assert component renders a `<select>` populated from `getStores()` on connect; assert selecting a store emits `store-selected` CustomEvent with `{ storeId }` detail; assert selecting the "All stores" option emits `store-cleared` CustomEvent — `frontend/src/components/store-filter.test.ts`
- [ ] T033 [P] [US2] Add storeId filter test coverage to `fabrics.test.ts` (RED): assert `GET /api/fabrics?storeId={id}` returns only fabrics belonging to that store; assert an unknown storeId returns `[]` — `api/src/routes/fabrics.test.ts`

### Implementation for User Story 2

- [ ] T034 [US2] Implement `store-filter.ts` (GREEN): shadow DOM `<select>` with an "All stores" first option; populates options from `getStores()` on `connectedCallback`; emits `store-selected` or `store-cleared` on change — `frontend/src/components/store-filter.ts`
- [ ] T035 [US2] Wire US2 in `main.ts`: register `store-filter` via `customElements.define()`; handle `store-selected` → call `getFabrics(storeId)` → update `<fabric-list>`; handle `store-cleared` → call `getFabrics()` → update `<fabric-list>` — `frontend/src/main.ts`

**Checkpoint**: US1 + US2 work independently. Filter narrows and restores the collection. Multi-store view is alphabetically ordered.

---

## Phase 5: User Story 3 — Edit an Existing Fabric Entry (Priority: P3)

**Goal**: A user can click edit on any fabric, change its fields (including re-assigning to a different store), and see the updated entry immediately in the grouped view. Cancelling leaves the entry unchanged.

**Independent Test**: Add a fabric with design name "Stripes" → click edit → change name to "Wide Stripes" → save → list shows "Wide Stripes" and the old value is gone; click edit again → cancel → no change.

**Acceptance scenarios**: [spec.md](spec.md) User Story 3 scenarios 1–3
**Functional requirements**: FR-004, FR-006, FR-007, FR-008

### Tests for User Story 3 (TDD — write and confirm FAIL before implementation)

- [ ] T036 [P] [US3] Add `GET /api/fabrics/:id` + `PUT /api/fabrics/:id` tests to `fabrics.test.ts` (RED): assert GET returns `FabricDetail` for valid id; returns 404 for unknown id; assert PUT updates only provided fields; assert changing `storeName` re-assigns to existing (case-insensitive) or new store; assert orphaned old store is deleted; assert `attributes` array on PUT replaces all existing attributes; assert empty `attributes: []` removes all attributes — `api/src/routes/fabrics.test.ts`
- [ ] T037 [US3] Add edit-mode tests to `fabric-form.test.ts` (RED): assert component in edit mode pre-populates fields from a `FabricDetail` input; assert `fabric-updated` CustomEvent is emitted with changed values on save; assert `edit-cancelled` CustomEvent is emitted on cancel; assert cancelling does not trigger any API call — `frontend/src/components/fabric-form.test.ts`

### Implementation for User Story 3

- [ ] T038 [P] [US3] Implement `GET /api/fabrics/:id` in `fabrics.ts` (GREEN): join store + attributes; return 404 if not found per [contracts/api.md](contracts/api.md) — `api/src/routes/fabrics.ts`
- [ ] T039 [US3] Implement `PUT /api/fabrics/:id` in `fabrics.ts` (GREEN): validate any provided fields; if `storeName` provided — normalise, upsert store, update `store_id`, check orphaned old store; if `attributes` provided — DELETE all existing then INSERT new set; update `updated_at`; return full `FabricDetail` — `api/src/routes/fabrics.ts`
- [ ] T040 [P] [US3] Add `getFabric(id)` + `updateFabric(id, data)` to `api-client.ts` — `frontend/src/services/api-client.ts`
- [ ] T041 [US3] Add edit mode to `fabric-form.ts` (GREEN): accept `fabricId` attribute; when set, call `getFabric(id)` in `connectedCallback` to pre-populate fields; emit `fabric-updated` on save; emit `edit-cancelled` on cancel — `frontend/src/components/fabric-form.ts`
- [ ] T042 [P] [US3] Add edit button to `fabric-list.ts`: per-fabric "Edit" button emits `edit-fabric` CustomEvent with `{ fabricId }` detail — `frontend/src/components/fabric-list.ts`
- [ ] T043 [US3] Wire US3 in `main.ts`: handle `edit-fabric` event → set `fabricId` attribute on `<fabric-form>` to open edit mode → handle `fabric-updated` → call `updateFabric(id, data)` → refresh `<fabric-list>`; handle `edit-cancelled` → reset `<fabric-form>` to add mode — `frontend/src/main.ts`

**Checkpoint**: US1 + US2 + US3 all work independently. Editing a fabric — including store re-assignment — reflects immediately in the grouped view.

---

## Phase 6: User Story 4 — Delete a Fabric Entry (Priority: P4)

**Goal**: A user can delete any fabric with a confirmation step. If it was the last fabric for a store, the store group disappears from the view.

**Independent Test**: Add a fabric → delete it → not in view; add one fabric to a store → delete it → that store group disappears from view; trigger delete → cancel confirmation → fabric remains.

**Acceptance scenarios**: [spec.md](spec.md) User Story 4 scenarios 1–3
**Functional requirements**: FR-005, FR-009

### Tests for User Story 4 (TDD — write and confirm FAIL before implementation)

- [ ] T044 [P] [US4] Add `DELETE /api/fabrics/:id` tests to `fabrics.test.ts` (RED): assert returns 204 on success; assert fabric is removed from DB; assert image file is removed from filesystem if `image_path` was set; assert 404 for unknown id; assert orphaned store is deleted after last fabric removed — `api/src/routes/fabrics.test.ts`
- [ ] T045 [US4] Add delete tests to `fabric-list.test.ts` (RED): assert each fabric entry renders a "Delete" button; assert clicking triggers a confirmation dialog (mock `window.confirm`); assert `delete-fabric` CustomEvent emitted with `{ fabricId }` when confirmed; assert no event emitted when cancelled — `frontend/src/components/fabric-list.test.ts`

### Implementation for User Story 4

- [ ] T046 [P] [US4] Implement `DELETE /api/fabrics/:id` in `fabrics.ts` (GREEN): load fabric record; delete image file from `IMAGES_DIR` if `image_path` is set; delete fabric row (cascades to `fabric_attributes`); check remaining fabric count for the store — delete store if count is 0; return 204 — `api/src/routes/fabrics.ts`
- [ ] T047 [P] [US4] Add `deleteFabric(id)` to `api-client.ts` — `frontend/src/services/api-client.ts`
- [ ] T048 [US4] Add delete button to `fabric-list.ts` (GREEN): per-fabric "Delete" button calls `window.confirm('Delete this fabric?')`; emits `delete-fabric` CustomEvent with `{ fabricId }` on confirm; does nothing on cancel — `frontend/src/components/fabric-list.ts`
- [ ] T049 [US4] Wire US4 in `main.ts`: handle `delete-fabric` event → call `deleteFabric(id)` → refresh `<fabric-list>` — `frontend/src/main.ts`

**Checkpoint**: All four user stories work independently. Deleting the last fabric from a store removes the store group from the view.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Optional fabric fields (image attachment — FR-013), `docs/dependencies.md` finalisation, coverage gate enforcement, and integration smoke test.

### Image attachment (FR-013)

- [ ] T050 [P] Add image endpoint tests to `fabrics.test.ts` (RED): assert `GET /api/fabrics/:id/image` returns 200 with binary body when image exists; 404 when fabric not found; 404 when no image; assert `POST /api/fabrics/:id/image` with valid image file returns updated `FabricDetail` with `imagePath` set; assert 415 for disallowed MIME types; assert `DELETE /api/fabrics/:id/image` returns 204 and sets `image_path` to NULL — `api/src/routes/fabrics.test.ts`
- [ ] T051 [P] Implement `GET /api/fabrics/:id/image` in `fabrics.ts`: stream file from `IMAGES_DIR/image_path`; set `Content-Type` from file extension; 404 if fabric missing or `image_path` is NULL — `api/src/routes/fabrics.ts`
- [ ] T052 [P] Implement `POST /api/fabrics/:id/image` in `fabrics.ts`: resolve multipart upload library (see Complexity Tracking in [plan.md](plan.md)); validate MIME type (`image/jpeg`, `image/png`, `image/webp`, `image/gif`) → 415 on mismatch; generate `{uuid}{ext}` filename; delete old image file if `image_path` was already set; write new file to `IMAGES_DIR`; update `image_path` in DB; return updated `FabricDetail` — `api/src/routes/fabrics.ts`
- [ ] T053 [P] Implement `DELETE /api/fabrics/:id/image` in `fabrics.ts`: delete file from `IMAGES_DIR`; set `image_path = NULL`; return 204 — `api/src/routes/fabrics.ts`
- [ ] T054 [P] Add image UI tests to `fabric-form.test.ts` (RED): assert file input renders; assert preview `<img>` shows when `imagePath` is set; assert "Remove image" button emits `image-deleted` event — `frontend/src/components/fabric-form.test.ts`
- [ ] T055 [P] Add image UI to `fabric-form.ts`: file `<input type="file" accept="image/*">`; on change call `uploadImage(fabricId, file)` via api-client; render preview `<img src="/api/fabrics/{id}/image">` when `imagePath` is not null; "Remove image" button calls `deleteImage(fabricId)` — `frontend/src/components/fabric-form.ts`
- [ ] T056 [P] Add `uploadImage(id, file)` + `deleteImage(id)` to `api-client.ts` — `frontend/src/services/api-client.ts`

### Final quality gates

- [ ] T057 Finalise `docs/dependencies.md`: document every runtime dependency for `api/` and `frontend/` with package name, version, purpose, and justification per Constitution Principle VI (including the chosen multipart image upload library from T052) — `docs/dependencies.md`
- [ ] T058 Run API coverage gate: `cd api && npm run test:coverage` — verify ≥ 80 % branch and statement coverage; fix gaps if threshold fails
- [ ] T059 Run frontend coverage gate: `cd frontend && npm run test:coverage` — verify ≥ 80 % branch and statement coverage; fix gaps if threshold fails
- [ ] T060 Integration smoke test: `aspire run` from repo root → manually walk every acceptance scenario in [spec.md](spec.md) User Stories 1–4; verify SC-001–SC-005 from [spec.md](spec.md) Success Criteria
- [ ] T061 Verify Aspire dashboard telemetry: exercise each major endpoint then open the Aspire dashboard → confirm traces visible for `GET /api/fabrics`, `POST /api/fabrics`, `PUT /api/fabrics/:id`, `DELETE /api/fabrics/:id`

---

## Dependencies & Execution Order

### Phase dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 completion — **blocks all user story phases**
- **Phase 3 (US1)**: Depends on Phase 2 completion — first user story, MVP deliverable
- **Phase 4 (US2)**: Depends on Phase 2 + Phase 3 (browse requires the grouped list from US1)
- **Phase 5 (US3)**: Depends on Phase 2 + Phase 3 (edit requires an existing fabric to edit)
- **Phase 6 (US4)**: Depends on Phase 2 + Phase 3 (delete requires an existing fabric to delete)
- **Phase 7 (Polish)**: Depends on all user story phases completing

### User story dependencies

- **US1 (P1)**: Can start after Phase 2 — no dependency on other user stories
- **US2 (P2)**: Depends on US1 (browse requires the list component and `GET /api/fabrics` from US1)
- **US3 (P3)**: Depends on US1 (edit requires an existing fabric)
- **US4 (P4)**: Depends on US1 (delete requires an existing fabric)
- **US3 and US4** can proceed in parallel with each other after US1 completes

### Within each user story

1. Test tasks (RED) — write first, confirm they fail
2. API route implementations (GREEN) — make route tests pass
3. Frontend component implementations (GREEN) — make component tests pass
4. `main.ts` wiring — event handling and data flow (no direct unit tests)
5. Manual checkpoint verification

### Parallel opportunities per phase

**Phase 1**: T002, T003, T004 can run in parallel (different package directories)

**Phase 2**:
- T008, T009 in parallel (different test files, no cross-dependency)
- T010, T011 in parallel after T008/T009 (different implementation files)
- T016, T017 in parallel (different test files — frontend, completely independent of API)
- T018, T019 in parallel after T016/T017 (different implementation files)

**Phase 3**:
- T022, T023 in parallel (stores.test.ts and fabrics.test.ts — different files)
- T024, T025, T026 in parallel after T022/T023 (different implementation files)
- T024/T025/T026 can run in parallel with T027/T028 — API and frontend are independent packages
- T027, T028 in parallel (fabric-form.test.ts and fabric-list.test.ts — different files)
- T029, T030 in parallel after T027/T028 (different implementation files)

**Phase 5**: T038 (GET) in parallel with T040 (api-client) — different files; T042 (fabric-list edit button) in parallel with T041 (fabric-form edit mode) — different component files

**Phase 6**: T046 (API DELETE) in parallel with T047 (api-client), T048 (fabric-list UI) — different files

**Phase 7**: T051–T053 in parallel (different route functions in same file — sequential within the file but logically independent); T055, T056 in parallel (frontend api-client and form component — different files)

---

## Parallel Example: User Story 1

With two implementers (Agent A and Agent B) after Phase 2 is complete:

```
Agent A: T022 → T026 (stores route: test → implement)
Agent B: T023 → T027 (fabrics route: test → implement)
         ↓ [T026 + T027 both done]
Agent A: T024 → T029 (fabric-form: test → implement)
Agent B: T025 → T030 (fabric-list: test → implement)
         ↓ [T029 + T030 both done, T028 done]
         T031 (main.ts US1 wiring — sequential, depends on both components)
```

---

## Implementation Strategy

**MVP scope**: Complete Phase 1 → Phase 2 → Phase 3 (US1) only. This delivers a working app where a user can add fabrics and see them grouped by store, data persists, and all tests pass.

**Incremental delivery order**:
1. Phase 1 + Phase 2 → working skeleton with passing tests
2. Phase 3 (US1) → MVP: add + view grouped fabrics
3. Phase 4 (US2) → store filter added
4. Phase 5 (US3) + Phase 6 (US4) → edit + delete (can be parallel if two agents)
5. Phase 7 (Polish) → image attachment, coverage gate, final smoke test

**Before starting Phase 7 (multipart upload)**: Resolve the multipart library decision first (see Complexity Tracking in [plan.md](plan.md)). Evaluate `@hono/node-server`'s `parseBody` with Node 22 native `File` vs. `busboy`. Add the chosen library to `api/package.json` and document in `docs/dependencies.md`.
