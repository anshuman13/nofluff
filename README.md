# nofluff

A skill (`/nofluff`) that defines how the assistant writes: direct, active voice, no stage performances, common words, no padding. It is on by default once loaded, and it also compresses existing text on demand.

## The problem

Ask a model a yes/no question and you get a restatement of your question, a narration of what it's about to check, the answer, a summary of the answer, and an offer to help further. The answer was one word. Everything else is packaging.

This isn't a style preference — padding costs reading time on every single reply, and it buries the one sentence you needed. The skill's core rule is **compress by deleting, not rewriting**: every fact, number, file path, command, and code block survives. Only filler dies. A rewrite that loses content has failed, because that's summarization, and you asked for a shorter reply.

## The voice

Four rules, applied to every reply:

- **Direct.** Lead with the answer. No preamble, no restating the question.
- **Active voice.** "The parser drops the token", not "the token is dropped by the parser".
- **No stage performances.** No announcing what's about to happen, no labelling a point as interesting, no building to a reveal.
- **The most common word.** "use" over "utilize", "so" over "as a result", "before" over "prior to".

## Usage

Once loaded the skill is **on by default** — it applies to every reply for the rest of the session with no command needed.

```
/nofluff --off    # back to normal output
/nofluff --on     # re-enable
```

It also compresses existing text on demand:

```
/nofluff              # compress the previous reply
/nofluff <text>       # compress pasted text
/nofluff notes.md     # compress a file
```

It triggers on plain requests too — "shorten that", "you're being too verbose."

There is no word-count target. It cuts until the cut list is exhausted and then stops, losslessly: every fact, number, path, and code block survives, while elaboration and transitions go. A padded draft collapses hard; an already-edited one barely moves.

## Before / after

A real run of the on-demand rewrite on a genuinely padded draft reply (363 words):

**Before:**

> Great question! You're asking about why the login endpoint has started returning intermittent 500s, and that's definitely a common issue that can be tricky to track down. Let me take a look at the auth service and see what I can find. [...] First, and this is really the load-bearing issue here, the connection pool in `authService.ts:88` is configured with a maximum of 10 connections. That might sound like it would generally be enough, but it's worth noting that the session lookup on every request holds a connection for the entire duration of the bcrypt comparison. [...] And third — here's the kicker — the health check endpoint at `/healthz` also uses the same shared pool. [...] To summarize what we've covered: the connection pool is too small for the workload [...] Let me know if you'd like me to dig deeper into any of these!

**After (109 words):**

> Three interacting causes:
>
> 1. The connection pool in `authService.ts:88` maxes at 10 connections. The session lookup holds a connection for the whole bcrypt comparison, so each request occupies one for ~200ms.
> 2. No queue timeout. On pool exhaustion requests wait indefinitely instead of failing fast, so load compounds rather than degrading.
> 3. `/healthz` shares the pool. When it saturates, the health check times out, the load balancer pulls the instance, and traffic shifts to the remaining instances — hence the cascade.
>
> Fix: move the bcrypt comparison outside the connection scope, raise pool max to 50, add a 5 second queue timeout, give the health check its own single-connection pool.

363 to 109 words, 30%. Every fact survived: the file path, all four numbers, the endpoint, and all four parts of the recommended fix.

## What testing showed

Measured against a control — a second agent given the same text and only the instruction "make it shorter," no skill:

| Source | Baseline (no skill) | Skill |
|---|---|---|
| Padded 363-word draft | 116 words (32%) | 109 words (30%) |
| Already-edited 251-word reply | 210 words (84%) | 205 words (82%) |

Two things worth being honest about:

**The skill does not compress much harder than simply asking for concision.** On both sources it beat the control by about two percentage points. If you want shorter output and you remember to ask for it, asking works nearly as well.

**Its value is the guardrail, not the ratio.** Both skill runs were lossless by construction — the cut list is a fixed set of constructions, and step 5 requires verifying every number, path, and code block survived. The control runs were lossless by luck. On text where a caveat is load-bearing, that difference is the point, and it is also why the skill is worth invoking on someone else's output rather than your own.

**Already-edited text barely compresses, and that is correct.** At 82% the skill is close to a no-op, because there was no filler to remove. A tool that squeezed that source much further would be deleting facts.

## Install

```bash
git clone https://github.com/anshuman13/nofluff
mkdir -p ~/.claude/skills
cp -r nofluff/nofluff ~/.claude/skills/
```

Requirements: an agent that loads skills from `~/.claude/skills`. No external CLI, no API key, no network calls — the skill is a single `SKILL.md`.

Status: the on-demand rewrite is tested across two sources with a no-skill control, in fresh sessions that had not seen the skill written. The always-on mode and the voice rules are written but not yet exercised.

## How it works

No subprocess and no second model. `SKILL.md` gives the model four voice rules, an explicit cut list (restated questions, work narration, double hedges, empty transitions, closing offers to help), and writing rules (one idea per sentence, lists over prose). For on-demand compression it adds a verification step: every number, path, and code block from the source must appear in the output.

Deletion is checkable in a way that "be more concise" is not. A model asked to shorten its own writing will trim clauses everywhere and quietly lose a caveat; a model handed a list of specific constructions to delete can be checked against it. That's the whole design — the rules are concrete enough to audit.

## Anti-goals

Correctness outranks format. The skill will not drop a caveat that changes what you should do, compress a command you have to run, or turn a clear explanation into shorthand you have to decode. Fewer words, same clarity. When the two conflict, being right wins and the reply gets a sentence longer.

Direct is not the same as rude. It is about structure — answer first, no padding — not about tone toward you.

## License

MIT
