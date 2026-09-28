---
name: negative-result-needs-control
description: Zero hits, "not found", or an empty list may be a query error rather than absence — run a positive control before concluding that something does not exist.
when_to_use: Before reporting "there is none" after a lookup, search, or API call returned nothing; when a remote answers "temporarily unavailable / retry later"; when writing code that fires many keys at once to collect results; and right before writing down a hypothesis that explains why something is missing.
---

# "Not found" only after a positive control

When a query returns nothing, run a **positive control** before reporting absence:
send one query that is certain to succeed and confirm the pipe is alive.

**Why** — absence and a broken query produce identical output. It happened twice in one day:

- **Public API** — a welfare-services API returned `totalCount=0` for a province. It wasn't absence:
  the province had been **renamed in an administrative reorganization**. Two counties actually had 19 and 9 entries.
  Asking twice with only the province name changed settled it in half a second.
- **Filesystem** — `ls docs/` came back empty and I nearly concluded the folder didn't exist.
  The agent's shell cwd had drifted into a subfolder. With an absolute path, everything was there.

## Each axis has its own positive control

| Axis | Positive control |
|---|---|
| API | Change one parameter and re-ask — suspect naming and code systems |
| Files | Check `pwd`, then retry with an absolute path |
| Search / grep | One word that is guaranteed to match |
| DB / query language | One known answer first |

⚠️ Word the report accordingly — "there is none" and "I could not find it" are different claims.
Without a positive control, write **"could not verify"**.

## Batch queries: count and report the failures

Firing a grid of keys fails differently — successes get merged and **failures appear nowhere**.
Counting successes alone makes the job look complete.

- An education-statistics API was given 44 district codes for one region. **All 16 districts of one province vanished.**
  A new provincial status had changed its region code, and the old code returns "no data exists" — not an error.
  It was caught only because the collector listed failed codes alongside results.
- A national open-data portal returns daily-quota-exceeded as **HTTP 200 with well-formed XML**
  (`LIMITED_NUMBER_OF_SERVICE_REQUESTS_EXCEEDS_ERROR`). A parser that only extracts tags cannot tell it from an empty result.
- **Lists get silently truncated** — a public API returned `list_total_count 42` but only 5 rows, and paging parameters
  returned the same 5. The item I wanted wasn't in the list, but requesting it by key worked.
  → **Compare the total-count field with the rows you actually received.** If they differ, you have a sample, not the full set.

⇒ Batch collection code should report three things: **success count, failed keys with reasons, and the comparison with the expected count.**
A collector that only prints "got N" can never show you what is missing.

### Total > 0 but zero rows: your own argument may have erased them

When exactly **one** result matches, the IEEE Xplore API with `max_records=1` returned `total_records: 1`
but dropped the `articles` key entirely, with HTTP 200.

| total_records | max_records | rows received |
|---|---|---|
| 1 | 1 | **0** ← read as absence |
| 1 | 2 | 1 |
| 2 | 1 | 1 |

That is exactly what happens when you fetch a single paper by DOI, and it produces "that paper doesn't exist".
A total that contradicts the row count is a signal, not absence — **wiggle your arguments by ±1**.
In code, put a floor on page size (never send 1; fetch more and slice).

## "Temporary error" is also an answer — a true answer from the wrong place

Worse is a remote that answers with a **plausible reason**. `retry later`, `temporarily unavailable`, `is_transient: true`
all read as "just wait", so nobody looks again.

- Validating an Instagram long-lived token against `graph.facebook.com/debug_token` returned
  `Service temporarily unavailable · is_transient: true`. **It was not temporary** — that host cannot even parse that
  token type. `graph.instagram.com/me` returned 200. The "transient remote error" verdict stood for **18 days**,
  during which the token-expiry guard never evaluated anything.
- Treat any "transient" label as a claim to test, not a fact: a wrong host, token type or resource ID can produce it.

**Here the positive control is "the same question somewhere else."** Change one axis at a time —
host, endpoint, auth type, resource ID. The axis whose change flips the answer is the cause.

⚠️ **Do this before writing retry logic.** Once classified as transient, code retries forever and tells no one.

## Missing permissions masquerade as an empty list

Some APIs return `success: true` with empty results — not 403 — when the token lacks permission.

