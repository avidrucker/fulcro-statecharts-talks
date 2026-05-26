# Statecharts Marathon — Tony Kay (Fulcro)

**Source:** Internal team session recorded over screen-share, posted to YouTube
([06-DMDXxSDM](https://www.youtube.com/watch?v=06-DMDXxSDM)). ~3 hours 28
minutes, framed as **advanced topics** with a fast review of the basics at
the start. Companion exercises (test-driven, with solutions) shared in the
team's Slack `#dev` channel.

**Audience:** Tony's team at Dataico (Luke/Lucas, Jeremy, Nico, Gabo named at
various points). Pace is slower than [the earlier Statecharts walkthrough](https://www.youtube.com/watch?v=RMONaWbgnA8) — this is a deep semantic dive
into the SCXML algorithm and how the library implements it, with each
section pausing for clarifying questions.

---

## TL;DR

A semantic-level lecture on the W3C SCXML statechart algorithm as
implemented by `com.fulcrologic/statecharts`. The thesis isn't "use
statecharts" (the previous talk made that case) — it's **here's how the
algorithm actually decides what to do** so you can write charts confidently
and debug them when they misbehave. The biggest *new* idea in this talk:
**an async/parking processor** Tony added recently so that browser-side
charts can do "blocking" IO in expressions (`fulcro load`, `transact`)
without ballooning the chart into a maze of intermediate await-states. It's
implemented as a swappable processor — the original synchronous processor
stays the default — which validates the protocol architecture from the
previous talk: a major engine swap shipped without breaking a single
existing chart.

---

## Salient Points

### 1. Statechart terminology, refreshed

- Every node in a chart gets an ID. If you don't assign one, the library
  invents one — **and re-evaluating the chart in the REPL re-generates the
  invented IDs**, which can cause weird bugs in integration tests that
  reload expressions. Assigning IDs explicitly also makes debug messages
  legible.
- A chart's **configuration** is the set of states currently active. It
  always includes the top state and the full path down to the leaf
  (because compound-state containment is implicit).
- In parallel regions, multiple leaves can be in the configuration
  simultaneously. The set is conceptually unordered but the algorithm
  internally uses **document order** for tie-breaking.
- Event names are tokenized at `.` and (Tony's extension) `/`; transitions
  match if the transition's token list is a prefix of the incoming event's
  token list. So `error` matches `error.send.failed`, but `error.send`
  does not match `error`. `*` as a trailing token is sugar for "match
  anything from here", but functionally equivalent to leaving the trailing
  token off entirely.

### 2. The protocol architecture, revisited with concrete implementations

The previous talk introduced the protocols. This one walks through the
Fulcro integration's actual implementations:

- **Data Model (Fulcro flavor)**: state-chart-local data is normalized into
  the Fulcro state map under a `[::sc/session-id <id>]` ident; `ops/assign`
  with a vector path can also write to the global Fulcro state map via
  **aliases** (see Actor/Alias below).
- **Working Memory Store**: get / save / delete by ident. Front-end is just
  swap-on-an-atom; backend is a SQL table holding byte-array-encoded EDN.
- **Event Queue**: two queues per session — external (delayed sends, user
  clicks, network responses) AND an *internal* queue for `raise` and
  algorithm-generated events. The external queue is what the protocol
  exposes; the internal queue is part of the processor's loop.
- **Processor**: the SCXML algorithm itself. Versioned per spec revision —
  this is the protocol that enabled the async-parking processor (below).

### 3. Why backend went multi-threaded (a war story)

The first SQL-backed event queue implementation was single-threaded for
simplicity. The team discovered subscription state charts were running **~90
days behind** because the queue couldn't keep up. Switching to a thread
pool meant:

- A configurable number of threads per node, each looping `receive-events`.
- A rule that two threads can never run an event for the *same session*
  concurrently — they'd race on the working-memory write.
- The session lock is enforced by a SQL row-lock on the event row PLUS a
  **Redis double-check** because multi-AZ AWS RDS doesn't always honor
  `SELECT … FOR UPDATE` across availability zones.
- The session-lock timeout was once set to **a day** (!) and caused an
  incident. Now it's 30–60 seconds.

### 4. Parallel regions — definition, conflicts, exit semantics

- A **region** is just a state that's an immediate child of a `parallel`
  element. All regions are active simultaneously; you cannot conditionally
  start a subset.
- Use parallel for **truly orthogonal concerns inside one chart** (e.g. the
  Brazil routing system's two-region parallel: one region holds the
  current route, the other handles "show the user a 'are you sure you want
  to leave?' dialogue when a form is dirty"). Use compound when state
  transitions actually flow between concerns. Use *separate* state charts
  when concerns are *completely* independent (no shared data, no shared
  events).
- **Enabled-transition algorithm** (memorize this):
  1. Source state is in the configuration.
  2. Event matches (or the transition is eventless).
  3. Guard (`cond`) is true.
  All three required → the transition is **enabled**. Enabled ≠ fires.
- **Selection per atomic state**: walk up from each active leaf to find
  the first enabled transition (deepest-first). Document order breaks
  ties on a single level.
- **Conflict resolution between parallel regions**: when two enabled
  transitions in different regions have *exit sets that overlap* (e.g.
  one wants to leave the parallel, the other wants to stay inside) —
  the one with the *deeper* source wins. Document order breaks remaining
  ties.

### 5. Internal vs. External transition *types* — the trap

There are two unrelated concepts both called "internal/external":

- **Event** types (internal queue vs. external queue) — what's covered above.
- **Transition** type (`:type :internal` or `:type :external`, default
  `:external`) — controls whether re-entering a state runs its on-exit and
  on-entry handlers. An external self-loop fires both; an internal
  self-loop fires neither.

Tony explicitly marks transitions `:type :external` even where it's the
default, just to make the intent obvious in charts that intentionally
re-trigger entry handlers.

### 6. Two event queues, one chart, run-to-stable

- The processor pops one external event, then loops **while the internal
  queue isn't empty**, processing eventless transitions until the chart
  reaches a stable configuration (no enabled transitions remain).
- This means a single external event can cause a long cascade — `raise`
  events on entry/exit handlers feed the internal queue, eventless choice
  nodes immediately transition on entry, etc.
- Internal events automatically generated: `done.state.X` when a region
  reaches a final node; `done.invoke.X` when an invoked sub-chart
  finishes; `error.execution` when an expression throws.

### 7. Final nodes and invocations

- **Final node** at the top of a chart → entering it terminates the entire
  chart session (working memory garbage collected).
- **Final node inside a parallel region** → fires a `done.state.X` event
  into the internal queue but doesn't terminate the chart; the final state
  ID stays in the configuration until all sibling parallel regions also
  finalize, at which point the whole parallel parent gets a `done` event.
- **Invocations** are sub-charts (or arbitrary other things via the
  invocation protocol) whose lifecycle is bound to the invoking state.
  Enter the state → invocation starts. Exit the state → invocation
  receives a termination event (not a hard kill). Optional `auto-forward`
  copies every parent event onto the child's queue. `finalize` lets the
  parent transform events the child sends back. Use case: routing makes
  each route's component an invocation, so the component's state chart
  auto-starts on entry and cleans up on exit.

### 8. History nodes

- **Shallow history** remembers only the top-level sibling you were in;
  child substate is lost.
- **Deep history** remembers the full nested configuration relative to
  the parent.
- To trigger history on transition-in, **target the history node
  explicitly**. Targeting the parent state will use that state's `initial`,
  which won't (and shouldn't) be the history node.
- **Tony deliberately avoids history in the subscription chart** —
  because subscription state has real-world side effects (invoices,
  payments), he doesn't want the chart to "remember it was in paid"; he
  wants restart to re-derive state from Datomic via initial-state
  conditions. History assumes the chart is the source of truth; if it
  isn't, don't use history.

### 9. Where to keep data: local-data vs. derived

Tony has been noticing AI-generated charts misuse the local data model.
His rule of thumb:

- **Good** uses of local data: things that must survive across multiple
  states for an arbitrary amount of time. Examples: a remembered "where
  the user wanted to go" URL during a multi-step password-reset detour;
  accumulated wizard answers; retry counters.
- **Bad** uses: anything you could compute in a guard condition
  immediately. Storing `should-retry` on entry and reading it in the cond
  is wrong — just put the computation in the cond. Storing `is-loading`
  is wrong — model the loading as a *state*.
- **Lifetime mismatch is the diagnostic**: if the value's truth has a
  shorter lifetime than the chart, don't store it.

### 10. Actors + Aliases (inherited from UISM)

The Fulcro integration ships an "actor" pattern that makes the chart
parameterizable over UI components:

- At session start, you bind named actors to specific component
  classes-and-idents (e.g. `:actor/report → [CompanyReport [:report/id 17]]`).
- Aliases declare named field paths on actors
  (e.g. `:alias/busy → [:actor/report :ui/busy?]`).
- Inside the chart, `ops/assign` and `ops/get` against an alias resolve
  to "the busy flag on whichever component is bound as the report this
  session." This is what makes a generic "report state chart" *reusable*
  across every report in the app.

### 11. Querying the running chart from your UI

To re-render based on a chart's current configuration, you must include
the chart-session ident in your component's `:query`:

```clojure
{:query [:chart/foo
         ['(::sc/session-id _) '_]      ;; whole chart-session table
         ;; OR for a known session:
         {[::sc/session-id ::well-known] [:configuration]}]}
```

Without this, the component never gets new props when the configuration
changes, so React/Fulcro skips it.

### 12. **THE BIG NEW THING: async parking processor**

This is the section Tony's most enthusiastic about. Background problem:

- On the backend, state-chart expressions can block. `plan-paid-for?` runs
  a Datomic query, the thread blocks ~seconds, the chart sits there waiting,
  great.
- In the browser, **expressions can't block** — all IO is async. So an
  "is the plan paid?" condition can't directly do `fulcro/load!`. You'd
  have to model the IO as an explicit `loading` state with on-entry that
  schedules the load and an event handler for the response — doubling or
  tripling the number of states the chart needs.

The realization: he needed **parking** (release the thread, resume later
on the same continuation point), not blocking. He started drafting a
question to Claude and the answer hit him mid-prompt: ask Claude to
implement an alternate processor that parks on promise-returning
expressions.

The result:

- New processor implementation lives alongside the original, selected
  via `(install-fulcro-state-charts! :async true)`.
- When an expression returns a promise, the parking processor releases
  the thread; when the promise resolves, processing resumes from the same
  point in the algorithm.
- This is the protocol architecture from the previous talk paying off
  *concretely*: a major engine swap shipped without breaking a single
  existing chart. Charts that don't use async expressions are unaffected.
- Trade-off: while parked, the chart's "in-flight" configuration isn't
  visible to the UI. Load markers (Fulcro's load tracking) cover that
  gap for the common case.
- Open semantics question Tony flagged: if the parked promise *fails*,
  what event fires? Probably should be `error.execution` like other
  expression errors, but it's "underdefined" as of this recording.

### 13. Closing thoughts

- "We're done with state charts" — Tony explicitly closes the semantic
  section saying these are all the elements; everything else is composition.
- Encouraged folding: invent your own custom nodes by writing functions
  that return primitive node maps, then let the chart-normalizer splice
  them in. That's how the routing system's `route` macro works.
- Exercises live on Slack; recommended approach: time-box each, peek at
  the solution if stuck. Solutions provided.

---

## Outline (rough timestamps)

1. **00:00–01:00** — Video-conf tech check (camera/mic/share)
2. **01:00–05:00** — Framing: advanced topics, basics review first, exercises on Slack
3. **05:00–22:00** — Statechart fundamentals review: hierarchy, IDs, configurations, document order, parallel intro
4. **22:00–30:00** — Events as tokenized keywords; internal vs external events; executable content
5. **30:00–60:00** — Protocol architecture deep-dive: data model, working memory, event queue, registry, processor
6. **45:00–55:00** — War story: single-threaded → multi-threaded backend; SQL transactions; Redis double-check on multi-AZ; session-lock timeout incident
7. **60:00–75:00** — Event matching with tokenization; dot/slash interchange; wildcards as fall-through
8. **75:00–100:00** — Parallel regions: definition, when-to-use vs. compound vs. separate charts; Brazil routing example with 2-region parallel
9. **100:00–115:00** — Enabled-transition algorithm; per-atomic-state document-order selection
10. **115:00–130:00** — Choice helper, send-after, eventless transitions; run-to-stable loop
11. **130:00–140:00** — Why statecharts at all: Harel's paper, state-explosion problem, airline software lineage
12. **140:00–150:00** — Implementation detail: library code follows W3C pseudocode line-for-line
13. **150:00–165:00** — Two event queues (external + internal); done events; processor loop split between external/internal
14. **165:00–180:00** — Internal vs. external transition *types* (different from event types); on-exit/on-entry on self-loops
15. **180:00–195:00** — Conflict resolution: deeper-source wins, document-order ties
16. **195:00–210:00** — Final nodes (top-level vs. parallel-region); done events
17. **210:00–222:00** — History nodes (shallow vs. deep); why subscriptions deliberately avoids history
18. **222:00–235:00** — Invocations: lifecycle-bound sub-charts; auto-forward; finalize
19. **235:00–250:00** — Local-data discipline: when to store vs. when to compute; AI-generated charts misuse this
20. **250:00–265:00** — Fulcro routing integration: custom nodes as parallel-or-state, on-entry route handling, master/detail with parallel
21. **265:00–275:00** — Actors + Aliases: class+ident binding pattern from UISM; what makes reusable form/report charts possible
22. **275:00–285:00** — Querying chart configuration from UI components; required query shape
23. **285:00–205:00** — **Async parking processor (the major new feature)**; protocol architecture pays off; trade-offs
24. **205:00–208:50** — "That's all I wanted to talk about"; wrap-up
