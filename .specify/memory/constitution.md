<!--
## Sync Impact Report
- **Version change**: 1.1.0 → 1.1.1
- **Modified principles**: I. TypeScript-First — updated pin to TypeScript 6.0
- **Added sections**: VII. Test-Driven Development (NON-NEGOTIABLE)
- **Removed sections**: None
- **Templates requiring updates**:
  - `.specify/templates/plan-template.md` ✅ — TDD gate should be added to Constitution Check; no structural change required
  - `.specify/templates/spec-template.md` ✅ — Compatible; no structural changes required
  - `.specify/templates/tasks-template.md` ✅ — TDD task pattern (write tests first) already supported by template format
  - `.specify/templates/commands/` — Directory not present; skip
- **Deferred TODOs**: None
-->

# Vanilla Web / Aspire Constitution

## Core Principles

### I. TypeScript-First

All source code MUST be authored in TypeScript **6.0** (latest stable release
as of ratification). The pinned version MUST be declared in `package.json` as
an exact version (`"typescript": "6.0.x"`) and updated as a dedicated PR when a
new stable release ships. No `.js` source files are permitted in `src/`; raw
JavaScript is an output artifact only. `tsconfig.json` MUST enable `strict:
true` and set `target` to `ES2022` (or a newer broadly-supported target) to
align with the modern common JavaScript baseline. The `noImplicitAny`,
`strictNullChecks`, and `noUncheckedIndexedAccess` flags MUST be enabled. Type
assertions (`as X`) MUST include an inline comment explaining why the cast is
safe.

**Rationale**: Pinning to 6.0 makes the toolchain reproducible across machines
and CI. Strict TypeScript eliminates whole classes of runtime errors and gives
AI coding agents precise type information to generate correct code. Targeting
ES2022 balances language modernity with browser compatibility without requiring
heavy polyfills.

### II. Aspire Orchestration

The full application stack — frontend static server, API services, containers,
databases — MUST be defined in an Aspire AppHost authored in TypeScript. No
service MUST be started, wired, or configured outside the AppHost. All
environment variables, service references, and connection strings MUST flow
through Aspire's configuration model. Local runs MUST use `aspire run`; CI/CD
deployments MUST use `aspire deploy`.

**Rationale**: A single code-centric topology definition eliminates environment
drift, simplifies onboarding, and ensures local development mirrors production
faithfully.

### III. Vanilla Web Standards

The frontend MUST be implemented using the browser's native Web APIs: the DOM,
Fetch API, Web Components / Custom Elements, and standard CSS. No JavaScript
UI framework (React, Vue, Angular, Svelte, etc.) is permitted as a runtime
dependency. Third-party frontend runtime dependencies require explicit
justification against an available Web API or a W3C/WHATWG specification.
Build tooling (bundlers, transpilers) is allowed; runtime framework overhead
is not.

**Rationale**: Vanilla web standards reduce payload size, eliminate framework
churn risk, and keep the architecture understandable to any developer familiar
with the platform.

### IV. Observability by Default

Every service MUST emit structured OpenTelemetry signals (logs, metrics, traces)
via Aspire's built-in OpenTelemetry integration. `console.log`-only
instrumentation is prohibited in production code. All HTTP endpoints MUST
propagate W3C trace context headers. The Aspire dashboard MUST be the primary
local observability tool; no additional monitoring stack is required in
development.

**Rationale**: Aspire provides zero-setup OpenTelemetry out of the box.
Encoding observability as a non-negotiable principle prevents it from being
deferred and dramatically reduces debugging time in both development and
production.

### V. Local-First, Production-Parity

`aspire run` MUST fully boot the entire application stack — including all
containerised dependencies — on any developer machine without manual
pre-configuration. Secrets MUST be managed via Aspire's local secrets store or
environment overrides; hard-coded credentials are prohibited anywhere in the
repository. The local topology MUST be structurally identical to the production
topology (same service graph, same connection wiring).

**Rationale**: Local-first development with production parity eliminates
"works on my machine" failures and makes CI validation reliable.

### VI. Simplicity and Minimal Footprint

