# Desi Market — bhiahvsh Backend (Step 2)

Node.js + Express + PostgreSQL backend implementing the API structure and
database design from the project spec (sections 2

## 1. Requirements

- Node.js 18+
- A PostgreSQL server (local install, Docker, or a hosted instance like
  Supabase/Neon/Railway — anything that gives you a `DATABASE_URL`)

## 2. Setup

```bash
cd desi-market-backend
cp .env.example .env
# edit .env and set DATABASE_URL to your Postgres connection string

npm install
npm run db:setup   # creates all tables (schema.sql) and inserts seed data (seed.sql)
npm run dev        # starts the server with auto-reload on http://localhost:4000
```

If you don't have Postgres running locally, the fastest way to get one is:

```bash
docker run --name desi-market-db -e POSTGRES_USER=desi_market \
  -e POSTGRES_PASSWORD=desi_market -e POSTGRES_DB=desi_market \
  -p 5432:5432 -d postgres:16
```

Then the default `DATABASE_URL` in `.env.example` will work as-is.

## 3. What's included

| Area | File |
|---|---|
| DB schema | `database/schema.sql` |
| Seed data (mirrors the frontend's mock shops/products/workers) | `database/seed.sql` |
| DB connection pool | `src/db.js` |
| JWT auth + role guard | `src/middleware/auth.js` |
| Mock OTP service | `src/utils/otp.js` |
| Routes | `src/routes/*.js` — one file per resource, matching spec section 26 |
| Server entry | `src/server.js` |

## 4. OTP is mocked — read this before demoing

There's no real SMS gateway wired in yet. `POST /api/auth/request-otp`
generates a 4-digit OTP, logs it to the server console, and — **only when
`NODE_ENV` is not `production`** — also returns it in the response body as
`dev_otp` so you can test end-to-end without an SMS provider.

Before a real pilot, replace the body of `sendOtp()` in `src/utils/otp.js`
with a call to an SMS gateway (MSG91, Twilio, or Gupshup are common choices
for Indian numbers — this lines up with the "OTP/Payment/Maps" budget line
in the project's cost plan), and stop returning `dev_otp`.

## 5. Try it with curl

```bash
# 1) Request an OTP (dev_otp appears in the response — copy it)
curl -X POST http://localhost:4000/api/auth/request-otp \
  -H "Content-Type: application/json" \
  -d '{"mobile":"9876543210"}'

# 2) Verify it, get a JWT back
curl -X POST http://localhost:4000/api/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"mobile":"9876543210","otp":"<paste dev_otp here>","name":"Test Customer"}'

# 3) Use the token for authenticated calls
TOKEN="<paste token from step 2>"

curl http://localhost:4000/api/sellers
curl http://localhost:4000/api/products?category=grocery
curl "http://localhost:4000/api/providers?category=plumber&lat=26.12&lng=85.39"

curl -X POST http://localhost:4000/api/orders \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"seller_id":1,"delivery_address":"मेरा घर, बिथौली","items":[{"product_id":1,"quantity":2}]}'

curl -X POST http://localhost:4000/api/bookings \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"provider_id":1,"scheduled_date":"2026-09-10","scheduled_slot":"9 - 11 AM","problem_description":"नल टपक रहा है"}'
```

## 6. Notable design choices (and what to upgrade before scaling)

- **OTP store is in-memory** (`src/utils/otp.js`) — fine for a single-server
  pilot, but won't survive a restart or work across multiple instances.
  Move to Redis or a DB table before running more than one server process.
- **Distance ranking uses the Haversine formula directly in SQL**
  (`src/routes/providers.js`) rather than PostGIS — good enough for a few
  thousand rows per panchayat; add PostGIS (`ST_DWithin`) once you need
  proper spatial indexes at scale.
- **Payments are record-keeping only** (`src/routes/payments.js`) — no real
  payment gateway is integrated. Wire one in and only mark a payment
  `paid` after the gateway confirms, not before.
- **Commission percentages are hardcoded** at 5% (product) / 10% (service)
  matching the illustrative example in the spec — move these to a config
  table once real pilot economics are known.
- Raw parameterized SQL is used throughout (no ORM) to keep the codebase
  small and easy to read for an MVP; introduce Prisma/Knex later if the
  schema grows a lot.

## 7. What's next (Step 3 / Step 4 per your 90-day plan)

- Step 3: wire the frontend prototype to these endpoints (replace mock
  data with real `fetch()` calls), real OTP login, live orders/bookings.
- Step 4: Seller Panel, Worker Panel, Admin Panel, Delivery Panel, and the
  Panchayat Dashboard UI (the backend for panchayat stats already exists
  at `GET /api/panchayats/:id/stats`, and admin metrics at
  `GET /api/admin/dashboard`).
