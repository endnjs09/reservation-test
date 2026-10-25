# reservation-test 명세

티켓 예매 서비스의 백엔드 세 개를 만든다: **예약 서버**, **대기열 서버**, **모의 결제사(mock-pg)**.
티켓 오픈 때의 흐름 **대기열 → 좌석 선점(여러 장·연석) → 결제(카드 2단계 / 입금 대기) → 확정**, 그리고 **취소표**를 다룬다.

### 기술 스택
| 항목 | 선택 |
|---|---|
| 언어 / 런타임 | Java 21 |
| 프레임워크 | Spring Boot 4.1.2 |
| 빌드 | Gradle (Wrapper 사용), 멀티 모듈 `server`, `queue`, `mock-pg` |
| 공통 의존성 (세 모듈) | Spring Web, Validation, Actuator (`/actuator/health`로 기동 확인) |
| 예약 서버 DB | PostgreSQL 17 (Docker Compose로 실행) |
| 예약 서버 DB 접근 | JdbcTemplate. JPA는 쓰지 않는다 |
| 스키마 마이그레이션 | Flyway |
| 예약 서버 테스트의 DB | Testcontainers (PostgreSQL) |
| 그외 | Lombok |
| 쓰지 않는 것 | Spring Boot Docker Compose 지원, Spring Security |

---

## 1. 범위와 원칙

- 정확성: 9장의 불변식을 항상(또는 명시된 시점에) 만족해야 한다.
- 범위 밖: 인증·인가, 실제 결제, 배포. 사용자는 `userId` 문자열로만 구분한다.
- 좌석 수, 정원, TTL, 실패율 등 모든 수치는 설정값이다. 대부분은 `/admin/reset`으로 실행 중에 바꿀 수 있다 (커넥션 풀 크기처럼 재시작이 필요한 값은 제외).
- 세 서버는 서로의 코드에 의존하지 않는다. HTTP로만 통신한다.
- 10장 "구현하지 말 것"은 구현하지 않는다.

---

## 2. 구조

### 2.1 모듈과 포트
```
reservation-test/
├── settings.gradle          # include 'server', 'queue', 'mock-pg'
├── build.gradle             # 공통 (Java 21)
├── docker-compose.yml       # postgres:17
├── server/    :8080  예약 서버
├── queue/     :8082  대기열 서버   (DB 없음, 메모리 기반 단일 인스턴스)
└── mock-pg/   :8081  모의 결제사   (DB 없음, 메모리 기반)
```
- 실행 순서: `docker compose up -d` → mock-pg → queue → server.
- DB 이름 `reservation`. 접속 정보는 `docker-compose.yml`과 예약 서버 설정에 같은 값으로 둔다.

### 2.2 요청 경로
```
사용자 ──① 진입·순번 조회──▶ 대기열 서버          순번, 입장키 발급
   │◀──────── 입장키 ──────────┘
   │
   ├──② 입장키 + 요청─────▶ 예약 서버 ──▶ DB        입장키 서명·만료만 검사 (대기열 서버 호출 없음)
   │                          ├──▶ 대기열 서버       자리 상태 알림 (선점 중 / 해제 / 완료)
   │                          └──▶ 결제사            승인·조회·취소 요청
   │
   └──③ 카드 인증──────────▶ 결제사                사용자가 직접
```
- 대기 중인 사용자의 순번 조회는 예약 서버에 **한 건도 닿지 않는다**.
- 입장 이후 요청은 사용자 → 예약 서버로 직접 간다. 예약 서버는 요청마다 대기열 서버를 부르지 않는다.

---

## 3. 시간

### 3.1 서버 시계
- 세 서버는 각자 하나의 **서버 시계**를 가진다. 동작 판단에 쓰는 "현재 시각"은 전부 이 시계에서 얻는다. 시스템 벽시계를 직접 읽지 않고, DB 쿼리의 시각 비교에도 DB의 현재 시각 함수 대신 서버 시계 값을 넘긴다.
- 서버 시계 = `기준 시각 + 기준을 잡은 뒤의 단조 경과 + 이동량`.
  - 기동 직후: 기준 시각 = 기동 시점의 시스템 시각, 이동량 0.
  - `/admin/reset`에 `anchorAt`이 오면: 기준 시각 = `anchorAt`, 단조 경과는 reset 요청을 받은 순간부터 다시 센다, 이동량 0.
  - `anchorAt` 없이 reset하면: 기준 시각 = 그 순간의 시스템 시각, 이동량 0.
  - 시계 이동 API(3.2)를 부르면 이동량에 더한다.
- 단조 경과를 쓰므로 시스템 벽시계가 바뀌어도 서버 시계는 영향받지 않는다.

### 3.2 시계 API (세 서버 공통)
- **`GET /admin/clock`** → 200 `{ "now" }`
- **`POST /admin/clock/advance`** `{ "seconds": 30 }` → 서버 시계를 그만큼 앞으로 옮긴다. `seconds`는 0보다 큰 수(소수 허용). 0 이하·누락은 400 `VALIDATION_FAILED`. 200 `{ "now" }`.
  - 시계를 뒤로 돌리는 API는 없다.
  - 옮긴 뒤의 판단(만료·기한·판매 종료 등)은 요청 처리 때와 각 스케줄러의 다음 주기에 반영된다.
- 세 서버의 시계는 서로 독립이다. 세 서버를 같은 시각으로 맞추려면 호출하는 쪽이 세 서버에 같은 `anchorAt`·같은 이동량을 보낸다.

### 3.3 서버 시계를 쓰지 않는 것
아래는 실제 시간(단조 시계)으로 동작한다. 시계 이동의 영향을 받지 않는다.
- 스케줄러 주기
- 결제사 승인 지연과 타임아웃 대기, 예약 서버의 결제사 호출 타임아웃
- 대기열 서버로의 알림 재시도 간격 (6.11)

### 3.4 표기
- 설정의 시간 길이 값(`...Sec`)은 서버 시계 기준 초다.
- API의 시각은 ISO-8601 UTC 문자열이다.

---

## 4. 도메인

### 4.1 좌석과 등급
- 행 `A`부터 `rows`개, 열 `1`부터 `cols`개. 기본 10 × 10 = 100석.
- 좌석 `id = 행 인덱스 × cols + 열 인덱스 + 1` (행·열 인덱스는 0부터), `label`은 `"A1"` 형식.
- 등급은 앞줄부터 행 단위로 정한다. 설정 `grades: [{ name, rows, price }]`, 행 수의 합은 `rows`와 같아야 한다.
- 등급 이름은 `VIP`, `S`, `A`, `B` 중에서 쓴다.

| 등급 | 행 수 | 기본 행 | 좌석 | 가격 |
|---|---|---|---|---|
| VIP | 1 | A | 10 | 150000 |
| S | 2 | B~C | 20 | 120000 |
| A | 3 | D~F | 30 | 90000 |
| B | 4 | G~J | 40 | 60000 |

