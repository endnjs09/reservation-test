# Quickstart & Validation Guide: reservation-test

Validation scenarios proving the three services work end to end. Endpoint details:
[contracts/](./contracts/); data: [data-model.md](./data-model.md).

## Prerequisites
- Linux / WSL2 with Java 21 (`java -version` → 21.x) and Docker with Compose v2.
- Ports 5432, 8080, 8081, 8082 free.
- `curl` and `jq` for the scenarios below.

## Build and test
```bash
./gradlew build          # compiles all modules and runs all tests (server tests start postgres:17 via Testcontainers)
./gradlew :server:test   # or :queue:test, :mock-pg:test
```

## Start (order from spec §2.1)
```bash
docker compose up -d                      # postgres:17, db/user/password = reservation
./gradlew :mock-pg:bootRun                # terminal 1 → :8081
./gradlew :queue:bootRun                  # terminal 2 → :8082
./gradlew :server:bootRun                 # terminal 3 → :8080 (Flyway migrates on start)
curl -s localhost:8081/actuator/health localhost:8082/actuator/health localhost:8080/actuator/health
```
Expected: three `{"status":"UP"}`.

## Scenario 1 — synchronized reset
```bash
ANCHOR=2026-11-01T11:00:00.000Z; END=2026-11-01T11:20:00.000Z; RUN=run-1
curl -s -XPOST localhost:8081/admin/reset -H 'Content-Type: application/json' -d "{\"anchorAt\":\"$ANCHOR\"}"
curl -s -XPUT  localhost:8081/admin/config -H 'Content-Type: application/json' -d '{"authFailureRate":0,"declineRate":0,"timeoutRate":0,"seed":42}'
curl -s -XPOST localhost:8082/admin/reset -H 'Content-Type: application/json' -d "{\"anchorAt\":\"$ANCHOR\",\"saleEndAt\":\"$END\",\"runEpoch\":\"$RUN\",\"maxActive\":2,\"admitPerSec\":1}"
curl -s -XPOST localhost:8080/admin/reset -H 'Content-Type: application/json' -d "{\"anchorAt\":\"$ANCHOR\",\"saleEndAt\":\"$END\",\"runEpoch\":\"$RUN\"}"
curl -s localhost:8080/admin/clock
```
Expected: `now` ≈ `2026-11-01T11:00:0x.xxxZ` on all three; server stats show 100 AVAILABLE seats.

## Scenario 2 — queue → hold → card payment → CONFIRMED
```bash
T=$(curl -s -XPOST localhost:8082/queue/enter -H 'Content-Type: application/json' -d '{"userId":"u-1"}' | jq -r .token)
# poll until ADMITTED (≤ 1–2 s)
KEY=$(curl -s "localhost:8082/queue/status?token=$T" | jq -r .admissionKey)
curl -s localhost:8080/seats -H "X-Admission-Key: $KEY" | jq '{availableSeats,soldOut,saleEndAt}'
R=$(curl -s -XPOST localhost:8080/holds -H "X-Admission-Key: $KEY" -H 'Idempotency-Key: k1' \
     -H 'Content-Type: application/json' -d '{"userId":"u-1","seatIds":[38,37]}')
echo "$R"; RID=$(echo "$R" | jq .reservationId)
O=$(curl -s -XPOST localhost:8080/holds/$RID/checkout -H "X-Admission-Key: $KEY" -H 'Content-Type: application/json' -d '{"userId":"u-1"}')
PK=$(curl -s -XPOST localhost:8081/auth -H 'Content-Type: application/json' -d "$O" | jq -r .paymentKey)
curl -s -XPOST localhost:8080/payments/confirm -H "X-Admission-Key: $KEY" -H 'Content-Type: application/json' \
  -d "{\"userId\":\"u-1\",\"orderId\":$(echo "$O"|jq .orderId),\"paymentKey\":\"$PK\",\"amount\":$(echo "$O"|jq .amount)}"
curl -s "localhost:8082/queue/status?token=$T"
```
Expected: hold 201 with seats `[37,38]` ascending, `totalPrice` 180000, `status` HELD; checkout `{orderId, amount:180000}`;
confirm 200 `{"status":"CONFIRMED"}`; queue token becomes COMPLETED; repeating the same hold request with key `k1`
returns the identical 201 body; a new hold with the same `X-Admission-Key` → 403 `KEY_REVOKED`.

## Scenario 3 — conflicts and all-or-nothing
Second user holds `[38,39]` → 409 `SEAT_UNAVAILABLE` with `unavailableSeatIds:[38]`; seat 39 remains AVAILABLE.
Same `Idempotency-Key` with different seats → 422.

## Scenario 4 — expiry via clock
Hold a seat, then `POST /admin/clock/advance {"seconds":421}` on the server. Within ~1 s the reservation is EXPIRED,
seats AVAILABLE, `counters.expiredByScheduler` = 1, and the queue slot receives HOLD_CLEARED. Checkout on it → 409.

## Scenario 5 — pending deposit and return batch
Hold → `POST /holds/{id}/deposit` → seats PENDING_DEPOSIT. Advance server clock 61 s → reservation DEPOSIT_EXPIRED,
seats RETURN_PENDING, `GET /seats` shows `releaseAt`. Advance 30 s → seats AVAILABLE, `reopenCount` 1.
Paying before the deadline with the right amount → 200 CONFIRMED instead.

## Scenario 6 — payment failure paths (mock-pg config)
- `declineRate: 1` → confirm 402 `PAYMENT_DECLINED`, seats AVAILABLE.
- `timeoutRate: 1` → confirm returns 200/402/202 depending on the stored outcome (query + reversal); with 202,
  after advancing the server clock 30 s the recovery scheduler resolves it. In every case mock-pg `/admin/payments`
  DONE entries match CONFIRMED card reservations (I3/I12).

## Scenario 7 — sale end
Advance all three clocks past `saleEndAt`: `GET /seats` and new holds → 409 `SALE_ENDED`; within ~1 s no HELD
reservations remain; queue stats show 0 WAITING/ADMITTED; `enter` returns a CLOSED token with reason `SALE_ENDED`.

## Scenario 8 — concurrency per strategy
For each `strategy` × `dbBackstop` reset, fire ~200 parallel holds on overlapping seats (automated in the server
integration tests). Snapshot checks: I1, I2, I4, I5, I10 hold for every combination except `naive` + backstop off,
where duplicate active `reservation_seats` per seat may appear (by design, §5.2).

## Invariant check
`GET :8080/admin/snapshot`, `GET :8082/admin/snapshot`, `GET :8081/admin/payments` provide everything needed for
I1–I17 (data-model.md §7).
