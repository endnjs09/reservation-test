---

description: "Task list for reservation-test (reservation server, queue server, mock-pg)"
---

# Tasks: reservation-test (reservation server, queue server, mock-pg)

**Input**: Design documents from `/specs/001-reservation-test/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md; decisions in `docs/decisions.md`

**Tests**: Included. The spec's tech stack mandates Testcontainers for reservation-server tests, and `CLAUDE.md`
requires a spec-section → test mapping in the completion report, so every story has test tasks.
Write each story's tests first and confirm they fail before implementing.

**User stories**: spec.md has no prioritized user stories. They are derived from the flow in spec §2.2/§4.3
(queue → hold → payment → confirmation, plus returns):

| Story | Priority | Spec sections |
|---|---|---|
| US1 Seat view & hold (hold, release, expiry, lookup, strategies, backstop) | P1 🎯 MVP | 4.1–4.5, 5.2, 6.4, 6.5, 6.9 (lookup), 6.12 |
| US2 Card payment (mock-pg, checkout, confirm, recovery) | P1 | 6.6, 8 |
| US3 Queue & admission (queue server, slot notifications) | P2 | 6.11, 7.1–7.5, 7.7, 7.8 |
| US4 Pending deposit, return batches, booking cancellation | P2 | 6.7, 6.8, 6.9 (cancel) |
| US5 Sale end (both servers) | P3 | 6.10, 7.6 (sale end) |
| Foundational (all stories) | — | 1, 2, 3, 5.1, 5.3, 6.1–6.3, 6.13, 6.14 |
| Polish (cross-cutting invariants) | — | 9 |

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: US1…US5
- Every task names exact file paths

## Path Conventions

- `server/src/main/java/com/example/reservation/` = **SRV**, `server/src/test/java/com/example/reservation/` = **SRVT**
- `queue/src/main/java/com/example/queue/` = **Q**, `queue/src/test/java/com/example/queue/` = **QT**
- `mock-pg/src/main/java/com/example/mockpg/` = **PG**, `mock-pg/src/test/java/com/example/mockpg/` = **PGT**
- Paths below are written out in full.

## Rules that apply to every task

- Read the referenced spec sections and D-xxx entries in `docs/decisions.md` before implementing.
- Never read the wall clock for behavior: use the module's `ServerClock.now()`; pass it to SQL as a parameter.
- If you must decide something the spec and `docs/decisions.md` do not settle, **append** a new D-xxx entry to
  `docs/decisions.md` (format in `CLAUDE.md`) at that moment. Never edit `specs/001-reservation-test/spec.md`.
- Do not run `git commit`/`push` or any history/branch-changing git command.
- Modules share no code (spec §1): duplicate small utilities per module.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Gradle multi-module skeleton, Docker Compose, empty bootable apps.

- [ ] T001 Create `settings.gradle` (`rootProject.name = 'reservation-test'`, `include 'server', 'queue', 'mock-pg'`) and root `build.gradle` applying to `subprojects`: `java` with toolchain 21, `org.springframework.boot` 4.1.2 (apply false at root, applied in modules), `io.spring.dependency-management`, `mavenCentral()`, dependencies `spring-boot-starter-web`, `spring-boot-starter-validation`, `spring-boot-starter-actuator`, Lombok (`compileOnly`/`annotationProcessor`/`testCompileOnly`/`testAnnotationProcessor`), `spring-boot-starter-test`, `junit-platform-launcher` (testRuntimeOnly), `tasks.withType(Test) { useJUnitPlatform() }`
- [ ] T002 Generate the Gradle wrapper (Gradle 9.x, ≥ 8.14 required by Boot 4) producing `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.properties`, `gradle/wrapper/gradle-wrapper.jar`; note the exact Gradle version for the completion report (section 2)
- [ ] T003 [P] Create `server/build.gradle`: apply Boot plugin; add `spring-boot-starter-jdbc`, `spring-boot-starter-flyway`, `org.flywaydb:flyway-database-postgresql`, `org.postgresql:postgresql` (runtimeOnly); tests: `spring-boot-testcontainers`, `org.testcontainers:testcontainers-junit-jupiter`, `org.testcontainers:testcontainers-postgresql`. No JPA, no Security, no `spring-boot-docker-compose`
- [ ] T004 [P] Create `queue/build.gradle` (Boot plugin only; common deps come from root)
- [ ] T005 [P] Create `mock-pg/build.gradle` (Boot plugin only)
- [ ] T006 [P] Create `docker-compose.yml` with service `postgres` image `postgres:17`, env `POSTGRES_DB=reservation`, `POSTGRES_USER=reservation`, `POSTGRES_PASSWORD=reservation`, port `5432:5432`, healthcheck `pg_isready -U reservation -d reservation`
- [ ] T007 [P] Create `server/src/main/java/com/example/reservation/ReservationServerApplication.java` (`@SpringBootApplication`, `@EnableScheduling`, `@ConfigurationPropertiesScan`) and `server/src/main/resources/application.yml` with all keys of spec §6.1 plus: `spring.datasource.url: jdbc:postgresql://localhost:5432/reservation`, user/password `reservation`, `spring.datasource.hikari.maximum-pool-size: 20` (§5.3), `spring.flyway.enabled: true`, `admission.run-epoch: initial` (D-003), `queue-server.connect-timeout-ms: 1000`, `queue-server.request-timeout-ms: 2000` (D-023), `management.endpoints.web.exposure.include: health`
- [ ] T008 [P] Create `queue/src/main/java/com/example/queue/QueueServerApplication.java` (`@EnableScheduling`) and `queue/src/main/resources/application.yml` with spec §7.8 keys plus `admission.run-epoch: initial` (D-003)
- [ ] T009 [P] Create `mock-pg/src/main/java/com/example/mockpg/MockPgApplication.java` and `mock-pg/src/main/resources/application.yml` with `server.port: 8081`, `spring.threads.virtual.enabled: true`, `mockpg: { auth-failure-rate: 0.03, decline-rate: 0.03, timeout-rate: 0, confirm-min-ms: 100, confirm-max-ms: 500, seed: null }` (spec §8 defaults)
- [ ] T010 Verify `./gradlew build -x test` succeeds and each app starts (`bootRun`) with `/actuator/health` = UP (server after `docker compose up -d`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Clock, error format, JSON time format in all modules; reservation-server schema, runtime config,
admin reset/stats/snapshot, counters, backstop, admission-key verification, notifier, test harness.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Clock, errors, JSON (one copy per module)

- [ ] T011 [P] Implement `server/src/main/java/com/example/reservation/clock/ServerClock.java`: immutable state `{baseInstant, baseNanos, offsetNanos}` in a volatile field; `now()` = base + (`System.nanoTime()` − baseNanos) + offset, **truncated to milliseconds** (D-001); `reset(Instant anchorAtOrNull)` (null → `Instant.now()` system time, monotonic restarts, offset 0; spec §3.1); `advance(BigDecimal seconds)` adds `seconds × 1e9` nanos rounded half-up (D-002); constructor = startup base
- [ ] T012 [P] Implement `server/src/main/java/com/example/reservation/clock/ClockController.java`: `GET /admin/clock` → `{"now"}`; `POST /admin/clock/advance` `{"seconds"}` — must be a JSON number > 0, else 400 `VALIDATION_FAILED` (spec §3.2, D-002)
- [ ] T013 [P] Implement `server/src/main/java/com/example/reservation/web/ApiException.java` (code, HTTP status, message, optional extra fields map) and `server/src/main/java/com/example/reservation/web/ErrorCode.java` enum with every row of spec §6.2 (code → HTTP status) plus `NOT_FOUND` 404, `METHOD_NOT_ALLOWED` 405, `INTERNAL_ERROR` 500 (D-004)
- [ ] T014 [P] Implement `server/src/main/java/com/example/reservation/web/GlobalExceptionHandler.java`: `ApiException` → `{code, message, ...extra}`; malformed JSON / type mismatch / `MethodArgumentNotValidException` / missing header / non-numeric path var → 400 `VALIDATION_FAILED`; `NoResourceFoundException` → 404 `NOT_FOUND`; method not supported → 405; other → 500 `INTERNAL_ERROR` (D-004)
- [ ] T015 [P] Implement `server/src/main/java/com/example/reservation/web/JsonConfig.java`: Jackson 3 customizer serializing `Instant` as `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` UTC and parsing any ISO-8601 instant with offset (D-001); ignore unknown properties
- [ ] T016 [P] Duplicate clock/errors/JSON for the queue: `queue/src/main/java/com/example/queue/clock/ServerClock.java`, `queue/src/main/java/com/example/queue/clock/ClockController.java`, `queue/src/main/java/com/example/queue/web/ApiException.java`, `queue/src/main/java/com/example/queue/web/GlobalExceptionHandler.java`, `queue/src/main/java/com/example/queue/web/JsonConfig.java` (same rules; 400 code `VALIDATION_FAILED`)
- [ ] T017 [P] Duplicate clock/errors/JSON for mock-pg: `mock-pg/src/main/java/com/example/mockpg/clock/ServerClock.java`, `mock-pg/src/main/java/com/example/mockpg/clock/ClockController.java`, `mock-pg/src/main/java/com/example/mockpg/web/ApiException.java`, `mock-pg/src/main/java/com/example/mockpg/web/GlobalExceptionHandler.java`, `mock-pg/src/main/java/com/example/mockpg/web/JsonConfig.java` (400 code `INVALID_REQUEST`, D-004)
- [ ] T018 [P] Unit test `server/src/test/java/com/example/reservation/clock/ServerClockTest.java`: anchor reset sets now ≈ anchor; advance(30) adds 30 s; advance(0.5) adds 500 ms; ms truncation; monotonic progression (spec §3.1, §3.2)
- [ ] T019 [P] Unit tests `queue/src/test/java/com/example/queue/clock/ServerClockTest.java` and `mock-pg/src/test/java/com/example/mockpg/clock/ServerClockTest.java` (same cases as T018)
- [ ] T020 [P] Web tests for clock API in all modules: `server/src/test/java/com/example/reservation/clock/ClockControllerTest.java` (needs T031 harness; may be written after T031), `queue/src/test/java/com/example/queue/clock/ClockControllerTest.java`, `mock-pg/src/test/java/com/example/mockpg/clock/ClockControllerTest.java`: advance 30 → 200 `{now}` moved; `seconds` 0, −1, missing, `"abc"` → 400

### Reservation server: schema and configuration

- [ ] T021 Create `server/src/main/resources/db/migration/V1__init.sql` exactly per data-model.md §1: tables `seats`, `reservations`, `reservation_seats`, `payments`, `release_batches`, `sale_period` with the listed types, `CHECK` constraints (seat status ∈ `AVAILABLE, HELD, PENDING_DEPOSIT, RETURN_PENDING, SOLD`; reservation status ∈ the 9 states of spec §4.2; payment status ∈ `REQUESTED, APPROVED, FAILED, CANCELED`; `payment_method` ∈ `CARD, DEPOSIT` or NULL; grade ∈ `VIP,S,A,B`; `sale_period.id = 1`), `reservations.idempotency_key varchar(64) NOT NULL UNIQUE`, `payments.reservation_id UNIQUE`, `seats.version bigint NOT NULL DEFAULT 0`, `release_batches.released boolean NOT NULL DEFAULT false`, `seat_count int NOT NULL DEFAULT 0`, the lookup indexes, and `CREATE UNIQUE INDEX ux_one_open_batch ON release_batches((true)) WHERE NOT released`. Do NOT create the two backstop indexes here (D-015)
- [ ] T022 [P] Implement `server/src/main/java/com/example/reservation/config/ReservationProperties.java` (`@ConfigurationProperties("reservation")` for every §6.1 key, incl. nested `seats`, `grades`, `payment.confirm-deadline-sec`, `scheduler.*`, `pg.*`), `server/src/main/java/com/example/reservation/config/AdmissionProperties.java` (`required`, `key-secret`, `run-epoch`), `server/src/main/java/com/example/reservation/config/QueueServerProperties.java` (`base-url`, timeouts), `server/src/main/java/com/example/reservation/config/InternalProperties.java` (`secret`)
- [ ] T023 Implement `server/src/main/java/com/example/reservation/config/RuntimeConfig.java`: thread-safe holder of an immutable `ConfigValues` record `{rows, cols, grades (list of {name, rows, price}), strategy (enum CONDITIONAL|PESSIMISTIC|OPTIMISTIC|NAIVE), dbBackstop, holdTtlSec, confirmDeadlineSec, maxSeatsPerUser, depositDeadlineSec, returnDelaySec, saleDurationSec}` + `runEpoch`; initialized from properties at startup (D-018); `toConfigJson()` returns exactly those 11 fields (D-027)
- [ ] T024 [P] Implement `server/src/main/java/com/example/reservation/seat/SeatLayout.java`: seat id = `rowIndex × cols + colIndex + 1`, label = row letter (`A` + rowIndex) + (colIndex + 1), grade/price by rows front-to-back from `grades` (spec §4.1); and `server/src/main/java/com/example/reservation/seat/SeatRepository.java` with `insertAll(layout, now)` (batch insert, status AVAILABLE, version 0), `count()`, `findByIds`, `existingIds(ids)`
- [ ] T025 [P] Implement `server/src/main/java/com/example/reservation/sale/SalePeriodRepository.java` (`find()`, `upsert(startAt, endAt)` with `end_processed=false`, `markEndProcessed()`) and `server/src/main/java/com/example/reservation/sale/SalePeriodService.java` caching `saleEndAt`/`endProcessed`; `isEnded(now)` = now ≥ saleEndAt (D-026); `checkNotEnded()` throws 409 `SALE_ENDED`
- [ ] T026 [P] Implement `server/src/main/java/com/example/reservation/admin/Counters.java`: `AtomicLong` per spec §6.13 name (`holdAttempts, holdSuccess, holdConflicts, confirms, declines, notPayable, expiredByScheduler, recoveryAttempts, recovered, immediateReturns, userCancels, depositsRequested, depositsPaid, depositsExpired, reopenCount, reopenSeats, slotNotifyDropped, acceptedInvalidKeys`), `snapshot()` → ordered map, `resetAll()`
- [ ] T027 [P] Implement `server/src/main/java/com/example/reservation/admin/BackstopManager.java`: `apply(boolean)` → true: `CREATE UNIQUE INDEX IF NOT EXISTS ux_rs_active_seat ON reservation_seats(seat_id) WHERE released_at IS NULL` and `CREATE UNIQUE INDEX IF NOT EXISTS ux_res_user_active ON reservations(user_id) WHERE status IN ('HELD','CONFIRMING')`; false: `DROP INDEX IF EXISTS` both (spec §5.2, D-015)
- [ ] T028 Implement `server/src/main/java/com/example/reservation/admin/AdminGate.java` (`ReentrantReadWriteLock`; `runIfOpen(Runnable)` uses `readLock().tryLock()` and skips the run if unavailable; `exclusive(Runnable)` for reset) and `server/src/main/java/com/example/reservation/config/SchedulingConfig.java` (`ThreadPoolTaskScheduler` pool size 8) (research §5)
- [ ] T029 Implement `server/src/main/java/com/example/reservation/admin/StartupInitializer.java` (`ApplicationRunner`): apply backstop from config; insert seats only if `seats` is empty; if no `sale_period` row, store start = now, end = now + `saleDurationSec`; load SalePeriodService cache (spec §6.10, D-018)
- [ ] T030 Implement `server/src/main/java/com/example/reservation/admin/ResetService.java` and `POST /admin/reset` in `server/src/main/java/com/example/reservation/admin/AdminController.java`: parse body as `JsonNode` (all fields optional, missing keep current); validate per spec §6.14 + D-027 (grades rows sum = rows; names ∈ VIP/S/A/B, no duplicates, rows ≥ 1, price ≥ 0; unknown strategy; any time or seat value ≤ 0; `maxSeatsPerUser ≥ 1`; bad ISO instant) → 400 `VALIDATION_FAILED`; under `AdminGate.exclusive`: clock reset (anchorAt or system now), apply config, `runEpoch` = given or random UUID, `TRUNCATE reservation_seats, payments, reservations, release_batches, seats, sale_period RESTART IDENTITY` (D-017), insert seats, store sale period (start = now, end = `saleEndAt` or now + `saleDurationSec`), apply backstop, clear revoked kids, notification queue (bump generation), dropped notifications, counters → 200 `{"config": …}`
- [ ] T031 Implement `GET /admin/stats` in `server/src/main/java/com/example/reservation/admin/StatsService.java` (+ AdminController): shape of spec §6.14 with every seat/reservation/payment state key present even if 0, `byGrade` for every configured grade, `releaseBatches {open, nextReleaseAt, totalReleased}`, `counters` (all §6.13 names), `config` (D-027), `saleEndAt`, `serverTime`, `runEpoch`
- [ ] T032 Implement `GET /admin/snapshot` in `server/src/main/java/com/example/reservation/admin/SnapshotService.java` (+ AdminController): one `REPEATABLE READ` read-only transaction reading all rows of `seats, reservations, reservation_seats, payments, release_batches` ordered by PK, keys = snake_case column names, timestamps per D-001; then `counters` and `droppedNotifications` from memory (spec §6.14, D-039)

### Reservation server: admission keys, notifications, shared seat ops

- [ ] T033 [P] Implement `server/src/main/java/com/example/reservation/admission/AdmissionKeyVerifier.java`: parse `v1.<payloadB64>.<sigB64>` (base64url no padding), HMAC-SHA256(`key-secret`, `"v1." + payloadB64`), compare with `MessageDigest.isEqual`; payload record `{kid, uid, iat, exp, run}`; returns typed result `INVALID | REVOKED | EXPIRED | OK` checking in spec §6.3 table order: format+signature, `run` = current runEpoch, `uid` = body userId (skippable), kid not revoked, now < exp (D-026)
- [ ] T034 [P] Implement `server/src/main/java/com/example/reservation/admission/IndependentKeyVerifier.java`: separately written check (split via `indexOf`, `java.util.Base64.getUrlDecoder`, `JsonNode` field reads, its own `Mac` instance, XOR-accumulate constant-time compare) of signature, run, uid, and expiry-or-exception (research §7)
- [ ] T035 [P] Implement `server/src/main/java/com/example/reservation/admission/RevokedKids.java` (concurrent set; `revoke(kid)`, `isRevoked`, `clear`)
- [ ] T036 Implement `server/src/main/java/com/example/reservation/admission/AdmissionGuard.java` + `server/src/main/java/com/example/reservation/admission/AdmissionContext.java` (request-scoped: parsed key, uid, target reservation id or orderId, whether the exception was used): `check(request, userIdOrNull, ExceptionTarget)`; missing/blank header → 403 `KEY_INVALID` (D-006); expiry exception per D-007 (target reservation exists, `user_id` = uid, status HELD/CONFIRMING; `GET /seats` and `POST /holds` never get it); skip everything when `admission.required = false` (spec §6.3)
- [ ] T037 Implement `server/src/main/java/com/example/reservation/admission/ReverificationInterceptor.java` (`HandlerInterceptor.afterCompletion`, registered in `server/src/main/java/com/example/reservation/config/WebConfig.java`): when the request carried an `AdmissionContext` and the response status is 2xx, run `IndependentKeyVerifier`; on failure `counters.acceptedInvalidKeys += 1`; never alters the response (spec §6.3)
- [ ] T038 Implement `server/src/main/java/com/example/reservation/notify/SlotNotifier.java`: `enqueueAfterCommit(kid, type, at)` (no-op if kid null, spec §6.11); `ScheduledExecutorService` (4 threads) sends `POST {queue-server.base-url}/internal/slots/{kid}/events` `{type, at}` with header `X-Internal-Secret` via `RestClient` (connect 1000 ms / read 2000 ms); non-2xx or I/O error → retry after 1 s, 2 s, 4 s real time; then drop: `slotNotifyDropped += 1` and add `{kid, type, at}` to `server/src/main/java/com/example/reservation/notify/DroppedNotifications.java`; reset bumps a generation, stale events are discarded silently (D-023). Also `completed(kid, at)` helper = `RevokedKids.revoke(kid)` + enqueue COMPLETED (D-008)
- [ ] T039 Implement `server/src/main/java/com/example/reservation/reservation/ReservationRepository.java` (JdbcTemplate): `insertHeld(...)`, `lockById(id)` (`FOR UPDATE`), `findById`, `transition(id, from, to, now)` conditional update returning boolean, `setConfirming(...)`, `setPendingDeposit(...)`, `findActiveByUser(userId)` (HELD/CONFIRMING), `hasBooking(userId)` (CONFIRMED/PENDING_DEPOSIT), `findByIdempotencyKey`; and `server/src/main/java/com/example/reservation/reservation/ReservationSeatRepository.java` (`insertAll`, `activeSeatIds(reservationId)` ordered asc, `markReleased(reservationId, now)`, `activeAssignmentsFor(seatIds)`)
- [ ] T040 Implement `server/src/main/java/com/example/reservation/seat/SeatTransitions.java` (used inside caller's transaction), per data-model.md §3: `returnImmediately(reservationId, seatIds, now)` = `UPDATE seats SET status='AVAILABLE', current_reservation_id=NULL, release_batch_id=NULL, version=version+1, updated_at=:now WHERE id = ANY(:ids) AND current_reservation_id = :rid` + `markReleased` + `counters.immediateReturns += returned` ; `toSold`, `toPendingDeposit`, `toReturnPending(batchId)` with the same `current_reservation_id = :rid` condition and `version+1`; seat ids always passed in ascending order
- [ ] T041 Implement `server/src/main/java/com/example/reservation/reservation/ReservationEnder.java`: one place for "reservation ends from HELD/CONFIRMING" (RELEASED, EXPIRED, PAYMENT_FAILED): conditional transition + `returnImmediately` + register after-commit HOLD_CLEARED for its kid (D-023)

### Test harness

- [ ] T042 Implement `server/src/test/java/com/example/reservation/support/IntegrationTestBase.java`: `@SpringBootTest(webEnvironment = RANDOM_PORT)`, static Testcontainers `PostgreSQLContainer("postgres:17")` with `@ServiceConnection` shared by all test classes, fast scheduler intervals via `@DynamicPropertySource` (e.g. 100–200 ms), `RestClient`/`TestRestTemplate` helpers, `reset(Map body)` called in `@BeforeEach`, `snapshot()` / `stats()` helpers
- [ ] T043 [P] Implement `server/src/test/java/com/example/reservation/support/PgStub.java` and `server/src/test/java/com/example/reservation/support/QueueStub.java` using `com.sun.net.httpserver.HttpServer` on a free port: programmable per-path responses (status, body, delay), recorded requests; wired via `reservation.pg.base-url` / `queue-server.base-url` properties in `IntegrationTestBase`
- [ ] T044 [P] Implement `server/src/test/java/com/example/reservation/support/TestKeys.java`: builds admission keys (`v1.payload.sig`) with arbitrary kid/uid/iat/exp/run and secret, including deliberately bad signatures (test-only copy of the queue format, spec §7.3)
- [ ] T045 [P] Implement `server/src/test/java/com/example/reservation/support/InvariantChecker.java`: from a server snapshot assert I1, I2 (unless naive+backstop off), I4, I5, I10, I11 and from `counters` I16 (spec §9); reusable by all stories
- [ ] T046 Foundational tests in `server/src/test/java/com/example/reservation/admin/AdminApiTest.java`: reset defaults → 100 seats with ids/labels/grades/prices of spec §4.1 table; reset with `rows: 4, cols: 5, grades` summing to 4 → 20 seats; invalid resets (grade rows sum ≠ rows, strategy `foo`, `holdTtlSec: 0`, grade name `X`) → 400; stats contains every state key with 0 and all counters; snapshot has the five tables with snake_case columns; ids restart at 1 after reset (D-017); `dbBackstop` false/true toggles the two indexes (query `pg_indexes`)
- [ ] T047 Foundational tests in `server/src/test/java/com/example/reservation/admission/AdmissionKeyVerifierTest.java` (unit, both verifiers): valid key OK; tampered payload/signature/format → INVALID; other runEpoch → INVALID; uid mismatch → INVALID; revoked kid → REVOKED; now ≥ exp → EXPIRED; both verifiers agree on all cases
- [ ] T048 Foundational test `server/src/test/java/com/example/reservation/notify/SlotNotifierTest.java` (with QueueStub): event delivered with `X-Internal-Secret` and `at` = event time; stub returns 500 always → exactly 4 attempts (≈1 s, 2 s, 4 s gaps), then `slotNotifyDropped` = 1 and `droppedNotifications` has `{kid, type, at}`; reset clears dropped list and stops pending retries

**Checkpoint**: All three apps boot; server reset/stats/snapshot/clock work; harness ready.

---

## Phase 3: User Story 1 — Seat view & hold (Priority: P1) 🎯 MVP

**Goal**: An admitted user can view seats, hold 1..`maxSeatsPerUser` seats all-or-nothing under four strategies
with idempotency and the DB backstop, release a hold, have it expire, and look it up.

**Independent Test**: With `admission.required` true (keys from `TestKeys`) and false, reset → `GET /seats` →
`POST /holds` → `GET /reservations/{id}` → `POST /holds/{id}/release` / clock advance 421 s → seats AVAILABLE;
concurrency tests show no double booking except naive + backstop off.

### Tests for User Story 1 ⚠️ (write first, must fail)

- [ ] T049 [P] [US1] `server/src/test/java/com/example/reservation/seat/SeatQueryTest.java`: `GET /seats` shape of spec §6.4 with all 7 summary fields always present; counts after holds; `soldOut` true only when 0 AVAILABLE and 0 HELD; `releaseAt` null without open batch; missing key → 403 `KEY_INVALID`; expired key → 403 `KEY_EXPIRED` even with a HELD reservation
- [ ] T050 [P] [US1] `server/src/test/java/com/example/reservation/hold/HoldValidationTest.java`: missing/65-char `Idempotency-Key` → 400; `userId` empty / 65 chars → 400; both `seatIds` and `seatId` / neither → 400; duplicates, 0, negative, > `maxSeatsPerUser` → 400; unknown seat → 404 `SEAT_NOT_FOUND`; `seatId: 5` works as one seat; `holdAttempts` counts only requests passing validation (D-005, D-025)
- [ ] T051 [P] [US1] `server/src/test/java/com/example/reservation/hold/HoldFlowTest.java`: 201 body `{reservationId, seats asc with label/grade/price, totalPrice, status:"HELD", holdExpiresAt = now + holdTtlSec}`; seats HELD with `current_reservation_id`; `reservations.seat_id` = smallest seat; `admission_kid` stored; all-or-nothing: second user `[38,39]` after first holds `[37,38]` → 409 with `unavailableSeatIds:[38]`, seat 39 still AVAILABLE, `holdConflicts` +1 (D-013); same user second hold → 409 `USER_ALREADY_HOLDING`
- [ ] T052 [P] [US1] `server/src/test/java/com/example/reservation/hold/IdempotencyTest.java`: same key + same content (seat order swapped) → identical 201 body, one reservation (I7, D-009); same key + different seats or user → 422 `IDEMPOTENCY_KEY_REUSED`; 20 concurrent requests with the same key → exactly one reservation, all 201 bodies identical (D-010)
- [ ] T053 [P] [US1] `server/src/test/java/com/example/reservation/hold/StrategyConcurrencyTest.java`: parameterized over 4 strategies × backstop {true,false}: 50 users × overlapping 2-seat requests on 10 seats in parallel; for all combos except naive+false assert `InvariantChecker` I1/I2/I4/I5/I10 and Σ acquired seats ≤ 10 with no seat in two active rows; naive+backstop true → violations surface as 409 `SEAT_UNAVAILABLE`/`USER_ALREADY_HOLDING`, never as duplicates; same-user parallel holds → at most one HELD (I4) for non-naive strategies
- [ ] T054 [P] [US1] `server/src/test/java/com/example/reservation/hold/ReleaseTest.java`: release HELD → 200 RELEASED, seats AVAILABLE, `released_at` set, `immediateReturns` += seat count; release again → 200 RELEASED unchanged; other user → 409 `RESERVATION_NOT_PAYABLE` and `notPayable` +1; unknown → 404
- [ ] T055 [P] [US1] `server/src/test/java/com/example/reservation/expiry/HoldExpirySchedulerTest.java`: advance clock `holdTtlSec + 1` → within a few scheduler ticks EXPIRED, seats AVAILABLE, `expiredByScheduler` +1; with `expiry-batch-size: 2` and 5 expired holds, all 5 are expired in a single run (repeat-while-full rule); expiry order is earliest `hold_expires_at` first; CONFIRMING reservations untouched (insert via repository); a reservation locked by another tx is skipped and handled next run
- [ ] T056 [P] [US1] `server/src/test/java/com/example/reservation/reservation/ReservationLookupTest.java`: `GET /reservations/{id}` fields per spec §6.9 incl. representative seat `seatId/label/grade`, `seats`, `paymentMethod` null, `payment` null; unknown → 404; non-numeric id → 400
- [ ] T057 [P] [US1] `server/src/test/java/com/example/reservation/admission/HoldAdmissionTest.java`: `POST /holds` with key for other uid → 403 `KEY_INVALID`; revoked kid → `KEY_REVOKED`; expired key → `KEY_EXPIRED` (no exception for new holds); expired key + HELD reservation → release allowed (expiry exception); `admission.required=false` skips checks and `admission_kid` is null; after normal successful requests `acceptedInvalidKeys` = 0 (I16)

### Implementation for User Story 1

- [ ] T058 [P] [US1] Implement `server/src/main/java/com/example/reservation/seat/SeatQueryService.java` and `server/src/main/java/com/example/reservation/seat/SeatController.java` (`GET /seats`): AdmissionGuard (no uid, no exception) → sale end check → layout + seats list ordered by id + summary counts + `soldOut` + `releaseAt` (open batch) + `saleEndAt` (spec §6.4)
- [ ] T059 [P] [US1] Implement request/response DTOs in `server/src/main/java/com/example/reservation/hold/dto/HoldRequest.java` (`userId` 1–64, `seatIds` or `seatId`, validation of "both or neither", positive, no duplicates, size ≤ current `maxSeatsPerUser` → 400) and `server/src/main/java/com/example/reservation/hold/dto/HoldResponse.java` (`reservationId, seats[{id,label,grade,price}], totalPrice, status, holdExpiresAt`) and `server/src/main/java/com/example/reservation/hold/dto/UserIdRequest.java` (`userId` 1–64)
- [ ] T060 [P] [US1] Implement `server/src/main/java/com/example/reservation/hold/SeatAcquirer.java` interface (`AcquireResult acquire(List<Long> sortedIds, long reservationId, Instant now)` → acquired ids / unavailable ids) and `server/src/main/java/com/example/reservation/hold/ConditionalStrategy.java` (single statement per D-011)
- [ ] T061 [P] [US1] Implement `server/src/main/java/com/example/reservation/hold/PessimisticStrategy.java` (`SELECT id, status FROM seats WHERE id = ANY(?) ORDER BY id FOR UPDATE`; update all only if every one AVAILABLE; else report non-AVAILABLE ids, D-012)
- [ ] T062 [P] [US1] Implement `server/src/main/java/com/example/reservation/hold/OptimisticStrategy.java` (unlocked read of status+version; per seat ascending `UPDATE … SET status='HELD', current_reservation_id=?, version=version+1 WHERE id=? AND version=? AND status='AVAILABLE'`; collect all failures; no retry, D-012)
- [ ] T063 [P] [US1] Implement `server/src/main/java/com/example/reservation/hold/NaiveStrategy.java` (unlocked `SELECT status`; if all AVAILABLE, `UPDATE seats SET status='HELD', current_reservation_id=?, version=version+1 WHERE id=?` with no condition; spec §6.5 "for comparison")
- [ ] T064 [US1] Implement `server/src/main/java/com/example/reservation/hold/HoldService.java` following D-005 inside one `TransactionTemplate` transaction: advisory lock `pg_advisory_xact_lock(1, hashtext(key))`; idempotency lookup → rebuild replay body (D-009) or 422; seat existence → 404; unless naive `pg_advisory_xact_lock(2, hashtext(userId))`; `hasBooking` → 409 `USER_ALREADY_PURCHASED`; active HELD/CONFIRMING → 409 `USER_ALREADY_HOLDING`; insert reservation (HELD, `hold_expires_at = now + holdTtlSec`, `admission_kid`, `seat_id` = min, `seat_count`); acquire via strategy from RuntimeConfig; on failure rollback + `holdConflicts += unavailable.size()` + 409 `SEAT_UNAVAILABLE {unavailableSeatIds}`; sale end re-check → rollback + 409 `SALE_ENDED`; insert `reservation_seats`; ensure `current_reservation_id` set; commit; `holdSuccess +1`; after commit HOLD_ACTIVE. Translate unique violations: `ux_rs_active_seat` → 409 `SEAT_UNAVAILABLE` (D-014 list), `ux_res_user_active` → 409 `USER_ALREADY_HOLDING`, `idempotency_key` unique → re-run lookup in a new tx (D-010)
- [ ] T065 [US1] Implement `server/src/main/java/com/example/reservation/hold/HoldController.java` `POST /holds` (validation → `holdAttempts +1` → AdmissionGuard with body userId → `SalePeriodService.checkNotEnded()` → HoldService → 201)
- [ ] T066 [US1] Implement `POST /holds/{id}/release` in `server/src/main/java/com/example/reservation/hold/ReleaseService.java` + HoldController: AdmissionGuard with expiry exception target `{id}`; lock reservation; unknown → 404; other user → 409 `RESERVATION_NOT_PAYABLE` (`notPayable +1`); HELD → `ReservationEnder` RELEASED → 200 `{reservationId, status:"RELEASED"}`; otherwise 200 with current status (spec §6.5)
- [ ] T067 [US1] Implement `server/src/main/java/com/example/reservation/expiry/HoldExpiryScheduler.java` (`fixedDelay = expiry-interval-ms`, wrapped in `AdminGate.runIfOpen`): loop { tx: `SELECT id FROM reservations WHERE status='HELD' AND hold_expires_at <= :now ORDER BY hold_expires_at LIMIT :batch FOR UPDATE SKIP LOCKED`; each → `ReservationEnder` EXPIRED; `expiredByScheduler += n` } while n == batch (spec §6.12)
- [ ] T068 [US1] Implement `server/src/main/java/com/example/reservation/reservation/ReservationQueryService.java` + `server/src/main/java/com/example/reservation/reservation/ReservationController.java` `GET /reservations/{id}` (no admission check; payment `{orderId, status}` or null; spec §6.9)
- [ ] T069 [US1] Run `./gradlew :server:test` for the US1 test classes; all pass

**Checkpoint**: MVP — holds work end to end with all strategies and invariants.

---

## Phase 4: User Story 2 — Card payment (Priority: P1)

**Goal**: mock-pg works per spec §8; the reservation server performs checkout → confirm with the single query +
reversal fallback and a recovery scheduler, so I3/I9/I12 hold.

**Independent Test**: mock-pg tests alone; server tests with `PgStub` covering DONE / decline / timeout→query
branches / recovery; an end-to-end test against a real mock-pg app is optional (Polish).

### Tests for User Story 2 ⚠️

- [ ] T070 [P] [US2] `mock-pg/src/test/java/com/example/mockpg/api/PgApiTest.java` (`@SpringBootTest` RANDOM_PORT): `/auth` success → AUTHORIZED payment `pk_…`; `authFailureRate 1` → 400 `AUTH_FAILED`, nothing stored; confirm DONE (200 `{status, approvedAt}`); `declineRate 1` → 400 `INSUFFICIENT_FUNDS`, DECLINED; mismatched order/amount, unknown key, second confirm → 400 `INVALID_REQUEST`; GET status / 404; cancel DONE → CANCELED, cancel again → 200 CANCELED, cancel DECLINED → 200 DECLINED, unknown → 404; `/admin/payments` lists all
- [ ] T071 [P] [US2] `mock-pg/src/test/java/com/example/mockpg/api/PgAdminTest.java`: config GET/PUT partial merge, explicit `seed:null`; invalid rates (−0.1, 1.1) or `confirmMinMs > confirmMaxMs` → 400 `INVALID_REQUEST`; same seed + same request order → same outcomes across two resets (D-035, D-036); stats counters and `confirmLatency {p50,p95,p99}` (D-037)
- [ ] T072 [P] [US2] `mock-pg/src/test/java/com/example/mockpg/api/PgTimeoutTest.java` (config `confirmMinMs 0, confirmMaxMs 50, timeoutRate 1`): client with 500 ms read timeout times out; payment status afterwards is DONE or AUTHORIZED; `timedOut` +1; held request finishes with 504 after `confirmMaxMs + 10000` ms (may use a shortened test-only property only if it does not change production behavior — otherwise assert only the client-side timeout)
- [ ] T073 [P] [US2] `server/src/test/java/com/example/reservation/payment/CheckoutTest.java`: checkout HELD → 200 `{orderId (UUID), amount = Σ prices}`, payment REQUESTED; repeat → same orderId; other user / expired (advance clock past hold) / not HELD → 409 `RESERVATION_NOT_PAYABLE` and `notPayable` +1; no PG call (PgStub records none)
- [ ] T074 [P] [US2] `server/src/test/java/com/example/reservation/payment/ConfirmTest.java` (PgStub): DONE → 200 CONFIRMED, payment APPROVED, seats SOLD, `confirms` +1, COMPLETED sent to QueueStub, kid revoked (next hold with same key → 403 `KEY_REVOKED`); PG 400 → 402 `PAYMENT_DECLINED`, payment FAILED with `fail_reason`, PAYMENT_FAILED, seats AVAILABLE, HOLD_CLEARED; checks before approval (other user, expired hold, amount mismatch, payment not REQUESTED, unknown/non-UUID orderId) → 409 and PgStub received **zero** confirm calls (I9, D-020)
- [ ] T075 [P] [US2] `server/src/test/java/com/example/reservation/payment/ConfirmUnknownResultTest.java` (PgStub delays confirm > request timeout or returns 500): query DONE → 200; CANCELED → 402 + payment CANCELED; 404 → 402 + payment FAILED; AUTHORIZED/DECLINED + cancel 200 → 402 + payment CANCELED; AUTHORIZED + cancel fails → 202 `{status:"CONFIRMING"}` and reservation stays CONFIRMING with `confirm_deadline = now + confirmDeadlineSec`; query fails → 202
- [ ] T076 [P] [US2] `server/src/test/java/com/example/reservation/payment/RecoverySchedulerTest.java`: CONFIRMING reservation before deadline untouched; after advancing clock past `confirm_deadline`: query DONE → CONFIRMED + COMPLETED + `recovered` +1; CANCELED/404 → PAYMENT_FAILED + HOLD_CLEARED; AUTHORIZED + cancel ok → PAYMENT_FAILED; query failing → stays CONFIRMING, `recoveryAttempts` increments each run; expiry scheduler never touches CONFIRMING (spec §4.3); concurrent resolution only applied once (D-038)
- [ ] T077 [P] [US2] `server/src/test/java/com/example/reservation/admission/PaymentAdmissionTest.java`: expired key + HELD reservation → checkout and confirm allowed (exception via path id / orderId, D-007); expired key + reservation of another user or unknown order → 403 `KEY_EXPIRED`; `acceptedInvalidKeys` stays 0

### Implementation for User Story 2 — mock-pg

- [ ] T078 [P] [US2] Implement `mock-pg/src/main/java/com/example/mockpg/core/PgConfig.java` (record `{authFailureRate, declineRate, timeoutRate, confirmMinMs, confirmMaxMs, seed}`, validation: probabilities in 0–1, `0 ≤ confirmMinMs ≤ confirmMaxMs`) and `mock-pg/src/main/java/com/example/mockpg/core/PgRandom.java` (synchronized `java.util.Random`, seeded when seed non-null; `reseed(config)`; `nextDouble()`, `nextLongBetween(min,max)`)
- [ ] T079 [P] [US2] Implement `mock-pg/src/main/java/com/example/mockpg/core/Payment.java` (`paymentKey, orderId, amount, status AUTHORIZED|DONE|DECLINED|CANCELED, processing, approvedAt`) and `mock-pg/src/main/java/com/example/mockpg/core/PaymentStore.java` (`ConcurrentHashMap`; `create`, `get`, `tryStartConfirm(key, orderId, amount)` atomic, `markDone`, `markDeclined`, `cancel` idempotent per spec §8, `all()`, `clear()`)
- [ ] T080 [P] [US2] Implement `mock-pg/src/main/java/com/example/mockpg/core/PgStats.java` (counters `authRequests, authFailed, confirmRequests, done, declined, timedOut, canceled`, `confirmInflight`, latency samples with nearest-rank p50/p95/p99, 0 when empty — D-037)
- [ ] T081 [US2] Implement `mock-pg/src/main/java/com/example/mockpg/api/PgController.java`: `POST /auth` (validation D-034; 1 draw; fail → 400 `AUTH_FAILED`; else 200 `{paymentKey}`), `POST /payments/confirm` (sequence of D-035: 3 draws → validate/`tryStartConfirm` → sleep delay → timeout: 4th draw 50% DONE stored, hold until arrival + `confirmMaxMs + 10000` ms then 504 `TIMEOUT` → decline: DECLINED 400 `INSUFFICIENT_FUNDS` → DONE 200 `{status:"DONE", approvedAt}`), `GET /payments/{key}`, `POST /payments/{key}/cancel` (404 `NOT_FOUND` for unknown)
- [ ] T082 [US2] Implement `mock-pg/src/main/java/com/example/mockpg/api/AdminController.java`: `GET/PUT /admin/config` (JsonNode partial merge, D-036), `POST /admin/reset` (`anchorAt` clock reset, clear payments + stats, reseed; keep config), `GET /admin/stats` (with `serverTime`), `GET /admin/payments`

### Implementation for User Story 2 — reservation server

- [ ] T083 [P] [US2] Implement `server/src/main/java/com/example/reservation/payment/PaymentRepository.java` (`insertRequested(uuid, reservationId, amount, now)`, `findByReservation`, `lockById(orderId)`, `setStatus(id, from, to, failReason, now)` conditional, `setPaymentKey`)
- [ ] T084 [P] [US2] Implement `server/src/main/java/com/example/reservation/payment/PgClient.java`: `RestClient` on `JdkClientHttpRequestFactory` (connect `pg.connect-timeout-ms`, read `pg.request-timeout-ms`); `confirm(key, orderId, amount)` → `Done | Declined(code) | Unknown`; `query(key)` → `Status(s) | NotFound | Failed`; `cancel(key)` → `Status(s) | NotFound | Failed`; no retries (spec §10)
- [ ] T085 [US2] Implement `server/src/main/java/com/example/reservation/payment/CheckoutService.java` + `POST /holds/{id}/checkout` in `server/src/main/java/com/example/reservation/payment/PaymentController.java`: AdmissionGuard (exception target `{id}`); lock reservation; unknown → 404; user mismatch / not HELD / now ≥ `hold_expires_at` → 409 `RESERVATION_NOT_PAYABLE` (+`notPayable`); existing payment → return; else insert REQUESTED with amount = Σ seat prices → 200 `{orderId, amount}`
- [ ] T086 [US2] Implement `server/src/main/java/com/example/reservation/payment/PaymentOutcomeApplier.java` (Transaction B logic shared with recovery, D-038): `applySuccess(resId)` (still CONFIRMING → payment APPROVED, reservation CONFIRMED, `toSold`, `confirms +1`, after commit `SlotNotifier.completed`), `applyFailure(resId, paymentStatus FAILED|CANCELED, reason)` (→ PAYMENT_FAILED via `ReservationEnder`, seats returned, HOLD_CLEARED); returns the resulting reservation status
- [ ] T087 [US2] Implement `server/src/main/java/com/example/reservation/payment/ConfirmService.java` + `POST /payments/confirm` in PaymentController: body validation (D-025); AdmissionGuard (exception target = orderId's reservation); Transaction A per spec §6.6 ③.1 (non-UUID/unknown orderId → 409, D-020); PG confirm outside tx; DONE → success 200; 4xx → failure FAILED, `declines +1`, 402 `PAYMENT_DECLINED` (D-021); unknown → one query → DONE 200 / CANCELED 402 / 404 402 (FAILED) / AUTHORIZED|DECLINED → cancel → ok 402 (CANCELED) / fail 202 `{status:"CONFIRMING"}` / query failed 202; when the applier finds the reservation already resolved, answer from current state (D-038)
- [ ] T088 [US2] Implement `server/src/main/java/com/example/reservation/payment/RecoveryScheduler.java` (`fixedDelay = recovery-interval-ms`, `AdminGate.runIfOpen`): select CONFIRMING with `confirm_deadline <= :now` ordered by deadline; per reservation `recoveryAttempts +1`, PG query, then table of spec §6.6 (DONE success; CANCELED → CANCELED; 404 → FAILED; AUTHORIZED/DECLINED → cancel → on success CANCELED; query/cancel failure → keep); `recovered +1` when resolved
- [ ] T089 [US2] Run `./gradlew :mock-pg:test :server:test` for US2 classes; all pass

**Checkpoint**: Card payments correct with all PG failure modes.

---

## Phase 5: User Story 3 — Queue & admission (Priority: P2)

**Goal**: Queue server admits users in seq order within capacity and issues admission keys accepted by the
reservation server; slot events and sale-state flow from the reservation server.

**Independent Test**: Queue module tests alone (admission rules, busy cap, sold-out, internal API); a cross-module
test vector proves a queue-issued key verifies on the reservation server.

### Tests for User Story 3 ⚠️

- [ ] T090 [P] [US3] `queue/src/test/java/com/example/queue/core/QueueStateTest.java` (unit, drives `runAdmission()` directly with a controllable clock): enter returns same token while active; seq ascending; position of new token (D-028); admission count = `min(maxActive − slots, floor(admitPerSec × intervalMs/1000))` in seq order (I8); log only runs with admitted > 0 and `activeMax`/`admittedPerTickMax`; non-busy slot expires at `expiresAt`; busy slot survives past `expiresAt` and is force-expired at `expiresAt + busyMaxExtraSec` with `busyCapExpirations` entry and `expiredByBusyCap +1` (D-033); re-entry after EXPIRED/LEFT/COMPLETED gets a new token at the end; slots never exceed `maxActive` (I17)
- [ ] T091 [P] [US3] `queue/src/test/java/com/example/queue/core/SlotEventTest.java`: HOLD_ACTIVE sets busy/busySince; HOLD_CLEARED clears busy and slot past expiry expires on next run; COMPLETED → token COMPLETED and slot released; older `at` ignored; events after COMPLETED ignored; unknown kid → ignored; `slotEvents.*` and `ignored` counters (D-033)
- [ ] T092 [P] [US3] `queue/src/test/java/com/example/queue/core/SoldOutTest.java`: soldOut true + `closeQueueOnSoldOut` → WAITING → CLOSED(SOLD_OUT), ADMITTED untouched, new enters CLOSED(SOLD_OUT), `closedSoldOut` counter; soldOut false → entries accepted again; `closeQueueOnSoldOut=false` → nobody closed (spec §7.6)
- [ ] T093 [P] [US3] `queue/src/test/java/com/example/queue/api/QueueApiTest.java` (`@SpringBootTest`): enter/status/leave shapes per contracts/queue-server.md (D-031); `pollAfterMs` thresholds 50/300/1000; status response has no seat fields; unknown/malformed token → 400 `INVALID_TOKEN`; missing token → 400; leave WAITING/ADMITTED → LEFT; internal API without/with wrong `X-Internal-Secret` → 401; reset validation (D-030); stats keys and `slotsBusyOverTtl`; snapshot keys
- [ ] T094 [P] [US3] `queue/src/test/java/com/example/queue/key/AdmissionKeyIssuerTest.java`: key format `v1.<b64url>.<b64url>` without padding; payload fields `kid, uid, iat, exp, run` with `exp` = slot `expiresAt` ms; signature = HMAC-SHA256(secret, `"v1." + payload`); emits a fixed test vector (fixed kid/uid/iat/exp/run/secret) whose expected string is also asserted in T095
- [ ] T095 [P] [US3] `server/src/test/java/com/example/reservation/admission/QueueKeyCompatibilityTest.java`: the same fixed vector string from T094 verifies OK on `AdmissionKeyVerifier` and `IndependentKeyVerifier` (keeps modules code-independent, spec §1)
- [ ] T096 [P] [US3] `server/src/test/java/com/example/reservation/notify/SlotEventsIntegrationTest.java` (QueueStub): hold → HOLD_ACTIVE; release / expiry / payment failure → HOLD_CLEARED; card CONFIRMED / deposit chosen → COMPLETED; `at` equals the event time; no events when `admission.required=false`; notification failure does not change the HTTP result
- [ ] T097 [P] [US3] `server/src/test/java/com/example/reservation/notify/SaleStatePublisherTest.java` (QueueStub): sends `{soldOut, releaseAt}` with secret at least every 5 s and within ~1 s after soldOut flips (D-024)

### Implementation for User Story 3

- [ ] T098 [P] [US3] Implement `queue/src/main/java/com/example/queue/config/QueueProperties.java` (`max-active` 200, `admit-per-sec` 20, `admission-ttl-sec` 420, `busy-max-extra-sec` 510, `close-queue-on-sold-out` true, `admission-interval-ms` 1000), `queue/src/main/java/com/example/queue/config/AdmissionProperties.java` (`key-secret`, `run-epoch`), `queue/src/main/java/com/example/queue/config/InternalProperties.java`, and `queue/src/main/java/com/example/queue/config/QueueRuntimeConfig.java` (reset-changeable fields + `saleEndAt` null at start, D-032)
- [ ] T099 [P] [US3] Implement `queue/src/main/java/com/example/queue/key/AdmissionKeyIssuer.java` (spec §7.3, research §7)
- [ ] T100 [P] [US3] Implement models `queue/src/main/java/com/example/queue/core/Token.java` (`tokenId, userId, seq, status WAITING|ADMITTED|COMPLETED|LEFT|EXPIRED|CLOSED, createdAt, admittedAt, endedAt, endReason COMPLETED|LEFT|EXPIRED|BUSY_CAP|SALE_ENDED|SOLD_OUT, position`), `queue/src/main/java/com/example/queue/core/Slot.java` (`keyId, userId, tokenId, admittedAt, expiresAt, busy, busySince, lastEventAt, admissionKey`), `queue/src/main/java/com/example/queue/core/AdmissionLogEntry.java` (`tickAt, admitted, seqFrom, seqTo, activeAfter`), `queue/src/main/java/com/example/queue/core/QueueCounters.java`
- [ ] T101 [US3] Implement `queue/src/main/java/com/example/queue/core/QueueState.java` guarded by one `ReentrantLock` (research §9): `enter(userId)`, `status(tokenId)`, `leave(tokenId)`, `runAdmission(now)` (D-029 step order; sale-end step delegates to a method finished in US5 — implement it now as: if `saleEndAt != null && now ≥ saleEndAt` close all WAITING/ADMITTED with SALE_ENDED), `applySlotEvent(kid, type, at)` (D-033), `applySaleState(soldOut, releaseAt)` (spec §7.6 sold-out), `reset(config)`, `stats(now)`, `snapshot(now)`
- [ ] T102 [US3] Implement `queue/src/main/java/com/example/queue/core/AdmissionScheduler.java` (`@Scheduled(fixedDelayString = "${queue.admission-interval-ms}")` → `QueueState.runAdmission(clock.now())`)
- [ ] T103 [US3] Implement `queue/src/main/java/com/example/queue/api/QueueController.java` (`/queue/enter`, `/queue/status`, `/queue/leave`; D-030, D-031), `queue/src/main/java/com/example/queue/api/InternalController.java` (secret check → 401 `UNAUTHORIZED`; `/internal/slots/{kid}/events` → `{ignored}`; `/internal/sale-state`), `queue/src/main/java/com/example/queue/api/AdminController.java` (`/admin/reset` with anchorAt + D-030 validation + random runEpoch if missing, `/admin/stats`, `/admin/snapshot`)
- [ ] T104 [US3] Implement `server/src/main/java/com/example/reservation/notify/SaleStatePublisher.java` (`fixedDelay 1000`, `AdminGate.runIfOpen`; compute soldOut/releaseAt; send `POST /internal/sale-state` when changed or ≥ 5 s since last send; no retry, log failures; D-024)
- [ ] T105 [US3] Verify every reservation-server path from spec §6.11's table enqueues the right event (HoldService, ReservationEnder, PaymentOutcomeApplier, DepositService once US4 exists, sale end once US5 exists); fix gaps
- [ ] T106 [US3] Run `./gradlew :queue:test :server:test`; all pass

**Checkpoint**: Full queue → key → hold → pay flow works across real services.

---

## Phase 6: User Story 4 — Pending deposit, return batches, booking cancellation (Priority: P2)

**Goal**: Users can choose pending deposit, pay it, have unpaid seats returned via batches, and cancel bookings.

**Independent Test**: Server tests: hold → deposit → pay → CONFIRMED; deposit → deadline → RETURN_PENDING in one
open batch → reopen at `release_at`; cancel CONFIRMED (card via PgStub, deposit) and PENDING_DEPOSIT.

### Tests for User Story 4 ⚠️

- [ ] T107 [P] [US4] `server/src/test/java/com/example/reservation/deposit/DepositTest.java`: deposit HELD → 200 `{status:"PENDING_DEPOSIT", depositDeadline = now + depositDeadlineSec, amount}`, seats PENDING_DEPOSIT, `payment_method` DEPOSIT, COMPLETED sent + kid revoked, `depositsRequested +1`; existing REQUESTED payment → CANCELED `DEPOSIT_CHOSEN` (D-019); not HELD / expired / other user → 409 `RESERVATION_NOT_PAYABLE`; user now has booking → new hold → 409 `USER_ALREADY_PURCHASED` (I11)
- [ ] T108 [P] [US4] `server/src/test/java/com/example/reservation/deposit/DepositPayTest.java`: pay correct amount before deadline → 200 CONFIRMED, seats SOLD, `depositsPaid +1`; wrong amount / wrong user / at or after deadline / not PENDING_DEPOSIT → 409 `DEPOSIT_NOT_ACCEPTABLE`; unknown → 404; no admission key needed
- [ ] T109 [P] [US4] `server/src/test/java/com/example/reservation/deposit/ReturnBatchTest.java`: two deposits expiring at different times join one batch whose `release_at` = first expiry + `returnDelaySec` (unchanged by joining), `seat_count` sums, seats RETURN_PENDING with `release_batch_id`, `GET /seats.releaseAt` set, `depositsExpired` counts; at most one open batch; after `release_at` all seats of the batch AVAILABLE together within one reopen run, batch `released=true, released_at`, `reopenCount +1`, `reopenSeats += n` (I13); a later expiry after reopen creates a new batch
- [ ] T110 [P] [US4] `server/src/test/java/com/example/reservation/reservation/CancelTest.java` (PgStub): CONFIRMED card + PG cancel 200 CANCELED → 200, payment CANCELED, seats AVAILABLE, `userCancels +1`; PG 500/timeout/404 → 502 `PG_UNAVAILABLE`, nothing changed (D-022); CONFIRMED deposit and PENDING_DEPOSIT → 200 CANCELED, seats AVAILABLE; HELD / EXPIRED / CANCELED → 409 `RESERVATION_NOT_CANCELABLE`; other user → 409; unknown → 404; after cancel the user can hold again

### Implementation for User Story 4

- [ ] T111 [P] [US4] Implement `server/src/main/java/com/example/reservation/deposit/ReleaseBatchRepository.java` (`lockOpenBatch()` `FOR UPDATE`, `create(releaseAt)`, `addSeats(batchId, n)`, `dueOpenBatches(now)`, `close(batchId, now)`)
- [ ] T112 [US4] Implement `server/src/main/java/com/example/reservation/deposit/DepositService.java` + `POST /holds/{id}/deposit` in `server/src/main/java/com/example/reservation/deposit/DepositController.java`: AdmissionGuard (exception target `{id}`); lock reservation; checks → 409 `RESERVATION_NOT_PAYABLE`; PENDING_DEPOSIT, `payment_method='DEPOSIT'`, `deposit_deadline = now + depositDeadlineSec`, `toPendingDeposit`, cancel REQUESTED payment (D-019); `depositsRequested +1`; after commit `SlotNotifier.completed(kid)` → 200
- [ ] T113 [US4] Implement `POST /deposits/{id}/pay` in `server/src/main/java/com/example/reservation/deposit/DepositPayService.java` + DepositController (no admission check; `{userId, amount}` validation D-025; conditions of spec §6.7; `toSold`; `depositsPaid +1`)
- [ ] T114 [US4] Implement `server/src/main/java/com/example/reservation/deposit/DepositDeadlineScheduler.java` (`fixedDelay = deposit-interval-ms`, `AdminGate.runIfOpen`): tx per run: `SELECT … WHERE status='PENDING_DEPOSIT' AND deposit_deadline <= :now ORDER BY deposit_deadline FOR UPDATE SKIP LOCKED`; each → DEPOSIT_EXPIRED, `depositsExpired +1`, `markReleased`; if sale end processed → `returnImmediately` (D-040); else lock/create open batch (`release_at = now + returnDelaySec`), `toReturnPending(batchId)`, `addSeats` (lock order: reservations → batch → seats, research §4)
- [ ] T115 [US4] Implement `server/src/main/java/com/example/reservation/deposit/ReopenScheduler.java` (`fixedDelay = reopen-interval-ms`): for each open batch with `release_at <= now`, one tx: lock batch, seats `WHERE release_batch_id = :b AND status = 'RETURN_PENDING'` → AVAILABLE, `current_reservation_id = NULL`, `release_batch_id = NULL`, version+1; close batch (`released=true`, `released_at = now`); `reopenCount +1`, `reopenSeats += n` (spec §6.8)
- [ ] T116 [US4] Implement `server/src/main/java/com/example/reservation/reservation/CancelService.java` + `POST /reservations/{id}/cancel` in ReservationController per spec §6.9 and D-022 (PG cancel outside tx using `PgClient` from US2; conditional DB update; `userCancels +1`; payment CANCELED for card)
- [ ] T117 [US4] Run `./gradlew :server:test` for US4 classes; all pass

**Checkpoint**: All reservation paths and seat-return paths of spec §4.4 work.

---

## Phase 7: User Story 5 — Sale end (Priority: P3)

**Goal**: After `saleEndAt`, no new holds or seat views; HELD reservations expire; open batch closes silently;
queue closes; ongoing CONFIRMING/PENDING_DEPOSIT finish under their own rules (I14).

**Independent Test**: Reset with `saleEndAt` = now + 10 s on server and queue, create HELD / PENDING_DEPOSIT /
RETURN_PENDING state, advance clocks 11 s, verify both services.

### Tests for User Story 5 ⚠️

- [ ] T118 [P] [US5] `server/src/test/java/com/example/reservation/sale/SaleEndTest.java`: after now ≥ saleEndAt, `GET /seats` and `POST /holds` → 409 `SALE_ENDED` immediately (before scheduler); within ~1 s all HELD → EXPIRED with seats AVAILABLE and HOLD_CLEARED; open batch seats → AVAILABLE, batch closed, `reopenCount`/`reopenSeats` unchanged; end processed once (`sale_period.end_processed`); checkout/deposit/release/confirm/deposit pay/cancel/lookup on existing reservations still follow their own rules; PENDING_DEPOSIT expiring after end processing → seats AVAILABLE immediately (D-040); sale period survives restart (re-run `StartupInitializer` without reset keeps `saleEndAt`)
- [ ] T119 [P] [US5] `server/src/test/java/com/example/reservation/sale/SaleEndRaceTest.java`: a hold whose lock wait crosses `saleEndAt` (hold seat with tx paused via advisory lock held by test, advance clock, release) → 409 `SALE_ENDED` with seats unacquired (spec §6.5 step 6); after processing 0 HELD (I14)
- [ ] T120 [P] [US5] `queue/src/test/java/com/example/queue/core/SaleEndQueueTest.java`: at now ≥ saleEndAt all WAITING/ADMITTED → CLOSED(SALE_ENDED), slots released, `closedSaleEnded` counted; enter afterwards → 200 CLOSED with reason SALE_ENDED; status shows reason

### Implementation for User Story 5

- [ ] T121 [US5] Implement `server/src/main/java/com/example/reservation/sale/SaleEndScheduler.java` (`fixedDelay 1000`, `AdminGate.runIfOpen`): if now ≥ saleEndAt and not processed, one tx: lock HELD reservations (ascending id) → `ReservationEnder` EXPIRED (HOLD_CLEARED after commit); lock open batch → its RETURN_PENDING seats AVAILABLE, batch `released=true, released_at=now` (no counters); `markEndProcessed`; refresh SalePeriodService cache (spec §6.10)
- [ ] T122 [US5] Confirm `HoldService` step 9 sale-end re-check, `SeatController` check, and `DepositDeadlineScheduler` post-end branch use `SalePeriodService` correctly in `server/src/main/java/com/example/reservation/hold/HoldService.java`, `server/src/main/java/com/example/reservation/seat/SeatController.java`, `server/src/main/java/com/example/reservation/deposit/DepositDeadlineScheduler.java`
- [ ] T123 [US5] Complete queue sale-end handling in `queue/src/main/java/com/example/queue/core/QueueState.java`: `enter` after end → new CLOSED(SALE_ENDED) token; admission run closes all WAITING/ADMITTED and releases slots; `closedSaleEnded` counter (spec §7.6)
- [ ] T124 [US5] Run `./gradlew test`; all modules pass

**Checkpoint**: All user stories complete.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T125 [P] `server/src/test/java/com/example/reservation/invariants/EndToEndInvariantTest.java`: mixed workload (holds, releases, card confirms with PgStub random outcomes incl. timeouts, deposits paid/unpaid, cancels), then advance clock to the quiescent state (`holdTtlSec + confirmDeadlineSec + 5`) and assert all server-side invariants I1–I7, I9–I13, I16 via `InvariantChecker` and PgStub records (I3/I12 using the stub's own payment state)
- [ ] T126 [P] `queue/src/test/java/com/example/queue/core/QueueInvariantTest.java`: randomized enter/leave/events over many runs; assert I8 and I17 after every run
- [ ] T127 Review every scheduler and request path against spec §4.3 "Any other → unchanged, log only" and add `log.info` for ignored transitions in the affected service files under `server/src/main/java/com/example/reservation/`
- [ ] T128 Walk through `specs/001-reservation-test/quickstart.md` scenarios 1–7 against the three running services (docker compose + bootRun); fix any defect found (then re-run `./gradlew test`)
- [ ] T129 Check `docs/decisions.md` covers every undetermined choice made during implementation (append new D-xxx entries if not; never rewrite earlier ones)
- [ ] T130 Run `./gradlew clean test` from the repo root, capture per-module passed/failed/skipped counts from `*/build/test-results/test/*.xml` (or Gradle output) with the run timestamp
- [ ] T131 Write `docs/completion-report.md` with the 8 sections required by `CLAUDE.md` (summary; build/run/test commands + Java/Gradle/Spring Boot/PostgreSQL/Docker versions; per-spec-section implementation table; exact test results from T130; spec-section → `file::test` mapping including "none" rows; decision list D-001…; known limitations; least confident areas). Do not change code or tests while writing it

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none.
- **Foundational (Phase 2)**: after Setup; blocks all stories. T042–T048 need T021–T041.
- **US1 (Phase 3)**: after Foundational.
- **US2 (Phase 4)**: after Foundational. mock-pg tasks (T070–T072, T078–T082) are independent of US1; server tasks T073–T077, T083–T088 need US1's holds (T064–T066) to create reservations.
- **US3 (Phase 5)**: queue module tasks (T090–T094, T098–T103) need only Phase 1–2 queue items (T008, T016); T095–T097, T104–T105 need US1 (and US2 for COMPLETED on card).
- **US4 (Phase 6)**: needs US1; T110/T116 (card cancel) need US2's `PgClient` (T084).
- **US5 (Phase 7)**: needs US1 and US4 (batches); T120/T123 need US3's `QueueState`.
- **Polish (Phase 8)**: after all stories; T131 is last.

### Within Each User Story

- Tests first (must fail) → repositories/models → services → controllers/schedulers → run tests.

### Parallel Opportunities

- Setup: T003–T009 in parallel.
- Foundational: T011–T017 (per-module clock/errors), T022/T024–T027, T033–T035, T043–T045 in parallel.
- US2 mock-pg (T078–T080) and the whole queue module of US3 (T098–T100) can be built in parallel with US1.
- All story test tasks marked [P] can be written in parallel.

---

## Parallel Example: User Story 1

```bash
# Tests together:
Task: "SeatQueryTest in server/src/test/java/com/example/reservation/seat/SeatQueryTest.java"
Task: "HoldValidationTest in server/src/test/java/com/example/reservation/hold/HoldValidationTest.java"
Task: "IdempotencyTest in server/src/test/java/com/example/reservation/hold/IdempotencyTest.java"
Task: "StrategyConcurrencyTest in server/src/test/java/com/example/reservation/hold/StrategyConcurrencyTest.java"

# Strategies together (different files):
Task: "ConditionalStrategy in server/src/main/java/com/example/reservation/hold/ConditionalStrategy.java"
Task: "PessimisticStrategy in server/src/main/java/com/example/reservation/hold/PessimisticStrategy.java"
Task: "OptimisticStrategy in server/src/main/java/com/example/reservation/hold/OptimisticStrategy.java"
Task: "NaiveStrategy in server/src/main/java/com/example/reservation/hold/NaiveStrategy.java"
```

## Parallel Example: US2 + US3 alongside US1

```bash
Task: "mock-pg PaymentStore in mock-pg/src/main/java/com/example/mockpg/core/PaymentStore.java"
Task: "queue Token/Slot models in queue/src/main/java/com/example/queue/core/"
Task: "queue AdmissionKeyIssuer in queue/src/main/java/com/example/queue/key/AdmissionKeyIssuer.java"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 Setup → Phase 2 Foundational.
2. Phase 3 (US1) → **STOP and VALIDATE**: holds with all four strategies, idempotency, expiry, invariants.

### Incremental Delivery

1. Foundation → US1 (MVP: seat holds)
2. US2 → card payments end to end with mock-pg
3. US3 → real queue and admission keys
4. US4 → deposits, return batches, cancellations
5. US5 → sale end
6. Polish → invariants end to end, quickstart run, completion report

---

## Notes

- [P] = different files, no dependency on an unfinished task.
- Append to `docs/decisions.md` at the moment of any new undetermined decision.
- No `git commit` — a human commits.
- Stop at any checkpoint to validate a story independently.
