# Ticket-to-Go — Movie Theater Booking Platform: Implementation Plan

## Context

Greenfield project in `C:\Users\Usuario\Documents\chess-platform\ticket-to-go` (currently empty, no git). Goal: a production-grade movie-theater ticket booking platform with:

- **Frontend**: Next.js (App Router, TypeScript)
- **Backend**: NestJS REST API + Prisma ORM
- **Database**: PostgreSQL, schema evolved only through versioned Prisma migrations
- **Tests**: unit + integration + end-to-end on both apps, with enforced coverage thresholds
- **Two project skills** (`.claude/skills/`) that codify market-standard conventions for frontend and backend, so all future work follows the same rules.

Confirmed scope: **core booking**, **auth & accounts**, **admin back-office**. No payments in MVP (booking confirms directly; a `PaymentProvider` interface is left as an extension point). Repo layout: **two separate folders** (`frontend/`, `backend/`), each with its own `package.json`.

After approval, this plan is also copied to `docs/IMPLEMENTATION_PLAN.md` in the repo.

---

## 1. Repository layout

```
ticket-to-go/
├── .claude/skills/
│   ├── frontend-nextjs/SKILL.md
│   └── backend-nestjs/SKILL.md
├── backend/                 # NestJS + Prisma
├── frontend/                # Next.js
├── docs/IMPLEMENTATION_PLAN.md, docs/adr/  # architecture decision records
├── docker-compose.yml       # postgres (dev) + postgres-test, optional api/web
├── .github/workflows/ci.yml
├── .editorconfig, .gitignore, README.md
```

Tooling: Node 24 (installed), **pnpm** per folder, Docker (installed) for Postgres/Testcontainers. Versions: latest stable at scaffold time (Next.js 16.x, NestJS 11.x, Prisma 6+/7), pinned via lockfiles. `git init` + Conventional Commits, Husky + lint-staged + commitlint in each app.

---

## 2. Skill files (deliverable #1)

### `.claude/skills/backend-nestjs/SKILL.md`

Frontmatter `name` / `description` (triggers on backend/API/Prisma/migration work). Contents:

- **Architecture**: feature modules (`modules/<feature>/{controller,service,repository,dto,entities}`), thin controllers, business logic in services, data access via repositories wrapping `PrismaService`; `common/` for guards, filters, interceptors, pipes, decorators.
- **REST standards**: `/api/v1` prefix (URI versioning), plural nouns, correct status codes (201/204/409/422), pagination `?page&limit` with `{ data, meta }` envelope, filtering/sorting conventions, idempotency for booking confirm, **RFC 9457 Problem Details** error format via a global exception filter.
- **Validation**: `class-validator` DTOs + global `ValidationPipe({ whitelist, forbidNonWhitelisted, transform })`.
- **Config**: `@nestjs/config` with Zod/Joi-validated env schema; no `process.env` outside config.
- **Security**: JWT access (15 min) + rotating refresh tokens (hashed in DB, httpOnly cookie), argon2 password hashing, RBAC `@Roles()` guard, Helmet, CORS allow-list, `@nestjs/throttler` rate limiting.
- **Observability**: `nestjs-pino` structured logs with request id, `@nestjs/terminus` `/health` (DB check).
- **Docs**: `@nestjs/swagger` OpenAPI at `/api/docs`; spec exported to `backend/openapi.json` for the frontend client generation.
- **Prisma & migrations**: schema conventions (cuid/uuid ids, `createdAt/updatedAt`, snake_case `@@map`, explicit indexes), `prisma migrate dev --name <verb_noun>` only, never edit applied migrations, `migrate deploy` in CI/prod, raw SQL in migrations for constraints Prisma can't express, idempotent `prisma/seed.ts`.
- **Transactions & concurrency** rules (see §4.3).
- **Testing rules**: pyramid, file naming (`*.spec.ts` unit, `*.int-spec.ts` integration, `test/*.e2e-spec.ts`), factories, coverage ≥ 80% lines/branches, no test touches the dev DB.
- **Checklist** to run before finishing a task (lint, typecheck, tests, migration created, OpenAPI updated).

### `.claude/skills/frontend-nextjs/SKILL.md`

