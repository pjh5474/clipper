# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Clipper is a booking app for an at-home dog grooming service in Seoul (prices in KRW, stored as `priceCents`). It is a Next.js App Router codebase run through **vinext** (Vite-based Next.js reimplementation) and deployed as a **Cloudflare Worker** with a **D1** (SQLite) database accessed via **Drizzle ORM**.

## Commands

```sh
npm install
npm run db:migrate:local   # apply drizzle/ migrations to local D1 (first run only)
npm run db:seed:local      # wipe and reseed local D1 from scripts/seed.sql
npm run dev                # vinext dev server, http://localhost:3000
npm run build              # build Worker output into dist/
npm run start              # run built Worker with wrangler (dist/server/wrangler.json)
npm run cf-typegen         # regenerate worker-configuration.d.ts after changing bindings
npx tsc --noEmit           # type-check
```

There is no test runner or linter configured. Local D1 state lives in `.wrangler/` (gitignored); no Cloudflare account is needed locally.

Schema changes: edit `db/schema.ts`, then `npx drizzle-kit generate` to add a SQL migration under `drizzle/` (that is wrangler's `migrations_dir`), then `npm run db:migrate:local`. Keep `scripts/seed.sql` in sync with the schema.

## Architecture

- **Runtime/config**: `vite.config.ts` wires the `vinext` plugin with `@cloudflare/vite-plugin` (RSC environment with SSR child). `wrangler.jsonc` declares bindings: `DB` (D1), `ASSETS`, `IMAGES`. `next.config.ts` is effectively empty.
- **DB access**: bindings come from `import { env } from "cloudflare:workers"`, not `process.env`. `lib/clipper/db.ts#getDatabase()` returns the Drizzle client — use it rather than constructing `drizzle(env.DB)` inline (`app/api/booking-options/route.ts` currently duplicates this).
- **Data layer**: `lib/clipper/data.ts` holds all queries. Read functions map rows to `AppointmentDetails` (`lib/clipper/types.ts`), which converts timestamp columns to ISO strings so they can cross the server→client boundary. Timestamps are stored as `timestamp_ms` integers.
- **Groomer assignment** (`findAvailableGroomer`): a groomer qualifies if active, serves the neighborhood (`groomer_service_areas`), has an available `groomer_schedules` row for the weekday whose `start_time`/`end_time` ("HH:MM" strings) cover the slot, and has no overlapping non-cancelled appointment. Weekday and time-of-day are computed in **UTC**.
- **Routes**:
  - `/` → `components/clipper/home-variants.tsx` (marketing page) embedding the client-side `BookingFlow`.
  - `BookingFlow` (`components/clipper/booking-flow.tsx`, `"use client"`) fetches `GET /api/booking-options` and posts to `POST /api/bookings`. It accepts optional `availableSlots`/`slots` in the options response, but the API currently returns only `packages` and `neighborhoods`.
  - `POST /api/bookings` validates input, computes `endsAt` from package duration, assigns a groomer (409 if none), generates a `CLP-XXXXXX` booking reference, and returns `{ ok, reference }`.
  - `/appointments` and `/appointments/[reference]` are server components (`dynamic = "force-dynamic"`) that call the data layer directly.
- **UI**: shadcn/ui ("new-york" style) primitives in `components/ui/` re-exported from `components/ui/index.ts`; Tailwind v4 via `@tailwindcss/postcss` with styles in `app/globals.css`. Page components mostly use inline Tailwind with hard-coded hex colors. Path alias `@/*` maps to the repo root.

## Conventions

- Migrations are generated, never hand-edited: change db/schema.ts, then 'npx drizzle-kit generate'.
- No test runner exists. Don't invent one; use 'npx tsc --noEmit'.