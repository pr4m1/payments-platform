# Idempotency

> **TEMPLATE — decisions pending.** This document lists the questions that define how `POST /transfers` behaves when a request is repeated, the reasonable options for each one, and their trade-offs. The "Decision" column will be filled in once each rule is chosen.

## Goal

A client may send the same transfer more than once (network timeouts, client retries, double clicks). Money must move **at most once** per logical transfer, and the client must be able to retry safely and learn the outcome.

The client identifies a logical transfer with the `Idempotency-Key` HTTP header.

## Questions and options

| # | Question | Option | Pros | Cons | Decision |
|---|---|---|---|---|---|
| 1 | What identifies a repeated request? | **A.** Key unique per client (client id + key) | Clients cannot collide with or see each other's keys. Common practice in public payment APIs. | Needs a client identity. Without authentication in v0.1 it would be a trusted header (e.g. `X-Client-Id`). | Pending |
| | | **B.** Global key (key alone) | Simplest: one unique constraint. Works without authentication. | Two clients choosing the same key collide (unlikely if UUIDs are required). One client could replay another client's key and read its result. | |
| | | **C.** No key: hash of the request body | No client cooperation needed. | Two legitimate identical transfers (same accounts and amount) would be treated as one. Does not use the required header. | |
| 2 | Same key, different body? | **A.** Reject with `422 Unprocessable Content` (store a fingerprint/hash of the request and compare) | Detects client bugs instead of hiding them. Matches the IETF `Idempotency-Key` header draft. | Must store and compare a fingerprint. It must be computed from the parsed request, not raw bytes (field order and whitespace differ). | Pending |
| | | **B.** Ignore the new body and return the original result | Simplest to implement. | Silently hides client bugs: the client believes a different transfer was executed. | |
| | | **C.** Reject with `409 Conflict` | Also explicit. | `409` is usually reserved for "request in progress" (question 4); using it for both makes them hard to tell apart. | |
| 3 | What does a retry of an already completed request return? | **A.** Replay the stored response: same status (`201`) and same body | Fully transparent for the client: a retry looks exactly like the first answer. | The response (or enough data to rebuild it) must be stored. | Pending |
| | | **B.** `200 OK` with the current state of the transfer | No response storage: rebuilt from the transfer row. Shows the current state. | Status differs from the first response; the client must handle `201` and `200`. | |
| | | **C.** `409 Conflict` pointing to the existing transfer | Explicit that it is a duplicate. | The client must treat an error status as success; retries no longer "just work". | |
| 4 | What if the first request is still in progress? | **A.** `409 Conflict` immediately (optionally with `Retry-After`) | Simple and non-blocking. Matches the IETF draft. | The client must retry later. | Pending |
| | | **B.** Wait until the first request finishes (with a maximum wait), then return its result | The client gets the final answer in one call. | Holds threads and connections; needs its own timeout handling; more complex. | |
| | | **C.** `202 Accepted` with a `Location` to `GET /transfers/{id}` | Fits long-running work; the client polls the status. | Changes the API to an asynchronous model; needs the GET endpoint; more scope. | |
| 5 | How is a race between two simultaneous requests with the same key avoided? | **A.** Database unique constraint on the key: insert an "in progress" record first; the loser gets a constraint violation and reads the existing record | Guaranteed by the database. Works with several instances. No extra infrastructure. | The record must be committed **before** calling account-service (its own short transaction), so records stuck "in progress" after a crash must be handled (question 6). | Pending |
| | | **B.** Pessimistic lock (`SELECT ... FOR UPDATE`) or PostgreSQL advisory lock on the key | Serializes requests with the same key cleanly. | Keeping the lock during the HTTP call to account-service keeps a transaction and a connection open during a remote call. Advisory locks are PostgreSQL-specific. | |
| | | **C.** Check in application code (SELECT, then INSERT) or an in-memory lock | Trivial to write. | Not safe: there is a window between the check and the insert. An in-memory lock does not work with more than one instance. | |
| | | **D.** Distributed lock (e.g. Redis) | Works across instances without touching the transfer table. | New infrastructure and a new dependency; lock expiry is a classic source of bugs. | |
| 6 | What if the service crashes halfway? (e.g. after account-service applied the transfer, before it is marked completed) | **A.** A retry with the same key resumes the transfer: if the record is "in progress" and older than a timeout, call account-service again with the same transfer reference | No extra components. Relies on account-service's unique transfer reference, so money never moves twice. | account-service must answer "already applied" with the result instead of an error. Needs a lease/timeout to tell "in progress" from "stuck". Only heals if the client retries. | Pending |
| | | **B.** Background reconciliation job: periodically finds stale "in progress" transfers and asks account-service for their status | Heals without client action. | More code: a scheduled job and a status endpoint in account-service. | |
| | | **C.** Mark stale transfers as failed / unknown and leave them for manual review | Simplest. Honest about uncertainty. | Money may have moved while the transfer says "failed": an inconsistent state. | |
| 7 | How long are keys kept? | **A.** Forever (stored with the transfer) | Simplest. The transfer is a permanent record anyway; full audit trail. | The table and its index grow without bound. Keys can never be reused. | Pending |
| | | **B.** A fixed retention (e.g. 24 hours) with a cleanup job | Bounded storage. Common practice in payment APIs. | A retry after expiry creates a new transfer. Needs a scheduled cleanup job. Must be documented for clients. | |
| | | **C.** Separate idempotency table with a retention period; transfers kept forever | Separates idempotency from business data; storage stays bounded. | Two tables to keep consistent. | |
| 8 | How are duplicate Kafka events handled? | **A.** Unique constraint on `eventId` in the notification table | Guaranteed by the database. Minimal: one table. | Duplicate detection relies on catching a constraint violation. Ties deduplication to this specific side effect. | Pending (deduplication by `eventId` with a DB unique constraint is already agreed; open: where the constraint lives) |
| | | **B.** Separate `processed_events` table (`eventId` as primary key), written in the same transaction as the notification | Generic: reusable for any future consumer or side effect. | One more table. | |
| | | **C.** Rely on Kafka exactly-once (idempotent producer + transactions) | Fewer duplicates inside Kafka. | Does not cover writes to the database: the consumer still needs deduplication. Not sufficient on its own. | |