YAGNI (You Aren't Gonna Need It) is the default stance. Features, abstractions,
and dependencies MUST NOT be added speculatively. Each new npm dependency
requires a recorded justification in `docs/dependencies.md`. Utility code MUST
NOT be extracted into shared modules until it is used in three or more distinct
places. Complexity introduced to satisfy a principle (e.g., observability
wiring) MUST be documented in the plan's Complexity Tracking table.

**Rationale**: A small, focused codebase is easier to understand, audit, and
hand off to AI coding agents. Complexity accumulation is the primary source of
long-term maintenance cost.

### VII. Test-Driven Development (NON-NEGOTIABLE)

All new functionality MUST follow the Red-Green-Refactor cycle:

1. **Red** — Write a failing test that captures the required behaviour before
   writing any implementation code. The test MUST be reviewed and approved (or
   self-reviewed in solo work) before proceeding.
2. **Green** — Write the minimum implementation code required to make the test
   pass. No speculative code.
3. **Refactor** — Clean up both implementation and test code without changing
   observable behaviour; all tests MUST remain green.

Tests MUST be co-located with the source they cover (e.g., `src/foo.ts` →
`src/foo.test.ts`) or placed under a mirrored `tests/` tree — choose one
convention per project and document it in the README. Unit tests MUST cover all
public interfaces. Integration tests MUST cover all Aspire service boundaries.
Test coverage MUST NOT drop below 80 % on any PR (measured by the project's
coverage tooling). Skipped tests (`it.skip`, `xit`) MUST carry a linked issue
reference and an expiry date; unresolved skips block release.

**Rationale**: TDD produces a safety net that makes refactoring and AI-assisted
code generation safe. Writing tests first forces explicit thinking about
contracts and edge cases before implementation details, reducing costly
rework.

## Technology Stack

**Language**: TypeScript 6.0 (pinned exact version; update via dedicated PR)
**Transpilation target**: ES2022; adjust upward only when browser support data
justifies it and document the change in this file
**Transpiler**: TypeScript compiler (`tsc`); a bundler (e.g., esbuild, Vite in
library mode) is permitted for the browser bundle
**Orchestration**: Aspire — AppHost authored in TypeScript
**Frontend paradigm**: Vanilla HTML, CSS, and TypeScript/JavaScript; Web
Components for reusable UI elements
**Observability**: OpenTelemetry via Aspire built-ins; `@opentelemetry/api` is
permitted for manual span creation only
**Package manager**: npm or pnpm — choose one per project and document in README
**Linting / formatting**: ESLint (TypeScript-aware rules) + Prettier; enforced
in CI
**Testing**: Vitest (unit + integration); coverage reporter enforcing ≥ 80 %
branch/statement coverage on every PR

No framework runtime dependencies may be added to the frontend bundle without
amending this constitution.

## Development Workflow

**Local development**: `aspire run` — boots all services; Aspire dashboard
available at the Aspire-provided URL
**Build**: `tsc --noEmit` (type-check) + bundler for browser output
**Type-check gate**: All PRs MUST pass `tsc --noEmit` with zero errors before
merge; `// @ts-ignore` and `// @ts-nocheck` suppressions require a linked issue
explaining why
**Linting gate**: All PRs MUST pass ESLint with zero errors or warnings at
`error`/`warn` severity
**Deployment**: `aspire deploy` generates deployment artifacts; no manual
infrastructure changes are permitted
**TDD gate**: Tests MUST be written before implementation; PRs MUST include
evidence of the Red step (failing test) either as a commit or a PR comment
**Coverage gate**: All PRs MUST maintain ≥ 80 % branch/statement coverage;
coverage reports are generated by `vitest --coverage` and uploaded as CI artifacts
**Dependency review**: New dependencies require a PR comment stating justification
and estimated bundle-size impact
**AI coding agents**: Initialise agents with `aspire agent init` to provide full
app-model context before code generation

All gates above are enforced in CI. A PR failing any gate MUST NOT be merged
regardless of reviewer approval.

## Governance

This constitution supersedes all other conventions, ADRs, and tooling defaults
for this project. When a conflict arises between a tooling recommendation and a
principle in this document, the constitution takes precedence.

**Amendment procedure**: Any change to this constitution requires:

1. A pull request modifying this file with a clear rationale in the PR description.
2. At least one peer review from a project maintainer.
3. A migration plan for any in-progress work that the amendment affects.s
4. A version bump following the semantic versioning rules below.

**Versioning policy**:
- MAJOR — backward-incompatible removal or redefinition of a principle
- MINOR — new principle or section added; material expansion of guidance
- PATCH — clarifications, wording, or typo fixes with no semantic change

**Compliance review**: All PR reviews MUST include a constitution check
confirming no principles are violated. Use the plan.md Constitution Check gate
as the checklist source.

**Version**: 1.1.1 | **Ratified**: 2026-04-17 | **Last Amended**: 2026-04-17
