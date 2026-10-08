# Data model

Schema shape, access-control posture, and how a synchronous UI sits on top of an asynchronous
database without lying to the student.

← [back to overview](../README.md)

---

## Access-control posture

52 tables, every one with row-level security switched on, governed by 102 policies. Every table falls
into one of four buckets, and the bucket is the design decision:

| Posture | Used for | Example |
|---|---|---|
| **RLS, own rows only** | Everything personal | chat history, progress, weakness, assessments |
| **RLS on, zero policies** | Service-role only | anonymous cost logs, one-time preview passes |
| **World-readable, no client writes** | Shared non-personal content | the platform content cache |
| **No client writes at all** | Anything money or quota touches | usage counters, unit balances, subscriptions |

"RLS on with zero policies" is the useful one. It denies anonymous and authenticated roles
entirely while the service role bypasses it — which is exactly right for a table that no user
should ever read, including their own row.

The consequence worth internalising: **because analytics tables are RLS-scoped to the caller, the
browser physically cannot aggregate across users.** The owner dashboard therefore can't be a
client-side query. It's a server route holding a service-role client behind a signed session
cookie. The service-role key never reaches the browser.

---

## Writes that matter go through functions, not tables

Anything that meters, charges, or grants runs through a `SECURITY DEFINER` function with no client
write policy on the underlying table. Identity comes from `auth.uid()` inside the function; tier
comes from a lookup inside the function. See [metering & cost](metering-and-cost.md) for the full
pattern and the two ways it has silently regressed.

The same shape covers referral claims and grants: the client-side module is untrusted glue, and
every rule that matters — self-referral checks, claim windows, daily reward caps — is enforced
inside the function.

---

## localStorage is a read cache, never the truth

The UI reads a lot of state synchronously: XP, streaks, progress, the predicted score. Making those
reads async would mean spinners on surfaces that should feel instant.

So localStorage is primary **for reads**, and every mutation mirrors to Postgres fire-and-forget.
On sign-in, one reconcile pass merges both directions:

```
counters   → MAX(local, remote)
sets/arrays→ UNION
daily fields→ stay local
              then push the merged result back so other devices converge
```

A cleared cache costs a student nothing. Two devices converge instead of fighting.

### The rule this creates

**The predicted score is computed synchronously on four separate surfaces** — the home card, the
tutor practice panel, the pricing page, and the parent report. If any one of them awaited the
network, the same student would see two different numbers on two screens.

So new inputs to that calculation get *mirrored into localStorage*, never fetched at call time.
Verified mastery works exactly this way: the durable record lives in Postgres, and a local mirror
is refreshed on load so the synchronous engine can read it.

---

## Three schema bugs worth documenting

### "Explored" and "complete" were the same field

A chapter row was created on the **first AI response** — the moment a student opened a chapter and
got a welcome message. Every reporting surface then read that row and called it *completed*.

One welcome reply on Gauss's Law ticked the sidebar, grew the garden plant, removed the chapter
from the study plan, and told a parent the chapter was done.

The strict metric already existed — learned *and* tested above a pass threshold — and exactly one
component used it. The fix was a status column with three states (`explored` / `learning` /
`complete`), one function that owns the completion rule, and an audit of every call site that said
"done."

Some call sites legitimately want "has opened this" — the recency feed, the is-this-student-new
heuristic. Those kept the loose metric and carry a comment saying so, so nobody "fixes" them later.

### A partial index took a whole write path down

A uniqueness index meant to make assessment writes idempotent was created with
`where source_event_id is not null`. Postgres cannot use a *partial* index as the conflict target of an
upsert unless the statement repeats the predicate — and the client library sends only a column list.
Every write raised `42P10`, a statement-level error, so whole batches were rejected.

Nothing looked broken: the client kept its local copy, and students saw correct marks on their own
device. The predicate also bought nothing, since NULLs are already distinct in a unique index. The
full story, and the real-Postgres test that now guards it: [silent failures](silent-failures.md).

### A retired table broke every sign-up for nine days

A table was retired and every `.from()` call removed. Sign-ups then failed for nine days, because a
database **trigger** still inserted into it.

When you retire a table, grep the SQL — triggers, functions, policies — not just the application
code. The application was clean the whole time.

---

## The types file is hand-maintained

The generated-types workflow isn't wired up, so the database types file is maintained by hand. Every
new table needs its row/insert/update block added or the compiler rejects the table name outright.

That's friction, and it's the good kind: it's a compile error rather than a runtime surprise, and it
forces a moment's thought about the shape of a new table before it exists.

---

## Dates are local, deliberately

Every day boundary in the product — streaks, daily budgets, study plans, heatmaps — is computed in
IST, never via a UTC ISO string.

For a student in India, `toISOString()` rolls the date over at 5:30 a.m. local time. A late-night
study session would land on tomorrow, break a streak, and be impossible to explain. The audience is
in one timezone; the correct answer is to use it, and to say so where it's easy to get wrong again.

→ [Architecture](architecture.md) · [Metering & cost](metering-and-cost.md) · [Decisions](decisions.md)
