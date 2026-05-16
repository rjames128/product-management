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

## Decision 7: Aspire version — current is 13.3, not 13.2

**Status**: CORRECTION — plan.md references "Aspire 13.2 TypeScript AppHost documentation" throughout Decision 4. The current stable release is **Aspire 13.3** (released May 7, 2026).

**Verified versions (as of May 2026)**:

| Component | Version |
|-----------|---------|
| Aspire CLI | 13.3.0 |
| `Aspire.Hosting.JavaScript` NuGet | 13.3.3 |
| `Aspire.Hosting.NodeJs` NuGet | **Deprecated** (renamed in Aspire 13.0 → use `Aspire.Hosting.JavaScript`) |
| .NET SDK minimum | .NET 10 |

**Breaking changes introduced between 13.2 and 13.3 relevant to this project**:

| Change | Impact |
|--------|--------|
| `withEnvironment*` per-kind helpers deprecated | New code should use the unified `withEnvironment(name, value)` — see Decision 11 |
| `ASPIREEXTENSION001` diagnostic renamed to `ASPIREJAVASCRIPT001` | Suppress the `AddViteApp` experimental warning with `ASPIREJAVASCRIPT001` |
| `aspire init` no longer fully wires AppHost on its own | `aspire init` drops a skeleton + `aspire.config.json`; wiring is completed via the `aspireify` agent skill |
| `--log-level` renamed to `--pipeline-log-level` on `aspire publish`/`aspire deploy` | Update any CI scripts |

