---
name: backend-nestjs
description: Market-standard conventions for the Ticket-to-Go backend (NestJS REST API + Prisma + PostgreSQL). Use for any work under backend/ — adding modules, endpoints, DTOs, guards, Prisma schema changes, migrations, seeds, or backend tests (unit, integration, e2e).
---

# Backend skill — NestJS + Prisma + PostgreSQL

Follow these rules for every change in `backend/`. When a rule conflicts with a quick hack, the rule wins.

## 1. Architecture

```
backend/src/
├── main.ts                 # bootstrap: helmet, CORS, versioning, pipes, swagger
├── app.module.ts
├── config/                 # env schema (zod) + typed ConfigService access
├── prisma/                 # PrismaModule (global) + PrismaService
├── common/
│   ├── decorators/         # @CurrentUser, @Roles, @Public
│   ├── filters/            # ProblemDetailsFilter (RFC 9457)
│   ├── guards/             # JwtAuthGuard, RolesGuard
│   ├── dto/                # PaginationQueryDto, PaginatedResponse
│   └── utils/
└── modules/<feature>/
    ├── <feature>.module.ts
    ├── <feature>.controller.ts     # HTTP only: parse, delegate, map status
    ├── <feature>.service.ts        # business rules, transactions
    ├── <feature>.repository.ts     # Prisma queries only
    ├── dto/                        # request/response DTOs (class-validator + swagger)
    └── *.spec.ts / *.int-spec.ts
```

- **Controllers are thin.** No Prisma calls, no business rules. They validate input (DTO), call one service method, return a response DTO.
- **Services own business logic** and transactions. They depend on repositories (and other services), never on `PrismaClient` directly — except when orchestrating a transaction via `prisma.$transaction(tx => ...)`, in which case pass `tx` into repository methods.
- **Repositories own queries.** They accept an optional `Prisma.TransactionClient`.
- One feature = one module. Cross-module access goes through exported services, never another module's repository.
- No circular imports. If you need `forwardRef`, redesign.

## 2. REST API standards

- Global prefix `api`, URI versioning → `/api/v1/...`.
- Plural resource nouns, kebab-case: `/movies`, `/showtimes/:id/seats`, `/bookings/holds`.
- Status codes: `200` read/update, `201` create (with body), `204` delete/no body, `400` malformed, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict (seat taken, duplicate email), `422` validation failure, `429` rate limited.
- Pagination: `?page=1&limit=20` (max 100) → `{ data: T[], meta: { page, limit, total, totalPages } }`.
- Filtering via explicit query params (`?status=NOW_SHOWING&genre=Drama&search=...`), sorting via `?sort=field` / `?sort=-field`.
- Mutations that must not duplicate (booking confirm) accept an `Idempotency-Key` header; replays return the original result.
- **Errors use RFC 9457 Problem Details** (`application/problem+json`), produced by the global `ProblemDetailsFilter`:
  ```json
  {
    "type": "https://ticket-to-go.dev/problems/seat-unavailable",
    "title": "Seat unavailable",
    "status": 409,
    "detail": "...",
    "instance": "/api/v1/bookings/holds",
    "errors": []
  }
  ```
  Throw Nest `HttpException` subclasses (or domain errors mapped in the filter); never return error objects manually.
- Never expose Prisma models directly — map to response DTOs (no `passwordHash`, no internal fields).
- Dates in ISO 8601 UTC. Money as integer cents (`priceCents`).

## 3. Validation & DTOs

