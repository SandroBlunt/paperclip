# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read First

`AGENTS.md` is the authoritative contributor guide and applies to AI contributors. Before non-trivial changes, also consult (in order): `doc/GOAL.md`, `doc/PRODUCT.md`, `doc/SPEC-implementation.md` (V1 build contract — controls when in conflict with `doc/SPEC.md`), `doc/DEVELOPING.md`, `doc/DATABASE.md`.

## Repo Layout (pnpm workspace)

- `server/` (`@paperclipai/server`) — Express REST API + orchestration services. Routes in `server/src/routes/`, services in `server/src/services/`. Entry: `server/src/index.ts`.
- `ui/` (`@paperclipai/ui`) — React + Vite board UI. In dev, served by the API server in middleware mode (same origin).
- `packages/db/` (`@paperclipai/db`) — Drizzle schema, migrations, DB clients. **`drizzle.config.ts` reads compiled schema from `dist/schema/*.js`** — `pnpm db:generate` compiles `packages/db` first.
- `packages/shared/` — shared types, constants, validators, API path constants.
- `packages/adapters/*` — agent adapter implementations (`claude-local`, `codex-local`, `cursor-local`, `gemini-local`, `acpx-local`, `opencode-local`, `pi-local`, `openclaw-gateway`).
- `packages/adapter-utils/` — shared adapter utilities.
- `packages/plugins/` — plugin SDK + examples. `sandbox-providers/**` and `examples/plugin-orchestration-smoke-example` are intentionally excluded from the workspace.
- `cli/` — `paperclipai` CLI (onboarding, configure, doctor, worktree commands, client ops). Run via `pnpm paperclipai <cmd>`.
- `skills/` — runtime-injected agent skills. `doc/plans/` — repo plan docs (`YYYY-MM-DD-slug.md`).

## Common Commands

```sh
pnpm dev              # API + UI watch (server proxies UI in dev middleware)
pnpm dev:once         # Same, no file watching; auto-applies pending migrations
pnpm dev:server       # Server only
pnpm dev:ui           # UI only
pnpm dev:list         # Inspect managed dev runner for this repo
pnpm dev:stop         # Stop managed dev runner

pnpm build            # All packages (runs preflight workspace-link check first)
pnpm typecheck        # `pnpm -r typecheck`
pnpm test             # Vitest only (the cheap default — no Playwright)
pnpm test:watch       # Vitest watch
pnpm test:run         # Vitest via stable runner script
pnpm test:e2e         # Playwright (only when touching browser flows or CI)
pnpm test:release-smoke

pnpm db:generate      # Compile packages/db, then drizzle-kit generate
pnpm db:migrate
pnpm db:backup
pnpm storybook        # ui/ Storybook on :6006
```

`pnpm dev`/`dev:once` are **idempotent for the current repo+instance**: if a runner is already alive it reports the existing process instead of spawning a duplicate.

Run a single Vitest test: `pnpm --filter @paperclipai/server exec vitest run path/to/file.test.ts -t "case name"` (swap the filter for `@paperclipai/ui`, `@paperclipai/db`, `@paperclipai/shared`, etc.).

## Local Dev Defaults

- Leave `DATABASE_URL` unset → embedded PGlite/Postgres at `~/.paperclip/instances/default/db`.
- API + UI on `http://localhost:3100`. Health: `curl http://localhost:3100/api/health`.
- Reset local DB: `rm -rf ~/.paperclip/instances/default/db && pnpm dev`.
- Override instance/home: `PAPERCLIP_HOME=/path PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run`.
- Bind presets for authenticated mode: `pnpm dev --bind lan` or `--bind tailnet`.

### Multi-worktree dev

**Never point two Paperclip servers at the same embedded Postgres data dir.** When using `git worktree`, run `paperclipai worktree init` (or `worktree:make <name>`) inside the worktree to create an isolated instance under `~/.paperclip-worktrees/instances/<id>/` plus repo-local `.paperclip/config.json` and `.paperclip/.env` (the server/CLI auto-load this env when run inside the worktree). `pnpm dev` fails fast in a linked worktree with no `.paperclip/.env`. See `doc/DEVELOPING.md` for `worktree repair`/`reseed` flows.

