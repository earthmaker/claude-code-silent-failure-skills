---
name: one-fact-many-places
description: When several places each record the same fact, fixing one leaves the rest stale and they quietly govern the next session — search by what the value counts, not by the value.
when_to_use: Before changing a number, status or label; before reporting "fixed", "closed" or "done"; when recomputing values after the sample changed; when flipping checklist statuses; and after revising a document body, to check that summaries, applications and outlines followed.
---

# One fact, many places

In long documents and document sets, the same fact is written more than once. Fix one spot and the others stay stale
without any error; the next reader — human or agent session — trusts the stale one. Unlike rules copied into a known set of
channels, numbers, statuses and labels have **no list of places**, so **how you search is the whole rule**.

## Four in one day

| What | How it happened |
|---|---|
| **One customer segment** | A count in one section of a sales report was corrected 37 → 38, but **two other sections counting the same customers didn't follow**. All three counted customers meeting the same condition |
| **Eight checklist labels** | A decision commit closed eight items, but the summary table still showed 🔴, so they looked like open work for a day |
| **"Source not obtained"** | Further down the same file: "the PDF was received that day". The endnote at the top still said "not obtained" |
| **"5 papers missing"** | The table had six rows. The number was typed into the sentence without counting the table |

The first three were caused by other sessions and found that day; the fourth was created that day.
**Without a procedure, neither the maker nor the finder catches it.**

## Searching by value doesn't find them

Grepping `37` / `38` finds only the section that was fixed. The other two counted the same customers inside **different numbers** (123, 103).

What you search for is not the value but **what the value counts**.

```
Value to fix:   "lapsed repeat buyers" 37 → 38
What it counts: "customers who ordered at least twice"
→ find every place counting that
   · lapsed repeat buyers (2+ orders, none this year)   §6
   · repeat rate among first-quarter buyers             §3-2   ← didn't follow
   · loyalty-program candidates (2+ orders)             §1-1   ← didn't follow
```

## Claims, not just numbers

An article body was revised after review, reversing the direction of its central claim. The **application summary and section
titles** in the same folder kept the pre-revision claim for eleven days. Because the application is submitted first and its approved
outline becomes the reference, the application would have promised what the body denied.

With several deliverables, the body's claims, section titles and axis lists are "what is being counted". Value-grep fails
because what changed is the direction of a sentence. Section titles are strings, so this does catch them:

```bash
diff <(grep '^## ' draft.md) <(grep '^## ' outline.md)
```

## Procedure

1. **Write down in one sentence what the value counts** — as a **criterion**, e.g. "customers with 2+ orders and none this year", not "number of lapsed buyers".
2. **Find places that share part of that criterion.** Scan section titles, table headers, endnotes; take grep vocabulary from the criterion.
3. **Follow derived values.** Ratios, multiples and totals computed from it are places too.
4. **Check the top and bottom of the same file.** Long documents state the same fact in the head and the tail.
5. **Change them all in the same commit.** "Later" never comes — it's silent, so no one asks.

## Status marks go stale faster than values

Checklist ✅ / 🔴 marks tend to be updated in a different commit from the work, so they are wrong **in both directions**.

| Direction | Example |
|---|---|
| Not done, marked ✅ | an item marked "all chapters done" was missing two chapters |
| Done, marked 🔴 | eight items closed by a decision still showed 🔴 |

**Update the label in the commit that does the work.** That is the only place that prevents this at the source.

## Before reporting "fixed"

```bash
grep -rn "<vocabulary of the criterion>" <docs>     # by what it counts, not by value
grep -rn "<things computed from it>" <docs>         # derived values
```

⚠️ **Zero hits means "nothing stale remains", not "it's everywhere it should be".** A place that should state the value but
doesn't is invisible to every grep — list the consumers (documents, agents, skills) and open each one.

## Mirror image — don't create the copy

Prevention is not duplicating. After receiving twenty source papers, I deliberately didn't write new extracts: seventeen
already had their sample sizes and statistics in the manuscript's endnotes, and two copies means one goes stale.
Instead I built only an **index** of where each paper's details live.

**Pointers, not copies.** Whenever you want to copy a value, first ask "if this goes stale, who will notice?"

## Related

- `exhaustive-check-completeness` — when "zero hits" can be trusted
