# Decisions

Decisions the specification (`specs/001-reservation-test/spec.md`) does not determine.
Append-only. A changed decision gets a new entry that names the one it replaces.

## D-001: Server clock resolution and time format
- Date: 2026-10-25
- Spec reference: 3.1, 3.4
- What was undetermined: "Times in the API are ISO-8601 UTC strings" — precision and exact format are not given; DB stores microseconds.
- Decision: Every server clock reading is truncated to milliseconds. All API times (and snapshot timestamps) are written as `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` (always 3 fraction digits, `Z` suffix). Inputs (`anchorAt`, `saleEndAt`) accept any ISO-8601 instant with `Z` or an offset; anything else → 400. Same in all three services.
- Alternatives considered: Jackson default `Instant` output (variable fraction digits, nanoseconds); microsecond precision to match PostgreSQL.
- Reason: Admission keys use epoch ms (7.3), so ms everywhere keeps key `exp`, DB values and responses comparable; a fixed width is easier for clients to parse.
- Affected files: `*/clock/ServerClock.java`, `*/web/` JSON time configuration
- Tests covering it: none yet
- Replaces: none

## D-002: Clock advance input handling
- Date: 2026-10-25
- Spec reference: 3.2
- What was undetermined: Precision of fractional `seconds`, and handling of non-numeric values.
- Decision: `seconds` must be a JSON number > 0; it is converted to nanoseconds (rounded half-up) and added to the offset; readings are still truncated to ms (D-001). Non-number, missing, 0 or negative → 400 `VALIDATION_FAILED` (mock-pg: same code).
- Alternatives considered: integer seconds only; ms rounding of the offset.
- Reason: Spec allows decimals; nanosecond offset loses nothing.
- Affected files: `*/clock/ClockController.java`, `*/clock/ServerClock.java`
- Tests covering it: none yet
- Replaces: none

## D-003: runEpoch before the first reset
- Date: 2026-10-25
- Spec reference: 6.14, 7.3
- What was undetermined: The runEpoch value right after startup (spec only says reset sets it, random if missing).
- Decision: Both the reservation server and queue server start with `admission.run-epoch` from configuration, default `"initial"` in both `application.yml` files. Reset without `runEpoch` generates a random UUID string.
- Alternatives considered: random value at each startup (keys unusable until a reset).
- Reason: Lets the services work together immediately after start (quickstart) while keeping reset semantics.
- Affected files: `server/src/main/resources/application.yml`, `queue/src/main/resources/application.yml`, runtime config classes
- Tests covering it: none yet
- Replaces: none

## D-004: Framework-level error responses
- Date: 2026-10-25
- Spec reference: 6.2, 7.4, 8
- What was undetermined: Responses for malformed JSON, wrong types, non-numeric path ids, unknown routes, wrong HTTP method, unexpected errors.
- Decision: All services use `{ "code", "message" }`. Malformed JSON / type mismatch / non-numeric path variable / bean-validation failure → 400 (`VALIDATION_FAILED` on server and queue, `INVALID_REQUEST` on mock-pg). Unknown route → 404 `NOT_FOUND`. Wrong method → 405 `METHOD_NOT_ALLOWED`. Unexpected exception → 500 `INTERNAL_ERROR`. Unknown JSON fields are ignored.
- Alternatives considered: Spring's default error body.
- Reason: One consistent error format for all clients and tests.
- Affected files: `*/web/GlobalExceptionHandler.java`
- Tests covering it: none yet
- Replaces: none

