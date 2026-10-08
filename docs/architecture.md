# Architecture

How a request actually moves through EduGarden, and why the pieces sit where they do.

← [back to overview](../README.md)

---

## The shape of it

```
Browser
  │  localStorage — synchronous read cache, never the source of truth
  │
  ▼
middleware.ts ──────────── route gate. Bounded, fails open.
  │
  ▼
API route (edge or node)
  │
  ├─► gate        consume_energy / consume_photo / consume_board_exam   (Postgres RPC)
  ├─► cache       content_cache lookup                                  (Postgres)
  ├─► prompt      router → modules → cadence → head/tail assembly
  │
  ▼
OpenAI  ──── SSE stream ────► Browser
  │
  └─► fire-and-forget writes: weakness, mastery, mood, session depth
```

Three rules explain most of the layout:

1. **Anything the client could lie about is decided server-side** — identity, tier, price, model.
2. **Anything on the critical path is bounded and fails open** — a metering outage must not take chat down.
3. **Anything deterministic is computed, not generated** — see [decisions](decisions.md).

---

## Edge and node, split by need

Of the 39 route handlers, 20 run on the edge runtime and 19 on Node. The split is deliberate:

| Runtime | Routes | Why |
|---|---|---|
| Edge (20) | chat, revision, derivations, visualise, diagnostic, solutions… | Streaming, low cold-start, close to the user |
| Node (19) | payments, admin, board-exam, chapter-test pools, cron, email | Need Node crypto, service-role clients, or long generations |

This split has one sharp edge worth naming. The preview-link feature verifies a signed token **in
middleware**, which is edge — so its crypto had to be Web Crypto, not `node:crypto`, even though an
almost identical module elsewhere in the codebase uses the Node API. They cannot share code. The
duplication is the correct answer, and a comment says so, because the "obvious cleanup" of merging
them would break every request on the domain.

---

## The middleware gate, and the outage that shaped it

App routes require a session or an explicit guest cookie. The gate lives in middleware so no page
can forget it and there's no flash of dashboard before a redirect.

Middleware runs in front of **every** request, which makes it the most dangerous file in the repo.
That is not theoretical: the free-tier Postgres project auto-paused, an unconditional `auth.getUser()`
hung, and every URL on the domain returned a middleware timeout — landing page included. Next was
fine. The host was fine. The site was gone.

Two rules came out of it, and they're load-bearing:

- **No auth call unless the request needs one.** Public paths return on the first line, so the
  database is off the critical path of the landing page and every API route.
- **Bounded and fails open.** No session cookie means zero network I/O. A cookie that can't be
  validated inside 2.5 s is let through — the gate is navigation state, and RLS is the real lock.

Degrading to "the app shell loads" beats degrading to "the site is gone."

---

## Assembling a prompt per turn

The tutor's system prompt is not one document. It's a lean always-on core plus situational modules,
selected per turn by a heuristic router.

```
classifyTurn(messages)
   → mode:    greeting | confused | numerical | exam | emotional | derivation | …
   → modules: only the blocks that mode needs

decideCadence(messages, mode, chapterId)
   → a deterministic budget: is a board hook permitted this turn?
     a micro-win? a curiosity hook? which analogy domains are already spent?
```

Pacing rules an LLM cannot honour from memory — *"a board-exam hook roughly one turn in three,"*
*"never reuse an analogy in a session"* — are computed in TypeScript and injected as a budget,
rather than written as instructions and hoped for.

### Position inside the prompt is behaviour, not style

Two findings here cost real measurement time and are worth stating plainly, because both failed
**silently** — types passed, tests passed, the UI looked right, and the feature was simply dead.

**A personality override must come last.** With the block sitting around 43% of the way through the
prompt, the module blocks below it re-asserted the default teaching pattern, and the model quietly
demoted a "board answer examiner" specialist into a tone of voice. All four tutors answered a given
turn near-identically. Moved to the end, the specialists fire correctly.

**A curiosity hook must be additive, not a replacement.** The first design told the model to close on
an open question *instead of* its usual comprehension check. It never once complied — four other
blocks in the prompt all demand a closing question, and four beat one. Reworded as *one extra line
after* the check, it fires reliably. That's also the better pedagogy.

### Cache-aware message layout

