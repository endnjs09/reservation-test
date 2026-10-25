# Data Model: reservation-test

Spec references: §4 (domain), §5 (tables), §7.1 (queue state), §8 (mock-pg). Decisions: `docs/decisions.md`.

## 1. Reservation server — PostgreSQL schema (`V1__init.sql`)

All timestamps are `timestamptz`, always written from the server clock (never `now()` in SQL).
Status columns use `CHECK` constraints listing exactly the §4.2 values.

### seats
| Column | Type | Constraints |
|---|---|---|
| id | bigint | PK (= row_index × cols + col_index + 1) |
| label | varchar(16) | NOT NULL, UNIQUE (`A1`) |
| row_index, col_index | int | NOT NULL, ≥ 0 |
| grade | varchar(8) | NOT NULL, CHECK in (VIP,S,A,B) |
| price | bigint | NOT NULL, ≥ 0 |
| status | varchar(16) | NOT NULL, CHECK in (AVAILABLE, HELD, PENDING_DEPOSIT, RETURN_PENDING, SOLD) |
| current_reservation_id | bigint | NULL (no FK, so naive mode can overwrite freely) |
| release_batch_id | bigint | NULL |
| version | bigint | NOT NULL DEFAULT 0, +1 on every change |
| updated_at | timestamptz | NOT NULL |

Indexes: `(status)`, `(release_batch_id)`.

### reservations
| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| seat_id | bigint | NOT NULL (smallest seat id) |
| user_id | varchar(64) | NOT NULL |
| status | varchar(16) | CHECK in the 9 states |
| seat_count | int | NOT NULL ≥ 1 |
| payment_method | varchar(8) | NULL, CHECK in (CARD, DEPOSIT) |
| hold_expires_at | timestamptz | NOT NULL |
| confirm_deadline | timestamptz | NULL |
| deposit_deadline | timestamptz | NULL |
| idempotency_key | varchar(64) | NOT NULL UNIQUE |
| admission_kid | varchar(64) | NULL |
| created_at, updated_at | timestamptz | NOT NULL |

Indexes: `(user_id, status)`, `(status, hold_expires_at)`, `(status, confirm_deadline)`, `(status, deposit_deadline)`.
Backstop (D-015, toggled): `ux_res_user_active ON reservations(user_id) WHERE status IN ('HELD','CONFIRMING')`.

### reservation_seats (source of truth for a reservation's seats)
| Column | Type | Constraints |
|---|---|---|
| reservation_id | bigint | PK part, FK → reservations |
| seat_id | bigint | PK part, FK → seats |
| released_at | timestamptz | NULL while held |

Index: `(seat_id) WHERE released_at IS NULL` (non-unique, lookup).
Backstop (D-015, toggled): `ux_rs_active_seat ON reservation_seats(seat_id) WHERE released_at IS NULL`.

### payments
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK (= orderId) |
| reservation_id | bigint | NOT NULL UNIQUE, FK |
| amount | bigint | NOT NULL |
| status | varchar(16) | CHECK in (REQUESTED, APPROVED, FAILED, CANCELED) |
| payment_key | varchar(128) | NULL until confirm |
| fail_reason | varchar(128) | NULL |
| created_at, updated_at | timestamptz | NOT NULL |

### release_batches
| Column | Type | Constraints |
|---|---|---|
| id | bigserial | PK |
| release_at | timestamptz | NOT NULL |
| released | boolean | NOT NULL DEFAULT false |
| released_at | timestamptz | NULL |
| seat_count | int | NOT NULL DEFAULT 0 |

`ux_one_open_batch ON release_batches((true)) WHERE NOT released` — at most one open batch (always on).

### sale_period (D-016; not in snapshot)
| Column | Type |
|---|---|
| id | int PK CHECK (id = 1) |
| start_at, end_at | timestamptz NOT NULL |
| end_processed | boolean NOT NULL DEFAULT false |

## 2. Reservation server — in-memory state