- **Architecture**: App Router, Server Components by default, `"use client"` only for interactivity; route groups `(public)`, `(auth)`, `(account)`, `admin/`; feature folders `src/features/<feature>/{components,hooks,api,schemas}`; shared `src/components/ui`.
- **Data**: typed API client generated from backend OpenAPI (`openapi-typescript` + `openapi-fetch`); server-side fetch in RSC with caching/revalidation tags; **TanStack Query** for client mutations/polling (seat map); Server Actions only for simple forms.
- **Forms/validation**: React Hook Form + Zod.
- **UI**: Tailwind CSS + shadcn/ui (Radix) primitives, design tokens, dark mode, responsive-first.
- **Auth**: httpOnly cookies set by API, `middleware.ts` protects `(account)` and `admin/` (role check), no tokens in localStorage.
- **Quality**: strict TS, ESLint (next + a11y + import order), Prettier, WCAG 2.2 AA (keyboard-navigable seat map, aria labels), `next/image`, metadata/SEO per route, `loading.tsx`/`error.tsx`/`not-found.tsx` boundaries, Core Web Vitals budget.
- **Testing rules**: Vitest + React Testing Library (unit), MSW (integration), Playwright (e2e) + `@axe-core/playwright`; coverage ≥ 80%.
- **Checklist** before finishing a task.

---

## 3. Domain model (Prisma, `backend/prisma/schema.prisma`)

| Model          | Key fields                                                                                                                             |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `User`         | id, email (unique), passwordHash, name, role (`CUSTOMER`/`ADMIN`)                                                                      |
| `RefreshToken` | id, userId, tokenHash, expiresAt, revokedAt                                                                                            |
| `Movie`        | id, title, slug (unique), synopsis, durationMin, rating, genres[], posterUrl, releaseDate, status                                      |
| `Cinema`       | id, name, address, city                                                                                                                |
| `Auditorium`   | id, cinemaId, name, rows, cols                                                                                                         |
| `Seat`         | id, auditoriumId, row, number, type (`STANDARD`/`VIP`/`ACCESSIBLE`); unique(auditoriumId,row,number)                                   |
| `Showtime`     | id, movieId, auditoriumId, startsAt, endsAt, basePriceCents; index(movieId,startsAt)                                                   |
| `ShowtimeSeat` | id, showtimeId, seatId, status (`AVAILABLE`/`HELD`/`BOOKED`), heldByUserId, holdExpiresAt, bookingId; **unique(showtimeId, seatId)**   |
| `Booking`      | id, code (unique, human-readable), userId, showtimeId, status (`PENDING`/`CONFIRMED`/`CANCELLED`), totalCents, idempotencyKey (unique) |
| `Ticket`       | id, bookingId, showtimeSeatId, priceCents                                                                                              |

Money stored as integer cents. Migrations under `backend/prisma/migrations/`, first ones: `init_schema`, `add_showtime_overlap_constraint` (raw SQL `EXCLUDE USING gist` preventing overlapping showtimes in the same auditorium — needs `btree_gist` extension).

---

## 4. Backend (`backend/`)

### 4.1 Modules & endpoints (`/api/v1`)

- **auth**: `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`
- **movies**: `GET /movies` (filter by status/genre/search, paginated), `GET /movies/:slug`; admin `POST/PATCH/DELETE /movies/:id`
- **cinemas / auditoriums**: public `GET`; admin CRUD; creating an auditorium generates its `Seat` grid
- **showtimes**: `GET /showtimes?movieId&cinemaId&date`, `GET /showtimes/:id/seats` (seat map with status); admin CRUD (creation materializes `ShowtimeSeat` rows in one transaction)
- **bookings**: `POST /bookings/holds` (hold seats), `DELETE /bookings/holds/:showtimeId` (release), `POST /bookings` (confirm, `Idempotency-Key` header), `GET /bookings/me`, `GET /bookings/:code`, `POST /bookings/:code/cancel`
- **admin reports**: `GET /admin/showtimes/:id/occupancy`
- **health**: `GET /health`

### 4.2 Folder structure

`src/{main.ts, app.module.ts, config/, prisma/ (PrismaModule, PrismaService), common/ (filters, guards, decorators, interceptors), modules/<feature>/...}`, `prisma/{schema.prisma, migrations/, seed.ts}`, `test/{e2e/, utils/ (factories, testcontainers setup)}`.