The model call sends **two** system messages: stable content before the conversation history,
per-turn content after it.

```
system  (stable: core prompt, modules, chapter structure)   ← cacheable prefix
history (grows by append)
system  (per-turn: profile, cadence, behavioural blocks)    ← changes every turn
```

Providers bill only tokens after the first byte that differs from a cached prefix. Per-turn blocks
placed *before* the history invalidated the entire conversation on every single turn.

A tempting follow-up — move the router modules into the tail too, since a mode change truncates the
cached prefix — was A/B tested against the live API and **lost by 8.9 percentage points.** Content in
the head gets cached sometimes; content in the tail can never be cached at all, because it always
sits behind a history that just grew. Measured cache hit rate: 64.1% with modules in the head,
56.7% with them in the tail.

---

## Three layers of caching

| Layer | Scope | Dedupes |
|---|---|---|
| `localStorage` | one device | repeat views by the same student |
| `content_cache` (Postgres) | the whole platform | the same chapter requested by different students |
| Provider prompt cache | one conversation | the stable prefix across turns |

Only deterministic generations are shared platform-wide. Chat is never cached. Board papers are
never cached, because "new paper" has to mean a new paper. Weakness-targeted practice is never
cached, because it's personal by construction.

One guard matters: **only validated output is stored.** A parse failure written into a shared cache
would poison every student on that chapter for the full TTL.

---

## When the model won't do what you asked

The tutor was designed to report per-turn metadata — explanation style, answer quality, topic,
whether mastery was demonstrated — through a tool call, so nothing leaks into the student's view.

`gpt-4o-mini` called that tool **0% of the time.** Not incapable: given a *functional* tool it calls it
reliably. It simply drops a pure side-channel tool that doesn't help it answer the student. Forcing
the call suppresses the visible reply instead.

Everything downstream — weakness scoring, the mistake notebook, mastery tracking — was silently
dead. The fix is a second, tiny extraction call after the stream completes, and it's gated hard:

- **Only on answer turns.** It runs only when the previous assistant message actually posed a
  question. Ungated, the extractor hallucinated "correct on first try" on plain explanation turns.
- **The question is passed in.** Without it, the extractor infers correctness from the *tutor's
  tone*, and a gentle correction reads as praise — roughly 40% misfires. With it: 6/6 correct
  answers classified correct, 6/6 wrong answers classified wrong.
- **Bounded at 2.5 s**, so a hung call can't hold a student's stream open. On timeout: no signals
  that turn.

The tool stayed in the request, unused, as a forward-compatible path. That turned out to be the
mistake. After the move to GPT-6 Luna, the new model *did* call it — **instead of** replying. On 3 of
its first 7 production turns the student got an empty bubble and a "send that again" recovery line,
because the model had answered with nothing but a tool call. The tool is now gone from the request
entirely, and a test pins its absence.

The general lessons: a side-channel the model gains nothing from is one it will drop, a different
model may do the opposite, and neither failure raises an error.

---

## One request shape, any model

Reasoning models reject the classic request: `max_tokens` must be `max_completion_tokens`, and any
non-default `temperature` is a 400. Sent the old way, every call fails — a total outage, and a common
way apps broke when this generation of models arrived.

So call sites keep one shape and a single normaliser rewrites it for whichever model the call
targets. It also sets reasoning effort explicitly, because the default ("medium") means seconds of
silence before the first streamed word and hidden reasoning tokens billed as output on every turn.
The tutor runs at `none`.

The chat route has one fallback: if the primary errors, it retries once on `gpt-4o-mini`. That is a
*different* model family on purpose, so a request-shape problem cannot take both down at once.

---

## Data flow

localStorage is the **read** cache, never the source of truth. Every mutation mirrors to Postgres
fire-and-forget; on sign-in a reconcile pass merges both directions — counters take the max, sets
take the union, daily fields stay local — then pushes the merged result back so other devices
converge.

This is why a cleared cache costs a student nothing, and why the synchronous parts of the UI can
stay synchronous. The predicted score, for instance, is computed on four different surfaces; if any
one of them awaited the network, the same student would see two different numbers on two screens.

→ [Data model](data-model.md) · [Metering & cost](metering-and-cost.md) · [Decisions](decisions.md)
