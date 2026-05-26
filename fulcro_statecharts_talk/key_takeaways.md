# Key Takeaways — Tony Kay on Fulcro Statecharts

**Video:** Statecharts (Fulcro internal team walkthrough)
**Link:** https://www.youtube.com/watch?v=RMONaWbgnA8

## Key takeaways for quick navigation

> Timestamps are approximate (deduced from the SRT). Click the YouTube link
> with `&t=XXmYYs` to jump.

- **00:00** Video-conf preamble — zoom check, screen share. Skip to ~01:00 for content.
- **01:27** Statecharts as immutable values — same philosophy as Fulcro: state-as-value, events mutate to the next value, `process-event!` is "pure but expressions may side-effect, hence the bang."
- **02:30** A state chart is a **declarative nested map**; the constructor function normalizes it into a Fulcro-style table-by-ID — every node gets a document-order index for tie-breaking transitions with the same event.
- **05:00** Why protocols: the SCXML standard leaves data model, event queue, expression language, and chart processor open-ended on purpose. Fulcro-statecharts surfaces each as a swappable protocol.
- **07:00** Six+ protocols enumerated: data model, event queue (send/cancel/receive), execution model, state chart processor (versioned per SCXML revision), invocations, registry, working memory store.
- **15:00** "Simple end" presets — a one-line constructor that fills in a default implementation of every protocol for the common case.
- **20:00** Traffic-light example introduced — declarative chart that compiles to the normalized structure shown earlier.
- **22:00** **Hierarchical states**: multiple states active at once via implicit parent containment (red active → its parent active → its parent active).
- **23:00** **Parallel nodes**: all immediate children active simultaneously. Traffic-light example has 4 parallel branches (E-W traffic, N-S traffic, E-W pedestrian, N-S pedestrian) all live at the same time.
- **26:00** Atomic state sibling rule: one-active-at-a-time **unless** they're in a parallel parent.
- **28:00** Pure-functional operation walk-through: load chart, run event, get new chart. Manual save required — nothing implicit unless you wire up a working-memory store.
- **30:00** **Front-end Fulcro integration**: one-line install (`install!` registers all protocol implementations on the Fulcro app). core.async drives the event queue so `(send! ... :delay 180000)` works transparently. Reload-tab loses state, which is fine for UI.
- **32:00** Future direction (3/4 baked): UI route composition as a state chart. Top-level routes become a chart, each route's component invokes a nested chart on enter.
- **34:00** **Backend distributed — the hard part begins.** Multi-node Clojure cluster (tens of nodes), but a state chart session is a singleton. Where does the chart "run"? Round-robin requests mean any node may receive an event for any session.
- **36:00** Coaching Evan through the design: "give me some options." Walks to **durable queue** (SQS/Kafka/SQL).
- **38:00** Adds the **exactly-once** constraint. Reframes it: external side effects can't be perfectly exactly-once → contract becomes "*expressions must be idempotent*; the system guarantees exactly-once *semantics* under that assumption."
- **40:00** The fragile sequence: load state → run event → save state → mark event delivered → delete from queue. Any partial failure breaks the guarantee.
- **42:00** Audience proposes CAS with timestamps. Tony: still has the "who undoes if the node crashes" problem.
- **45:00** **The insight**: use SQL transactions for everything. Put the event queue AND the working-memory store in the same SQL database. One transaction wraps read-event → mark-delivered → load-state → run-handler → save-state. All-or-nothing.
- **48:00** Row-level lock (`SELECT … FOR UPDATE`) on the event row prevents two threads from claiming the same event.
- **50:00** Concurrency: many threads × many nodes. Configurable threads-per-node. **Randomized shuffle** of candidate events so nodes don't race for the same one.
- **52:00** **Multi-AZ AWS gotcha**: SQL row locks across AZs aren't always reliable in practice → a Redis double-check sits on top of the SQL lock.
- **55:00** **Subscriptions chart** — their largest production chart, 248 lines, expressions kept in a separate namespace so the chart file shows only structure.
- **56:00** **Delayed events**: entering `paid` immediately schedules a month-long-delay "expired" event.
- **58:00** External-system sync (Datomic / HubSpot / Dion payment processor) is *outside* the SQL transaction → all interactions must be idempotent.
- **60:00** **Self-healing**: initial-state condition queries Datomic and routes to `paid` / `grace` / `inactive`. Delete the session, restart fresh, the chart re-derives the right state from the source of truth.
- **62:00** **Pause flag** on a session: useful for debugging stuck production sessions — event delivery system skips paused sessions.
- **65:00** "We've got exactly-once if your state chart doesn't side-effect. As soon as external systems are involved, you're back to ‘is my external process synced with my chart?'"
- **67:00** Why statecharts at all? Their pre-statechart subscription system had ~5% of the features the current one has and was already unintelligible. Statecharts organize the logic so a human can reason about it and (ideally) draw it.
- **68:00** **Visualization gap** — open invitation for contribution. Three paths discussed.
- **69:00** Path 1: convert EDN to xstate JSON (xstate follows the same SCXML standard). Half-baked converter in `dev/.../xstate.cljc`. Paste output into the xstate online visualizer.
- **71:00** Path 2: Fulcro Inspect has a state chart tab using an Eclipse Layout Kernel (ELK) JavaScript port. Works for front-end charts.
- **72:00** Path 3 (wanted): a Java-ELK port to Clojure so backend charts can be visualized. Tony started but didn't finish — **open contribution opportunity**.
- **73:00** Path 4 (stretch wanted): a well-known Pathom resolver exposing chart definitions over the network, so Fulcro Inspect can query a backend, pick a session, and visualize live production charts (dev/prod).
- **74:00** Operational war story: Shaki's null-pointer bug from durable events queued against session IDs whose underlying Datomic data had been wiped. Root cause: the durable-event DB was recently switched from H2-in-memory to persistent Postgres, so seeding-against-fresh-Datomic stopped also wiping the event queue.
- **76:00** Final Q&A wrap-up — "any more questions?"
