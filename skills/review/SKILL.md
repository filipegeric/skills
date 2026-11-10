---
name: review
description: Interactive code review of a PR or uncommitted changes. Finds issues prioritized as regressions > correctness > style > everything else, then walks through them one at a time, letting the user choose per finding - skip, fix, or done (already fixed it themselves). Use when the user asks to review a PR, a branch, or their local changes interactively.
---

# Interactive Code Review

Review a diff and present findings to the user **one at a time** as interactive questions. Never dump all findings as a wall of text.

## Step 1 — Determine what to review

Parse the argument (`$ARGUMENTS` if provided):

- **A PR number or URL** (e.g. `123`, `#123`, a GitHub PR URL): use `gh pr view <n>` for title/description and `gh pr diff <n>` for the diff. Check out the branch only if you need to run something or read surrounding context beyond what the diff shows (prefer reading files at the PR's head via `gh` or a checkout).
- **A branch name**: diff it against the repo's default branch (`git diff <default-branch>...<branch>`).
- **No argument**: review uncommitted changes — `git diff HEAD` plus untracked files (`git status --porcelain`; read new untracked source files in full). If there are no uncommitted changes at all, fall back to the current branch's diff against the default branch and tell the user that's what you're reviewing. If that's empty too, say there's nothing to review and stop.

Always read enough surrounding code (the full changed files, callers/callees where relevant) to judge changes in context, not just the diff hunks.

## Step 2 — Build the findings list

Analyze the diff and collect findings, ordered by priority class:

1. **Regressions** (highest priority): behavior that worked before this change and is now broken or silently altered — removed handling of an existing case, changed semantics of an existing function that callers rely on, dropped error handling, broken backwards compatibility, changed defaults, removed validation. Compare old vs. new behavior explicitly; use `git log`/blame and callers to understand what the old code guaranteed.
2. **Correctness**: bugs in the new code itself — logic errors, off-by-ones, null/undefined handling, race conditions, unhandled error paths, wrong types, security issues, edge cases the new code misses.
3. **Code style**: naming, dead code, duplication, inconsistency with the surrounding codebase's idioms, missing/misleading comments, overly complex constructs that have a simpler equivalent.
4. **Everything else**: performance, test coverage gaps, docs, architectural suggestions, nitpicks.

Within each class, order by severity/confidence. Only include findings you can defend with a concrete failure scenario or a clear benefit — no speculative "you might consider" filler. If the code is clean, a short list (or none) is the correct output.

## Step 3 — Present findings one by one

First, give the user a one-paragraph overview: what was reviewed and a count per category (e.g. "Reviewed PR #123 (14 files). Found 2 potential regressions, 3 correctness issues, 4 style points.").

**Scope check**: if there are more than 5 findings total, ask one upfront `AskUserQuestion` before walking through them: review everything, only regressions + correctness, or only regressions. Then honor that choice; findings excluded by it still get one line each in the final summary.

Then, for **each finding in priority order**, use the `AskUserQuestion` tool — one call per finding, never batched:

- **Re-validate first**: immediately before presenting a finding, re-read the relevant code. Earlier fixes (yours or the user's) may have shifted line numbers or already resolved it. Update the location and snippet, or silently drop the finding if it no longer applies (note it as "resolved by earlier fix" in the summary).
- In the `question` text, explain the finding fully: what the issue is, where it is (`file:line`), why it matters (for regressions: what the old behavior was and what breaks), and what the suggested change is.
- `header`: max 12 characters — use compact labels like "Reg 1/2", "Bug 2/3", "Style 1/4", "Other 1/2".
- Options (max 4 — the tool's limit; "Other" is added automatically):
  - When there is genuinely more than one reasonable way to fix it (e.g. patch in place vs. restructure, quick guard vs. proper handling), offer **two fix options**: the recommended one **first**, its label ending in "(Recommended)", each label naming the approach in a few words (e.g. "Add null guard (Recommended)", "Make field required"). Then **Skip** and **Done**. Use each option's `description` to state the trade-off in one line.
  - When there is only one sensible fix, keep it to three options: **Fix**, **Skip**, **Done**.
  - Only offer alternatives when they are real, defensible choices with different trade-offs — never pad with a strawman second fix just to have two.
  - **Skip** — "Leave as is, move to the next finding." **Done** — "I already fixed this myself, move on."
- **Code snippets go in `preview`**, not crammed into the question text: whenever there is relevant code (almost always), set the `preview` field on the options so the snippet renders in the side panel. On each **fix option**, preview that option's resulting code (or a small before/after) — with two fix options this lets the user compare the alternatives side-by-side by moving focus between them. On **Skip** and **Done**, preview the current problematic code with its `file:line`. Keep previews short and focused — the offending lines plus minimal context, not whole functions. Only omit previews for the rare finding with no concrete code location (e.g. "this PR has no tests").

Handle the answer before presenting the next finding:

- **Any fix option**: apply the chosen fix with Edit/Write immediately, keeping the change minimal and consistent with surrounding code style. Briefly confirm what was changed, then present the next finding. Do not commit or push unless the user asks.
  - For a PR review where fixes require checking out the PR branch: **check `git status` first**. If the working tree has uncommitted changes, do NOT check out — stop and ask the user how to proceed (stash their changes, work in a separate git worktree, or continue review-only without applying fixes).
- **Skip**: move on without changes.
- **Done**: re-read the relevant spot; if their fix looks good, move on. If it doesn't actually address the issue, say so briefly and ask once more (same options) before moving on.

If the user picks "Other" and types something, treat it as instructions for that finding (e.g. a different fix approach) and act on it.

## Step 4 — Verify and wrap up

If any fixes were applied (by you or reported via "Done"), run the project's checks **once** after the last finding — not after every fix. Prefer the narrowest useful command: the tests covering the touched files if identifiable, otherwise the project's standard test/build/lint command (from CLAUDE.md, package.json scripts, Makefile, etc.). If a check fails because of an applied fix, fix it; report the result either way. If no test command is discoverable, say so and name the files worth testing manually.

Then print a short summary table: each finding, its category, and the outcome (fixed by Claude / fixed by user / skipped / resolved by earlier fix / excluded by scope choice). If any fixes were applied, remind the user they are uncommitted and list the files touched. Do not commit anything unless asked.
