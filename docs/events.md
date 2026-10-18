# Event contracts

> **DRAFT — pending review.** Field names, types and the topic name may still change. Items marked **TBD** depend on decisions not taken yet.

There is no shared library between services. Each service defines its own Java class for an event; the contract below (JSON on the wire) is the single source of truth. Producer and consumer must both follow it.

## TransferCompleted

Published by **transfer-service** when a transfer has been applied by account-service and marked as completed. Consumed by **notification-service**.

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
| `amount` | **TBD** | yes | Transferred amount, always positive. Depends on the money decision: either a decimal string (`"125.50"`) or an integer in minor units (`12550`). **Never a JSON floating-point number.** |
| `currency` | string (ISO 4217) | yes | Currency code, e.g. `"EUR"`. |

### Example (amount format TBD)

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

## Deduplication rule

Kafka delivers at least once, so the same event can arrive more than once (producer retries, consumer rebalances, a crash before the offset is committed).

1. notification-service stores the `eventId` of every processed event under a **database unique constraint**.
2. Storing the notification and recording the `eventId` happen **in the same database transaction**.
3. If the insert fails because the `eventId` already exists, the event is a duplicate: it is logged at debug/info level, no new notification is stored, and the message is acknowledged as successfully processed.
4. The Kafka offset is committed only after the database transaction commits. A crash before the commit leads to a redelivery, which step 3 absorbs.

Result: one transfer produces exactly one notification, even if the event is delivered several times.

## Versioning rules

- **Backward-compatible changes** (adding an optional field) keep the same `eventVersion`. Consumers must ignore unknown fields.
- **Breaking changes** (removing or renaming a field, changing a type or its meaning) increase `eventVersion`. The producer publishes the new version only once consumers support it.
- Consumers must not rely on producer-specific Java class names (for example, type headers added by a JSON serializer); they map the JSON to their own class.
