# Implementation Plan: reservation-test (reservation server, queue server, mock-pg)

**Branch**: `001-reservation-test` (feature id; work happens on `main`) | **Date**: 2026-10-25 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-reservation-test/spec.md`

## Summary

Build three independent Spring Boot 4.1.2 / Java 21 services in one Gradle multi-module build:

- **server** (:8080) — reservation server. PostgreSQL 17 via JdbcTemplate + Flyway. Seat holds with four
  acquisition strategies (conditional / pessimistic / optimistic / naive), optional DB backstop indexes,
  two-step card payment against mock-pg with single query + reversal + recovery scheduler, pending deposit with
  return batches, sale end processing, admission-key verification (HMAC, constant-time, with an independent
  second verifier), async slot notifications to the queue server, admin reset/stats/snapshot/clock.
- **queue** (:8082) — in-memory waiting queue with a periodic admission scheduler, HMAC admission keys,
  busy-slot deferred expiry with busy cap, sale-end / sold-out closing, admin API.
- **mock-pg** (:8081) — in-memory payment gateway with seeded randomness for auth failure, decline, timeout and
  latency; admin config/stats/payments.

Every service has a controllable **server clock** (base + monotonic elapsed + offset). All behavior decisions use it;
scheduler intervals and HTTP timeouts use real time. Technical approach and every choice the spec leaves open are
in [research.md](./research.md); decisions that change observable behavior are also logged in
[`docs/decisions.md`](../../docs/decisions.md) (D-001 …).

## Technical Context

**Language/Version**: Java 21 (OpenJDK 21.0.12 available in the WSL environment)

**Primary Dependencies**: Spring Boot 4.1.2 (spring-boot-starter-web/webmvc, validation, actuator; server adds
spring-boot-starter-jdbc, spring-boot-starter-flyway, flyway-database-postgresql, postgresql driver), Lombok,
Jackson 3 (Boot 4 default), `RestClient` over the JDK `HttpClient` for outbound calls. No JPA, no Spring Security,
no spring-boot-docker-compose.

**Storage**: PostgreSQL 17 (Docker Compose, DB `reservation`) for the reservation server only; queue and mock-pg are
in-memory.

**Testing**: JUnit 5 + Spring Boot Test (`spring-boot-starter-test`), Testcontainers 2.x (`postgres:17`) with
`@ServiceConnection` for server tests; JDK `com.sun.net.httpserver` stubs for mock-pg / queue during server tests;
plain unit tests for clock, admission key, queue core, mock-pg core.

**Target Platform**: Linux (WSL2 Ubuntu) with Docker; services run via `./gradlew :<module>:bootRun`.

**Project Type**: Multi-module backend web services (3 HTTP services).

**Performance Goals**: Not specified beyond correctness under concurrency. Design target: hundreds of concurrent
users (queue `maxActive` 200) against a 20-connection pool without invariant violations.

**Constraints**: Invariants I1–I17 (spec §9) must hold; JdbcTemplate only; time from the server clock only
(DB time functions not used for comparisons); payment gateway calls outside DB transactions; no retries/backoff for
PG calls beyond §6.6; nothing from §10.

**Scale/Scope**: Default 100 seats (10×10), configurable; ~25 HTTP endpoints across 3 services; 5 schedulers on the
reservation server, 1 on the queue server.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is the unfilled template (no ratified principles), so there are no constitution
gates to evaluate. The binding project rules come from `CLAUDE.md` and are treated as gates instead:

| Gate (CLAUDE.md) | Status |
|---|---|
| Do not edit `specs/001-reservation-test/spec.md` | PASS — untouched |
| No `git commit` / `push` / history- or branch-changing commands | PASS — only read-only git used; no branch created (plan stays on `main`) |
| Record every undetermined decision in `docs/decisions.md`, append-only, at decision time | PASS — D-001…D-040 written during this planning step |
| Completion report only after all tasks | N/A at plan stage |
| Spec §10 "Do not implement" respected | PASS — design has no Redis, retries for PG, rate limiting, 429/503, auth, Docker images, metrics |

**Post-design re-check**: PASS — Phase 1 artifacts introduce no §10 items. The only schema addition beyond §5.1
(`sale_period` table, D-016) is required by §5.1's own sentence about storing the sale period, and it is excluded
from the snapshot's five tables.

## Project Structure

### Documentation (this feature)

```text
specs/001-reservation-test/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   ├── reservation-server.md
│   ├── queue-server.md
│   └── mock-pg.md
└── tasks.md             # Phase 2 output (/speckit-tasks — not created here)

