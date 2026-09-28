---
name: parser-fallback-fabricates
description: When parsing someone else's output, a fallback that invents a plausible value instead of returning "not found" disables every downstream guard — and tests built from invented inputs cannot catch it.
when_to_use: When writing or changing code that parses output from a model, API or document; when adding range/size/format guards to parsed results; when writing parser tests; and when all tests passed but production failed.
---

# A fabricating fallback punches through every guard

Two rules for parsing output you don't control.

1. **A fallback ends empty-handed.** If you couldn't read it, return "could not read" and let the caller back off.
   If you manufacture something, downstream safeguards accept it as normal input.
2. **Parser tests use real outputs.** Inputs you invent test "the format I imagined" — real failures happen outside your imagination.

## The failure — ask a VLM where the label is, crop, read again

A vision-language model (Qwen2.5-VL) was asked for the label's location; that region was cropped and read again.
**The error metric (lower is better) went from 0.14 to 0.43.** Three things stacked up.

~~~~text
Requested   {"boxes": [[x1, y1, x2, y2], ...]}      coordinates normalized to 0–1000
Actual      ```json
            [ {"bbox_2d": [483, 167, 564, 221], "label": "paper label card"} ]
            ```                                      absolute pixels of the resized input
~~~~

- The model ignored the instruction and used its own format and coordinate system.
- `obj.get("boxes")` was empty, so parsing fell to a **"scrape all numbers" fallback**.
- That fallback grabbed every number and chunked by four. **The `2` in `bbox_2d` became a coordinate**,
  turning `[483,167,564,221]` into `[2,483,167,564]` — shifted by one.

## So all three guards let it through

There were three fallback-to-original guards. **None fired in 200 images.**

| Guard | Why it didn't fire |
|---|---|
| area (2–92%) | shifted coordinates still make a plausibly sized rectangle |
| aspect ratio (≤ 12:1) | likewise plausible |
| "no boxes → use original" | the fallback **made** boxes, so there were boxes |

**Guards check "is the value odd?", not "is the value real?"** Once a fabrication lands inside the normal range, they're useless.
The place to stop it is **the fallback itself**.

## Fifteen tests passed, and it still failed

Every test in that commit was green, because every input was **in the format we requested**.

~~~~python
parse_boxes('{"boxes": [[100, 200, 300, 400]]}', W, H)                   # the imagined world
parse_boxes('```json\n[{"bbox_2d": [483, 167, 564, 221]}]```', W, H)     # the real world
~~~~

When "all tests passed but it failed", **check where the test inputs came from first**. Invented inputs exercise the main path;
failures live in the fallback path, which by definition is where unexpected input arrives.

## The same failure in data plumbing

I measured a new signal (per-field log-probabilities) and got **exactly the old numbers**. I nearly concluded "the signal
doesn't help". In fact the payload builder picked columns selectively and the new ones were never shipped. The receiver had a fallback:

```python
row = {"d_mean": float(rec.get("d_mean_lp") or rec.get("mean_lp"))}   # ← silently the old value
```

No error. What caught it was an **exact match** — identical to the fourth decimal. "No improvement" and "nothing changed" differ.

- When testing something new, **measure the old thing in the same run**. If the old value reproduces, the wiring is right;
  if the new value is **exactly** the old one, that's a fault, not a result.
- Don't cherry-pick columns. Passing `dict(r)` whole makes this failure impossible.

## If you add a guard, watch it fire on input built to trip it

A guard that never fired may be dead, not idle. The two look the same.

```
normal input   → 0 triggers   ← indistinguishable from a dead guard
broken input   → N triggers   ← only now can you say it's alive
```

Example — a submission CSV's axis-order guard only `print`ed and blocked nothing (swapped axes silently score zero).
It became an `assert`, verified with a **deliberately swapped CSV**.

Counts are the same family. Writing `ALL` in a prompt without counting what comes back treats "I asked for it" as fact.

## So write it like this

- **Look at real output once before writing the parser.** Ten minutes of sampling saves hours.
- **Log diagnostics regardless of failure type.** I logged raw output only when zero boxes were found; the real failure was
  "found them all, in the wrong place", so nothing was logged. Don't assume you know the failure mode.
- **Log what the parser chose, alongside the raw text.**
- **Narrow the fallback.** The fix went from "every number in the body" to "four numbers inside brackets".
- **Discard values you can't explain.** Coordinates beyond the image size mean a wrong interpretation — drop the box, don't correct it.
- **Don't keep instructions the model ignores.** Match the instruction to what the model actually does; convert in code.

## Related

- `negative-result-needs-control` — the opposite direction: reading empty as absent, versus fabricating where empty was correct
- `shipped-config-mismatch` — other reasons "I changed it and nothing changed"
- External: [`writing-good-tests`](https://github.com/obra/superpowers/blob/main/skills/test-driven-development/writing-good-tests.md) (superpowers) — prefer real components over mocks; this skill is the parsing case of the same idea (test inputs must come from real outputs)
- External: [`defense-in-depth`](https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/defense-in-depth.md) (superpowers) — adds validation layers. Read the two together: extra layers help against invalid data, but not against a fabrication that lands inside every layer's valid range — that has to be stopped at the fallback
