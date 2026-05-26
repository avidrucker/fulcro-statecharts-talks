# Key Takeaways — Statecharts Marathon (Tony Kay)

**Video:** Statecharts Marathon — advanced topics
**Link:** https://www.youtube.com/watch?v=06-DMDXxSDM
**Duration:** 3h 28m 50s

## Key takeaways for quick navigation

> Timestamps are approximate (deduced from the SRT). Append `&t=XXmYYs` to
> the YouTube link to jump.

- **00:00** Video-conf preamble — camera off, mic muted, finding the slides. Skip to ~01:00 for content.
- **01:00** Framing: this is the **advanced** talk; quick review of basics first, then deep semantics. Companion exercises with solutions live on Slack `#dev`.
- **03:00** Statecharts since the 80s — defined by **Harel**, refined into W3C SCXML standard. Fulcrologic implementation follows the standard's pseudocode line-for-line so behavior is auditable.
- **05:00** Every node has an ID — explicit or auto-generated. **Auto-generated IDs morph on REPL reload**, which can break integration tests. Always assign IDs explicitly for debuggability.
- **08:00** **Configuration** = the set of currently active states (always includes top + path to leaf). Internally ordered by document order for tie-breaking.
- **10:00** Events are keywords; tokenized at `.` *and* `/` (Tony's extension). Match is prefix-based: `error` matches `error.send.failed`; `error.send` does NOT match plain `error`.
- **13:00** **Two event queues per session**: external (delayed sends, user clicks, network) AND internal (raise, done events, error.execution).
- **20:00** **Protocol architecture refresher** — every concrete choice is swappable: data model, event queue, processor, working memory store, registry, invocations. The previous talk introduced these; this one walks through the Fulcro integration's actual implementations.
- **30:00** Working memory store in Fulcro front-end = swap-on-an-atom; backend = SQL table with byte-array-encoded EDN.
- **35:00** **War story**: backend was originally single-threaded; subscription state charts ran **~90 days behind**. Now uses a thread pool with per-session locking via SQL row-lock + Redis double-check (because multi-AZ AWS RDS doesn't always honor `SELECT … FOR UPDATE`).
- **45:00** Session-lock timeout was once set to a full day — caused an incident. Now 30–60 seconds.
- **55:00** **Event matching tokenization**: trailing `*` is wildcard sugar (equivalent to omitting the trailing token). `*` in the middle is undefined behavior. Use the fall-through pattern like try/catch: specific token-match transition first, generic catchall after.
- **65:00** **Parallel regions** intro. A region = an immediate child state of a `parallel` element. All regions in a parallel are active simultaneously. **You cannot conditionally start a subset.**
- **75:00** When to use parallel vs compound vs separate charts: parallel for orthogonal concerns inside one chart; compound for concerns that transition between each other; separate charts for completely independent concerns.
- **80:00** Brazil routing example uses parallel: one region holds the current route, the other handles "user is leaving a dirty form, show confirmation dialog". Decoupled cleanly via parallel.
- **90:00** **Enabled-transition algorithm** — three conditions required: (1) source state in config, (2) event matches, (3) guard true. Enabled ≠ fires.
- **95:00** Per-atomic-state selection: walk *up* from each leaf, take the first enabled transition (deepest-first). Document order breaks ties on a single level.
- **105:00** Choice node (`elements/choice`) — convenience wrapper that emits a state full of eventless transitions with conditions. Like `cond`/`case` for routing decisions inside a chart.
- **110:00** **Run-to-stable loop**: the processor keeps firing eventless transitions until no more are enabled. A single external event can cascade through many micro-steps via `raise` and eventless transitions.
- **120:00** **Why statecharts at all** — Harel paper context: finite state machines have an **exponential state-explosion problem** for complex systems. Hierarchical composition collapses it. Origin in airline / critical-systems software.
- **130:00** Five-minute break.
- **135:00** **Two event queues, in depth**. External event queue is the protocol-exposed one (delayed sends etc). Internal queue is the algorithm's working queue (filled by `raise` from executable content, by `done.state.X` events when regions finalize, by `done.invoke.X` when invocations complete, by `error.execution` when expressions throw).
- **145:00** **Internal vs. external transition TYPES** — totally different from event types! `:type :external` (default) re-fires source state's on-exit and on-entry on self-loop. `:type :internal` suppresses both. "Unfortunate overloading of words."
- **155:00** **Conflict resolution** in parallel regions: if two enabled transitions have overlapping exit sets (one wants to leave the parallel, the other wants to stay inside), **deeper source wins**, document order breaks remaining ties.
- **170:00** Exit/entry ordering when a transition crosses parallel boundaries: exits run inner-out in document order, then transition's executable content runs, then entries run outer-in in document order. **No duplicates, no random order.**
- **180:00** **Final nodes**. Top-level final terminates the entire chart session. Final inside a parallel region fires `done.state.X` to the internal queue but doesn't terminate; all sibling regions reaching final triggers a `done` on the parallel parent itself.
- **190:00** **History nodes**. `:type :shallow` remembers only top-level sibling; `:type :deep` remembers full nested config. **Transition target must be the history node explicitly** — targeting the parent state will use that state's `initial` instead.
- **195:00** Tony's subscription chart **deliberately avoids history** — payments and invoices are real-world side effects, so the chart should re-derive state from Datomic on restart rather than "remember" where it was. History assumes the chart is the source of truth; when it isn't, don't use history.
- **205:00** **Invocations** = lifecycle-bound child state charts (or anything else via the invocation protocol). Enter the invoking state → child starts. Exit → child receives termination event (not hard-kill). `auto-forward` copies every parent event to child; `finalize` lets parent transform events the child sends back.
- **215:00** Routing system uses invocations heavily: each route's component has a colllocated state chart that auto-starts on entry to the route and cleans up on exit.
- **225:00** **Local data discipline**. Good uses: things that must survive across multiple states for arbitrary time (remembered redirect URL, accumulated wizard answers, retry counters). Bad uses: anything you could compute inline in a guard, anything that should be modeled as a state (e.g. `is-loading`). **Diagnostic: lifetime mismatch — if the data's truth is shorter-lived than the chart, don't store it.**
- **240:00** **Fulcro routing integration walkthrough**: custom nodes emit `parallel` or `state` based on options, on-entry handlers parse URL params and update component query, master/detail done with parallel regions.
- **250:00** **Actor + Alias pattern** (inherited from UISM): bind named actors to component class+ident at session start; declare aliases as named paths on actors. This is what makes one "form state chart" or "report state chart" reusable across every form/report in the app.
- **260:00** **Querying chart config from UI**: must include `[::sc/session-id <id>]` in the component's `:query` or Fulcro/React won't re-render when configuration changes.
- **270:00** **THE BIG NEW FEATURE: async/parking processor.** Background: backend expressions can block (Datomic IO, several seconds, no problem). Browser expressions can't — all IO is async, which means modeling "is this paid?" as a guard requires inventing intermediate `loading` states everywhere, tripling chart complexity.
- **275:00** Tony's "aha" moment came mid-prompt to Claude: he needed **parking** (release the thread, resume on continuation), not blocking. Asked Claude to implement an alternate processor. New processor lives alongside the original, selected via `:async true` on install.
- **285:00** Parking processor proves the protocol architecture: a major engine swap shipped without breaking any existing chart. Charts that don't use async expressions are unaffected.
- **290:00** Trade-off of async: while parked, the chart's intermediate configuration isn't observable to the UI. Fulcro load markers cover that gap for the common case.
- **295:00** Open question Tony flagged: if the parked promise *rejects*, what event fires? Should probably be `error.execution`, currently "underdefined" — feature is new.
- **205:00** "That's all I wanted to talk about." Wrap-up; exercises on Slack; time-box each, peek at solutions if stuck.
