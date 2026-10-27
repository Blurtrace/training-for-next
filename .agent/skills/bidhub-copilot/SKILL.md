
---
name: bidhub-copilot
description: Copilot for building BidHub, a fullstack online auction platform (Next.js 16 App Router, TypeScript, TypeORM, PostgreSQL on Supabase, Vercel). Use whenever the user works on any BidHub module (data model, auth, categories, auctions, bids, admin, frontend, i18n, Docker, CI, README, deployment) or asks to generate, review or explain BidHub code. The skill generates the code AND a Markdown explanation per module, working one module at a time.
---

# BidHub Copilot

You are a copilot: you **generate the code** and you **explain every step in a Markdown file**. The user works module by module. Never jump ahead to a later module without being asked.

## Language

Everything is in **English**: identifiers, comments, commit messages, error messages in the API, and the explanation MDs. UI strings live in i18n dictionaries (es/en).

## Fixed stack (do not change without asking)

| Area | Decision |
|---|---|
| Framework | Next.js 16, App Router, TypeScript strict, **no `any`** |
| Package manager | npm |
| ORM / DB | TypeORM + PostgreSQL (Supabase), migrations + seed |
| Validation | Zod (shared schemas, inferred types) |
| Auth | `jsonwebtoken` + `bcrypt`; access + refresh tokens in `httpOnly`, `secure`, `sameSite` cookies |
| Styling | Tailwind CSS + shadcn/ui |
| i18n | i18next / react-i18next (es + en, visible language switcher) |
| Tests | Jest |
| Concurrency | Optimistic locking with a `version` column |
| Extras (all required) | Rate limiting (login + bids), bid idempotency, concurrency handling, docker-compose, CI pipeline, business-rule tests |

## Module order

Work in this order, one module per session/step:

1. **Data model** (entities, relations, migrations, seed)
2. **Backend**, in this hierarchy: Auth → Categories → Auctions → Bids → Admin moderation
3. **Frontend** (route groups, layouts, pages, i18n, countdown)
4. **Docker, docker-compose, CI, README, deployment** (and any remaining extras)

At the start of each module, state: goal, files that will be created/changed, and the business rules (numbered as in the spec) it covers. Then generate the code. Then write the module MD.

## Required deliverable per module: explanation MD

For every module, create `docs/modules/NN-module-name.md` with:

1. **Goal** and the business rules / technical requirements covered (by number).
2. **Files created or modified** (tree).
3. **Step-by-step walkthrough**: for each file, explain the code **line by line** (or by tight line groups) — what it does and *why*, including the framework/ORM concept behind it.
4. **Decisions and trade-offs** (what was chosen, what was discarded, why).
5. **How to verify** (exact commands, sample requests, expected status codes).
6. **Common mistakes / exam traps** for this module.

Be thorough: the user is learning from these files and must be able to defend the code in the evaluation.

## Architecture conventions

- Layers: `route handler` (HTTP only) → `service` (business rules) → `repository`/TypeORM. Handlers never contain business logic.
- One responsibility per function; no duplicated logic; small pure functions for rules so they are unit-testable.
- Consistent error format for every handler, e.g. `{ "error": { "code": "BID_TOO_LOW", "message": "...", "details": [...] } }`, via a single `AppError` hierarchy and one `handleError` helper.
- HTTP codes: 400 validation, 401 no/invalid session, 403 forbidden, 404 not found, 409 conflict (rule violations, version conflicts).
- Every handler validates input with Zod and validates session + role **itself** (never rely only on `proxy.ts`).
- Suggested folders: `src/app` (routes), `src/modules/<domain>` (schemas, service, repository), `src/lib` (db, auth, errors, rate-limit), `src/components`, `src/i18n`, `src/entities`, `src/migrations`, `src/seed`.

## Key technical decisions to apply

**Auction status (rules 12–13)**: never rely on cron jobs. Store `cancelledAt` and `cancelReason` plus `closesAt`; derive status in one pure function `getAuctionStatus(auction, bidCount, now)`: cancelled → `cancelledAt` set; active → `now < closesAt`; closed → ended with bids; deserted → ended without bids. Status filters in the catalog must be translated to equivalent SQL conditions. Use the DB/server time as the single source of truth.

**Bids (rules 6–11)**: one service method inside a transaction that checks: not own auction, active, amount ≥ start price (first bid) or ≥ highest + minimum increment, bidder is not already the highest bidder, then inserts the bid and applies the 5-minute anti-sniping extension (if ≤ 5 min remain, `closesAt += 5 min`). Bids are immutable (no update/delete endpoints).

**Concurrency (optimistic locking)**: `@VersionColumn` on `Auction`. Updating the auction row (current highest bid / `closesAt`) inside the same transaction bumps the version; a stale write fails and returns **409** so the client can retry. Add a test that fires two simultaneous bids.

**Idempotency**: `Idempotency-Key` header on `POST` bid; store key + user + response hash in a table with a unique constraint; a repeated key returns the original result instead of creating a second bid.

**Rate limiting**: login and bids. Serverless on Vercel makes in-memory counters unreliable, so **ask the user** at that module whether to use a DB-backed counter table or an external store (e.g. Upstash) and explain the trade-off.

**Auth**: bcrypt-hashed passwords; refresh token stored **hashed** in DB; logout revokes it; refresh rotates tokens; endpoints: register, login, refresh, logout. Tokens are never returned in the body nor stored in `localStorage`. `proxy.ts` runs in the Node.js runtime in Next.js 16, so `jsonwebtoken` can be used there; keep it limited to redirecting guests to login and blocking non-admins from admin routes.

**Privacy (rules 18–20)**: public bid history returns amount, date and a partial alias (e.g. `jo***`), never full name or email. Ownership checks on every edit/cancel (ignore the id in the URL as proof of ownership). Full "my bids" list only for the authenticated user.

**Categories (rule 17)**: deleting a category with auctions returns 409.

**Rendering strategy**: Home = SSG, catalog and auction detail = SSR (`dynamic`/no cache), dashboard = CSR (client fetching).

**Hydration**: the countdown is a client component that renders a stable placeholder on the server and starts ticking only after mount (`useEffect`), derived from the `closesAt` ISO string, to avoid hydration mismatches.

**Images**: `next/image`; configure `remotePatterns` for the image host.

**Seed**: admin user, categories, and auctions in all four states (active, closed, deserted, cancelled) with realistic bids.

## Final deliverables checklist

- `.env.example` with every variable, no secrets (DB URL, JWT secrets, token expirations, etc.).
- `Dockerfile` (multi-stage, builds and runs the app) and `docker-compose.yml`.
- CI pipeline (lint, type-check, tests, build).
- Jest tests covering at least bid rules (first bid, increment, own auction, highest bidder, anti-sniping, closed auction).
- **README** that lets a stranger run everything from scratch: description, prerequisites, env var explanations (how to get the Supabase connection string, how to generate secrets), dev steps in order (clone/install → `.env` → migrations → seed → dev), admin credentials, Docker commands, scripts table, folder structure, deployed URL. Verify every command actually works.
- Deployment: Supabase (DB) + Vercel (app); confirm route handlers, dynamic routes and auth work in production (cookies `secure`, env vars set in Vercel).

## Working rules

- Generate complete, runnable code for the current module only.
- If a requirement is ambiguous or a choice is not fixed above, ask one short question before coding.
- After generating a module, give a brief summary in chat and point to the module MD; do not paste the full explanation twice.
- Flag any place where the generated code could break a business rule or the "no `any`" rule.