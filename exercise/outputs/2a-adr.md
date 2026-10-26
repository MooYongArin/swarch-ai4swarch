Agent: Claude Code (VS Code extension), model: Claude Opus 5.5 (`claude-opus-5-5`). Run a (zero-shot, context: none).

# ADR-0002: Publish an "order placed" event and process post-order actions asynchronously

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Not given (see Open questions)

## Context and problem

Source: only the requirement text given for this run. No system files were read, so nothing here is based on the current code.

After an order is placed, Online Boutique must:

1. send the order confirmation email through **MailFast**, a new external email provider. MailFast replaces the mock `emailservice` logic.
2. award loyalty points through a new **`loyaltyservice`** that still has to be built.
3. record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year**: SMS, fraud scoring and a recommendations update.

The problem: how should the service that places the order trigger a growing number of follow-up actions? Some of these actions are external (MailFast) and some are owned by other teams (marketing analytics).

- **[Assumption]** One service completes the order and today triggers the confirmation email directly. Which service does this, and over which protocol, is not evidenced in the provided files (context: none).
- **[Inferred]** Each action we add directly to the order flow adds latency and another way for checkout to fail. The external MailFast dependency matters most here.

## Decision drivers

Listed most important first. The order is **[Inferred]** from the requirement, because no quality goals were given.

- **Modifiability / extensibility:** next year's actions (SMS, fraud scoring, recommendations) should not require changing the order-placing service each time.
- **Availability of checkout:** an outage or slowdown at MailFast, `loyaltyservice` or analytics must not stop customers from placing orders.
- **Performance (order latency):** post-order work should not add to the time the customer waits for an order response.
- **Reliability of each action:** an email or points award must not be silently lost. Each action must be retryable.
- **Cost and operational complexity:** keep the new infrastructure the team must run to a minimum.

## Considered options

1. **A: Synchronous direct calls.** The order-placing service calls the MailFast adapter, `loyaltyservice` and analytics one after another before it responds.
2. **B: Event-driven publish/subscribe.** The order-placing service publishes one `OrderPlaced` event to a message broker. Each post-order action is an independent subscriber: a MailFast email adapter, `loyaltyservice` and an analytics consumer.
3. **C: Post-order orchestrator service.** The order-placing service makes one call to a new `postorderservice`, which calls each action in turn, synchronously or with its own retries.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous direct calls | Simplest to build. No new infrastructure. Immediate success or failure for each action. Easy to trace in one request. | Order latency is the sum of all calls. Checkout availability is the product of all dependencies, including external MailFast. Every new action (SMS, fraud, recs) changes the order-placing service. Partial failure (email sent, points failed) is handled inline. |
| B: Event publish/subscribe | Producer is decoupled: new actions are just new subscribers, with no change to the producer. Checkout does not wait for, or depend on, MailFast, loyalty or analytics. Each consumer retries on its own and scales on its own. Analytics gets the event stream naturally. | New broker to run, monitor and pay for. Eventual consistency: email and points arrive "shortly after", not at once. Needs idempotent consumers (at-least-once delivery), a dead-letter queue and an event schema contract. Harder end-to-end tracing. Publishing reliably together with the order commit needs care (e.g. a transactional outbox). |
| C: Orchestrator service | Producer change happens once. The post-order logic sits in one place, which is easy to reason about. Can add retries and ordering, e.g. fraud scoring before email. | The orchestrator becomes a new coupling point that must change for every new action. If it is called synchronously, checkout still depends on its availability. Without a queue it reproduces A's problems one hop later. With a queue it moves towards B but keeps central coupling. |

## Decision

Choose **Option B: event-driven publish/subscribe on an `OrderPlaced` event.**

- After an order is placed, the order-placing service publishes one `OrderPlaced` event (order id, customer, items, totals, timestamp). Then it returns to the customer.
- Three independent consumers subscribe:
  - **email adapter:** replaces the mock `emailservice` logic and sends the confirmation through MailFast.
  - **`loyaltyservice`:** awards the points.
  - **analytics consumer:** records the marketing event.
