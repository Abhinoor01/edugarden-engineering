# Decisions

The trade-offs, in the order they mattered — including the ones that turned out wrong.

← [back to overview](../README.md)

---

## Use the cheap model everywhere, and one expensive one on purpose

**Decision:** `gpt-4o-mini` for chat, vision and generation. `gpt-4o` for derivation walkthroughs only.

Roughly 15× cheaper, and for explaining Class 12 Physics in Hinglish the difference is not
detectable. Derivations are the exception because the smaller model drops algebra terms mid-proof,
and a derivation is cached platform-wide for 30 days — so one bad generation would teach the same
wrong step to every student who opens that chapter for a month.

Paying more for the one output that gets copied into an exam answer is the trade.

**Staying on an older model is an economics decision, not an availability one.** Newer options were
priced at 1.5× to 12× per token. The free tier's viability is built on the current rate. Model
choice here is also two-dimensional in a way that's easy to miss — model *and* reasoning effort. One
candidate's time-to-first-token ranged from 0.65 s to 95 s depending on that setting alone, which
would be fatal for a streaming tutor. Migrating without encoding the effort level ships a
95-second tutor by accident.

---

## Make the meter a database function

**Decision:** usage limits enforced by a `SECURITY DEFINER` Postgres function, not application code.

In-memory rate limiting doesn't survive cold starts or multiple instances, and anything the client
sends can be edited. Putting identity, tier, budget and the atomic increment inside one function
removes every one of those failure modes in a single round trip.

Full detail, including the two ways it has silently regressed: [metering & cost](metering-and-cost.md).

---

## Compute what doesn't need generating

**Decision:** the study planner, the predicted score, the weakness model and the mistake notebook
are pure TypeScript. Zero API calls.

An LLM-generated weekly plan would be slower, non-reproducible, unverifiable, and no better than a
scheduling algorithm — while costing money on every regeneration. Deterministic code can also be
unit-tested, which matters most for the number a student is trusting.

The corollary is that these functions are *pure*, so they're testable and they can run on the client
without a round trip. That's what makes the same forecast appear identically on four different
screens.

---

## Give the score an error bar

**Decision:** the predicted board score reports mastery, confidence and coverage as **three separate
claims**, not one number.

This started as a bug. Every unassessed chapter was assigned a fixed low mastery, which with 70
chapters made the score mostly a constant assertion of failure: a student who aced three chapter
tests read **25%**, and acing every available diagnostic topped out at **37%**. The forecast barely
responded to studying, which makes it worse than useless — it's discouraging *and* wrong.

The rebuild rests on one distinction:

> **An unknown chapter is not a weak chapter.**

Demonstrated failure pulls a chapter down. Never having opened it does not — it gets *projected* at
the rate the student has actually demonstrated, and carries uncertainty instead. Same student, after
the rebuild: **73% ± 7, low confidence.**

Three guards keep it honest:

- **Coverage counts evidence, not chapters.** Without that, one correct answer in each of 70
  chapters reported 100% at high confidence — thin breadth laundered into certainty.
- **The error bar has a floor.** It briefly reached zero and printed `100% ± 0 · high confidence`,
  the one output the feature must never emit. The band represents doubt about the whole syllabus and
  about exam day, and neither of those ever reaches zero.
- **Nothing is predicted before anything is measured.** With no evidence the gauge shows `—`. A
  neutral prior is a real number to the model and a fabricated claim to a student.

A related rule: the raw ranked list of at-risk chapters is never shown. For a new student it sums to
roughly 280 of 370 marks "at risk" — true, and useless, because that isn't a leak, it's the course.
The UI shows only *actionable* leaks: chapters assessed weak, or opened but never tested.

---

## Build voice, then delete it

**Decision, reversed:** speech-to-text and text-to-speech were built, shipped, and removed.

By the time it was removed it had been unreachable for months in **three independent ways** — a
disabled feature flag hid the microphone, the speak call had been stripped from the stream handler,
and the replay button required a state that was only ever set *by pressing replay*. Circular. It
could not fire even if the flag were flipped.

Dead in the product, alive in the codebase, the pricing table, the documentation, and one line of
landing-page copy promising students voice mode. That last part is what made it worth deleting
rather than leaving: the marketing claim outlived the feature.

The costing was also wrong in a way that's worth recording. Voice ran about 1.6× the cost per unit
of text while being charged at parity — so it looked cheap and wasn't. If it comes back it gets its
own allowance, not a share of the text pool.

**Residue left deliberately:** the markdown renderer still threads an optional word-highlight
context through ~15 branches. Unpicking it is a markdown-rendering risk, not a voice one, so it
waits for a dedicated pass. Noted rather than half-removed.

---

## Ship a mascot, then don't mount it

**Decision, deferred:** a full roaming animated mascot rig was built, mounted for one day, and taken
back out.

Not because the code was wrong — because the art wasn't good enough, and a cute character that looks
cheap is worse than no character. The rig, its celebration triggers, and its anchor point all remain
in place, unmounted and harmless, waiting for the art to be redone.

Keeping working-but-unmounted code is a real cost. It's justified here because the blocker is
external to the code and specifically identified. It is *not* justified when the blocker is "we
might want this someday" — which is exactly what voice turned into.

---

## Trust measurement over reasoning

**Decision:** cost and prompt-behaviour changes are A/B tested against the live API. Reasoning about
them is not sufficient.

Three cases where a confident, plausible prediction was simply wrong:

| Expected | Measured |
|---|---|
| Moving prompt modules after the history improves cache hits | **8.9 points worse** — tail content can never be cached |
| The model will report metadata through a tool call | **0% of the time**, across 12+ prompt configurations |
| Instructing the model to replace its closing question will work | Never complied — four other blocks outvoted it |

All three failed *silently*: types passed, tests passed, the UI looked correct, and the feature was
dead. That combination is the argument for measuring — a loud failure teaches you something for
free, and these don't.

---

## Say "traceable", not "protected"

**Decision:** preview links watermark the viewer's name across the page rather than attempting to
prevent screenshots.

There is no browser API for blocking screenshots, and every right-click or blur-on-blur trick is
defeated by the operating system's own screenshot tool. So the feature makes a leak *attributable*
and the documentation says exactly that.

Describing it as protection would be a lie to the person relying on it — which is a worse outcome
than the feature not existing.

---

## Let a blank field break the dev site

**Decision:** missing legal identity fields throw in development, and only log in production.

An unfilled operator address or grievance contact takes the entire local site down — every route,
because the check runs in the root layout. That is aggressive on purpose: it's a legal requirement
that is otherwise invisible until someone reads the published page.

The cost is a confusing failure mode — a blank field reads as "the app is broken" rather than
"paperwork missing" — so it's written down as the first thing to check when every route 500s at
once.

→ [Architecture](architecture.md) · [Metering & cost](metering-and-cost.md) · [Data model](data-model.md)
