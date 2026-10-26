**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It had no numbers; it lists latency/availability targets and MailFast's rate limits/SLA as open questions | Yes: 500 ms p95 limit vs MailFast p95 1.5 s (rules out sync calls); 200 orders/s peak vs 50 emails/s limit, so about 150 emails/s must queue |
| Options considered (list them) | A: direct synchronous calls; B: event + message broker (chosen); C: orchestrator service | 1: add to existing sync gRPC chain; 2: background goroutines; 3: RabbitMQ events (chosen); 4: Kafka |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | No. Neither requirement was in its input; broker choice and limits left open | Exactly once: loyaltyservice deduplicates on `order_id` (idempotent consumer, since the broker itself delivers at least once). Rate limit: email has its own durable queue, so excess emails wait instead of failing checkout. Lost-event risk (outbox) left as an open question |
| Anything invented (a number, tool or fact not in the scenario)? | Nothing evident; current flow marked [Assumption]/[Inferred] | "Nobody on the team has used Kafka": the scenario only says two developers know RabbitMQ, so this is not stated. "Kafka is heavier to run" is general knowledge, not from the scenario |


## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | delete some Kafka's cons (no team experience) | The scenario says two developers know RabbitMQ but says nothing about Kafka, so 'no team experience' is invented |
| 2 | Email consumer: changed "≤ 50 msgs/s (prefetch + rate limiter)" to a single consumer replica limited to 50/s, restarted by Kubernetes on failure | A per-replica limiter would exceed MailFast's 50/s limit once emailservice scales to 2+ replicas. One replica keeps the total ≤ 50/s; the queue holds emails during a restart (trade-off: availability of email sending vs simplicity) |
| 3 | Replaced "publish with a short retry; event may be lost if RabbitMQ is down (open question)" with a transactional outbox: checkout writes `OrderPlaced` to an outbox table in a new database, and a relay publishes it to RabbitMQ with retries. | A failed publish would lose the event, so loyalty points would never be awarded, which breaks "exactly once" in the scenario.
