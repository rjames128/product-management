# Quickstart: Sewing Fabric Organizer

**Branch**: `001-sewing-fabric-organizer`

This guide gets the application running locally in under 5 minutes.

---

## Prerequisites

| Tool | Minimum version | Install |
|------|----------------|---------|
| Node.js | 22 LTS | https://nodejs.org |
| npm | 10+ (bundled with Node 22) | — |
| Aspire CLI | latest | `npm install -g @aspire/cli` |

Verify:
```bash
node --version   # v22.x.x
aspire --version
```

---

## Repository layout

```
apphost/    Aspire TypeScript AppHost (orchestration)
api/        Node.js REST API — Hono + better-sqlite3
frontend/   Vite SPA — vanilla HTML/CSS/TypeScript 6.0
docs/       Project-level docs including dependencies.md
```

---

## First-time setup

```bash
# Install dependencies for all three packages
npm install --prefix apphost
npm install --prefix api
npm install --prefix frontend
```

---

## Run locally

```bash
aspire run --project apphost
```

Aspire starts both services in dependency order:
1. **API** — Node.js server with SQLite database at `api/data/fabric.db`
2. **Frontend** — Vite dev server with hot module replacement

Open the **Aspire dashboard** URL printed in the terminal to see service health, logs, and traces.

Open the **frontend URL** (also printed) to use the application.

### Environment variables (managed by Aspire — do not set manually)

| Variable | Consumed by | Purpose |
|----------|-------------|---------|
| `PORT` | api | HTTP port for the API server |
| `DATABASE_PATH` | api | Absolute path to the SQLite file |
| `IMAGES_DIR` | api | Absolute path to the image storage directory |
| `VITE_API_URL` | frontend | Base URL of the API server |

---

## Run tests

Tests follow the **TDD convention**: test files are co-located with source as `*.test.ts`.

```bash
# API unit + integration tests
cd api && npm test

# API tests with coverage report
cd api && npm run test:coverage

# Frontend component + service tests
cd frontend && npm test

# Frontend tests with coverage report
cd frontend && npm run test:coverage
```

Coverage threshold: **≥ 80 % branch/statement** enforced in both packages. A failing threshold fails the test run.

### TDD workflow

Per constitution Principle VII, the Red-Green-Refactor cycle is mandatory:

1. **Red** — write a failing test in a `*.test.ts` file and confirm it fails: `npm test`
2. **Green** — write the minimum implementation to make it pass: `npm test`
3. **Refactor** — clean up; confirm all tests remain green: `npm test`

---

## Build for production

```bash
aspire deploy
```

Aspire generates deployment artifacts. No manual `npm run build` steps are required — Aspire invokes the correct build commands per service.

---

## Data storage locations

| Data | Location |
|------|---------|
| SQLite database | `api/data/fabric.db` |
| Image files | `api/data/images/` |

Both paths are **gitignored**. To reset all data: stop Aspire, delete `api/data/`, and restart.

---

## Linting and type-checking

```bash
# Type-check all packages
npm run typecheck --prefix api
npm run typecheck --prefix frontend

# Lint all packages
npm run lint --prefix api
npm run lint --prefix frontend
```

Both `tsc --noEmit` (zero errors) and ESLint (zero errors/warnings) must pass before opening a PR.

---

## Adding a new npm dependency

Per constitution Principle VI, every new dependency requires a justification entry in `docs/dependencies.md` before it is added to `package.json`. Open a PR with both changes together.
