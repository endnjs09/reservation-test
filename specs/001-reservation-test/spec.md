# reservation-test Specification

Build three backend services for a ticket booking service: a **reservation server**, a **queue server**, and a **mock payment gateway (mock-pg)**.
They cover the ticket-opening flow **queue → seat hold (multiple seats, adjacent seats) → payment (two-step card / pending deposit) → confirmation**, plus **returned tickets**.

### Tech stack
| Item | Choice |
|---|---|
| Language / runtime | Java 21 |
| Framework | Spring Boot 4.1.2 |
| Build | Gradle (using the Wrapper), multi-module: `server`, `queue`, `mock-pg` |
| Common dependencies (all three modules) | Spring Web, Validation, Actuator (startup check via `/actuator/health`) |
| Reservation server DB | PostgreSQL 17 (run with Docker Compose) |
| Reservation server DB access | JdbcTemplate. Do not use JPA |
| Schema migration | Flyway |
| DB for reservation server tests | Testcontainers (PostgreSQL) |
| Other | Lombok |
| Not used | Spring Boot Docker Compose support, Spring Security |

---

## 1. Scope and principles

- Correctness: the invariants in section 9 must hold at all times (or at the times stated).
- Out of scope: authentication and authorization, real payments, deployment. Users are identified only by a `userId` string.
- Every number (seat count, capacity, TTLs, failure rates, etc.) is a setting. Most can be changed at runtime through `/admin/reset` (except values that need a restart, such as the connection pool size).
- The three services do not depend on each other's code. They communicate only over HTTP.
- Do not implement anything in section 10, "Do not implement".

---

## 2. Structure

### 2.1 Modules and ports
```
reservation-test/
├── settings.gradle          # include 'server', 'queue', 'mock-pg'
├── build.gradle             # common (Java 21)
├── docker-compose.yml       # postgres:17
├── server/    :8080  reservation server
├── queue/     :8082  queue server        (no DB, in-memory, single instance)
└── mock-pg/   :8081  mock payment gateway (no DB, in-memory)
```
- Start order: `docker compose up -d` → mock-pg → queue → server.
- The DB name is `reservation`. Use the same connection settings in `docker-compose.yml` and in the reservation server configuration.

### 2.2 Request paths
```
User ──① enter / poll position──▶ Queue server          position, admission key issued
  │◀────────── admission key ─────────┘
  │
  ├──② admission key + request──▶ Reservation server ──▶ DB     checks only the key's signature and expiry (no call to the queue server)
  │                                 ├──▶ Queue server            slot status notifications (holding / cleared / completed)
  │                                 └──▶ Payment gateway         approve, query, cancel
  │
  └──③ card authorization──────▶ Payment gateway               called by the user directly
```
- Position polling by waiting users **never reaches** the reservation server.
- After admission, requests go directly from the user to the reservation server. The reservation server does not call the queue server per request.

---

## 3. Time

### 3.1 Server clock
- Each of the three services has one **server clock**. Every "current time" used for behavior decisions comes from this clock. Do not read the system wall clock directly, and in DB queries pass the server clock value instead of using the DB's current-time functions for time comparisons.
- Server clock = `base time + monotonic time elapsed since the base was set + offset`.
  - Right after startup: base time = system time at startup, offset 0.
  - When `/admin/reset` includes `anchorAt`: base time = `anchorAt`, monotonic elapsed time restarts from the moment the reset request is received, offset 0.
  - Reset without `anchorAt`: base time = system time at that moment, offset 0.
  - Calling the clock advance API (3.2) adds to the offset.
- Because monotonic elapsed time is used, changes to the system wall clock do not affect the server clock.

### 3.2 Clock API (all three services)
- **`GET /admin/clock`** → 200 `{ "now" }`
- **`POST /admin/clock/advance`** `{ "seconds": 30 }` → moves the server clock forward by that amount. `seconds` must be greater than 0 (decimals allowed). 0 or less, or missing → 400 `VALIDATION_FAILED`. 200 `{ "now" }`.
  - There is no API to move the clock backward.
  - Decisions after moving the clock (expiry, deadlines, sale end, etc.) take effect when requests are processed and on each scheduler's next run.
- The three clocks are independent. To keep the three services at the same time, the caller sends the same `anchorAt` and the same advance amounts to all three.

### 3.3 Things that do not use the server clock
These run on real (monotonic) time and are not affected by moving the clock:
- scheduler intervals
- payment gateway approval delay and timeout hold, and the reservation server's timeouts for payment gateway calls
- retry intervals for notifications to the queue server (6.11)

### 3.4 Notation
- Duration settings (`...Sec`) are seconds of server clock time.
- Times in the API are ISO-8601 UTC strings.

---

## 4. Domain

### 4.1 Seats and grades
- `rows` rows starting at `A`, `cols` columns starting at `1`. Default 10 × 10 = 100 seats.
- Seat `id = row index × cols + column index + 1` (row and column indexes start at 0); `label` has the form `"A1"`.
- Grades are assigned by rows, starting from the front row. Setting `grades: [{ name, rows, price }]`; the sum of `rows` must equal `rows`.
- Grade names are chosen from `VIP`, `S`, `A`, `B`.

| Grade | Rows | Default rows | Seats | Price |
|---|---|---|---|---|
| VIP | 1 | A | 10 | 150000 |
| S | 2 | B–C | 20 | 120000 |
| A | 3 | D–F | 30 | 90000 |
| B | 4 | G–J | 40 | 60000 |

### 4.2 Status values
| Entity | Values |
|---|---|
| Seat | `AVAILABLE`, `HELD`, `PENDING_DEPOSIT`, `RETURN_PENDING`, `SOLD` |
| Reservation | `HELD`, `CONFIRMING`, `CONFIRMED`, `EXPIRED`, `RELEASED`, `PAYMENT_FAILED`, `PENDING_DEPOSIT`, `DEPOSIT_EXPIRED`, `CANCELED` |
| Payment (reservation server) | `REQUESTED`, `APPROVED`, `FAILED`, `CANCELED` |
| Queue token (queue server) | `WAITING`, `ADMITTED`, `COMPLETED`, `LEFT`, `EXPIRED`, `CLOSED` |