## D-005: Check order for `POST /holds`
- Date: 2026-10-25
- Spec reference: 6.5, 6.3, 6.13
- What was undetermined: Where body validation, seat existence (404) and the idempotency check sit relative to the admission key and sale end checks.
- Decision: (1) header/body format validation → 400; `holdAttempts` +1 once this passes; (2) admission key → 403; (3) sale end → 409 `SALE_ENDED`; (4) idempotency lookup → replay 201 or 422; (5) seat existence → 404 `SEAT_NOT_FOUND`; (6) per-user serialization; (7) user checks → 409; (8) seat acquisition → 409 `SEAT_UNAVAILABLE`; (9) sale end re-check → 409; (10) insert + commit → 201.
- Alternatives considered: seat existence before idempotency; key check before body validation.
- Reason: The uid check needs a parsed body; a replay must return the first response even if config changed later; follows the spec's numbered order otherwise.
- Affected files: `server/.../hold/HoldService.java`, `HoldController.java`
- Tests covering it: none yet
- Replaces: none

## D-006: Missing admission key header
- Date: 2026-10-25
- Spec reference: 6.3
- What was undetermined: Response when `X-Admission-Key` is absent or blank while `admission.required = true`.
- Decision: 403 `KEY_INVALID`.
- Alternatives considered: 400 `VALIDATION_FAILED`.
- Reason: A missing key is a key-format failure, which the table maps to `KEY_INVALID`.
- Affected files: `server/.../admission/AdmissionGuard.java`
- Tests covering it: none yet
- Replaces: none

## D-007: Resolving the expiry exception
- Date: 2026-10-25
- Spec reference: 6.3
- What was undetermined: How "for that reservation" is determined, and the result when the target reservation does not exist.
- Decision: The target reservation is the path `{id}` (checkout/deposit/release) or the reservation of the payment with the body `orderId` (confirm). The exception applies only if that reservation exists, its `user_id` equals the key's `uid`, and its status is HELD or CONFIRMING (read without locking). Otherwise the expired key gives 403 `KEY_EXPIRED` — including when the reservation/order is unknown.
- Alternatives considered: allow if the user has any HELD/CONFIRMING reservation; 404 before 403.
- Reason: Narrowest reading of "for that reservation"; key checks precede resource lookup.
- Affected files: `server/.../admission/AdmissionGuard.java`
- Tests covering it: none yet
- Replaces: none

## D-008: When a kid enters the revoked list
- Date: 2026-10-25
- Spec reference: 6.3, 6.11
- What was undetermined: Whether the kid is revoked when the COMPLETED notification is queued or only when delivered.
- Decision: The kid is added to the revoked list right after the transaction that produced COMPLETED commits, independent of delivery success.
- Alternatives considered: revoke only after the queue server acknowledges.
- Reason: The purchase is finished regardless of the notification's fate; dropped notifications must not re-enable a key.
- Affected files: `server/.../admission/RevokedKids.java`, `notify/SlotNotifier.java`
- Tests covering it: none yet
- Replaces: none

## D-009: Idempotent replay body
- Date: 2026-10-25
- Spec reference: 6.5 step 2, I7
- What was undetermined: How "the same 201 body as the first successful response" is produced without a column for it (5.1 is the final schema).
- Decision: Rebuild the body from the stored reservation: `reservationId`, its `reservation_seats` joined with `seats` (ascending id, grade, price), `totalPrice` = sum, `status` = `"HELD"` (always, as in the first response), `holdExpiresAt` = stored `hold_expires_at`. Seat grade/price only change through reset, which deletes reservations, so the body is identical.
- Alternatives considered: store the JSON in a new column or in memory.
- Reason: Keeps the §5.1 schema and survives restarts.
- Affected files: `server/.../hold/HoldService.java`
- Tests covering it: none yet
- Replaces: none

## D-010: Serialization of concurrent hold requests
- Date: 2026-10-25
- Spec reference: 6.5 steps 2–3
- What was undetermined: Mechanism for "only one reservation per key under concurrency" and "serialize hold requests from the same user".
- Decision: Inside the hold transaction: `pg_advisory_xact_lock(1, hashtext(idempotencyKey))` before the idempotency lookup (all strategies, including naive); then `pg_advisory_xact_lock(2, hashtext(userId))` before the user check (not for naive). Hash collisions only cause extra waiting. The unique constraint on `reservations.idempotency_key` remains as a backstop; a violation there is answered by re-running the step-2 lookup in a new transaction.
- Alternatives considered: in-JVM lock maps; `SELECT … FOR UPDATE` on a user row (no users table).
- Reason: Locks are released at commit/rollback automatically and work across connections.
- Affected files: `server/.../hold/HoldService.java`
- Tests covering it: none yet
- Replaces: none

