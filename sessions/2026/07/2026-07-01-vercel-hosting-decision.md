# 2026-07-01 — Vercel Hosting Strategy for Org-Owned Repos

## Summary

Worked through hosting strategy for a project whose canonical repo lives under a GitHub Organization, with the goal of avoiding paid hosting during MVP polishing while staying compliant once the product goes live.

## Options Considered

1. **Mirror the org repo into a personal GitHub account, connect that to Vercel Hobby, PR each release tag into the mirror as "production."** Dodges Vercel's org-repo restriction, but adds a permanent two-repo sync burden and does nothing about the commercial-use restriction — Hobby forbids commercial use regardless of which account owns the repo.
2. **Deploy via GitHub Actions + Vercel CLI directly from the org repo**, bypassing Vercel's Git integration entirely. Works on Hobby without a second repo, but still can't be used commercially without upgrading.
3. **Go straight to Vercel Pro connected to the org repo.** Removes both restrictions at once ($20/mo minimum); simplest long-term, only needed once the product is commercial.

## Vercel Plan Facts Confirmed (as of 2026-07)

- **Hobby**: free, but (a) restricted to non-commercial personal use per Vercel's fair use guidelines, and (b) cannot connect to a GitHub Organization-owned repo at all — only personal-account-scoped repos.
- **Pro**: $20/mo per deploying seat (1 seat included), $20 usage credit, 1TB transfer / 10M edge requests included. Supports commercial use and connects directly to org-owned repos.
- **Commercial usage definition** (Vercel fair use guidelines): financial gain of anyone involved in producing the deployment, including a paid employee/consultant — but enforcement in practice centers on the deployment itself doing something monetizable (payments, ads, donations, affiliate links, marketed sale).

## Decision

Promoted to [decisions.md — D6](../../../memory/decisions.md#d6-vercel-hosting-plan-follows-commercial-status-not-repo-ownership): stay on Hobby only while genuinely pre-commercial; upgrade to Pro connected directly to the org repo at the point of going live, rather than maintaining a mirror-repo workaround long-term.

## Context Specific to This Case

Project is being built solo (no paid employees/contractors), currently in MVP-polishing phase, not yet live or monetized — so the "paid consultant" clause in Vercel's commercial-use definition does not apply yet.
