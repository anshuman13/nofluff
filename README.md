# llm-brevity

A [Claude Code](https://claude.com/claude-code) skill (`/llm-brevity`) that cuts LLM padding — either compressing a reply after the fact, or switching the session to terse output for good.

## The problem

Ask a model a yes/no question and you get a restatement of your question, a narration of what it's about to check, the answer, a summary of the answer, and an offer to help further. The answer was one word. Everything else is packaging.

This isn't a style preference — padding costs reading time on every single reply, and it buries the one sentence you needed. The skill's core rule is **compress by deleting, not rewriting**: every fact, number, file path, command, and code block survives. Only filler dies. A rewrite that loses content has failed, because that's summarization, and you asked for brevity.

## Usage

```
/llm-brevity [tier] [--on|--off] [text]
```

With no arguments it compresses the previous reply at the default tier. Paste text after the tier to compress that instead. It also triggers on plain requests like "shorten that" or "you're being too verbose."

| Tier | Target | What survives |
|------|--------|---------------|
| `tight` | ~50% | Everything. Pure filler removal, no content judgment. |
| `terse` (default) | ~25% | Every fact, number, path, and code block. Elaboration and transitions go. |
| `telegram` | 1–3 sentences | The answer alone, plus any command needed to act on it. |

`tight` and `terse` are lossless on content. `telegram` is the one tier allowed to drop facts.

### Standing mode

```
/llm-brevity --on     # terse output for the rest of the session
/llm-brevity --off    # back to normal
```

Standing mode applies the rules while writing rather than compressing afterward: answer in the first sentence, no tool-call narration, reply length matched to question complexity.

## Before / after

**Before:**

> Great question! You're asking about why the retry logic is failing. Let me take a look at the sync pipeline to understand what's happening here. So, after digging into this, I found that there are essentially three separate issues that are all contributing to the problem. First, and this is really the load-bearing issue, `syncQueue.ts:142` is swallowing the `ETIMEDOUT` error instead of re-queuing the job. Second, it's worth noting that the backoff caps out at 2 seconds, which is generally going to be too low for mobile networks. And third — here's the kicker — the dedupe key includes a timestamp, which means retries are never actually deduplicated. Let me know if you'd like me to dig deeper into any of these!

**After (`/llm-brevity`):**

> Three bugs in the retry logic:
>
> - `syncQueue.ts:142` swallows `ETIMEDOUT` instead of re-queuing the job.
> - Backoff caps at 2s, too low for mobile networks.
> - The dedupe key includes a timestamp, so retries are never deduplicated.
>
> Fix: strip the timestamp from the key, raise the cap to 30s, re-throw the timeout.

Same three bugs, same file path, same numbers. 104 words to 48.

## Install

```bash
git clone https://github.com/anshuman13/llm-brevity
mkdir -p ~/.claude/skills
cp -r llm-brevity/llm-brevity ~/.claude/skills/
```

Requirements: Claude Code. No external CLI, no API key, no network calls — the skill is a single `SKILL.md`.

## How it works

No subprocess and no second model. `SKILL.md` gives Claude an explicit cut list (restated questions, work narration, double hedges, empty transitions, closing offers to help) plus structural rules (one idea per sentence, active voice, lists over prose), then a verification step: every number, path, and code block from the source must appear in the output.

Deletion is checkable in a way that "be more concise" is not. A model asked to shorten its own writing will trim clauses everywhere and quietly lose a caveat; a model handed a list of specific constructions to delete can be checked against it. That's the whole design — the rules are concrete enough to audit.

## Anti-goals

Brevity never overrides correctness. The skill will not drop a caveat that changes what you should do, compress a command you have to run, or turn a clear explanation into shorthand you have to decode. Fewer words, same clarity. When the two conflict, being right wins and the reply gets a sentence longer.

## Credit

The one-shot-rewrite shape is borrowed from [nobuzz](https://github.com/adnanakil/nobuzz), which pipes replies through a second model to strip Claude's voice. This one targets length instead of tone and does it in-process, so there's nothing to install.

## License

MIT
