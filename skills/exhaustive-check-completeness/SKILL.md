---
name: exhaustive-check-completeness
description: When a claim of "checked exhaustively" actually holds — you truncating output, grep not seeing empty places, servers truncating lists, tools skipping files, and patterns cutting matches. All of them leak without an error.
when_to_use: Right before reporting "checked everything", "zero hits", "all fixed" or "unobtainable"; when running leftover checks after renaming paths, names or rules; when a list API is the basis of a verdict; and right before saying "the solution is unique" after solving for unknowns.
---

# Before you say "checked exhaustively"

Many agent procedures rest on **"grep it; zero hits means done"**. If that search leaks, the whole procedure is silently void.
It leaks in several ways, none of which raise an error.

## 1 — Truncating the output and calling it exhaustive

🔴 **Never pipe an exhaustive check into `head` or `tail`.**

After `grep -rn "<old rule>" … | head -40`, I reported "checked all five places". Two files were cut off past line 40,
and one of them still carried the old rule.

```bash
grep -rc "<pattern>" <paths>          # count per file
grep -rl "<pattern>" <paths>          # file list — narrowing here keeps it exhaustive
grep -rn "<pattern>" <paths> | wc -l  # total count first
```

- If you must truncate, count first and report **"looked at the top M of N"**. Hiding a truncation is worse than truncating.
- Same trap, different face — **searching too narrow a path** (scanning one folder and structurally missing files next door).

## 2 — grep cannot find empty places

Fixing stale text and filling a missing place are different jobs. I replaced an old phrase everywhere, but the agent definition
that actually had to execute the rule **didn't contain it at all**. Nothing absent ever matches a grep.

1. **List who executes the rule** — which agents, skills, documents do this job.
2. **Open each one** and check the rule is there.
3. Zero hits means "nothing stale remains", not "it's everywhere it should be".

## 3 — The server truncates for you

List APIs truncate at a default size and return a normal-looking response with no hint.

Kaggle `competitions/data/list` returned **20 files** and looked complete (`page` gives 400; the right parameters are
`pageSize` + `pageToken`). From the truncated response, a competition read as "15 test files, no train — not a training
competition". Paginated, it had **2,526 files** — the opposite conclusion.

1. **Paginate to the end.** If there is `nextPageToken`, `next` or `offset`, there is more.
2. **Positive-control with a target of known size.** Counting another competition with the same code went 20 → 3,502, proving pagination worked.
3. **Distrust round numbers.** 20, 25, 50, 100, 1000 are more likely a default page size than data.

Database `LIMIT` defaults, the first page of a web UI, and caps in listing tools are the same trap.

## 4 — The tool skips whole files

Your shell's `grep` may be an alias or function calling another tool (here: ugrep via a shell function) with `-I` (skip files that look binary).
Check with `type grep`.

**One NUL byte and the whole file is skipped.** Text extracted from a PDF (valid UTF-8, 229 KB, 18 NULs):

| Command | Result |
|---|---|
| `grep -c "Endoscopic" x.txt` | **prints nothing**, exit 1 |
| `grep -a -c "Endoscopic" x.txt` | 89 |
| `/usr/bin/grep -c "Endoscopic" x.txt` | 89 |

Worse than zero — even `-c` prints nothing. I nearly reported "the document has no schedule". PDF and word-processor extracts
often contain NULs; search them with `grep -a` or Python `re`.

## 5 — The pattern itself cuts matches

A **context window** in the pattern silently drops matches near the start of a line.

```bash
grep -o ".\{28\}leverage.\{22\}" draft.md    # ✘ no match when fewer than 28 chars precede it
grep -c "leverage" draft.md                  # ✅ pin the total first
```

Hunting a banned word, this missed twice in a row (reported 7 / actual 8, reported 3 / actual 7). Matches with short left context
are usually **the first sentence of a paragraph** — exactly where the phrases you're hunting cluster.

**Separate counting from reading context.** Pin the total with `grep -c`; get context from whole lines (`grep -n`) or Python `finditer`.

### Same family — a second grep in the pipe

