# Plan 001: Keep execute-mode worktrees workspace-local and ignored

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report -- do not improvise. When done, update the status row for this plan
> in `plans/README.md` -- unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 5428507..HEAD -- README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: dx
- **Planned at**: commit `5428507`, 2026-06-12
- **Upstream issue**: https://github.com/shadcn/improve/issues/2

## Why this matters

Issue #2 reports that `/improve execute <plan>` requires an isolated disposable
git worktree but does not say where that worktree should be created. In VS Code
agent environments, a sibling path such as `/path/project-exec-002` can fall
outside the opened workspace and trigger repeated filesystem approval prompts,
even though the worktree belongs to the same repo task. The fix should preserve
the safety model -- executor edits happen in an isolated worktree and the
advisor never edits source directly -- while making the default path
workspace-local, deterministic, ignored, and reported in the verdict.

## Current state

- `README.md` -- public usage docs; it describes execute-mode isolation but not
  the filesystem location for the disposable worktree.
- `skills/improve/SKILL.md` -- the installed skill instructions; it names
  execute-mode isolation but leaves path selection to the host agent/model.
- `skills/improve/references/closing-the-loop.md` -- detailed execute/review
  procedure; this is the main file that must define the workspace-local
  worktree path, ignore rule, and reporting requirement.
- No `plans/` directory existed before this advisor run, and the repo has no
  root `.gitignore` in the current file list. Do not solve the issue by adding
  repo-specific ignored worktrees here; solve it by changing the skill's
  instructions for future target repos.
- Repo shape: this is a Markdown Agent Skill package. There is no `package.json`,
  no build script, no test script, and no CI config in the current file list.
  Verification is therefore text inspection plus `git diff --check`.

Relevant excerpts at commit `5428507`:

```markdown
README.md:96
- **`execute <plan>`** spawns a cheaper executor subagent in an isolated git worktree, hands it the plan, then reviews the result like a tech lead -- re-runs every done criterion, checks scope compliance, reads the diff against intent. Verdict: approve (merging stays your call), send back for revision (max 2 rounds), or block and refine the plan.
```

```markdown
skills/improve/SKILL.md:18
1. **Never modify source code yourself.** No edits, no fixes, no "quick wins while you're in there." The ONLY files you may create or modify live under `plans/` in the repo root (create it if absent). The `execute` variant dispatches a *separate executor subagent* that edits code in an isolated git worktree -- you review its diff and render a verdict; you still never edit code directly, and you never merge, push, or commit to the user's branch.

skills/improve/SKILL.md:116
- `execute <plan>` -> dispatch a cheaper executor subagent on one plan (isolated worktree), then review its diff like a tech lead -- re-run done criteria, check scope, read the code -- and render a verdict. Treat the executor's diff as untrusted until reviewed: verify every hunk traces to a plan step and reject any out-of-scope change, however plausible it looks. Requires a host agent that can spawn subagents in an isolated worktree; if yours can't, say so and hand the plan over for manual execution instead. **Read [references/closing-the-loop.md](references/closing-the-loop.md) before the first dispatch.**
```

```markdown
skills/improve/references/closing-the-loop.md:19
Spawn **one** `general-purpose` subagent with `isolation: "worktree"`. Executor model: default `sonnet`; use what the user named if they named one (`execute 003 haiku`).

skills/improve/references/closing-the-loop.md:23-31
1. **The full plan file text, inlined.** The worktree contains only committed files -- if `plans/` is uncommitted, the executor can't read it. Never assume; always inline.
2. The executor preamble:

> You are the executor for the implementation plan below. Follow it step by
> step. Run every verification command and confirm the expected result before
> moving on. Touch only the files listed as in scope. If any STOP condition
> occurs, stop immediately and report. Do not improvise around obstacles.
> Commit your work in the worktree following the plan's git workflow section.
> One override: SKIP the plan's instruction to update `plans/README.md` --
> your reviewer maintains the index.

skills/improve/references/closing-the-loop.md:65
| **APPROVE** | Criteria pass, scope clean, quality holds | Update index status to DONE. Present to the user: diff summary, worktree path and branch, anything from NOTES. **Merging is the user's decision -- never merge, push, or commit to their branch.** |
```

Conventions to match:

- Markdown-first documentation with short sections, direct imperatives, and
  tables for procedural state.
- `skills/improve/SKILL.md` is the canonical skill entrypoint; keep detailed
  operational rules in `skills/improve/references/closing-the-loop.md` and link
  to them from the entrypoint instead of duplicating long procedures.
- The README is user-facing and shorter; summarize behavior there without
  copying the full dispatch checklist.
- Recent commits use concise conventional messages such as
  `docs: drop named-tool references, keep generic "composes with such repos"`
  and `feat: ingest intent & design docs (ADRs, PRDs, CONTEXT.md, DESIGN.md) in recon`.