### 4.2 상태값
| 대상 | 값 |
|---|---|
| 좌석 | `AVAILABLE`, `HELD`, `PENDING_DEPOSIT`, `RETURN_PENDING`, `SOLD` |
| 예약 | `HELD`, `CONFIRMING`, `CONFIRMED`, `EXPIRED`, `RELEASED`, `PAYMENT_FAILED`, `PENDING_DEPOSIT`, `DEPOSIT_EXPIRED`, `CANCELED` |
| 결제 (예약 서버) | `REQUESTED`, `APPROVED`, `FAILED`, `CANCELED` |
| 대기 토큰 (대기열 서버) | `WAITING`, `ADMITTED`, `COMPLETED`, `LEFT`, `EXPIRED`, `CLOSED` |

- 좌석 `HELD`는 그 좌석을 가진 예약이 `HELD` 또는 `CONFIRMING`인 동안 유지된다.
- 한 예약은 좌석 1~`maxSeatsPerUser`개를 가진다. 한 예약의 좌석은 **항상 같은 상태**다 (일부만 선점·일부만 반환하는 일은 없다).

### 4.3 예약 상태 전이
| 현재 | 사건 | 조건 | 다음 | 좌석·부수 효과 |
|---|---|---|---|---|
| (없음) | 선점 성공 | 요청한 좌석 전부 확보 | HELD | 좌석 → HELD |
| HELD | 선점 해제 | - | RELEASED | 좌석 즉시 반환 |
| HELD | 만료 스케줄러 | now ≥ 선점 만료 시각 | EXPIRED | 좌석 즉시 반환 |
| HELD | 승인 요청 | 6.6의 확인 통과 | CONFIRMING | 승인 기한 = now + `confirmDeadlineSec` |
| HELD | 승인 요청 | 만료 시각 지남 | (유지) | 409 `RESERVATION_NOT_PAYABLE`, 결제사 호출 없음 |
| HELD | 입금 대기 선택 | 만료 전 | PENDING_DEPOSIT | 좌석 → PENDING_DEPOSIT, 대기열 자리 COMPLETED 알림 |
| CONFIRMING | 결제사 승인 성공 (요청 안에서든 복구에서든) | - | CONFIRMED | 결제 APPROVED, 좌석 → SOLD, COMPLETED 알림 |
| CONFIRMING | 승인 거절 / 복구에서 취소 확정 | - | PAYMENT_FAILED | 결제 FAILED 또는 CANCELED, 좌석 즉시 반환 |
| PENDING_DEPOSIT | 입금 | 기한 전, 금액 일치 | CONFIRMED | 좌석 → SOLD |
| PENDING_DEPOSIT | 입금 마감 스케줄러 | now ≥ 입금 기한 | DEPOSIT_EXPIRED | 좌석 → RETURN_PENDING (취소표 배치에 들어감) |
| PENDING_DEPOSIT | 사용자 취소 | - | CANCELED | 좌석 즉시 반환 |
| CONFIRMED | 사용자 취소 | 카드면 결제사 취소 성공 | CANCELED | 좌석 즉시 반환 |
| (좌석) RETURN_PENDING | 취소표 오픈 | 배치 오픈 시각 도달 | - | 좌석 → AVAILABLE (배치 단위로 한꺼번에) |
| 그 외 | 모든 사건 | - | 유지 | 기록만 |

- **CONFIRMING 예약은 만료 스케줄러가 건드리지 않는다.** 승인 기한이 지나도 CONFIRMING으로 남고, 복구 스케줄러(6.6)가 결론을 낸다.
- **좌석 반환은 항상 조건부다**: 그 좌석의 현재 소유 예약이 반환하려는 예약일 때만 좌석을 되돌린다. 예약이 끝날 때 그 예약의 좌석 배정 행에 해제 시각(`released_at`)을 기록한다.
- "즉시 반환"은 좌석을 `AVAILABLE`로 되돌리고 현재 소유 예약을 비우는 것이다.

### 4.4 좌석이 돌아오는 경로
| 경로 | 방식 |
|---|---|
| 선점 만료, 선점 해제, 카드 결제 실패(거절·망취소), 판매 종료로 인한 선점 만료 | 즉시 |
| 사용자 예매 취소 (CONFIRMED, PENDING_DEPOSIT) | 즉시 |
| 입금 대기 미입금 | 반환 대기 → **배치로 한꺼번에** (6.8) |

### 4.5 정책
| 항목 | 정책 |
|---|---|
| 선점 유효시간 | `holdTtlSec` 기본 420 |
| 1인 1선점 | 사용자당 HELD 또는 CONFIRMING 예약 최대 1건 |
| 1인 1예매 | CONFIRMED 또는 PENDING_DEPOSIT 예약이 있으면 새 선점 불가 (409 `USER_ALREADY_PURCHASED`). 한 예매 최대 `maxSeatsPerUser`(기본 4)매 |
| 결제 주문 | 예약당 결제 1건. checkout을 다시 요청하면 기존 주문을 그대로 돌려준다 |
| 승인 직전 확인 | 예약이 HELD이고, 만료 전이고, 사용자·금액이 일치할 때만 결제사 승인을 호출한다 |
| 승인 결과 불명 | 결제사에 상태 조회 1회 → 그래도 불명이면 CONFIRMING 유지, 복구 스케줄러가 처리 |
| 선점 재요청 | `Idempotency-Key`로 멱등. 같은 키 + 같은 내용 → 처음과 같은 응답, 같은 키 + 다른 내용 → 422 |

---

## 5. 데이터 (예약 서버)

### 5.1 테이블
스키마 마이그레이션 도구를 쓰고, 최종 스키마는 아래와 같다. 컬럼 이름은 `/admin/snapshot`(6.14)에 그대로 드러난다.

**seats**
| 컬럼 | 설명 |
|---|---|
| `id` | 좌석 id (4.1의 공식), PK |
| `label` | `"A1"`, 유일 |
| `row_index`, `col_index` | 0부터 |
| `grade`, `price` | 등급, 가격 |
| `status` | 4.2의 좌석 상태 5개만 허용 |
| `current_reservation_id` | 좌석을 지금 가진 예약. 없으면 null |
| `release_batch_id` | RETURN_PENDING일 때 속한 취소표 배치. 아니면 null |
| `version` | 좌석이 바뀔 때마다 1 증가 (기본 0) |
| `updated_at` | |

**reservations**
| 컬럼 | 설명 |
|---|---|
| `id` | PK, 증가 |
| `seat_id` | 대표 좌석 (예약 좌석 중 가장 작은 id) |
| `user_id` | |
| `status` | 4.2의 예약 상태 9개만 허용 |
| `seat_count` | 좌석 수 |
| `payment_method` | `CARD` 또는 `DEPOSIT`, 정해지기 전에는 null |
| `hold_expires_at`, `confirm_deadline`, `deposit_deadline` | 선점 만료 시각, 승인 기한, 입금 기한 |
| `idempotency_key` | 유일 |
| `admission_kid` | 선점할 때 쓴 입장키의 `kid`. 입장키 검사를 끈 경우 null |
| `created_at`, `updated_at` | |

