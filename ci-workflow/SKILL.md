---
name: ci-workflow
description: Automated worktree + PR + CI workflow for shipping code through GitHub Actions. Creates feature branch in git worktree, commits, pushes, creates PR, watches CI, and merges — all automated. Use when implementing stories or shipping any code changes that need CI validation. IMPORTANT — always use this skill when the user mentions shipping code, creating PRs, running CI, implementing stories, or pushing changes. Even partial requests like "push this" or "open a PR" should trigger this workflow.
allowed-tools:
  - "Bash"
  - "Read"
  - "Edit"
  - "Write"
  - "Glob"
metadata:
  version: "3.0.0"
  domain: ci-cd
  triggers: ship code, create pr, run ci, worktree, ci workflow, open pr, merge pr, push changes, implement story, dev story
  role: automation
  scope: git-workflow
  output-format: text
---

# CI Workflow — Worktree + PR + GitHub Actions

## STOP — Read This First

Every code change shipped through this workflow MUST happen inside a git worktree — never on the main branch. The reason is simple: the main branch is the user's working copy, and editing it directly while also trying to create a PR from it leads to conflicts, lost local changes, and broken state. The worktree gives you an isolated copy where you can commit, push, and iterate on CI failures without touching main.

**If you are about to edit a source file and you have not yet created a worktree, STOP. Go back and create the worktree first.** No exceptions — not "just this one file," not "I'll move it later." Create the worktree, then edit files inside it.

## Detect Project Context

Before anything else, detect the project context so you never need to ask the user for repo info:

```bash
# Project root (use this instead of hardcoded paths)
PROJECT_ROOT=$(git rev-parse --show-toplevel)

# Repo owner/name (e.g., "octocat/my-project")
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

# Default branch (usually "main" but not always)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
```

If `gh` fails with an auth error, tell the user to run `gh auth login` — do not try to source shell configs or set tokens yourself.

## Worktree Setup

Before writing any code, create a worktree with a feature branch. Worktrees live OUTSIDE the project directory — one level up from the working directory — to keep them cleanly separated:

```bash
git worktree add ../worktrees/<name> -b <branch-name>
```

Use descriptive names matching the task (e.g., `feat/user-auth`, `fix/login-crash`). From this point forward, every file edit targets the worktree path, not the project root.

Double-check: are you editing a path under `../worktrees/<name>/`? If not, you are editing the wrong copy.

## Understand the CI Pipeline

Before creating a PR, read the CI configuration so you know what checks will run:

```bash
ls ../worktrees/<name>/.github/workflows/
```

Read the workflow files to understand triggers, steps, and timeouts. This tells you what the PR needs to pass — build, test, lint, etc.

## Implement & Commit

Edit files directly in the worktree using the Edit tool on worktree paths. Never use `cp` between worktree and main — it hangs on interactive overwrite prompts.

Commit from the worktree directory. Keep commits focused and conventional (`feat:`, `fix:`, `test:`, `refactor:`):

```bash
cd ../worktrees/<name>
git add <files>
git commit -m "feat: description"
git push -u origin <branch-name>
```

## Create PR

```bash
cd ../worktrees/<name> && gh pr create \
  --title "feat: short description" \
  --body "$(cat <<'EOF'
## Summary
- Bullet points of changes

## Test plan
- [ ] CI passes

EOF
)"
```

## Watch CI

```bash
# Find the latest run on this branch
gh run list --branch <branch-name> --limit 1

# Watch until it completes (exit code reflects pass/fail)
gh run watch <run-id> --exit-status
```

Do NOT use `gh pr checks --watch` — it requires `checks:read` scope which fine-grained PATs don't support. Use `gh run watch` instead (Actions API).

If CI fails: read the failure with `gh run view <run-id> --log-failed`, fix in the worktree, commit, push, and watch the new run.

## Merge — REQUIRES EXPLICIT USER APPROVAL

Never merge automatically. Always stop and ask the user for permission first. The user needs to confirm that reviews are complete before merging.

Before merging, check if the repo has a preferred merge strategy (look at CONTRIBUTING.md, branch protection rules, or prior merge commits). Default to `--squash` if no preference is found — it keeps one clean commit per PR:

```bash
# Only after the user explicitly says "merge"
cd ../worktrees/<name> && gh pr merge --squash --delete-branch
```

## Cleanup

After merge, return to the project root and clean up:

```bash
# Restore files in main that overlap with the merged PR
git restore <files-that-were-also-edited-locally>
rm -rf <untracked-files-that-were-in-pr>

# Pull merged changes
git pull

# Remove the worktree
git worktree remove ../worktrees/<name>
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `gh: command not found` | Install GitHub CLI: <https://cli.github.com> |
| `gh` auth errors | Run `gh auth login` to authenticate |
| `gh pr checks --watch` fails | Don't use it. Use `gh run watch <id> --exit-status` instead |
| CI not triggering | Check workflow triggers — some repos use `paths-ignore` filters |