**Reference**: [Aspire 13.3 release notes](https://aspire.dev/whats-new/aspire-13-3/), [Aspire 13.2 release notes](https://aspire.dev/whats-new/aspire-13-2/)

---

## Decision 8: `addNpmApp` does not exist — correct TypeScript AppHost APIs

**Status**: CORRECTION — Decision 4 in this file and plan.md state `builder.addNpmApp("api", "../api")`. This method **does not exist** in the TypeScript AppHost SDK. The correct APIs are `addJavaScriptApp`, `addNodeApp`, or `addViteApp`.

**Background**: `AddNpmApp` existed in older C# AppHost code under the now-deprecated `Aspire.Hosting.NodeJs` package. The TypeScript AppHost SDK (generated by `aspire add`/`aspire restore`) exposes a different set of methods.

**TypeScript AppHost resource registration methods (Aspire 13.3)**:

| Method | Use case | Notes |
|--------|----------|-------|
| `builder.addJavaScriptApp("name", "../dir")` | npm-script-based Node.js app | Runs `npm run dev`; best for TypeScript APIs (dev script can invoke `tsx`) |
| `builder.addNodeApp("name", "../dir", "src/server.js")` | Run a specific JS file with Node.js | No npm script involved; runs `node <file>` directly |
| `builder.addViteApp("name", "../dir")` | Vite-based frontend | Auto-registers `http` endpoint + `PORT` env var; do NOT call `.withHttpEndpoint()` separately |
| `builder.addNextJsApp("name", "../dir")` | Next.js apps | Experimental — suppress with `ASPIREJAVASCRIPT001` |

**Decision for Taskify**: The Hono API (`api/`) uses TypeScript source files (e.g. `tsx watch src/server.ts` as the dev npm script). Use `builder.addJavaScriptApp("api", "../api")` so Aspire invokes the `dev` script from `api/package.json`. `builder.addViteApp("frontend", "../frontend")` remains correct for the Vite frontend.

**Corrected AppHost wiring**:
```typescript
// apphost.ts (TypeScript AppHost entry point)
import { createBuilder } from './.modules/aspire.js';

const builder = await createBuilder();

const api = await builder
  .addJavaScriptApp("api", "../api")              // runs api/package.json "dev" script
  .withHttpEndpoint({ env: "PORT" })
  .withEnvironment("DATABASE_PATH", process.env.DATABASE_PATH ?? "")
  .withEnvironment("IMAGES_DIR", process.env.IMAGES_DIR ?? "");

const apiEndpoint = await api.getEndpoint("http");

await builder
  .addViteApp("frontend", "../frontend")          // Vite dev server
  .withReference(api)
  .withEnvironment("VITE_API_URL", apiEndpoint);

await builder.build().run();
```

**Reference**: [Set up JavaScript apps in the AppHost — Aspire 13.3](https://aspire.dev/integrations/frameworks/javascript/)

---

## Decision 9: TypeScript AppHost project structure and `aspire.config.json`

**Status**: NEW — the plan describes the AppHost structure as a traditional npm package at `apphost/src/index.ts`, but the Aspire 13.2+ TypeScript AppHost scaffold and CLI tooling expect a specific layout.

**TypeScript AppHost layout (Aspire 13.3 scaffold)**:

```text
<repo-root>/
├── apphost.ts               # Entry point — imports createBuilder from .modules/
├── aspire.config.json       # Replaces legacy .aspire/settings.json + apphost.run.json
├── package.json             # "type": "module"; "dev": "aspire run"
├── tsconfig.json
└── .modules/                # Auto-generated TypeScript SDK — DO NOT COMMIT
    ├── aspire.ts            # Typed API for all installed integrations
    ├── base.ts
    └── transport.ts
```

**`aspire.config.json` format (Aspire 13.2+)**:
```json
{
  "appHost": {
    "path": "apphost.ts",
    "language": "typescript/nodejs"
  },
  "packages": {
    "Aspire.Hosting.JavaScript": "13.3.0"
  },
  "profiles": {
    "https": {
      "applicationUrl": "https://localhost:17000;http://localhost:15000",
      "environmentVariables": {
        "ASPIRE_DASHBOARD_OTLP_ENDPOINT_URL": "https://localhost:21169",
        "ASPIRE_RESOURCE_SERVICE_ENDPOINT_URL": "https://localhost:22260"
      }
    }
  }
}
```

**Key operational notes**:
- The `.modules/` directory is **auto-generated** by `aspire add <integration>` or `aspire restore`. Add it to `.gitignore`.
- Running `aspire run` automatically restores the SDK if `aspire.config.json` package list has changed.
- Running `aspire restore` manually regenerates the TypeScript SDK (useful after upgrading Aspire or switching branches).
- `aspire run` validates TypeScript with `tsc --noEmit` before starting the AppHost.

**Impact on plan.md project structure**: The plan shows `apphost/src/index.ts` as the entry point. If the project uses a nested `apphost/` directory, `aspire.config.json` at the repo root should set `"path": "apphost/src/index.ts"`. The `.modules/` directory lives at the same level as `aspire.config.json` (repo root), not inside `apphost/`.

**Reference**: [TypeScript AppHost project structure — Aspire 13.3](https://aspire.dev/app-host/typescript-apphost/)

---

## Decision 10: OTel Node.js — `@opentelemetry/api` alone is insufficient

**Status**: CORRECTION — Decision 6 states "Aspire auto-instruments process-level telemetry; `@opentelemetry/api` is only needed for manual spans." This is incorrect for Node.js services.

**What Aspire actually does for Node.js**:
Aspire **only injects OpenTelemetry environment variables** into the Node.js process at startup:
- `OTEL_SERVICE_NAME` — set to the resource name (e.g. `"api"`)
- `OTEL_RESOURCE_ATTRIBUTES` — includes `service.instance.id`
- `OTEL_EXPORTER_OTLP_ENDPOINT` — the HTTP OTLP endpoint for the Aspire dashboard (e.g. `http://localhost:4318`)
- `OTEL_BSP_SCHEDULE_DELAY`, `OTEL_BLRP_SCHEDULE_DELAY`, `OTEL_METRIC_EXPORT_INTERVAL` — fast export intervals for dashboard responsiveness

**What Aspire does NOT do for Node.js**: Aspire does NOT auto-instrument Node.js processes. The `@opentelemetry/api` package is only a zero-implementation facade; by itself it emits no telemetry. A full OTel SDK must be initialised in the Node.js app.

**Required npm packages for API (`api/package.json`)**:

| Package | Purpose |
|---------|---------|
| `@opentelemetry/api` | Already in plan — facade for manual spans |
| `@opentelemetry/sdk-node` | **MISSING** — Node.js SDK (TracerProvider, MeterProvider, LogRecordProcessor) |
| `@opentelemetry/exporter-trace-otlp-http` | **MISSING** — sends traces to Aspire dashboard via OTLP/HTTP |
| `@opentelemetry/exporter-metrics-otlp-http` | **MISSING** — sends metrics (optional for v1, but useful) |
| `@opentelemetry/auto-instrumentations-node` | **MISSING** — auto-instruments Node.js `http`/`https`, `sqlite3`, etc. |
| `@opentelemetry/semantic-conventions` | **MISSING** — `ATTR_SERVICE_NAME` and other standard attribute keys |

**Minimal SDK initialisation (must run before any other imports)**:
```typescript
// api/src/telemetry.ts — load via --import flag or first import in server.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),     // reads OTEL_EXPORTER_OTLP_ENDPOINT automatically
  instrumentations: [getNodeAutoInstrumentations({ '@opentelemetry/instrumentation-fs': { enabled: false } })],
});

sdk.start();
```

Aspire injects `OTEL_SERVICE_NAME` and `OTEL_EXPORTER_OTLP_ENDPOINT` automatically, so no hard-coded values are needed in the SDK initialisation.

**Implication for plan.md**: The `api/` runtime dependencies list must be updated to include the OTel SDK packages. The `docs/dependencies.md` per-package justification table must be updated accordingly. Decision 6 rationale ("Aspire provides the SDK at runtime") is only true for .NET services via Service Defaults — not for Node.js.

**Reference**: [Use the Aspire dashboard with Node.js apps](https://aspire.dev/dashboard/standalone-for-nodejs/), [OpenTelemetry environment variables — Aspire 13.3](https://aspire.dev/fundamentals/telemetry/#opentelemetry-environment-variables)

---

## Decision 11: `withEnvironment` unified API (Aspire 13.3)

**Status**: NEW — In Aspire 13.3, the per-kind `withEnvironment*` helpers in the TypeScript AppHost SDK are deprecated in favour of the unified `withEnvironment(name, value)` API.

**Deprecated helpers → unified replacement**:

| Deprecated (≤13.2) | Replacement (13.3+) |
|---------------------|---------------------|
| `withEnvironmentExpression(name, expr)` | `withEnvironment(name, expr)` |
| `withEnvironmentEndpoint(name, endpoint)` | `withEnvironment(name, endpoint)` |
| `withEnvironmentParameter(name, param)` | `withEnvironment(name, param)` |
| `withEnvironmentConnectionString(name, resource)` | `withEnvironment(name, resource)` |

The `value` argument of `withEnvironment` accepts a plain `string`, `ReferenceExpression`, `EndpointReference`, parameter builder, connection string resource builder, or `IExpressionValue`.

**Impact on plan.md**: Any code that calls `withEnvironmentEndpoint` or similar per-kind helpers should be written using `withEnvironment` from the start to avoid deprecation warnings.

**Reference**: [Aspire 13.3 — TypeScript `withEnvironment` migration](https://aspire.dev/whats-new/aspire-13-3/#typescript-withenvironment-migration)

---

## Decision 12: Aspire CLI installation — not an npm package

**Status**: CORRECTION — The quickstart (`quickstart.md`) states `npm install -g @aspire/cli`. There is no `@aspire/cli` npm package. The Aspire CLI is a standalone native tool.

**Correct install methods (Aspire 13.3)**:

```bash
# macOS / Linux (recommended — installs latest stable)
curl -sSL https://aspire.dev/install.sh | bash

# Windows (PowerShell)
# (See https://aspire.dev/get-started/install-cli/ for current PowerShell command)

# Alternative (requires .NET 10 SDK — NativeAOT, instant startup, Aspire 13.3+)
dotnet tool install -g Aspire.Cli
```

**Verify**:
```bash
aspire --version   # 13.3.0+{commitSHA}
```

**Impact**: `quickstart.md` Prerequisites table and install command must be corrected. The curl install method has no .NET dependency (the CLI bundles a minimal .NET runtime). The `dotnet tool` method requires .NET 10 SDK.

**Reference**: [Install the Aspire CLI — Aspire 13.3](https://aspire.dev/get-started/install-cli/)

---

## Decision 13: `aspire run` command syntax

**Status**: CORRECTION — The quickstart states `aspire run --project apphost`. In Aspire 13.2+, the flag is `--apphost` (though `--project` is still accepted as a compatibility alias). The preferred invocation when `aspire.config.json` is present at the repo root is simply `aspire run` (no flag needed).

**Preferred commands (Aspire 13.3)**:
```bash
# Run (foreground — recommended for development)
aspire run

# Run in background (new in 13.2)
aspire start            # or: aspire run --detach

# Stop a running apphost
aspire stop

# Check running apphosts
aspire ps

# Restore TypeScript SDK (after aspire add or Aspire version upgrade)
aspire restore
```

**Reference**: [Aspire CLI reference — Aspire 13.3](https://aspire.dev/reference/cli/commands/aspire-run/)

---

## All NEEDS CLARIFICATION resolved

All original research questions (Decisions 1–6) are resolved. Decisions 7–13 capture critical corrections and new findings arising from Aspire's rapid evolution between the plan's reference version (13.2) and the current stable release (13.3). These findings must be reflected in `plan.md`, `quickstart.md`, and the AppHost implementation before tasks are generated.
