Agent: Codex; exact model identifier not exposed.

Reviewed the staged final ADR at /private/tmp/swarch-ai4swarch-20261026-final/docs/adr/0002-post-order-processing.md, intended for identical installation as docs/adr/0002-post-order-processing.md. All mechanisms remain proposals; no tests or approvals are evidenced.

## Quality-attribute scenarios

1. **Performance/capacity:** Source: sale customers. Stimulus: 200 orders/s, each requiring confirmation. Environment: MailFast p95 1.5 s, p99 8 s, 0.5% failures, limit 50 emails/s. Response: checkout durably records work; bounded email execution leaves loyalty scheduling available. Measure: PlaceOrder p95 ≤500 ms, aggregate MailFast traffic ≤50/s including retries, and loyalty committed within 60 s. Sustained email backlog grows by at least 150/s. Sources: docs/scenario.md, “Quality goals and constraints”; ADR, “Decision.”

2. **Correctness/recoverability:** Source: coordinator crash or expired lease. Stimulus: loyalty commits points but its response/local task completion is lost. Environment: restart and concurrent replay of the same logical order. Response: durable loyalty deduplication returns the original award; recovery completes the pending task. Measure: exactly one points mutation per logical order, no lost pending action, and completion within 60 s of order placement under an explicitly bounded recovery interval. Sources: docs/scenario.md, loyalty constraint; ADR, “Loyalty” and “Crash recovery.” That recovery bound is unspecified.

3. **Availability/timeliness conflict:** Source: loyaltyservice outage. Stimulus: service remains unavailable for more than 60 s. Environment: checkout and durable storage remain healthy during the sale. Response: orders continue and pending awards survive for catch-up. Measure: zero checkout failures attributable to loyalty; the original ≤60 s award target cannot also hold. Source: docs/scenario.md, availability/loyalty constraints. ADR, “Business sign-off,” correctly leaves the exception unapproved.

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Durable per-action state plus atomic loyalty deduplication addresses replay explicitly (ADR, “Decision”). | Sound proposed correctness mechanism; request keys alone would not prevent duplicate points. |
| Risk | Stable order identity and payment/shipping reconciliation are prerequisites but remain open (ADR, “Order-completion boundary”; “Open questions”). | A crash before final persistence could leave paid/shipped orders without recoverable work or cause repeated effects. |
| Risk | Checkout and an accumulating email backlog share durable storage (ADR, “Consequences”). | Email failure can indirectly exhaust storage and prevent successful checkout, challenging the availability requirement despite separate goroutine pools. |
| Sensitivity point | Pool budgets, leases, ready-work indexes and database capacity remain unselected (ADR, “Open questions”). | These determine both loyalty's 60 s deadline and checkout's roughly 200 ms p95 headroom; budgets alone do not prove either. |
| Trade-off point | One coordinator reduces deployment count but couples action releases and process failures (ADR, options B/C). | Operability preference sacrifices independent scaling, availability and modifiability as SMS/fraud/recommendations arrive. |
| Sensitivity point | Peak duration, MailFast retry share and global limiter behavior determine email delay (ADR, “MailFast”). | Capacity is already insufficient before failures; deployment overlap must preserve the aggregate 50/s ceiling. |

## Missing or weak

The ADR candidly distinguishes proposed mechanisms from evidence. However, its database engine, failure policy and recoverable order state model are unresolved, making the central durable-handoff choice incomplete. Source-code claims citing checkoutservice/main.go are **Not evidenced in the provided files**; the scenario independently supports only the synchronous call sequence and warning-only email failure.

Three genuinely different options are compared. A useful missing variant is a database-backed task store with independently deployable workers, retaining the handoff while relaxing coordinator coupling without RabbitMQ. Maintenance savings remain an unmeasured preference. Email backlog storage limits, accepted confirmation delay, analytics delivery deadline, and data retention need explicit acceptance criteria.

## Verdict

**Accept with changes** as a Proposed direction; implementation acceptance remains premature. Priorities:

1. Specify the durable order-attempt state machine, stable identities, database choice and crash/reconciliation behavior, including storage failure after remote effects.
2. Resolve the loyalty-outage contradiction with an explicit business decision; define email delay, backlog capacity, recovery deadlines and escalation ownership.
3. Set measurable resource/recovery budgets and execute the proposed load, crash and outage validation before claiming compliance with 500 ms, 60 s, exactly-once effects or isolation.
