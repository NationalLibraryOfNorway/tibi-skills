---
name: human-code-review
description: Make sure the developer understands the code change before merging. Improve code that is hard to understand. Quiz the developer on the actual diff's behaviour, rationale, dependencies, and failure modes.
disable-model-invocation: true
---

# Human code review

Results in pass or fail. It exists to confirm the developer responsible for a change understands its behaviour, rationale, dependencies, and failure modes — not to test trivia, and not to punish. When a wrong answer traces back to code that's genuinely hard to follow, fix the code; don't just fail the human. Re-derive every question from the actual current diff — never reuse from a question bank, never ask about identifiers or line numbers.

## 1. Check for unstaged changes

If there are unstaged changes, ask the developer to either stage or discard them before proceeding with the review.

## 2. Gather context and identify the target change

Gather:

- full SHA of the current HEAD
- full SHA of the base commit, found as described under "Finding the base" below
- the diff (`git diff <base>..<head>`, so only this branch's own layer)
- surrounding code the diff touches
- relevant tests
- any architecture/domain docs for the area (treat them as supplementary evidence, every question must be traceable to the actual current code, not to what the docs say it should do)
- if there's a PR for the current branch, get its title and description (`gh pr view --json title,body`)
- if there's a ticket or issue, get its title and description

Don't require the code to build or CI to be green; comprehension of the diff as written doesn't depend on either.

### Finding the base

The base is the commit this branch's own work starts from. On a stacked branch (a branch built on top of another unmerged branch) that is the parent branch, not `main`. Reviewing against `main` would quiz the developer on the parent layers' changes too.

Try these in order and stop at the first that gives an answer. `gh stack` and `git` are read-only here; never run `gh stack` commands that change the stack (`rebase`, `sync`, `submit`, `unstack`) as part of a review.

1. **gh stack.** Run `gh stack view --json` (never bare `gh stack view`, it opens a TUI). Exit code 2, or a failing command, means the branch is not in a stack or `gh stack` is not installed; go to 2.

   ```bash
   gh stack view --json | jq -r --arg b "$(git branch --show-current)" '.branches[] | select(.name == $b) | .base'
   ```

   `base` is the SHA of the parent layer's tip that this branch was last built on (the trunk SHA for the bottom layer). It stays valid after the parent has been squash-merged or its local branch deleted, because the commit is still in this branch's history. If `needsRebase` is true for this branch, the parent was rewritten since; the base SHA still delimits this layer's commits, so tell the developer that the stack needs a `gh stack rebase` but continue. State the base and where it came from; no confirmation needed.

2. **Stacked branch without a gh stack.** Some branches are stacked without anyone having run `gh stack init`. List local and `origin/` branches whose tip is an ancestor of HEAD but not of the trunk (`origin/main`), nearest first:

   ```bash
   for b in $(git for-each-ref --format='%(refname:short)' refs/heads refs/remotes/origin); do
     [ "$(git rev-parse "$b")" = "$(git rev-parse HEAD)" ] && continue
     git merge-base --is-ancestor "$b" HEAD && ! git merge-base --is-ancestor "$b" origin/main &&
       echo "$(git rev-list --count "$b"..HEAD) $b"
   done | sort -n
   ```

   Branches sharing HEAD's commit or already part of `origin/main`'s history are not candidates. If there are candidates, ask the developer which to review against, nearest as the suggestion, `origin/main` last:

   > Which branch do you want to review against: <nearest> (suggested), <next>, …, origin/main?

   The base is `git merge-base HEAD <chosen>`.

   This finds a parent whose commits are still in this branch's history. It misses a parent that was amended or rebased after this branch was built (its new tip is no longer an ancestor). If the developer says the parent was rewritten, let them name the branch and use `git merge-base HEAD <branch>`.

3. **Not stacked.** With no candidates, use `git merge-base HEAD origin/main` and say so.

If `gh pr view --json baseRefName` names a branch other than the trunk, mention it when it disagrees with the base you found. Don't use it as the source: a PR's base is often left at `main` on stacked branches.

Before generating questions, show the developer the base (short SHA, source: gh stack / chosen branch / merge-base with origin/main), the number of commits and the changed-file count in `<base>..<head>`, and ask them to confirm. If the source was gh stack and nothing disagrees, this is a statement, not a question.

## 3. Map comprehension-critical areas

Before writing questions, scan the diff for what the developer must understand:

- control flow, business rules, invariants, and the design decisions behind them
- data flow, side effects, dependencies, and failure modes
- invalid/unexpected input, API contracts, transactions, concurrency, security implications
- what breaks if this logic is later modified or removed

Separately, flag comprehensibility risks in the code itself: ambiguous naming, deep nesting, hidden state, surprising mutation, implicit side effects, vague abstractions, error handling far from its trigger, unclear transaction or concurrency boundaries. These are candidates for code improvement, not for the quiz — a risk you flag here is a hint you're about to test the code's clarity, not just the developer's memory.

## 4. Generate the questions

Count: 3 for a small/simple change, 5 by default, up to 7 for a complex one.

Mix across these five kinds, grounding every one in the actual diff — include at least one prediction or failure-mode question whenever the change has meaningful behaviour of that kind:

1. **Behaviour** — what does the code do?
2. **Reasoning** — why this way?
3. **Failure mode** — what happens when something goes wrong?
4. **Dependency** — what else does this touch?
5. **Prediction** — given a concrete input/state, what happens?

## 5. Generating the options to an answer

1. First, generate exactly one objectively correct option.
2. Then, generate 3 plausible distractors. These must have about the same number of words and characters, and the same level of detail, so that the length or level of detail does not give it away. Use `echo $(($RANDOM % 2))` to decide if the option should be longer (output is "1") or shorter (output is "0") than the correct option.
3. Then, Randomize which letter (A–D) holds the correct answer for each question by running `echo $((1 + $RANDOM % 4))` in the shell (1=A, 2=B, 3=C, 4=D) and placing the correct answer in that slot — don't pick the slot yourself.
4. Then, assign the remaining letters (A-D) to the incorrect options.
5. Always include a fifth option (E) for "I don't know."

## 6. Before asking the questions

If there is a corresponding pull request, show a link to it.

Show this note once: "If you understand what happens but believe it is a bug, use (F) Bug. A justification is optional."

## 7. Ask the questions

One question at a time. Record the answer; allow an explicit "I don't know" — it counts as incorrect but is a real signal of uncertainty, not noise to discard. Also allow an explicit "Bug" - it counts as incorrect, but changes text in the summary.

Each question should be formatted like so:

```
Question <CURRENT> of <TOTAL>

❓ **<question title>**: <question body, might be multiple paragraphs>

(A) <option A>
(B) <option B>
(C) <option C>
(D) <option D>
(E) I don't know
(F) Bug (optional justification)
```

After an answer has been collected, say which option was chosen and whether it was correct.

If the answer was "(F) Bug", ask for a short justification if none was given, then assess whether this is likely a genuine bug, using any justification as supporting evidence. Explain the reasoning to the user. Either way, count it as a wrong answer for scoring purposes — but if it is likely a genuine bug, also record it as a confirmed bug.

## 8. Collect comprehension feedback

After each answer, collect comprehension feedback. Take free text if offered, but don't demand it.

```
❓ **<ask for comprehension feedback for the previous question>**

(A) I understand it well
(B) Unsure
(C) Hard to understand from the code
(D) The question itself was unclear
```

## 9. Diagnose weak understanding

Where answers are wrong or uncertainty clusters, decide whether the cause is (A) a knowledge gap, (B) unclear or needlessly complex code, or (C) an ambiguous question — possibly a mix. Don't default to (A). If (C), say so explicitly and don't treat that question as a reliable signal.

## 10. Improve code the questions exposed as hard to understand

When feedback or diagnosis points to (B): name the likely source of difficulty, explain it briefly, propose a clearer implementation, and — where the workflow allows — offer to apply it. Good targets: clearer names, flatter control flow, explicit state transitions and side effects, error handling moved closer to its trigger, clearer transaction boundaries, decomposed dense logic. The bar is "easier to predict correctly," not "longer."

## 11. Score

Pass threshold 70%, required-correct rounded up:

| questions | required |
| --------- | -------- |
| 3         | 2        |
| 4         | 3        |
| 5         | 4        |
| 6         | 5        |
| 7         | 5        |

Say: "<CORRECT> correct out of <TOTAL>."

## 12. Record the result

Only run this step if the score meets the pass threshold AND no confirmed bugs were found — a confirmed bug always blocks the understanding statement, even on a passing score.

On a pass, output a human understanding statement. Fill in your name, repo, full commit SHA of HEAD, full commit SHA of the base (the one confirmed in step 2, so a stacked branch's statement covers only its own layer).

Use the following format when the GitHub repo is known:

```
<!-----BEGIN HUMAN UNDERSTANDING STATEMENT----->
🖋️ <your name> understands this change: https://github.com/<repo>/compare/<base>...<head>
<!-----END HUMAN UNDERSTANDING STATEMENT----->
```

Use the following format if the GitHub repo is not known:

```text
<!-----BEGIN HUMAN UNDERSTANDING STATEMENT----->
🖋️ <your name> understands this change:
`git diff <base>..<head>`
<!-----END HUMAN UNDERSTANDING STATEMENT----->
```

## 13. Sending a comment to the PR

Offer to post the text as a comment to the PR of the reviewed branch (on a stack, that layer's own PR, not the bottom one). Only post the human understanding statement, not the score or failure details.

If this repo has a signature skill (e.g. `.agents/skills/signature`) for marking AI-generated PR/ticket text, append its signature line to the posted comment.

## 14. Handle failure

If one or more confirmed bugs were found, DISAPPROVAL_TEXT is "Too many errors in the code. Fix them and run human-code-review again." — this always applies when a confirmed bug exists, even if the score meets the pass threshold; a confirmed bug blocks the understanding statement regardless of score. Otherwise, if the score does not meet the pass threshold, DISAPPROVAL_TEXT is "You have not demonstrated that you understand this change."

No approval record. Show the score, name the weakest areas, surface the comprehension feedback gathered, and suggest reviewing and/or improving the flagged code before trying again:

```text
<DISAPPROVAL_TEXT>

Result: <CORRECT>/<TOTAL>

Weakest areas:
- <AREA>
- <AREA>

Review these parts of the change before trying again.
```

## Language

The text in this skill is English. Write questions, answers, statements and templates in the language the developer uses; if unclear, use English. Translate the templates and fixed phrases above (including the disapproval texts and the marker line of the understanding statement) into that language. Keep the `BEGIN`/`END HUMAN UNDERSTANDING STATEMENT` markers as they are.
