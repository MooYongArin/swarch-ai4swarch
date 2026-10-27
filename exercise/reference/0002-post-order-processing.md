# ADR-0002: Use a durable background coordinator for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Sippakorn

Decision and priorities supplied by the student in conversation; wording edited with Codex assistance. No review-board or business approval is asserted. All new mechanisms below are proposed, not implemented features.

## Context and problem

R-12 requires confirmation through MailFast, points through a new loyaltyservice, and an order-placed analytics event. More actions, including SMS, fraud scoring and recommendation updates, are expected next year (docs/scenario.md:9–14).

Today checkout calls payment, shipping and email synchronously; email failure is logged without failing the order (docs/scenario.md:18). The PlaceOrder implementation charges, ships, empties the cart, builds an in-memory result and attempts confirmation before returning (system/online-boutique/src/checkoutservice/main.go:230–280).

The scenario supplies these constraints (docs/scenario.md:24–30):

| Topic | Constraint |
|---|---|
| Checkout | PlaceOrder p95 ≤ 500 ms; approximately 300 ms today. |
| Peak | 200 orders/s during the sale. |
| MailFast | p95 1.5 s, p99 8 s, approximately 0.5% call failures, maximum 50 emails/s. |
| Loyalty | Points within one minute and exactly once, without duplicate awards. |
| Availability | Orders must not fail because email, loyalty or analytics is down. |
| Team and hosting | Four developers, strong in Go/gRPC; two know RabbitMQ. Existing Kubernetes; preference for open-source, self-hosted components. |

Waiting for MailFast threatens the checkout target. Assuming one confirmation per order, sustained peak arrivals exceed MailFast capacity by at least 150 emails/s before retries. Moving work into the background does not remove this capacity limit.

A durable order/work table and recoverable order-completion transaction are not evidenced in the provided PlaceOrder implementation (system/online-boutique/src/checkoutservice/main.go:230–280). Introducing these is an implementation prerequisite. No performance, reliability or maintenance-cost improvement has been measured.

## Decision drivers

1. **Performance and availability:** keep slow or unavailable post-order destinations outside the checkout execution path, while targeting p95 ≤ 500 ms at 200 orders/s (docs/scenario.md:24–28).
2. **Simple maintenance:** the student's priority is one Go coordinator service to deploy, monitor and debug, rather than separate action worker deployments; this is a design preference, not a measured cost result.
3. **Correctness and recovery:** preserve pending work across crashes and award points only once per order (docs/scenario.md:11,27).
4. **Action isolation:** slow email processing must not delay loyalty or analytics; this is the student's additional guardrail.
5. **Modifiability:** support future actions while accepting that coordinator changes will be required (docs/scenario.md:14).

## Considered options

1. **A — Direct synchronous calls:** checkout calls each post-order destination before returning.
2. **B — One durable background coordinator:** checkout saves pending work; one coordinator service tracks and executes actions concurrently.
3. **C — RabbitMQ fan-out:** separate durable queues and independent action consumer deployments.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Direct calls | Familiar Go/gRPC flow; few new moving parts; immediate results. | MailFast delays checkout; warning-only errors do not preserve required work; new actions extend checkout. |
| B — Coordinator | One action service and central per-order progress; async execution isolates checkout from destinations; fits the student's maintenance priority. | Requires durable state, leases, retries and recovery; shares a process and database across actions; adding actions requires coordinator changes. |
| C — RabbitMQ | Stronger deployment-level isolation and independent scaling; new actions can subscribe independently; some stated team experience. | Broker operation plus consumer deployments and reliable publication add work; duplicate effects and end-to-end tracking still need application logic. |

## Decision

Choose **B: one durable background coordinator**. I prioritize fast checkout and simple maintenance for the four-developer Go team. I accept changes to the coordinator when new actions are added and the weaker process-level isolation compared with independent workers.

The design is intended to meet the checkout target; asynchronous execution alone does not prove the ≤ 500 ms requirement. Durable writes and resource contention must be included in validation.

- **Durable handoff:** before reporting successful checkout, persist the completed order and its three pending actions in one local database transaction. Track status and retry state independently for email, loyalty and analytics, with a uniqueness constraint on each order/action pair. The database engine and schema are still to be selected; no existing order store is assumed.
- **Order-completion boundary:** local task persistence does not make remote payment and shipping atomic. Define durable attempt state, stable retry identities, and reconciliation around those remote effects before release. A crash after payment/shipping but before final persistence must be recoverable without repeating effects or losing work. Saving only at the final return point is insufficient. New database failure/full-storage behavior must be decided; if the durable save fails, checkout cannot acknowledge success as though work were safely recorded.
- **One coordinator service, independent bounded pools:** use separate Go goroutine pools and ready-work selection for email, loyalty and analytics. Never wait for email before scheduling loyalty. Each pool has its own concurrency budget, timeouts, retry schedule and backlog monitoring; a shared first-in-first-out bottleneck must not let email tasks consume all execution slots. This does not remove shared process/database failure risks.
- **Crash recovery:** claim tasks atomically with expiring leases, persist outcomes, and reclaim unfinished work after crashes. Recovery may execute a task again, so downstream idempotency remains necessary. Quarantine repeatedly failing tasks for alerting and controlled replay; do not silently discard them.
- **MailFast:** cap total requests at 50/s, including retries, across all coordinator instances and overlapping deployments. A process-local limiter is sufficient only while a single active instance is enforced; otherwise coordinate the limit globally. Use bounded concurrency and backoff. MailFast latency/failures do not occupy loyalty slots. Provider idempotency or outcome lookup is needed to resolve an accepted email whose response is lost; those capabilities are unknown.
- **Loyalty:** send order_id as the award idempotency key. The new loyaltyservice must enforce that key durably and atomically with the points change, for example through a unique award record in the same transaction. A key passed in a request alone does not prevent double points. A retry after an uncertain result must return the existing award rather than add points again. Client retries must resolve to the same logical order identity.
- **Analytics:** record the event using a stable order/event identity and duplicate suppression supported by the selected sink. Store only the customer/order data each action needs; payload, access and retention are still to be defined.

