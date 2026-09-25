# Deployment: Neon + Cloudflare

```
browser ──► ticket-to-go-web (Worker, Next.js via OpenNext)
              │  /api/* rewrite
              ▼
            ticket-to-go-api (Worker) ──► ApiContainer (NestJS, Cloudflare Containers)
                                              │  pooled TLS connection
                                              ▼
                                            Neon Postgres
```

Deploy order: **Neon → backend → frontend**. The frontend build prerenders pages from the API and
bakes the `/api` rewrite target in, so the API must be live first.

## 1. Neon (PostgreSQL)

1. Create a project at <https://console.neon.tech>: Postgres **17**, database name `ticket_to_go`,
   region close to your users (for example AWS `us-east-1` or `eu-central-1`).
2. Open **Connect** on the project dashboard and copy **two** connection strings:

   | Variable              | Connection pooling toggle | Host looks like                        | Used by                 |
   | --------------------- | ------------------------- | -------------------------------------- | ----------------------- |
   | `DATABASE_URL`        | **on** (PgBouncer)        | `ep-xxx-pooler.<region>.aws.neon.tech` | the running API         |
   | `DIRECT_DATABASE_URL` | **off**                   | `ep-xxx.<region>.aws.neon.tech`        | `prisma migrate deploy` |

   Both must end with `?sslmode=require&channel_binding=require` (Neon adds it). Migrations need the
   direct endpoint because PgBouncer's transaction mode doesn't support Prisma's migration locking.

3. Nothing else to enable: the `btree_gist` extension (showtime overlap constraint) is created by our
   migration and is supported on Neon.
4. Optional, to load the demo data from your machine (`backend/`):

   ```bash
   DATABASE_URL="<direct url>" pnpm db:deploy
   DATABASE_URL="<direct url>" pnpm db:seed
   ```

   Migrations also run automatically every time the API container starts.

Neon computes suspend when idle (the free plan does this automatically); the first query after
a suspend takes a moment longer.

## 2. Backend: Cloudflare Worker + Container (`backend/`)

Requires the **Workers Paid** plan ($5/month; see pricing notes below) and Docker running locally.

```bash
cd backend
npx wrangler login                               # once
npx wrangler secret put DATABASE_URL             # Neon pooled URL
npx wrangler secret put DIRECT_DATABASE_URL      # Neon direct URL
npx wrangler secret put JWT_ACCESS_SECRET        # random, ≥ 32 chars (e.g. openssl rand -base64 48)
pnpm cf:deploy                                   # builds the Docker image, pushes it, deploys the Worker
```

- The API URL is `https://ticket-to-go-api.<your-subdomain>.workers.dev`
  (health check: `/api/v1/health`). The first deploy takes a few minutes before the container answers.
- In `wrangler.jsonc`, set `vars.CORS_ORIGINS` to the frontend URL once you know it, then redeploy.
- Config: `wrangler.jsonc` (container `basic`: ¼ vCPU, 1 GiB; `max_instances: 1`), `cloudflare/worker.ts`
  (forwards requests; `sleepAfter = '10m'` scales to zero), `Dockerfile` (runs `prisma migrate deploy`
  and then the API on port 8080).
- Rate limiting keys on `cf-connecting-ip`, because every request reaches the container from the Worker.

## 3. Frontend: Cloudflare Worker via OpenNext (`frontend/`)

Recommended: **Workers Builds** connected to the `frontend-ticket-to-go` repo. Don't use the umbrella
repo: it has no app at its root, which is what caused
`Could not detect a directory containing static files`.

Dashboard → Workers & Pages → Create → Import a repository → `Xeubaka/frontend-ticket-to-go`:

| Setting         | Value                                                                                                 |
| --------------- | ----------------------------------------------------------------------------------------------------- |
| Build command   | `pnpm cf:build`                                                                                       |
| Deploy command  | `npx wrangler deploy`                                                                                 |
| Build variables | `API_INTERNAL_URL=https://ticket-to-go-api.<your-subdomain>.workers.dev`, `NEXT_PUBLIC_TIME_ZONE=UTC` |

Also set `vars.API_INTERNAL_URL` in `frontend/wrangler.jsonc` to the same URL (read at runtime by
Server Components). The Worker name in `wrangler.jsonc` (`ticket-to-go-web`) must match the
dashboard project name.

To deploy from a machine instead: `pnpm cf:deploy` (use WSL/Linux; on native Windows the OpenNext
packaging step fails with `EPERM: symlink` unless Developer Mode is enabled).

Remove or disconnect the existing Cloudflare project that builds the umbrella repo.

## 4. Verify

1. `https://ticket-to-go-api.<sub>.workers.dev/api/v1/health` returns `"database": "up"`.
2. Open the frontend URL, sign in with a seeded account (if you seeded), and book seats.

## Costs (checked 2026-09)

- Workers Paid: $5/month, including 25 GiB-h memory, 375 vCPU-min and 200 GB-h disk of Containers usage.
  The container bills only while awake. At low traffic it stays around the included usage; running 24/7
  on `basic` costs roughly $20 on top.
- Frontend Worker: request-based, covered by the plan for normal traffic.
- Neon: the free plan is enough for a demo; check current limits at <https://neon.com/pricing>.

## Known MVP limitations

- No R2 incremental cache: catalogue pages render per request instead of using ISR. Add an R2 bucket
  and `r2IncrementalCache` in `open-next.config.ts` to enable caching.
- One container instance (`max_instances: 1`). Raise it once the hold-expiry cron is moved to a
  Cloudflare Cron Trigger. Holds are correct regardless, since expired holds count as available.
