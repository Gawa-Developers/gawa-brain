---
name: paymongo-integration
description: Use when integrating PayMongo (PH-native payment gateway) into any Gawa project — implementing checkout/payment creation, webhook handlers, or per-tenant/multi-company payment gateway credentials. Covers the Sources-API workaround for the broken Links redirect, PayMongo's signed-header webhook verification format, centavos conversion, idempotency, and the per-tenant credential pattern used in travel-it-all. Triggers on "PayMongo", "GCash", "payment webhook", "payment gateway", "linked account" (payment context), or work in src/lib/payments/.
---

Patterns already solved in `travel-it-all` (a Gawa whitelabel booking app). Re-derive nothing here from scratch — read this first, then adapt.

## Gateway abstraction

Define one interface, implement PayMongo (and any other provider) behind it — never call the PayMongo SDK/fetch directly from route handlers or business logic:

```typescript
interface PaymentGateway {
  createIntent(params: CreatePaymentParams): Promise<{ redirectUrl: string; gatewayIntentId: string }>
  parseWebhook(rawBody: Buffer, signature: string): Promise<WebhookEvent>
}
class PayMongoGateway implements PaymentGateway { /* ... */ }
```

This is the org-wide headless-interface pattern — see gawa-brain `memory/decisions.md` D1.

## Use the Sources API, not Payment Links or Payment Intents

`POST /v1/links` **silently drops the `redirect` parameter** — you cannot get PayMongo to bounce the user back to your success/failure URL through Links. Use `POST /v1/sources` instead: it supports `redirect: { success, failed }` explicitly.

- Only `gcash` is wired up as `SOURCE_TYPE` in the reference implementation. The `PaymentMethod` type may list `card | gcash | paymaya | grab_pay | brankas_bdo` — treat anything beyond gcash as aspirational/unimplemented unless you build it.
- **The two-step charge gotcha**: a Source becoming `chargeable` does NOT mean money has moved. You must call `POST /v1/payments` against the chargeable source yourself (`chargeSource()`) — normally triggered from the webhook handler once it sees `source.chargeable`. Treating `chargeable` as `paid` is the single most common mistake in a naive PayMongo integration.

## Amounts: PHP decimal in the DB, centavos only at the API boundary

Store `Decimal @db.Decimal(10,2)` in Prisma (or equivalent). Convert with `Math.round(amount * 100)` only when calling PayMongo, and divide by 100 only when parsing amounts back out of a webhook payload. Never persist centavos.

## Webhook signature verification

Route handler pattern (`src/app/api/checkout/webhook/route.ts`):

1. Read the raw body as a `Buffer` **before** any JSON parsing — parsing first corrupts the bytes the HMAC was computed over (see gawa-brain `memory/engineering.md` — Webhooks).
2. Header is `paymongo-signature` (lowercase), formatted `t=<timestamp>,te=<test-sig>,li=<live-sig>` — split on `,` then `=`.
3. Signed string is **`${timestamp}.${rawBodyString}`** — not the raw body alone. This matches PayMongo's own SDK (`WebhookService.constructEvent`), not the generic "HMAC over the body" pattern used elsewhere.
4. `crypto.createHmac('sha256', webhookSecret)` → hex digest. Compare with `crypto.timingSafeEqual`, guarding the length check first so it doesn't throw on a mismatched-length forged header.
5. Prefer the `li` (live) signature over `te` (test) when both are present in the header — matches PayMongo SDK behavior.
6. On any verification failure, throw and return `400`. Never process an unverified webhook body.

## Idempotency

Dedicated `WebhookEvent` table, unique on `gatewayIntentId`. Check-and-insert **before** any business-logic side effects, to close the race window against PayMongo's documented webhook retries. Early-return `200 { ok: true }` if already processed.

## Multi-tenant / per-company credentials

Each tenant can store its own PayMongo keys rather than sharing one global merchant account (this is NOT PayMongo's own "Linked Accounts" product — it's a plain per-row credential store):

```prisma
model TenantPaymentGateway {
  id            String  @id @default(cuid())
  tenantId      String
  tenant        Tenant  @relation(fields: [tenantId], references: [id])
  gateway       String
  enabled       Boolean @default(true)
  isDefault     Boolean @default(false)
  publicKey     String
  secretKey     String
  webhookSecret String
  @@unique([tenantId, gateway])
}
```

`resolveGateway(tenantId)` looks up the tenant's enabled+default row, falling back to global env-based credentials if none exists (dev convenience, and a soft-launch path before every tenant is onboarded).

**Known unsolved gap, fix before relying on this in production**: payment *creation* correctly resolves per-tenant credentials, but if the webhook route always builds the gateway from global env vars instead of re-resolving per tenant, signature verification will use the wrong `webhookSecret` the moment a tenant has its own keys — because PayMongo's webhook payload carries no tenant identifier to resolve by. Fix by encoding the tenant id in the webhook URL path (e.g. `/api/checkout/webhook/[tenantId]`) or an equivalent out-of-band routing key, then resolve credentials from that before calling `parseWebhook`. Do not assume the global-env webhook path is safe once a second tenant gets its own PayMongo account.

There is no self-service onboarding UI for a tenant to connect their own PayMongo account yet in the reference implementation — rows are inserted admin/DB-side. Build one if the product requires tenants to self-onboard.

## Auth to PayMongo's API

HTTP Basic, secret key as username, empty password: `Basic ${base64(secretKey + ':')}`. Sandbox vs. live mode is controlled purely by which key you use (`sk_test_...` vs `sk_live_...`) — there is no separate code path or `NODE_ENV` branch for test mode.

## What this skill does NOT cover

Storage (R2/S3 signed URLs for uploads) is a separate concern from payments and is not part of PayMongo integration — don't conflate the two. Note also that the reference implementation's own upload flow does direct server-side proxied uploads (not presigned PUT/GET), which contradicts gawa-brain's own D4 decision (signed-URL direct-to-storage) — that's a gap in the reference app, not a pattern to copy.