## Commands you will need

Run from the repo root.

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Inspect status | `git status --short` | only intentional changes are shown |
| Check execute guidance anchors | `rg -n "plans/\\.worktrees|plans/\\.gitignore|WORKTREE PATH|BRANCH|workspace-local|nested repos" README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` | matches appear in the updated docs; exit 0 |
| Check Markdown diff hygiene | `git diff --check -- README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` | exit 0, no whitespace errors |

No install, build, lint, or test command exists in this repo at commit `5428507`.

## Scope

**In scope** (the only source files you should modify):

- `skills/improve/references/closing-the-loop.md`
- `skills/improve/SKILL.md`
- `README.md`
- `plans/README.md` status row only, unless your reviewer says they maintain it

**Out of scope** (do NOT touch, even though they look related):

- `skills/improve/references/audit-playbook.md` -- audit guidance is unrelated.
- `skills/improve/references/plan-template.md` -- this issue concerns execute
  dispatch mechanics, not the per-plan template.
- `examples/001-extract-shadow-config-resolution.md` -- historical sample plan;
  do not rewrite it for this behavior.
- `.claude-plugin/plugin.json`, `LICENSE.md`, or any package metadata.
- Creating an actual disposable worktree in this repository as part of the doc
  fix. The implementation should document the behavior future `/improve execute`
  runs must follow.
- Publishing or commenting on the GitHub issue. This plan was not invoked with
  `--issues`.

## Git workflow

- Branch: `advisor/001-workspace-local-execute-worktrees`
- Commit once after all verification passes.
- Commit message: `docs: keep execute worktrees workspace-local`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Define the workspace-local worktree procedure in `closing-the-loop.md`

Edit `skills/improve/references/closing-the-loop.md`. In the `Dispatch` section,
replace the current generic one-line instruction to spawn a subagent with
explicit workspace-local path rules before the subagent prompt requirements.

The updated procedure must say all of the following, in the style of the file:

- Default disposable worktree root: `<repo root>/plans/.worktrees/`.
- Default disposable worktree path: `<repo root>/plans/.worktrees/<plan-id>-<slug>/`,
  where `<plan-id>-<slug>` comes from the plan filename without `.md` (for
  example, `002-some-plan`).
- For nested repos or multi-repo workspaces, still prefer the selected repo's
  own `<repo root>/plans/.worktrees/`. If a host API forces a workspace-level
  worktree root, prefix the folder name with the sanitized repo directory name,
  for example `js-store-002-selected-period-low-utilisation/`.
- Before dispatch, create or maintain `<repo root>/plans/.gitignore` with a
  `.worktrees/` entry. Preserve existing lines and do not add duplicates. Do not
  edit the target repo's root `.gitignore` unless `plans/.gitignore` is
  impossible for that repo.
- If the host's worktree-isolation API lets the advisor specify a path, use the
  path above. If it does not, create the git worktree at that path yourself and
  launch the executor rooted there. Do not silently accept a sibling path outside
  the workspace.
- If the computed path would be outside the current workspace, or the advisor
  cannot create/use a workspace-local worktree, stop and tell the user manual
  execution is needed.

Keep the existing safety properties: executor edits only in the isolated
worktree, advisor never edits source directly, and merging remains the user's
decision.

Update the executor report format in the same file so it requires these fields:

```text
STATUS: COMPLETE | STOPPED
WORKTREE PATH: <absolute or workspace-relative path>
BRANCH: <branch name or detached HEAD>
STEPS: per step -- done/skipped + verification command result
STOPPED BECAUSE: (only if STOPPED) which STOP condition, what was observed
FILES CHANGED: list
NOTES: anything the reviewer should know (deviations, surprises, judgment calls)
```

Also update review/reconcile wording where needed:

- Scope checks should use the explicit worktree path from the report.
- The final verdict should include the worktree path and branch.
- The `reconcile` `IN PROGRESS` note should mention checking the default
  `plans/.worktrees/` location for stale executor worktrees.

**Verify**: `rg -n "plans/\\.worktrees|plans/\\.gitignore|WORKTREE PATH|BRANCH|workspace-local|nested repos" skills/improve/references/closing-the-loop.md` -> matches for the path, ignore rule, report fields, workspace-local wording, and nested-repo wording.

### Step 2: Summarize the new execute default in `SKILL.md`

Edit `skills/improve/SKILL.md` without duplicating the full dispatch procedure.

Make these targeted updates:

- In Hard Rule 1 or 2, clarify that execute-mode may create and maintain an
  ignored workspace-local disposable worktree under `plans/.worktrees/` and a
  `plans/.gitignore` ignore entry, while the advisor still never edits source
  directly.