### 4.3 Seat-hold concurrency (critical logic)

- Hold = single atomic `updateMany` on `ShowtimeSeat` `WHERE id IN (...) AND (status='AVAILABLE' OR (status='HELD' AND holdExpiresAt < now()))` inside a transaction; if affected count ≠ requested count → rollback + `409 Conflict`. Hold TTL 8 min, max 10 seats per user per showtime.
- Confirm = transaction that verifies all seats are `HELD` by the user and not expired, creates `Booking` + `Ticket`s, sets seats `BOOKED`. `idempotencyKey` unique prevents double bookings on retries.
- Expired holds are treated as available by queries; a `@nestjs/schedule` cron job sweeps them every minute.
- Cancel (before showtime start) returns seats to `AVAILABLE`.

### 4.4 Backend testing

| Layer                                                                                                                                       | Tooling                                                                                                  | What                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unit (`*.spec.ts`)                                                                                                                          | Jest, mocked repositories (`jest-mock-extended`)                                                         | services (pricing, hold rules, auth token rotation), guards, pipes, filters                                                                                         |
| Integration (`*.int-spec.ts`)                                                                                                               | Jest + **Testcontainers PostgreSQL**, `prisma migrate deploy` on container start, truncate between tests | repositories + services against a real DB, including a **concurrent-hold race test** (N parallel holds on same seat → exactly 1 succeeds), migrations apply cleanly |
| E2E (`test/e2e/*.e2e-spec.ts`)                                                                                                              | Jest + Supertest on the full Nest app + Testcontainers                                                   | register → login → browse → hold → confirm → my bookings → cancel; admin CRUD; authz (403/401); validation (422) and Problem Details shape                          |
| Coverage: Jest `coverageThreshold` global 80% (lines, branches, functions, statements); scripts `test`, `test:int`, `test:e2e`, `test:cov`. |

---

## 5. Frontend (`frontend/`)

### 5.1 Routes

- `(public)`: `/` (now showing / coming soon), `/movies/[slug]` (details + showtimes by date/cinema), `/showtimes/[id]` (interactive seat map, hold countdown), `/checkout/[showtimeId]` (review + confirm), `/bookings/[code]` (confirmation with QR code)
- `(auth)`: `/login`, `/register`
- `(account)`: `/account/bookings` (history, cancel)
- `admin/`: dashboard, `movies`, `cinemas`, `auditoriums`, `showtimes` (tables + forms), occupancy view

### 5.2 Structure

`src/app/...`, `src/features/{movies,showtimes,seat-map,booking,auth,admin}/`, `src/lib/api/` (generated `schema.d.ts` + client with cookie forwarding for RSC), `src/components/ui/` (shadcn), `middleware.ts`. Seat map: accessible grid (`role="grid"`, arrow-key navigation), polling via TanStack Query (5 s) to reflect other users' holds, optimistic hold with rollback on 409.

### 5.3 Frontend testing

| Layer                                        | Tooling                                                      | What                                                                                                                                                                                                          |
| -------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unit                                         | Vitest + RTL                                                 | components (SeatMap, SeatLegend, HoldTimer, forms), hooks, utils (price formatting, seat selection rules)                                                                                                     |
| Integration                                  | Vitest + RTL + **MSW**                                       | feature flows against mocked API: selecting seats → hold → 409 conflict handling; login form → redirect; admin form validation                                                                                |
| E2E                                          | **Playwright** against real backend + test Postgres (seeded) | full booking journey, concurrent booking (two browser contexts, one gets conflict), auth redirects, admin creates showtime visible publicly, axe a11y scan on key pages; Chromium + Firefox + mobile viewport |
| Coverage: Vitest v8 coverage thresholds 80%. |

---

## 6. Database & migrations workflow

- `docker-compose.yml`: `postgres:17` (dev, port 5432, volume) and `postgres-test` (port 5433, tmpfs) used by Playwright.
- Dev: `pnpm prisma migrate dev --name <change>` → commit generated SQL. Prod/CI: `pnpm prisma migrate deploy`.
- CI guard: `prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma --exit-code` fails if schema changed without a migration.
- `prisma/seed.ts`: 1 admin, 2 cinemas, 4 auditoriums, ~8 movies, a week of showtimes (idempotent upserts).

