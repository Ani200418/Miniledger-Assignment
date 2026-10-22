# MiniLedger

A mobile app (Expo / React Native) for tracking money you have lent to friends and who has repaid. Phone-number login with a **mock OTP**, loan tracking, overdue highlighting and a summary dashboard.

## Features
- Phone + 6-digit OTP login (OTP printed in the server log, expires, single use, locked after 5 wrong tries); JWT session
- Add loans (friend, amount, due date), list/filter (All / Pending / Repaid / Overdue), mark repaid (with confirmation)
- Dashboard summary: Total Lent, Total Repaid, Overdue Amount (₹, `en-IN`)
- Loading, empty and error states, pull-to-refresh, logout; works on Android, iOS and in the browser
- Every loan is scoped to its owner (other users get 404)

## Stack
Backend: Node.js, TypeScript, Express, PostgreSQL (`pg`, parameterised SQL), JWT, Zod, Vitest + Supertest.
Mobile: Expo SDK 52, React Native, TypeScript, React Navigation, Axios, expo-secure-store.

## Structure
```
backend/  src/{config,middleware,routes,services,repositories,validators,scripts}, sql/schema.sql, tests/
mobile/   App.tsx, src/{api,components,hooks,screens,types}
.replit, package.json   cloud run scripts
```

## Run in the cloud (no installs on your machine) — Replit
1. Push this repo to GitHub, then in Replit choose **Import from GitHub**.
2. Add a PostgreSQL database (Replit "Database"/PostgreSQL tool, or a free Neon/Supabase database) and set **Secrets**:
   `DATABASE_URL` (connection string), `JWT_SECRET` (long random string), and `DATABASE_SSL=true` if the provider requires SSL.
3. Press **Run** (`npm run cloud:start`): it builds the Expo **web** app, builds the API and starts one server on port 4000 that serves both the API and the web app. Open the Replit web view/URL — the app is at `/`, health check at `/health`.
4. Because the web app and API share one origin, **no `EXPO_PUBLIC_API_URL` is needed**.
5. Login: enter a 10-digit number, then read the OTP from the Replit **Console** (`[DEV OTP] +91… -> 123456`).

Other hosts (Render, Railway, Fly) work the same way: build command `npm run cloud:build`, start command `cd backend && npm start`, set the same env vars.
Native testing (Expo Go on a phone) needs a device; use the browser version if you have none.

## Run locally (alternative)
```bash
# Postgres: docker compose up -d   (or any Postgres; then npm run db:init in backend/)
cd backend && cp .env.example .env && npm install && npm run dev      # API on :4000, schema applied on start
cd mobile  && npm install && npx expo start                           # press w for web, or scan the QR in Expo Go
```
Mobile → API URL resolution (`mobile/src/api/client.ts`): `EXPO_PUBLIC_API_URL` if set → same origin on web → Expo host on port 4000 for native dev. If the API and app run in different hosted places, set `EXPO_PUBLIC_API_URL` to the backend's **public https URL** (never `localhost`, never a private internal URL) and restart Expo. The login screen performs a real `GET /health` and shows the URL it tried if it fails.

## Environment variables (`backend/.env`)
| Name | Description |
|---|---|
| `DATABASE_URL` | Postgres connection string (required) |
| `DATABASE_SSL` | `true` for managed DBs needing SSL |
| `JWT_SECRET` | JWT signing secret (required) |
| `JWT_EXPIRES_IN` | e.g. `7d` |
| `OTP_TTL_MINUTES` | OTP validity, default 5 |
| `PORT` | default 4000 |
| `WEB_DIR` | optional path of the Expo web build to serve (default `mobile/dist`) |

Mobile: only the optional, non-secret `EXPO_PUBLIC_API_URL`.

## Mock OTP
`POST /api/auth/send-otp` creates a 6-digit OTP held in server memory, logs `[DEV OTP] +91XXXXXXXXXX -> 123456` and **never returns it**. It expires after `OTP_TTL_MINUTES`, works once, and is discarded after 5 wrong attempts. Production would need a real SMS provider, persistent storage and rate limiting per phone/IP.

## API
`/api/loans*` require `Authorization: Bearer <token>`. Errors are `{"error":"message"}`.

| Endpoint | Request | Response |
|---|---|---|
| `POST /api/auth/send-otp` | `{"phone":"9876543210"}` | 200 `{message}` |
| `POST /api/auth/verify-otp` | `{"phone":"9876543210","otp":"123456"}` | 200 `{token,user}` · 401 bad/expired/used OTP |
| `POST /api/loans` | `{"friendName":"Rahul","amount":1500,"dueDate":"2026-12-01"}` | 201 loan · 400 invalid |
| `GET /api/loans?status=all\|pending\|repaid\|overdue` | | 200 `[loan]` |
| `GET /api/loans/summary` | | 200 `{"totalLent":1500,"totalRepaid":0,"overdueAmount":1500}` |
| `GET /api/loans/:id` | | 200 loan · 400 bad id · 404 |
| `PATCH /api/loans/:id/repay` | | 200 loan · 404 · 409 already repaid |
| `GET /health` | | 200 `{"status":"ok"}` |

Loan: `{id, friendName, amount, dueDate:"YYYY-MM-DD", isRepaid, createdAt, repaidAt, receiptUrl, overdue}`.
"Overdue" = unpaid and due date before today in India time (`Asia/Kolkata`). Amounts are `NUMERIC(12,2)` in the DB.

## Tests
```bash
cd backend
npm test            # 15 tests; boots a throw-away embedded PostgreSQL automatically (or set TEST_DATABASE_URL)
npm run typecheck   # backend      |   cd ../mobile && npm run typecheck
npm run build       # backend      |   cd ../mobile && npm run build:web
```
Covered: phone validation; wrong / expired / reused / brute-forced OTPs; login; missing and invalid tokens; loan validation; create, list, filter, get, repay, repeat-repay (409); cross-user isolation (get, repay, list, summary); summary and overdue maths including `0.1 + 0.2 = 0.3` exactness.

Last run (in the build environment): 15/15 passed on PostgreSQL 18; backend and mobile type-check clean; web export and backend build succeed; manual smoke test of `/health`, web app serving, OTP → login → create loan → summary over HTTP succeeded.

## Known limitations
- Receipt upload (optional) is not implemented.
- The native iOS/Android UI and the browser UI were not exercised on a device/simulator in the build environment (only type-checked and bundled); the cloud deployment config (`.replit`) has not been run on Replit.
- Mock OTP is in-memory (lost on restart, single instance only); no per-IP rate limiting.
- On web the token is kept in `localStorage` (SecureStore is native-only).
- Mobile has no automated UI tests.
# Miniledger-Assignment