| Asked | Answer | Truth |
|---|---|---|
| `/zones?name=<domain>` | `success: true` · count 0 | no permission (the domain exists) |
| `/accounts` | `success: true` · count 0 | no permission |
| `/accounts/{id}/registrar/domains` | `Authentication error` | the same problem, **stated explicitly** |

The third row acted as the positive control. When absence is suspicious, also hit a **neighbouring endpoint that fails loudly**.
Right after widening permissions, check "what now works" and "what used to work" in the same round —
editing permissions can drop existing grants, and that loss also looks like an empty list.

### One server, failing per tool family

A PubMed tool's search returned 0 even for the positive control (a single common word). Calling it broken was right,
but the report ended as "PubMed is down". The same server's ID-conversion tool worked, and could confirm five DOIs on the spot.

- Once you judge a tool broken, **try one tool from another family on the same server.**
- Don't write the verdict per server — "search is down" and "the server is down" imply different next steps.
- Don't read a connector's error text as the cause. It said `check your request parameters`, yet a one-word query
  failed too; hitting the upstream (NCBI esearch) directly with curl returned HTTP 500. **Probe the source behind the connector.**
- Empty results and such errors usually come back as **successful responses**, so tool-failure logs miss them.
  If you don't write it in your output, nobody will know.

## Non-ASCII filenames on macOS are NFD — existing files "don't exist"

On macOS, filenames are often decomposed (NFD) — HFS+ always stored them that way, and on APFS they keep whatever form the creating app, download or sync client used — while text typed in code is usually composed (NFC). For scripts such as Korean
the bytes differ, so **identical-looking strings don't match** — you get zero hits, not an error.
A file visible in `ls` returned nothing from `glob("*<Korean word>*.pdf")`, and a cleanup routine silently never ran.

🔴 **Deletion is the dangerous case** — a zero-hit search leaves a wrong report; a zero-hit delete silently leaves old files behind.

```python
import os, unicodedata as ud
want = ud.normalize("NFC", name)
hit = [f for f in os.listdir(d) if ud.normalize("NFC", f) == want]
```

## When a person says "it's not there"

After a test email, the reply was "didn't arrive". I took that as an observation and spent 30 minutes on SMTP, MX
and routing. Both emails were in **spam**. It wasn't "not there" — it was "not seen in the inbox".

→ Before asking why, ask **"where did you look?"** A false absence doesn't just cost time; it **hides the real cause**
(here, spam classification of forwarded mail).

## Don't adopt an explanation for an absence without testing it

Even when the absence is real, the hypothesis explaining it can be wrong.
🔴 **The better a hypothesis fits the observations, the more dangerous it is** — the fit substitutes for testing.

| Observation | Cause I nearly wrote | Actual cause |
|---|---|---|
| Preview URL fails TLS handshake | "no access path, so it's safe" | certificate issued 3 hours later; all 200 |
| Test email missing | "sent before verification, so dropped" (even the timing fit) | it had arrived (spam) |
| Data repository returns 403 everywhere | "rate limit from my parallel probes" | **fake-browser User-Agent blocked** — 403 from the very first request |
| Download at 11 KB/s | "server is slow, unusable" | three other downloads saturating the link; 778 KB/s after they finished |
| Segfault on a large job | memory (a true observation, and the fix was valid) | an audio library's codec dies at 240–270 s |

The last row is the most dangerous shape — **finding a true fact is not the same as finding the cause.**

→ Before writing a cause, run a **control that changes exactly one condition**. Four of the five were settled by a one-line
control (four User-Agents; re-measure after competing downloads end; grid over duration). Measuring the same condition twice is
not a control. Writing the verdict as **"pending"** costs nothing if it later flips.

## Mirror image — rule out your own misreading of a "contradiction"

Two official statistics differed by 3×, and I was about to argue "the counting systems are inconsistent".
They **weren't counting the same thing** (one was a modus-operandi classification in official statistics, the other an
agency's operational tally).

→ Fix the order: (1) confirm from primary sources that both values count the same population under the same definition;
(2) only then is a mismatch a contradiction. A time series is the cheapest discriminator — definitional differences create a
constant offset, not jumps in ratio or sign flips.

## Related

- `exhaustive-check-completeness` — when the whole search scope, not one query, leaks
- `parser-fallback-fabricates` — the opposite failure: producing something where there should be nothing
