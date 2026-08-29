# Feedback log

Raw observations about how `nofluff` performs in real sessions. The
[`nofluff-improve`](nofluff-improve/SKILL.md) skill reads this file, groups repeated
entries, and proposes edits to [`nofluff/SKILL.md`](nofluff/SKILL.md).

This file is input, not documentation. Entries are appended as they happen and are never
edited to read better. An entry that has been folded into the skill gets `[applied]` and
the commit that did it, so the improver stops counting it as open signal.

## Format

```
## YYYY-MM-DD — short label
**What happened:** the actual output, quoted where possible.
**Expected:** what the skill should have produced.
**Why it matters:** the cost of getting it wrong.
```

Quote real output. "Too verbose" is not actionable; a pasted reply with the padding still
in it is. Warp's finding was that a few detailed reports beat many generic ones, so one
precise entry here is worth more than ten reading "too long again".

---
