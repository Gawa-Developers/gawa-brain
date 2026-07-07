# 2026-07-07 — Skills Bootstrap: gawa-brain as the Canonical Skills Source

## Summary

Set up `gawa-brain/skills/` as the single, version-controlled source for custom Claude Code skills, symlinked into `~/.claude/skills/` so every Gawa project picks them up as personal skills without per-repo duplication. Also revised the Impeccable install policy (see decisions.md D8, revised) from per-project to once-globally, triggered by finding a 2.2MB untracked duplicate at `travel-it-all/.claude/skills/impeccable`.

## What Changed

1. **Plugins enabled** (user scope, via `claude plugin install <name> --scope user`):
   - `skill-creator@claude-plugins-official` — meta-skill for authoring/improving skills interactively instead of hand-writing SKILL.md.
   - `frontend-design@claude-plugins-official` — frontend interface quality skill.
   - Both are Anthropic marketplace plugins, not gawa-brain content — no files to track for these beyond the `~/.claude/settings.json` `enabledPlugins` entries.

2. **Impeccable migrated to global scope.** Moved `travel-it-all/.claude/skills/impeccable` → `~/.claude/skills/impeccable`. The CLI's own `--scope=global` flag did not reliably work (kept writing to cwd's `.claude/` regardless of flag combination tried) — see feedback.md F8. Manual `mv` was the reliable fix. D8 revised accordingly.

3. **Custom skills authored in `gawa-brain/skills/`:**
   - `paymongo-integration/SKILL.md` — extracted from travel-it-all's actual `src/lib/payments/` implementation: Sources-API workaround for the broken `/v1/links` redirect, the Source→Payment two-step charge gotcha, `paymongo-signature` webhook header format (`t=,te=,li=`, HMAC over `${timestamp}.${rawBody}`), idempotency via `WebhookEvent`, and the `TenantPaymentGateway` per-tenant credential model — including a documented **unfixed gap**: the webhook route always builds the gateway from global env vars, never re-resolving per-tenant, so a tenant with its own webhook secret will fail signature verification once onboarded.
   - `docx/` and `pdf/` — copied from `github.com/anthropics/skills` (Anthropic's public example skills repo), unmodified aside from dropping the nested `.git`.

4. **Symlinked into `~/.claude/skills/`:** `paymongo-integration`, `docx`, `pdf` (plus the pre-existing `impeccable` from step 2). All four now load automatically in every Gawa project.

## Deferred

- **design.md-as-skill**: the plan was to convert `travel-it-all/DESIGN.md` (whitelabel app UI/animation conventions) into a skill. On inspection, that file actually documents **Replicate.com's** design system (cream canvas, `rb-freigeist-neue` font, orange accent — an unrelated AI/ML platform), not travel-it-all's own UI, and has no animation/motion section. No record of an animation/motion discussion exists elsewhere in gawa-brain either. User chose to skip this rather than build a skill from the wrong source — revisit once the correct DESIGN.md / motion notes are available.

## Gotcha Hit and Confirmed

Same mid-session-restart caveat as [[2026-07-07-impeccable-standard-tooling]] applies to all four skills added this session — they won't be invocable in the current running harness until the session restarts.

## Also Surfaced (not acted on)

Researching the PayMongo integration turned up that travel-it-all's actual upload implementation (`src/lib/storage.ts`) does direct server-side proxied uploads, not presigned R2/S3 URLs — which contradicts gawa-brain's own **D4** (signed-URL direct-to-storage). Flagged in the paymongo-integration skill as "a gap in the reference app, not a pattern to copy," but D4 itself was not changed — someone should decide whether to fix the app or revisit D4.

## Decision

Promoted to [decisions.md — D8, revised](../../../memory/decisions.md#d8-install-impeccable-once-globally--not-per-project) and [feedback.md — F8](../../../memory/feedback.md#f8-npx-impeccable-skills-install---scopeglobal-doesnt-reliably-honor-the-flag). New convention documented in [operations.md — Custom skills](../../../memory/operations.md#custom-skills-gawa-brainskills).
