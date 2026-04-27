# Research: Sewing Fabric Organizer

**Phase**: 0 — Unknowns resolved before Phase 1 design
**Branch**: `001-sewing-fabric-organizer`

All NEEDS CLARIFICATION items from Technical Context are resolved below.

---

## Decision 1: SQLite library for Node.js

**Decision**: `better-sqlite3` + `@types/better-sqlite3`

**Rationale**:
- Synchronous API eliminates async/await complexity for a single-user local service; no concurrency concerns.
- Direct filesystem persistence — no WASM loading delay, no in-memory-only mode risk.
- Mature, well-typed, widely used. Actively maintained.
- Fits Principle VI (minimal footprint): one package, one transitive native binding.

**Alternatives considered**:

| Alternative | Rejected Because |
|-------------|-----------------|
| `sql.js` (SQLite WASM) | WASM startup overhead; in-memory by default (file persistence requires manual serialisation); better fit for browser environments, not Node.js server |
| `@libsql/client` (Turso/libSQL) | Designed for cloud sync; overkill for local-only single-user use; adds remote connectivity surface |
| Bun built-in SQLite | We target Node.js 22 LTS for wider developer compatibility; switching to Bun is a future option if runtime is ever changed |
| `knex` / `drizzle` ORM | Constitution Principle VI — no ORM abstraction needed for a handful of simple tables; raw SQL with TypeScript interfaces is sufficient and simpler |

---

## Decision 2: Node.js HTTP framework

**Decision**: `hono` (Hono v4)

**Rationale**:
- Ultra-minimal (~14 KB), TypeScript-first, zero dependencies of its own.
- First-class Node.js adapter (`@hono/node-server`) while remaining portable (future backend migration path per clarification Q2 answer).
- Middleware model makes OTel trace-context propagation (Principle IV) a single middleware registration.
- Justified under Principle VI: routing complexity of 10+ REST endpoints would require significant boilerplate with plain `node:http`.

**Alternatives considered**:

| Alternative | Rejected Because |
|-------------|-----------------|
| Plain `node:http` | Too verbose for routing; manual URL parsing, method dispatch, and content-type negotiation add ~300 lines of boilerplate with no type safety |
| `express` | Legacy codebase; no native TypeScript; heavier than Hono |
| `fastify` | Heavier than Hono; schema validation overkill for local single-user API |

---

## Decision 3: Image file storage strategy

**Decision**: Save image files to `api/data/images/{uuid}{originalExtension}` on local filesystem; store the relative path (e.g., `images/abc123.jpg`) in the SQLite `fabric.image_path` column. Serve via `GET /api/fabrics/:id/image` — API reads the file and streams it back with the correct `Content-Type`.

**Rationale**:
- Images never leave the local machine (satisfies FR-013 and clarification answer).
- `crypto.randomUUID()` is Node.js built-in — no extra dependency.
- Storing relative path in SQLite (relative to `api/data/`) keeps the data portable if the project directory is moved.
- Serving via API endpoint keeps the frontend decoupled from filesystem paths and allows future transition to remote storage without frontend changes.

**Alternatives considered**:

| Alternative | Rejected Because |
|-------------|-----------------|
| Store images as Base64 blobs in SQLite | Bloats database; poor performance for large images; not idiomatic SQLite usage |
| Store absolute filesystem path and serve as `file://` URLs | `file://` URLs blocked by browser CORS policy; requires browser-side workaround |
| Copy images to `frontend/public/` | Vite rebuild triggered on every image add; images mixed with source assets; no cleanup on delete |

---

## Decision 4: Aspire TypeScript AppHost — registering a Node.js API

**Decision**: Use `builder.addNpmApp("api", "../api")` in the TypeScript AppHost to register the Node.js API service; use `builder.addViteApp("frontend", "../frontend")` for the frontend.

**Rationale**:
- `addNpmApp` is the standard Aspire TypeScript AppHost primitive for Node.js/npm projects that are not Vite apps.
- `addViteApp` is the Aspire-native way to register a Vite frontend dev server.
- Aspire injects environment variables (`DATABASE_PATH`, `IMAGES_DIR`, `PORT`) into the API process automatically via `.withEnvironment()` on the resource.
- The frontend receives `VITE_API_URL` injected by Aspire so no hard-coded localhost ports exist anywhere.

**Reference**: Aspire 13.2 TypeScript AppHost documentation — `addNpmApp` for executables, `addViteApp` for Vite projects.

---

## Decision 5: Vitest configuration for mixed browser/Node environments

**Decision**:
- `frontend/` — Vitest with `environment: 'happy-dom'` (lighter than `jsdom`; covers Custom Elements and DOM APIs used in Web Components)
- `api/` — Vitest with default Node environment
- Coverage: `@vitest/coverage-v8` in both packages; ≥ 80 % branch/statement enforced via `coverageThreshold` in `vitest.config.ts`

**Rationale**:
- `happy-dom` fully supports Custom Elements v1 (needed for Web Component unit tests) and is 2–3× faster than jsdom.
- Keeping separate Vitest configs per package avoids environment conflicts between browser-context and Node-context code.
- `@vitest/coverage-v8` uses Node's built-in V8 coverage — no Babel transform needed; compatible with TypeScript 6.0 via `tsx` or `ts-node`.

**Alternatives considered**:

| Alternative | Rejected Because |
|-------------|-----------------|
| `jsdom` for frontend tests | Heavier than happy-dom; slower; both cover the same Custom Elements API surface |
| Single root Vitest config with multiple environments | Requires Vitest workspace config; adds complexity; keeping per-package configs is simpler and more explicit (Principle VI) |

---

## Decision 6: OpenTelemetry instrumentation on Hono API

**Decision**: Add `@opentelemetry/api` to `api/` for manual span creation. Use Hono middleware to extract and propagate W3C `traceparent` / `tracestate` headers on all HTTP responses. Aspire's built-in OTel integration auto-instruments Node.js process logs and metrics without additional packages.

**Rationale**:
- Constitution Principle IV: all HTTP endpoints MUST propagate W3C trace context.
- Aspire auto-instruments process-level telemetry; `@opentelemetry/api` is only needed for manual spans around SQLite operations.
- `@opentelemetry/api` is the zero-implementation facade — it adds no concrete exporter or overhead unless an SDK is attached (Aspire provides the SDK at runtime).

**Complexity note**: This is logged in the plan's Complexity Tracking table. Hono middleware implementation is ~10 lines.

---

## All NEEDS CLARIFICATION resolved

No unknowns remain. All decisions above are reflected in the Technical Context section of `plan.md`.