## D-011: `conditional` strategy SQL
- Date: 2026-10-25
- Spec reference: 6.5 (strategy table, lock order)
- What was undetermined: How a "single conditional update" also guarantees ascending lock order.
- Decision: One statement: `UPDATE seats SET status='HELD', current_reservation_id=?, version=version+1, updated_at=? WHERE id IN (SELECT id FROM seats WHERE id = ANY(?) AND status='AVAILABLE' ORDER BY id FOR UPDATE) AND status='AVAILABLE' RETURNING id`. Fewer returned ids than requested → rollback, `unavailableSeatIds` = requested − returned. The reservation row is inserted first in the same transaction so its id can be set (rolled back on failure; ids may have gaps).
- Alternatives considered: plain `UPDATE … WHERE id = ANY(?) AND status='AVAILABLE'` (row visit order not guaranteed → deadlocks possible).
- Reason: Single statement does check+change, ordered row locks prevent deadlocks.
- Affected files: `server/.../hold/ConditionalStrategy.java`
- Tests covering it: none yet
- Replaces: none

## D-012: Unavailable seat reporting for pessimistic/optimistic
- Date: 2026-10-25
- Spec reference: 6.5
- What was undetermined: Whether acquisition stops at the first unavailable seat (then `unavailableSeatIds` is incomplete).
- Decision: All requested seats are examined in ascending order; every seat not acquired is reported. Pessimistic: `SELECT … WHERE id = ANY(?) ORDER BY id FOR UPDATE`, non-AVAILABLE ones are reported, update only if none. Optimistic: unlocked read of status+version, then per seat ascending `UPDATE … WHERE id=? AND version=? AND status='AVAILABLE'`; seats read as non-AVAILABLE or with 0 rows updated are reported; any failure rolls back; no retry.
- Alternatives considered: stop at first failure.
- Reason: Clients get the full conflict list; behavior is the same across strategies.
- Affected files: `server/.../hold/PessimisticStrategy.java`, `OptimisticStrategy.java`
- Tests covering it: none yet
- Replaces: none

## D-013: `holdConflicts` count
- Date: 2026-10-25
- Spec reference: 6.5 step 5, 6.13
- What was undetermined: "for each seat not acquired" could mean every requested seat (none are kept) or only the unavailable ones.
- Decision: Increment by the size of `unavailableSeatIds`.
- Alternatives considered: increment by the number of requested seats.
- Reason: Counts actual contention; matches the reported list.
- Affected files: `server/.../hold/HoldService.java`
- Tests covering it: none yet
- Replaces: none

## D-014: `naive` strategy and backstop violations
- Date: 2026-10-25
- Spec reference: 6.5, 5.2
- What was undetermined: Contents of `unavailableSeatIds` when the backstop unique index rejects a naive hold.
- Decision: After rollback, query the requested seats that have a `reservation_seats` row with `released_at IS NULL`; report those, or all requested seats if the query finds none. A user-backstop violation gives 409 `USER_ALREADY_HOLDING`.
- Alternatives considered: parse the PostgreSQL error detail.
- Reason: Does not depend on driver message text.
- Affected files: `server/.../hold/NaiveStrategy.java`, `HoldService.java`
- Tests covering it: none yet
- Replaces: none