| Item | Content | Cleared on |
|---|---|---|
| RuntimeConfig | §6.1 values changeable by reset (strategy, dbBackstop, TTLs, seats, grades, maxSeatsPerUser, saleDurationSec) + `runEpoch` | reset (fields overwritten), restart (reload from YAML, D-018) |
| Sale period cache | `saleEndAt`, `endProcessed` mirrored from `sale_period` | reset / startup load |
| Revoked kids | `Set<String>` | reset |
| Counters | §6.13 names, `AtomicLong` each | reset |
| Dropped notifications | list of `{kid, type, at}` | reset |
| Notification generation | int, bumps on reset (D-023) | — |
| Last sent sale-state | `soldOut`, time sent (D-024) | reset |

## 3. Reservation state machine (§4.3)

```
            hold ok
 (none) ───────────▶ HELD ──release──────────────▶ RELEASED      seats → AVAILABLE (immediate)
                      │ ├─expiry sched / sale end─▶ EXPIRED       seats → AVAILABLE (immediate)
                      │ ├─choose deposit──────────▶ PENDING_DEPOSIT  seats → PENDING_DEPOSIT, COMPLETED
                      │ └─confirm (checks pass)──▶ CONFIRMING
                      │                               ├─PG DONE─────────────▶ CONFIRMED  seats → SOLD, COMPLETED
                      │                               └─decline/canceled/404▶ PAYMENT_FAILED seats → AVAILABLE
 PENDING_DEPOSIT ─pay─▶ CONFIRMED (seats SOLD)
 PENDING_DEPOSIT ─deadline─▶ DEPOSIT_EXPIRED (seats RETURN_PENDING in open batch; AVAILABLE if sale end processed, D-040)
 PENDING_DEPOSIT ─user cancel─▶ CANCELED (seats AVAILABLE)
 CONFIRMED ─user cancel (card: PG cancel ok; deposit: always)─▶ CANCELED (seats AVAILABLE)
```

Every transition is a conditional update `UPDATE reservations SET status=:to … WHERE id=:id AND status=:from`
under a row lock; 0 rows → "any other state → unchanged, log only".

**Seat side effects** (always for all seats of the reservation from `reservation_seats` where `released_at IS NULL`):
- *Immediate return*: `UPDATE seats SET status='AVAILABLE', current_reservation_id=NULL, release_batch_id=NULL,
  version=version+1, updated_at=:now WHERE id = ANY(:ids) AND current_reservation_id = :rid` (conditional, §4.3),
  then `reservation_seats.released_at = :now`, `immediateReturns += n`.
- *To SOLD / PENDING_DEPOSIT*: same condition `current_reservation_id = :rid`.
- *To RETURN_PENDING*: condition as above; `release_batch_id = :batch`; `released_at` is recorded on
  `reservation_seats` when the reservation ends (DEPOSIT_EXPIRED); `current_reservation_id` is kept until reopen
  (owner = the DEPOSIT_EXPIRED reservation; I5 checks RETURN_PENDING ⇔ open batch).
- *Reopen*: `UPDATE seats SET status='AVAILABLE', current_reservation_id=NULL, release_batch_id=NULL …
  WHERE release_batch_id=:b AND status='RETURN_PENDING'`.

Payment states: `REQUESTED → APPROVED | FAILED | CANCELED`; `APPROVED → CANCELED` (user cancel);
`REQUESTED → CANCELED` (D-019 deposit chosen; reversal success).

## 4. Hold acquisition per strategy (§6.5; D-010 … D-014)

| Strategy | Statement(s) | Per-user lock | Failure detection |
|---|---|---|---|
| conditional | one `UPDATE … WHERE id IN (SELECT … ORDER BY id FOR UPDATE) AND status='AVAILABLE' RETURNING id` | yes | returned count < requested |
| pessimistic | `SELECT id,status FROM seats WHERE id=ANY(?) ORDER BY id FOR UPDATE` → `UPDATE` | yes | any non-AVAILABLE |
| optimistic | `SELECT id,status,version` (no lock) → per seat asc `UPDATE … WHERE id=? AND version=? AND status='AVAILABLE'` | yes | any 0-row update / non-AVAILABLE read |
| naive | `SELECT status` (no lock) → `UPDATE seats SET status='HELD' … WHERE id=?` (no condition) | **no** | non-AVAILABLE read; backstop violation (D-014) |

