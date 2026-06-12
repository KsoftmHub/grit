# GRIT framework

- GRIT (Git Rules for Intelligent Tools) is a git discipline contract installed here
- Any agent performing git operations in this repository MUST follow GRIT exactly
- GRIT's memory lives in the `## State` section below: read it before any git action, change it only through INIT or RECONFIGURE
- GRIT is tool-agnostic and language-agnostic: it works in any project, any stack, any agent that can read Markdown and run git

## Keywords

- MUST / NEVER — hard rule; breaking it means the task failed
- SHOULD — follow unless the user explicitly says otherwise
- ASK — stop and get user confirmation before continuing
- record — write the value into `## State`

## Core Contract

- The default branch is protected. NEVER commit on it, merge into it, rebase it, rename it, force-push it, or delete it without explicit user approval in the current session
- Every commit is created on a dedicated work branch — never on the default branch
- Commits follow Conventional Commits, one logical change per commit
- Work branches follow the recorded branch format exactly
- If `## State` shows `status: uninitialized`, run INIT before any write operation (branch, commit, merge, push)
- Do not rely on memory of previous sessions. Re-read this file and `## State` at the start of every session that touches git

## INIT

Run when the user says `init`, `grit init`, or `initialize grit` — and automatically before the first write operation in an uninitialized repository.

1. Verify a repository exists: `git rev-parse --is-inside-work-tree`. If not, ASK whether to run `git init`
2. Detect the remote: `git remote get-url origin`. record the URL, or `none` if absent. If multiple remotes exist, list them and ASK which one is canonical
3. Detect the default branch, trying each step in order until one succeeds:
   1. `git symbolic-ref --short refs/remotes/origin/HEAD` (strip the `origin/` prefix)
   2. `git remote show origin` → read the `HEAD branch:` line
   3. `git ls-remote --symref origin HEAD`
   4. If exactly one of `main`, `master`, `trunk`, `develop` exists locally, propose it and ASK to confirm
   5. Otherwise ASK the user to name the default branch — NEVER guess silently. The user can correct this value at any time
4. record the confirmed default branch
5. Propose a project code: 2–6 lowercase characters derived from the repo or remote name (example: `husk-mail` → `hmail`). ASK to confirm or replace, then record it
6. Show the branch format filled with two real examples and ASK to confirm:
   `{type}/{task-slug}_{project_code}` → `feat/inbox-realtime_hmail`, `fix/mx-check_hmail`
7. Set `status: ready` and report a one-screen summary of everything recorded

## State

<!-- Written by INIT. Agents change values here only via INIT or RECONFIGURE. Never edit the rules in this file unless the user asks. -->

```yaml
grit: 1
status: uninitialized            # uninitialized | ready
remote_url: null                 # e.g. git@github.com:ksoftm/husk-mail.git | none
default_branch: null             # e.g. main — never assumed, always detected or user-confirmed
project_code: null               # 2–6 lowercase chars, e.g. hmail
branch_format: "{type}/{task-slug}_{project_code}"
branch_granularity: per-task     # per-task | per-commit
push_branches: auto              # auto: push work branch after each commit when a remote exists | manual
default_branch_updates: pr-only  # pr-only | merge-on-approval
```

## Branches

- Allowed `{type}` values (identical to commit types): `feat` `fix` `chore` `docs` `refactor` `test` `perf` `ci` `build` `style` `revert`
- `{task-slug}`: the task in short — 2–4 words, lowercase kebab-case, characters `[a-z0-9-]`, max 30 chars
- `{project_code}`: the recorded value from `## State`
- Branch names MUST match:
  `^(feat|fix|chore|docs|refactor|test|perf|ci|build|style|revert)/[a-z0-9]+(-[a-z0-9]+)*_[a-z0-9]{2,6}$`
- Create every work branch from the up-to-date default branch:
  - with a remote: `git fetch origin && git switch -c <branch> origin/<default_branch>`
  - without a remote: `git switch <default_branch> && git switch -c <branch>`
- `branch_granularity: per-task` (default) — one branch per unit of work; related Conventional Commits may stack on it
- `branch_granularity: per-commit` — one branch per commit; switch only when the user asks for it
- NEVER reuse a merged branch. SHOULD delete branches after merge: `git branch -d <branch>`, and with approval `git push origin --delete <branch>`

## Commits

- Format: `type(scope)?: subject` — Conventional Commits
- Allowed types: `feat` `fix` `chore` `docs` `refactor` `test` `perf` `ci` `build` `style` `revert`
- Subject: imperative mood, lowercase start, no trailing period, ≤ 72 chars (aim for 50)
- One logical change per commit. Mixed concerns MUST be split (`git add -p`)
- Body (when the change is not self-evident): wrap at 72 chars, explain why, not what
- Footers when applicable: `BREAKING CHANGE: …`, `Closes #123`
- The subject line MUST match:
  `^(feat|fix|chore|docs|refactor|test|perf|ci|build|style|revert)(\([a-z0-9-]+\))?!?: .{1,72}$`
- Run CHECK before every commit

## Push & Sync

- If `remote_url` is set and `push_branches: auto`: after committing, push the work branch — `git push -u origin <branch>` on the first push, `git push` afterwards
- The default branch reaches the remote only through a pull request, or through a merge the user explicitly approved — never through a direct agent commit or push
- When the user approves landing the work: open a PR targeting the default branch; or, if `default_branch_updates: merge-on-approval` and the user instructs it, run `git switch <default_branch> && git pull && git merge --no-ff <branch> && git push`, then delete the work branch
- SHOULD `git fetch origin` before starting any new branch so work begins from the latest default

## CHECK

Pre-flight — run before every commit and push:

1. `status` in `## State` is `ready`; otherwise run INIT first
2. Current branch is NOT the default branch
3. Current branch name matches the branch regex
4. Staged diff is one logical change with no leftovers: `git diff --staged --stat`
5. No secrets staged: scan the staged diff for private keys, `.env` values, tokens, passwords. On any hit, STOP and ASK
6. The commit subject line matches the commit regex

## RECOVER

Self-healing for common mistakes:

- On the default branch with uncommitted changes → `git switch -c <proper-branch>` (the changes follow), then continue normally
- Committed on the default branch but not pushed → ASK, then `git branch <proper-branch> && git reset --hard origin/<default_branch>` (or the pre-commit SHA if no remote) `&& git switch <proper-branch>`
- Pushed to the default branch by mistake → STOP and report to the user. NEVER force-push a fix on your own
- Detached HEAD with work on it → `git switch -c <proper-branch>` immediately

## Forbidden

Hard stops — perform only if the user explicitly commands it in the current session:

- Any commit, merge, rebase, reset, or push on/onto the default branch
- `git push --force` (only `--force-with-lease`, only on your own work branch, only with approval)
- Rewriting history of any pushed commit (`commit --amend`, `rebase`) on shared branches
- `git branch -D` or any deletion of unmerged work
- `git reset --hard` or `git clean -fd` that would discard uncommitted work
- Changing remote URLs, git config, or hooks
- Bypassing hooks with `--no-verify`
- Committing secrets, credentials, or files larger than 50 MB

## RECONFIGURE

When the user changes the default branch name, project code, branch format, or any policy value: update only the affected values in `## State`, echo the new values back for confirmation, and follow them from the next operation onward.

## Closeout

After every task that touched git, report:

1. Working tree clean, or intentionally dirty (state which and why)
2. Work branch pushed, if `push_branches: auto` and a remote exists
3. Branch name, the list of commits made (`type: subject`), and PR/merge status
4. `## State` still accurate — update it if any recorded fact changed
