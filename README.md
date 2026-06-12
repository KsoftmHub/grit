# GRIT

**Git Rules for Intelligent Tools** — a tiny Markdown framework that gives AI agents git discipline.

GRIT is one file. No installation, no dependencies, no package, no runtime. Just a Markdown contract that any agent (Claude Code, Codex, Cursor, OpenCode, Agent Zero, Gemini CLI, …) reads and obeys before touching git — in any language, any stack, any project.

Inspired by [DOX](https://github.com/agent0ai/dox) by Agent Zero. DOX teaches agents to maintain project *documentation*; GRIT teaches agents to maintain project *version control*. They work great together.

## How GRIT works

The agent treats `GRIT.md` as a binding contract for every git operation:

- the **default branch is read-only** to the agent — no commits, merges, rebases, force-pushes, renames, or deletions, ever, without explicit approval
- every commit is born on a **dedicated work branch** with a universal name format: `{type}/{task-slug}_{project_code}` (e.g. `feat/inbox-realtime_hmail`)
- commits follow **Conventional Commits** (`feat:` `fix:` `chore:` `docs:` `refactor:` …), one logical change per commit
- if a **remote exists**, work branches are pushed automatically; the default branch is only updated through a PR or a user-approved merge
- the file carries its own **memory**: a `## State` block records the remote URL, the detected default branch, the project code, and the policies — so the agent never guesses twice
- every task is bracketed by **Read Before Editing** and **Update After Editing** passes — branch, sync, and leftover-work checks before any file is touched; commit, push, and report before the task counts as done

## How to use

1. Copy [`GRIT.md`](GRIT.md) into your project root.
2. If you already have an `AGENTS.md` / `CLAUDE.md`, add one line to it:

   ```
   Git operations are governed by GRIT.md in the project root. Read it and obey it before any git action.
   ```

   (Or paste the whole contents of GRIT.md directly into your AGENTS.md as a section — both work.)
3. Tell your agent: **`grit init`**

The agent will then:

- check whether a remote URL is available and record it
- find the default branch (remote HEAD → `git remote show origin` → `ls-remote` → local heuristic) and **ask you if it can't** — it never guesses silently, and you can correct the value any time
- propose a 2–6 character project short code and confirm it with you
- confirm the branch name format with real examples
- write everything into the `## State` block and flip `status` to `ready`

From then on, every branch, commit, and push follows the contract automatically.

## What GRIT adds over a plain instruction file

| Plain "git rules" prose | GRIT |
|---|---|
| Rules the agent may interpret loosely | RFC-2119 style keywords: MUST / NEVER / SHOULD / ASK |
| "Use main or master" guesswork | A 5-step **detection ladder** with "ask the user and record it" as the final fallback |
| Rules forgotten between sessions | **Self-recording State block** — the file is the memory |
| Naming "conventions" | Exact format `{type}/{task-slug}_{project_code}` plus a **validation regex** for branches and commit subjects |
| Hope the agent behaves | A **CHECK pre-flight** before every commit/push (right branch, right name, one logical change, no secrets) |
| Panic when something goes wrong | A **RECOVER** section with self-healing steps for the four most common mistakes |
| Implicit danger zones | An explicit **Forbidden** list of hard stops (force-push, history rewrite, `--no-verify`, destroying unmerged work, …) |
| Silent drift | **RECONFIGURE** and **Closeout** procedures keep the recorded state and the report honest |
| Edits start anywhere, end with loose files | **Read/Update passes** bracket every task: right branch and synced before editing, committed and pushed after |

## Configuration

All knobs live in the `## State` block of `GRIT.md`:

| Key | Values | Meaning |
|---|---|---|
| `default_branch` | branch name | Detected or user-confirmed. Update it yourself any time the repo changes |
| `project_code` | 2–6 lowercase chars | The suffix in every branch name |
| `branch_granularity` | `per-task` (default) / `per-commit` | One branch per unit of work, or strictly one branch per commit |
| `push_branches` | `auto` (default) / `manual` | Push the work branch after each commit when a remote exists |
| `default_branch_updates` | `pr-only` (default) / `merge-on-approval` | How work is allowed to land on the default branch |

## Changelog

- **1.1** — Read Before Editing & Update After Editing: DOX-style pre/post passes, applied to git. Before any file is touched the agent verifies branch, sync state, and leftover work; after editing, every meaningful change must be committed, pushed, and reported before the task is done.
- **1.0** — Initial contract: protected default branch, Conventional Commits, universal branch format, INIT detection ladder, self-recording State, CHECK, RECOVER, Forbidden list.

## Credits

Created by **Ksoftm**.
Concept inspired by [agent0ai/dox](https://github.com/agent0ai/dox) — Self-documenting AGENTS.md by [Agent Zero](https://www.agent-zero.ai/).

## License

MIT