---

## 7. CI (`.github/workflows/ci.yml`)

Jobs (parallel where possible): **backend** (install → lint → typecheck → unit → integration → e2e → coverage), **frontend** (install → lint → typecheck → unit/integration → build), **e2e-fullstack** (Postgres service → migrate deploy → seed → start API + `next start` → Playwright; upload report/traces). Coverage reports uploaded as artifacts; thresholds fail the build.

---

## 8. Execution phases

1. **Foundation**: `git init`, root files, docker-compose, both skill files.
2. **Backend scaffold**: Nest app, config, Prisma, `PrismaService`, global pipes/filters, health, Swagger, test infra (Jest projects, Testcontainers helper, factories).
3. **Schema & migrations**: full Prisma schema, `init_schema` + overlap-constraint migrations, seed.
4. **Backend features** (each with unit + integration + e2e tests): auth → movies → cinemas/auditoriums/seats → showtimes → bookings/holds → admin reports.
5. **Frontend scaffold**: Next.js, Tailwind + shadcn, ESLint/Prettier, Vitest + MSW + Playwright setup, generated API client.
6. **Frontend features** (with tests): public catalog → auth → seat map & hold → checkout/confirmation → account bookings → admin back-office.
7. **Full-stack E2E + CI**: Playwright journeys, GitHub Actions workflow, README with setup instructions.

---

## 9. Verification

- `docker compose up -d` → `cd backend && pnpm prisma migrate deploy && pnpm prisma db seed && pnpm start:dev` → Swagger at `http://localhost:3001/api/docs`, `/api/v1/health` returns OK.
- `cd backend && pnpm lint && pnpm test:cov && pnpm test:int && pnpm test:e2e` → all green, coverage ≥ 80%.
- `cd frontend && pnpm dev` → manually book seats at `http://localhost:3000` as seeded customer; log in as admin and create a showtime.
- `cd frontend && pnpm lint && pnpm test:cov && pnpm test:e2e` → all green, coverage ≥ 80%, axe finds no serious violations.
- `prisma migrate diff ... --exit-code` returns 0 (no schema drift).
- Race check: integration test proves only one of N concurrent holds on a seat succeeds.

---

## 10. Implementation notes: deviations from this plan (as built, 2026-09-25)

| Plan                                                  | As built                                                                                                                           | Why                                                                                            |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Jest (backend)                                        | **Vitest** + `vitest-mock-extended`; **oxlint** instead of ESLint                                                                  | NestJS 12's CLI scaffolds with Vitest/oxlint; we followed the framework default (see ADR 0003) |
| `middleware.ts`                                       | **`src/proxy.ts`**                                                                                                                 | Next.js 16 renamed Middleware to Proxy                                                         |
| RSC fetch with forwarded cookies for user data        | Authenticated data fetched **client-side** (TanStack Query) behind a same-origin `/api` rewrite; RSC only for the public catalogue | Keeps the refresh-token flow in one place (ADR 0002)                                           |
| `Booking.status` incl. `PENDING`                      | `CONFIRMED` / `CANCELLED` only                                                                                                     | No payment step in the MVP                                                                     |
| `prisma db seed`                                      | `pnpm db:seed` (`node --import tsx prisma/seed.ts`)                                                                                | The Prisma CLI's exit code is unreliable on Windows                                            |
| `prisma migrate diff --from-migrations …` drift check | `prisma migrate diff --from-config-datasource --to-schema …` after `migrate deploy`                                                | Prisma 7 needs a shadow DB for the former                                                      |
| Separate Playwright DB via Testcontainers             | docker compose `postgres-test` (5433), reset + seeded per run                                                                      | Playwright's webServer needs a fixed URL                                                       |
| shadcn CLI                                            | Hand-written shadcn-style primitives (`cva` + Tailwind)                                                                            | Same output, no CLI or network step                                                            |
| Husky/commitlint in each app                          | One root `package.json` holding only git-hook tooling                                                                              | Hooks are per repository                                                                       |

Additions: the venue timezone for displaying and entering times (`NEXT_PUBLIC_TIME_ZONE`), a DB exclusion
constraint for showtime overlap, idempotent booking confirmation, refresh-token family revocation on
reuse, and CI checks that the OpenAPI document and generated client types are up to date.
