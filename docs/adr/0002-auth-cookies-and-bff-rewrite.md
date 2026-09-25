# ADR 0002 — httpOnly cookie auth behind a same-origin rewrite

**Status:** accepted · 2026-09-25

## Context

The SPA-style parts of the site (seat map, checkout, admin) need authenticated calls from the
browser, and some routes must be protected before rendering. Tokens must not be readable by JS.

## Decision

- The API issues a short-lived JWT access token (15 min) and an opaque refresh token (7 days,
  stored hashed, **rotated on every use**; reuse of a rotated token revokes the whole family).
  Both are `httpOnly`, `SameSite=Lax` cookies. The refresh cookie is path-scoped to `/api/v1/auth`.
- The browser calls **same-origin** `/api/*`; Next.js rewrites it to the API. The cookies are
  first-party, so there are no CORS preflights in the normal flow.
- Client data fetching goes through `fetchWithRefresh`: on a 401 it performs one single-flight
  refresh and replays the request.
- `src/proxy.ts` (Next 16 middleware) decodes the access token **without verifying it**, for
  optimistic redirects only (`/account`, `/checkout`, `/bookings`, `/admin`). The API verifies and
  authorises every request.
- The refresh cookie isn't visible to page routes, so when the access cookie has expired the proxy
  sends the user to `/login?next=…`, where `SessionRestorer` silently refreshes and bounces back.

## Consequences

- Authenticated data is fetched client-side (TanStack Query); Server Components fetch only public
  catalogue data. This keeps the refresh flow in one place.
- The API URL is baked into the rewrite at build time (`API_INTERNAL_URL`), so rebuild when it changes.
