# Decisions

The trade-offs, in the order they mattered — including the ones that turned out wrong.

← [back to overview](../README.md)

---

## Pick models on measured cost per turn, and one expensive one on purpose

**Decision:** GPT-6 Luna for chat, vision and generation; `gpt-4o-mini` as the fallback; `gpt-4o` for
derivation walkthroughs only.

For most of the product's life the tutor ran on `gpt-4o-mini`, and staying on it was an economics
call, not an availability one: newer options were priced at 1.5× to 12× per token, and the free tier
was built on the existing rate. Model choice also turned out to be two-dimensional — model *and*
reasoning effort. One candidate's time-to-first-token ranged from 0.65 s to 95 s on that setting
alone, which would be fatal for a streaming tutor.

The switch to Luna happened when the arithmetic flipped. On real traffic it costs **$0.00057 per chat
turn against $0.00115** for `gpt-4o-mini` — about half — with reasoning effort pinned to `none` in one
place so nobody ships a 95-second tutor by accident. Those are different weeks of traffic, not a
controlled A/B, and Luna's replies ran shorter; the numbers are quoted with that caveat.

What the switch did **not** come with is a re-run of the teaching-quality evals. Cost and request
acceptance are measured in production; whether Luna teaches Class 12 Physics as well is not yet
measured, and the write-up says so rather than implying otherwise. Two production bugs came with the
switch and are written up in [silent failures](silent-failures.md): blank replies from a leftover tool
definition, and chapter tests that stopped generating because one long reply ran past its token limit.

Derivations stay on `gpt-4o`, because the smaller model drops algebra terms mid-proof and a derivation
is cached platform-wide for 30 days — one bad generation would teach the same wrong step to every
student who opens that chapter for a month. Paying more for the one output that gets copied into an
exam answer is the trade, and moving it needs its accuracy re-checked first.

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

Five cases where a confident, plausible prediction was simply wrong:

| Expected | Measured |
|---|---|
| Moving prompt modules after the history improves cache hits | **8.9 points worse** — tail content can never be cached |
| The model will report metadata through a tool call | **0% of the time**, across 12+ prompt configurations |
| Instructing the model to replace its closing question will work | Never complied — four other blocks outvoted it |
| A new model will ignore an unused side-channel tool, like the old one did | It answered with the tool call **instead of** a reply on 3 of 7 turns |
| A dedicated prompt block will fix a mis-read numerical ("10 cm *from the focus*") | **0 of 8** correct, twice — fixed only by parsing the givens in a separate call and doing the conversion in TypeScript (0/8 → 10/10) |

None of them raised an error: types passed, tests passed, the UI looked correct, and the feature
was dead or quietly wrong. That combination is the argument for measuring — a loud failure teaches you something for
free, and these don't.

---

## Let only human-reviewed labels become evidence

**Decision:** a wrong answer counts as evidence of a specific misconception only when a person has
reviewed what that wrong option means.

Generated chapter-test questions now label their wrong options with the documented misconception
each one embodies ("current gets used up in a resistor"). That makes a wrong answer carry meaning —
but a model-written label is a guess, and a system that counts guesses as evidence will confidently
tell a student they hold a belief they don't.

So every generated label is stored as **unreviewed** and produces nothing. A command-line review tool
promotes labels one at a time, with a named reviewer and a timestamp, and deliberately has no bulk
"accept all" mode — the moment it gets one, the feature becomes laundered model output with a
human-shaped wrapper. Most wrong options are expected to stay unlabelled: an arithmetic slip has no
diagnosable meaning, and labelling it would fabricate evidence.

Two further rules keep the count honest. A misconception is only called *recurring* after it shows up
on **different** questions — chapter tests are cached for 90 days, so the same question answered wrong
three times is one piece of evidence, not three. And the tables holding this evidence have **no
client write policy at all**: a browser cannot author a claim about a student's understanding,
including its own.

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