```bash
grep -rn "normalized" pipeline/*.py | grep -i "open\|read"   # ✘ 0 → "no consumers"
grep -rn "normalized" pipeline/*.py                          # ✅ 4 hits, 2 of them consumers
```

One consumer only built the path with `os.path.join(OUT, 'normalized')` (no `open` on that line); the other only added it to a zip.
**Don't mix "how is it used" into a search that counts.** Find all by name, then open each.

## 6 — Line breaks in the target cut the phrase

```bash
grep -c "<a six-word phrase quoted from the paper>" paper.txt                      # 0
tr '\n' ' ' < paper.txt | tr -s ' ' | grep -o "<same phrase>" | wc -l              # 1
```

PDFs break lines at the column width, so the phrase spanned two lines in the extract — and it was already quoted in the manuscript.
**Fold line breaks before counting multi-word phrases**, and write verification commands in folded form.

## 7 — Reading failure responses (0, 404, 403) as absence

The response was honest; **the target was wrong**. All three in one session:

| Got | Real cause | How to catch |
|---|---|---|
| Europe PMC `AUTH:"<group author>"` → 0 | that field doesn't index group authors | `JOURNAL:"<journal>"` alone → thousands |
| `PMC…/supplementaryFiles` → 404 | wrong PMCID | resolved from the DOI, the answer became "not open access" — **a different fact** |
| one path on an agency site → 403 | only that path is blocked | another path on the same domain → 200 |

- Before writing "absent" or "unobtainable", ask: is the identifier right? does that field hold this kind of value? does another request to the same server succeed?
- **404 and "exists but can't be served" are different facts** and lead to different sentences.
- Several mirrors all returning 403 with **byte-identical bodies** are one backend — one attempt, not several.

## 8 — Not scanning an axis you "know", then claiming uniqueness

I plugged a score into a closed form `S = f(m, t)`. `m` was a value I had set, so I scanned only `t`. One value fit, and I reported
"the solution is unique". Scanning all `(m, t)` gave **two solutions**, and the second pointed to the **opposite conclusion** (precision 57% vs 19%).

- "This axis is my own setting, so it's certain" is exactly the sentence that stops you from scanning.
- Several solutions with the same conclusion is a minor correction; one with the opposite conclusion means the verdict has lost its basis.
- The fix is not reducing solutions but **moving the evidence** — from "the equation gives a unique solution" to "an earlier observation verified the mechanism, and the mechanism fixes m".

### Same family — a zero from a check that never ran

- **A loop's counter after every iteration died** — all five iterations raised `TypeError`, yet the summary printed "0/5".
  → Print **how many were actually evaluated** next to any tally.
- **The tool doesn't know the option** — `grep -rPo "Name(?!ing)" … | wc -l` on BSD grep: no `-P`, usage went to stderr,
  `wc` counted the empty stdout and printed **0**. `set -o pipefail` doesn't help (the last command is `wc`).
  → Run a zero-result check once **on input built to match**. Prefer arithmetic: if `grep -o "Name"` equals `grep -o "Naming"`, zero plain "Name" remain.

## 9 — Word-boundary patterns die at one length

In ugrep 7.8.4, a boundary pattern meant to avoid substring hits **always returned 0 for exactly four-letter words**.

```bash
grep -aoE "(^|[^A-Za-z0-9])JSON([^A-Za-z0-9]|$)" x.txt | wc -l   # ✘ 0 (actually 1)
grep -aowE "JSON" x.txt | wc -l                                   # ✅ 1
```

Three- and eight-letter words (`SQL`, `Postgres`) worked, so a badly chosen sample hides the defect. Without boundaries you fail the other way:
`API` also matches inside `RAPID`. **Short uppercase acronyms can be wrong in both directions** — compute both and open the source if they differ.
Choose positive controls **shaped like what you're looking for**.

## Related

- `negative-result-needs-control` — is one query's zero an absence or a query error?
- `one-fact-many-places` — how to find every place a fact is recorded
- External: [`verification-before-completion`](https://github.com/obra/superpowers/tree/main/skills/verification-before-completion) (superpowers) — the general rule "run the verification before claiming success". This skill picks up where that one ends: the verification ran, and it still leaked