- Next year's SMS, fraud scoring and recommendations update are added as new subscribers.

Why B fits the drivers:

- It is the only option where adding next year's actions needs **no change to the order-placing service** (modifiability).
- It **removes MailFast and the other consumers from the checkout path**, which protects checkout availability and latency.
- Its main cost, a broker plus eventual consistency, is acceptable **[Inferred]**, because none of the three actions needs to finish before the customer sees the order confirmation.

Caveat: if fraud scoring must **block** an order, it cannot be an after-the-fact subscriber. It would need a separate synchronous decision before the order is accepted (see Open questions).

The broker technology (e.g. RabbitMQ, Kafka, Google Pub/Sub, NATS) is **not chosen here**. It depends on facts we do not have (see Open questions).

## Consequences

- **Positive:** new post-order actions are added as new consumers without changing the order-placing service. (quality attribute: modifiability)
- **Positive:** a MailFast, `loyaltyservice` or analytics outage no longer blocks order placement. Events queue up and are processed when the consumer recovers. (quality attribute: availability)
- **Positive:** order response time no longer includes email, loyalty or analytics calls. (quality attribute: performance)
- **Positive:** each consumer can scale and be deployed independently, and analytics gets a natural event stream. (quality attributes: scalability, deployability)
- **Negative:** a message broker is a new component to deploy, operate and monitor. (quality attributes: cost, operability)
- **Negative:** email and points become eventually consistent, so customers may see them seconds or more after ordering. (quality attribute: usability / consistency)
- **Negative:** end-to-end tracing of one order across several consumers is harder. (quality attribute: observability)
- **Risks and mitigations:**
  - *Lost event if the service crashes between saving the order and publishing:* use a transactional outbox or publish-after-commit with retry. (reliability)
  - *Duplicate delivery (at-least-once):* make consumers idempotent and key them on order id, so there is no double email and no double points. (correctness)
  - *Poison messages or a consumer that keeps failing:* use a dead-letter queue, alerts and replay. (reliability)
  - *MailFast rate limits or outage:* the email adapter backs off and retries, with a circuit breaker. (availability)
  - *Event schema drift as more teams subscribe:* version the `OrderPlaced` schema and only make backward-compatible changes. (modifiability)
  - *The broker itself is a single point of failure:* run it highly available and alert on queue depth. (availability)

## Validation

- **Load / fault test:** with MailFast, `loyaltyservice` and analytics stubbed to be slow or down, order placement latency and success rate stay unchanged compared with a baseline without post-order calls. The target numbers are not given (see Open questions).
- **Metrics:**
  - time from `OrderPlaced` to email sent
  - time from `OrderPlaced` to points awarded
  - consumer lag / queue depth
  - dead-letter queue count
  - duplicate-processing rate (should be 0 effects)
- **Extensibility check:** add a dummy fourth subscriber (e.g. SMS stub) with zero code changes to the order-placing service.
- **Review date:** revisit when the first of next year's actions (SMS, fraud scoring, recommendations) is designed. Not given; to be set by the deciders.

## Open questions

- Which service places the order today, and how does it call `emailservice`? Protocol and code paths are not evidenced, because no system files were in context.
- Who are the deciders and owners: the checkout team, the marketing team, the `loyaltyservice` team?
- What are the quality targets? Order latency (e.g. p99), checkout availability, and the maximum acceptable delay for the email and the loyalty points.
- Are there constraints or limits on MailFast: rate limits, SLA, API style (REST/webhook), cost per email, and whether it offers delivery-status callbacks?
- Which message broker, if any, does the team already run or know? Is there a cloud platform preference (e.g. managed Pub/Sub)? What is the budget?
- Must loyalty points be awarded exactly once and be auditable? Can points be revoked if an order is cancelled or refunded?
- Where does marketing analytics live (an internal store or an external analytics product), and which event fields and retention does it need?
- Must fraud scoring be able to **block** an order before it is accepted? If yes, it needs a synchronous step and cannot only be an `OrderPlaced` subscriber.
- Are there data-protection constraints (PII such as email address and name) on sending customer data to MailFast and to analytics?