**reservation_seats** (예약의 좌석 목록. **좌석 목록의 기준은 이 테이블**이고 `reservations.seat_id`는 대표 좌석일 뿐이다)
| 컬럼 | 설명 |
|---|---|
| `reservation_id`, `seat_id` | 함께 PK |
| `released_at` | 예약이 끝나 좌석을 놓은 시각. 아직 가지고 있으면 null |

**payments**
| 컬럼 | 설명 |
|---|---|
| `id` | UUID, PK. 주문 번호(`orderId`)로 쓴다 |
| `reservation_id` | 유일 (예약당 결제 1건) |
| `amount` | 좌석 가격 합 |
| `status` | 4.2의 결제 상태 4개만 허용 |
| `payment_key` | 결제사가 발급한 키 |
| `fail_reason` | |
| `created_at`, `updated_at` | |

**release_batches** (취소표 배치)
| 컬럼 | 설명 |
|---|---|
| `id` | PK, 증가 |
| `release_at` | 오픈 예정 시각 |
| `released` | 오픈(또는 판매 종료로 정리)됐는지. 기본 false |
| `released_at` | 실제로 오픈한 시각 |
| `seat_count` | 배치에 들어온 좌석 수. 기본 0 |

- **열린 배치(`released = false`)는 동시에 최대 1개**다.
- 판매 기간(시작 시각, 종료 시각, 종료 처리를 마쳤는지)도 DB에 저장해서, 예약 서버를 재시작해도 판매 종료 시각이 유지되게 한다 (`/admin/reset` 전까지).

### 5.2 DB 보호 제약 (backstop)
설정 `dbBackstop`이 true면 DB에 아래 두 유일 제약을 두고, false면 없앤다. 서버 시작 때와 `/admin/reset` 때 설정에 맞춘다.
- 좌석당 `released_at`이 null인 `reservation_seats` 행은 최대 1개
- 사용자당 HELD 또는 CONFIRMING 예약은 최대 1개

제약 위반은 각각 409 `SEAT_UNAVAILABLE`, 409 `USER_ALREADY_HOLDING`으로 바꿔 응답한다.
`naive` 전략(6.5) + backstop false 조합에서는 같은 좌석이 동시에 여러 예약에 배정될 수 있다. 이 조합은 **보호 장치가 없을 때의 동작을 확인하기 위한 비교용**이다. 그 외 조합에서는 일어나면 안 된다.

### 5.3 커넥션 풀
- DB 커넥션 풀 최대 크기 기본 20. 설정 파일 값이며, 바꾸려면 재시작이 필요하다.

---

## 6. 예약 서버 (:8080)

### 6.1 설정
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
  required: true                 # false면 입장키 검사(6.3)를 하지 않는다
  key-secret: dev-only-change-me # 대기열 서버와 같은 값
