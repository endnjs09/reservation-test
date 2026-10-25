# Phase 0 Research: reservation-test

All Technical Context items are resolved; there are no remaining NEEDS CLARIFICATION markers. Items that change
observable behavior are logged in [`docs/decisions.md`](../../docs/decisions.md) and referenced as D-xxx.

## 1. Build and libraries

**Decision**: Gradle wrapper 9.x (pinned when the wrapper is generated; Spring Boot 4 requires Gradle ≥ 8.14),
Java 21 toolchain, `org.springframework.boot` 4.1.2 + `io.spring.dependency-management`. Common dependencies in the
root `build.gradle` (`subprojects {}`): `spring-boot-starter-web`, `-validation`, `-actuator`, Lombok
(`compileOnly` + `annotationProcessor`, also for tests), `spring-boot-starter-test`. Server adds
`spring-boot-starter-jdbc`, `spring-boot-starter-flyway`, `org.flywaydb:flyway-database-postgresql`,
`org.postgresql:postgresql`, and for tests `spring-boot-testcontainers`, `org.testcontainers:testcontainers-junit-jupiter`,
`org.testcontainers:testcontainers-postgresql` (Testcontainers 2.x artifact names, version from the Boot BOM).

**Rationale**: Matches the spec's stack table. Boot 4 split auto-configuration into modules, so Flyway needs its own
starter, and PostgreSQL support in Flyway ≥ 10 needs `flyway-database-postgresql`.

**Alternatives considered**: Maven (spec says Gradle); JPA (forbidden); `spring-boot-docker-compose` (forbidden).

## 2. JSON (Jackson 3)

**Decision**: Boot 4 ships Jackson 3 (`tools.jackson.*`). Times are serialized through a custom `Instant`
serializer with fixed `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` format (D-001); unknown properties ignored (D-004). Request DTOs
are Java records with Jakarta Validation annotations; endpoints that must distinguish "missing" from `null`
(mock-pg `PUT /admin/config`, D-036; reset bodies) read a `JsonNode`.

**Alternatives considered**: Jackson 2 compatibility module — unnecessary.

## 3. Server clock (all three services)

**Decision**: `ServerClock` holds `baseInstant`, `baseNanos` (`System.nanoTime()` at base set), `offsetNanos`.
`now() = baseInstant + (nanoTime − baseNanos) + offsetNanos`, truncated to ms (D-001). `reset(anchorAt|null)` and
`advance(seconds)` (D-002) are synchronized; `now()` reads a volatile immutable state object. A `java.time.Clock`
adapter is **not** exposed to Spring so that nothing accidentally reads the wall clock; every service/repository
takes `ServerClock` and passes `now` as a JDBC parameter (no `now()`/`CURRENT_TIMESTAMP` in SQL comparisons).

**Rationale**: Spec §3.1 formula; monotonic source is immune to wall-clock changes.

## 4. Reservation server persistence and transactions

**Decision**: `JdbcTemplate`/`NamedParameterJdbcTemplate` + `TransactionTemplate` (programmatic transactions so
PG calls are visibly outside transactions). Default isolation READ COMMITTED. HikariCP `maximum-pool-size: 20`
(config file, §5.3). Flyway `V1__init.sql` creates the §5.1 tables + `sale_period` (D-016) + supporting indexes;
the two backstop indexes are created/dropped by `BackstopManager` (D-015). A partial unique index
`ux_one_open_batch ON release_batches((true)) WHERE NOT released` enforces "at most one open batch" permanently.

**Global lock order** (deadlock prevention; internal design):
`advisory(idempotency key)` → `advisory(user)` → `reservations` rows (ascending id) → `payments` row →
`release_batches` row → `seats` rows (ascending id) → `reservation_seats`.
Every multi-row seat update uses `ORDER BY id FOR UPDATE` or a single ordered statement (D-011).

**Hold strategies**: see D-010 … D-014 and data-model.md §4.

**Rationale**: Programmatic transactions make the "Transaction A → PG call → Transaction B" structure of §6.6
explicit and avoid holding a pooled connection during remote calls.

## 5. Schedulers

**Decision**: Spring `@EnableScheduling` with a `ThreadPoolTaskScheduler` (pool size 8) so one slow job doesn't
delay others; each job uses `fixedDelay` from configuration (real time, §3.3):

| Job | Interval | Notes |
|---|---|---|
| Hold expiry | `expiry-interval-ms` | `FOR UPDATE SKIP LOCKED`, `ORDER BY hold_expires_at LIMIT batch`, loop while full batch (§6.12) |
| Payment recovery | `recovery-interval-ms` | one reservation at a time, PG query outside tx (D-038) |
| Deposit deadline | `deposit-interval-ms` | `FOR UPDATE SKIP LOCKED`; batch join/create (§6.8, D-040) |
| Return batch reopen | `reopen-interval-ms` | one tx per batch |
| Sale end | 1000 ms | once per sale period (`end_processed`) |
| Sale-state publisher | 1000 ms | D-024 |

