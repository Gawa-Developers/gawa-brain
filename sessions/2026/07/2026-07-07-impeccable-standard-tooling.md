# 2026-07-07 — Adopt Impeccable as Standard Design Tooling

## Summary

While working on travel-it-all, installed [Impeccable](https://impeccable.style) — a design-quality CLI/skill pack built for AI coding agents (Claude, Cursor, Codex, Gemini) — to get deterministic anti-pattern detection and structured design commands instead of relying on ad-hoc judgment each session. Decided this should become standard on every Gawa project, not just this one.

## What It Is

45 deterministic detector rules covering:
- **AI-generated-UI "tells":** gradient text on headings, side-tab accent borders, purple/violet gradients, glowing dark-mode accents
- **Accessibility/contrast:** WCAG AA violations, gray-on-tinted-background text
- **Typography/layout:** overused fonts, flat hierarchy, nested cards, monotonous spacing, tiny touch targets

Plus a command set (`craft`, `shape`, `audit`, `critique`, `polish`, `brand`, `colorize`, `harden`, `document`, `extract`, `live`, ...) installed as a Claude Code skill, each backed by a reference doc the skill's own SKILL.md requires reading before that command runs.

## Install Steps Confirmed Working

```bash
npx impeccable skills install -y --providers=claude --scope=project
```

Installs into `.claude/skills/impeccable/` (SKILL.md + scripts/ + reference/). No new npm dependency in package.json — it's a skill-files install, not a library import.

## Gotcha Hit and Confirmed

Tried running `/impeccable init` immediately after installing, in the same session — failed with "Unknown command" (and the Skill tool itself returned "Unknown skill: impeccable"). This is a session-freshness issue, not a broken install: the harness enumerates available skills once at session start, so anything written to `.claude/skills/` mid-conversation isn't visible until the session restarts. Confirmed the SKILL.md was well-formed (proper frontmatter: name, description, version, user-invocable, argument-hint, allowed-tools) before concluding it was a restart issue, not a config problem.

## Decision

Promoted to [decisions.md — D8](../../../memory/decisions.md#d8-install-impeccable-as-standard-design-quality-tooling-on-every-project): install on every project, new or existing, via the command above, then restart the session before running `/impeccable init`. Added the restart gotcha to [feedback.md — F7](../../../memory/feedback.md) and the setup steps to [operations.md — Project Setup](../../../memory/operations.md).