## D-015: How the DB backstop is applied
- Date: 2026-10-25
- Spec reference: 5.2
- What was undetermined: Mechanism to add/remove the constraints at startup and reset.
- Decision: The two constraints are partial unique indexes managed by the application, not Flyway: `ux_rs_active_seat ON reservation_seats(seat_id) WHERE released_at IS NULL` and `ux_res_user_active ON reservations(user_id) WHERE status IN ('HELD','CONFIRMING')`, created with `CREATE UNIQUE INDEX IF NOT EXISTS` when true and `DROP INDEX IF EXISTS` when false. If creation fails because existing data violates it, startup/reset fails (reset clears tables first, so only startup can hit this).
- Alternatives considered: Flyway repeatable migrations; table constraints.
- Reason: Must toggle at runtime.
- Affected files: `server/.../admin/BackstopManager.java`
- Tests covering it: none yet
- Replaces: none

## D-016: Storage of the sale period
- Date: 2026-10-25
- Spec reference: 5.1, 6.10
- What was undetermined: Where the sale period is stored (5.1 lists only five tables).
- Decision: Extra single-row table `sale_period(id=1, start_at, end_at, end_processed)`. It is not part of `/admin/snapshot` (which has the five §5.1 tables); `saleEndAt` appears in stats and `/seats`.
- Alternatives considered: a generic key/value settings table.
- Reason: Spec requires DB persistence across restarts; a dedicated table is simplest.
- Affected files: `server/src/main/resources/db/migration/V1__init.sql`, `server/.../sale/SalePeriodRepository.java`
- Tests covering it: none yet
- Replaces: none

## D-017: Identity values after reset
- Date: 2026-10-25
- Spec reference: 6.14
- What was undetermined: Whether auto-increment ids continue after "clear all tables".
- Decision: `TRUNCATE … RESTART IDENTITY` — reservation and batch ids start at 1 again after each reset.
- Alternatives considered: `DELETE` (ids continue).
- Reason: Reproducible runs.
- Affected files: `server/.../admin/ResetService.java`
- Tests covering it: none yet
- Replaces: none

## D-018: Reservation server state at startup
- Date: 2026-10-25
- Spec reference: 1, 6.1, 6.10, 6.14
- What was undetermined: What survives a restart besides the sale period.
- Decision: Runtime config is reloaded from `application.yml` (reset-time changes are lost). Seats are created from config only if the `seats` table is empty; existing rows are kept. Counters, revoked list, dropped notifications are empty. Backstop is applied from config. The sale period is kept if stored, otherwise created as now + `saleDurationSec`.
- Alternatives considered: wipe the DB on startup; persist runtime config.
- Reason: Spec says only the sale end time explicitly survives; keeping data avoids destroying bookings on restart.
- Affected files: `server/.../admin/StartupInitializer.java`
- Tests covering it: none yet
- Replaces: none

## D-019: Payment rows for the deposit path
- Date: 2026-10-25
- Spec reference: 4.5, 5.1, 6.6, 6.7
- What was undetermined: Whether choosing pending deposit creates a `payments` row, and what happens to a REQUESTED card payment created by an earlier checkout.
- Decision: Deposit creates no payment row (`payment` in `GET /reservations/{id}` is null). If a REQUESTED payment exists from an earlier checkout, it becomes CANCELED with `fail_reason = 'DEPOSIT_CHOSEN'` in the same transaction. The deposit `amount` is the sum of seat prices.
- Alternatives considered: a payment row with a new status (not allowed by 4.2); leaving the REQUESTED payment.
- Reason: Payment states are card-specific; a stale REQUESTED payment would otherwise stay forever.
- Affected files: `server/.../deposit/DepositService.java`
- Tests covering it: none yet
- Replaces: none

## D-020: Malformed `orderId` on confirm
- Date: 2026-10-25
- Spec reference: 6.6 ③
- What was undetermined: Response for an `orderId` that is not a UUID.
- Decision: 409 `RESERVATION_NOT_PAYABLE` (same as an unknown order). Missing/blank `orderId` → 400.
- Alternatives considered: 400 for non-UUID.
- Reason: Spec treats unknown orders as not payable.
- Affected files: `server/.../payment/ConfirmService.java`
- Tests covering it: none yet
- Replaces: none

