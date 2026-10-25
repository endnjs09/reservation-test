# Contract: Mock payment gateway (:8081)

Normative source: spec §8. Open details fixed by D-034 … D-037. Errors `{ "code", "message" }`; 400 code is
`INVALID_REQUEST` for malformed input; times per D-001.

| Method & path | Body | Success | Errors |
|---|---|---|---|
| `POST /auth` | `{orderId, amount}` | 200 `{paymentKey}` (new AUTHORIZED payment, `pk_<uuid>`) | 400 AUTH_FAILED (p = authFailureRate), 400 INVALID_REQUEST |
| `POST /payments/confirm` | `{paymentKey, orderId, amount}` | 200 `{status:"DONE", approvedAt}` | 400 INSUFFICIENT_FUNDS (DECLINED), 400 INVALID_REQUEST, 504 TIMEOUT after hold |
| `GET /payments/{key}` | — | 200 `{status}` | 404 NOT_FOUND |
| `POST /payments/{key}/cancel` | — | 200 `{status}` (DONE/AUTHORIZED → CANCELED; others unchanged) | 404 NOT_FOUND |
| `GET /admin/config` | — | 200 config | — |
| `PUT /admin/config` | partial config (D-036) | 200 config | 400 INVALID_REQUEST (rates ∉ [0,1], not 0 ≤ min ≤ max) |
| `POST /admin/reset` | `{anchorAt?}` | 200 `{}` (keeps config, reseeds) | 400 |
| `GET /admin/stats` | — | `{serverTime, authRequests, authFailed, confirmRequests, done, declined, timedOut, canceled, confirmInflight, confirmLatency:{p50,p95,p99}}` | — |
| `GET /admin/payments` | — | `[{paymentKey, orderId, amount, status}]` | — |
| `GET /admin/clock`, `POST /admin/clock/advance` | §3.2 | `{now}` | 400 |

Config: `{ authFailureRate: 0.03, declineRate: 0.03, timeoutRate: 0, confirmMinMs: 100, confirmMaxMs: 500, seed: null }`.

## Confirm processing (D-035)
1. Draw delay U(min,max), timeout flag, decline flag (always 3 draws, in this order).
2. Validate: key exists, `orderId`/`amount` match, status AUTHORIZED and not processing → else 400 INVALID_REQUEST.
   Mark processing.
3. Sleep delay (real time).
4. Timeout → 4th draw: 50% DONE (stored) else stays AUTHORIZED; respond 504 TIMEOUT at arrival + confirmMaxMs + 10 s.
5. Decline → DECLINED, 400 INSUFFICIENT_FUNDS.
6. Else DONE, 200.
