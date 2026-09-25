# Ticket-to-Go — project guide for Claude

Movie theater ticket booking platform. Two independent apps, each a **git submodule with its own repo**:

- `backend/` → `Xeubaka/backend-ticket-to-go`: NestJS 12 REST API (`/api/v1`), Prisma 7 + PostgreSQL 17, ESM (`.js` import suffixes), Vitest + oxlint.
- `frontend/` → `Xeubaka/frontend-ticket-to-go`: Next.js 16 App Router, React 19, Tailwind v4, TanStack Query, RHF + zod 4, Vitest + MSW, Playwright.
- The root repo holds docs, skills, CI, `docker-compose.yml` and git-hook tooling (Husky + commitlint: Conventional Commits).
- Git workflow: commit (and push) inside `backend/` or `frontend/` first, then `git add backend frontend` + commit in the root to bump the submodule pointers. Each repo has its own `.gitignore`/`.gitattributes`.

## Read before working

| Working on                                | Load first                                                                                                                                                                      |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| anything in `backend/`                    | skill `backend-nestjs` (`.claude/skills/backend-nestjs/SKILL.md`)                                                                                                               |
| anything in `frontend/`                   | skill `frontend-nextjs` (`.claude/skills/frontend-nextjs/SKILL.md`), and `frontend/AGENTS.md`: Next 16 APIs differ from older versions, so check `node_modules/next/dist/docs/` |
| seat holds / booking concurrency          | `docs/adr/0001-seat-holds-and-concurrency.md`                                                                                                                                   |
| auth, cookies, `/api` rewrite, `proxy.ts` | `docs/adr/0002-auth-cookies-and-bff-rewrite.md`                                                                                                                                 |
| tests, Playwright setup, Windows quirks   | `docs/adr/0003-testing-strategy.md`                                                                                                                                             |
| original scope and deviations             | `docs/IMPLEMENTATION_PLAN.md` (§10 lists what changed)                                                                                                                          |

For "where is X / what calls Y" questions, use the graph (see the graphify section below) before grepping.

## Domain in one paragraph

`Movie` → `Showtime` (in an `Auditorium` of a `Cinema`) → one `ShowtimeSeat` per `Seat` with
`status AVAILABLE|HELD|BOOKED`. A customer **holds** seats (atomic conditional `updateMany`, 8-min TTL,
max 10, replaces their previous hold; expired holds count as available), then **confirms** (idempotent via
`Idempotency-Key`) → `Booking` (`TTG-XXXXXXXX` code) + `Ticket`s priced from `basePriceCents` × seat
multiplier (VIP 1.5). Cancellation before start frees seats. Admins manage movies, cinemas, auditoriums
(seat grid generated) and showtimes (DB exclusion constraint prevents overlaps; 15-min cleaning buffer).
Money is integer cents; errors are RFC 9457 Problem Details with codes like `seats-unavailable`.

## Key files

- Backend: `src/app.module.ts` (global guards/pipe/filter), `src/app.setup.ts` (prefix, versioning, helmet, swagger),
  `src/modules/bookings/bookings.{service,repository}.ts` (hold/confirm logic), `src/common/filters/problem-details.filter.ts`,
  `src/common/guards/jwt-auth.guard.ts` (global; opt out with `@Public()`), `prisma/schema.prisma`, `prisma/migrations/`, `prisma/seed.ts`.
- Frontend: `src/lib/api/client.ts` (typed client + 401→refresh retry), `src/lib/api/errors.ts` (`ApiError`, `unwrap`),
  `src/lib/format.ts` (venue timezone formatting), `src/proxy.ts` (route guard), `src/features/seat-map/`, `src/features/booking/`,
  `src/features/admin/`, `src/app/**` (thin pages).
- Contract: `backend/openapi.json` → `frontend/src/lib/api/schema.d.ts` (generated; never edit by hand).

## Commands

```bash
docker compose up -d                       # postgres :5432 (dev) + postgres-test :5433 (Playwright)

# backend/
pnpm start:dev                             # API on :3001, docs at /api/docs
pnpm lint && pnpm typecheck
pnpm test | test:int | test:e2e | test:cov # unit | Testcontainers | Supertest | all + 80% thresholds
pnpm db:migrate --name <verb_noun>         # schema change → commit the generated SQL
pnpm db:deploy · db:seed · db:drift
pnpm build && pnpm openapi:export          # after any API contract change

# frontend/
pnpm dev                                   # :3000 (rewrites /api → API_INTERNAL_URL, resolved at build time)
pnpm lint && pnpm typecheck                # lint is --max-warnings=0
pnpm test | test:cov                       # Vitest unit + MSW integration, 80% thresholds
pnpm test:e2e                              # Playwright: API :3101 + Next :3100 + postgres-test (Chromium, Firefox, mobile)
pnpm api:generate                          # after backend openapi.json changes
```

Seed logins: `admin@tickettogo.dev` / `Admin12345`, `customer@tickettogo.dev` / `Customer123`.

## Rules that bite

- **Schema changes only via migrations**; never edit an applied migration. CI runs `db:drift` and fails on `openapi.json` / `schema.d.ts` drift.
- Backend DI needs decorator metadata: run through `nest`/`node dist/main.js`, never `tsx src/main.ts`.
- Seat availability is **never** read-then-write; use the conditional update and compare counts (ADR 0001).
- Frontend: authenticated data is fetched client-side (TanStack Query); Server Components fetch only public catalogue data.
- Always format dates with the venue timezone helpers (`lib/format.ts`); raw `toLocale*` causes hydration mismatches.
- Use `text-primary-text` for amber text (contrast); no route-level `loading.tsx` on pages that call `notFound()`.
- Windows: don't trust `prisma db seed`'s exit code (use `pnpm db:seed`); don't `process.exit()` right after `fetch`;
  interrupted Playwright runs can orphan `node` on ports 3100/3101, so free them before re-running.
- Definition of done: lint + typecheck + `test:cov` green in the app you touched, and `test:e2e` for affected user journeys.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:

- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
