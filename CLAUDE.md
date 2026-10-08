# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
EventHub is a full-stack event ticket booking platform built for QA training. Users can browse events, book tickets, manage bookings, and create events. Each user operates in an isolated sandbox.

## Tech Stack
- **Frontend**: Next.js 14 (App Router), React 18, TypeScript (mixed with `.jsx`), Tailwind CSS, React Query v5, Axios
- **Backend**: Express.js, Prisma 5 ORM, MySQL 8+, express-validator, Swagger UI at `/api/docs`
- **Auth**: JWT (7-day expiry), bcryptjs
- **Testing**: Playwright E2E (Chromium only)

## Commands
```bash
npm run setup        # npm install in backend/ and frontend/ (root has only Playwright + concurrently)
npm run db:push      # Push Prisma schema to MySQL (non-interactive); `npm run migrate` is interactive
npm run seed         # Seed 10 static events
npm run dev          # Frontend (3000) + backend (3001) concurrently
npm run build        # Build Next.js frontend
npm run lint         # next lint (frontend only; backend has no linter)

npm run test         # Run all Playwright tests
npm run test:ui      # Playwright UI mode
npm run test:report  # Open last HTML report
npx playwright test tests/<file>.spec.js --reporter=line        # Single file
npx playwright test tests/<file>.spec.js -g "<test name>"       # Single test
docker compose up --build   # Run the suite in the official Playwright image
```

Env setup: copy `backend/.env.example` → `backend/.env` (needs `DATABASE_URL`, `JWT_SECRET`, `CORS_ORIGIN`) and `frontend/.env.local.example` → `frontend/.env.local` (`NEXT_PUBLIC_API_URL=http://localhost:3001/api`).

## Important: Tests target the live site
`playwright.config.ts` sets `baseURL` to `https://eventhub.rahulshettyacademy.com`, and existing specs also hardcode `BASE_URL` to that host. Tests do **not** run against the local dev server unless you change this. The `playwright.yml` workflow runs the suite against production on every push to `main`.

## Backend Architecture
Layered: `routes → controllers → services → repositories → Prisma`. Entry is `backend/server.js` (DB connect, shutdown) → `backend/app.js` (middleware, Swagger, route mounting, `/api/health`, `/api/config` feature flags).

- **Auth**: `/api/auth` is public; `/api/events` and `/api/bookings` require `Authorization: Bearer <jwt>` (`middleware/authMiddleware.js` sets `req.user`). The frontend stores the token in `localStorage` key `eventhub_token` (`frontend/lib/hooks/useAuth.tsx`).
- **Per-user sandbox**: implemented in the repository layer, not middleware. Event queries filter with `OR: [{ isStatic: true }, { userId }]`, so each user sees seeded events plus only their own. Bookings are filtered by `userId`; services throw `ForbiddenError` on cross-user access.
- **Errors**: services throw domain errors from `src/utils/errors.js` (`NotFoundError`, `ValidationError`, `InsufficientSeatsError`, `ForbiddenError`, each with `statusCode`); `middleware/errorHandler.js` maps them to `{ success: false, error }` responses.
- **Limits / FIFO pruning**: `MAX_USER_DYNAMIC_EVENTS = 6` in `eventService.js`, `MAX_USER_BOOKINGS = 9` in `bookingService.js`. Booking pruning prefers deleting the oldest booking for a *different* event.
- **Data model** (`backend/prisma/schema.prisma`): `User`, `Event` (`isStatic`, nullable `userId`, `availableSeats`), `Booking` (`bookingRef` unique). Deletes cascade.

## Frontend Notes
- API calls go through `frontend/lib/api/` (Axios client) and React Query hooks in `frontend/lib/hooks/`. Both `.js` and `.ts` versions of some API modules exist (e.g. `client.js`/`client.ts`); check which one the importing file resolves before editing.
- Pages: `app/login`, `app/register`, `app/events`, `app/bookings`, `app/admin`.

## Testing Conventions
- Test files go in `tests/` as `<feature-name>.spec.js`
- Follow `.claude/skills/playwright-best-practices/SKILL.md`
- Locator priority: data-testid > role > label/placeholder > ID > CSS class
- No `page.waitForTimeout()` — use `expect().toBeVisible()`
- Tests must be self-contained (login → action → assert); `fullyParallel: false`, `retries: 0`
- Test account: `rahulshetty1@gmail.com` / `Magiclife1!`
- Because of FIFO pruning (6 events / 9 bookings per user), tests must not assume earlier-created data still exists.

## Key Business Rules
- Max 6 user-created events (FIFO pruning on overflow)
- Max 9 bookings per user (FIFO pruning on overflow)
- Booking ref format `<X>-XXXXXX`, where `<X>` = event title's first character (uppercase). The README's `EVT-XXXXXX` is outdated.
- Seat count reduces on booking, restores on cancellation
- Refund eligibility: 1 ticket = eligible, >1 ticket = not eligible (client-side only)
- Cross-user booking access returns "Access Denied"
- Static events (seeded) are immutable (`ForbiddenError` on update/delete)

## Skills (`.claude/skills/`)
Agent-style skills used for the QA workflow:
- `generate-tests` — generates Playwright tests for a feature
- `review-tests` — reviews test code quality
- `create-scenarios` — writes test scenario documents (output in `docs/test-scenarios.md`)
- `test-strategy` — assigns scenarios to test pyramid layers (output in `docs/test-strategy.md`)
- `playwright-best-practices` — Playwright standards reference
- `eventhub-domain` — business rules, API reference, UI selectors, user flows (more current than the README's selector/API tables)

## CI/CD (`.github/workflows/`)
- `ci.yml` — PR checks: backend syntax check, Prisma validate/format/generate, schema drift check against the production DB, frontend type check + production build
- `playwright.yml` — E2E against production on push to `main`
- `deploy.yml` — SSH deploy to production with PM2; schema changes must be applied manually on the server

## Code Style
- Backend: CommonJS JavaScript with JSDoc; routes carry Swagger JSDoc annotations
- Frontend: TypeScript, React hooks, Tailwind utility classes
- Tests: JavaScript (ESM `import`) with Playwright test runner; add step comments