## Sub-questions to settle alongside the table

- Is the `Idempotency-Key` header mandatory? What is returned when it is missing (`400 Bad Request`?) and what format is accepted (any string up to N characters, or UUID only)?
- If the first request ended in a **business error** (e.g. insufficient funds), does a retry replay that error, or is it executed again?
- If the call to account-service **times out**, the outcome is unknown: what status does the transfer get and what does the client receive?

## Tests that will prove it works

All tests run against real PostgreSQL (and Kafka where relevant) with Testcontainers.

| Test | Scenario | Expected result |
|---|---|---|
| Concurrency with the same key | 20 simultaneous `POST /transfers` requests with the same key and the same body (started together with a latch). | Exactly one transfer stored; exactly one debit and one credit movement; balances change once. Every response is either the original result or the "in progress" answer chosen in question 4. |
| Same key, different body | A completed transfer, then the same key with a different amount. | The response chosen in question 2; no new movements; balances unchanged. |
| Retry of a completed request | The same request sent twice, one after the other. | The response chosen in question 3; only one debit and one credit movement in total. |
| Crash after the debit | account-service applies the transfer, but transfer-service fails before marking it completed (simulated). Then the client retries (or the job from question 6 runs). | No second movement of money; the transfer ends as completed. |

Related tests that complete the guarantee:

- **Rollback on credit failure** (account-service): if the credit fails, the debit is rolled back and no movement is stored.
- **Duplicate event** (notification-service): the same `TransferCompleted` delivered twice produces a single notification.
