# Implementation Plan: Sewing Fabric Organizer

**Branch**: `001-sewing-fabric-organizer` | **Date**: 2026-04-17 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-sewing-fabric-organizer/spec.md`

## Summary

A local-first web application that organises a personal sewing fabric collection grouped by store. Built with a Vite + vanilla TypeScript 6.0 frontend and a lightweight Node.js REST API backed by a local SQLite database. Metadata (design name, quantity, store, notes, key-value attributes) is stored in SQLite; fabric images are saved to the local filesystem and never uploaded externally. Aspire's TypeScript AppHost orchestrates both services locally.

> **Spec amendment**: The clarification session recorded "browser storage (v1)" as the persistence mechanism. The plan arguments override this with a local SQLite database via a Node.js API. The spec's Assumptions section has been updated to reflect this decision. The future backend service enhancement remains valid and is now a natural evolution path.

## Technical Context

**Language/Version**: TypeScript 6.0 — all source (frontend, API, AppHost)
**Primary Dependencies**: Vite 6 (frontend bundler/dev-server), Hono v4 (Node.js HTTP API), better-sqlite3 (synchronous SQLite for Node.js), `@opentelemetry/sdk-node` + `@opentelemetry/exporter-trace-otlp-http` + `@opentelemetry/auto-instrumentations-node` + `@opentelemetry/api` (API telemetry pipeline — Principle IV), Vitest (testing); see [research.md](research.md) Decisions 1–2 and 10 for per-dependency rationale
**Storage**: SQLite file via better-sqlite3 (metadata); local filesystem under `api/data/images/` (image files)
**Testing**: Vitest — `environment: 'happy-dom'` for frontend unit tests; default Node environment for API unit + integration tests; ≥ 80 % branch/statement coverage enforced
**Target Platform**: Local machine — Node.js 22 LTS for API; modern desktop/tablet browser (Chrome 120+, Firefox 120+, Safari 17+) for frontend
**Project Type**: Local web application — Vite SPA frontend + Node.js REST API, orchestrated by Aspire TypeScript AppHost
**Performance Goals**: <100 ms p95 API response for collection queries; instant UI re-render on add/edit/delete (no page reload)
**Constraints**: No external network calls for data or images; images stored on local filesystem only; offline-capable (API runs locally via Aspire); single-user, no auth
**Scale/Scope**: Single user; hundreds of fabric entries; all data local

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Status | Notes |
|-----------|------|--------|-------|
| I. TypeScript-First | All source in TypeScript 6.0; `strict: true`; `target: ES2022`; no JS source files | ✅ PASS | Three `tsconfig.json` files (apphost, api, frontend) all set strict + ES2022 |
| II. Aspire Orchestration | Full stack declared in TypeScript AppHost; `aspire run` / `aspire deploy` only | ✅ PASS | `apphost/src/index.ts` registers frontend (`addViteApp`) + API (`addJavaScriptApp`) — see [research.md](research.md) Decision 8 |
| III. Vanilla Web Standards | No runtime UI framework in browser bundle; Vite is build-tool only | ✅ PASS | Vite permitted as bundler; no React/Vue/Angular runtime |
| IV. Observability by Default | API emits OTel logs/traces; W3C trace context propagated on HTTP endpoints | ✅ PASS | Full OTel SDK on API (`sdk-node` + OTLP HTTP exporter + auto-instrumentations); Hono middleware propagates `traceparent` header — see [research.md](research.md) Decision 10 and Complexity Tracking |
| V. Local-First, Production-Parity | `aspire run` boots everything; no hard-coded credentials | ✅ PASS | SQLite path + image dir injected via Aspire env vars; no secrets in source |
| VI. Simplicity & Minimal Footprint | Each dep justified in `docs/dependencies.md`; no premature abstractions | ✅ PASS | All runtime deps justified in [research.md](research.md); `docs/dependencies.md` created in Phase 0 |
| VII. TDD (NON-NEGOTIABLE) | Tests before implementation; ≥ 80 % coverage; Vitest | ✅ PASS | Vitest in all three packages; happy-dom for frontend; Red step required per PR |
| VIII. Web Component-First UI | All reusable UI as Custom Elements; `BaseComponent` base class; shadow DOM; typed attributes; `CustomEvent` upward communication | ✅ PASS | `fabric-list`, `fabric-form`, `store-filter` are Custom Elements extending `BaseComponent`; `base-component.ts` is the project shared base class |

**Post-Phase-1 re-check**: All gates remain PASS. OTel wiring is the only non-trivial complexity item (logged below).

## Project Structure

### Documentation (this feature)

```text
specs/001-sewing-fabric-organizer/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── api.md           # REST API contract (Phase 1 output)
└── tasks.md             # Phase 2 output (NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
# Repository root
aspire.config.json                # Aspire 13.3 AppHost config — registers apphost/src/index.ts
.modules/                         # Auto-generated Aspire TypeScript SDK — add to .gitignore
docs/
└── dependencies.md               # Root-level per-package dependency justification (Principle VI)

