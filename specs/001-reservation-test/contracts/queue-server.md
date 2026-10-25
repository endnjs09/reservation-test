# Contract: Queue server (:8082)

Normative source: spec §7. Open details fixed by D-028 … D-033 (`docs/decisions.md`).
Errors `{ "code", "message" }`; times per D-001.

## User API

| Method & path | Body / query | Success | Errors |
|---|---|---|---|
| `POST /queue/enter` | `{userId}` (1–64) | 200 `{token, status, position, pollAfterMs, reason}` | 400 VALIDATION_FAILED |
| `GET /queue/status?token=` | query | 200 (shape by status, below) | 400 VALIDATION_FAILED (missing), 400 INVALID_TOKEN (unknown/malformed) |
| `POST /queue/leave` | `{token}` | 200 `{left:true}` | 400 VALIDATION_FAILED (missing), 400 INVALID_TOKEN |

`enter`: returns the user's WAITING/ADMITTED token if any; otherwise creates a token: CLOSED(`SALE_ENDED`) if the
sale has ended, CLOSED(`SOLD_OUT`) if `soldOut ∧ closeQueueOnSoldOut`, else WAITING with position = current WAITING
count (D-028). `position`/`pollAfterMs` null unless WAITING; `reason` only for CLOSED (D-031).

`status` shapes:
- WAITING `{ status, position, pollAfterMs }` — pollAfterMs: ≤50→1000, ≤300→2000, ≤1000→4000, else 6000
- ADMITTED `{ status, admissionKey, admissionExpiresAt }`
- other `{ status, reason }` — `reason` ∈ `SALE_ENDED | SOLD_OUT | null`
No seat-availability fields anywhere.

## Internal API (header `X-Internal-Secret`, else 401 `UNAUTHORIZED`)

| Method & path | Body | Success |
|---|---|---|
| `POST /internal/slots/{kid}/events` | `{type: HOLD_ACTIVE\|HOLD_CLEARED\|COMPLETED, at}` | 200 `{ignored:false}` applied / `{ignored:true}` |
| `POST /internal/sale-state` | `{soldOut, releaseAt}` | 200 `{}` |

Ignore rules (D-033): unknown kid, slot not ADMITTED, `at` < `lastEventAt`. Bad `type`/`at` → 400.
`sale-state`: `soldOut` true ∧ `closeQueueOnSoldOut` → all WAITING → CLOSED(SOLD_OUT) immediately; later enters
CLOSED until `soldOut=false` arrives.

## Admission key (§7.3)
`v1.<b64url(payload)>.<b64url(HMAC-SHA256(secret, "v1." + b64url(payload)))>`, payload
`{"kid","uid","iat","exp","run"}` (epoch ms, server clock; `exp` = slot `expiresAt`).

## Admission run (`admission-interval-ms`, real time) — D-029
1. Expire slots: not busy ∧ now ≥ expiresAt → EXPIRED; now ≥ expiresAt + busyMaxExtraSec → EXPIRED(BUSY_CAP),
   `busyCapExpirations` += `{kid, userId, at}`.
2. If now ≥ saleEndAt (non-null): all WAITING/ADMITTED → CLOSED(SALE_ENDED), slots released.
3. Admit `min(maxActive − slots, floor(admitPerSec × intervalMs/1000))` WAITING in seq order;
   `expiresAt = now + admissionTtlSec`.
4. Recompute positions; update `activeMax`, `admittedPerTickMax`; log run if admitted > 0.

## Admin API

| Method & path | Body | Response |
|---|---|---|
| `POST /admin/reset` | `{anchorAt, maxActive, admitPerSec, admissionTtlSec, busyMaxExtraSec, saleEndAt, closeQueueOnSoldOut, runEpoch}` all optional (D-030 validation; missing runEpoch → random) | 200 `{config}` |
| `GET /admin/stats` | — | §7.7 shape |
| `GET /admin/snapshot` | — | `{serverTime, config, tokens, slots, admissions, activeMax, admittedPerTickMax, busyCapExpirations}` |
| `GET /admin/clock`, `POST /admin/clock/advance` | §3.2 | `{now}` |

`config` = `{ maxActive, admitPerSec, admissionTtlSec, busyMaxExtraSec, saleEndAt, closeQueueOnSoldOut, runEpoch,
admissionIntervalMs }`.
