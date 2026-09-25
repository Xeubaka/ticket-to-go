---
name: frontend-nextjs
description: Market-standard conventions for the Ticket-to-Go frontend (Next.js 16 App Router + TypeScript + Tailwind v4 + shadcn-style primitives + TanStack Query). Use for any work under frontend/ — pages, layouts, components, data fetching, forms, auth routing (proxy.ts), accessibility, and frontend tests (Vitest unit/integration with MSW, Playwright e2e).
---

# Frontend skill — Next.js 16 (App Router)

Follow these rules for every change in `frontend/`. Next 16 differs from older versions (e.g.
`middleware.ts` is now `proxy.ts`, `params`/`searchParams` are Promises): check
`node_modules/next/dist/docs/` before using an API you are unsure about.

## 1. Architecture

```
frontend/src/
├── app/
│   ├── (public)/           # /, movies/[slug], showtimes/[id]   (server-rendered catalogue)
│   ├── (auth)/             # login, register
│   ├── (account)/          # account/bookings, checkout/[showtimeId], bookings/[code]
│   ├── admin/              # back-office (ADMIN role)
│   ├── layout.tsx, providers.tsx, globals.css (design tokens)
│   └── error.tsx, not-found.tsx
├── features/<feature>/     # auth, movies, showtimes, seat-map, booking, admin
│   ├── components/         # feature UI
│   ├── api.ts              # query options + mutations (client)   | api.server.ts (server-only)
│   ├── hooks.ts            # client hooks when a feature needs them
│   └── schemas.ts          # zod schemas for forms
├── components/ui/          # shadcn-style primitives (cva variants): Button, Input, Card, Alert…
├── components/layout/      # header, user menu, footer
├── lib/api/                # schema.d.ts (generated), client.ts, errors.ts, server.ts
├── lib/                    # format.ts (venue timezone!), utils.ts, auth/token.ts, env.ts
├── proxy.ts                # optimistic route protection (Next 16's middleware)
└── test/                   # setup, MSW server/handlers, fixtures, render helpers
```

- **Server Components by default.** Add `"use client"` only for state, effects, event handlers or browser APIs — and push it as far down the tree as possible.
- `app/` files stay thin: fetch public data and compose feature components.
- No cross-feature deep imports except through a feature's public `api`/`components`.
- Absolute imports via `@/`.

## 2. Data fetching

- Typed client generated from `backend/openapi.json` (`pnpm api:generate` → `src/lib/api/schema.d.ts`), consumed via `openapi-fetch`. Never hand-write types that exist in the contract. Wrap calls in `unwrap()` which throws `ApiError`.
- The browser only talks to **same-origin `/api/*`**, rewritten to the NestJS API in `next.config.ts` (rewrites resolve at **build** time). Auth cookies therefore stay first-party.
- **Public catalogue** (movies, showtime lists): Server Components via `serverApi` in `*.api.server.ts` (`import 'server-only'`), with `next: { revalidate, tags }`; volatile data (availability) uses `cache: 'no-store'`.
- **Authenticated or live data** (session, seat map, holds, bookings, admin): client components with TanStack Query. `fetchWithRefresh` retries once after a single-flight `POST /auth/refresh` on 401. Query keys are centralised per feature (`seatMapKeys.detail(id)`).
- Handle errors as RFC 9457 Problem Details: branch on `ApiError.code` (e.g. `seats-unavailable`, `no-active-hold`), show `errorMessage(error)`, never raw JSON.
- Prefer **derived state** over `useEffect` + `setState` (e.g. reconcile a seat selection against fresh data during render).

## 3. Forms

- React Hook Form + zod (`@hookform/resolvers/zod`). Use `useForm<Input, unknown, Output>` when schemas transform values.
- zod 4: normalise **before** validating (`z.string().trim().toLowerCase().pipe(z.email())`); `z.email().trim()` validates the untrimmed value.
- Every control goes through `FormField` (label + hint + error wired with `aria-describedby`/`aria-invalid`). Map server field errors with `ApiError.fieldErrors()`.
- Submit buttons use `pending` (disabled + `aria-busy`). Mutations that must not duplicate send an `Idempotency-Key` generated once per attempt.

## 4. UI & styling

