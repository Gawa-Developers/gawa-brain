# 2026-06-26 — Travel It All Phase 1 Delivery

## Summary

Completed Phase 1 of the travel-it-all white-label booking engine. All Phase 1 items shipped in `patch/v1.0.0` → PR #1 → `develop`.

## What Shipped

- **Landing page CMS** — admin settings form now has a Landing Page section: hero background image upload, heading, subheading, CTA text, about title + body. All stored on the `Tenant` model. Homepage reads from DB with fallbacks.
- **Public packages pages** — `/packages` listing and `/packages/[slug]` detail, matching tours pages.
- **Availability enforcement** — checkout checks `TourAvailability { tourId, date, maxPax, bookedPax }` inside a Prisma `$transaction`. Returns `409` with remaining spots if the date is full. Atomically increments `bookedPax` on success.
- **My Bookings** — `/my-bookings` for authenticated travelers. Shows all bookings with status badges and voucher download links. Nav shows "My Bookings" / "Sign In" based on session.
- **Packages in checkout** — `/api/checkout` now handles both `tourId` and `packageId` booking paths.

## Engineering Discoveries (promoted to gawa-brain/engineering.md)

### Prisma v7
- `url` removed from `datasource db {}` in schema — fails with clear error if left
- `prisma.config.ts` handles CLI config; runtime uses `PrismaPg` adapter
- `import 'dotenv/config'` only reads `.env` — use `config({ path: '.env.local' })` for local dev

### Next.js 16
- `middleware.ts` deprecated → `src/proxy.ts` exporting `proxy` function
- Route groups strip prefix from URLs — `(admin)/tours/[id]` collides with `(booking)/tours/[slug]`; fix: nest admin pages under `(admin)/admin/[route]`

### Zod v4 + React Hook Form
- `z.coerce.number()` infers input as `unknown`, breaks resolver types — use `z.number()` + `{ valueAsNumber: true }` in `register()`

### node-vibrant v4
- Named import only: `import { Vibrant } from 'node-vibrant/node'`

### BullMQ
- `connection` must be plain `{ host, port }` — not an IORedis instance

### Availability enforcement
- Check + increment `bookedPax` in a single `$transaction` — separate queries create a race that allows overselling

## Stack Confirmed (Phase 0 + Phase 1)

- Next.js 16.2.9 App Router
- Prisma v7 + PostgreSQL + `@prisma/adapter-pg`
- NextAuth v5 beta (JWT strategy)
- BullMQ + Redis
- node-vibrant v4 for logo palette extraction
- Local filesystem storage for Phase 0 (`LocalStorageProvider`); R2-ready interface for Phase 1+
- PayMongo via headless `PaymentGateway` interface

## Branch/Release Pattern for This Project

- Branch type: `patch/v*.*.*` (not `hotfix/`)
- Base branch: `develop` (not `main`)
- Release: `patch/v*.*.* → PR → develop → merge → tag`
