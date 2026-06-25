# Engineering — Gawa Developers

General engineering patterns and best practices applicable across all Gawa projects.

---

## API Design

### Headless Interface Pattern

Never hardcode a vendor SDK into business logic. Always define an interface and depend on it — not on the implementation.

```typescript
// Define the interface first
interface PaymentGateway {
  createIntent(params: CreatePaymentParams): Promise<PaymentIntent>
  parseWebhook(rawBody: Buffer, signature: string): Promise<WebhookEvent>
}

// Implement per provider
class PayMongoGateway implements PaymentGateway { ... }
class PayMayaGateway implements PaymentGateway { ... }

// Resolve at runtime — business logic never changes when providers change
function resolveGateway(config: TenantConfig, method: PaymentMethod): PaymentGateway
```

Apply this pattern to: payment gateways, email providers, storage providers, SMS providers, search engines.

### Validation at System Boundaries

Validate all external input with Zod (or equivalent schema library). Never trust `req.body` directly.

```typescript
const schema = z.object({ amount: z.number().positive(), method: PaymentMethodSchema })
const body = schema.parse(await request.json()) // throws on invalid input
```

Internal code, framework guarantees, and DB output do not need re-validation.

---

## Webhooks

**Critical pattern — apply to every webhook handler:**

1. Read raw body as `Buffer` BEFORE any JSON parsing:
   ```typescript
   const rawBody = Buffer.from(await request.arrayBuffer())
   ```
2. Verify HMAC-SHA256 signature against raw body. Throw if invalid. Never process an unverified webhook.
3. Parse JSON only after verification.
4. Write to an idempotency table (`WebhookEvent { gatewayIntentId, processedAt }`) — skip already-processed events.
5. Enqueue processing via a job queue. Return `200` immediately. Never process synchronously in the handler.

Calling `request.json()` before signature verification corrupts the raw body — this is a common mistake that breaks HMAC validation.

---

## Async Jobs

Never run these synchronously in a web request handler:
- PDF / document generation (CPU-intensive, 1–5s)
- Transactional email sending
- Payment webhook processing
- External API calls with unpredictable latency
- File processing after upload

Use BullMQ (or equivalent job queue) for all of the above. Return a `202 Accepted` or redirect immediately, process in a worker.

---

## File Uploads

Always use signed URL direct-to-storage upload (S3/R2/GCS). Never proxy file bytes through the application server.

Pattern:
1. Client requests a signed URL from `/api/uploads/sign`
2. Client PUTs directly to storage using the signed URL
3. Client notifies the server the upload is complete
4. **Server validates before committing URL to DB**: MIME type whitelist, max file size, malware scan hook

Skipping step 4 allows arbitrary file upload — including executables, oversized files, and malicious content.

---

## Multi-Tenancy

### Isolation hierarchy (in order of trust)

1. **Database-level RLS** — primary guard. Enforced regardless of application code. Must be tested independently with no application code in the path.
2. **ORM middleware tenant injection** — convenience layer for query efficiency. Not a security boundary. Bypassable by raw queries, workers, and webhook handlers.
3. **Application-level scoping** — defense in depth. Always pass `tenantId` explicitly in job payloads and webhook contexts.

### PgBouncer + RLS

RLS policies that use `SET LOCAL` / `current_setting('app.tenant_id')` are **incompatible with PgBouncer transaction-mode pooling**. Session variables bleed or disappear across reused connections. Always use **session-mode pooling** when RLS relies on session variables.

### Admin tenant scoping

Admin routes must scope from `session.user.tenantId` (from the auth session), NOT from the request hostname. Shared admin subdomains fail hostname-based tenant resolution.

---

## Database

### Prisma v7 adapter pattern

Prisma v7 removed `url` from the `datasource` block in `schema.prisma`. The DB URL now lives in `prisma.config.ts` for CLI use and in an explicit adapter for the runtime client.

**`prisma.config.ts` (CLI only):**
```typescript
import { config } from 'dotenv'
import { defineConfig } from 'prisma/config'
config({ path: '.env.local' })   // must specify .env.local explicitly — 'dotenv/config' only reads .env
export default defineConfig({
  schema: 'prisma/schema.prisma',
  datasource: { url: process.env.DATABASE_URL },
})
```

**`src/lib/db.ts` (runtime):**
```typescript
import { PrismaClient } from '@prisma/client'
import { PrismaPg } from '@prisma/adapter-pg'
import { Pool } from 'pg'

function createPrismaClient() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL })
  const adapter = new PrismaPg(pool)
  return new PrismaClient({ adapter })
}
```

**Common pitfalls:**
- `datasource db { url = env("DATABASE_URL") }` in `schema.prisma` will throw — remove the `url` line entirely from the schema
- `import 'dotenv/config'` in `prisma.config.ts` only reads `.env`, not `.env.local` — use `config({ path: '.env.local' })` instead
- `new PrismaClient({ datasourceUrl })` no longer works — use the adapter pattern above