A `ReentrantReadWriteLock` "admin gate": scheduler runs take the read lock (`tryLock`, skip run if unavailable);
`/admin/reset` takes the write lock so no job runs against half-reset state. Requests are not gated (spec: reset
only without traffic).

**Queue server**: one `AdmissionScheduler` at `admission-interval-ms`.

## 6. Payment gateway client (reservation server)

**Decision**: `RestClient` on `JdkClientHttpRequestFactory` with `HttpClient.connectTimeout(1000 ms)` and
per-request `readTimeout(3000 ms)` from `reservation.pg.*`. Outcome classification: 2xx → parsed body;
4xx → decline (D-021) / not-found (GET) ; 5xx, timeout, I/O → "unknown". Exactly the call pattern of §6.6
(confirm → one query → one reversal); no retries (§10).

## 7. Admission keys

**Decision**: Queue: `v1.` + base64url(no padding) of compact JSON payload + `.` + base64url(HMAC-SHA256(secret,
`"v1." + payloadB64`)). The HMAC input is the literal ASCII string `v1.<payloadB64>`.
Reservation server primary verifier: Jackson-bound payload record, `MessageDigest.isEqual` for signature.
Independent re-verifier (§6.3): separate class, splits with `indexOf`, decodes with `java.util.Base64` and reads
fields via a `JsonNode` tree, computes HMAC with a separately constructed `Mac`, constant-time XOR-accumulate
loop; re-evaluates signature, run, uid, and expiry-or-exception (using a fresh DB read for the exception).
Run after a 2xx in a response-commit hook (`HandlerInterceptor.afterCompletion` with status check) so the
response itself is unaffected; failures increment `acceptedInvalidKeys`.

**Error precedence** follows the §6.3 table order; details in D-005 … D-008.

## 8. Notifications (reservation server → queue server)

**Decision**: `TransactionSynchronization.afterCommit` (or explicit post-commit call in programmatic tx code)
enqueues `{kid, type, at, generation}` to a `ScheduledExecutorService` (4 threads). Retry 1 s/2 s/4 s real time,
then drop → `slotNotifyDropped` +1 and `droppedNotifications` entry. Details D-023; sale-state D-024.
Revocation timing D-008.

## 9. Queue server state and concurrency

**Decision**: One `QueueState` object guarded by a single `ReentrantLock` (all mutations and the admission run).
Structures: `LinkedHashMap<String, Token>` by tokenId (insertion = seq order), `TreeMap<Long seq, Token>` of WAITING
tokens, `HashMap<userId, Token>` active index, `HashMap<kid, Slot>`, admission log list. Token ids are random
UUIDs; kid is a random UUID. `pollAfterMs` thresholds per §7.4. Sale end and sold-out handling per §7.6,
D-029 … D-033.

**Rationale**: Single instance, in-memory (spec); a single lock makes I8/I17 trivially safe and the operations are
O(1)/O(n) small.

## 10. mock-pg

**Decision**: `ConcurrentHashMap<String, Payment>`; payment transitions with per-payment `synchronized`.
Spring MVC with **virtual threads** (`spring.threads.virtual.enabled=true`) so held timeout requests (up to
`confirmMaxMs + 10 s`) and delays (`Thread.sleep`) don't exhaust the Tomcat pool. Randomness and order D-035;
admin D-036; stats D-037.

## 11. Testing approach

**Decision**:
- Unit: `ServerClock` (all modules), HMAC issue/verify round trip (queue key accepted by server verifier using
  copied test vectors, since modules share no code), queue core (admission rules, busy cap, sold-out, sale end),
  mock-pg (seeded reproducibility, state machine).
- Reservation server integration: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + Testcontainers `postgres:17`
  (`@ServiceConnection`, one container reused across test classes); `admission.required` toggled per test class;
  a JDK `HttpServer` stub for mock-pg (programmable responses/delays) and for the queue server (records events).
  Concurrency tests fire parallel holds per strategy/backstop and assert invariants via `/admin/snapshot`.
- Fast scheduler intervals in tests via properties; time-driven scenarios use `/admin/clock/advance`.

**Alternatives considered**: WireMock (extra dependency, not needed); H2 (behavior differs from PostgreSQL locking).

## 12. Undetermined spec points — index

| Topic | Decision |
|---|---|
| Time precision/format | D-001, D-002 |
| Initial runEpoch | D-003 |
| Framework errors | D-004 |
| `POST /holds` check order | D-005 |
| Missing key header / expiry exception / revocation timing | D-006, D-007, D-008 |
| Idempotent replay body, concurrency | D-009, D-010 |
| Strategy SQL, conflict lists, counters, naive+backstop | D-011 … D-014 |
| Backstop mechanism, sale period storage, ids after reset, startup | D-015 … D-018 |
| Deposit vs. payment rows | D-019 |
| Confirm/cancel edge cases | D-020, D-021, D-022, D-038 |
| Notifications | D-023, D-024 |
| Validation and time boundaries | D-025, D-026, D-027 |
| Queue server details | D-028 … D-033 |
| mock-pg details | D-034 … D-037 |
| Snapshot | D-039 |
| Deposit after sale end | D-040 |
