# Silent failures

Seven bugs that passed every test, shipped, and raised no error. For each: what broke, how it hid,
how it was found, and the guard that now makes it fail loudly.

← [back to overview](../README.md)

---

## Why these are the interesting ones

A crash teaches you something for free. These didn't crash. Types passed, the test suite passed, the
UI rendered, and a feature was dead or quietly wrong. They share three causes, and each section below
is an instance of one:

| Cause | The fix that generalises |
|---|---|
| A **mock** checked the shape of a call, not whether the real system accepts it | Test the contract against the real thing (real Postgres, the real route) |
| Each side of a **seam** was correct and tested on its own | Test the writer and the reader together |
| A **guard** existed, but nothing proved it could fail | Mutation-test it: break it on purpose, and a test must go red |

---

## 1. A partial index took the whole assessment write path down

**What broke.** Graded answers are written locally first, then mirrored to Postgres. To make that
mirror idempotent, a unique index was added on `(user_id, source_event_id)` — as a *partial* index,
`where source_event_id is not null`, so older rows without an id were left alone.

Postgres will not infer a partial index as the target of `INSERT … ON CONFLICT` unless the statement
repeats the predicate, and the client library sends only a column list. Every mirror write raised
`42P10: there is no unique or exclusion constraint matching the ON CONFLICT specification`. That is a
statement-level error, so it rejected the **entire batch** — diagnostics, chapter tests and board
papers alike.

**How it hid.** The mirror swallows errors by design, so a database blip can't cost a student a mark,
and local storage is what the UI reads. Every student's own device showed correct marks. The only
symptom was that a downstream feature that reads the mirrored rows never fired. 1,502 tests passed —
the database client was mocked, and a mock knows nothing about conflict-target inference.

**The fix.** A non-partial index. The predicate had bought nothing: NULLs are already distinct in a
unique btree index, so legacy rows were never at risk.

**The guard.** The test now runs against **PGlite** — real Postgres compiled to WASM, no server or
credentials. It applies the actual migration SQL, reads the actual conflict-target string out of the
client module, executes the real statement, and separately asserts that the old index still raises
`42P10`. Also learned: "the migration is applied" (the index exists) is not the same as "the client's
statement can use it". `EXPLAIN` against the real schema shows the inferred arbiter.

**And it's no longer silent.** The mirror still never throws, but every write to a table that holds a
student's durable record now checks the error it gets back. A rejection *from the server* (anything
carrying a Postgres error code) raises one alert per table and code per page load, with no row
contents attached. A dropped connection carries no code and is only logged, so a flaky phone network
doesn't page anyone.

---

## 2. A cooldown that never cooled down

**What broke.** After a student finishes a recommended action, it shouldn't be recommended again for a
while. The UI recorded completion under the action's `signature`. The recommendation engine checked
the cooldown under its `suppressionKey`. Different keys, by design — the suppression key is coarser,
so that finishing "practise topic X" also quiets near-identical variants. They could never match. A
student who finished a recommendation was offered it again on the next render.

**How it hid.** 43 tests passed. The engine's tests pinned the pure contract with the right key; the
writer stored a key; both halves were correct in isolation. Nothing tested the two together.

**The fix and guard.** The writer now takes the action object and stores `suppressionKey` itself, so
passing the wrong string is a type error. A seam test drives the real writer and the real reader
together, including a case that fails if storing the signature ever suppresses anything again.

---

## 3. A full browser cache replaced finished answers with an error

**What broke.** The browser keeps a local cache of AI generations (revision sheets, formula cards) to
avoid regenerating them. Its budget was 4 MB — against a per-site storage quota of roughly 5 million
characters. On a long-used account the cache filled the quota, the progress save that runs *after* a
reply threw `QuotaExceededError`, and the chat's catch-all error handler replaced an answer that had
**already arrived** with "Sorry, something went wrong". On every turn.

A second bug sat underneath: the cache's own "out of space, clean up and retry" path called itself
through a helper, and with nothing expired to clean up it recursed until the stack overflowed.

**How it hid.** Only accounts with enough history were affected, and it reproduced only against a
production build with a filled cache.

**The fix and guard.** One storage-write helper that never throws: on a full quota it frees the
regenerable cache (oldest half, then all) and retries once. Streaks, XP and progress write through it.
A `finalized` flag marks a reply as delivered, after which a bookkeeping failure is reported to error
tracking instead of touching the answer. The cache budget is now 2 MB, and a one-time trim runs on
load for caches written under the old budget.

---

## 4. Switching models quietly broke chapter tests