## D-021: Payment gateway 4xx responses on approval
- Date: 2026-10-25
- Spec reference: 6.6 ③ step 3
- What was undetermined: Whether every 4xx (including `INVALID_REQUEST`, 404) counts as "decline".
- Decision: Any 4xx from PG `POST /payments/confirm` is a decline: payment FAILED (`fail_reason` = PG `code`, or `HTTP_<status>`), reservation PAYMENT_FAILED, seats returned, 402 `PAYMENT_DECLINED`, `declines` +1.
- Alternatives considered: only `INSUFFICIENT_FUNDS` as decline, other 4xx as unknown.
- Reason: Spec says "4xx decline"; mock-pg never charges on a 4xx.
- Affected files: `server/.../payment/ConfirmService.java`
- Tests covering it: none yet
- Replaces: none

## D-022: Card booking cancellation edge cases
- Date: 2026-10-25
- Spec reference: 6.9
- What was undetermined: PG 404/unexpected status on cancel, and concurrent cancels.
- Decision: PG cancel is called outside a transaction; only a 200 with `status = CANCELED` counts as success. Timeout, 5xx, 404, or other status → 502 `PG_UNAVAILABLE`, nothing changes. The DB update is conditional on the reservation still being CONFIRMED; a losing concurrent cancel gets 409 `RESERVATION_NOT_CANCELABLE`.
- Alternatives considered: treat PG 404 as already canceled.
- Reason: Never mark CANCELED unless the gateway confirms the money is returned (I12).
- Affected files: `server/.../reservation/CancelService.java`
- Tests covering it: none yet
- Replaces: none

## D-023: Slot notification delivery details
- Date: 2026-10-25
- Spec reference: 6.11
- What was undetermined: HTTP timeouts, ordering, when HOLD_CLEARED fires, behavior across reset.
- Decision: Events are queued after commit and sent by a small executor; each event is retried independently (1 s, 2 s, 4 s real time) — no per-kid ordering (the queue server ignores older `at`). Timeouts: connect 1000 ms, read 2000 ms. Any non-2xx or I/O error is a failure. HOLD_CLEARED is sent when a HELD/CONFIRMING reservation with a kid ends as RELEASED, EXPIRED or PAYMENT_FAILED (one active reservation per user, so this is "all ended"). Reset increments a generation number; queued and retrying events from an older generation are discarded without counting as dropped.
- Alternatives considered: single ordered sender thread per kid.
- Reason: Simple, non-blocking; ordering is not required by the receiver.
- Affected files: `server/.../notify/SlotNotifier.java`
- Tests covering it: none yet
- Replaces: none

## D-024: Sale-state notifications
- Date: 2026-10-25
- Spec reference: 6.11
- What was undetermined: How a `soldOut` change is detected, and retry/drop handling (dropped records need a `kid`).
- Decision: A sale-state publisher runs every 1000 ms (real time), computes `soldOut`/`releaseAt` from the DB, and sends `POST /internal/sale-state` when `soldOut` differs from the last sent value or 5 s have passed since the last send. No retry; failures are logged only and not counted in `slotNotifyDropped`.
- Alternatives considered: recompute after every seat-changing commit.
- Reason: Bounded DB load; the 5 s resend is the spec's own recovery for lost sale-state.
- Affected files: `server/.../notify/SaleStatePublisher.java`
- Tests covering it: none yet
- Replaces: none

