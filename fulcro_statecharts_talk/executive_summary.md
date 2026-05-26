# Statecharts — Tony Kay (Fulcro)

**Source:** Internal team tech-talk recorded over screen-share, posted to YouTube
([RMONaWbgnA8](https://www.youtube.com/watch?v=RMONaWbgnA8)). ~76 minutes,
~50/50 split between presentation and Q&A.

**Audience:** Tony's team at Dataico (Evan, Miguel, Shaki, Valentina are
named). Tony is the library author and walks the team through both *how* his
`com.fulcrologic/statecharts` library works and *why* the distributed-backend
design ended up the way it did.

---

## TL;DR

Statecharts are hierarchical state machines you express as a declarative
nested map. The `com.fulcrologic/statecharts` library treats every concrete
choice (data model, event queue, expression language, processor version,
working-memory storage) as a swappable protocol so the same chart definition
can run in the browser via core.async OR on a 30-node backend cluster with
exactly-once event delivery. The clever bit on the backend is making the
**event queue and the working-memory store live in the same SQL database** so
"read event → load state → run chart → save state → mark event delivered" all
commits or rolls back as one ACID transaction — turning a hard distributed
exactly-once problem into an ordinary SQL transaction problem (plus an
idempotency contract on the chart's expressions). External systems outside
the transaction (Datomic, HubSpot, payment processor) must be reached
idempotently, and complex charts (e.g. their 248-line subscription chart)
get self-healing initial-state logic that re-derives state from a source of
truth on restart.

---

## Salient Points

### Statecharts as immutable values

- A state chart is a **declarative nested map** that the library's
  constructor function normalizes (similar to how Fulcro normalizes its
  state database) — every node gets an ID, plus a `document-order` index for
  tie-breaking equal-priority transitions.
- The chart itself is an immutable value; **events transform one immutable
  state into the next**, mirroring Fulcro's state-as-value philosophy.
- `process-event!` carries an exclamation point because *your expressions*
  may side-effect; the chart-value transition itself is pure.

### Hierarchical + parallel state

- Multiple states can be active at once because parent containment is
  implicit (red active → its parent active → its parent active).
- A **parallel** node makes *all* its immediate children active
  simultaneously. The traffic-light example has four parallel branches:
  east-west traffic, north-south traffic, east-west pedestrian signal,
  north-south pedestrian signal — all live at the same time.
- Atomic state siblings have one-active-at-a-time semantics unless wrapped
  in a parallel parent.

### Everything pluggable behind protocols

The SCXML standard leaves a lot open. Tony surfaces each open question as a
protocol so users can plug in their own implementation:

| Protocol | What you decide |
|---|---|
| **Data Model** | How data is stored & accessed inside the chart |
| **Event Queue** | How events get sent / delayed / cancelled / received (core.async, vector-atom, SQL, Kafka, …) |
| **Execution Model** | What language the chart's expressions are in (Clojure lambdas, JS strings, …) |
| **State Chart Processor** | The algorithm implementing the spec — versioned per SCXML revision so a future 2025 standard can ship as a new namespace without breaking the 2015 algorithm |
| **Invocations** | What a chart can start: another chart, a future, a thread, a remote process |
| **State Chart Registry** | Where chart *definitions* live (RAM vs. DB; constrained to RAM in practice because Clojure lambdas can close over local state) |
| **Working Memory Store** | Where a *running* chart's current value lives |

### Two preset "ends" for common cases

- **simple end** (CLJ): standard 2015 processor, future-capable invocations,
  manual event queue.
- **CLJS end**: standard processor, lambda execution, manual queue, flat map
  data model.
- **Fulcro front-end integration**: one-line install registers all
  necessary protocol implementations on a Fulcro app; uses **core.async for
  the event queue** so delayed events (`deliver after 3 minutes`) work
  transparently in the browser. Working memory is RAM only — reload the tab,
  the chart is gone, which is fine for UI state.
- **Future direction (3/4-baked)**: invocation-driven route composition so a
  Fulcro app's top-level routes are themselves a state chart, with nested
  charts auto-started by the route component.

### The backend distributed problem

The hard part of the talk. State chart sessions are *singletons* — there's
one logical "this user's subscription chart, session-ID 1234" — but the
infrastructure is 10–30+ Clojure nodes (UI/API/compute) processing requests
round-robin. "Where is the chart running?" is the wrong question.

Tony walks Evan (a newer team member) through the design constraints:

1. **Durable queue is necessary** (SQS / Kafka would qualify) — events must
   survive node crashes.
2. **Exactly-once delivery** is required — but Tony reframes it: external
   side effects from chart expressions can't be perfectly exactly-once, so
   the contract is "*expressions* must be idempotent; the system will give
   you exactly-once *semantics* under that assumption."
3. **The sequence is fragile**: load state → run event → save new state →
   mark event delivered → delete from queue. Any partial failure
   (especially a node crash between save and delete) breaks the guarantee.

### The insight: same SQL DB holds queue AND working memory

This collapses the distributed-systems problem to a single SQL transaction:

```
BEGIN TRANSACTION
  SELECT event FROM event_queue WHERE trigger_time < now() AND NOT delivered
    FOR UPDATE             -- row lock, only one thread/node gets it
  UPDATE event_queue SET delivered = true WHERE id = ?
  SELECT working_memory FROM chart_sessions WHERE session_id = ?
  -- run handler (state-chart algorithm) against the loaded state
  UPDATE chart_sessions SET working_memory = ? WHERE session_id = ?
COMMIT
```

All-succeeds or all-fails. If the transaction rolls back, the event row
stays not-delivered and gets retried (because expressions are idempotent,
this is safe).

### Concurrency mechanics

- Multiple threads on multiple nodes call `receive-events` in a loop.
- Configurable thread count per node — typically tens of threads × tens of
  nodes processing events in parallel.
- A **shuffle** randomizes candidate-event order so nodes don't always race
  for the same event.
- **Multi-AZ AWS gotcha**: SQL row locks across AZs aren't always reliable
  in practice, so there's a **Redis double-check** on top of the SQL lock.

### Subscriptions — their largest state chart in production (248 lines)

- Subscription lifecycle handled as a single chart. Expressions live in a
  separate namespace so the chart file itself shows only structure.
- **Delayed events**: entering `paid` state sends a "subscription expired"
  event with a month-long delay.
- **External-system sync**: Datomic is the source of truth for subscription
  state; HubSpot tracking; Dion as the external payment processor. None of
  these are inside the SQL transaction, so all interactions with them must
  be idempotent.
- **Self-healing**:
  - Initial-state condition queries Datomic and routes to the appropriate
    chart state (`paid` / `grace-period` / `inactive`) — so deleting the
    chart session and starting fresh just re-derives the correct state.
  - The `unpaid` branch re-checks Datomic on entry in case a payment landed
    while the chart was out of sync.
- **Pause flag** on a session marks "don't deliver events to this chart" —
  useful when debugging a stuck session in production.

### The visualization gap (open contribution)

Tony actively wants a working Clojure/Fulcro state-chart visualizer but
doesn't have one shipped.

- **xstate JSON path**: xstate (the JS lib) follows the same SCXML standard.
  An EDN→xstate-JSON converter exists in
  `fulcrologic-statecharts/dev/.../xstate.cljc` (half-baked); paste output
  into the xstate visualizer for an interactive diagram.
- **Fulcro Inspect path**: the front-end Inspect already has a state chart
  tab that renders the chart using a CLJS port of Eclipse Layout Kernel (ELK)
  for graph layout — only works for front-end charts.
- **Wanted**: a Clojure-side ELK port (Java ELK exists, Tony started porting
  but didn't finish — explicit "open-source contribution wanted" signal).
- **Stretch wanted**: a well-known Pathom resolver that returns the chart
  definition, so Fulcro Inspect can query a backend, list sessions, and
  visualize live production state charts.

### Why state charts at all

Their pre-state-chart subscription system had **~5% of the features the
current one has and was already unintelligible**. The chart isn't faster or
cheaper — it's a way to **organize the logic so a human can reason about it
and (ideally) draw it**. The hard part isn't the algorithm; it's making the
distributed plumbing robust enough that the readable logic actually runs
reliably.

---

## Outline (rough timestamps relative to start of talk)

1. **00:00–01:00** — Video-conf tech check (zoom level, "bigger / yeah / okay")
2. **01:27–08:00** — Statecharts as immutable values; declarative nested map; normalized internal representation; document-order tie-breaking
3. **08:00–15:00** — The 6+ protocols (data model, event queue, execution model, processor, invocations, registry, working memory) and why each exists
4. **15:00–22:00** — "Simple end" presets for CLJ and CLJS; what a turnkey use looks like
5. **22:00–28:00** — Hierarchical states, parallel states, the traffic-light example walked through transition-by-transition
6. **28:00–32:00** — Front-end Fulcro integration; one-line install; core.async event queue; tab-reload semantics
7. **32:00–37:00** — Why backend distributed is hard; "where does the chart run?"; coaching Evan through the design
8. **37:00–47:00** — Exactly-once semantics; idempotent-expressions contract; the load → run → save → mark → delete sequence and where it can break
9. **47:00–55:00** — Insight: SQL transaction wrapping the queue + working memory; row locking; shuffle; multi-AZ Redis double-check
10. **55:00–67:00** — Subscriptions chart in production: delayed events, external-system sync, self-healing initial states, the pause flag
11. **67:00–73:00** — Visualization: xstate convert path, Fulcro Inspect with ELK, the Java-ELK porting opportunity
12. **73:00–76:00** — Operational war story (Shaki's null-pointer bug from the H2→Postgres switch in dev), final Q&A
