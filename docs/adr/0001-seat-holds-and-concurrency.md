# ADR 0001 — Seat holds via atomic conditional updates

**Status:** accepted · 2026-09-25

## Context

Many customers can try to book the same seat at the same moment. A booking must never be sold
twice, and a customer needs time at checkout without losing the seats they picked.

## Decision

- Each showtime materialises one `ShowtimeSeat` row per auditorium seat, with
  `status ∈ {AVAILABLE, HELD, BOOKED}`, `heldByUserId`, `holdExpiresAt` and
  `UNIQUE(showtime_id, seat_id)`.
- **Hold** = a single `UPDATE … WHERE id IN (…) AND (status = 'AVAILABLE' OR hold expired OR held by me)`
  inside a transaction. If the affected row count differs from the requested count the
  transaction throws `409 seats-unavailable` and rolls back (including releasing the user's
  previous hold). Postgres row locks serialise concurrent claims, and under READ COMMITTED the
  loser re-evaluates the `WHERE` and matches fewer rows. No read-then-write.
- **Confirm** converts the caller's still-valid holds to `BOOKED` in one transaction and creates the
  `Booking` + `Ticket`s. `UNIQUE(user_id, idempotency_key)` makes retries return the original booking.
- Expired holds are treated as available by every query; a per-minute cron only tidies the columns.
- Showtime overlap per auditorium is enforced by a Postgres `EXCLUDE USING gist` constraint as well
  as a service-level check.

## Consequences

- Proven by an integration test in which 8 parallel holds on one seat yield exactly one success, and by a
  two-browser Playwright test.
- No distributed locks or Redis needed; everything relies on Postgres guarantees.
- Hold TTL (8 min) and max seats (10) are configurable via env.