## D-025: Request field validation (reservation server)
- Date: 2026-10-25
- Spec reference: 6.5–6.9
- What was undetermined: Validation of `userId`, `amount`, `paymentKey`, `seatIds` size on non-hold endpoints.
- Decision: `userId` required, 1–64 chars on every endpoint that takes it. `seatIds` longer than current `maxSeatsPerUser` → 400. `amount` required, integer ≥ 0 (a mismatch is 409, not 400). `paymentKey`/`orderId` required, non-blank. `Idempotency-Key` 1–64 chars.
- Alternatives considered: reject `amount` 0 with 400.
- Reason: Format errors → 400, business mismatches → 409 as the spec's error table implies.
- Affected files: `server/.../*/dto/*.java`
- Tests covering it: none yet
- Replaces: none

## D-026: Time boundary rules
- Date: 2026-10-25
- Spec reference: 4.3, 6.3, 6.6, 6.7, 6.10
- What was undetermined: Whether "expired" is `>` or `≥` at the exact instant in request checks.
- Decision: A hold is expired when now ≥ `hold_expires_at`; a key is valid only while now < `exp`; a deposit is accepted only while now < `deposit_deadline`; the sale has ended when now ≥ `saleEndAt`. Same as the scheduler conditions in the spec.
- Alternatives considered: strict `>` for request checks.
- Reason: Consistent with the scheduler rules (≤ now).
- Affected files: server services
- Tests covering it: none yet
- Replaces: none

## D-027: Reset validation and `config` shape (reservation server)
- Date: 2026-10-25
- Spec reference: 6.14
- What was undetermined: Grade name/duplicate/price rules, `maxSeatsPerUser` bounds, past `saleEndAt`, contents of `config`.
- Decision: Grade names must be one of VIP/S/A/B, no duplicates, `rows ≥ 1`, `price ≥ 0`; grade order = front to back. `maxSeatsPerUser ≥ 1`. `saleEndAt` may be in the past (sale ends immediately). `config` = `{ rows, cols, grades, strategy, dbBackstop, holdTtlSec, confirmDeadlineSec, maxSeatsPerUser, depositDeadlineSec, returnDelaySec, saleDurationSec }` (same in reset response and stats). Changing only `rows` without matching `grades` → 400.
- Alternatives considered: free-form grade names.
- Reason: Spec lists the allowed names; echoing reset fields makes config round-trippable.
- Affected files: `server/.../admin/ResetService.java`
- Tests covering it: none yet
- Replaces: none

## D-028: Position of a newly entered token
- Date: 2026-10-25
- Spec reference: 7.2, 7.4
- What was undetermined: Position returned by `enter` before the next admission run computes positions.
- Decision: At enter, position = number of WAITING tokens including the new one (O(1) counter). Admission runs recompute positions (1-based, ascending seq) for all WAITING tokens.
- Alternatives considered: null until the next run.
- Reason: Gives a usable value immediately without per-request scans.
- Affected files: `queue/.../core/QueueState.java`
- Tests covering it: none yet
- Replaces: none

## D-029: Admission run details
- Date: 2026-10-25
- Spec reference: 7.2, 7.1, I8
- What was undetermined: Rounding of `admitPerSec × interval seconds`, order of steps in a run, which runs are logged.
- Decision: Per run: (1) expire slots (non-busy past `expiresAt`, busy past busy cap), (2) apply sale end close, (3) admit `min(maxActive − slots, floor(admitPerSec × admissionIntervalMs / 1000))` in seq order, (4) recompute positions. `activeMax`/`admittedPerTickMax` are updated every run; the admission log keeps only runs with `admitted > 0`.
- Alternatives considered: log every run.
- Reason: Freed slots are reused in the same run; the log stays small while I8 remains checkable.
- Affected files: `queue/.../core/AdmissionScheduler.java`
- Tests covering it: none yet
- Replaces: none

## D-030: Queue server errors and reset validation
- Date: 2026-10-25
- Spec reference: 7.4, 7.5, 7.7
- What was undetermined: Error codes, `leave` with an unknown token, reset validation.
- Decision: Unknown or malformed token (status, leave) → 400 `INVALID_TOKEN`; missing token / missing userId → 400 `VALIDATION_FAILED`; bad internal secret → 401 `UNAUTHORIZED`; unknown event type → 400. Reset: `maxActive`, `admitPerSec`, `admissionTtlSec` must be > 0, `busyMaxExtraSec ≥ 0`, else 400.
- Alternatives considered: `leave` with unknown token → 200.
- Reason: Same treatment as `status`.
- Affected files: `queue/.../api/*.java`
- Tests covering it: none yet
- Replaces: none

