# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Not evidenced in the provided files (to be filled by the architecture team)

## Context and problem

After an order is placed, Online Boutique must do three things:

1. Send the order confirmation email through **MailFast**, a new external email provider. MailFast replaces the mock `emailservice` logic.
2. Award loyalty points through a new **`loyaltyservice`**, which still has to be built.
3. Record an **"order placed" analytics event** for the marketing team.

Marketing also expects **more post-order actions next year**: SMS, fraud scoring and a recommendations update.

The problem: how should the order-placing flow trigger these post-order actions? There are three now and more are planned. One of them depends on an external provider we don't control (MailFast).

Not given in the requirement, so not assumed here: current order volume and peak load, checkout latency targets, MailFast's SLA and rate limits, whether loyalty points must be awarded exactly once, the team's experience with message brokers, and the infrastructure budget. See **Open questions**.

## Decision drivers

Ordered by importance as read from the requirement. **[Inferred]** The requirement doesn't rank them.

- **Modifiability / extensibility:** adding SMS, fraud scoring and a recommendations update next year should not require changing the order-placing code every time.
- **Availability of order placement:** a failure or slowdown at MailFast, `loyaltyservice` or analytics should not make a customer's order fail. **[Inferred]**: the requirement doesn't state this explicitly.
- **Reliability of each action:** confirmation email and loyalty points should not be lost silently. Correctness rules for duplicates (e.g. points awarded twice) are not given. See **Open questions**.
- **Performance (checkout latency):** post-order work should not add noticeably to the time the customer waits. No latency target is given.
- **Operability and cost:** new infrastructure needs to be run, monitored and paid for. No budget or team-skill information is given.

## Considered options

1. **Synchronous calls from the order flow.** The service that places the order calls MailFast, `loyaltyservice` and analytics directly, one after another, before returning to the customer.
2. **Asynchronous fire-and-forget calls from the order flow.** The same service starts the calls in the background (e.g. goroutines/threads) and returns without waiting for them.
3. **Publish an `OrderPlaced` event to a message broker (publish/subscribe).** The order flow publishes one event. Separate consumers (email sender for MailFast, `loyaltyservice`, analytics) subscribe and act independently.
4. **Transactional outbox + message broker.** Like option 3, but the event is first written to an outbox in the same store transaction that records the order, and a relay publishes it to the broker. That guarantees an event exists for every recorded order.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls | Simplest to build and debug. No new infrastructure. Customer gets immediate success/failure for each step. | Checkout latency is the sum of all calls. Any failure (e.g. MailFast down) can fail or stall the order. Each new action (SMS, fraud, recs) means editing and redeploying the order flow: poor modifiability. |
| 2. Async fire-and-forget | Low checkout latency. No new infrastructure. | Work is lost if the process crashes or restarts. No built-in retry or backlog. Still tightly coupled: each new action is a code change in the order flow. Hard to observe failures. |
| 3. Broker pub/sub | Order flow is decoupled from consumers. New actions are new subscribers with no order-flow change: high modifiability. Consumers retry independently, and the broker buffers bursts and absorbs MailFast outages or slowness. Low checkout latency. | New infrastructure to run, secure and monitor (cost, operability). Eventual consistency: email and points arrive later, not at checkout. Consumers must handle redelivery (at-least-once). Order flow can still lose an event if it crashes between recording the order and publishing. |
| 4. Outbox + broker | All benefits of option 3. No lost events between "order recorded" and "event published". | Needs a durable store for orders and the outbox, plus a relay process. Most moving parts. Is there a persistent order store at all? Not evidenced in the provided files. |

## Decision

**Chosen: Option 3, publish an `OrderPlaced` event to a message broker, with one consumer per post-order action.** Option 4 (transactional outbox) stays as a follow-up if the open questions show that losing an event is unacceptable.

Why:

- It best fits the top driver, **modifiability**: SMS, fraud scoring and recommendations next year become new subscribers to the same event. The order flow does not change.
- It isolates order placement from **MailFast** and the not-yet-built **`loyaltyservice`**. If either is slow or down, events wait in the broker and are retried. The customer's order still succeeds.
- It keeps **checkout latency** low because the order flow only publishes one message.
- Option 1 couples availability and latency to every downstream system and grows worse with each new action. Option 2 saves latency but silently loses work and keeps the coupling. Option 4 is stronger but adds a persistence requirement we can't confirm from the requirement alone.

The specific broker product (e.g. a managed pub/sub service, RabbitMQ, Kafka) is **not chosen here**. That depends on team skills, hosting and expected volume, none of which are given (see **Open questions**).

## Consequences

- **Positive:** New post-order actions can be added as independent subscribers without changing order placement (quality attribute: **modifiability**).
- **Positive:** Orders keep succeeding when MailFast, `loyaltyservice` or analytics are unavailable. Their work is queued (quality attribute: **availability**).
- **Positive:** Checkout response time no longer includes email, loyalty or analytics calls (quality attribute: **performance**).
- **Positive:** Each consumer can be scaled, deployed and retried on its own (quality attribute: **scalability**, **deployability**).
- **Negative:** A broker is new infrastructure to deploy, secure, monitor and pay for (quality attribute: **cost**, **operability**).
- **Negative:** Email and loyalty points are eventually consistent. The customer may see them seconds or minutes after the order (quality attribute: **user-perceived consistency**).
- **Negative:** End-to-end flow is harder to trace across async hops (quality attribute: **testability**, **observability**).
- **Risks and mitigations:**
  - *Duplicate delivery* (at-least-once) could send two emails or award points twice. → Include a unique order ID in the event and make consumers **idempotent**, e.g. `loyaltyservice` records processed order IDs. Whether "exactly once" is a hard requirement is an open question.
  - *Lost event* if the order flow crashes after recording the order but before publishing. → Accept for now, or adopt **Option 4 (outbox)** if this is unacceptable.
  - *Poison messages / repeated failures* at MailFast. → Retries with backoff, a dead-letter queue, and alerting.
  - *Throttling by MailFast*, if it limits request rate (unknown). → The email consumer controls its own send rate, and the queue absorbs the backlog.
  - *Event schema changes* break consumers. → Version the `OrderPlaced` schema and add fields in a backward-compatible way only.

## Validation

- **Load/failure test:** place orders while MailFast (or a stub of it) is down or slow. Orders must still succeed, and emails must be delivered once MailFast recovers.
- **Duplicate test:** redeliver the same `OrderPlaced` event. Loyalty points must be awarded only once per order.
- **Metrics:** checkout latency before vs. after, broker queue depth and consumer lag, dead-letter count, email delivery success rate.
- **Extensibility check:** add a dummy subscriber (e.g. a logging consumer standing in for SMS) without changing the order flow.
- **Review date:** revisit before next year's SMS / fraud scoring / recommendations work, or once volume and MailFast limits are known. Target thresholds are not given and must be set (see **Open questions**).

## Open questions

- What are the current and peak order volumes, and the checkout latency target?
- What are MailFast's SLA, rate limits, retry/idempotency support and cost per email?
- Must loyalty points be awarded **exactly once**? What is the business impact of a duplicate or a missed award?
- Is losing an occasional post-order event acceptable? (This decides Option 3 vs. Option 4.)
- Does Online Boutique have a durable order store where an outbox could live? Not evidenced in the provided files.
- Which service currently places the order and calls the mock `emailservice`? Not evidenced in the provided files (no code was read in this run).
- Is fraud scoring (planned next year) meant to **block** an order before it is confirmed? If so, it can't be a post-order async subscriber and needs a separate decision.
- What messaging technologies does the team already know and operate, and is there a hosting or budget constraint (managed vs. self-hosted broker)?
- How fast must marketing see the analytics event (real-time vs. daily)?
