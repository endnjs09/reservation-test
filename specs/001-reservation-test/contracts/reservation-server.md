# Contract: Reservation server (:8080)

Normative source: spec §6. This file fixes the details the spec leaves open (D-xxx in `docs/decisions.md`).
JSON UTF-8. Errors: `{ "code", "message" }` (+ `unavailableSeatIds` for `SEAT_UNAVAILABLE`).
Times: `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` (D-001). "Key" = admission key check (§6.3) when `admission.required`.

## Error precedence (all endpoints)
400 format (D-004, D-025) → 403 key (§6.3 table order; D-006, D-007) → 409 `SALE_ENDED` (where applicable) →
404 resource → 409 business rule. Framework errors: 404 `NOT_FOUND`, 405 `METHOD_NOT_ALLOWED`, 500 `INTERNAL_ERROR`.

## Endpoints

| Method & path | Key | Body | Success | Errors |
|---|---|---|---|---|
| `GET /seats` | yes (no uid, no exception) | — | 200 §6.4 | 403, 409 SALE_ENDED |
| `POST /holds` | yes | `{userId, seatIds}` or `{userId, seatId}`; header `Idempotency-Key` | 201 | 400, 403, 409 SALE_ENDED, 422, 404 SEAT_NOT_FOUND, 409 USER_ALREADY_PURCHASED / USER_ALREADY_HOLDING / SEAT_UNAVAILABLE |
| `POST /holds/{id}/release` | yes (+exception) | `{userId}` | 200 `{reservationId, status}` | 400, 403, 404, 409 RESERVATION_NOT_PAYABLE (other user) |
| `POST /holds/{id}/checkout` | yes (+exception) | `{userId}` | 200 `{orderId, amount}` | 400, 403, 404, 409 RESERVATION_NOT_PAYABLE |
| `POST /holds/{id}/deposit` | yes (+exception) | `{userId}` | 200 `{status:"PENDING_DEPOSIT", depositDeadline, amount}` | 400, 403, 404, 409 RESERVATION_NOT_PAYABLE |
| `POST /payments/confirm` | yes (+exception via orderId) | `{userId, orderId, paymentKey, amount}` | 200 `{status:"CONFIRMED"}` / 202 `{status:"CONFIRMING"}` | 400, 403, 409 RESERVATION_NOT_PAYABLE, 402 PAYMENT_DECLINED |
| `POST /deposits/{id}/pay` | no | `{userId, amount}` | 200 `{status:"CONFIRMED"}` | 400, 404, 409 DEPOSIT_NOT_ACCEPTABLE |
| `POST /reservations/{id}/cancel` | no | `{userId}` | 200 `{reservationId, status:"CANCELED"}` | 400, 404, 409 RESERVATION_NOT_CANCELABLE, 502 PG_UNAVAILABLE |
| `GET /reservations/{id}` | no | — | 200 §6.9 | 400, 404 |
| `POST /admin/reset` | no | §6.14 (all optional) | 200 `{config}` | 400 |
| `GET /admin/stats` | no | — | 200 §6.14 | — |
| `GET /admin/snapshot` | no | — | 200 §6.14 (D-039) | — |
| `GET /admin/clock` | no | — | 200 `{now}` | — |
| `POST /admin/clock/advance` | no | `{seconds}` | 200 `{now}` | 400 (D-002) |
| `GET /actuator/health` | no | — | 200 | — |

### `POST /holds` details
- Validation: `Idempotency-Key` 1–64; `userId` 1–64; exactly one of `seatIds` (1..`maxSeatsPerUser`, positive,
  distinct) / `seatId` (positive) → else 400. `holdAttempts` +1 after validation (D-005).
- Order: D-005. Replay body: D-009 (status `"HELD"`, original `holdExpiresAt`). Different `userId` or seat set with
  same key → 422 `IDEMPOTENCY_KEY_REUSED`.
- 201: `{ reservationId, seats:[{id,label,grade,price}] (asc), totalPrice, status:"HELD", holdExpiresAt }`.
- 409 `SEAT_UNAVAILABLE`: `{ code, message, unavailableSeatIds:[…] (asc) }` (D-012, D-014).
- After commit: HOLD_ACTIVE notification (if kid).

### `POST /holds/{id}/release`
Unknown → 404; other user → 409 `RESERVATION_NOT_PAYABLE` (`notPayable` +1); HELD → RELEASED + immediate return +
HOLD_CLEARED; not HELD → 200 with current status, nothing changes.

### `POST /holds/{id}/checkout`
Locks reservation; user mismatch / not HELD / now ≥ `hold_expires_at` → 409. Existing payment (any status) is
returned as `{orderId, amount}`; otherwise new REQUESTED payment, amount = Σ seat prices.

### `POST /payments/confirm`
Spec §6.6 ③ exactly; unknown or non-UUID `orderId` → 409 (D-020); any PG 4xx → 402 (D-021); concurrent recovery
resolution → answer from current state (D-038). `notPayable` +1 per 409; `declines` +1 per 402 caused by a PG
decline; `confirms` +1 on CONFIRMED.

### `POST /holds/{id}/deposit`
Checks as checkout. Effects §6.7 + D-019. Then COMPLETED notification and kid revoked (D-008). `depositsRequested` +1.

### `POST /deposits/{id}/pay`
PENDING_DEPOSIT ∧ now < `deposit_deadline` ∧ user ∧ `amount` = Σ seat prices → CONFIRMED, seats SOLD,
`depositsPaid` +1. Otherwise 409 `DEPOSIT_NOT_ACCEPTABLE`; unknown → 404 `RESERVATION_NOT_FOUND`.

### `POST /reservations/{id}/cancel`
§6.9 + D-022. `userCancels` +1 on success.

### `GET /reservations/{id}`
`{ reservationId, seatId, label, grade, seats:[…], userId, status, holdExpiresAt, paymentMethod, depositDeadline,
payment: {orderId, status} | null }`.

### `GET /seats` (§6.4)
`{ rows, cols, grades:[{name,rows,price}], seats:[{id,label,grade,status}], availableSeats, heldSeats,
pendingDepositSeats, returnPendingSeats, soldOut, releaseAt, saleEndAt }`.

### `POST /admin/reset`
Fields and validation: §6.14 + D-027. `anchorAt`, `saleEndAt` ISO instants (D-001). Response
`{ config: { rows, cols, grades, strategy, dbBackstop, holdTtlSec, confirmDeadlineSec, maxSeatsPerUser,
depositDeadlineSec, returnDelaySec, saleDurationSec } }`. Truncate with identity restart (D-017).

### `GET /admin/stats`
§6.14 shape; `config` as above; all state keys always present; `releaseBatches.totalReleased` = number of batches
with `released = true` that were opened by the reopen scheduler or sale end.

### `GET /admin/snapshot`
§6.14 shape; table arrays keyed `seats, reservations, reservation_seats, payments, release_batches` with snake_case
column keys (D-039).

## Outbound calls

| Target | Call | Timeouts | Retry |
|---|---|---|---|
| mock-pg | `POST /payments/confirm`, `GET /payments/{key}`, `POST /payments/{key}/cancel` | connect 1000 ms, request 3000 ms (config) | none (§10) |
| queue | `POST /internal/slots/{kid}/events` `{type, at}` + `X-Internal-Secret` | 1000/2000 ms (D-023) | 1 s, 2 s, 4 s then drop |
| queue | `POST /internal/sale-state` `{soldOut, releaseAt}` | 1000/2000 ms | none (D-024) |