apphost/                          # Aspire TypeScript AppHost
├── src/
│   └── index.ts                  # createBuilder(), addJavaScriptApp("api"), addViteApp("frontend")
├── package.json
└── tsconfig.json

api/                              # Node.js REST API (TypeScript 6.0 + Hono + better-sqlite3)
├── src/
│   ├── db/
│   │   ├── schema.ts             # SQLite CREATE TABLE statements + migration runner
│   │   ├── schema.test.ts
│   │   ├── client.ts             # better-sqlite3 database singleton
│   │   └── client.test.ts
│   ├── routes/
│   │   ├── stores.ts             # GET /api/stores (stores created/deleted implicitly via fabrics)
│   │   ├── stores.test.ts
│   │   ├── fabrics.ts            # CRUD /api/fabrics, image upload/serve/delete
│   │   └── fabrics.test.ts
│   ├── models/
│   │   └── types.ts              # TypeScript interfaces: Store, Fabric, FabricAttribute, FabricDetail, StoreGroup
│   ├── telemetry.ts              # OTel NodeSDK init — must be imported before all other modules
│   └── server.ts                 # Hono app entry point, route registration, PORT binding
├── data/                         # Gitignored — SQLite file + image files live here
│   └── images/                   # Fabric images stored as {uuid}{ext}
├── package.json
└── tsconfig.json

