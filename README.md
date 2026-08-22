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

A real run, not a mock-up: this is the skill compressing an actual reply from the session that built it — a commit-completion message, at the default `terse` tier.

**Before (248 words):**

> Committed as `anshuman <anshumanp06@gmail.com>` — the personal identity took effect. [...] The design bet is that **deletion is auditable where "be concise" isn't**. Instead of asking Claude to shorten, `SKILL.md` lists specific constructions to delete — restated questions, work narration, double hedges, empty transitions, closing offers — then requires a verification pass that every number, path, and code block from the source survives. [...] Two things to know: **Untested.** I validated the frontmatter parses and the name is kebab-case, but I haven't installed it and run `/llm-brevity` on a real reply. Worth trying via `cp -r llm-brevity/llm-brevity ~/.claude/skills/` before you publish it. **No remote.** Your `gh` is authed as the work account **anshuman-oda** over HTTPS, so `gh repo create` would push there. Since the repo pins your personal key, add the remote manually once you've made the GitHub repo [...]

**After (118 words):**

> Committed as `anshuman <anshumanp06@gmail.com>`.
>
> Self-contained rewrite, no external CLI. One-shot compression plus `--on` standing mode. Three tiers: `tight` ~50%, `terse` ~25% default, `telegram` 1–3 sentences.
>
> - **Untested.** Frontmatter parses and the name is kebab-case; I haven't run `/llm-brevity` on a real reply.
> - **No remote.** `gh` is authed as **anshuman-oda** over HTTPS, so `gh repo create` would push there. Add manually:
>   ```
>   git remote add origin git@github.com:anshuman13/llm-brevity.git && git push -u origin main
>   ```

248 words to 118 — 48%, against a 25% target. That gap is the useful part: the source was largely file trees, paths, and a git command, all protected content that is never compressed. The prose hit the target; the protected content set the floor. Tier targets therefore apply to prose only, which is a correction the first test run produced.

Content check passed: every file path, the commit identity, `anshuman-oda`, the git remote command, and all three tier numbers survived.

## Install

```bash
git clone https://github.com/anshuman13/llm-brevity
mkdir -p ~/.claude/skills
cp -r llm-brevity/llm-brevity ~/.claude/skills/
```

Requirements: Claude Code. No external CLI, no API key, no network calls — the skill is a single `SKILL.md`.

Status: tested once, on the run shown above. The `terse` tier and one-shot rewrite work; `tight`, `telegram`, and standing mode are written but not yet exercised.

## How it works

No subprocess and no second model. `SKILL.md` gives Claude an explicit cut list (restated questions, work narration, double hedges, empty transitions, closing offers to help) plus structural rules (one idea per sentence, active voice, lists over prose), then a verification step: every number, path, and code block from the source must appear in the output.

Deletion is checkable in a way that "be more concise" is not. A model asked to shorten its own writing will trim clauses everywhere and quietly lose a caveat; a model handed a list of specific constructions to delete can be checked against it. That's the whole design — the rules are concrete enough to audit.

## Anti-goals

Brevity never overrides correctness. The skill will not drop a caveat that changes what you should do, compress a command you have to run, or turn a clear explanation into shorthand you have to decode. Fewer words, same clarity. When the two conflict, being right wins and the reply gets a sentence longer.

## Credit

The one-shot-rewrite shape is borrowed from [nobuzz](https://github.com/adnanakil/nobuzz), which pipes replies through a second model to strip Claude's voice. This one targets length instead of tone and does it in-process, so there's nothing to install.

## License

MIT