## D-031: Queue `enter` response fields
- Date: 2026-10-25
- Spec reference: 7.4
- What was undetermined: Values of `position`, `pollAfterMs`, `reason` for non-WAITING tokens; whether `enter` returns the admission key.
- Decision: `position` and `pollAfterMs` are numbers only for WAITING, otherwise null; `reason` is `SALE_ENDED`/`SOLD_OUT` for CLOSED, otherwise null. The admission key is only returned by `GET /queue/status`.
- Alternatives considered: include the key in `enter` for ADMITTED tokens.
- Reason: Keeps the documented enter shape; one place to get the key.
- Affected files: `queue/.../api/QueueController.java`
- Tests covering it: none yet
- Replaces: none

## D-032: Queue sale end before the first reset
- Date: 2026-10-25
- Spec reference: 7.1, 7.6
- What was undetermined: `saleEndAt` on the queue server at startup.
- Decision: `null` (no sale end) until a reset provides `saleEndAt`.
- Alternatives considered: a `sale-duration-sec` setting like the reservation server.
- Reason: 7.8 defines no such setting; the caller sends the same `saleEndAt` to both services.
- Affected files: `queue/.../core/QueueState.java`
- Tests covering it: none yet
- Replaces: none

## D-033: Queue counter semantics and end reasons
- Date: 2026-10-25
- Spec reference: 7.1, 7.5, 7.7
- What was undetermined: Whether `expired` includes busy-cap expiries; what `slotEvents.<TYPE>` counts; `endReason` values.
- Decision: `expired` counts normal TTL expiries only; busy-cap expiries count only in `expiredByBusyCap` (token status EXPIRED in both cases). `slotEvents.HOLD_ACTIVE/HOLD_CLEARED/COMPLETED` count applied events; every ignored event (unknown kid, ended slot, older `at`, after COMPLETED) counts in `ignored`. `endReason` ∈ `COMPLETED, LEFT, EXPIRED, BUSY_CAP, SALE_ENDED, SOLD_OUT`. Events with `at` equal to `lastEventAt` are applied.
- Alternatives considered: `expired` as a total.
- Reason: Separates the two expiry causes for I15 analysis.
- Affected files: `queue/.../core/QueueState.java`
- Tests covering it: none yet
- Replaces: none

## D-034: mock-pg `/auth` behavior
- Date: 2026-10-25
- Spec reference: 8
- What was undetermined: Repeated auth for the same order, validation, key format.
- Decision: Each successful `/auth` creates a new AUTHORIZED payment with a new key `pk_<uuid>`; an earlier key for the same order is unaffected. `orderId` non-blank and `amount` integer ≥ 0 required, else 400 `INVALID_REQUEST`. Unknown key on GET/cancel → 404 `NOT_FOUND`.
- Alternatives considered: one payment per orderId.
- Reason: Simplest model of card authorizations.
- Affected files: `mock-pg/.../core/PaymentStore.java`
- Tests covering it: none yet
- Replaces: none

