# SlotPilot

A booking & scheduling engine for service businesses (salons, clinics, tutors,
studios) with **first-class double-booking protection**. Customers see real-time
availability; staff calendars never get two bookings in the same slot.

> Portfolio note: clean **ports & adapters** architecture, a pure deterministic
> scheduling engine, strict TypeScript, and full unit + e2e test coverage.

## Stack

- **TypeScript** (strict, `noUncheckedIndexedAccess`) · **Node 22**
- **NestJS** (HTTP) · **class-validator** request validation
- **Jest** + **supertest** (unit + e2e) · **Docker** · **GitHub Actions**

## Architecture

```
src/
  domain/          pure scheduling logic — no framework, no I/O
    time.ts        half-open intervals, overlap, UTC helpers
    scheduling.ts  slot generation + bookability rules
    booking.ts     entities (Service, Resource, Booking)
  application/     use-cases + ports (repository/clock interfaces)
  infrastructure/  adapters (in-memory repos; swap for Postgres in prod)
  http/            NestJS controller, DTOs, domain-error → HTTP filter
```

The **scheduling engine is pure**: same inputs always produce the same slots,
so it is trivially testable. The **no-overlap invariant** is enforced atomically
at the repository boundary — in production this maps to a Postgres
`EXCLUDE USING gist (resource_id WITH =, tstzrange(start,end) WITH &&)`
exclusion constraint.

### Correctness guarantees (covered by tests)

- Half-open intervals: back-to-back bookings (09:00–09:30, 09:30–10:00) never
  collide, but any true overlap is rejected.
- Slots in the past, outside business hours, or crossing UTC midnight are rejected.
- Booked slots disappear from availability; canceling frees them again.
- Concurrent identical requests cannot both succeed (single check-and-set).

## Run

```bash
npm install
npm run start:dev      # http://localhost:3000
```

### Try it

```bash
# Demo data: resource "res_alex", services "svc_haircut" (30m) / "svc_color" (90m),
# available Mon–Fri 09:00–17:00 UTC.
curl "localhost:3000/availability?resourceId=res_alex&serviceId=svc_haircut&day=2026-06-15"

curl -X POST localhost:3000/bookings -H 'content-type: application/json' -d '{
  "resourceId":"res_alex","serviceId":"svc_haircut",
  "startsAt":"2026-06-15T09:00:00Z",
  "customerName":"Jane","customerEmail":"jane@example.com"
}'
```

### Docker

```bash
docker build -t slotpilot . && docker run -p 3000:3000 slotpilot
```

## Quality gates

```bash
npm run typecheck   # tsc --strict, no emit
npm test            # jest unit + e2e
npm run build       # compile to dist/
```

## Frontend (`web/`)

A **Next.js 14** (App Router, TypeScript, Tailwind) public booking page with a
warm editorial design — a four-step flow (service → day → time → details) that
reads live availability and confirms instantly against this API.

```bash
# 1. start the API (CORS is enabled for the frontend origin)
npm run build && PORT=3000 npm start

# 2. start the web app
cd web && npm install
NEXT_PUBLIC_API_URL=http://localhost:3000 npm run dev   # http://localhost:3001
```

Set `CORS_ORIGINS` on the API (comma-separated) to lock down allowed origins in
production; it defaults to `*`.

## Roadmap

- [ ] TypeORM + Postgres adapter with `tstzrange` exclusion constraint
- [ ] Organization timezone handling at the API edge (store UTC)
- [ ] Notifications (email / SMS / Telegram) on booking & reminders
- [ ] Online payment (deposit) on booking confirmation
- [x] Next.js public booking page (`web/`)

## License

MIT
