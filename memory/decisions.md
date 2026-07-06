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

---

## D6: Vercel Hosting Plan Follows Commercial Status, Not Repo Ownership

**Decision:** For projects hosted on Vercel, stay on the Hobby (free) plan only while the deployment is genuinely non-commercial (no payments, ads, donation asks, or public marketing/sale of the product). Upgrade to Pro the moment any of those go live — do not try to stay on Hobby via workarounds once the product is commercial.

**Why:** Vercel's fair use guidelines define commercial usage as any Deployment used for the financial gain of anyone involved in production of the project, including a paid employee or consultant. Enforcement in practice targets deployments that process payments, run ads, solicit donations, or are marketed for sale. Separately, Vercel's Hobby plan cannot connect to a GitHub Organization-owned repository at all — only personal-account-scoped repos — regardless of commercial status. A mirror-to-personal-account repo can work around the org-repo restriction pre-launch, but it does not work around the commercial-use restriction, and is unnecessary overhead once Pro is required anyway (Pro connects directly to org repos).

**Tradeoff accepted:** Pre-launch MVP work on a Hobby-tier personal-account mirror adds a sync step (PR per release) if the canonical repo is org-owned. This is only worth doing pre-launch; switch to Pro connected directly to the org repo at the point of going live rather than continuing to maintain two repos.

---

## D7: Prove Payment Demand on a Zero-Registration Rail Before Committing to a Verified Processor

**Decision:** When a product's primary transaction flow doesn't strictly require a fully-verified payment processor (e.g. a COD-first business where online payment is a secondary channel), default to a zero-registration payment rail (e.g. a personal PayPal account) to validate that real demand exists before starting business verification (KYC/KYB) with a processor like PayMongo, Stripe, or similar. Build and sandbox-test the verified-processor integration in parallel so it's code-complete and ready to flip live — but gate the actual business registration on observed transaction volume, not on project calendar milestones.

**Why:** Processors that support real payouts (PayMongo, Stripe, etc.) gate live API keys behind mandatory business verification — often a multi-step review (e.g. PayMongo separately reviews "can accept payments" vs "can pay out to a bank account") taking on the order of two weeks, with no way to soft-launch or test real-money demand pre-verification. That registration effort is wasted if the product's online-payment volume never materializes. A zero-registration rail like a personal PayPal account can receive and withdraw real money in many regions without business registration, making it a legitimate low-commitment way to validate demand first — the registration effort for the verified processor becomes justified by data instead of a guess.

**Tradeoff accepted:** Zero-registration personal payment rails are generally intended for "occasional" use per their own terms — sustained or high-volume business-pattern activity risks an account hold pending retroactive verification, which is worse timing than proactive registration. This approach is only appropriate while volume is genuinely low/occasional; treat rising volume as the trigger to start the verified processor's registration process (accounting for its multi-week lead time) rather than waiting until the zero-registration rail gets flagged.

---

## D8: Install Impeccable as Standard Design-Quality Tooling on Every Project

**Decision:** Install the `impeccable` CLI/skill pack ([impeccable.style](https://impeccable.style)) on every Gawa project — new projects at bootstrap, existing projects retroactively. Run `npx impeccable skills install -y --providers=claude --scope=project`, **restart the Claude Code session** (skills installed mid-session aren't picked up by the running harness — `/impeccable init` will fail with "Unknown command" until then), then run `/impeccable init` to generate the project's PRODUCT.md (and DESIGN.md when relevant) design context.

**Why:** Impeccable provides 45 deterministic anti-pattern detectors (accessibility/contrast violations, AI-generated-UI "tells" like gradient text and side-tab accents, typography and layout quality issues) plus a structured command set (craft, audit, critique, polish, brand, colorize, harden, etc.) that give every project a consistent, tool-enforced design quality bar instead of relying on ad-hoc judgment calls each session. It's built specifically for AI coding agents (Claude, Cursor, Codex, Gemini), so the commands and reference docs are already tuned for exactly this kind of workflow.

**Tradeoff accepted:** Adds a third-party dependency and a `.claude/skills/impeccable/` directory to every repo, plus mandatory reference-doc reads before certain design tasks (per the skill's own SKILL.md) — added upfront context-reading overhead per session is worth the consistency gain. The mid-session-install-needs-a-restart gotcha is a one-time friction point per project, not recurring.
