# Ticket-to-Go 🎟️

Movie theater ticket booking platform: browse what's showing, pick exact seats on a live seat map,
hold them for 8 minutes and confirm a booking — with an admin back-office for movies, cinemas,
auditoriums and showtimes.

| Part                                                                        | Stack                                                                                             |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [`frontend/`](https://github.com/Xeubaka/frontend-ticket-to-go) (submodule) | Next.js 16 (App Router), React 19, TypeScript, Tailwind v4, TanStack Query, React Hook Form + zod |
| [`backend/`](https://github.com/Xeubaka/backend-ticket-to-go) (submodule)   | NestJS 12 REST API (`/api/v1`), Prisma 7 ORM, PostgreSQL 17, JWT + rotating refresh tokens        |
| Tests                                                                       | Vitest (unit + integration), Testcontainers, Supertest, MSW, Playwright + axe                     |

Conventions for each side are codified as Claude Code skills in
[`.claude/skills/backend-nestjs`](.claude/skills/backend-nestjs/SKILL.md) and
[`.claude/skills/frontend-nextjs`](.claude/skills/frontend-nextjs/SKILL.md).
Deployment (Neon + Cloudflare Workers/Containers) is in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).
Design decisions live in [`docs/adr`](docs/adr) and the original plan in
[`docs/IMPLEMENTATION_PLAN.md`](docs/IMPLEMENTATION_PLAN.md).

## Quick start

Requirements: Node 24, pnpm 12 (`npm i -g pnpm`), Docker.

`backend/` and `frontend/` are git submodules with their own repositories and history:

```bash
git clone --recurse-submodules <this repo>   # or, in an existing clone: git submodule update --init
```

Commit app changes inside the submodule first (and push it), then commit the updated submodule
pointer here (`git add backend frontend`).

```bash
docker compose up -d                 # postgres (5432) + postgres-test (5433)

cd backend
cp .env.example .env
pnpm install                         # also generates the Prisma client
pnpm db:deploy                       # apply migrations
pnpm db:seed                         # demo data (idempotent)
pnpm start:dev                       # http://localhost:3001  · docs: /api/docs

cd ../frontend
cp .env.example .env.local
pnpm install
pnpm dev                             # http://localhost:3000
```

Demo accounts (from the seed):

| Role     | Email                     | Password      |
| -------- | ------------------------- | ------------- |
| Admin    | `admin@tickettogo.dev`    | `Admin12345`  |
| Customer | `customer@tickettogo.dev` | `Customer123` |

## Testing

| Command                       | Where    | What                                                                                                              |
| ----------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                   | backend  | unit tests (mocked dependencies)                                                                                  |
| `pnpm test:int`               | backend  | services + repositories against a real Postgres (Testcontainers)                                                  |
| `pnpm test:e2e`               | backend  | HTTP flows through the full Nest app (Supertest + Testcontainers)                                                 |
| `pnpm test:cov`               | backend  | all three, with ≥ 80 % coverage thresholds                                                                        |
| `pnpm test` / `pnpm test:cov` | frontend | unit + MSW integration tests, ≥ 80 % thresholds                                                                   |
| `pnpm test:e2e`               | frontend | Playwright: real browser → Next build → API → seeded `postgres-test` (Chromium, Firefox, mobile) + axe a11y scans |

Docker must be running for backend integration/e2e and for Playwright. The Playwright run uses
ports 3100/3101 and the `postgres-test` database, so it never touches your dev stack or data.

## Database & migrations

The schema is `backend/prisma/schema.prisma`; it only changes through versioned migrations:

```bash
cd backend
pnpm db:migrate --name add_something   # create + apply a migration in dev (commit the SQL)
pnpm db:deploy                         # apply pending migrations (CI / production)
pnpm db:drift                          # fail if the migrated DB and schema.prisma disagree
```

Constraints Prisma can't express (e.g. the exclusion constraint preventing overlapping showtimes)
live in hand-written migration SQL.

## CI

`.github/workflows/ci.yml` runs three jobs: **backend** (lint, types, migration drift, OpenAPI
freshness, all tests + coverage), **frontend** (API-type freshness, lint, types, tests + coverage)
and **e2e** (Playwright full stack). Commits follow Conventional Commits (enforced by commitlint
via Husky; `pnpm install` at the repo root installs the hooks).
