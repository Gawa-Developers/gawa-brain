# Feedback — Gawa Developers

Validated agent behavior corrections applicable across all Gawa projects.

---

## F1: Never Push Directly to `main` or `develop`

Never push directly to a protected base branch. Always create a working branch (`hotfix/v*.*.*` or `feat/[description]`), push that branch, and open a PR.

**Why:** Direct pushes bypass code review, skip CI, and can corrupt the release history. The user corrected this explicitly after a direct `git push origin develop` was attempted on the PMC project.

**How to apply:** Before every commit + push cycle, verify `git branch --show-current`. If it's `main` or `develop`, stop and create the right working branch first.

---

## F2: Always Ask for the Version String Before Tagging

Never auto-increment the version. Always ask the project owner for the version string before bumping files, creating a tag, or creating a GitHub Release.

**Why:** Version strings communicate meaning to the client and other developers. Auto-incrementing removes the project owner's intentional control over version semantics (patch vs minor vs major).

**How to apply:** When a release task is requested, pause at the version bump step and confirm the version string before writing to any file.

---

## F3: Tags Must Point to Commits Already on the Base Branch

The sequence is: commit → push working branch → merge PR → tag → release. Never create a tag before the PR is merged.

**Why:** A tag on an unmerged commit creates a dangling release — the tag exists but the code isn't in `main`/`develop`. This confuses deploys and rollbacks.

**How to apply:** Verify the working branch is merged before running `git tag`.

---

## F4: `hotfix/v*.*.*` Is the Default Branch Type

Unless the project owner explicitly says "feature branch," always use `hotfix/v*.*.*` for new work branches.

**Why:** Most iterative work is incremental (bug fixes, content updates, minor enhancements). Feature branches are only needed for parallel long-running work that must be gated. Using `hotfix/` as default keeps branches short-lived and PRs focused.

**How to apply:** On every new task that requires a branch, default to `hotfix/vX.X.X` matching the intended release version.

---

## F5: Read a File Before Editing It

Always read a file with the Read tool before calling Edit or Write on an existing file. The Edit tool will error if the file hasn't been read first in the session.

**Why:** This is a harness requirement. It also prevents accidentally overwriting content that was changed since the session started.

**How to apply:** If you are about to call Edit and haven't read the target file yet this session, read it first.

---

## F6: Validate Memory Before Acting on It

A memory that names a file path, function, or flag is a claim about what existed when the memory was written — not what exists now. Before recommending or acting on a specific file or symbol from memory, verify it still exists.

**Why:** Code changes. Memories don't update automatically. Acting on a stale memory (e.g. editing a file that was renamed) wastes time and may introduce errors.

**How to apply:** For any memory-derived file path or symbol, do a quick `ls` or `grep` before using it as a basis for edits.

---

## F7: A Skill Installed Mid-Session Isn't Usable Until the Session Restarts

Running `npx impeccable skills install` (or installing any other Claude Code skill) writes files to `.claude/skills/` immediately, but the running harness has already enumerated its available skills for this session and won't pick up the new one. Trying to invoke it — via the Skill tool or the user typing the slash command — fails ("Unknown skill" / "Unknown command") even though the files are correctly in place on disk.

**Why:** Skill discovery happens once per session start, not on every message. This isn't a bug in the install — it's confirmed by checking the skill's own SKILL.md is well-formed and still failing to invoke it.

**How to apply:** After installing a new skill mid-conversation, tell the user to restart the Claude Code session (new conversation / reopen the window) before trying to invoke it. Don't spend time debugging the skill's manifest first — verify it's a session-freshness issue by attempting the Skill tool call once; if it errors with "Unknown skill" despite a well-formed SKILL.md, that confirms it, not a real config problem.

---

## F8: `npx impeccable skills install --scope=global` Doesn't Reliably Honor the Flag

Tried `--scope=global`, `--scope global`, and `-y ... --scope=global --force` in combination — every variant still wrote to `<cwd>/.claude/skills/impeccable` instead of `~/.claude/skills/impeccable`. This held even though the CLI's own source claims explicit `--scope` should win over the `-y` project-default behavior.

**Why:** Unclear — possibly a version mismatch between the published npm CLI and the scope-handling logic, or a detection heuristic that overrides the flag when run inside a directory Claude Code already recognizes as a project. Not worth debugging further per-session.

**How to apply:** Don't loop retrying flag variants. Let it install into the project as it will, then move the result once: `mkdir -p ~/.claude/skills && mv .claude/skills/impeccable ~/.claude/skills/impeccable && rmdir .claude/skills`. See [decisions.md — D8](decisions.md#d8-install-impeccable-once-globally--not-per-project) and [operations.md — Project Setup](operations.md#project-setup) for the standing policy this produced (install once, globally, per machine).