frontend/                         # Vite + vanilla HTML/CSS/TypeScript 6.0
├── src/
│   ├── components/
│   │   ├── base/
│   │   │   ├── base-component.ts        # BaseComponent — shadow DOM, lifecycle, typed emit<T>()
│   │   │   └── base-component.test.ts
│   │   ├── fabric-list.ts               # <fabric-list> — grouped fabric display
│   │   ├── fabric-list.test.ts
│   │   ├── fabric-form.ts               # <fabric-form> — add/edit form, image upload
│   │   ├── fabric-form.test.ts
│   │   ├── store-filter.ts              # <store-filter> — store filter control
│   │   └── store-filter.test.ts
│   ├── services/
│   │   ├── api-client.ts                # Typed Fetch wrapper for all API calls
│   │   └── api-client.test.ts
│   └── main.ts                          # App entry point, customElements.define() registrations
├── index.html                           # Single HTML file; Web Components mounted here
├── package.json
├── vite.config.ts
└── tsconfig.json
```

**Structure Decision**: Three sibling packages (`apphost/`, `api/`, `frontend/`) at repository root, coordinated by `aspire.config.json` at the root. `aspire.config.json` points to `apphost/src/index.ts` as the AppHost entry. The `.modules/` directory (auto-generated Aspire TypeScript SDK) lives at the root and is gitignored. Tests are co-located with source files as `*.test.ts` in each package — see [quickstart.md](quickstart.md).

## Complexity Tracking

| Item | Why Needed | Simpler Alternative Rejected Because |
|------|-----------|--------------------------------------|
| OTel middleware on Hono (Principle IV) | Constitution requires all HTTP endpoints to propagate W3C trace context; Aspire's built-in OTel integration hooks into this | Plain `console.log` instrumentation prohibited by Principle IV; skipping trace propagation violates a non-negotiable gate |
| Three sibling packages (apphost + api + frontend) | Aspire requires an AppHost package; API and frontend are separate runtimes (Node.js vs browser) requiring separate `tsconfig.json`, `package.json`, and test environments | A single package cannot serve both browser and Node.js targets without complex dual-build configuration, which would violate Principle VI (no unnecessary complexity) |
| Multipart image upload handler | `POST /api/fabrics/:id/image` accepts `multipart/form-data`; Hono has no built-in multipart parser | Plain JSON cannot carry binary image data; a multipart library (e.g. Hono's `parseBody` with Node 22 `File` + `@hono/node-server`, or `busboy`) must be evaluated and justified in `docs/dependencies.md` before Phase 2 implementation begins |

---

## Core Implementation Phases

TDD is mandatory throughout — Red-Green-Refactor per [quickstart.md](quickstart.md). All phases are sequential; later phases depend on earlier ones compiling and passing tests.

### Phase 0 — Project Scaffolding

**Goal**: Repository skeleton, tooling, and dependency installation verified. No feature code yet.

**Outputs**:
- `aspire.config.json` at repo root
- `apphost/`, `api/`, `frontend/` each with `package.json`, `tsconfig.json`, `vitest.config.ts`
- `.modules/` added to `.gitignore`
- `docs/dependencies.md` stub (per Principle VI; entries added as each dep is introduced)
- `api/data/` and `api/data/images/` added to `.gitignore`

**Key references**:
- [research.md](research.md) Decision 9 — `aspire.config.json` format, `.modules/` generation, `aspire restore`
- [research.md](research.md) Decision 12 — Aspire CLI install (`curl -sSL https://aspire.dev/install.sh | bash`; not npm)
- [research.md](research.md) Decision 13 — `aspire run` (no flag needed when `aspire.config.json` is at root)
- [research.md](research.md) Decision 10 — full OTel SDK package list to add to `api/package.json`
- [quickstart.md](quickstart.md) — expected layout, first-time setup commands, Node.js version requirement

**Notes**:
- Node.js ≥ 20.19.0 or ≥ 22.13.0 required for the TypeScript AppHost (not just the API).
- Run `aspire restore` after scaffold to generate `.modules/` before any AppHost TypeScript is written.

---

### Phase 1 — Data Layer

**Goal**: SQLite schema, migration runner, and database singleton fully tested before any routes exist.

**Outputs** (TDD — write tests first):
- `api/src/models/types.ts`
- `api/src/db/schema.ts` + `schema.test.ts`
- `api/src/db/client.ts` + `client.test.ts`

**Key references**:
- [data-model.md](data-model.md) — complete SQLite DDL, indexes, TypeScript interfaces (`Store`, `Fabric`, `FabricAttribute`, `FabricDetail`, `StoreGroup`)
- [data-model.md](data-model.md) "State Transitions" — store implicit create/delete lifecycle
- [data-model.md](data-model.md) "Image file conventions" — `image_path` format (`images/{uuid}{ext}`)
- [research.md](research.md) Decision 1 — better-sqlite3 rationale

**Notes**:
- Use `process.env.DATABASE_PATH ?? ':memory:'` in tests to avoid touching the real filesystem.
- Create `IMAGES_DIR` directory at process start if it does not exist (`fs.mkdirSync(..., { recursive: true })`).
- Run `schema.ts` migrations at process start in `server.ts`, not lazily on first request.

---

### Phase 2 — REST API Routes

**Goal**: All endpoints implemented and tested. Image upload, serving, and deletion included.

**Outputs** (TDD — write tests first):
- `api/src/routes/stores.ts` + `stores.test.ts`
- `api/src/routes/fabrics.ts` + `fabrics.test.ts`
- `api/src/server.ts`

**Key references**:
- [contracts/api.md](contracts/api.md) — every endpoint, request/response shape, HTTP status codes, error body format `{ "error": "..." }`
- [data-model.md](data-model.md) "Image file conventions" — stored path format, UUID naming
- [data-model.md](data-model.md) "State Transitions" — store implicit create on fabric add; store cleanup on last fabric delete
- [data-model.md](data-model.md) "Sort order" — `stores.name_normalised ASC`, `fabrics.design_name COLLATE NOCASE ASC`
- [spec.md](spec.md) FR-007 — case-insensitive store name deduplication (`name_normalised = name.trim().toLowerCase()`)
- [research.md](research.md) Decision 2 — Hono rationale and version

