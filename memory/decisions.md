# Decisions — Gawa Developers

Durable org-wide engineering decisions. Include context and the tradeoff accepted.

---

## D1: Headless Interface for External Provider Integrations

**Decision:** Define an interface for any external provider (payment, email, SMS, storage, search). Never call provider SDKs directly in business logic.

**Why:** We routinely serve clients who need to swap providers (PayMongo → PayMaya, Mailgun → Postmark) without rewriting business logic. Interfaces let us do this at the injection layer. It also makes unit testing possible without network calls.

**Tradeoff accepted:** More boilerplate upfront per integration. Worth it on any project that expects >1 provider or multi-tenant toggle behavior.

---

## D2: Row-Level Security as Primary Tenant Isolation Guard

**Decision:** For multi-tenant applications on PostgreSQL, enforce tenant isolation via RLS policies at the database layer — not via ORM middleware alone.

**Why:** ORM middleware can be bypassed by raw queries, background job workers, webhook handlers, and database migrations. RLS cannot be bypassed without explicit `SECURITY DEFINER` escalation or superuser credentials. One enforcement layer at the database means all access paths are protected.

**Tradeoff accepted:** Requires session-mode PgBouncer (not transaction-mode) when using `SET LOCAL` session variables. Adds complexity to local dev setup. Worth it — tenant data leaks are catastrophic and hard to fully audit in ORM middleware alone.

---

## D3: Async for All Non-Trivial Side Effects

**Decision:** PDF generation, transactional email, file processing, payment webhook processing, and any external API call with unpredictable latency must be handled asynchronously via a job queue (BullMQ or equivalent).

**Why:** Synchronous side effects in request handlers block the event loop and degrade P99 latency for all users. They also create partial-success failure modes (e.g. booking saved but confirmation email never sent). A job queue gives retry logic, observability, and decoupling for free.

**Tradeoff accepted:** Adds Redis as a required dependency. Workers must be monitored separately. Worth it for any production-bound service.

---

## D4: Signed URL Direct-to-Storage Uploads

**Decision:** All file upload flows use signed URL patterns (S3/R2/GCS presigned PUT URLs). The application server never proxies file bytes.

**Why:** Proxying uploads through the app server creates a memory/CPU bottleneck that scales poorly and can destabilize the app under concurrent large uploads. Signed URLs let clients upload directly to storage at full network speed without touching the app server.

**Tradeoff accepted:** Requires a post-upload validation step (the server must verify the uploaded file before committing the URL to the DB). Slightly more complex client-side flow. Worth it — the performance and stability gains are large.

---

## D5: monolith-first, Extract Later

**Decision:** Start new projects as a single deployable unit (e.g. Next.js monolith, single Laravel app). Do not decompose into microservices until there is a concrete, proven scaling or team-boundary reason to do so.

**Why:** Microservices add operational complexity (service discovery, distributed tracing, inter-service auth, deployment orchestration) that is unjustified at early stage. Most startup-scale products never reach the point where this complexity is worth it. Monoliths are faster to develop, easier to debug, and simpler to deploy.

**Tradeoff accepted:** Refactoring a monolith into services later is non-trivial. Mitigated by designing clear internal module boundaries from the start so extraction is feasible if needed.
