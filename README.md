# payments-platform

A small payments platform built with Java 25 and Spring Boot 4 microservices.
It focuses on the hard parts of a payments backend: two-step transfers with an atomic and idempotent confirmation, event-driven notifications with Kafka, and a domain-oriented architecture.

> **Status: work in progress.** Targeting `v0.1.0`. Services are being built one by one.

## Architecture

Three microservices in a monorepo, each with its own PostgreSQL database:

- **account-service** — accounts, balances and movements; applies the debit and credit of a transfer in one database transaction, at most once.
- **transfer-service** — transfers API: create (and edit), then confirm with an idempotent confirmation; calls account-service and publishes `TransferCompleted` to Kafka.
- **notification-service** — consumes `TransferCompleted` and records notifications, without duplicates.

See [docs/architecture.md](docs/architecture.md) for the diagram, the transfer flow, design decisions and limitations. Event contracts are in [docs/events.md](docs/events.md) and idempotency rules in [docs/idempotency.md](docs/idempotency.md).

## Tech stack

- Java 25, Spring Boot 4.1, Maven (wrapper)
- PostgreSQL, Flyway
- Apache Kafka
- JUnit 5, Testcontainers, ArchUnit
- Docker Compose, GitHub Actions

## How to run

> TODO: Docker Compose setup is not ready yet.

## License

[MIT](LICENSE)
