# Operations — Gawa Developers

## Project Setup

**Impeccable is installed once, globally, at `~/.claude/skills/impeccable/` — not per project.** See [decisions.md — D8](decisions.md#d8-install-impeccable-once-globally--not-per-project) for the why. Check `ls ~/.claude/skills/impeccable` before reinstalling — if it's already there, skip straight to `/impeccable init` below.

If it's missing, the CLI's `--scope=global` flag has not reliably honored itself in practice:

```bash
npx impeccable skills install -y --providers=claude --scope=global
# if it writes to ./.claude instead of ~/.claude, move it manually:
mkdir -p ~/.claude/skills
mv .claude/skills/impeccable ~/.claude/skills/impeccable
rmdir .claude/skills 2>/dev/null
```

Then **restart the Claude Code session** — skills added to `~/.claude/skills/` aren't picked up by the running harness, so `/impeccable init` will fail with "Unknown command" if you try it before restarting. After restarting, per project:

```
/impeccable init
```

This generates that project's PRODUCT.md (and DESIGN.md when relevant). The skill install is one-time per machine; `/impeccable init` is still per-project — run it retroactively the next time you touch an existing repo that doesn't have a PRODUCT.md yet.

### Custom skills (gawa-brain/skills/)

Skills authored in-house (not third-party installs like Impeccable) live in `gawa-brain/skills/<name>/`, version-controlled like the rest of the brain, and are symlinked into `~/.claude/skills/<name>` so every project picks them up without duplicating files:

```bash
ln -s /path/to/gawa-brain/skills/<name> ~/.claude/skills/<name>
```

New machine setup: after cloning gawa-brain, symlink every folder under `gawa-brain/skills/` into `~/.claude/skills/` before starting work on any other Gawa project.

---

## Git Flow

**All Gawa projects follow the same branch conventions.**

### Branch naming
- `hotfix/v*.*.*` — default for fixes, patches, and incremental releases (e.g. `hotfix/v1.3.8`)
- `feat/[description]` — only when the project owner explicitly requests a feature branch
- Never invent other branch types without direction

### Rules
- **Never push directly to `main` or `develop`.** Every change goes through a pull request.
- Before committing, confirm the current branch is NOT the protected branch (`main` or `develop`).
- If on a protected branch, create the working branch first: `git checkout -b hotfix/vX.X.X`
- Every sub-branch push must have an accompanying PR to the base branch.

---

## Release Flow

Order matters. Never skip steps.

1. Commit all work to the working branch (`hotfix/v*.*.*` or `feat/...`)
2. Push the branch to origin
3. Open a PR to `main` (or `develop` — match the repo's base branch convention)
4. Merge the PR
5. **Only after merge:** bump the version in all required places
6. Create and push the tag pointing to the merged commit: `git tag vX.X.X && git push origin vX.X.X`
7. Create a GitHub Release titled `[Project Name] vX.X.X`

### Version bump locations (minimum — verify per project)
- `package.json` (for Node.js projects)
- Plugin/theme header comments (for WordPress projects)
- Version constants defined in source (e.g. `PMC_B2B_CORE_VERSION`)

### Rules
- **Never tag before merging.** Tags must point to commits already in the base branch.
- **Never auto-increment.** Always ask the project owner for the version string before tagging.
- **Never create a release before the tag is on the base branch.**

---

## PR Conventions

- Title: imperative, under 70 characters (`fix(basecamp-hero): wire CTA to #basecamp-programs`)
- Body: Summary (bullet points) + Test plan (checklist)
- Every PR merges into `develop` or `main` — never into another feature branch unless explicitly instructed.

---

## Commit Message Format

```
type(scope): short description

Longer explanation if needed.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `style`
