# 2026-06-26 — gawa-brain Bootstrap

## What Happened

- Created `Gawa-Developers/gawa-brain` GitHub repo for org-wide engineering knowledge
- Cloned to `~/projects/gawa-developments/gawa-brain/`
- Wrote canonical `memory/` structure: `MEMORY.md`, `operations.md`, `engineering.md`, `decisions.md`, `feedback.md`
- Content sourced from: PMC B2B WordPress project patterns, travel-it-all adversarial review findings, and general Gawa engineering conventions established across sessions

## What's in Each File

- **`operations.md`** — git flow, branch naming (`hotfix/v*.*.*` default, `feat/` on request), release sequence (merge before tag), PR conventions, commit message format
- **`engineering.md`** — headless interface pattern, webhook security (raw Buffer before signature verify), async job rules, signed URL upload pattern, multi-tenancy isolation hierarchy, PgBouncer session-mode requirement, security baseline, anti-patterns table
- **`decisions.md`** — 5 durable decisions: headless provider interfaces, RLS as primary tenant guard, async for side effects, signed URL uploads, monolith-first
- **`feedback.md`** — 6 agent behavior rules: no direct push to protected branches, ask for version string, tag after merge only, hotfix branch default, read before edit, validate memory before acting

## Scope Rule

This brain is **general engineering patterns only**. No project-specific content (no PMC, no travel-it-all, no client data). Project-specific memory lives in each project's own `memory/` folder.
