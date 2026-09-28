---
name: paid-api-budget-guard
description: A lock procedure for an agent session about to call APIs that cost money (image/text generation, hourly GPUs) — user permission, balance check, estimate vs balance and cap, accumulate actual charges, stop before the cap, and never guess unit prices.
when_to_use: Before calling a paid model or API; before scaling up generation (count, repetitions); when a free tier runs out and you're about to switch to a paid one; and when estimating the time or cost of a long job.
---

# Paid APIs — lock before you call

Being careful in words isn't enough. **Put the cap in code.**
Tokens can be rolled back; charges can't. So every guard has to sit **before** the call.

**Scope** — this skill is for the moment an agent, during a working session, is about to spend money on the user's behalf:
a batch of image generations, a paid model run, a rented GPU. If you are designing **production code** that calls paid APIs
(retry policies, concurrency, idempotency keys, fan-out), use a production cost-safety skill such as
[`runaway-guard`](https://github.com/sickn33/agentic-awesome-skills/tree/main/plugins/agentic-awesome-skills/skills/runaway-guard)
as well — the two cover different moments.

## 0. Tell the user and get permission first

Before calling, state **which model, how many calls, roughly how much**, and get a yes. A budget guard limits "how much";
it doesn't answer "may I call at all".

Example — when a free image-generation quota ran out, the agent switched to a paid model without asking and made 10 images (about $1.40).
The user learned of it from the results report. It was under the cap and the balance was fine, but there was no permission.

A free path running out is exactly this moment:
"The free quota is used up. Generate N images with paid model X (about $Y), or wait for the free quota to reset?" — **stop and ask.**
Same under deadline pressure: urgency is information for the user to weigh, not grounds to decide for them.

## 1. Is there a flat-rate path you already pay for?

Check whether a **flat-rate subscription** (a web product) can do the same job. With flat rate, trial and error is free; with an API, exploration is money.
There's usually no API, so a person downloads results and hands over files — weigh whether that round trip is worth it.

## 2. Procedure — the order is the guard

1. **Check the balance** before the first call. If the check fails, that's a reason to stop.
   Some providers have no balance API (console only). That's a change of procedure, not an exemption: a person reads the console,
   sets `--budget` from it, and the code still does steps 3–5.
2. **If the estimate exceeds the balance, don't call — exit.**
3. **If the estimate exceeds the cap, don't call — exit.** Balance answers "can I", the cap answers "should I". Both must pass.
4. **Accumulate and print the actual charge per call** — the value the response returned (e.g. OpenRouter's `usage.cost`, where the provider returns usage accounting), not an estimate.
   If the response has no amount, accumulate tokens and convert with **a unit price annotated with the date you checked it**.
5. **Stop before the next call would cross the cap** — not after. `if spent + unit_cost > budget: break`

⚠️ **Balances can go negative.** One test image took a balance to −$0.17. Blocking after the fact is not blocking.

⚠️ **The code-side cap has a backstop: the provider-side hard cap.** A cap in your script is bypassed by a bug in your script.
Where the provider offers a hard spend limit (billing dashboard, workspace budget), set it before the first call, and check
whether it actually blocks or only alerts — some providers' "budgets" are soft thresholds. That setting lives outside the
repository, so a person has to confirm it once in the dashboard.

⚠️ **Running out of credit becomes silent omission.** An unattended loop of Anthropic API calls ran the balance dry and died with
`HTTP 400 credit balance is too low`; the SDK doesn't retry 400s, and the batch recorded those items as "no result" — hundreds lived only in an error log.
Unattended batches should **promote this error to a state plus a notification**.

⚠️ **Count and print failures too.** An account once got 429 on every model with zero usage. Swallowed failures turn
"no money spent, no results either" into a reported success.

## 3. Don't guess unit prices — the first call is a one-item measurement

An image was estimated at $0.003 and actually charged **$0.0387** (10×).
Make one item, **confirm the actual charge**, then estimate the full job from that.

### Read every component of the price sheet

A free-tier calculation read only "4.80 units per 512×512 tile" and got 520 images/day, but the same line also said
"9.60 units per step". At 8 steps, the step charge was four times the tile charge — really 104/day, 5× optimistic.

- Multiply every price component (tiles, steps, input tokens, output tokens, cache) into one per-item value.
- Parameters change the components. When you change `steps`, `quality` or resolution, **measure again**.

### If that one item is pathological, so is the estimate

An indexing job was estimated at 8.7 hours and took 45 minutes. The sampled run's log showed
`timed out … splitting in half and retrying`, and even with a note that the sample was inflated, it was extrapolated anyway.

- **Annotating a value is not fixing it.** Runs with retries, timeouts or cache hits can't be samples.
- **Check extrapolations mid-run on long jobs.** If progress isn't visible, use growth of by-products (e.g. cache files) as the signal.

## 4. Implementation shape

A sketch — `balance()` and `generate()` are yours to write for your provider:

```python
import sys

def run(items, budget, unit_cost_probe):
    bal = balance()                              # 1. raises on failure → stop
    est = unit_cost_probe * len(items)
    if est > bal:    sys.exit(f"insufficient balance: est ${est:.2f} > ${bal:.2f}")
    if est > budget: sys.exit(f"over cap: est ${est:.2f} > ${budget:.2f}")
    spent, fails = 0.0, []
    for it in items:
        if spent + unit_cost_probe > budget:     # 5. stop before crossing
            print(f"cap reached — spent {spent:.4f}, rest not run"); break
        out, cost, reason = generate(it)         # returns the actual charge
        spent += cost or 0.0
        if out is None: fails.append((it, reason))
        print(f"spent ${spent:.4f}")             # 4.
    print(f"{len(fails)} failed: {fails[:5]}")
```

## 5. Keep unit prices out of the body text

Prices go stale, and a stale price looks like compliance with "don't guess" while giving the wrong number. Keep procedure in the body
and numbers in an **appendix with measurement dates**. If you hard-code a price constant, **comment the date you checked it** — an undated
price constant is indistinguishable from a guess. Appendix values are a starting point; the evidence is always the one item you just ran.

## Related

- `shipped-config-mismatch` — why changing settings (resolution, steps) invalidates a measurement
- External: [`runaway-guard`](https://github.com/sickn33/agentic-awesome-skills/tree/main/plugins/agentic-awesome-skills/skills/runaway-guard) — cost safety for production code (loop bounds, retries, concurrency, provider caps). Complementary: it computes cost from list prices; this skill insists on measuring one item, because list-price estimates were off by 10× here