docs/
├── decisions.md          # D-xxx decision log (CLAUDE.md)
└── completion-report.md  # written after all tasks (CLAUDE.md)
```

### Source Code (repository root)

```text
settings.gradle                 # rootProject.name = 'reservation-test'; include 'server', 'queue', 'mock-pg'
build.gradle                    # common: Java 21 toolchain, Boot BOM, Lombok, web/validation/actuator, JUnit
gradle/wrapper/ , gradlew       # Gradle wrapper
docker-compose.yml              # postgres:17, db/user/password = reservation

server/
├── build.gradle                # jdbc, flyway, postgresql, testcontainers
└── src/
    ├── main/java/com/example/reservation/
    │   ├── ReservationServerApplication.java
    │   ├── clock/              # ServerClock, ClockController (/admin/clock)
    │   ├── config/             # @ConfigurationProperties, RuntimeConfig (mutable via reset)
    │   ├── admission/          # AdmissionKeyVerifier, IndependentKeyVerifier, RevokedKids, AdmissionGuard
    │   ├── seat/               # SeatRepository, SeatQueryService, GET /seats
    │   ├── hold/               # HoldService + strategies (Conditional/Pessimistic/Optimistic/Naive), HoldController
    │   ├── payment/            # CheckoutService, ConfirmService, RecoveryScheduler, PgClient
    │   ├── deposit/            # DepositService, DepositDeadlineScheduler, ReopenScheduler
    │   ├── reservation/        # ReservationRepository, cancel + lookup
    │   ├── sale/               # SalePeriodRepository, SaleEndScheduler
    │   ├── expiry/             # HoldExpiryScheduler
    │   ├── notify/             # SlotNotifier (async + retry), SaleStatePublisher, DroppedNotifications
    │   ├── admin/              # reset, stats, snapshot, Counters, BackstopManager
    │   └── web/                # ApiException, ErrorCode, GlobalExceptionHandler, time JSON format
    ├── main/resources/
    │   ├── application.yml
    │   └── db/migration/V1__init.sql
    └── test/java/com/example/reservation/   # Testcontainers integration + unit tests, stub PG/queue

queue/
└── src/
    ├── main/java/com/example/queue/
    │   ├── clock/ config/ web/
    │   ├── key/                # AdmissionKeyIssuer (HMAC-SHA256)
    │   ├── core/               # QueueState (tokens, slots, user index, log), AdmissionScheduler
    │   └── api/                # QueueController, InternalController, AdminController
    └── test/java/com/example/queue/

mock-pg/
└── src/
    ├── main/java/com/example/mockpg/
    │   ├── clock/ config/ web/
    │   ├── core/               # PaymentStore, PgRandom (seeded), LatencyStats
    │   └── api/                # PgController, AdminController
    └── test/java/com/example/mockpg/
```

**Structure Decision**: Gradle multi-module as fixed by spec §2.1 (`server`, `queue`, `mock-pg`). Modules share no
code (spec §1): the clock, error format, and HMAC key code are duplicated per module on purpose. Packages are
feature-oriented inside each module.

## Phase summary

- **Phase 0** → [research.md](./research.md): technology versions, clock design, concurrency/locking design per
  strategy, notification design, mock-pg randomness, and every undetermined spec point (cross-referenced to
  `docs/decisions.md`).
- **Phase 1** → [data-model.md](./data-model.md) (tables, in-memory state, transitions, lock order),
  [contracts/](./contracts/) (HTTP contracts per service, incl. validation and error precedence),
  [quickstart.md](./quickstart.md) (build/run/validate scenarios).

## Complexity Tracking

No constitution violations to justify. Deliberate duplications (clock, error handling, HMAC code in each module) are
required by spec §1 ("services do not depend on each other's code").
