# Implementation Plan: Sewing Fabric Organizer

**Branch**: `001-sewing-fabric-organizer` | **Date**: 2026-04-17 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-sewing-fabric-organizer/spec.md`

## Summary

A local-first web application that organises a personal sewing fabric collection grouped by store. Built with a Vite + vanilla TypeScript 6.0 frontend and a lightweight Node.js REST API backed by a local SQLite database. Metadata (design name, quantity, store, notes, key-value attributes) is stored in SQLite; fabric images are saved to the local filesystem and never uploaded externally. Aspire's TypeScript AppHost orchestrates both services locally.

> **Spec amendment**: The clarification session recorded "browser storage (v1)" as the persistence mechanism. The plan arguments override this with a local SQLite database via a Node.js API. The spec's Assumptions section has been updated to reflect this decision. The future backend service enhancement remains valid and is now a natural evolution path.

## Technical Context

**Language/Version**: TypeScript 6.0 — all source (frontend, API, AppHost)
**Primary Dependencies**: Vite 6 (frontend bundler/dev-server), Hono (Node.js HTTP API, ~14 KB), better-sqlite3 (synchronous SQLite for Node.js), @opentelemetry/api (manual span creation — Principle IV), Vitest (testing)
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
| II. Aspire Orchestration | Full stack declared in TypeScript AppHost; `aspire run` / `aspire deploy` only | ✅ PASS | `apphost/src/index.ts` registers frontend (Vite) + API (npm app) |
| III. Vanilla Web Standards | No runtime UI framework in browser bundle; Vite is build-tool only | ✅ PASS | Vite permitted as bundler; no React/Vue/Angular runtime |
| IV. Observability by Default | API emits OTel logs/traces; W3C trace context propagated on HTTP endpoints | ✅ PASS | `@opentelemetry/api` on API; Hono middleware propagates `traceparent` header — see Complexity Tracking |
| V. Local-First, Production-Parity | `aspire run` boots everything; no hard-coded credentials | ✅ PASS | SQLite path + image dir injected via Aspire env vars; no secrets in source |
| VI. Simplicity & Minimal Footprint | Each dep justified in `docs/dependencies.md`; no premature abstractions | ✅ PASS | 5 runtime deps justified in research.md; `docs/dependencies.md` created at project setup |
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
apphost/                          # Aspire TypeScript AppHost
├── src/
│   └── index.ts                  # Declares frontend + API resources, wires dependencies
├── package.json
└── tsconfig.json

api/                              # Node.js REST API (TypeScript 6.0 + Hono + better-sqlite3)
├── src/
│   ├── db/
│   │   ├── schema.ts             # SQLite CREATE TABLE statements + migration runner
│   │   └── client.ts             # better-sqlite3 database singleton
│   ├── routes/
│   │   ├── stores.ts             # GET /api/stores, POST /api/stores
│   │   └── fabrics.ts            # CRUD /api/fabrics, image upload/serve/delete
│   ├── models/
│   │   └── types.ts              # Shared TypeScript interfaces (Store, Fabric, FabricAttribute)
│   └── server.ts                 # Hono app entry point, OTel init, route registration
├── data/                         # Gitignored — SQLite file + image files live here
│   └── images/                   # Fabric images stored as {uuid}{ext}
├── src/
│   ├── db/
│   │   ├── schema.test.ts
│   │   └── client.test.ts
│   └── routes/
│       ├── stores.test.ts
│       └── fabrics.test.ts
├── docs/
│   └── dependencies.md           # Per constitution Principle VI
├── package.json
└── tsconfig.json

frontend/                         # Vite + vanilla HTML/CSS/TypeScript 6.0
├── src/
│   ├── components/
│   │   ├── base/
│   │   │   └── base-component.ts # Shared BaseComponent — shadow root, lifecycle, CustomEvent helpers
│   │   ├── fabric-list.ts        # Web Component — grouped fabric display (<fabric-list>)
│   │   ├── fabric-form.ts        # Web Component — add/edit fabric form (<fabric-form>)
│   │   └── store-filter.ts       # Web Component — store filter control (<store-filter>)
│   ├── services/
│   │   └── api-client.ts         # Typed Fetch wrapper for all API calls
│   └── main.ts                   # App entry point, component registration
├── src/
│   ├── components/
│   │   ├── fabric-list.test.ts
│   │   ├── fabric-form.test.ts
│   │   └── store-filter.test.ts
│   └── services/
│       └── api-client.test.ts
├── index.html                    # Single HTML file; Web Components mounted here
├── package.json
├── vite.config.ts
└── tsconfig.json

docs/
└── dependencies.md               # Root-level; per-package deps documented here
```

**Structure Decision**: Web application pattern — Vite SPA frontend + Node.js API + Aspire AppHost as three sibling packages at repository root. Tests are co-located with source files as `*.test.ts` in each package (chosen convention, documented in quickstart.md).

## Complexity Tracking

| Item | Why Needed | Simpler Alternative Rejected Because |
|------|-----------|--------------------------------------|
| OTel middleware on Hono (Principle IV) | Constitution requires all HTTP endpoints to propagate W3C trace context; Aspire's built-in OTel integration hooks into this | Plain `console.log` instrumentation prohibited by Principle IV; skipping trace propagation violates a non-negotiable gate |
| Three sibling packages (apphost + api + frontend) | Aspire requires an AppHost package; API and frontend are separate runtimes (Node.js vs browser) requiring separate `tsconfig.json`, `package.json`, and test environments | A single package cannot serve both browser and Node.js targets without complex dual-build configuration, which would violate Principle VI (no unnecessary complexity) |
