Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ATAM-style review: ADR-0002 "Publish an order placed event to RabbitMQ"

Reviewed: `docs/adr/0002-publish-order-events-via-rabbitmq.md` against `docs/scenario.md`.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | Shoppers | 200 `PlaceOrder` requests/s | 11.11 peak, all services up | Order completes; the outbox insert replaces `sendOrderConfirmation` | p95 `PlaceOrder` ≤ 500 ms (today approx. 300 ms, so the insert and everything else get ≤ 200 ms) |
| QA2 Availability | MailFast / `loyaltyservice` / analytics | Goes down for 30 min | 11.11 peak | Orders still succeed; events wait in queues; backlog drains after recovery | 0 orders failed; 0 events lost; email backlog drains at ≤ 50/s |
| QA3 Reliability | RabbitMQ / outbox relay | Delivers the same `OrderPlaced` twice (relay retry or consumer crash before ack) | Peak, at-least-once delivery | `loyaltyservice` dedups on `order_id` | Points awarded exactly once; p99 order-to-points < 60 s |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | "If the outbox write fails, the order fails", but the write happens **after** `chargeCard` and `shipOrder` (Decision). | The customer is charged and the shipment is created, yet they get an error. Nothing compensates for this. A new PostgreSQL instance is also now on the critical path, which works against driver 1. |
| Risk | The ADR contradicts itself. The Decision adds an outbox and PostgreSQL, but Consequences still say "adds only one confirmed publish", the risks say "consider an outbox once checkout has a store", and the Open question "is an outbox required?" is still open. | The board can't tell what is being approved. PostgreSQL (a second new stateful component) is missing from the cons and from Validation. |
| Risk | Email runs as "a single consumer replica rate-limited to ≤ 50 msgs/s", and MailFast has p95 1.5 s / p99 8 s. | By Little's law, 50/s × 1.5 s ≈ 75 calls must be in flight at once. A single replica processing one message at a time manages about 0.7/s. A single replica also means a restart pauses all email. |
| Risk | Email and analytics consumers are at-least-once with no dedup (only loyalty is idempotent). The MailFast idempotency-key support is still an open question. | Customers get duplicate confirmation emails on retries, and marketing double-counts orders. |
| Sensitivity point | Email consumer concurrency/prefetch and the share of the 50/s limit used by retries (0.5 % failures + backoff). | These decide whether real throughput reaches 50/s and how long the peak backlog (approx. 150 emails/s of peak) takes to drain. |
| Sensitivity point | Outbox relay polling interval, batch size and the number of relay replicas. | These add directly to the 60 s loyalty budget. With several relay replicas and no row locking (e.g. `SKIP LOCKED`), duplicate publishes increase. |
| Trade-off point | Outbox + PostgreSQL + RabbitMQ. | Durability goes up (no lost events), but latency (a synchronous DB write), operability and cost (two clusters for a 4-person team) all get worse. |
| Trade-off point | RabbitMQ vs Kafka. | RabbitMQ is easier to operate and fits the team better. The cost is no replay for analytics and next year's recommendations consumer. |
| Non-risk | One exchange with one durable queue per action. | Each consumer fails on its own. Adding SMS/fraud/recs is a new binding, so driver 4 is met without touching `checkoutservice`. |
| Non-risk | Loyalty idempotency keyed on `order_id` with a unique constraint. | This is the right way to get "exactly once" on top of at-least-once delivery, provided the dedup row and the points are committed in the same transaction (not stated). |

## 3. Missing or weak

- **Option not considered: PostgreSQL as the queue** (outbox table polled by the consumers themselves, without RabbitMQ). The decision already adds PostgreSQL, so dropping RabbitMQ would remove one stateful component. This needs a real comparison.
- **Option not considered: publisher confirms with no outbox**, plus a local retry buffer. This is cheaper, and the ADR's own Risks section originally proposed it.
- **Unstated consequences:** who owns and backs up the new PostgreSQL; checkout fails when PostgreSQL is down; where the relay runs (sidecar or separate deployment) and that it is a single point of failure; trace context must also pass through the outbox row, not only through AMQP headers.
- **Likely wrong:** "Only option that meets drivers 1–4". With the outbox, driver 1 now depends on PostgreSQL being up. Kafka, or PostgreSQL alone, would meet drivers 1–4 just as well.
- **Validation gaps:** no chaos test for PostgreSQL, RabbitMQ or relay outages. No test for duplicate emails. The 50/s check doesn't prove that throughput actually *reaches* 50/s.

## 4. Verdict: **Accept with changes**

1. **Fix the failure semantics of the outbox write.** Either write the order and outbox row *before* charging the card, or treat a failed write as non-fatal with a fallback. Never fail an order that has already been charged.
2. **Make the ADR consistent and include PostgreSQL.** Update Consequences, Risks, Open questions and Validation so they match the outbox decision. Add "PostgreSQL as queue" and "confirms without outbox" as options, and justify why RabbitMQ is still needed.
3. **Size the email consumer and add dedup.** State the concurrency (approx. 75 in flight), use at least 2 replicas sharing a global 50/s limiter, and add idempotency for email and analytics (MailFast idempotency key or a sent-log keyed by `order_id`).
