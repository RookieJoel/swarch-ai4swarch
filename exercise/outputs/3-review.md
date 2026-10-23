> Reviewer: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`)

# ATAM-style review: ADR-0002 "Publish an OrderPlaced event through RabbitMQ"

Reviewed file: `exercise/outputs/2-write-adr.en.md`. Evidence allowed: that ADR and `docs/scenario.md`. Code line citations in the ADR (`main.go` l. …) could not be checked under these rules; the scenario confirms the synchronous chain and the "email failure only logs" behaviour.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | Customers | Place orders | 11.11 peak, 200 orders/s, persistent publish with confirms inside `PlaceOrder` | Order returns after payment, shipping and publish | p95 `PlaceOrder` ≤ 500 ms (today approximately 300 ms → 200 ms headroom for publish + retry) |
| QA2 Correctness | `checkoutservice` pod or RabbitMQ | Crashes / unreachable right after the card is charged | Normal or peak load | Order succeeds; loyalty still awarded | Points awarded **exactly once** within **1 min** for 100 % of successful orders |
| QA3 Availability + external limit | MailFast | p99 8 s, 0.5 % failures, 50 emails/s cap | Peak 200 orders/s | Emails queue and drain; no order fails | 0 failed orders; outbound ≤ 50/s; email delay ≤ a target the ADR does not set |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | On publish failure checkout logs and returns success (Decision, bullet 2). No event → no points for that order. | Scenario demands loyalty **exactly once**. Idempotent consumer only protects against duplicates, not against a missing event. QA2 fails. |
| **Risk** | Mitigation is "log failed publishes for a reconciliation job", but checkout has no store and no order record exists anywhere else in the design. | The Lost-event metric in Validation ("reconciliation finds no order without an event") has nothing to reconcile against. Top risk is effectively unmitigated. |
| **Risk** | The ADR defers its key choice to "Open questions" four times (Deciders, Decision, Risks), but the file has **no Open questions section**. | The lost-event / outbox decision is left with no owner and no record. The board cannot see what is still undecided. |
| **Trade-off point** | "Publish failure never fails the order" (availability) vs "points exactly once" (correctness). | Both are hard constraints. The ADR picks availability silently. Note: the scenario only says an order must not fail because **email, loyalty or analytics** is down; it says nothing about the broker. Treating RabbitMQ as a critical dependency is a legitimate alternative the ADR does not weigh. |
| **Sensitivity point** | Publisher-confirm timeout and "short bounded retry" inside `PlaceOrder`. | Each retry eats the 200 ms headroom. No values given, so QA1 is untestable as written. |
| **Sensitivity point** | Peak duration × (200 − 50) emails/s. | The 9,000-email / 3-min example assumes a 1-min peak, which the scenario does not state. A 1-hour peak ≈ 540,000 queued emails, ≈ 3 h behind; drives broker disk sizing and customer-visible delay. |
| **Sensitivity point** | How "≤ 50/s across all replicas" is enforced; retries of the 0.5 % failures share the same budget. | Scaling the email consumer breaks the MailFast limit unless there is a single consumer, a shared limiter, or 50/N per replica. Not specified. |
| **Non-risk** | One queue per action. | MailFast backlog cannot delay loyalty, so the 1-minute target is independent of email. |
| **Non-risk** | `loyaltyservice` dedupes on `order_id` (unique constraint, same transaction, ack after commit). | Correct exactly-once *effect* for redelivered messages. |
| **Non-risk** | RabbitMQ over Kafka. | 200 msg/s, no replay requirement, two devs know RabbitMQ, self-hosted. Well justified by the scenario. |

## 3. Missing or weak

- **Missing option: hybrid.** Synchronous idempotent gRPC call to `loyaltyservice` (keyed by `order_id`, short timeout, fallback to the queue), with email and analytics on the broker. Fits the team's gRPC strength and attacks the one exactly-once action directly. Compare with option 5.
- **Missing option: fail-closed publish.** Publish before charging the card and fail the order if RabbitMQ is down. Allowed by the scenario wording; trades checkout availability for guaranteed events.
- **Overstated claim:** "the only option that meets drivers 1–4 together". Option 3 does not fully meet driver 3 (lost-event gap); option 5 meets it better.
- **Left undecided in the Decision:** "fanout/topic" exchange, message format, value of N before DLQ, acceptable email delay.
- **Consequence not stated:** `OrderPlaced` carries the customer's email and items, so broker and DLQ now hold personal data (retention, access).
- **Weak point:** "out-of-order delivery" is listed as a cost, but each order produces one event; ordering barely matters here.
- **Not addressed:** can `loyaltyservice` sustain 200 writes/s at peak to stay within 1 minute? No sizing given.
- **Housekeeping:** the first line (merge notes about run a/run b) is process history, not decision content; Deciders is empty.

## 4. Verdict: **Accept with changes**

1. **Resolve exactly-once vs availability for loyalty inside the Decision:** outbox (option 5), hybrid gRPC call, or fail-closed publish. Pick one and state why.
2. **Restore the Open questions section** (or remove the references to it) and replace the log-based reconciliation with a real source of truth.
3. **Put numbers on the sensitive parameters:** publish timeout + retries within 200 ms, how 50/s is enforced across replicas, and an email-delay target sized for a stated peak duration.
