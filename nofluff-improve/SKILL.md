---
name: nofluff-improve
description: Improve the nofluff skill from accumulated real-world feedback. Reads FEEDBACK.md plus GitHub issues and PR comments on the nofluff repo, finds patterns that recur, and opens a pull request editing nofluff/SKILL.md. Use when the user invokes /nofluff-improve, asks to "improve nofluff", "update the nofluff skill from feedback", or says the feedback log has built up enough to act on.
---

# nofluff-improve — turn feedback into skill edits

This skill edits another skill. It reads what went wrong in real sessions and proposes
targeted changes to [`nofluff/SKILL.md`](../nofluff/SKILL.md), as a pull request you review.

It never edits `SKILL.md` on `main` directly. Every change lands through a PR, because a
bad edit to the voice rules degrades every reply in every session afterwards, silently.

Repo: `~/workspace_personal/nofluff`, remote `git@github.com-newaccount:anshuman13/nofluff.git`.
Git operations use SSH, which authenticates as the repo owner. Do not use `gh` to push or
open PRs — the local `gh` is authenticated as a different account with read-only access.
`gh api` for *reading* issues and comments is fine.

## The rule underneath

**Change the skill only where evidence says it is wrong.** The base skill is short on
purpose; a skill that accretes a clause per complaint becomes a document the model skims
instead of follows. Deleting or sharpening an existing rule beats appending a new one.

Two entries reporting the same failure is a pattern. One entry is an anecdote — log it,
leave the skill alone. The exception is a losslessness failure: nofluff dropping a fact,
caveat, number, or code block is a correctness bug in the skill's core promise, and one
credible report is enough to act on.

## Gather

Read all three sources before proposing anything.

1. **The feedback log.** `FEEDBACK.md` in the repo root. Skip entries marked `[applied]`.
2. **GitHub issues.** `gh api repos/anshuman13/nofluff/issues --paginate` (this returns
   PRs too; filter to entries without a `pull_request` key for issues alone).
3. **PR review comments.** `gh api repos/anshuman13/nofluff/pulls/comments --paginate`,
   and issue-style comments on PRs via
   `gh api repos/anshuman13/nofluff/issues/comments --paginate`.

If every source is empty, say so and stop. Do not invent improvements to justify the run.
A no-op run is the correct outcome when nothing has gone wrong.

Also diff the installed copy against the repo, since a stale install produces feedback
about bugs already fixed:

```bash
diff ~/workspace_personal/nofluff/nofluff/SKILL.md ~/.claude/skills/nofluff/SKILL.md
```

If they differ, report it. Feedback generated against a stale copy is evidence about the
old text, not the current one, and must be discounted accordingly.

## Weigh

Rank the evidence before writing anything. Warp's finding, which holds here: detailed
feedback from someone who knows the domain outweighs volume of generic feedback.

Weight **up**:

- Entries that quote actual output, so the failure is visible rather than described.
- Losslessness failures — a dropped caveat, number, path, or code block.
- Failures where the skill's own text told the model to do the wrong thing. These are
  bugs in the rules, not gaps in them.
- The same failure reported across different sessions or different kinds of task.

Weight **down**:

- "Too long" or "too short" with no example. Not actionable.
- One-off taste disagreements about a single word.
- Anything the current `SKILL.md` already covers, where the real fault is that the model
  ignored a rule that exists. Restating a rule the model already skipped rarely helps;
  consider whether the rule is buried, vague, or contradicted elsewhere in the file.

## Propose

For each pattern that clears the bar, write the change against `nofluff/SKILL.md`:

- **Prefer editing an existing rule** over adding one. Most feedback means a rule is
  vague, not missing.
- **Write principles, not special cases.** "Name the failure, not how bad it feels"
  generalizes; "don't say catastrophic" does not. A rule the model can reason from covers
  cases you have not seen yet.
- **Give the reason.** A rule that explains why it exists survives contact with a case its
  author did not anticipate. A bare prohibition does not.
- **Keep the cut list concrete.** Its value is that it is auditable — specific
  constructions the model can check itself against. Do not blur it into general advice.
- **Check for contradictions** against the rest of the file, including the anti-goals. The
  anti-goals outrank new rules: an edit that would drop a load-bearing caveat is wrong no
  matter how much feedback asked for shorter output.
- **Say what you are not changing.** If feedback did not clear the bar, list it and why.
  That list is the honest part of the report.

Keep the README in sync when a change alters documented behaviour. It describes the voice
rules and the usage, and a skill edit that contradicts it leaves the repo lying about
itself.

## Open the PR

Branch, commit, push over SSH, then hand back a compare URL. `gh pr create` will fail on
the read-only account — do not attempt it.

```bash
cd ~/workspace_personal/nofluff
git fetch origin main && git checkout -b nofluff-improve/<short-kebab-topic> FETCH_HEAD
# edit nofluff/SKILL.md (and README.md if behaviour changed)
git add -A && git commit -m "<single line, imperative, no attribution>"
git push -u origin HEAD
echo "https://github.com/anshuman13/nofluff/compare/<branch>?expand=1"
```

Never pass `origin/main` as the start point to `git checkout -b` — it silently sets the
upstream to `main` so a later `git push` targets `main` itself. `FETCH_HEAD` avoids this.

Commit messages are one line, imperative, no body and no attribution trailer.

After pushing, mark the entries you acted on in `FEEDBACK.md` with `[applied]` and the
commit sha, in the same PR. Otherwise the next run counts them again and proposes the
same edit.

## Report

Tell the user, briefly:

- Which patterns cleared the bar, and the change each one produced.
- Which feedback you deliberately left alone, and why.
- The compare URL.
- Whether the installed copy at `~/.claude/skills/nofluff/SKILL.md` is stale.

Merging is the user's call. The PR is a proposal, not a decision.

## Anti-goals

- **Do not grow the skill for its own sake.** A run whose honest output is "nothing
  cleared the bar" is a successful run. Length is the failure mode this whole project
  exists to fight; a bloated `SKILL.md` would be the joke writing itself.
- **Do not tune the skill toward shorter output.** It is lossless by design. Feedback
  asking for more aggressive cutting must be checked against the anti-goals first, and
  refused where it would cost a caveat.
- **Do not edit the installed copy** at `~/.claude/skills/nofluff/` as a shortcut. The
  repo is the source; installing is a separate, deliberate `cp`.
- **Do not rewrite feedback entries** to read better. They are evidence.
