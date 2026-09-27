---
name: shipped-config-mismatch
description: The setting you chose may not be the setting that ships — nominal and shipped values diverge without errors, while the dry run prints the approved value.
when_to_use: Right before exporting a submission or deployment; after changing run settings via flags or constants; before executing (or writing) a handoff note's "next run" prescription; and before interpreting "I changed it and the result didn't move".
---

# Does the shipped artifact run with the setting you chose?

"I changed the setting" and "the changed setting shipped" are different. The gap raises no error, and the dry run
prints the value you chose, so it surfaces only at the next submission or deploy. There are four causes, and **no single check catches all of them**.

| Cause | Shape |
|---|---|
| Not delivered | someone else's runner ignores your flags |
| Clipped | a cap, budget or clamp quietly reduces your value |
| Overridden | a middle layer (CDN, proxy) replaces your value |
| Ignored | the value arrived, but the condition that consumes it depends on another flag |

## 1 — Someone else's runner ignores your flags

A code-submission competition's evaluator ran `script.py` **with no arguments**. We chose settings via CLI flags and the dry run
printed `--mode A --variant B`, but the `argparse` defaults inside the zip were **three different values**.
Caught right before submitting — otherwise we'd have spent a submission on something other than what we had decided to submit.

## 2 — A cap quietly clips your value

`--max-chars` was raised to widen the context, but the model's maximum context length fixed the prompt budget,
and the budget-fitting function trimmed excerpts. **The measured median context was identical to the old setting.**

→ Look at the **measured distribution**, not the nominal value. Measure what the setting actually produced (context length,
token count, call count) on a sample and compare with before. Same median means nothing changed; only the max changing means only the tail changed.

## 3 — A middle layer overrides you

To shorten image caching, the static host's `_headers` was set to `max-age=3600`, deployed, and **verified** on `<project>.pages.dev`.
At the same moment the custom domain said otherwise.

| Host | What visitors receive |
|---|---|
| `<project>.pages.dev` | `max-age=3600` ← what we wrote |
| `img.<custom-domain>` | **`max-age=14400`** ← overridden by the CDN zone |

The Cloudflare zone's Browser Cache TTL (default 4 hours) took precedence over the origin header — it overrides origin values shorter than itself unless the zone is set to respect existing headers. Published URLs used the custom
domain, so the real value was four times ours.

→ **If visitors reach you through several paths, measure all of them side by side.** When the overriding value is a default,
nobody ever set it, so it's in no settings history and no repository — **only in the response**.

## 4 — The flag shipped; its precondition didn't

`--use-scores` was baked in with `default=True`, and reading it back confirmed `True`. But the line that requests scores from the model was
`request_scores = 5 if (thresholds or probe_scores) else 0` — **the flag wasn't in the condition**. With zero requested, every score
came back `None`, and the fill code took its **normal fallback** ("no probabilities → default rule"). Two submissions shipped that way,
and one nearly led to "this feature has no effect".

The dry run couldn't catch it in principle: the mock runner never returns scores, so every configuration takes the same fallback.

## Checks — not "did it change?" but "what is the value?"

1. **Open the shipped artifact and read the value** (primary). Extract from the zip and parse (AST, argparse, regex).
   sha256 and diff only say "something changed".
2. **Run without any setting flags** — the same condition as someone else's runner.
3. **List what changed** (secondary). Compare per-entry CRCs in the zip's central directory without extracting — one second even for
   gigabytes, proving file by file that "only one thing changed" and "no new file slipped in".
4. **For every baked flag, also read the condition that consumes it** — not the declaration (`default=True`) but the point of use.
5. **Positive-control "it took effect", not "it shipped".** Count whether output with the flag on and off actually differs (changed cells, trigger rate).

There are three layers of read-back — **value, consuming condition, output difference.** Stop at one and failures in the others read as "that axis doesn't matter".

⚠️ A substitution script that can't tell "already correct" from "pattern not found" dies there. Reading back the value distinguishes them.

## Reading Δ = 0

There are two causes and no way to tell them apart without checks:

1. that axis isn't the bottleneck ← what you wanted to establish
2. **what you changed never shipped**

Reading it as (1) without ruling out (2) kills an axis wrongly. Even a noise-free scorer doesn't prevent this — it's attribution, not noise.

## Handoff prescriptions may be measured on another baseline

A handoff note (a file one working session leaves for the next) saying "next run = X" records a value **measured on that day's baseline and build path**. If the baseline has since changed, or the
measurement wasn't on the shipping path, executing the prescription verbatim may leave the intended knob disconnected or mix in another axis.

- Once, the configured path built with **a different random seed** than the note assumed, which would have mixed in a difference an order of magnitude larger than the effect.
- Once, the prescribed model column was **overwritten by a downstream rule** in the shipping configuration and changed zero output rows. The note's face-value prediction reproduced on no baseline.

**Guard both ends.**

- **Before executing a prescription** — run the current shipped artifact and the new build on the same input and count changed columns/rows.
  Is the only change the knob you meant, and is it non-zero? If zero, it's a dead knob — don't ship it.
- **When writing a prescription** — put next to the predicted number **the baseline it was measured on** and **how many final output rows it changes**.

"The baseline changed, but it doesn't touch that column, so the value is the same" is an argument, not a measurement.

## Related

- `parser-fallback-fabricates` — the same family: a "normal fallback" hiding a fault
- `negative-result-needs-control` — the mirror: reading absence where something exists, versus a different value where you expected yours
- `rebuild-overwrites-human-copy` — another danger of executing handoff notes verbatim