**Notes**:
- **Multipart parsing (unresolved — see Complexity Tracking)**: Evaluate and select a library for `POST /api/fabrics/:id/image` before writing this route. Options: Hono's `parseBody` with Node 22 native `File` support via `@hono/node-server`, or `busboy`. Add the chosen library to `api/package.json` and `docs/dependencies.md`.
- MIME type validation for images: accept `image/jpeg`, `image/png`, `image/webp`, `image/gif`; return 415 for anything else.
- No `POST /api/stores` endpoint — stores are created implicitly when a fabric is added with a new store name.
- Store cleanup: after deleting a fabric, check `SELECT COUNT(*) FROM fabrics WHERE store_id = ?`; delete the store if count is 0.

---

### Phase 3 — Aspire AppHost Wiring

**Goal**: Both services declared and bootable via `aspire run`. Aspire dashboard shows both services healthy.

**Outputs**:
- `apphost/src/index.ts` — full AppHost wiring
- `aspire.config.json` updated with `Aspire.Hosting.JavaScript` package version and launch profiles

**Key references**:
- [research.md](research.md) Decision 8 — `addJavaScriptApp("api", "../api")` (NOT `addNpmApp`); `addViteApp("frontend", "../frontend")`
- [research.md](research.md) Decision 9 — `createBuilder()` import from `.modules/aspire.js`; `aspire.config.json` format
- [research.md](research.md) Decision 11 — unified `withEnvironment(name, value)` (not deprecated per-kind helpers)
- [quickstart.md](quickstart.md) "Environment variables" table — `PORT`, `DATABASE_PATH`, `IMAGES_DIR`, `VITE_API_URL`

**Notes**:
- `addViteApp` auto-registers an `http` endpoint and injects `PORT`. Do **not** call `.withHttpEndpoint()` separately on the frontend resource.
- `DATABASE_PATH` and `IMAGES_DIR` must be injected as absolute paths (use `path.join(__dirname, ...)` or `process.cwd()` in the AppHost).
- Verify by running `aspire run` and confirming the Aspire dashboard URL appears and both services are shown as running.

---

### Phase 4 — OTel Telemetry (API)

**Goal**: Node.js OTel SDK initialised; all API request traces visible in the Aspire dashboard; W3C `traceparent` propagated on every HTTP response.

**Outputs** (TDD):
- `api/src/telemetry.ts`
- Hono middleware in `server.ts` for `traceparent`/`tracestate` header propagation
- Middleware test in `fabrics.test.ts` or `server.test.ts` asserting `traceparent` present on responses

**Key references**:
- [research.md](research.md) Decision 10 — required OTel packages, why `@opentelemetry/api` alone is insufficient, minimal `NodeSDK` init pattern
- [research.md](research.md) Decision 6 — Hono middleware for W3C trace context propagation

**Notes**:
- `telemetry.ts` MUST be the first import in `server.ts` (or loaded via `--import` in the `dev` npm script). The `NodeSDK.start()` call must complete before Hono begins listening.
- Aspire injects `OTEL_SERVICE_NAME` and `OTEL_EXPORTER_OTLP_ENDPOINT` automatically — do not hard-code them.
- This phase can be done immediately after Phase 2 (the server exists) or after Phase 3 (Aspire is running and the dashboard is available to receive traces).

---

### Phase 5 — Frontend Foundation

**Goal**: `BaseComponent` base class, API client service, and `<store-filter>` component functional. Application shell renders in the browser.

**Outputs** (TDD with `happy-dom`):
- `frontend/src/components/base/base-component.ts` + `base-component.test.ts`
- `frontend/src/services/api-client.ts` + `api-client.test.ts`
- `frontend/src/components/store-filter.ts` + `store-filter.test.ts`
- `frontend/src/main.ts`, `frontend/index.html`