- In the `execute <plan>` invocation variant, mention the default path
  `plans/.worktrees/<plan-id>-<slug>/`, the ignored-directory requirement, and
  the requirement to report the worktree path and branch in the verdict.
- Keep the detailed mechanics delegated to
  `references/closing-the-loop.md`.

Do not weaken the existing rules about source edits, verification, diff review,
or merging.

**Verify**: `rg -n "plans/\\.worktrees|plans/\\.gitignore|worktree path|branch|closing-the-loop" skills/improve/SKILL.md` -> matches in the hard rules or execute variant; exit 0.

### Step 3: Update the public README summary

Edit `README.md` so users see the new default without reading the reference
file first.

Make these targeted updates:

- In the "How to use" execute step, say `/improve execute` uses an ignored
  workspace-local disposable worktree under `plans/.worktrees/<plan-id>-<slug>/`
  by default.
- In "Closing the loop", update the `execute <plan>` bullet with the same
  summary and mention that the verdict reports the worktree path and branch.
- In "Hard rules", keep the current short rule but add that executor worktrees
  are disposable and ignored under `plans/.worktrees/` by default.

Keep this user-facing section concise; the full algorithm belongs in
`closing-the-loop.md`.

**Verify**: `rg -n "plans/\\.worktrees|worktree path|branch|ignored" README.md` -> matches in the execute documentation; exit 0.

### Step 4: Run final documentation checks

Run the cross-file checks from the command table.

**Verify**:

- `rg -n "plans/\\.worktrees|plans/\\.gitignore|WORKTREE PATH|BRANCH|workspace-local|nested repos" README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` -> exit 0 with matches in all three files.
- `git diff --check -- README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` -> exit 0.
- `git status --short` -> only `README.md`, `skills/improve/SKILL.md`, `skills/improve/references/closing-the-loop.md`, and the allowed `plans/README.md` status update are modified. If your reviewer told you they maintain `plans/README.md`, it should not be modified.

## Test plan

- No automated tests exist for this Markdown-only skill repo at commit
  `5428507`.
- Treat the `rg` checks above as regression checks: they prove the exact issue
  terms are now represented in the entrypoint, detailed execute procedure, and
  user-facing README.
- Treat `git diff --check` as the formatting gate.
- Human review should read the final `closing-the-loop.md` dispatch section and
  confirm it gives an executor no reason to choose a sibling path outside the
  workspace by default.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] `skills/improve/references/closing-the-loop.md` defines
  `<repo root>/plans/.worktrees/<plan-id>-<slug>/` as the default execute-mode
  disposable worktree path.
- [ ] `skills/improve/references/closing-the-loop.md` instructs advisors to
  create or maintain `<repo root>/plans/.gitignore` with a `.worktrees/` entry,
  preserving existing lines and avoiding duplicates.
- [ ] `skills/improve/references/closing-the-loop.md` covers nested repos or
  multi-repo workspaces and says to stop rather than use a path outside the
  current workspace.
- [ ] The executor report format requires `WORKTREE PATH:` and `BRANCH:`.
- [ ] The final verdict instructions still require reporting worktree path and
  branch to the user.
- [ ] `README.md` and `skills/improve/SKILL.md` summarize the new default
  without duplicating the full detailed procedure.
- [ ] `rg -n "plans/\\.worktrees|plans/\\.gitignore|WORKTREE PATH|BRANCH|workspace-local|nested repos" README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` exits 0.
- [ ] `git diff --check -- README.md skills/improve/SKILL.md skills/improve/references/closing-the-loop.md` exits 0.
- [ ] No source files outside `README.md`, `skills/improve/SKILL.md`, and
  `skills/improve/references/closing-the-loop.md` are modified. The only
  allowed non-source change is the `plans/README.md` status row, if applicable.

## STOP conditions

Stop and report back (do not improvise) if:

- The code at the locations in "Current state" does not match the excerpts
  after the drift check.
- The host-agent execute API documented in this repo explicitly cannot choose a
  worktree path and also cannot run an executor from a manually created git
  worktree.
- The change appears to require editing files outside the in-scope source docs.
- You find an existing issue, PR, or local branch in this repo that already
  implements the same worktree-location policy differently; report the conflict
  instead of creating a competing policy.
- The maintainer explicitly decides the skill should edit root `.gitignore`
  instead of `plans/.gitignore`; that is a product decision outside this plan.

## Maintenance notes

- Future changes to execute-mode dispatch must keep three things in sync:
  default path, ignore rule, and verdict/report fields. Reviewers should reject
  execute docs that mention one without the others.
- If this skill later gains executable code instead of Markdown-only procedures,
  port the same policy into code and keep `closing-the-loop.md` as the behavior
  spec.
- Cleanup policy is intentionally not expanded here beyond discoverability in
  `plans/.worktrees/`; a future follow-up can define automatic pruning if users
  ask for it.