queue-server.base-url: http://localhost:8082   # 자리 상태 알림(6.11) 대상
internal.secret: dev-internal-secret            # 대기열 서버와 같은 값
```
- `scheduler.*`는 각 스케줄러의 주기(실제 시간 ms)와 만료 스케줄러의 한 번 처리량이다: `expiry` = 선점 만료(6.13), `recovery` = 결제 복구(6.6), `deposit` = 입금 마감(6.8), `reopen` = 취소표 오픈(6.8).
- `strategy`, `db-backstop`, 시간 값, 좌석·등급, 결제 값은 실행 중 설정으로 들고 `/admin/reset`(6.14)으로 바꾼다.

### 6.2 공통
- 요청·응답은 JSON, UTF-8.
- 에러 응답 형식 `{ "code", "message" }`.

| code | HTTP | 의미 |
|---|---|---|
| VALIDATION_FAILED | 400 | 요청 형식 오류 |
| KEY_INVALID | 403 | 입장키 형식·서명·runEpoch·userId 불일치 |
| KEY_EXPIRED | 403 | 입장키 만료 (6.3의 예외 제외) |
| KEY_REVOKED | 403 | 구매를 끝낸 입장키 |
| SEAT_NOT_FOUND | 404 | 없는 좌석 |
| RESERVATION_NOT_FOUND | 404 | 없는 예약 |
| SEAT_UNAVAILABLE | 409 | 이미 선점·판매된 좌석 (`unavailableSeatIds` 포함) |
| USER_ALREADY_HOLDING | 409 | 이미 HELD/CONFIRMING 예약이 있음 |
| USER_ALREADY_PURCHASED | 409 | 이미 예매함 (CONFIRMED/PENDING_DEPOSIT) |
| RESERVATION_NOT_PAYABLE | 409 | HELD가 아님, 만료, 다른 사용자, 금액 불일치 |
| DEPOSIT_NOT_ACCEPTABLE | 409 | 입금 불가 (상태·기한·금액) |
| RESERVATION_NOT_CANCELABLE | 409 | 취소할 수 없는 상태 |
| SALE_ENDED | 409 | 판매 종료 (6.10) |
| PAYMENT_DECLINED | 402 | 결제사 승인 거절 |
| IDEMPOTENCY_KEY_REUSED | 422 | 같은 키, 다른 내용 |
| PG_UNAVAILABLE | 502 | 결제사 호출 실패 (checkout·취소) |

### 6.3 입장키 검사
- 대상: `GET /seats`, `POST /holds`, `POST /holds/{id}/*`, `POST /payments/confirm`.
  (`POST /deposits/{id}/pay`, `POST /reservations/{id}/cancel`, `GET /reservations/{id}`는 검사하지 않는다.)
- 헤더 `X-Admission-Key` (형식 7.3).

| 검사 | 실패 시 |
|---|---|
| 형식·서명 | 403 KEY_INVALID |
| `run` = 현재 runEpoch | 403 KEY_INVALID |
| `uid` = 요청 본문의 `userId` (`GET /seats`는 본문이 없으므로 생략) | 403 KEY_INVALID |
| `kid`가 회수 목록에 없음 | 403 KEY_REVOKED |
| now < `exp` | 아래 예외가 아니면 403 KEY_EXPIRED |

- **만료 예외 (결제 도중 보호)**: 키가 만료됐어도 그 사용자에게 HELD 또는 CONFIRMING 예약이 있으면, **그 예약에 대한** `/holds/{id}/checkout`, `/holds/{id}/deposit`, `/holds/{id}/release`, `/payments/confirm`은 통과시킨다. `GET /seats`와 새 `POST /holds`는 막는다.
- **회수 목록**: COMPLETED 알림(6.11)을 보낸 `kid`. 메모리에 두고 reset 때 비운다.
- 예약 서버는 입장키 검사를 위해 대기열 서버를 호출하지 않는다.
- 서명 비교는 비교 시간으로 정보가 새지 않는 방식(상수 시간 비교)으로 한다.
- **독립 재검증**: 검사 대상 요청이 2xx로 끝나면, 위 검사와 **따로 구현한 두 번째 검사**로 같은 규칙(서명·runEpoch·uid·만료 예외)을 다시 확인한다. 이 검사가 실패하면 `counters.acceptedInvalidKeys`를 1 늘린다 (불변식 I16). 응답은 바꾸지 않는다.
- `admission.required = false`면 입장키 검사와 독립 재검증을 모두 하지 않는다.

### 6.4 좌석 조회 `GET /seats`
```json
{ "rows": 10, "cols": 10,
  "grades": [ { "name": "VIP", "rows": 1, "price": 150000 } ],
  "seats": [ { "id": 1, "label": "A1", "grade": "VIP", "status": "AVAILABLE" } ],
  "availableSeats": 0, "heldSeats": 10, "pendingDepositSeats": 4, "returnPendingSeats": 2,
  "soldOut": false, "releaseAt": null, "saleEndAt": "2026-11-01T11:20:00Z" }
```
- 요약 필드 7개(`availableSeats`, `heldSeats`, `pendingDepositSeats`, `returnPendingSeats`, `soldOut`, `releaseAt`, `saleEndAt`)는 **항상** 있다.
- `soldOut` = AVAILABLE 좌석 0개이고 HELD 좌석 0개 (PENDING_DEPOSIT·RETURN_PENDING·SOLD만 남음).
- `releaseAt` = 열린 취소표 배치의 오픈 예정 시각, 없으면 null.
- 일관된 한 시점의 값일 필요는 없다 (약간 오래된 값 허용).
- 판매 종료 후에는 409 `SALE_ENDED` (6.10).

### 6.5 선점
**`POST /holds`** 헤더 `Idempotency-Key`(1~64자, 필수), `X-Admission-Key`
```json
{ "userId": "u-0001", "seatIds": [37, 38] }
```
- `userId`: 1~64자.
- `seatIds`: 1~`maxSeatsPerUser`개, 양수, 중복 금지. 대신 단일 `seatId` 필드도 받는다 (1개로 취급). 둘 다 있거나 둘 다 없으면 400.
- 형식 오류는 400 `VALIDATION_FAILED`. 없는 좌석이 하나라도 있으면 404 `SEAT_NOT_FOUND`.
- **전부 아니면 전무**: 하나라도 못 잡으면 아무것도 잡지 않고 409 `SEAT_UNAVAILABLE` + `unavailableSeatIds`(못 잡은 좌석 id 목록).
- **잠금 순서**: 여러 좌석은 항상 좌석 id 오름차순으로 잡는다 (교착 방지).
- 처리 순서 (한 트랜잭션)
  1. 입장키 검사(6.3), 판매 종료 확인(6.10).
  2. 멱등 확인: 같은 `Idempotency-Key`의 예약이 있으면, 내용(`userId`와 좌석 id 집합 — 순서 무관)이 같을 때 **처음 성공 응답과 같은 201 본문**을 돌려주고, 다르면 422 `IDEMPOTENCY_KEY_REUSED`. 새 예약은 만들지 않는다.
  3. 같은 사용자의 선점 요청은 서로 직렬화한다 (`naive` 전략은 하지 않는다).
  4. 사용자 확인: 예매가 있으면 409 `USER_ALREADY_PURCHASED`, HELD/CONFIRMING 예약이 있으면 409 `USER_ALREADY_HOLDING`.
  5. 좌석 확보 (전략별 아래 표). 실패하면 못 잡은 좌석마다 `counters.holdConflicts` +1.
  6. 판매 종료를 다시 확인한다. 잠금을 기다리는 동안 종료 시각이 지났으면 409 `SALE_ENDED`로 끝낸다 (좌석은 잡지 않은 상태로 되돌린다).
  7. 예약 생성 (HELD, 선점 만료 시각 = now + `holdTtlSec`, 입장키 `kid`), 좌석 배정 행, 좌석의 현재 소유 예약.
  8. 커밋 후 201, 그리고 HOLD_ACTIVE 알림 (6.11).
- 같은 키의 동시 요청이 함께 들어와도 예약은 1건만 생긴다. 나중 요청은 2번의 규칙대로 응답한다.
- 응답 201: `{ "reservationId", "seats": [{ "id", "label", "grade", "price" }], "totalPrice", "status": "HELD", "holdExpiresAt" }` (`seats`는 id 오름차순)

| 전략 | 좌석 확보 방법 |
|---|---|
| conditional | 상태가 AVAILABLE인 좌석만 HELD로 바꾸는 **조건부 갱신 한 번**으로 확인과 변경을 함께 한다. 바뀐 좌석 수가 요청 수와 다르면 실패 |
| pessimistic | 요청 좌석의 행을 id 순으로 **잠근 뒤** 전부 AVAILABLE일 때만 바꾼다 |
| optimistic | 좌석마다 읽어 둔 `version`과 같을 때만 바꾸는 갱신. 하나라도 실패하면 전체 실패. 재시도 없음 |
| naive | 잠금 없이 상태를 읽고, AVAILABLE이면 조건 없이 바꾼다. 사용자 직렬화(3번)도 하지 않는다. **비교용** (5.2) |

**`POST /holds/{id}/release`** `{ "userId" }`
- 없는 예약 404 `RESERVATION_NOT_FOUND`, 다른 사용자 409 `RESERVATION_NOT_PAYABLE`.
- HELD면 RELEASED + 좌석 즉시 반환 → 200 `{ "reservationId", "status": "RELEASED" }`.
- 이미 HELD가 아니면 아무것도 바꾸지 않고 200으로 현재 상태를 돌려준다 (멱등).

### 6.6 카드 결제 (2단계)
```
사용자 ──① checkout──▶ 예약 서버         주문(orderId, 금액) 생성
사용자 ──② 카드 인증──▶ 결제사           paymentKey 발급 (출금 없음)
사용자 ──③ confirm───▶ 예약 서버 ──④ 확인 후 승인──▶ 결제사   이때 출금
```
**① `POST /holds/{id}/checkout`** `{ "userId" }`
- 예약을 잠그고, 사용자 불일치·HELD 아님·만료면 409 `RESERVATION_NOT_PAYABLE`.
- 결제가 이미 있으면 그대로, 없으면 새로 만든다 (REQUESTED, 금액 = 좌석 가격 합). 결제사는 호출하지 않는다.
- 200 `{ "orderId", "amount" }`.

**② 카드 인증**: 사용자가 결제사 `POST /auth`를 직접 호출한다 (8장).

**③ `POST /payments/confirm`** `{ "userId", "orderId", "paymentKey", "amount" }`
1. (트랜잭션 A) 결제와 예약을 잠근다. 다음 중 하나면 409 `RESERVATION_NOT_PAYABLE`이고 **결제사 승인을 호출하지 않는다**: 사용자 불일치, 예약이 HELD 아님, 선점 만료 시각 지남, 금액 불일치, 결제가 REQUESTED 아님. 통과하면 예약 CONFIRMING, 승인 기한 기록, `payment_key`와 `payment_method = CARD` 저장. 커밋.
2. (트랜잭션 밖) 결제사 `POST /payments/confirm`.
3. (트랜잭션 B) 결과 반영
   - DONE → 결제 APPROVED, 예약 CONFIRMED, 좌석 SOLD(조건부), COMPLETED 알림 → **200** `{ "status": "CONFIRMED" }`
   - 4xx 거절 → 결제 FAILED, 예약 PAYMENT_FAILED, 좌석 즉시 반환 → **402** `PAYMENT_DECLINED`
   - 타임아웃·5xx·연결 실패 → 결제사 `GET /payments/{paymentKey}` 1회
     - DONE → 성공 처리 → 200
     - CANCELED → 결제 CANCELED, 예약 PAYMENT_FAILED, 좌석 반환 → 402
     - 404 → 결제 FAILED, 예약 PAYMENT_FAILED, 좌석 반환 → 402
     - AUTHORIZED / DECLINED → 망취소 `POST /payments/{key}/cancel`: 성공하면 결제 CANCELED, 예약 PAYMENT_FAILED, 좌석 반환 → 402 / 실패하면 CONFIRMING 유지 → **202** `{ "status": "CONFIRMING" }`
     - 조회 실패 → CONFIRMING 유지 → **202** `{ "status": "CONFIRMING" }` (사용자는 `GET /reservations/{id}`로 결과를 확인)
- 없는 주문(`orderId`)도 409 `RESERVATION_NOT_PAYABLE`이다.

**복구 스케줄러** (`recovery-interval-ms`): 대상 = CONFIRMING이고 승인 기한 ≤ now인 예약. 예약마다 **항상 결제사 상태 조회부터** 다시 한다.

| 조회 결과 | 처리 |
|---|---|
| DONE | 성공 처리 (결제 APPROVED, 예약 CONFIRMED, 좌석 SOLD, COMPLETED 알림과 kid 회수) |
| CANCELED / 404 | 결제 CANCELED(404면 FAILED), 예약 PAYMENT_FAILED, 좌석 반환, 필요하면 HOLD_CLEARED 알림 |
| AUTHORIZED / DECLINED | 망취소 → 성공하면 위와 같이 실패 처리, 실패하면 CONFIRMING 유지 |
| 조회 실패 | CONFIRMING 유지, 다음 주기에 다시 |
- 재시도 횟수 제한은 없다. 시도마다 `counters.recoveryAttempts` +1, 결론을 내면 `counters.recovered` +1.

### 6.7 입금 대기·입금
**`POST /holds/{id}/deposit`** `{ "userId" }`
- HELD이고 만료 전이며 사용자가 일치해야 한다. 아니면 409 `RESERVATION_NOT_PAYABLE`.
- 예약 PENDING_DEPOSIT, `payment_method = DEPOSIT`, 입금 기한 = now + `depositDeadlineSec`. 좌석 PENDING_DEPOSIT. COMPLETED 알림 (자리 반납).
- 200 `{ "status": "PENDING_DEPOSIT", "depositDeadline", "amount" }`. 가상계좌는 흉내만 낸다 (결제사 호출 없음).

**`POST /deposits/{reservationId}/pay`** `{ "userId", "amount" }` (은행 역할, 입장키 검사 없음)
- PENDING_DEPOSIT, now < 입금 기한, 사용자·금액 일치 → CONFIRMED, 좌석 SOLD → 200 `{ "status": "CONFIRMED" }`.
- 그 외 409 `DEPOSIT_NOT_ACCEPTABLE`. 없는 예약 404.

### 6.8 입금 마감 · 취소표 배치
- **입금 마감 스케줄러** (`deposit-interval-ms`): PENDING_DEPOSIT이고 입금 기한 ≤ now인 예약 → DEPOSIT_EXPIRED, 좌석 RETURN_PENDING, 배치 편입.
  - 열린 배치가 없으면 `release_at = now + returnDelaySec`인 배치를 새로 만들고, 있으면 그 배치에 합류한다 (첫 미입금 기준으로 카운트다운. 합류해도 `release_at`은 바뀌지 않는다).
  - 좌석의 `release_batch_id`를 그 배치로, 배치의 `seat_count`를 늘린다.
  - 판매 종료 처리(6.10)가 끝난 뒤의 미입금 좌석은 배치에 넣지 않고 즉시 AVAILABLE로 되돌린다.
- **취소표 오픈 스케줄러** (`reopen-interval-ms`): `release_at ≤ now`인 열린 배치마다 **한 트랜잭션에서** 그 배치의 RETURN_PENDING 좌석을 전부 AVAILABLE로 바꾸고 `release_batch_id`를 비운 뒤, 배치를 `released = true`, `released_at = now`로 닫는다. `counters.reopenCount` +1, `counters.reopenSeats` + 좌석 수.

### 6.9 예매 취소·조회
**`POST /reservations/{id}/cancel`** `{ "userId" }` (입장키 검사 없음)
- 사용자 불일치는 409 `RESERVATION_NOT_CANCELABLE`, 없는 예약 404.
- CONFIRMED + CARD: 결제사 `POST /payments/{key}/cancel` → 성공하면 결제 CANCELED, 예약 CANCELED, 좌석 즉시 반환 → 200 / 결제사 호출 실패면 502 `PG_UNAVAILABLE`, 상태 그대로.
- CONFIRMED + DEPOSIT, PENDING_DEPOSIT: 예약 CANCELED, 좌석 즉시 반환 → 200.
- 200 본문 `{ "reservationId", "status": "CANCELED" }`.
- 그 외 상태: 409 `RESERVATION_NOT_CANCELABLE`.

**`GET /reservations/{id}`** → `{ "reservationId", "seatId", "label", "grade", "seats": [{ "id", "label", "grade", "price" }], "userId", "status", "holdExpiresAt", "paymentMethod", "depositDeadline", "payment": { "orderId", "status" } | null }` (`seatId`·`label`·`grade`는 대표 좌석). 없는 예약 404.

### 6.10 판매 종료
- 판매 기간: `/admin/reset` 때 정한다 (6.14). 저장된 판매 기간이 없는 상태로 기동하면 기동 시각부터 `saleDurationSec` 동안으로 정해 저장한다. 저장된 판매 기간이 있으면 재시작해도 그대로 쓴다.
- now ≥ `saleEndAt`이면
  - `GET /seats`, 새 `POST /holds` → 409 `SALE_ENDED` (종료 처리 스케줄러가 돌기 전이라도).
  - 이미 있는 예약의 checkout·deposit·release·confirm·입금·취소·조회는 각자 규칙대로 계속 처리한다.
- **판매 종료 처리** (스케줄러, 1초 주기, 판매 기간마다 한 번만): 한 트랜잭션에서
  - HELD 예약 전부 → EXPIRED, 좌석 즉시 반환, 해당 사용자에게 HOLD_CLEARED 알림.
  - 열린 취소표 배치의 좌석은 오픈 이벤트 없이 AVAILABLE로 되돌리고 배치를 닫는다 (`reopenCount`·`reopenSeats`는 늘리지 않는다).
  - 종료 처리를 마친 것을 기록한다.
- 종료 처리가 끝난 뒤에는 HELD 예약이 남아 있으면 안 된다 (I14). 종료 시각 근처에 처리 중이던 선점은 6.5의 6번에서 거절되거나 종료 처리에서 만료된다.
- CONFIRMING·PENDING_DEPOSIT은 각자 규칙대로 결론까지 진행한다 (복구·입금 마감 계속).
- 대기 토큰의 종료는 대기열 서버가 한다 (7.6).

### 6.11 대기열 서버로 알림
- 예약이 생기거나 끝나는 지점에서 대기열 서버 `POST /internal/slots/{kid}/events`(7.5)로 보낸다. 요청 처리 트랜잭션이 **커밋된 뒤** 보내며, 요청 응답을 기다리게 하지 않는다 (비동기).
- 입장키 검사를 꺼서 `kid`가 없는 예약은 알림을 보내지 않는다.

| 사건 | 보내는 이벤트 |
|---|---|
| 선점 성공 | HOLD_ACTIVE |
| 그 사용자의 HELD/CONFIRMING 예약이 모두 끝남 (해제·만료·결제 실패·판매 종료. 요청 밖에서 일어난 것 포함) | HOLD_CLEARED |
| 카드 CONFIRMED (요청 안이든 복구 스케줄러든), 입금 대기 신청 | COMPLETED, 그리고 그 `kid`를 회수 목록에 추가 |
| `soldOut` 값이 바뀜, 그리고 5초마다 | `POST /internal/sale-state` `{ soldOut, releaseAt }` |

- 이벤트의 `at`은 사건이 일어난 시각(서버 시계)이다. 재시도한 시각이 아니다.
- 실패하면 재시도 3회 (실제 시간 1초, 2초, 4초 뒤). 그래도 실패하면 버리고 `counters.slotNotifyDropped` +1, `droppedNotifications`에 `{ kid, type, at }`을 남긴다 (스냅샷에 포함, reset 때 비움).
- 알림이 실패해도 원래 요청의 결과는 바뀌지 않는다.

### 6.12 선점 만료 스케줄러 (`expiry-interval-ms`)
- HELD이고 선점 만료 시각 ≤ now인 예약을 만료 시각이 이른 순으로 최대 `expiry-batch-size`건씩 처리한다: EXPIRED, 좌석 조건부 반환, `released_at` 기록, HOLD_CLEARED 알림, `counters.expiredByScheduler` +1.
- 다른 처리가 잡고 있는 예약은 기다리지 않고 건너뛰어 다음 주기에 다시 본다.
- CONFIRMING은 대상이 아니다.
- 한 번에 `expiry-batch-size`건을 꽉 채워 처리했으면 같은 주기에 이어서 반복한다.

### 6.13 카운터
메모리에 두고 reset 때 0으로 한다. 이름과 증가 시점:

| 이름 | 증가 시점 |
|---|---|
| `holdAttempts` | `POST /holds`가 형식 검사를 통과할 때마다 (멱등 재요청 포함) |
| `holdSuccess` | 새 예약을 만들었을 때 |
| `holdConflicts` | 좌석 확보 실패 시 못 잡은 좌석마다 |
| `confirms` | 카드 결제가 CONFIRMED가 됐을 때 (요청 안, 복구 모두) |
| `declines` | 결제사 승인 거절(402)일 때 |
| `notPayable` | `RESERVATION_NOT_PAYABLE` 응답마다 |
| `expiredByScheduler` | 선점 만료 스케줄러가 만료시킨 예약마다 |
| `recoveryAttempts`, `recovered` | 6.6 |
| `immediateReturns` | 좌석을 즉시 반환할 때마다 좌석 수만큼 |
| `userCancels` | 예매 취소 성공마다 |
| `depositsRequested`, `depositsPaid`, `depositsExpired` | 입금 대기 신청, 입금 성공, 입금 마감 |
| `reopenCount`, `reopenSeats` | 6.8 |
| `slotNotifyDropped` | 6.11 |
| `acceptedInvalidKeys` | 6.3 독립 재검증 실패 |

### 6.14 관리 API
**`POST /admin/reset`** (트래픽이 없을 때만 부른다)
```json
{ "rows": 10, "cols": 10, "grades": [ ... ],
  "strategy": "conditional", "dbBackstop": true,
  "holdTtlSec": 420, "confirmDeadlineSec": 30, "maxSeatsPerUser": 4,
  "depositDeadlineSec": 60, "returnDelaySec": 30, "saleDurationSec": 1200,
  "anchorAt": "...", "saleEndAt": "...", "runEpoch": "..." }
```
- 모든 필드 선택. 빠진 필드는 현재 값을 유지한다. `grades`의 행 수 합 ≠ `rows`, 알 수 없는 전략, 0 이하의 시간·좌석 값은 400.
- `anchorAt`: 3.1대로 서버 시계 기준을 새로 잡는다.
- 판매 기간: 시작 = (새 기준의) now, 종료 = `saleEndAt`이 오면 그 값, 없으면 now + `saleDurationSec`.
- `runEpoch`: 입장키 검사에 쓸 실행 식별자. 없으면 새 무작위 값.
- 처리: 모든 테이블 비우기 → 좌석 다시 만들기 → backstop 적용 → 회수 목록·알림 대기열·`droppedNotifications`·카운터 초기화 → 200 `{ "config" }`.

**`GET /admin/stats`**
```json
{ "serverTime": "...", "runEpoch": "...", "config": { ... }, "saleEndAt": "...",
  "seats": { "total": 100, "AVAILABLE": 12, "HELD": 30, "PENDING_DEPOSIT": 4, "RETURN_PENDING": 0, "SOLD": 54,
             "byGrade": { "VIP": { "AVAILABLE": 0, "HELD": 0, "PENDING_DEPOSIT": 0, "RETURN_PENDING": 0, "SOLD": 10 } } },
  "reservations": { "HELD": 0, "CONFIRMING": 0, "CONFIRMED": 0, "EXPIRED": 0, "RELEASED": 0, "PAYMENT_FAILED": 0,
                    "PENDING_DEPOSIT": 0, "DEPOSIT_EXPIRED": 0, "CANCELED": 0 },
  "payments": { "REQUESTED": 0, "APPROVED": 0, "FAILED": 0, "CANCELED": 0 },
  "releaseBatches": { "open": 1, "nextReleaseAt": "...|null", "totalReleased": 3 },
  "counters": { ... 6.13의 전부 ... } }
```
- 상태별 개수는 DB 기준, 값이 0인 상태도 빠짐없이 넣는다.

**`GET /admin/snapshot`** (불변식 확인용)
```json
{ "serverTime": "...", "runEpoch": "...",
  "seats": [ ... ], "reservations": [ ... ], "reservation_seats": [ ... ], "payments": [ ... ], "release_batches": [ ... ],
  "counters": { ... }, "droppedNotifications": [ { "kid", "type", "at" } ] }
```
- 다섯 테이블의 **전체 행**을 5.1의 컬럼 이름 그대로 담는다. 한 시점의 일관된 상태여야 한다.

**시계**: 3.2.

---

## 7. 대기열 서버 (:8082)

### 7.1 상태 (메모리, 재시작하면 사라진다)
| 항목 | 내용 |
|---|---|
| 대기 토큰 | `{ tokenId, userId, seq, status, createdAt, admittedAt, endedAt, endReason }`. 끝난 토큰도 reset 전까지 보관 |
| 입장 자리(slot) | ADMITTED 토큰마다 `{ keyId, userId, admittedAt, expiresAt, busy, busySince, lastEventAt }` |
| 사용자 인덱스 | userId → 활성 토큰 (WAITING/ADMITTED는 사용자당 1개) |
| 입장 기록 | 입장 주기마다 `{ tickAt, admitted, seqFrom, seqTo, activeAfter }`. 최댓값 `activeMax`, `admittedPerTickMax` 유지 |
| 판매 상태 | `saleEndAt`(reset), `soldOut`(예약 서버 알림) |

### 7.2 입장 규칙
- 입장 스케줄러 (`admission-interval-ms`, 기본 1000): 입장 수 = `min(maxActive − 현재 자리 수, admitPerSec × 주기초)`, seq 오름차순. 입장하면 입장키를 발급하고 자리 `expiresAt = now + admissionTtlSec`.
- 순번(position)은 주기마다 계산해 두고 조회 요청 때는 읽기만 한다.
- 자리 반납: COMPLETED(예약 서버 알림), LEFT(`/queue/leave`), EXPIRED(now ≥ `expiresAt`이고 busy가 아님).
- **만료 미룸**: busy인 자리는 `expiresAt`이 지나도 만료하지 않는다 (결제 도중 보호). 결제 중 유효시간을 넘긴 busy 자리는 **정상**이다.
- **busy 상한**: busy여도 now ≥ `expiresAt + busyMaxExtraSec`이면 강제 만료한다. 기본 510초 (= 선점 유효시간 420 + 승인 기한 30 + 여유 60). 강제 만료마다 `expiredByBusyCap` +1, `busyCapExpirations`에 `{ kid, userId, at }`.
- EXPIRED·LEFT·COMPLETED 사용자가 다시 진입하면 맨 뒤에 새 토큰을 받는다.
- 대기열 서버가 대기자를 임의로 내보내는 경우는 7.6뿐이다.

### 7.3 입장키
- 형식 `v1.<payload>.<sig>` (base64url, 패딩 없음)
  - payload: `{ "kid": "<UUID>", "uid": "<userId>", "iat": <ms>, "exp": <ms>, "run": "<runEpoch>" }` (`iat`·`exp`는 서버 시계 기준 epoch ms, `exp` = 자리 `expiresAt`)
  - sig: `HMAC-SHA256(keySecret, "v1." + payload)`
- `keySecret`은 예약 서버와 같은 값(설정). `runEpoch`는 reset마다 정하며 두 서버에 같은 값을 준다.

### 7.4 사용자 API
**`POST /queue/enter`** `{ "userId" }`
- 활성 토큰(WAITING/ADMITTED)이 있으면 그것을, 없으면 새 WAITING 토큰(또는 7.6의 경우 CLOSED 토큰)을 돌려준다.
- 200 `{ "token", "status", "position", "pollAfterMs", "reason" }`

**`GET /queue/status?token=`**
- WAITING: `{ "status", "position", "pollAfterMs" }`
- ADMITTED: `{ "status", "admissionKey", "admissionExpiresAt" }`
- 그 외: `{ "status", "reason": "SALE_ENDED" | "SOLD_OUT" | null }`
- 없는 토큰·형식 오류 400.
- `pollAfterMs`: position ≤ 50 → 1000, ≤ 300 → 2000, ≤ 1000 → 4000, 그 외 6000.
- 응답에 **좌석 상황 필드는 없다.** 대기 중인 사용자는 좌석 상황을 알 수 없다.

**`POST /queue/leave`** `{ "token" }` → WAITING/ADMITTED면 LEFT, 자리 반납. 그 외 상태면 아무것도 바꾸지 않는다. 200 `{ "left": true }`. 토큰이 없으면 400.

### 7.5 내부 API (예약 서버 → 대기열 서버)
헤더 `X-Internal-Secret` 필수. 틀리거나 없으면 401.

**`POST /internal/slots/{kid}/events`** `{ "type": "HOLD_ACTIVE" | "HOLD_CLEARED" | "COMPLETED", "at" }`
| type | 처리 |
|---|---|
| HOLD_ACTIVE | busy = true, `busySince = at` |
| HOLD_CLEARED | busy = false (만료 시각이 이미 지났으면 다음 입장 주기에 EXPIRED) |
| COMPLETED | 토큰 COMPLETED, 자리 반납 |
- 없는 kid, 이미 끝난 자리 → 200 `{ "ignored": true }`.
- kid마다 마지막으로 반영한 이벤트의 `at`보다 오래된 이벤트는 무시한다. COMPLETED 이후의 모든 이벤트는 무시한다. 무시하면 `slotEvents.ignored` +1.

**`POST /internal/sale-state`** `{ "soldOut", "releaseAt" }` → 7.6에만 쓴다.

| 알림 유실 | 영향 | 회복 |
|---|---|---|
| HOLD_ACTIVE | 결제 중 자리가 만료될 수 있음 (예약 서버의 만료 예외로 결제는 계속됨) | 없음 (정원이 잠깐 1 늘어남) |
| HOLD_CLEARED | 자리가 busy로 묶임 | busy 상한 |
| COMPLETED | 끝난 사용자가 자리를 차지 | 만료 시각 (+ busy 상한) |
| sale-state | 매진 반영 지연 | 5초 주기 재전송 |

### 7.6 판매 상태 처리
- **판매 종료** (now ≥ `saleEndAt`): WAITING/ADMITTED 토큰 전부 CLOSED(`SALE_ENDED`), 자리 전부 반납. 이후 enter는 200 + CLOSED(`SALE_ENDED`) 토큰.
- **`closeQueueOnSoldOut`** (기본 true):
  - true이고 soldOut이면 **WAITING만** CLOSED(`SOLD_OUT`)로 바꾸고, 이후 enter도 CLOSED(`SOLD_OUT`) 토큰을 준다. soldOut = false가 오면 다시 접수한다.
  - false면 매진이어도 아무도 내보내지 않는다.
- **ADMITTED는 매진으로 닫지 않는다.** 입장한 사람의 자리는 7.2의 규칙(COMPLETED, LEFT, 만료, busy 상한)으로만 반납한다. 그래서 다시 열려도 실제 빈 자리만큼만 입장시키고, **유효한 입장키를 가진 사람 수 ≤ maxActive**가 항상 지켜진다 (입장키는 서명 토큰이라 대기열 서버가 상태를 바꿔도 `exp`까지 예약 서버에서 유효하기 때문).
- 입장키는 회수하지 않는다. 입장한 사용자는 키 만료까지 `GET /seats`를 쓸 수 있다.

### 7.7 관리 API
- **`POST /admin/reset`** `{ "anchorAt", "maxActive", "admitPerSec", "admissionTtlSec", "busyMaxExtraSec", "saleEndAt", "closeQueueOnSoldOut", "runEpoch" }` (모두 선택, 빠지면 현재 값 유지. `anchorAt`은 3.1. 토큰·자리·기록·카운터를 모두 비운다) → 200 `{ "config" }`
- **`GET /admin/stats`** → `{ "serverTime", "WAITING", "ADMITTED", "COMPLETED", "LEFT", "EXPIRED", "CLOSED", "busy", "slotsBusyOverTtl", "counters": { "entered", "admitted", "keysIssued", "expired", "expiredByBusyCap", "closedSoldOut", "closedSaleEnded", "slotEvents": { "HOLD_ACTIVE", "HOLD_CLEARED", "COMPLETED", "ignored" } } }`
  - `slotsBusyOverTtl` = busy이면서 now ≥ `expiresAt`인 자리 수.
- **`GET /admin/snapshot`** → `{ "serverTime", "config", "tokens": [...], "slots": [...], "admissions": [...], "activeMax", "admittedPerTickMax", "busyCapExpirations": [...] }`
- **시계**: 3.2.

### 7.8 설정
```yaml
server.port: 8082
queue: { max-active: 200, admit-per-sec: 20, admission-ttl-sec: 420, busy-max-extra-sec: 510,
         close-queue-on-sold-out: true, admission-interval-ms: 1000 }
admission.key-secret: dev-only-change-me
internal.secret: dev-internal-secret
```

---

## 8. 모의 결제사 (mock-pg, :8081)

결제 정보는 메모리에 둔다.

- **`POST /auth`** `{ orderId, amount }` → 확률 `authFailureRate`로 400 `AUTH_FAILED`. 성공하면 결제를 AUTHORIZED로 저장하고 200 `{ paymentKey }`.
- **`POST /payments/confirm`** `{ paymentKey, orderId, amount }`
  - 지연 U(`confirmMinMs`, `confirmMaxMs`) (실제 시간).
  - 확률 `timeoutRate`로 응답하지 않고 `confirmMaxMs + 10000`ms 동안 붙잡아 둔다. 이때도 내부적으로 50% 확률로 승인(DONE)할지 정해 저장한다 (결과를 모르는 상황 재현).
  - 확률 `declineRate`로 400 `INSUFFICIENT_FUNDS` (DECLINED로 저장).
  - 키·주문·금액 불일치, 이미 처리된 결제 → 400 `INVALID_REQUEST`.
  - 성공 200 `{ status: "DONE", approvedAt }`.
- **`GET /payments/{key}`** → 200 `{ status }` (AUTHORIZED / DONE / DECLINED / CANCELED) 또는 404.
- **`POST /payments/{key}/cancel`** → DONE·AUTHORIZED면 CANCELED로 바꾸고 200 `{ status }`. 그 외 상태(CANCELED, DECLINED)는 바꾸지 않고 현재 상태로 200 (멱등). 없는 키 404.
- **관리**
  - `GET/PUT /admin/config` `{ authFailureRate, declineRate, timeoutRate, confirmMinMs, confirmMaxMs, seed }`. 확률은 0~1, `0 ≤ confirmMinMs ≤ confirmMaxMs`가 아니면 400 `INVALID_REQUEST`. 같은 `seed`면 같은 요청 순서에 같은 무작위 결과, `seed`가 null이면 매번 다른 결과.
  - `POST /admin/reset` → 결제·통계 초기화 (본문에 `anchorAt`이 있으면 3.1).
  - `GET /admin/stats` `{ serverTime, authRequests, authFailed, confirmRequests, done, declined, timedOut, canceled, confirmInflight, confirmLatency: { p50, p95, p99 } }`
  - `GET /admin/payments` → 모든 결제 `{ paymentKey, orderId, amount, status }`.
  - 시계: 3.2.
- 기본 설정: `authFailureRate` 0.03, `declineRate` 0.03, `timeoutRate` 0, `confirmMinMs` 100, `confirmMaxMs` 500, `seed` null.

---

## 9. 불변식

`/admin/snapshot`(세 서버)과 `/admin/payments`로 확인할 수 있어야 한다. "정지 상태"는 트래픽이 멈춘 뒤 `holdTtlSec + confirmDeadlineSec + 5`초가 지난 상태다.

| ID | 불변식 |
|---|---|
| I1 | 좌석 SOLD ⇔ 그 좌석을 가진(`released_at` null) CONFIRMED 예약이 정확히 1건 |
| I2 | 좌석마다 `released_at`이 null인 `reservation_seats` 행은 최대 1개 (`naive` + backstop 끔 제외) |
| I3 | 결제사에서 DONE(취소 안 됨)인 결제 ⇔ 예약 서버의 APPROVED 결제 + CONFIRMED 예약 (돈만 빠지고 좌석이 없는 경우 0) |
| I4 | 사용자당 HELD/CONFIRMING 예약 최대 1건 |
| I5 | 좌석 상태별 수의 합 = 전체 좌석. HELD ⇔ 소유 예약이 HELD/CONFIRMING, PENDING_DEPOSIT ⇔ 소유 예약이 PENDING_DEPOSIT, RETURN_PENDING ⇔ 열린 배치 소속 |
| I6 | 정지 상태에서 HELD/CONFIRMING 예약 0 |
| I7 | 같은 `Idempotency-Key` ⇒ 예약 1건, 같은 응답 본문 |
| I8 | 대기열 서버: 어느 시점에도 자리 수 ≤ `maxActive`, 입장은 seq 순서, 주기당 입장 수 ≤ `admitPerSec × 주기초` |
| I9 | 승인 직전 확인에서 만료된 예약에 대해 결제사 승인 호출 0 |
| I10 | 한 예약의 좌석은 항상 같은 상태 |
| I11 | 사용자당 CONFIRMED + PENDING_DEPOSIT 예약 최대 1건, 그 좌석 합 ≤ `maxSeatsPerUser` |
| I12 | 결제사 DONE(취소 안 됨) 결제 ⇔ CARD 방식 CONFIRMED 예약. CANCELED 예약의 카드 결제는 결제사에서도 CANCELED |
| I13 | RETURN_PENDING 좌석은 배치의 `release_at` + 2초 안에 AVAILABLE이 된다 (판매 종료 처리는 예외) |
| I14 | 판매 종료 후: 대기열 서버의 WAITING·ADMITTED 0, 예약 서버의 HELD 0 (종료 처리 주기 이내) |
| I15 | busy 상한 강제 만료(`busyCapExpirations`)의 각 kid에 대해 (a) 예약 서버 `droppedNotifications`에 그 kid의 HOLD_CLEARED 또는 COMPLETED가 있거나 (b) 그 시각에 그 kid의 예약이 CONFIRMING이었다. 결제 중 유효시간을 넘긴 busy 자체는 위반이 아니다 |
| I16 | 예약 서버 `counters.acceptedInvalidKeys` = 0 |
| I17 | 유효한 입장키(만료 전, 현재 runEpoch)를 가진 사용자 수 ≤ `maxActive` |

---

## 10. 구현하지 말 것

| 제외 |
|---|
| 서버 다중 인스턴스, Redis, 분산 락 |
| 결제사 호출 재시도·백오프·서킷 브레이커 (6.6의 조회 1회 + 망취소 + 복구 스케줄러까지만) |
| 대기열의 외부 저장소 이전, 봇·매크로 차단, 요청 수 제한, 공정성 정책(대기열 무작위화 등), 입장 속도 자동 조절 |
| 과부하 대응 (요청 버리기, 429/503) |
| 인증·인가, 실제 결제, 애플리케이션 Docker 이미지, 메시지 큐, Kubernetes |
| 성능 계측·대시보드·화면 |