- A seat stays `HELD` while the reservation that owns it is `HELD` or `CONFIRMING`.
- A reservation has 1 to `maxSeatsPerUser` seats. All seats of one reservation are **always in the same state** (never partially held or partially returned).

### 4.3 Reservation state transitions
| Current | Event | Condition | Next | Seat and side effects |
|---|---|---|---|---|
| (none) | Hold succeeds | All requested seats acquired | HELD | Seats → HELD |
| HELD | Hold released | - | RELEASED | Seats returned immediately |
| HELD | Expiry scheduler | now ≥ hold expiry time | EXPIRED | Seats returned immediately |
| HELD | Approval request | Passes the checks in 6.6 | CONFIRMING | Approval deadline = now + `confirmDeadlineSec` |
| HELD | Approval request | Hold expiry time has passed | (unchanged) | 409 `RESERVATION_NOT_PAYABLE`, no payment gateway call |
| HELD | Choose pending deposit | Before expiry | PENDING_DEPOSIT | Seats → PENDING_DEPOSIT, COMPLETED notification for the queue slot |
| CONFIRMING | Payment gateway approval succeeds (in the request or in recovery) | - | CONFIRMED | Payment APPROVED, seats → SOLD, COMPLETED notification |
| CONFIRMING | Approval declined / cancellation confirmed in recovery | - | PAYMENT_FAILED | Payment FAILED or CANCELED, seats returned immediately |
| PENDING_DEPOSIT | Deposit | Before the deadline, amount matches | CONFIRMED | Seats → SOLD |
| PENDING_DEPOSIT | Deposit deadline scheduler | now ≥ deposit deadline | DEPOSIT_EXPIRED | Seats → RETURN_PENDING (added to a return batch) |
| PENDING_DEPOSIT | User cancels | - | CANCELED | Seats returned immediately |
| CONFIRMED | User cancels | For card: payment gateway cancellation succeeds | CANCELED | Seats returned immediately |
| (seat) RETURN_PENDING | Return batch opens | Batch open time reached | - | Seats → AVAILABLE (all seats of the batch together) |
| Any other | Any event | - | unchanged | Log only |

- **The expiry scheduler never touches CONFIRMING reservations.** They stay CONFIRMING after the approval deadline passes, and the recovery scheduler (6.6) resolves them.
- **Returning seats is always conditional**: a seat is returned only when its current owning reservation is the reservation being returned. When a reservation ends, record the release time (`released_at`) on that reservation's seat assignment rows.
- "Returned immediately" means the seat goes back to `AVAILABLE` and its current owning reservation is cleared.

### 4.4 How seats come back
| Path | How |
|---|---|
| Hold expiry, hold release, card payment failure (declined or reversed), hold expiry due to sale end | Immediately |
| User cancels a booking (CONFIRMED, PENDING_DEPOSIT) | Immediately |
| Pending deposit not paid | Return pending → **released together as a batch** (6.8) |

### 4.5 Policies
| Item | Policy |
|---|---|
| Hold lifetime | `holdTtlSec`, default 420 |
| One hold per user | At most one HELD or CONFIRMING reservation per user |
| One booking per user | A user with a CONFIRMED or PENDING_DEPOSIT reservation cannot make a new hold (409 `USER_ALREADY_PURCHASED`). At most `maxSeatsPerUser` (default 4) seats per booking |
| Payment order | One payment per reservation. A repeated checkout returns the existing order |
| Check before approval | Call payment gateway approval only when the reservation is HELD, not expired, and user and amount match |
| Unknown approval result | Query the payment gateway once → if still unknown, keep CONFIRMING; the recovery scheduler handles it |
| Repeated hold requests | Idempotent by `Idempotency-Key`. Same key + same content → same response as the first; same key + different content → 422 |

---

## 5. Data (reservation server)

### 5.1 Tables
Use a schema migration tool. The final schema is below. Column names appear as-is in `/admin/snapshot` (6.14).

**seats**
| Column | Description |
|---|---|
| `id` | Seat id (formula in 4.1), PK |
| `label` | `"A1"`, unique |
| `row_index`, `col_index` | Start at 0 |
| `grade`, `price` | Grade, price |
| `status` | Only the 5 seat states in 4.2 |
| `current_reservation_id` | Reservation that currently owns the seat. Null if none |
| `release_batch_id` | Return batch the seat belongs to while RETURN_PENDING. Otherwise null |
| `version` | Incremented by 1 on every change (default 0) |
| `updated_at` | |

**reservations**
| Column | Description |
|---|---|
| `id` | PK, auto-increment |
| `seat_id` | Representative seat (smallest seat id in the reservation) |
| `user_id` | |
| `status` | Only the 9 reservation states in 4.2 |
| `seat_count` | Number of seats |
| `payment_method` | `CARD` or `DEPOSIT`; null until decided |
| `hold_expires_at`, `confirm_deadline`, `deposit_deadline` | Hold expiry time, approval deadline, deposit deadline |
| `idempotency_key` | Unique |
| `admission_kid` | `kid` of the admission key used for the hold. Null when admission key checks are disabled |
| `created_at`, `updated_at` | |

