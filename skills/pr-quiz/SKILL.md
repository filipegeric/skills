---
name: pr-quiz
description: Interactive quiz that tests how well the user understands a PR, a branch, or their uncommitted changes. Reads the diff and surrounding code, builds questions about purpose, mechanism, behavior changes, edge cases and risks, asks them one at a time, grades each answer with evidence from the code, and ends with a score and a list of gaps to re-read. Use whenever the user wants to check, test, verify or quiz their understanding of a change - before reviewing someone else's PR, before approving, before a walkthrough or demo, or after writing a PR with heavy AI assistance. Trigger on phrases like "quiz me", "test my understanding", "do I actually get this PR", "check if I understand this diff", even if they don't say "quiz".
---

# PR Quiz

Turn a diff into a short oral exam. The goal is to find out whether the user could defend or review this change with confidence, and to point them at exactly what they missed. It is open book: the user may read the code while answering, because the test is reasoning about the change, not recall. Ask questions **one at a time**, grade each answer before moving on, and never reveal the answer to a question that hasn't been asked yet.

## Step 1 — Determine what to quiz on

Parse `$ARGUMENTS`. It may contain a target, a depth word, or both (e.g. `123 deep`, `quick`, `feature/x hard`).

**Target** (same resolution as `/review`):

- **A PR number or URL**: `gh pr view <n>` for title/body, `gh pr diff <n>` for the diff. Do not check out the branch. For surrounding context, read files at the PR head: `gh pr view <n> --json headRefOid,headRepository` then `git fetch origin <sha>` and `git show <sha>:<path>`, or `gh api repos/{owner}/{repo}/contents/<path>?ref=<sha>` if the repo isn't local.
- **A branch name**: `git diff <default-branch>...<branch>`.
- **No target**: uncommitted changes (`git diff HEAD` plus untracked files from `git status --porcelain`). If there are none, fall back to the current branch vs. the default branch and say so. If that's empty too, say there's nothing to quiz on and stop.

**Depth** (default `standard`):

| word | questions | focus |
|---|---|---|
| `quick` | 4 | purpose + the one or two most consequential changes |
| `standard` | 7 | purpose, mechanism, behavior change, edge case, risk, tests |
| `deep` | 10 | everything above, plus cross-file interactions and "what if we removed this line" |
| `hard` | same count as standard/deep | skip the easy purpose questions; every question needs tracing, not recall |

If no depth word is given and the diff is large (more than ~15 files or ~800 changed lines), ask one upfront `AskUserQuestion`: quiz the whole PR at standard depth, or focus on the 3-5 most consequential files (list them). Otherwise just start.

## Step 2 — Understand the change yourself first

You cannot grade answers you haven't verified. Before writing any question, read the full diff and enough surrounding code to answer these for yourself, with `file:line` evidence:

1. **Purpose**: what problem does this solve, and what would be broken or missing without it? Use the PR body, linked issues and commit messages, but confirm against the code; descriptions lie.
2. **Mechanism**: the 2-5 changes that actually carry the PR. Most diffs are one real change plus plumbing (renames, imports, tests, formatting). Find the real one.
3. **Behavior delta**: for each real change, what did the old code do in the cases the new code handles differently? Read the pre-image (the `-` lines and the untouched code around them), not just the `+` lines. When the repo is local, `git log -L` or blame on the touched function helps; when it isn't, the diff pre-image plus the head-version file from `gh api` is enough. Also read the commit list (`gh pr view <n> --json commits`): a late "fix" commit often means the description no longer matches the code.
4. **Edge cases and failure modes**: nulls, empty inputs, concurrency, errors swallowed or newly thrown, defaults changed, ordering assumptions.
5. **Blast radius**: who calls the changed code? Does anything outside the diff depend on the old behavior?
6. **Tests**: which new or changed tests pin which behavior, and what is left untested?

Write these down privately (in your reasoning, not to the user). They are your answer key. When a claim in it depends on arithmetic or a boundary condition (rounding at a threshold, an off-by-one, whether a guard is reachable), verify it by running a throwaway script in the scratchpad directory rather than trusting mental math; you will be grading someone on it. Never edit the repository itself.

## Step 3 — Build the question set

Draft all questions before asking the first one, so the set is balanced and you don't drift into trivia. Open with the easiest question and close with the hardest; in between, alternate multiple choice and open-ended rather than sorting strictly by difficulty, so the user isn't typing four essays in a row. When the focus list has fewer items than the question count, fill with extra thought-experiment questions; they carry the most signal per question.

### What makes a good question

A good question is one a strong reviewer would need to answer before approving. Test understanding, not memory:

- **Purpose**: "What goes wrong for a user if this PR is reverted?" rather than "What does the PR title say?"
- **Mechanism**: "After this change, what happens when `X` is called with an empty list?" with the trace as the answer.
- **Before vs. after**: "Before this PR, what did `foo()` return when the cache was cold? And now?"
- **Thought experiment**: "If line N of `file` were removed, what would break, and how would you notice?" This is the single best question type: it forces the user to know *why* a line exists.
- **Risk**: "Which change here is most likely to affect code outside this diff, and why?"
- **Tests**: "Which test would fail if the new null check were deleted? If none, what's untested?"
- **Cross-file** (deep only): "The new field in `A` is consumed where? What happens if the consumer sees the old shape?"
- **Description vs. code**: when the PR body or a comment describes behavior the code doesn't have (stale after a later commit, or just wrong), that mismatch is one of the best questions you can ask, because a reviewer who trusts the body will approve the wrong thing.
- **Scope**: "What does this PR deliberately *not* change?" (e.g. the JSON output, the other call site). On a well-described PR this is a better opener than a purpose question, because the purpose is already in the body.

Avoid anything a `grep` would answer: variable names, line counts, file names, the number of tests added. Avoid questions whose answer is in the PR title or description verbatim; if the body is thorough, swap the purpose question for a scope question.

### Multiple choice vs. open-ended

Use roughly **half and half**.

- **Multiple choice** (via `AskUserQuestion`) suits before/after, "which line", "which test" and "what happens when" questions. Write 3-4 options where every distractor is a *plausible misreading* of the code: the old behavior, an off-by-one, a swapped branch, a reasonable-but-wrong assumption. If you can't think of a plausible distractor, the question is too easy or should be open-ended. Vary the position of the correct answer; do not put it first by habit.
- **Open-ended** suits purpose, thought-experiment and risk questions. Ask in plain text and end your turn so the user types freely. Say how long an answer you expect ("one or two sentences").

### Using `AskUserQuestion` for multiple choice

- `header`: "Q 3/7" style, max 12 characters.
- `question`: the full question. Reference locations as `file:line` so the user can look if they want; looking is allowed, this is open book. The test is whether they can reason about the code, not whether they memorized it.
- `options`: 3-4 answers. Never label one "(Recommended)". Keep labels short; put any elaboration in `description`.
- `preview`: put the relevant code snippet on **every** option, identical across options, so the user sees the code while choosing. Keep it to the lines that matter plus minimal context. If the question is about before vs. after, show both hunks. If the answer depends on code in two or three places, stitch them into one snippet with a `// path:line` comment above each piece.

## Step 4 — Ask, grade, repeat

For each question:

1. Ask it (tool call for multiple choice, plain text and end turn for open-ended).
2. When the answer arrives, grade it in **at most four lines**: verdict (correct / partially correct / incorrect), the right answer in one sentence, and the `file:line` evidence. For a partially correct answer, name the missing piece. For an incorrect one, say what the user's answer would have implied and why the code doesn't do that.
3. Treat "I don't know" or "skip" as incorrect, grade it the same way, and move on without comment.
4. Move to the next question. Do not preview or hint at later questions.

Grade open-ended answers on substance, not phrasing. If the user names the right mechanism in different words, that's correct. If they're right but for the wrong reason, that's partial, and say which part was wrong.

If the user asks a question back ("wait, does X call Y?"), answer it briefly with evidence, then continue; curiosity is the point. If the user's answer reveals the *code* is wrong rather than their understanding (they expected the sensible behavior and the code does something else), say so plainly and count the question as correct. Likewise, "nothing happens, that line is a no-op" is a legitimate answer to a thought experiment and is fully correct when the user can say why it is unreachable or redundant; it is partial if they assert it without the reasoning.

If the user wants to stop early, skip to Step 5 with the questions answered so far.

## Step 5 — Score and point at gaps

After the last question, print:

1. **A score line**: correct / partial / incorrect out of N.
2. **A table**: question number, the topic in a few words, the result.
3. **Gaps**: for every partial or incorrect answer, one bullet with the `file:line` to re-read and one sentence on what to look for. This is the most useful output of the whole exercise, so make it specific. Skip this section if everything was correct.
4. **A one-line verdict** on readiness: for a PR the user is reviewing, whether they understand it well enough to approve or push back; for their own PR, whether they could defend it in review. Be honest. A 3/7 is a 3/7.

Then offer, in one line, to re-quiz only on the gaps or to go a level deeper.

Never modify code during a quiz. If grading surfaces a real bug, note it in the final summary under a "Noticed along the way" line and suggest `/review`; don't fix it.