### Prisma connection pooling

- `connection_limit=1` is for serverless/edge functions (new process per request). Do NOT use in a long-running container.
- For a containerized app, set `connection_limit` to 10–25. Use PgBouncer in front.
- Prisma + PgBouncer: use session-mode if RLS session variables are required; transaction-mode otherwise.

### Migrations

- Use expand/contract pattern for large table migrations to avoid locking.
- Never run a migration that locks a high-traffic table during a production deployment without a maintenance window.

---

## Security Baseline

- **CSP headers**: configure via framework/middleware. Never rely on the browser default.
- **Rate limiting**: apply to all payment, auth, and high-value mutation routes.
- **No secrets in source**: all secrets via environment variables. `.env` in `.gitignore`. Use Doppler, Infisical, or equivalent for secret management.
- **Signed cookies / JWT**: set `httpOnly`, `secure`, `sameSite=lax` minimum. Never store JWTs in localStorage.
- **CORS**: whitelist explicit origins. Never `Access-Control-Allow-Origin: *` on authenticated routes.

---

## Next.js 16+

### `proxy.ts` replaces `middleware.ts`

Next.js 16 deprecated `middleware.ts`. The equivalent is `src/proxy.ts` exporting a `proxy` function (not `middleware`):

```typescript
// src/proxy.ts
export function proxy(request: NextRequest) { /* tenant resolution etc. */ }
export const config = { matcher: ['/((?!_next/static|...).*)'] }
```

### Route group collision

Route groups `(name)` strip their prefix from URLs. `(booking)/tours/[slug]` and `(admin)/tours/[id]` both resolve to `/tours/*` — causing an ambiguous route error.

**Fix:** prefix admin pages with an explicit `admin/` segment inside the group:
```
src/app/
  (booking)/tours/[slug]/   → resolves to /tours/[slug]
  (admin)/admin/tours/[id]/ → resolves to /admin/tours/[id]  ✓ no collision
```

Never rely on the group name `(admin)` to scope the URL — it doesn't.

---

## Zod + React Hook Form

### `z.coerce.number()` breaks resolver types in Zod v4

`z.coerce.number()` infers its input type as `unknown` in Zod v4, which is incompatible with react-hook-form's `Resolver` generic. TypeScript will error on the schema.

**Fix:** use `z.number()` with `{ valueAsNumber: true }` in the `register()` call instead:
```typescript
// Schema
const schema = z.object({ price: z.number().positive() })

// Component
<input type="number" {...register('price', { valueAsNumber: true })} />
```

---

## Third-Party Libraries

### node-vibrant v4 — named import from node subpath

Default import no longer exists in v4. Use the named export from the node-specific subpath:

```typescript
import { Vibrant } from 'node-vibrant/node'
const palette = await Vibrant.from(imageBuffer).getPalette()
```

### BullMQ — pass plain connection object, not an IORedis instance

BullMQ's `connection` option expects `{ host, port }` (or a `ConnectionOptions` object), not an `IORedis` instance. Passing an IORedis instance causes a TypeScript error and runtime failure:

```typescript
// ✓ correct
new Queue('pdf', { connection: { host: 'localhost', port: 6379 } })

// ✗ wrong
const redis = new Redis(process.env.REDIS_URL)
new Queue('pdf', { connection: redis })
```

---

## Availability Enforcement (Transactional Inventory)

For any time-slot or capacity-limited booking, enforce availability atomically inside a DB transaction:

```typescript
const booking = await db.$transaction(async (tx) => {
  const avail = await tx.tourAvailability.findUnique({
    where: { tourId_date: { tourId, date: bookingDate } },
  })
  if (avail && (avail.maxPax - avail.bookedPax) < pax) {
    throw Object.assign(new Error('Not enough availability'), { code: 'AVAILABILITY_EXCEEDED' })
  }
  if (avail) {
    await tx.tourAvailability.update({
      where: { tourId_date: { tourId, date: bookingDate } },
      data: { bookedPax: { increment: pax } },
    })
  }
  return tx.booking.create({ data: { ... } })
})
```

Catch the thrown error outside the transaction and return a `409 Conflict` response. The check and increment must be in the same transaction — doing them in separate queries creates a race condition that allows overselling.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why | Do instead |
|---|---|---|
| Vendor SDK in business logic | Locks to one provider, untestable | Headless interface pattern |
| `request.json()` before webhook signature verify | Breaks HMAC, security gap | Read raw `Buffer` first |
| `connection_limit=1` in a container | Serializes all DB access | Size pool to concurrency |
| Synchronous PDF/email in request handler | Blocks event loop | BullMQ worker |
| Prisma middleware as sole tenant guard | Bypassable by raw queries | RLS + explicit scoping |
| Committing secrets to git | Irreversible leak | Env vars + secret manager |
| i18n as a retrofit | Requires extracting all strings | Scaffold i18n infrastructure early |