**reservation_seats** (seat list of a reservation. **This table is the source of truth for a reservation's seats**; `reservations.seat_id` is only the representative seat)
| Column | Description |
|---|---|
| `reservation_id`, `seat_id` | Composite PK |
| `released_at` | Time the reservation ended and let go of the seat. Null while it still holds it |

**payments**
| Column | Description |
|---|---|
| `id` | UUID, PK. Used as the order id (`orderId`) |
| `reservation_id` | Unique (one payment per reservation) |
| `amount` | Sum of seat prices |
| `status` | Only the 4 payment states in 4.2 |
| `payment_key` | Key issued by the payment gateway |
| `fail_reason` | |
| `created_at`, `updated_at` | |

**release_batches** (return batches)
| Column | Description |
|---|---|
| `id` | PK, auto-increment |
| `release_at` | Scheduled open time |
| `released` | Whether opened (or cleaned up by sale end). Default false |
| `released_at` | Actual open time |
| `seat_count` | Number of seats added to the batch. Default 0 |

- **At most one batch is open (`released = false`) at any time.**
- The sale period (start time, end time, whether end processing is done) is also stored in the DB, so the sale end time survives a reservation server restart (until `/admin/reset`).

### 5.2 DB protection constraints (backstop)
When the setting `dbBackstop` is true, the DB has the two uniqueness constraints below; when false, they are removed. Apply the setting at server startup and on `/admin/reset`.
- At most one `reservation_seats` row with `released_at` null per seat
- At most one HELD or CONFIRMING reservation per user

Violations are returned as 409 `SEAT_UNAVAILABLE` and 409 `USER_ALREADY_HOLDING` respectively.
With the `naive` strategy (6.5) and backstop false, the same seat can be assigned to several reservations at once. This combination exists **for comparison, to observe behavior without protection**. It must not happen with any other combination.

### 5.3 Connection pool
- DB connection pool maximum size, default 20. A configuration file value; changing it requires a restart.

---

## 6. Reservation server (:8080)

### 6.1 Configuration
```yaml
server.port: 8080
reservation:
  strategy: conditional          # conditional | pessimistic | optimistic | naive
  db-backstop: true
  hold-ttl-sec: 420
  max-seats-per-user: 4
  deposit-deadline-sec: 60
  return-delay-sec: 30
  sale-duration-sec: 1200
  seats: { rows: 10, cols: 10 }
  grades: [ {name: VIP, rows: 1, price: 150000}, {name: S, rows: 2, price: 120000},
            {name: A, rows: 3, price: 90000},    {name: B, rows: 4, price: 60000} ]
  payment.confirm-deadline-sec: 30
  scheduler: { expiry-interval-ms: 1000, recovery-interval-ms: 5000, deposit-interval-ms: 1000,
               reopen-interval-ms: 1000, expiry-batch-size: 500 }
  pg: { base-url: http://localhost:8081, connect-timeout-ms: 1000, request-timeout-ms: 3000 }
admission:
  required: true                 # if false, admission key checks (6.3) are skipped
  key-secret: dev-only-change-me # same value as the queue server
queue-server.base-url: http://localhost:8082   # target of slot notifications (6.11)
internal.secret: dev-internal-secret            # same value as the queue server
```
- `scheduler.*` are each scheduler's interval (real time, ms) and the expiry scheduler's batch size: `expiry` = hold expiry (6.12), `recovery` = payment recovery (6.6), `deposit` = deposit deadline (6.8), `reopen` = return batch opening (6.8).
- `strategy`, `db-backstop`, time values, seats and grades, and payment values are held as runtime configuration and changed by `/admin/reset` (6.14).

### 6.2 Common
- Requests and responses are JSON, UTF-8.
- Error response format `{ "code", "message" }`.

| code | HTTP | Meaning |
|---|---|---|
| VALIDATION_FAILED | 400 | Malformed request |
| KEY_INVALID | 403 | Admission key format, signature, runEpoch, or userId mismatch |
| KEY_EXPIRED | 403 | Admission key expired (except for the exception in 6.3) |
| KEY_REVOKED | 403 | Admission key whose purchase is finished |
| SEAT_NOT_FOUND | 404 | Seat does not exist |
| RESERVATION_NOT_FOUND | 404 | Reservation does not exist |
| SEAT_UNAVAILABLE | 409 | Seat already held or sold (includes `unavailableSeatIds`) |
| USER_ALREADY_HOLDING | 409 | User already has a HELD/CONFIRMING reservation |
| USER_ALREADY_PURCHASED | 409 | User already booked (CONFIRMED/PENDING_DEPOSIT) |
| RESERVATION_NOT_PAYABLE | 409 | Not HELD, expired, another user, or amount mismatch |
| DEPOSIT_NOT_ACCEPTABLE | 409 | Deposit not accepted (state, deadline, or amount) |
| RESERVATION_NOT_CANCELABLE | 409 | State cannot be canceled |
| SALE_ENDED | 409 | Sale has ended (6.10) |
| PAYMENT_DECLINED | 402 | Payment gateway declined approval |
| IDEMPOTENCY_KEY_REUSED | 422 | Same key, different content |
| PG_UNAVAILABLE | 502 | Payment gateway call failed (checkout, cancellation) |

### 6.3 Admission key check
- Applies to: `GET /seats`, `POST /holds`, `POST /holds/{id}/*`, `POST /payments/confirm`.
  (`POST /deposits/{id}/pay`, `POST /reservations/{id}/cancel`, and `GET /reservations/{id}` are not checked.)
- Header `X-Admission-Key` (format in 7.3).

| Check | On failure |
|---|---|
| Format and signature | 403 KEY_INVALID |
| `run` = current runEpoch | 403 KEY_INVALID |
| `uid` = `userId` in the request body (skipped for `GET /seats`, which has no body) | 403 KEY_INVALID |
| `kid` is not in the revoked list | 403 KEY_REVOKED |
| now < `exp` | 403 KEY_EXPIRED unless the exception below applies |

- **Expiry exception (protects payment in progress)**: even if the key has expired, when the user has a HELD or CONFIRMING reservation, `/holds/{id}/checkout`, `/holds/{id}/deposit`, `/holds/{id}/release`, and `/payments/confirm` **for that reservation** are allowed. `GET /seats` and a new `POST /holds` are blocked.
- **Revoked list**: `kid`s for which a COMPLETED notification (6.11) was sent. Kept in memory and cleared on reset.
- The reservation server does not call the queue server to check admission keys.
- Compare signatures in a way that does not leak information through timing (constant-time comparison).
- **Independent re-verification**: when a checked request ends with 2xx, re-check the same rules (signature, runEpoch, uid, expiry exception) with **a second, separately implemented check**. If this check fails, increment `counters.acceptedInvalidKeys` by 1 (invariant I16). The response is not changed.
- When `admission.required = false`, both the admission key check and the independent re-verification are skipped.

### 6.4 Seat query `GET /seats`
```json
{ "rows": 10, "cols": 10,
  "grades": [ { "name": "VIP", "rows": 1, "price": 150000 } ],
  "seats": [ { "id": 1, "label": "A1", "grade": "VIP", "status": "AVAILABLE" } ],
  "availableSeats": 0, "heldSeats": 10, "pendingDepositSeats": 4, "returnPendingSeats": 2,
  "soldOut": false, "releaseAt": null, "saleEndAt": "2026-11-01T11:20:00Z" }
```
- The 7 summary fields (`availableSeats`, `heldSeats`, `pendingDepositSeats`, `returnPendingSeats`, `soldOut`, `releaseAt`, `saleEndAt`) are **always** present.
- `soldOut` = 0 AVAILABLE seats and 0 HELD seats (only PENDING_DEPOSIT, RETURN_PENDING, and SOLD remain).
- `releaseAt` = scheduled open time of the open return batch, or null if none.
- The values need not come from one consistent point in time (slightly stale values are allowed).
- After the sale ends: 409 `SALE_ENDED` (6.10).

### 6.5 Hold
**`POST /holds`** headers `Idempotency-Key` (1–64 characters, required), `X-Admission-Key`
```json
{ "userId": "u-0001", "seatIds": [37, 38] }
```
- `userId`: 1–64 characters.
- `seatIds`: 1 to `maxSeatsPerUser` items, positive, no duplicates. A single `seatId` field is also accepted instead (treated as one seat). Both present or both missing → 400.
- Malformed request → 400 `VALIDATION_FAILED`. If any seat does not exist → 404 `SEAT_NOT_FOUND`.
- **All or nothing**: if any seat cannot be acquired, acquire none and return 409 `SEAT_UNAVAILABLE` with `unavailableSeatIds` (the ids that could not be acquired).
- **Lock order**: multiple seats are always acquired in ascending seat id order (deadlock prevention).
- Processing order (one transaction)
  1. Admission key check (6.3), sale end check (6.10).
  2. Idempotency check: if a reservation with the same `Idempotency-Key` exists, return **the same 201 body as the first successful response** when the content (`userId` and the set of seat ids, order-insensitive) is the same, otherwise 422 `IDEMPOTENCY_KEY_REUSED`. No new reservation is created.
  3. Serialize hold requests from the same user (not done for the `naive` strategy).
  4. User check: if the user has a booking → 409 `USER_ALREADY_PURCHASED`; if the user has a HELD/CONFIRMING reservation → 409 `USER_ALREADY_HOLDING`.
  5. Acquire seats (per strategy, table below). On failure, increment `counters.holdConflicts` by 1 for each seat not acquired.
  6. Check sale end again. If the end time passed while waiting for locks, finish with 409 `SALE_ENDED` (seats are left unacquired).
  7. Create the reservation (HELD, hold expiry time = now + `holdTtlSec`, admission key `kid`), the seat assignment rows, and the seats' current owning reservation.
  8. After commit, 201, then a HOLD_ACTIVE notification (6.11).
- Even if concurrent requests with the same key arrive together, only one reservation is created. Later requests are answered by the rule in step 2.
- 201 response: `{ "reservationId", "seats": [{ "id", "label", "grade", "price" }], "totalPrice", "status": "HELD", "holdExpiresAt" }` (`seats` in ascending id order)

| Strategy | How seats are acquired |
|---|---|
| conditional | A **single conditional update** that changes only AVAILABLE seats to HELD does the check and the change together. Fails if the number of changed seats differs from the number requested |
| pessimistic | **Lock** the requested seat rows in id order, then change them only if all are AVAILABLE |
| optimistic | Per seat, an update that applies only if `version` equals the value read earlier. If any fails, the whole hold fails. No retry |
| naive | Read status without locking, and if AVAILABLE, change it without a condition. No per-user serialization (step 3) either. **For comparison** (5.2) |

**`POST /holds/{id}/release`** `{ "userId" }`
- Reservation does not exist → 404 `RESERVATION_NOT_FOUND`; another user → 409 `RESERVATION_NOT_PAYABLE`.
- If HELD: RELEASED and seats returned immediately → 200 `{ "reservationId", "status": "RELEASED" }`.
- If already not HELD: change nothing and return 200 with the current status (idempotent).

### 6.6 Card payment (two-step)
```
User ──① checkout──────▶ Reservation server     create order (orderId, amount)
User ──② card auth─────▶ Payment gateway        paymentKey issued (no charge)
User ──③ confirm───────▶ Reservation server ──④ check, then approve──▶ Payment gateway   charged here
```
**① `POST /holds/{id}/checkout`** `{ "userId" }`
- Lock the reservation; user mismatch, not HELD, or expired → 409 `RESERVATION_NOT_PAYABLE`.
- If a payment exists, return it; otherwise create one (REQUESTED, amount = sum of seat prices). The payment gateway is not called.
- 200 `{ "orderId", "amount" }`.

**② Card authorization**: the user calls the payment gateway `POST /auth` directly (section 8).

**③ `POST /payments/confirm`** `{ "userId", "orderId", "paymentKey", "amount" }`
1. (Transaction A) Lock the payment and the reservation. In any of the following cases return 409 `RESERVATION_NOT_PAYABLE` and **do not call payment gateway approval**: user mismatch, reservation not HELD, hold expiry time passed, amount mismatch, payment not REQUESTED. Otherwise set the reservation to CONFIRMING, record the approval deadline, store `payment_key` and `payment_method = CARD`. Commit.
2. (Outside a transaction) Payment gateway `POST /payments/confirm`.
3. (Transaction B) Apply the result
   - DONE → payment APPROVED, reservation CONFIRMED, seats SOLD (conditional), COMPLETED notification → **200** `{ "status": "CONFIRMED" }`
   - 4xx decline → payment FAILED, reservation PAYMENT_FAILED, seats returned immediately → **402** `PAYMENT_DECLINED`
   - Timeout, 5xx, or connection failure → payment gateway `GET /payments/{paymentKey}` once
     - DONE → success handling → 200
     - CANCELED → payment CANCELED, reservation PAYMENT_FAILED, seats returned → 402
     - 404 → payment FAILED, reservation PAYMENT_FAILED, seats returned → 402
     - AUTHORIZED / DECLINED → reversal `POST /payments/{key}/cancel`: on success payment CANCELED, reservation PAYMENT_FAILED, seats returned → 402 / on failure keep CONFIRMING → **202** `{ "status": "CONFIRMING" }`
     - Query fails → keep CONFIRMING → **202** `{ "status": "CONFIRMING" }` (the user checks the result with `GET /reservations/{id}`)
- An unknown order (`orderId`) is also 409 `RESERVATION_NOT_PAYABLE`.

**Recovery scheduler** (`recovery-interval-ms`): targets = reservations that are CONFIRMING with approval deadline ≤ now. For each reservation, **always start again with a payment gateway status query**.

| Query result | Handling |
|---|---|
| DONE | Success handling (payment APPROVED, reservation CONFIRMED, seats SOLD, COMPLETED notification and kid revoked) |
| CANCELED / 404 | Payment CANCELED (FAILED for 404), reservation PAYMENT_FAILED, seats returned, HOLD_CLEARED notification when applicable |
| AUTHORIZED / DECLINED | Reversal → on success, failure handling as above; on failure keep CONFIRMING |
| Query fails | Keep CONFIRMING, try again next run |
- No limit on retries. Increment `counters.recoveryAttempts` on each attempt and `counters.recovered` when a reservation is resolved.

### 6.7 Pending deposit and deposit
**`POST /holds/{id}/deposit`** `{ "userId" }`
- Must be HELD, not expired, and the user must match. Otherwise 409 `RESERVATION_NOT_PAYABLE`.
- Reservation PENDING_DEPOSIT, `payment_method = DEPOSIT`, deposit deadline = now + `depositDeadlineSec`. Seats PENDING_DEPOSIT. COMPLETED notification (slot released).
- 200 `{ "status": "PENDING_DEPOSIT", "depositDeadline", "amount" }`. The virtual account is only simulated (no payment gateway call).

**`POST /deposits/{reservationId}/pay`** `{ "userId", "amount" }` (acts as the bank, no admission key check)
- PENDING_DEPOSIT, now < deposit deadline, user and amount match → CONFIRMED, seats SOLD → 200 `{ "status": "CONFIRMED" }`.
- Otherwise 409 `DEPOSIT_NOT_ACCEPTABLE`. Unknown reservation → 404.

### 6.8 Deposit deadline and return batches
- **Deposit deadline scheduler** (`deposit-interval-ms`): reservations that are PENDING_DEPOSIT with deposit deadline ≤ now → DEPOSIT_EXPIRED, seats RETURN_PENDING, added to a batch.
  - If no batch is open, create a new batch with `release_at = now + returnDelaySec`; if one is open, join it (the countdown starts from the first unpaid reservation; joining does not change `release_at`).
  - Set the seats' `release_batch_id` to that batch and increase the batch's `seat_count`.
  - Unpaid seats after sale end processing (6.10) is done are not added to a batch; they go back to AVAILABLE immediately.
- **Return batch opening scheduler** (`reopen-interval-ms`): for each open batch with `release_at ≤ now`, **in one transaction**, change all RETURN_PENDING seats of the batch to AVAILABLE, clear their `release_batch_id`, then close the batch with `released = true`, `released_at = now`. `counters.reopenCount` +1, `counters.reopenSeats` + number of seats.

### 6.9 Booking cancellation and lookup
**`POST /reservations/{id}/cancel`** `{ "userId" }` (no admission key check)
- User mismatch → 409 `RESERVATION_NOT_CANCELABLE`; unknown reservation → 404.
- CONFIRMED + CARD: payment gateway `POST /payments/{key}/cancel` → on success payment CANCELED, reservation CANCELED, seats returned immediately → 200 / if the payment gateway call fails, 502 `PG_UNAVAILABLE` and nothing changes.
- CONFIRMED + DEPOSIT, or PENDING_DEPOSIT: reservation CANCELED, seats returned immediately → 200.
- 200 body `{ "reservationId", "status": "CANCELED" }`.
- Any other state: 409 `RESERVATION_NOT_CANCELABLE`.

**`GET /reservations/{id}`** → `{ "reservationId", "seatId", "label", "grade", "seats": [{ "id", "label", "grade", "price" }], "userId", "status", "holdExpiresAt", "paymentMethod", "depositDeadline", "payment": { "orderId", "status" } | null }` (`seatId`, `label`, `grade` are the representative seat's). Unknown reservation → 404.

### 6.10 Sale end
- Sale period: set on `/admin/reset` (6.14). When the server starts with no stored sale period, it sets the period to `saleDurationSec` starting at startup and stores it. If a stored sale period exists, it is kept across restarts.
- When now ≥ `saleEndAt`:
  - `GET /seats` and a new `POST /holds` → 409 `SALE_ENDED` (even before the sale end scheduler runs).
  - Checkout, deposit, release, confirm, deposit payment, cancellation, and lookup for existing reservations continue under their own rules.
- **Sale end processing** (scheduler, every 1 second, once per sale period): in one transaction
  - All HELD reservations → EXPIRED, seats returned immediately, HOLD_CLEARED notification for the affected users.
  - Seats of the open return batch go back to AVAILABLE without an open event, and the batch is closed (`reopenCount` and `reopenSeats` are not incremented).
  - Record that end processing is done.
- After end processing, no HELD reservation may remain (I14). A hold being processed near the end time is either rejected in step 6 of 6.5 or expired by end processing.
- CONFIRMING and PENDING_DEPOSIT reservations proceed to their conclusion under their own rules (recovery and deposit deadline continue).
- The queue server closes queue tokens (7.6).

### 6.11 Notifications to the queue server
- Sent to the queue server `POST /internal/slots/{kid}/events` (7.5) where reservations are created or end. Send them **after** the request transaction commits, without making the request wait for them (asynchronous).
- Reservations without a `kid` (admission key checks disabled) produce no notifications.

| Event | Notification sent |
|---|---|
| Hold succeeds | HOLD_ACTIVE |
| All of the user's HELD/CONFIRMING reservations have ended (release, expiry, payment failure, sale end; including those outside a request) | HOLD_CLEARED |
| Card CONFIRMED (in the request or by the recovery scheduler), pending deposit chosen | COMPLETED, and add the `kid` to the revoked list |
| `soldOut` changes, and every 5 seconds | `POST /internal/sale-state` `{ soldOut, releaseAt }` |

- The event's `at` is the time the event happened (server clock), not the retry time.
- On failure, retry 3 times (after 1, 2, and 4 seconds of real time). If it still fails, drop it, increment `counters.slotNotifyDropped`, and record `{ kid, type, at }` in `droppedNotifications` (included in the snapshot, cleared on reset).
- A failed notification does not change the result of the original request.

### 6.12 Hold expiry scheduler (`expiry-interval-ms`)
- Process reservations that are HELD with hold expiry time ≤ now, earliest expiry first, up to `expiry-batch-size` at a time: EXPIRED, conditional seat return, record `released_at`, HOLD_CLEARED notification, `counters.expiredByScheduler` +1.
- Reservations held by another operation are skipped without waiting and checked again on the next run.
- CONFIRMING reservations are not targets.
- If a run processed a full `expiry-batch-size`, repeat within the same run.

### 6.13 Counters
Kept in memory and reset to 0 on reset. Names and when they increase:

| Name | Increases when |
|---|---|
| `holdAttempts` | each `POST /holds` that passes format validation (including idempotent repeats) |
| `holdSuccess` | a new reservation is created |
| `holdConflicts` | per seat not acquired when acquisition fails |
| `confirms` | a card payment becomes CONFIRMED (in the request or in recovery) |
| `declines` | the payment gateway declines approval (402) |
| `notPayable` | each `RESERVATION_NOT_PAYABLE` response |
| `expiredByScheduler` | per reservation expired by the hold expiry scheduler |
| `recoveryAttempts`, `recovered` | 6.6 |
| `immediateReturns` | by the number of seats, each time seats are returned immediately |
| `userCancels` | each successful booking cancellation |
| `depositsRequested`, `depositsPaid`, `depositsExpired` | pending deposit chosen, deposit paid, deposit deadline passed |
| `reopenCount`, `reopenSeats` | 6.8 |
| `slotNotifyDropped` | 6.11 |
| `acceptedInvalidKeys` | 6.3 independent re-verification failed |

### 6.14 Admin API
**`POST /admin/reset`** (call only when there is no traffic)
```json
{ "rows": 10, "cols": 10, "grades": [ ... ],
  "strategy": "conditional", "dbBackstop": true,
  "holdTtlSec": 420, "confirmDeadlineSec": 30, "maxSeatsPerUser": 4,
  "depositDeadlineSec": 60, "returnDelaySec": 30, "saleDurationSec": 1200,
  "anchorAt": "...", "saleEndAt": "...", "runEpoch": "..." }
```
- Every field is optional. Missing fields keep their current values. Sum of `grades` rows ≠ `rows`, an unknown strategy, or time or seat values of 0 or less → 400.
- `anchorAt`: reset the server clock base as in 3.1.
- Sale period: start = now (on the new base), end = `saleEndAt` if given, otherwise now + `saleDurationSec`.
- `runEpoch`: run identifier used by admission key checks. If missing, a new random value.
- Processing: clear all tables → recreate seats → apply backstop → clear the revoked list, the notification queue, `droppedNotifications`, and counters → 200 `{ "config" }`.

**`GET /admin/stats`**
```json
{ "serverTime": "...", "runEpoch": "...", "config": { ... }, "saleEndAt": "...",
  "seats": { "total": 100, "AVAILABLE": 12, "HELD": 30, "PENDING_DEPOSIT": 4, "RETURN_PENDING": 0, "SOLD": 54,
             "byGrade": { "VIP": { "AVAILABLE": 0, "HELD": 0, "PENDING_DEPOSIT": 0, "RETURN_PENDING": 0, "SOLD": 10 } } },
  "reservations": { "HELD": 0, "CONFIRMING": 0, "CONFIRMED": 0, "EXPIRED": 0, "RELEASED": 0, "PAYMENT_FAILED": 0,
                    "PENDING_DEPOSIT": 0, "DEPOSIT_EXPIRED": 0, "CANCELED": 0 },
  "payments": { "REQUESTED": 0, "APPROVED": 0, "FAILED": 0, "CANCELED": 0 },
  "releaseBatches": { "open": 1, "nextReleaseAt": "...|null", "totalReleased": 3 },
  "counters": { ... all of 6.13 ... } }
```
- Counts by state come from the DB; include every state, even when its count is 0.

**`GET /admin/snapshot`** (for invariant checks)
```json
{ "serverTime": "...", "runEpoch": "...",
  "seats": [ ... ], "reservations": [ ... ], "reservation_seats": [ ... ], "payments": [ ... ], "release_batches": [ ... ],
  "counters": { ... }, "droppedNotifications": [ { "kid", "type", "at" } ] }
```
- Contains **all rows** of the five tables, with the column names from 5.1. Must be a consistent state at a single point in time.

**Clock**: 3.2.

---

## 7. Queue server (:8082)

### 7.1 State (in memory, lost on restart)
| Item | Content |
|---|---|
| Queue tokens | `{ tokenId, userId, seq, status, createdAt, admittedAt, endedAt, endReason }`. Ended tokens are kept until reset |
| Admission slots | Per ADMITTED token `{ keyId, userId, admittedAt, expiresAt, busy, busySince, lastEventAt }` |
| User index | userId → active token (one WAITING/ADMITTED token per user) |
| Admission log | Per admission run `{ tickAt, admitted, seqFrom, seqTo, activeAfter }`. Keep the maximums `activeMax` and `admittedPerTickMax` |
| Sale state | `saleEndAt` (reset), `soldOut` (notified by the reservation server) |

### 7.2 Admission rules
- Admission scheduler (`admission-interval-ms`, default 1000): number admitted = `min(maxActive − current slot count, admitPerSec × interval seconds)`, in ascending seq order. On admission, issue an admission key and set the slot's `expiresAt = now + admissionTtlSec`.
- Positions are computed every run and only read when requested.
- Slot release: COMPLETED (reservation server notification), LEFT (`/queue/leave`), EXPIRED (now ≥ `expiresAt` and not busy).
- **Deferred expiry**: a busy slot is not expired even after `expiresAt` (protects payment in progress). A busy slot past its lifetime during payment is **normal**.
- **Busy cap**: even if busy, force expiry when now ≥ `expiresAt + busyMaxExtraSec`. Default 510 seconds (= hold lifetime 420 + approval deadline 30 + margin 60). Each forced expiry increments `expiredByBusyCap` and adds `{ kid, userId, at }` to `busyCapExpirations`.
- A user whose token is EXPIRED, LEFT, or COMPLETED gets a new token at the end of the queue on re-entry.
- Section 7.6 describes the only cases where the queue server removes waiting users on its own.

### 7.3 Admission key
- Format `v1.<payload>.<sig>` (base64url, no padding)
  - payload: `{ "kid": "<UUID>", "uid": "<userId>", "iat": <ms>, "exp": <ms>, "run": "<runEpoch>" }` (`iat` and `exp` are epoch ms on the server clock; `exp` = the slot's `expiresAt`)
  - sig: `HMAC-SHA256(keySecret, "v1." + payload)`
- `keySecret` is the same value as on the reservation server (configuration). `runEpoch` is set on each reset, and the same value is given to both services.

### 7.4 User API
**`POST /queue/enter`** `{ "userId" }`
- Returns the active token (WAITING/ADMITTED) if one exists; otherwise a new WAITING token (or a CLOSED token in the cases of 7.6).
- 200 `{ "token", "status", "position", "pollAfterMs", "reason" }`

**`GET /queue/status?token=`**
- WAITING: `{ "status", "position", "pollAfterMs" }`
- ADMITTED: `{ "status", "admissionKey", "admissionExpiresAt" }`
- Otherwise: `{ "status", "reason": "SALE_ENDED" | "SOLD_OUT" | null }`
- Unknown or malformed token → 400.
- `pollAfterMs`: position ≤ 50 → 1000, ≤ 300 → 2000, ≤ 1000 → 4000, otherwise 6000.
- The response has **no seat availability fields.** Waiting users cannot see seat availability.

**`POST /queue/leave`** `{ "token" }` → if WAITING/ADMITTED, LEFT and the slot is released. In any other state nothing changes. 200 `{ "left": true }`. Missing token → 400.

### 7.5 Internal API (reservation server → queue server)
Header `X-Internal-Secret` required. Wrong or missing → 401.

**`POST /internal/slots/{kid}/events`** `{ "type": "HOLD_ACTIVE" | "HOLD_CLEARED" | "COMPLETED", "at" }`
| type | Handling |
|---|---|
| HOLD_ACTIVE | busy = true, `busySince = at` |
| HOLD_CLEARED | busy = false (if the expiry time has already passed, EXPIRED on the next admission run) |
| COMPLETED | Token COMPLETED, slot released |
- Unknown kid or already-ended slot → 200 `{ "ignored": true }`.
- Per kid, ignore events whose `at` is older than the last applied event. Ignore every event after COMPLETED. Each ignored event increments `slotEvents.ignored`.

**`POST /internal/sale-state`** `{ "soldOut", "releaseAt" }` → used only by 7.6.

| Lost notification | Effect | Recovery |
|---|---|---|
| HOLD_ACTIVE | Slot may expire during payment (payment continues thanks to the reservation server's expiry exception) | None (capacity is briefly exceeded by 1) |
| HOLD_CLEARED | Slot stays tied up as busy | Busy cap |
| COMPLETED | A finished user keeps a slot | Expiry time (+ busy cap) |
| sale-state | Sold-out state applied late | Resent every 5 seconds |

### 7.6 Sale state handling
- **Sale end** (now ≥ `saleEndAt`): all WAITING/ADMITTED tokens → CLOSED (`SALE_ENDED`), all slots released. After that, enter returns 200 with a CLOSED (`SALE_ENDED`) token.
- **`closeQueueOnSoldOut`** (default true):
  - If true and sold out, change **only WAITING** tokens to CLOSED (`SOLD_OUT`), and later enters also get a CLOSED (`SOLD_OUT`) token. When soldOut = false arrives, accept entries again.
  - If false, nobody is removed even when sold out.
- **ADMITTED tokens are not closed because of sold-out.** An admitted user's slot is released only by the rules in 7.2 (COMPLETED, LEFT, expiry, busy cap). So after reopening, only as many users as there are actual free slots are admitted, and **the number of people holding a valid admission key ≤ maxActive** always holds (an admission key is a signed token, so it stays valid on the reservation server until `exp` regardless of the queue server's state).
- Admission keys are not revoked. An admitted user can use `GET /seats` until the key expires.

### 7.7 Admin API
- **`POST /admin/reset`** `{ "anchorAt", "maxActive", "admitPerSec", "admissionTtlSec", "busyMaxExtraSec", "saleEndAt", "closeQueueOnSoldOut", "runEpoch" }` (all optional; missing fields keep current values; `anchorAt` as in 3.1; clears all tokens, slots, logs, and counters) → 200 `{ "config" }`
- **`GET /admin/stats`** → `{ "serverTime", "WAITING", "ADMITTED", "COMPLETED", "LEFT", "EXPIRED", "CLOSED", "busy", "slotsBusyOverTtl", "counters": { "entered", "admitted", "keysIssued", "expired", "expiredByBusyCap", "closedSoldOut", "closedSaleEnded", "slotEvents": { "HOLD_ACTIVE", "HOLD_CLEARED", "COMPLETED", "ignored" } } }`
  - `slotsBusyOverTtl` = number of slots that are busy with now ≥ `expiresAt`.
- **`GET /admin/snapshot`** → `{ "serverTime", "config", "tokens": [...], "slots": [...], "admissions": [...], "activeMax", "admittedPerTickMax", "busyCapExpirations": [...] }`
- **Clock**: 3.2.

### 7.8 Configuration
```yaml
server.port: 8082
queue: { max-active: 200, admit-per-sec: 20, admission-ttl-sec: 420, busy-max-extra-sec: 510,
         close-queue-on-sold-out: true, admission-interval-ms: 1000 }
admission.key-secret: dev-only-change-me
internal.secret: dev-internal-secret
```

---

## 8. Mock payment gateway (mock-pg, :8081)

Payment data is kept in memory.

- **`POST /auth`** `{ orderId, amount }` → with probability `authFailureRate`, 400 `AUTH_FAILED`. On success, store the payment as AUTHORIZED and return 200 `{ paymentKey }`.
- **`POST /payments/confirm`** `{ paymentKey, orderId, amount }`
  - Delay U(`confirmMinMs`, `confirmMaxMs`) (real time).
  - With probability `timeoutRate`, do not respond and hold the request for `confirmMaxMs + 10000` ms. Even then, internally decide with 50% probability whether to approve (DONE) and store it (simulates an unknown result).
  - With probability `declineRate`, 400 `INSUFFICIENT_FUNDS` (stored as DECLINED).
  - Key, order, or amount mismatch, or an already processed payment → 400 `INVALID_REQUEST`.
  - Success 200 `{ status: "DONE", approvedAt }`.
- **`GET /payments/{key}`** → 200 `{ status }` (AUTHORIZED / DONE / DECLINED / CANCELED) or 404.
- **`POST /payments/{key}/cancel`** → DONE or AUTHORIZED becomes CANCELED, 200 `{ status }`. Other states (CANCELED, DECLINED) are unchanged and return their current status with 200 (idempotent). Unknown key → 404.
- **Admin**
  - `GET/PUT /admin/config` `{ authFailureRate, declineRate, timeoutRate, confirmMinMs, confirmMaxMs, seed }`. Probabilities must be in 0–1 and `0 ≤ confirmMinMs ≤ confirmMaxMs`, otherwise 400 `INVALID_REQUEST`. The same `seed` gives the same random results for the same request order; a null `seed` gives different results each time.
  - `POST /admin/reset` → clears payments and statistics (`anchorAt` in the body as in 3.1).
  - `GET /admin/stats` `{ serverTime, authRequests, authFailed, confirmRequests, done, declined, timedOut, canceled, confirmInflight, confirmLatency: { p50, p95, p99 } }`
  - `GET /admin/payments` → all payments `{ paymentKey, orderId, amount, status }`.
  - Clock: 3.2.
- Defaults: `authFailureRate` 0.03, `declineRate` 0.03, `timeoutRate` 0, `confirmMinMs` 100, `confirmMaxMs` 500, `seed` null.

---

## 9. Invariants

These must be checkable with `/admin/snapshot` (all three services) and `/admin/payments`. "Quiescent state" means `holdTtlSec + confirmDeadlineSec + 5` seconds after traffic stops.

| ID | Invariant |
|---|---|
| I1 | Seat SOLD ⇔ exactly one CONFIRMED reservation owns that seat (`released_at` null) |
| I2 | At most one `reservation_seats` row with `released_at` null per seat (except `naive` with backstop off) |
| I3 | Payment DONE (not canceled) at the payment gateway ⇔ APPROVED payment + CONFIRMED reservation on the reservation server (0 cases of money taken without a seat) |
| I4 | At most one HELD/CONFIRMING reservation per user |
| I5 | Sum of seats by state = total seats. HELD ⇔ owning reservation is HELD/CONFIRMING; PENDING_DEPOSIT ⇔ owning reservation is PENDING_DEPOSIT; RETURN_PENDING ⇔ belongs to the open batch |
| I6 | 0 HELD/CONFIRMING reservations in the quiescent state |
| I7 | Same `Idempotency-Key` ⇒ one reservation, same response body |
| I8 | Queue server: at any time slot count ≤ `maxActive`, admission in seq order, admissions per run ≤ `admitPerSec × interval seconds` |
| I9 | 0 payment gateway approval calls for reservations found expired in the check before approval |
| I10 | All seats of one reservation are always in the same state |
| I11 | At most one CONFIRMED + PENDING_DEPOSIT reservation per user, with total seats ≤ `maxSeatsPerUser` |
| I12 | Payment gateway DONE (not canceled) payment ⇔ CONFIRMED reservation paid by CARD. Card payments of CANCELED reservations are also CANCELED at the payment gateway |
| I13 | RETURN_PENDING seats become AVAILABLE within the batch's `release_at` + 2 seconds (except via sale end processing) |
| I14 | After the sale ends: 0 WAITING/ADMITTED on the queue server and 0 HELD on the reservation server (within one sale end processing interval) |
| I15 | For each kid force-expired by the busy cap (`busyCapExpirations`), either (a) the reservation server's `droppedNotifications` contains a HOLD_CLEARED or COMPLETED for that kid, or (b) that kid's reservation was CONFIRMING at that time. A busy slot past its lifetime during payment is not a violation by itself |
| I16 | Reservation server `counters.acceptedInvalidKeys` = 0 |
| I17 | Number of users holding a valid admission key (not expired, current runEpoch) ≤ `maxActive` |

---

## 10. Do not implement

| Excluded |
|---|
| Multiple server instances, Redis, distributed locks |
| Retries, backoff, or circuit breakers for payment gateway calls (only the single query + reversal + recovery scheduler in 6.6) |
| Moving the queue to external storage, bot or macro blocking, request rate limiting, fairness policies (queue randomization, etc.), automatic admission rate control |
| Overload handling (dropping requests, 429/503) |
| Authentication and authorization, real payments, application Docker images, message queues, Kubernetes |
| Performance metrics, dashboards, UI |