**Business sign-off is required:** delayed confirmation emails during peak traffic must be acceptable. I also propose that checkout availability takes priority over the one-minute loyalty deadline during a loyalty outage, with durable catch-up afterward. This is a requested exception to the original requirement, not an approved change. Until the business agrees, the original one-minute rule remains the baseline and compliance during an outage lasting longer than that is unresolved. Set explicit delay/recovery limits and escalation responsibility before acceptance.

## Consequences

- **Positive:** checkout waits for a durable handoff rather than slow MailFast or other post-order calls (performance, availability), subject to measured database latency and capacity.
- **Positive:** one coordinator provides a central view of action outcomes and reduces separate service deployments (operability); actual maintenance effort still needs evaluation.
- **Positive:** separate pools protect loyalty scheduling from email latency and backlog; durable action state supports retries (timeliness, reliability).
- **Negative:** one coordinator process and its database remain shared failure/resource domains; a full database or saturated connection pool can affect every action and checkout (availability). Use separate pool budgets, indexed ready-work queries, alerts and capacity tests; do not claim complete isolation.
- **Negative:** future actions require coordinator changes, and independent deployment/scaling is weaker than RabbitMQ fan-out (modifiability, scalability).
- **Negative:** email backlog grows by at least 150/s at a sustained 200 orders/s under the one-email assumption; retries may worsen it (capacity, timeliness). Peak duration, acceptable delay, storage retention and any provider-capacity increase require agreement.
- **Risks and mitigations:** crashes near remote payment/shipping and durable completion can lose or repeat effects; require a reviewed recovery design and fault-injection evidence (reliability). Task durability alone is insufficient.
- **Risks and mitigations:** duplicate execution can double-award points unless loyalty's atomic idempotency contract is implemented and tested (correctness).
- **Risks and mitigations:** keeping checkout available during prolonged loyalty outages conflicts with the stated deadline; obtain explicit approval for the proposed exception and define catch-up behavior (availability versus timeliness).

## Validation

These are proposed acceptance checks; none has been executed.

1. Run checkout at 200 orders/s and measure end-to-end p95 ≤ 500 ms, including durable writes. Simulate MailFast p95 1.5 s, p99 8 s and 0.5% failures. Verify aggregate MailFast requests never exceed 50/s, including retries and deployment overlap.
2. Saturate or stop email processing and verify loyalty/analytics pools still progress. Under agreed normal operating conditions, require order-completion-to-loyalty-commit ≤ one minute. Measure database contention and backlog age as well as worker utilization.
3. Disable each post-order destination and crash/restart the coordinator. Verify accepted orders retain durable pending work, unaffected actions progress, and incomplete work resumes after lease recovery without repeating completed effects.
4. Inject crashes around payment, shipping, durable order/task commit, remote loyalty success and local status updates. Retry requests and expire leases; verify recoverable order state and one points award per logical order. Document unresolved provider outcomes instead of claiming success.
5. Test database unavailability, full storage, poison tasks and controlled replay. Verify the agreed checkout failure/recovery policy, alerts and record retention; do not lose pending actions to cleanup.
6. Model the agreed peak duration and recovery traffic against MailFast capacity. Obtain written business acceptance of email delay and the proposed loyalty-outage exception, with explicit recovery targets, before implementation acceptance.

## Open questions

- Which durable database and order-attempt/completion model will support atomic handoff? How will payment/shipping reconciliation and stable order identity across client retries work?
- What must checkout do if durable storage is unavailable or full, especially after a remote payment/shipping effect has occurred?
- How long does peak traffic last, what email delay is acceptable, and can MailFast capacity increase? What retry-window and idempotency/status-query APIs does MailFast provide?
- What loyalty-outage exception will the business approve, with what recovery deadline and escalation owner? What are the points calculation and cancellation/reversal rules?
- What pool sizes, lease durations, query indexes and database capacity are needed to prevent email backlog from degrading loyalty or checkout? Measurements must determine these values.
- Which analytics sink and event fields are needed, and what duplicate, retention and access rules apply?