**What broke.** A chapter test is a 15-question pool generated in one call. After the move to GPT-6
Luna, that single reply ran past its output-token limit, the JSON was cut off mid-question, and parsing
failed — on every generation.

**How it hid.** The route logged "all parse attempts failed", which reads like the model producing bad
questions. The clue was that the structural validator's rejection counts were *empty*: nothing had been
rejected, because nothing had parsed at all. The reply wasn't wrong, it was truncated.

**The fix.** Three parallel batches of five, each aimed at one segment of the chapter, with a 20 s
timeout and no automatic retry (a retry would double the wait inside a 60 s function budget). A
salvage step walks a cut-off reply with a string- and escape-aware brace scanner and keeps every
question that *closed*; a half-written question is dropped, never repaired. Losing one batch now costs
a third of a test, not the whole thing.

**The guard.** A starved pool logs its rejection counts in production, so the next "parse failed" says
which kind it was.

---

## 5. The new model answered with nothing

**What broke.** The chat request carried a side-channel tool for reporting turn metadata, which the
old model had never once called (see [architecture](architecture.md)). It stayed in the request as a
harmless forward-compatible path. Luna called it — **instead of** writing a reply — on 3 of its first 7
production turns. The student saw an empty bubble and a "send that again" line.

**How it hid.** A tool call is a valid completion. No error, no 4xx, and the token log recorded
20–28 output tokens for a "reply".

**The fix and guard.** The tool is removed from the request; metadata comes from the separate gated
extraction call, which was already the only path that worked. A test pins that the chat request offers
no tools.

---

## 6. Generated questions with no correct answer

**What broke.** Generated chapter-test questions passed a structural gate: four options, one key, an
in-range index, distinct options, labelled distractors that make sense. In one live batch **7 of 7
passed and 2 were factually broken** — in both, the correct answer was not among the options at all,
and the key, the options and the explanation agreed with each other. One was a regeneration of a fault
that had been fixed by hand the day before.

**How it hid.** The structural gate cannot decide whether the keyed option is *true*. It never could;
that limit was simply never tested.

**The fix.** A second call solves every question **blind** — it gets the stem and options, never the
key. Two questions that differ only in their key produce a byte-identical verifier prompt, and a test
pins that. The model returns an index or "none of these"; code, not the model, decides accept or
reject. "None of these" is what catches the observed failures, because a verifier that actually
solves them lands on a value that isn't in the list. A failed or timed-out call rejects the question:
an empty slot is a cheaper mistake than a question with no right answer.

On a live re-run it accepted 6 of 7 and rejected the one it should have.

**The honest ceiling.** A verifier on the same model as the generator shares its blind spots. It
reliably catches inconsistency and "nothing here is correct"; it does not make the model better at
physics. Agreement is not proof, and the docs say so.

**The guard.** Mutation testing found the first version's tests were decorative: a mutant that
published *every* rejected question to the shared cache and the question bank passed all 1,297 tests,
because the test checked the order of two calls in the source rather than what was stored. The test
now drives the real route with only the network and storage edges mocked, and asserts what actually
reaches the cache and the bank — including that a verifier outage publishes nothing.

---

## 7. The chat re-rendered every message sixty times a second

**What broke.** Nothing, functionally. The app just felt like mud mid-reply. The typing animation
updates the message list on every animation frame, and the message component wasn't memoised, so
**every** message re-rendered on every frame and re-parsed its markdown and KaTeX. A 20-message
conversation was doing about 1,200 markdown parses a second.

A second bug rode along: the animation advanced a fixed number of characters per *frame*, so a 120 Hz
phone typed at twice the intended speed and committed React state twice as often.

**How it hid.** Profilers are not part of anyone's test suite, and a fast laptop hides it.

**The fix.** Memoise the message component, and make every function prop it receives identity-stable
— handlers take a message id so the parent passes one function instead of a closure per message. A
single inline arrow function silently turns the whole memo off again, so the convention is written
down where the next change will see it. The animation now reads the clock, not the frame count.

---

## And one found while writing this

The diagram verifier — which re-derives the maths and science in 152 chapter figures — had been
reading a component file that the diagram dispatcher was moved out of. 50 of its 1,241 registry checks
failed. Nobody noticed, because it wasn't in CI. It now reads the right file, passes, and runs on every
push.

That is the whole pattern in miniature: a guard that isn't run is not a guard.

→ [Architecture](architecture.md) · [Metering & cost](metering-and-cost.md) · [Data model](data-model.md) · [Decisions](decisions.md)