## Lockfile Policy

**Do not commit `pnpm-lock.yaml` in PRs.** GitHub Actions owns the lockfile: pushes to `master` regenerate it with `--lockfile-only --no-frozen-lockfile` and commit it back, then verify with `--frozen-lockfile`.

## Architecture (big picture)

Paperclip is a control plane that orchestrates AI agents into a multi-company "company" model — not a workflow builder, not a chatbot. Core invariants you must preserve:

1. **Company-scoped everything.** Every domain entity is scoped to a company; routes and services must enforce company boundaries. Agent API keys must not access other companies.
2. **Single-assignee tasks with atomic checkout.** Status transitions to `in_progress` go through atomic checkout (no double-work). Parent/child issue hierarchy + first-class blocker dependencies.
3. **Heartbeat execution.** Agents wake on a DB-backed wakeup queue with coalescing. Each run does: budget check → workspace resolution → secret injection → skill loading → adapter invocation → structured logs/cost events/audit entries. Recovery preserves explicit ownership (see `doc/execution-semantics.md`).
4. **Governance gates.** Approval workflows (hires, CEO strategy), pause/resume/terminate, full activity log on mutating actions.
5. **Budgets are hard stops.** Monthly UTC window. Overspend auto-pauses agents and cancels queued work. Token + cost rollups by company/agent/project/goal/issue/provider/model.
6. **Adapters are pluggable.** Built-ins (`process`, `http`, local CLI/session, OpenClaw gateway) live under `packages/adapters/`. External adapters load via the plugin flow (`~/.paperclip/adapter-plugins.json`); the plugin loader has zero hardcoded adapter imports.
7. **Workspaces.** Project workspaces (config), execution workspaces (git worktrees / operator branches with optional `workspaceStrategy.provisionCommand`), and runtime services (managed dev servers, preview URLs).
8. **Company portability.** Export/import full orgs (agents, skills, projects, routines, issues) with secret scrubbing + collision handling.

Two deployment modes: `local_trusted` (implicit board, fastest local run) and `authenticated` with `private`/`public` exposure (see `doc/DEPLOYMENT-MODES.md`). API is mounted at `/api`.

## Contracts Must Stay Synced

When you change schema or API behavior, update **all** layers in lockstep: `packages/db` schema + exports → `packages/shared` types/constants/validators → `server` routes/services → `ui` API clients and pages.

## DB Change Workflow

1. Edit `packages/db/src/schema/*.ts`.
2. Export new tables from `packages/db/src/schema/index.ts`.
3. `pnpm db:generate` (compiles `packages/db` first, then generates the migration).
4. `pnpm -r typecheck` to validate.

## API/Auth Expectations

- Base path: `/api`.
- Board access = full-control operator context.
- Agent access uses bearer API keys (`agent_api_keys`), hashed at rest. Agent keys must be company-scoped.
- New endpoints must: enforce company access, distinguish board vs agent permissions, write activity-log entries on mutations, return consistent HTTP codes (`400/401/403/404/409/422/500`).

## Verification Discipline

For normal issue work, run the **smallest** targeted check that proves the change. Reserve the full sweep — `pnpm -r typecheck && pnpm test:run && pnpm build` — for PR-ready handoff or broad changes. If anything cannot be run, report what was skipped and why.

## PR Requirements

Every PR must use `.github/PULL_REQUEST_TEMPLATE.md` with all sections filled in: **Thinking Path** (top-down reasoning from project context to this change — see `CONTRIBUTING.md` for examples), **What Changed**, **Verification**, **Risks**, **Model Used** (provider, exact model ID, context window, capabilities — or "None — human-authored"), **Checklist**. Greptile must score 5/5 with all comments addressed before merge. The `Model Used` requirement is mandatory for AI contributors.

## Telemetry

Anonymous usage telemetry is on by default. Disable via `PAPERCLIP_TELEMETRY_DISABLED=1`, `DO_NOT_TRACK=1`, `CI=true`, or `telemetry.enabled: false` in the Paperclip config.