**Key references**:
- [contracts/api.md](contracts/api.md) — response shapes consumed by `api-client.ts`; `GET /api/stores` and `GET /api/fabrics?storeId=`
- [data-model.md](data-model.md) TypeScript interfaces — `Store`, `FabricDetail`, `StoreGroup`
- [spec.md](spec.md) FR-002 — store name autocomplete/selection when adding/editing a fabric
- [spec.md](spec.md) FR-015 — filter control narrows grouped view to one store; clearing restores full view
- [research.md](research.md) Decision 3 — vanilla Web Component rationale; `BaseComponent` pattern

**Notes**:
- **BaseComponent API is not formally specified** in the design artifacts. The implementer must decide: shadow DOM mode (`open`), attribute observation pattern (`observedAttributes` + `attributeChangedCallback`), and the signature for the typed `emit<T>(eventName: string, detail: T)` helper. These choices must be consistent across all three components and noted in a JSDoc comment block in `base-component.ts`.
- `VITE_API_URL` is injected by Aspire at build/dev time; use `import.meta.env.VITE_API_URL` in `api-client.ts`.
- `<store-filter>` dispatches a `CustomEvent` (e.g. `store-selected`) that `main.ts` listens to, triggering a filtered `GET /api/fabrics?storeId=` call.

---

### Phase 6 — Frontend Fabric Components

**Goal**: `<fabric-list>` and `<fabric-form>` components complete; the full user journey is functional end-to-end.

**Outputs** (TDD with `happy-dom`):
- `frontend/src/components/fabric-list.ts` + `fabric-list.test.ts`
- `frontend/src/components/fabric-form.ts` + `fabric-form.test.ts`

**Key references**:
- [spec.md](spec.md) User Stories 1–4 and their acceptance scenarios — the primary correctness checklist for this phase
- [spec.md](spec.md) "Edge Cases" — empty design name, zero/negative quantity, case-insensitive store matching
- [spec.md](spec.md) Success Criteria SC-001–SC-005 — measurable outcomes to verify manually
- [spec.md](spec.md) FR-001–FR-015 — full functional requirement list; FR-005, FR-006, FR-009, FR-010, FR-011, FR-013, FR-014 are especially relevant here
- [contracts/api.md](contracts/api.md) — `POST /api/fabrics`, `PUT /api/fabrics/:id`, `DELETE /api/fabrics/:id`, `POST /api/fabrics/:id/image`, `DELETE /api/fabrics/:id/image` request/response shapes
- [data-model.md](data-model.md) "Sort order" — store groups alphabetical; fabrics within groups alphabetical by `design_name`

**Notes**:
- Deletion confirmation (FR-005, User Story 4 scenario 3): use a `confirm()` dialog or an inline confirmation state in the component — not a silent delete.
- Attribute key-value pairs (FR-014): `<fabric-form>` needs a dynamic "add row / remove row" UI for attributes. Each row validates non-empty `key` and `value` before save. On PUT, the full attribute array replaces the existing set.
- Image preview: show `<img src="/api/fabrics/{id}/image">` when `imagePath` is not null. On image delete, clear the preview immediately.
- Empty state (FR-010): `<fabric-list>` renders an empty-state message when the fabrics array is empty.

---

### Phase 7 — Coverage Gate + Integration Smoke Test

**Goal**: ≥ 80 % branch/statement coverage in both packages; all spec acceptance scenarios pass manually against the running stack.

**Actions**:
1. `cd api && npm run test:coverage` — verify threshold passes (fail build if not)
2. `cd frontend && npm run test:coverage` — verify threshold passes
3. `aspire run` — start the full stack
4. Walk through each acceptance scenario in [spec.md](spec.md) User Stories 1–4
5. Verify Aspire dashboard shows traces from the API for each endpoint exercised

**Key references**:
- [spec.md](spec.md) User Stories 1–4 — acceptance scenarios for manual verification
- [spec.md](spec.md) Success Criteria SC-001–SC-005 — measurable outcomes
- [quickstart.md](quickstart.md) "Run tests" — test commands and coverage flags
- [checklists/requirements.md](checklists/requirements.md) — scope boundary reference