- Global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true, errorHttpStatusCode: 422 })`.
- Every body/query/param has a DTO with `class-validator` decorators and `@ApiProperty` for Swagger.
- Use `ParseUUIDPipe` / `ParseIntPipe` for params.

## 4. Configuration

- `@nestjs/config` + a **zod** schema in `src/config/env.schema.ts`; app fails fast on invalid env.
- Read config only through `ConfigService<Env, true>`; `process.env` is forbidden outside `config/` and Prisma CLI.
- Every variable documented in `backend/.env.example`.

## 5. Security

- Passwords: `argon2id`.
- Auth: short-lived JWT access token (15 min) + opaque refresh token (7 days) **rotated on every refresh**, stored **hashed** in `RefreshToken`; reuse of a revoked token revokes the whole family. Both delivered as `httpOnly`, `secure` (prod), `sameSite=lax` cookies; access token also accepted as `Authorization: Bearer`.
- `JwtAuthGuard` is global; opt out with `@Public()`. Authorization with `@Roles(Role.ADMIN)` + `RolesGuard`.
- `helmet()`, CORS allow-list from env (`credentials: true`), `@nestjs/throttler` globally with stricter limits on `/auth/*`.
- Never log secrets, tokens or passwords (pino redaction).

## 6. Observability

- `nestjs-pino` structured JSON logs, request id propagated (`x-request-id`).
- `@nestjs/terminus` health endpoint `GET /api/v1/health` with a Prisma DB check.

## 7. OpenAPI

- `@nestjs/swagger` at `/api/docs`. Every endpoint has `@ApiTags`, `@ApiOperation`, response decorators.
- `pnpm openapi:export` writes `backend/openapi.json`; the frontend generates its typed client from it. Re-export whenever the contract changes.

## 8. Prisma & migrations

- Schema conventions: `id String @id @default(uuid()) @db.Uuid`, `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt`, PascalCase models mapped to snake_case tables (`@@map("showtime_seats")`) and columns (`@map("created_at")`), explicit `@@index` for every FK and frequent filter, enums for finite states.
- **Schema changes only via migrations**: edit `schema.prisma` → `pnpm prisma migrate dev --name <verb_noun>` (e.g. `add_booking_code`) → commit the generated SQL together with the schema.
- **Never edit or delete a migration that has been applied/shared.** Fix forward with a new migration.
- Constraints Prisma can't express (exclusion constraints, partial indexes, check constraints) go into a migration created with `prisma migrate dev --create-only` and hand-edited SQL.
- CI/prod run `prisma migrate deploy` only. CI also runs a drift check (`prisma migrate diff --exit-code`).
- `prisma/seed.ts` is idempotent (upserts) and safe to run repeatedly.

## 9. Transactions & concurrency

- Any multi-write operation runs in `prisma.$transaction`.
- Seat holds are **atomic conditional updates** (`updateMany ... WHERE status = 'AVAILABLE' OR hold expired`) and compare `count` to the requested amount; mismatch → throw `ConflictException` so the transaction rolls back. Never "read then write" availability.
- Rely on DB constraints (unique, exclusion) as the last line of defence; map Prisma `P2002` to `409`.

## 10. Testing (coverage ≥ 80% lines/branches/functions/statements)

NestJS 12 scaffolds with **Vitest** (not Jest) and **oxlint**; follow the framework default. One `vitest.config.ts` defines three projects:

| Type        | Files                    | Tooling                             | Scope                                                                          |
| ----------- | ------------------------ | ----------------------------------- | ------------------------------------------------------------------------------ |
| Unit        | `src/**/*.spec.ts`       | Vitest, `vitest-mock-extended`      | services, guards, filters, pipes, utils — dependencies mocked                  |
| Integration | `src/**/*.int-spec.ts`   | Vitest + Testcontainers Postgres    | repositories/services against a real migrated DB (`test/utils/integration.ts`) |
| E2E         | `test/e2e/*.e2e-spec.ts` | Vitest + Supertest + Testcontainers | full HTTP flows through the real `AppModule` (`test/utils/e2e.ts`)             |

- Tests never touch the dev database. `test/utils/global-setup.ts` starts a Postgres container per DB project, runs `prisma migrate deploy`, and hands the URL to workers via `provide/inject`; suites truncate between tests (`test/utils/db.ts`) and run with `fileParallelism: false`.
- Use factories in `test/utils/factories.ts`; no hand-built fixtures repeated across tests.
- Arrange/Act/Assert, one behaviour per test, descriptive names (`it('returns 409 when a seat is already held')`).
- Concurrency-sensitive code gets a race test (parallel requests, assert exactly one wins).
- Assert domain errors by their problem code: `rejects.toMatchObject({ code: 'seats-unavailable' })`.
- Coverage is measured across **all three projects combined** (`pnpm test:cov`); module/DTO/generated files are excluded.
- Scripts: `pnpm test` (unit), `pnpm test:int`, `pnpm test:e2e`, `pnpm test:all`, `pnpm test:cov`. Docker must be running.

### Known pitfalls

- Decorator metadata is required for DI: run the app with `nest start`/`node dist/main.js`, never plain `tsx src/main.ts` (esbuild drops `emitDecoratorMetadata`). Standalone scripts (seed, reset) can use `node --import tsx`.
- Imports are ESM: always use the `.js` suffix in relative imports.
- On Windows, don't rely on `prisma db seed`'s exit code; call the seed script directly (`pnpm db:seed`).

## 11. Definition of done (run before finishing any backend task)

1. `pnpm lint` and `pnpm typecheck` clean.
2. `pnpm test:cov` green (unit + integration + e2e); coverage thresholds met.
3. Schema changed → migration committed; drift check passes.
4. Contract changed → Swagger decorators updated and `pnpm openapi:export` run.
5. New env vars added to `env.schema.ts` and `.env.example`.
