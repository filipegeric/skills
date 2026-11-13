---
name: simplify
description: Interactive quality review of a PR or uncommitted changes. Finds reuse, simplification, efficiency, altitude, and excess test/comment cleanups (not bugs - use /review or /code-review for those), then walks through them one at a time, letting the user choose per finding - skip, fix, or done (already fixed it themselves). Use when the user asks to simplify, clean up, or tidy a PR, a branch, or their local changes interactively.
---

# Interactive Simplify

Review a diff for **code quality** - reuse, simplification, efficiency, altitude, excess tests and comments - and present findings to the user **one at a time** as interactive questions. Never dump all findings as a wall of text.

This is not a bug hunt. Do not look for correctness issues, regressions, or security problems; that is what `/review` and `/code-review` are for. If you stumble on an obvious bug while reading, mention it in one line in the final summary and move on.

## Step 1 — Determine what to review

Parse the argument (`$ARGUMENTS` if provided):

- **A PR number or URL** (e.g. `123`, `#123`, a GitHub PR URL): use `gh pr view <n>` for title/description and `gh pr diff <n>` for the diff. Check out the branch only if you need to run something or read surrounding context beyond what the diff shows (prefer reading files at the PR's head via `gh` or a checkout).
- **A branch name**: diff it against the repo's default branch (`git diff <default-branch>...<branch>`).
- **A file path**: review that file's changes (`git diff HEAD -- <path>`, or the whole file if it is untracked).
- **No argument**: review uncommitted changes — `git diff HEAD` plus untracked files (`git status --porcelain`; read new untracked source files in full). If there are no uncommitted changes at all, fall back to the current branch's diff against the default branch and tell the user that's what you're reviewing. If that's empty too, say there's nothing to review and stop.

Always read enough surrounding code (the full changed files, callers/callees, nearby shared/utility modules) to judge changes in context, not just the diff hunks. Reuse findings in particular depend on knowing what helpers the codebase already has — Grep for them.

## Step 2 — Build the findings list

Analyze the diff from five angles. For a large diff (roughly more than 10 changed files), launch 5 agents via the Agent tool in a single message, one per angle, each given the diff and its angle, and merge their results; otherwise do all five passes yourself.

1. **Reuse**: new code that re-implements something the codebase already has — a helper, a utility, an existing abstraction. Grep shared/utility modules and files adjacent to the change. The finding must name the existing thing to call instead.
2. **Simplification**: unnecessary complexity the diff adds — redundant or derivable state, copy-paste with slight variation, deep nesting, dead code left behind, overly clever constructs with a plainer equivalent. The finding must name the simpler form that does the same job.
3. **Efficiency**: wasted work the diff introduces — redundant computation, repeated I/O, independent operations run sequentially, blocking work added to startup or hot paths. Also long-lived objects built from closures or captured environments that keep the whole enclosing scope alive; prefer a class/struct that copies only the fields it needs. The finding must name the cheaper alternative.
4. **Altitude**: changes that patch a symptom instead of fixing the root cause at the right depth. Special cases layered on shared infrastructure are the usual sign. The finding must name the simpler, more general change to the underlying mechanism.
5. **Excess**: tests and comments the diff adds that don't earn their upkeep. Tests: ones that exercise a language or library guarantee rather than the code's own logic, cover an input the code cannot treat differently from an already-tested one, or only made sense for an earlier design. Prefer merging into a neighbouring test over deleting when a real assertion would be lost. Comments: ones that restate the code, repeat an explanation given elsewhere in the diff, or narrate how the code got here. Keep comments that explain a non-obvious why. The finding must name what to cut or merge and what coverage, if any, is lost.

Dedup findings that point at the same line or mechanism; keep the one with the clearest fix. Order them by payoff: how much duplication, waste, or maintenance burden each removes, weighted by confidence. Only include findings you can defend with a concrete cost (what is duplicated, wasted, or harder to maintain) and a concrete alternative — no speculative "you might consider" filler. If the code is clean, a short list (or none) is the correct output.

Drop any finding whose fix would change intended behavior or require changes well outside the reviewed diff. Note those in the final summary as "out of scope" rather than presenting them. One exception: when cutting an excess test leaves production code that only that test pinned, present deleting that code as a separate fix option, labelled as behavior-changing and stating what changes.

## Step 3 — Present findings one by one

First, give the user a one-paragraph overview: what was reviewed and a count per angle (e.g. "Reviewed uncommitted changes (6 files). Found 1 reuse, 3 simplification, 1 efficiency, 1 altitude, 2 excess cleanups.").

**Scope check**: if there are more than 5 findings total, ask one upfront `AskUserQuestion` before walking through them: go through everything, only reuse + altitude (the structural ones), or only the top 5 by payoff. Then honor that choice; findings excluded by it still get one line each in the final summary.

Then, for **each finding in payoff order**, use the `AskUserQuestion` tool — one call per finding, never batched:

- **Re-validate first**: immediately before presenting a finding, re-read the relevant code. Earlier fixes (yours or the user's) may have shifted line numbers or already resolved it. Update the location and snippet, or silently drop the finding if it no longer applies (note it as "resolved by earlier fix" in the summary).
- In the `question` text, explain the finding fully: what the code does now, where it is (`file:line`), what the concrete cost is (what is duplicated, wasted, or harder to maintain), and what the suggested change is. For reuse findings, name the existing helper and where it lives.
- `header`: max 12 characters — use compact labels like "Reuse 1/1", "Simpl 2/3", "Effic 1/1", "Alt 1/1".
- Options (max 4 — the tool's limit; "Other" is added automatically):
  - When there is genuinely more than one reasonable way to do it (e.g. call the existing helper vs. extract a new shared one, inline the special case vs. generalize the mechanism), offer **two fix options**: the recommended one **first**, its label ending in "(Recommended)", each label naming the approach in a few words (e.g. "Use existing formatDate (Recommended)", "Extract shared helper"). Then **Skip** and **Done**. Use each option's `description` to state the trade-off in one line.
  - When there is only one sensible fix, keep it to three options: **Fix**, **Skip**, **Done**.
  - Only offer alternatives when they are real, defensible choices with different trade-offs — never pad with a strawman second fix just to have two.
  - **Skip** — "Leave as is, move to the next finding." **Done** — "I already fixed this myself, move on."
- **Code snippets go in `preview`**, not crammed into the question text: whenever there is relevant code (almost always), set the `preview` field on the options so the snippet renders in the side panel. On each **fix option**, preview that option's resulting code (or a small before/after) — with two fix options this lets the user compare the alternatives side-by-side by moving focus between them. On **Skip** and **Done**, preview the current code with its `file:line`. Keep previews short and focused — the relevant lines plus minimal context, not whole functions. Only omit previews for the rare finding with no concrete code location.

Handle the answer before presenting the next finding:

- **Any fix option**: apply the chosen fix with Edit/Write immediately, keeping the change minimal, behavior-preserving (unless the option was labelled behavior-changing), and consistent with surrounding code style. Briefly confirm what was changed, then present the next finding. Do not commit or push unless the user asks.
  - For a PR review where fixes require checking out the PR branch: **check `git status` first**. If the working tree has uncommitted changes, do NOT check out — stop and ask the user how to proceed (stash their changes, work in a separate git worktree, or continue review-only without applying fixes).
- **Skip**: move on without changes.
- **Done**: re-read the relevant spot; if their change looks good, move on. If it doesn't actually address the finding, say so briefly and ask once more (same options) before moving on.

If the user picks "Other" and types something, treat it as instructions for that finding (e.g. a different approach) and act on it.

## Step 4 — Verify and wrap up

If any fixes were applied (by you or reported via "Done"), run the project's checks **once** after the last finding — not after every fix. Prefer the narrowest useful command: the tests covering the touched files if identifiable, otherwise the project's standard test/build/lint command (from CLAUDE.md, package.json scripts, Makefile, etc.). Apart from options labelled behavior-changing, simplifications must not change behavior, so a failing test after a fix means the fix is wrong — fix or revert it, and report the result either way. If no test command is discoverable, say so and name the files worth testing manually.

Then print a short summary table: each finding, its angle, and the outcome (fixed by Claude / fixed by user / skipped / resolved by earlier fix / excluded by scope choice / out of scope). If any fixes were applied, remind the user they are uncommitted and list the files touched. Do not commit anything unless asked.
