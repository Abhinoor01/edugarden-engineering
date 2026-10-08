# Metering & cost

How usage is limited, why the meter lives in the database, and how the cost of a conversation
is measured rather than guessed.

← [back to overview](../README.md)

---

## Why this is a hard problem here

The users are school students in India. A free tier isn't a growth tactic — for most of them it's
the only tier they will ever use. So the free tier has to be genuinely useful *and* affordable
indefinitely, and the paid tiers have to stay profitable even when someone uses every unit they
paid for.

That rules out the two easy answers. Unlimited paid plans lose money on the heaviest users, who
are exactly the users a tutoring product attracts. A stingy free tier makes the product pointless
for the audience it exists for.

What's left is metering that's precise, cheap to enforce, and impossible to spoof.

---

## One pool, not per-feature caps

Every AI action draws from a single daily **energy** pool. A chat message costs 1 unit; a revision
sheet, a derivation, or a full mock test costs more, roughly in proportion to what it costs to
generate.

The alternative — a separate quota per feature — was rejected because it makes the product feel
like a maze of arbitrary walls, and because it forces a pricing decision every time a feature ships.
One pool means one number to tune.

Unit prices live in exactly one module. Tuning the *budget* is the lever; re-pricing individual
features is almost never the right move.

---

## The meter is a Postgres function

This is the part worth stealing.

```sql
consume_energy(p_units, p_base_budget, p_weekly_budget, p_premium_budget)
  SECURITY DEFINER
```

Everything security-relevant happens **inside** the function:

| Decided inside the RPC | Never accepted from the client |
|---|---|
| Identity, from `auth.uid()` | a user id parameter |
| Tier, from the subscriptions table | a "premium: true" claim |
| The day boundary (IST) | a client clock |
| Whether the balance covers the request | a client-side check |

The whole thing is one atomic check-then-increment in a single round trip. There's no window
between reading the balance and spending it, so parallel requests can't both slip through.

Three deliberate design choices:

**It fails open.** If the RPC is unreachable, the request is allowed and the failure is logged
loudly. A metering outage taking down every student's tutor is a far worse outcome than a few
uncharged messages.

**The last action may overshoot.** Consumption is permitted whenever the current total is below
budget, even if the action would exceed it. Refusing a student mid-request over a rounding error is
worse than eating the difference.

**The expensive path fails closed instead.** Full board-paper generation is the single most
expensive call in the product, and it has its own weekly per-subject quota that denies on error.
That's the one thing not handed out during an outage.

### The failure mode nobody warns you about

These functions have been silently reverted twice — both times by a *later* migration doing
`drop function` + `create function` to add one feature and carrying an older body forward.

Once, a referral perk clause vanished from the photo gate, so a pass that was supposed to grant a
higher daily photo allowance granted one photo a day for weeks. Once, a smoothed onboarding ramp
was reverted to an older step-function version. Both times the migration file *looked* applied,
because it was.

**The migration files are not the source of truth. The database is.** Before and after any
`drop`/`create` on a gate function, diff the body against what's actually live:

```sql
select proname, pg_get_functiondef(oid)
from pg_proc
where proname in ('consume_energy', 'consume_photo', 'consume_board_exam');
```

A related trap, same family: never call an unqualified extension function inside a
`SECURITY DEFINER` function that sets `search_path`. It resolves fine when you test the body in a
SQL console — whose path is wider — and fails only when called as an RPC. That one broke account
deletion outright.

---

## Purchased units are not a tier

Small top-up packs let a student who runs dry mid-homework buy more units without committing to a
subscription.

The invariant: **a top-up credits units and touches nothing else.** It must never write to the
subscriptions table, or a small purchase would silently unlock premium daily caps and the paid
specialist tutors. Finalisation branches on this before the plan lookup ever happens, and a unit
test pins it.

Two supporting details:

- **A pack's per-unit price must never undercut the monthly plan's per-unit price at full
  utilisation** — otherwise a heavy user rationally buys packs forever and the subscription becomes
  dead product. A test enforces the floor, and it earned its keep: a proposed pack size failed it
  and had to be reduced.
- **Purchased units survive the daily reset**, so they live in their own balance table rather than
  the daily counter. The daily allowance is spent first; only then does the balance drain. Unlike
  the daily pool, the balance must cover a request in full — no overshoot on money someone paid for.

Replay safety comes from a unique index on the payment id, so a repeated verification reports
success without double-crediting.

---

## Measuring cost instead of estimating it

For a long time the cost figures on the internal dashboard were **hardcoded guesses.** The token
logging existed but was wrapped in a development-only branch, so production measured nothing.

Now every call's real usage is persisted — prompt tokens, cached tokens, completion tokens, and the
computed cost — read from the provider's own usage frame rather than estimated from character
counts. The dashboard reports measured cost when data exists and **says which basis it's using**,
because presenting an estimate as a measurement is how you end up confidently repricing on fiction.

What it shows today: a chat turn averages about 8,900 prompt tokens and costs **$0.00057** on GPT-6
Luna (n = 207), against $0.00115 on `gpt-4o-mini` (n = 186) — so the planning assumption the energy
prices were built on is now conservative by more than half. Those figures are what moved the model
choice; see [decisions](decisions.md).

Two details that are easy to get backwards:

- **Cached tokens are a subset of prompt tokens, not an addition.** Treating them as additive
  roughly doubles apparent input cost. There's a test pinning this.
- **The log is anonymous by design.** It answers *"what does a turn cost us?"*, never *"what did
  this student cost us?"* Per-user consumption already lives in the metering counters, which is
  also where the abuse signal is. Most users are minors, and the cheapest way to satisfy a privacy
  obligation is not to collect the data.

### Measure in tokens, charge in messages

Tokens are the accounting reference. The student-facing meter stays one message = one unit.

Token-metering the visible meter would make the same question cost different amounts depending on
invisible conversation length — and would tax precisely the multi-turn guided discovery the tutor is
built around. The variance is bounded anyway, because context summarisation caps prompt growth.

---

## Cutting cost without cutting the product

Three changes, all measured against the live API rather than reasoned about.

**Stop injecting the default persona.** The core prompt already establishes the default tutor's
identity, voice and teaching pattern. Injecting her persona block on top restated all of it, plus a
precedence block explaining that she outranks the core — costing a **measured 1,131 prompt tokens on
every turn.** Since non-paying callers are downgraded to that default persona, this was the *common*
case: most turns paid about 13% of their prompt to say nothing new.

Result: **12.1% fewer prompt tokens, 14.5% lower input cost.** The three genuinely different premium
tutors still get their block.

**Frozen-cut context summarisation.** Older messages are summarised once the conversation grows, but
the cut point advances in buckets rather than every turn. Between re-cuts the summary input is
byte-identical, so a memo returns the same summary with zero extra calls — and the message list only
appends, which keeps the cached prefix alive.

**A platform-wide content cache.** Described in [architecture](architecture.md) — one generation
serves every student on that chapter, 6.7 s down to 511 ms.

Students are still charged energy on a cache hit. The cache reduces *our* cost, not their
allowance — conflating those would mean a student's daily budget silently depended on whether a
stranger had opened the same chapter first.

---

## What the eval harness can and can't referee

The teaching-quality harness is LLM-judged. Three runs of an **identical build** scored 34, 37 and
43 out of 48 at the tutor's normal temperature, with single fixtures swinging from 3/3 to 0/3.

So it catches gross breakage — a leaked internal tag, a dead signal path — and those were stable
across all runs. It cannot referee a change worth a few criteria, and treating it as though it could
produced three wrong diagnoses before that was understood.

For cost questions the referee is an A/B against the real API, reading the provider's reported
cached-token count. Not the harness, and not reasoning.

→ [Architecture](architecture.md) · [Data model](data-model.md) · [Decisions](decisions.md)