- Tailwind v4 with design tokens declared as CSS variables in `globals.css` (`@theme inline`). Light is the base, dark follows `prefers-color-scheme`.
- Use `text-primary-text` (not `text-primary`) for amber text/icons: plain amber fails contrast on light backgrounds.
- `cn()` for class merging, `cva` for variants. No inline styles, no CSS-in-JS.
- `next/image` for posters (allowed hosts in `next.config.ts`; unknown hosts render `unoptimized`); `next/font` for fonts.
- **Dates/times are shown and entered in the venue timezone** (`NEXT_PUBLIC_TIME_ZONE`, see `lib/format.ts`): never `toLocaleString()` without a `timeZone`, or server and browser output will differ (hydration errors). Convert admin input with `venueLocalToIso`.
- Avoid a route-level `loading.tsx` on pages that can `notFound()`: streaming commits a 200 before the 404. Use in-component skeletons instead.

## 5. Auth & routing

- The API sets `httpOnly` cookies: `access_token` (path `/`) and `refresh_token` (path `/api/v1/auth`). Tokens are never readable by JS or stored in `localStorage`.
- `src/proxy.ts` optimistically guards `/account`, `/checkout`, `/bookings` (valid, unexpired token) and `/admin` (role `ADMIN`) by **decoding** the JWT; the API still authorises every request. Anonymous → `/login?next=…`; non-admin → `/?forbidden=1`.
- `/login` renders `SessionRestorer` first: if the refresh cookie is still valid it rotates it silently and bounces back to `next`.
- Always sanitise redirects with `safeNextPath()`.

## 6. Quality bar

- TypeScript `strict`, no `any`. ESLint (`next/core-web-vitals`, full `jsx-a11y` recommended, consistent type imports) + Prettier (tailwind plugin); **zero warnings** (`pnpm lint` uses `--max-warnings=0`).
- **Accessibility WCAG 2.2 AA**: semantic HTML, skip link, visible focus, contrast, reduced motion. The seat map is an ARIA `grid` with a roving tabindex (arrows/Home/End) and labels like `Row C, seat 7, VIP, $22.50, available`.
- SEO: `generateMetadata` per route (Open Graph for movies); private pages set `robots: { index: false }`.
- Env: server-only vars validated with zod in `src/lib/env.ts`; only `NEXT_PUBLIC_*` reach the browser. Document every var in `.env.example`.

## 7. Testing (coverage ≥ 80%)

| Type        | Files                   | Tooling                             | Scope                                                             |
| ----------- | ----------------------- | ----------------------------------- | ----------------------------------------------------------------- |
| Unit        | `src/**/*.test.ts(x)`   | Vitest (jsdom) + RTL + user-event   | components, utils, schemas, proxy (`// @vitest-environment node`) |
| Integration | `src/**/*.int.test.tsx` | Vitest + RTL + **MSW**              | feature flows against a mocked API (hold → 409 → UI recovers)     |
| E2E         | `e2e/*.spec.ts`         | Playwright + `@axe-core/playwright` | real browser → real Next build → real API → seeded test DB        |

- Query by role/label/text, never by class names; `renderWithClient()` from `src/test/render.tsx` gives a fresh QueryClient.
- MSW: defaults in `src/test/msw/handlers.ts` (signed out); override per test with `server.use(...)` and the `problem()` helper; fixtures in `src/test/fixtures.ts` are typed from the OpenAPI schema. `onUnhandledRequest: 'error'`.
- `next/navigation` and `next/image` are mocked in `src/test/setup.ts` (`mockRouter`, `navigationState`).
- Playwright (`playwright.config.ts`): starts the API on **3101** against the `postgres-test` compose service (reset + seed on every run) and a separate Next build (`.next-e2e`) on **3100** — the dev stack on 3000/3001 is never touched. Page objects/helpers live in `e2e/fixtures.ts`. Runs serially (shared DB) on Chromium, Firefox and a mobile viewport. Key pages must pass axe with no serious/critical violations.
- In Playwright, `getByRole('alert')` also matches Next's route announcer — assert on text instead.
- Scripts: `pnpm test`, `pnpm test:cov`, `pnpm test:e2e` (needs `docker compose up -d`).

## 8. Definition of done (run before finishing any frontend task)

1. `pnpm lint` and `pnpm typecheck` clean.
2. `pnpm test:cov` green with thresholds met; `pnpm test:e2e` green for affected journeys.
3. Backend contract changed → `pnpm api:generate` run and `schema.d.ts` committed (CI diffs it).
4. New UI is keyboard-accessible, responsive, and has loading/error/empty states.
