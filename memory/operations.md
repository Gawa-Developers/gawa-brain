# Operations — Gawa Developers

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