## D-035: mock-pg confirm sequence, randomness, timeout outcome
- Date: 2026-10-25
- Spec reference: 8
- What was undetermined: Order of validation vs. random outcomes, which random draws happen, stored state when a timed-out request is not approved, final response of a held request.
- Decision: One shared `Random` (seeded if `seed` non-null) with synchronized draws. `/auth` draws 1 value. `/payments/confirm` always draws 3 values in this order at arrival: delay, timeout, decline. Then: validate (mismatch/already processed → 400 `INVALID_REQUEST`, atomically marking the payment as processing) → wait delay → if timeout: decide DONE with 50% (4th draw) and store, hold until `confirmMaxMs + 10000` ms from arrival, then answer 504 `TIMEOUT`; if not approved the payment stays AUTHORIZED → else if decline: DECLINED, 400 `INSUFFICIENT_FUNDS` → else DONE, 200.
- Alternatives considered: validate after delay; draw only as needed.
- Reason: Fixed draw count per request keeps results reproducible for the same request order.
- Affected files: `mock-pg/.../api/PgController.java`, `core/PgRandom.java`
- Tests covering it: none yet
- Replaces: none

## D-036: mock-pg admin config and reset
- Date: 2026-10-25
- Spec reference: 8
- What was undetermined: Partial PUT, effect of reset on config and RNG.
- Decision: `PUT /admin/config` merges present fields (explicit `"seed": null` clears the seed); the merged result is validated as a whole and the RNG is re-created. `POST /admin/reset` keeps config, clears payments/statistics, and re-creates the RNG from the current seed.
- Alternatives considered: PUT replaces everything; reset restores defaults.
- Reason: Reset + same seed reproduces a run.
- Affected files: `mock-pg/.../api/AdminController.java`
- Tests covering it: none yet
- Replaces: none

## D-037: mock-pg latency statistics
- Date: 2026-10-25
- Spec reference: 8
- What was undetermined: Samples and percentile method for `confirmLatency`.
- Decision: Wall-clock ms from arrival to response for every `/payments/confirm` that produced a response (including 400s and held 504s); nearest-rank percentiles; 0 when there are no samples. `confirmInflight` = requests currently being processed.
- Alternatives considered: exclude errors.
- Reason: Reflects what the reservation server experiences.
- Affected files: `mock-pg/.../core/LatencyStats.java`
- Tests covering it: none yet
- Replaces: none

## D-038: Recovery and Transaction B concurrency
- Date: 2026-10-25
- Spec reference: 6.6
- What was undetermined: What happens if the request thread and the recovery scheduler both resolve the same CONFIRMING reservation.
- Decision: Every resolution re-locks the reservation and applies its result only if the reservation is still CONFIRMING (and payment still REQUESTED); otherwise it does nothing. If the request thread loses, it answers from the reservation's current state (CONFIRMED → 200, PAYMENT_FAILED → 402, CONFIRMING → 202). Recovery processes candidates one by one ordered by `confirm_deadline`.
- Alternatives considered: skip recovery for reservations with an in-flight request (needs in-memory tracking).
- Reason: Idempotent, restart-safe.
- Affected files: `server/.../payment/ConfirmService.java`, `RecoveryScheduler.java`
- Tests covering it: none yet
- Replaces: none

## D-039: Snapshot format
- Date: 2026-10-25
- Spec reference: 6.14
- What was undetermined: Consistency mechanism, value formats, row order.
- Decision: The five tables are read in one `REPEATABLE READ, READ ONLY` transaction; rows ordered by primary key; keys are the §5.1 snake_case column names; timestamps formatted per D-001; counters and dropped notifications are copied from memory right after the read.
- Alternatives considered: table locks.
- Reason: Consistent DB view without blocking traffic.
- Affected files: `server/.../admin/SnapshotService.java`
- Tests covering it: none yet
- Replaces: none

## D-040: Unpaid deposit seats after sale end and counters
- Date: 2026-10-25
- Spec reference: 6.8, 6.13
- What was undetermined: Counter updates when unpaid deposit seats go straight to AVAILABLE after sale end processing.
- Decision: Reservation → DEPOSIT_EXPIRED, `depositsExpired` +1, seats returned immediately (conditional), `immediateReturns` + seat count; no batch involvement.
- Alternatives considered: count nothing.
- Reason: It is an immediate return by definition.
- Affected files: `server/.../deposit/DepositDeadlineScheduler.java`
- Tests covering it: none yet
- Replaces: none
