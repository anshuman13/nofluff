---
name: llm-brevity
description: Compress a verbose LLM reply into a shorter one, or switch the session into a terse output mode. Length tiers (tight/terse/telegram) control how aggressive the compression is. Use whenever the user invokes /llm-brevity, asks to "shorten that", "cut that down", "tl;dr that", "say that in fewer words", complains that a reply was too long, padded, repetitive, or over-explained, or asks you to be more concise from now on.
---

# llm-brevity — cut the word count, keep the content

LLM replies pad. They restate the question, narrate what they are about to do, hedge every claim twice, add a summary of what was just said, and close with an offer to help further. None of that carries information. This skill removes it.

The rule that makes this work: **preserve content, cut words.** Every fact, number, file path, command, code block, caveat, and name in the source survives. Deletion is the primary tool — most padding is removable outright — but rewriting a clause to say the same thing in fewer words is fine. What is never fine is dropping a fact. If the output lost content, it failed; that is summarization, and the user asked for brevity.

## Arguments

`/llm-brevity [tier] [--on|--off] [text]` — all parts optional.

- **tier**: if the first word is `tight`, `terse`, or `telegram`, that is the tier. Otherwise `terse`.
- **--on / --off**: toggle standing mode instead of doing a one-shot rewrite. See Standing mode.
- **text**: whatever remains is the text to compress. If it is a readable file path, read that file and compress its contents. If empty, compress your own most recent substantive reply — the one immediately before this invocation. Reproduce it faithfully from the conversation before compressing.

Examples: `/llm-brevity` (terse, last reply) · `/llm-brevity telegram` · `/llm-brevity tight <pasted text>` · `/llm-brevity --on`.

## Tiers

| Tier | Target | Keeps |
|------|--------|-------|
| `tight` | ~50% of prose words | Everything. Pure filler removal — no content judgment at all. |
| `terse` (default) | ~25% of prose words | Every fact, number, path, and code block. Drops elaboration, examples that repeat a point, and all transitions. |
| `telegram` | 1–3 sentences | The answer and nothing else. The single load-bearing fact, plus any command or path needed to act on it. |

`tight` and `terse` are lossless on content. `telegram` is explicitly lossy — it is the one tier permitted to drop facts, and only because the user asked for the answer alone.

Targets are calibrated against padded first-draft output. Measured runs on a genuinely padded draft landed at 30% with the cut list exhausted, so treat ~25% as the floor of a 25–35% band rather than a bar to clear. Already-edited text lands far higher and that is correct.

**Losslessness wins over the target.** The tier number is aspirational; `tight` and `terse` never drop facts to hit it. If the two conflict, miss the number and say so. An agent that deletes a caveat to reach 25% has broken the skill.

**Prose** means text outside fenced code blocks, inline code spans, file paths, commands, and tables. Measure only that. Bold inline labels (`**Repo:**`) are prose; the path or command after them is not.

Targets apply to **prose**, not to the whole reply. Protected content — code blocks, file paths, commands, tables — is never compressed, so a reply that is mostly code has a floor well above its tier target. Hitting 50% on a code-heavy reply whose prose was fully stripped is a success, not a miss. Measure the prose, not the total.

## The cut list

Delete on sight, at every tier:

- **Restating the question.** "You're asking about X" — they know.
- **Narrating the work.** "Let me check…", "I'll start by…", "First, I looked at…" Report findings, not the path to them.
- **Preamble before the answer.** "Great question." "That's a common issue." Lead with the answer.
- **Summary after the answer.** A closing paragraph that repeats the body. If the body was clear, it is redundant; if it wasn't, fix the body.
- **Double hedges.** "It might possibly", "generally tends to". One hedge or none. Keep a hedge only where the uncertainty is real and load-bearing.
- **Offers to continue.** "Let me know if…", "Happy to dig deeper." The user knows they can ask.
- **Self-praise and drama.** "the load-bearing assumption", "here's the kicker", "this is the interesting part". Interesting things are interesting without the label.
- **Adverbs of emphasis.** "quite", "really", "actually", "essentially", "basically", "simply", "just", "very".
- **Empty transitions.** "That said", "With that in mind", "It's worth noting that", "Importantly".

## Structural rules

- Convert prose to a list when it enumerates things; convert a one-item list back to prose.
- One idea per sentence. Split any sentence with two independent clauses joined by "and" or "but" where both carry weight.
- Prefer the shorter word: "use" over "utilize", "so" over "as a result", "before" over "prior to".
- Active voice. "The parser drops the token", not "the token is dropped by the parser".
- Tables and lists beat paragraphs for anything enumerable. Do not narrate a list in prose first.
- Code blocks, file paths, commands, and error strings are never paraphrased or trimmed. They are content.
- No markdown `#` heading over a section shorter than three lines. This does not apply to bold inline labels, which are content.

## How (one-shot rewrite)

1. Take the source text — the pasted argument, or your own previous reply reproduced faithfully.
2. Delete everything on the cut list. Do not rewrite yet; just remove.
3. Apply the structural rules to what remains.
4. Check the *prose* against the tier's target. If it is over, re-read the cut list once and remove anything you missed.

   **Then stop.** If the cut list is exhausted and the prose is still over target, the source was already dense — that is a valid outcome, not a failure. Do not cut facts, protected content, or caveats to reach the number. Already-edited text often compresses only to 70–85%, and a source with no filler may not compress at all. Report the ratio if asked; never manufacture it.
5. Verify nothing was lost: every number, path, command, and code block in the source appears in the output (except in `telegram`, which may drop them).
6. Output the compressed text as your entire reply. Do not introduce it, explain what you cut, or comment on the improvement — that would re-add exactly what you removed.

   Two qualifications. **Use the `terse:` lead only when the compressed text would otherwise be mistaken for a fresh answer** — in a running conversation where the user needs to see it is a rewrite. Omit it when the output goes to a file or is obviously a rewrite. And **this step governs the compressed artifact, not surrounding task reporting**: if the caller asked for the result in a file plus a critique in the reply, follow the caller. The no-commentary rule protects the compressed text itself from re-acquiring padding.

## Standing mode

`/llm-brevity --on` switches the rest of the session to terse output. Confirm with a single line — `Brevity on (terse).` — and nothing more.

While active, apply the cut list and structural rules to every reply as you write it, rather than compressing afterward. Specifics:

- Answer in the first sentence. Context, if genuinely needed, comes after.
- Skip tool-call narration. Do the work, report the result.
- No summary section unless the reply is over roughly 200 words.
- Match reply length to question complexity. A yes/no question gets a yes or no plus one sentence of why.
- Code and paths stay complete. Brevity applies to prose, never to a command the user has to run.

Brevity never overrides correctness. If a caveat changes what the user should do, keep it — a wrong short answer is worse than a right long one. When forced to choose, be right and add a sentence.

`/llm-brevity --off` restores normal output. Confirm with `Brevity off.`

## Anti-goals

Not terseness for its own sake. Specifically do not:

- Drop a warning, caveat, or failure case that affects the user's decision.
- Compress a code block, command, or file path.
- Turn a clear explanation into cryptic shorthand the user has to decode. Fewer words, same clarity — not fewer words, less clarity.
- Refuse to elaborate when the user asks a follow-up. Brevity is the default, not a rule against detail on request.
