# Silent-failure skills for Claude Code

Seven Claude Code skills for the places where an agent goes wrong **without raising an error**:
empty results that aren't really empty, parser fallbacks that fabricate, builds that overwrite human edits,
settings that never ship, and paid APIs called without a budget lock.

Each skill came out of a failure that actually happened, and each includes that case plus a check you can run the next time.

## Why these exist

Most agent mistakes announce themselves: a stack trace, a failing test, a 500. The expensive ones don't.

- A search returns 0 hits because the region was renamed — and the agent reports "none exist".
- A list API quietly returns the first 20 of 2,526 files — and the agent concludes the dataset has no training split.
- A parser falls back to "grab every number" and shifts coordinates by one — and all three sanity guards pass it through.
- A CLI flag is set in the dry run but the evaluator runs the script with no arguments — and the "no effect" result kills a good idea.
- A build is re-run while the user has the output open in their editor — and their morning of edits is gone.

None of these raise anything. Every case here happened in real daily agent work:
data pipelines, public-API collection, document generation and ML competitions.
They're written as skills so that Claude loads the right checklist at the moment the mistake is about to happen.

## Skills

| Skill | Loads when | What it prevents |
|---|---|---|
| [`negative-result-needs-control`](skills/negative-result-needs-control/SKILL.md) | a lookup returns nothing, or a remote says "retry later" | Reading 0 hits, 404 or "temporarily unavailable" as absence — run a positive control first |
| [`exhaustive-check-completeness`](skills/exhaustive-check-completeness/SKILL.md) | about to say "checked everything" / "zero left" | Nine ways that claim becomes false (truncation, server paging, skipped binary files, patterns that cut matches…) |
| [`one-fact-many-places`](skills/one-fact-many-places/SKILL.md) | changing a number, status or claim | Fixing one copy of a fact and leaving the others stale — search by what the value counts, not by the value |
| [`parser-fallback-fabricates`](skills/parser-fallback-fabricates/SKILL.md) | writing a parser for model/API output | Fallbacks inventing plausible values that pass every guard; tests built from imagined inputs |
| [`shipped-config-mismatch`](skills/shipped-config-mismatch/SKILL.md) | exporting a submission or deployment | The chosen setting differing in the shipped artifact — not delivered, clipped, overridden, or ignored |
| [`rebuild-overwrites-human-copy`](skills/rebuild-overwrites-human-copy/SKILL.md) | about to re-run a builder | Overwriting a file someone has open or has edited |
| [`paid-api-budget-guard`](skills/paid-api-budget-guard/SKILL.md) | about to spend money on an API or GPU | Calling paid APIs without permission, balance check, a measured unit price, or a cap |

## Install

**As a plugin** (recommended — one command, updates with the repo):

```
/plugin marketplace add earthmaker/claude-code-silent-failure-skills
/plugin install silent-failure-skills@silent-failure
```

From a terminal, the same thing is `claude plugin marketplace add …` and `claude plugin install …`.
Plugin skills are namespaced, e.g. `silent-failure-skills:negative-result-needs-control`.

**Or copy the files:**

```bash
git clone https://github.com/earthmaker/claude-code-silent-failure-skills
cp -R claude-code-silent-failure-skills/skills/* ~/.claude/skills/          # all your projects
# or: cp -R claude-code-silent-failure-skills/skills/* <project>/.claude/skills/
```

Take only the ones you want — each skill is a single self-contained `SKILL.md`.

## How they work

Claude Code shows the model each skill's `name`, `description` and `when_to_use`, and loads the full body only when the situation matches.
You don't invoke them; they come in when, for example, a query returns nothing and the agent is about to write "there are none".
If you want to test that a skill loads, ask Claude to do the triggering task and watch for the skill in the transcript.

The files are plain Markdown with YAML frontmatter, so other agents that read `SKILL.md` files can use them too.

## How to read them

Every skill has the same shape: a one-line rule, the real cases behind it, and commands to run when you're in the same spot.
The numbers in the cases are what was measured at the time. Tools and APIs change — **re-measure as the skill tells you**; that is the point.

Some cases involve Korean public APIs and Korean filenames, because that's where they happened. The rules are general.

## Related work

These are meant to sit alongside, not replace, broader discipline skills:

- [obra/superpowers](https://github.com/obra/superpowers) — `verification-before-completion`, `systematic-debugging`, test writing
- [`runaway-guard`](https://github.com/sickn33/agentic-awesome-skills/tree/main/plugins/agentic-awesome-skills/skills/runaway-guard) — cost safety for production code that calls paid APIs

## Contributing

Issues and pull requests are welcome, especially **new cases**: a real silent failure, what it looked like, and the check that caught it.
A case with a reproducible check is worth more than a new rule.

## License

Text: [CC BY 4.0](LICENSE). Code snippets inside the skills: [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) — copy them without attribution.
Use and adapt freely; for the text, please credit
"Silent-failure skills for Claude Code (github.com/earthmaker/claude-code-silent-failure-skills)".
