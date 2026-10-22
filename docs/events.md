# Event contracts

> **Status: proposed — pending review.** Field names, types and the topic name may still change until notification-service is built.

There is no shared library between services. Each service defines its own Java class for an event; the contract below (JSON on the wire) is the single source of truth. Producer and consumer must both follow it.

## TransferCompleted

Published by **transfer-service** when a transfer moves to `COMPLETED`: after a successful confirmation, or when a cancellation discovers that account-service had already applied it. Consumed by **notification-service**.

| Property | Value |
|---|---|
| Topic | `transfer-completed` |
| Message key | `transferId` (all events of the same transfer go to the same partition, so they keep their order) |
| Serialization | JSON, UTF-8 |
| Producer | transfer-service |
| Consumers | notification-service (consumer group `notification-service`) |
| Delivery guarantee | At least once: consumers **must** tolerate duplicates |

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `eventId` | string (UUID) | yes | Unique identifier of this event. Generated once by the producer; a redelivered event keeps the same `eventId`. Used for deduplication. |
| `eventType` | string | yes | Always `"TransferCompleted"`. |
| `eventVersion` | integer | yes | Version of this contract. Starts at `1`. |
| `occurredAt` | string (ISO-8601 instant, UTC) | yes | When the transfer was completed, e.g. `"2026-10-20T10:15:30.123Z"`. |
| `transferId` | string (UUID) | yes | Identifier of the transfer in transfer-service. |
| `sourceAccountId` | string (UUID) | yes | Debited account. |
| `targetAccountId` | string (UUID) | yes | Credited account. |
| `amount` | string (decimal) | yes | Transferred amount, always positive, with exactly the number of decimals of the currency (e.g. `"125.50"` for EUR, `"1250"` for JPY). **Never a JSON number**, so that no client parses it as a floating-point value. |
| `currency` | string (ISO 4217) | yes | Currency code, e.g. `"EUR"`. |

### Example

```json
{
  "eventId": "7f1c2b9e-3a4d-4c55-9d1e-2b8f6a0c1d23",
  "eventType": "TransferCompleted",
  "eventVersion": 1,
  "occurredAt": "2026-10-20T10:15:30.123Z",
  "transferId": "c0a8012e-91b2-4f3e-8a7d-5e6f7a8b9c0d",
  "sourceAccountId": "3b9d6e1a-0f2c-4a7b-9c8d-1e2f3a4b5c6d",
  "targetAccountId": "9e8d7c6b-5a4f-4e3d-2c1b-0a9f8e7d6c5b",
  "amount": "125.50",
  "currency": "EUR"
}
```

## One event per transfer (producer rule)

Deduplication by `eventId` only works if each transfer produces a single `eventId`. transfer-service guarantees it: the event is created and published **only by the request that wins the conditional transition to `COMPLETED`**. Any other request (a concurrent retry, or an instance whose lease was taken over) loses that transition and publishes nothing.

Duplicates can still happen on the way (producer retries, consumer rebalances, a crash before the offset is committed), but they all carry the same `eventId`.

## Deduplication rule (Idempotent Consumer)

notification-service processes each event in **one database transaction**:

1. **Claim the event first**, as the first statement of the transaction:
   ```sql
   INSERT INTO processed_events (event_id, processed_at) VALUES (?, now())
   ON CONFLICT (event_id) DO NOTHING
   ```
2. **0 rows inserted:** the event was already processed. Nothing else is done, and the message is acknowledged as successfully processed.
3. **1 row inserted:** store the notifications in the same transaction. One event may produce several notifications (for example, one for the source account and one for the target account).
4. The Kafka offset is committed **only after** the database transaction commits. A crash before that leads to a redelivery, which step 2 absorbs.

Why claiming with `ON CONFLICT DO NOTHING` instead of catching a constraint violation: in PostgreSQL a failed statement aborts the whole transaction, and a "check, then insert" leaves a window in which two consumers both see the event as new. With `ON CONFLICT`, if two consumers process the same event at the same time, the second insert waits for the first transaction: it inserts nothing if the first one committed, and claims the event if the first one rolled back.

Result: each transfer produces its notifications exactly once, even if the event is delivered several times.

## Versioning rules

- **Backward-compatible changes** (adding an optional field) keep the same `eventVersion`. Consumers must ignore unknown fields.
- **Breaking changes** (removing or renaming a field, changing a type or its meaning) increase `eventVersion`. The producer publishes the new version only once consumers support it.
- Consumers must not rely on producer-specific Java class names (for example, type headers added by a JSON serializer); they map the JSON to their own class.