Then (all): sale end re-check → insert `reservation_seats` rows → set `seats.current_reservation_id`.
The reservation row is inserted before acquisition in the same transaction (D-011) so its id is available.

## 5. Queue server — in-memory model (§7.1)

**Token** `{ tokenId (UUID), userId, seq (long, from 1 per reset), status, createdAt, admittedAt, endedAt,
endReason, position (cached) }`
Status transitions:
```
WAITING ──admission run──▶ ADMITTED ──COMPLETED event──▶ COMPLETED
   │                          ├──leave──────────────────▶ LEFT
   │                          ├──expiry (not busy) / busy cap──▶ EXPIRED (endReason EXPIRED | BUSY_CAP)
   │                          └──sale end──────────────▶ CLOSED (SALE_ENDED)
   ├──leave──▶ LEFT
   └──sale end / sold-out (closeQueueOnSoldOut)──▶ CLOSED (SALE_ENDED | SOLD_OUT)
(enter while sale ended / sold-out-closed) ──▶ new token created directly as CLOSED
```

**Slot** (per ADMITTED token) `{ keyId (=kid), userId, tokenId, admittedAt, expiresAt, busy, busySince,
lastEventAt }`. Removed when the token leaves ADMITTED.

**User index**: `userId → token` for WAITING/ADMITTED only.

**Admission log**: `{ tickAt, admitted, seqFrom, seqTo, activeAfter }` (only runs with admitted > 0, D-029),
plus `activeMax`, `admittedPerTickMax`.

**Sale state**: `saleEndAt` (null until reset, D-032), `soldOut`, `releaseAt`, `saleEnded` flag.

**Busy cap log**: `busyCapExpirations` list `{ kid, userId, at }`.

**Counters**: `entered, admitted, keysIssued, expired, expiredByBusyCap, closedSoldOut, closedSaleEnded,
slotEvents{HOLD_ACTIVE, HOLD_CLEARED, COMPLETED, ignored}` (D-033).

**Config** (reset-changeable): `maxActive, admitPerSec, admissionTtlSec, busyMaxExtraSec, saleEndAt,
closeQueueOnSoldOut, runEpoch`; restart-only: `admissionIntervalMs`, secrets.

## 6. mock-pg — in-memory model (§8)

**Payment** `{ paymentKey ("pk_<uuid>"), orderId, amount, status, processing (bool), createdAt, approvedAt }`
```
AUTHORIZED ──confirm ok──▶ DONE ──cancel──▶ CANCELED
AUTHORIZED ──decline─────▶ DECLINED
AUTHORIZED ──cancel──────▶ CANCELED
AUTHORIZED ──timeout, 50% not approved──▶ AUTHORIZED (processing stays true → later confirm = INVALID_REQUEST)
```
**Config** `{ authFailureRate, declineRate, timeoutRate, confirmMinMs, confirmMaxMs, seed }` + `Random`.
**Stats** `{ authRequests, authFailed, confirmRequests, done, declined, timedOut, canceled, confirmInflight,
latency samples }`.

## 7. Invariant support

| Invariant | Data used |
|---|---|
| I1, I2, I5, I10, I11 | `seats`, `reservation_seats`, `reservations` in one snapshot |
| I3, I12 | snapshot `payments` + `reservations` vs mock-pg `/admin/payments` |
| I4, I6 | `reservations` status per user |
| I7 | `idempotency_key` unique; replay body D-009 |
| I8, I17 | queue `admissions`, `activeMax`, `admittedPerTickMax`, slots |
| I9 | mock-pg confirm count vs reservations rejected in Transaction A (`notPayable`) |
| I13 | `release_batches.release_at/released_at` |
| I14 | queue stats + reservations after sale end |
| I15 | queue `busyCapExpirations` vs server `droppedNotifications` / reservations |
| I16 | `counters.acceptedInvalidKeys` |
