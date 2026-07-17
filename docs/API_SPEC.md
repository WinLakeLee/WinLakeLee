# TCG 오리파 플랫폼 — API 명세서

`docs/ERD.md`의 데이터 모델을 기반으로 한 REST API 설계. FastAPI 구현을 전제로 하며, 서비스 경계는
프로젝트 계획서 §2 아키텍처(Auth / Card / Oripa / Inventory / Payment / Event / Admin)를 따른다.

## 목차

0. [공통 규약](#0-공통-규약)
1. [Auth & User Service](#1-auth--user-service)
2. [Card Catalog Service](#2-card-catalog-service-public)
3. [Oripa (뽑기) Service](#3-oripa-뽑기-service)
4. [Inventory & Shipping Service](#4-inventory--shipping-service)
5. [Payment & Wallet Service](#5-payment--wallet-service)
6. [Event Service](#6-event-service)
7. [Admin Service](#7-admin-service)
8. [Consignment & Settlement Service](#8-consignment--settlement-service)
9. [WebSocket — 실시간 뽑기 피드](#9-websocket--실시간-뽑기-피드)
10. [에러 코드 표준](#10-에러-코드-표준)

---

## 0. 공통 규약

**Base URL**: `https://api.<domain>/api/v1`

**인증**
- `Authorization: Bearer <access_token>` (JWT, 만료 15분)
- 만료 시 `POST /auth/refresh`로 Refresh Token Rotation
- Admin 라우트(`/admin/*`)는 `role=admin` 이상 + 2FA 통과 세션 클레임(`amr: ["mfa"]`) 필수

**멱등성**
- 상태를 변경하는 금전/뽑기 API(`POST /wallet/charges`, `POST /oripa-packs/{id}/draws`)는 `Idempotency-Key` 헤더 필수.
  동일 키 재요청 시 최초 응답을 그대로 반환(신규 처리 안 함).

**페이지네이션**
- 커서 기반: 쿼리 파라미터 `cursor`, `limit`(기본 20, 최대 100)
- 응답 공통 포맷:
```json
{
  "data": [ ... ],
  "next_cursor": "eyJpZCI6MTIzfQ==",
  "has_more": true
}
```

**에러 포맷 (RFC 7807 변형)**
```json
{
  "error": {
    "code": "PACK_SOLD_OUT",
    "message": "이 팩은 이미 완판되었습니다.",
    "request_id": "req_9f2c..."
  }
}
```

**레이트 리밋**: 응답 헤더 `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Redis 기반, IP+계정 단위).

---

## 1. Auth & User Service

| Method | Path | Auth | 설명 |
|---|---|---|---|
| POST | `/auth/register` | - | 이메일 회원가입 (가입 직후 `POST /auth/verify-email/request`로 이메일 인증 발송) |
| POST | `/auth/login` | - | 이메일 로그인 → access/refresh 토큰 발급 |
| GET | `/auth/oauth/{provider}/redirect` | - | OAuth 인가 URL 리다이렉트 (`provider`: kakao, naver, google, apple) |
| POST | `/auth/oauth/{provider}/callback` | - | OAuth 콜백 처리 → 신규면 자동 회원가입, 기존이면 로그인 (§1.1) |
| POST | `/auth/refresh` | Refresh Token | Access Token 재발급 (Rotation) |
| POST | `/auth/logout` | Bearer | 현재 Refresh Token 폐기 |
| POST | `/auth/2fa/enable` | Bearer | TOTP 시크릿 발급 (QR용 otpauth URI 반환) |
| POST | `/auth/2fa/verify` | Bearer | TOTP 코드 검증 후 2FA 활성화 확정 |
| GET | `/users/me` | Bearer | 내 프로필 조회 (연결된 SNS 계정 목록 포함) |
| PATCH | `/users/me` | Bearer | 닉네임/프로필이미지 등 민감하지 않은 정보 즉시 수정 |
| GET | `/users/me/oauth-accounts` | Bearer | 연결된 SNS 계정 목록 |
| POST | `/users/me/oauth-accounts/link` | Bearer | 로그인 상태에서 추가 SNS 계정 연결 |
| DELETE | `/users/me/oauth-accounts/{provider}` | Bearer | SNS 계정 연결 해제 |
| POST | `/users/me/email-change/request` | Bearer + 비밀번호 재확인 | 신규 이메일로 인증 코드 발송 |
| POST | `/users/me/email-change/confirm` | Bearer | 인증 코드 검증 후 이메일 변경 확정 |
| POST | `/users/me/phone-change/request` | Bearer | 신규 전화번호로 SMS 인증 코드 발송 |
| POST | `/users/me/phone-change/confirm` | Bearer | 인증 코드 검증 후 전화번호 변경 확정 |
| POST | `/users/me/password` | Bearer + 현재 비밀번호 확인 | 비밀번호 변경 (전 세션 강제 로그아웃) |
| GET | `/users/me/change-logs` | Bearer | 본인 정보 변경 이력 조회 |
| GET | `/users/me/addresses` | Bearer | 배송지 목록 |
| POST | `/users/me/addresses` | Bearer | 배송지 등록 |
| PATCH | `/users/me/addresses/{id}` | Bearer | 배송지 수정 |
| DELETE | `/users/me/addresses/{id}` | Bearer | 배송지 삭제 |

**POST /auth/login — 요청/응답 예시**
```json
// Request
{ "email": "user@example.com", "password": "..." }

// 200 Response
{
  "access_token": "eyJ...",
  "refresh_token": "8f3a...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": { "id": 101, "nickname": "otaku_hunter", "role": "user" }
}
```
- 실패 시 `401 INVALID_CREDENTIALS`. 5회 연속 실패 시 캡차 요구(`428 CAPTCHA_REQUIRED`).

### 1.1 SNS 회원가입/로그인 (Kakao, Naver, Google, Apple)

지원 우선순위(한국 시장 기준): **카카오 → 네이버 → 구글 → 애플**. 4개 provider 모두 동일한 콜백 계약을
따르며, provider별 클라이언트 구현만 전략 패턴으로 분리한다(`AppleOAuthClient`는 최초 로그인에만
이름이 내려오고 private relay 이메일을 쓸 수 있다는 점만 별도 처리).

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Auth API
    participant P as SNS Provider
    participant DB as PostgreSQL

    C->>API: GET /auth/oauth/{provider}/redirect
    API-->>C: 302 Provider 인가 URL
    C->>P: 로그인/동의
    P-->>C: redirect_uri?code=...
    C->>API: POST /auth/oauth/{provider}/callback { code }
    API->>P: code → access_token 교환, 프로필 조회
    API->>DB: user_oauth_accounts 조회 (provider, provider_user_id)
    alt 이미 연결된 계정 존재
        API->>DB: 해당 users로 로그인 처리
    else provider 이메일이 기존 users.email과 일치 + 인증됨
        API-->>C: 200 { needs_link_confirmation: true, existing_user_masked }
        C->>API: POST /auth/oauth/{provider}/callback { code, confirm_link: true }
        API->>DB: user_oauth_accounts INSERT (기존 계정에 연결)
    else 신규
        API->>DB: users INSERT (signup_method=provider) + user_oauth_accounts INSERT
    end
    API-->>C: 200 access/refresh token
```

**POST /auth/oauth/{provider}/callback — 응답 예시**
```json
// 신규 가입 또는 기존 연결 계정 로그인
{
  "access_token": "eyJ...",
  "refresh_token": "8f3a...",
  "is_new_user": true,
  "user": { "id": 205, "nickname": "리자몽매니아", "signup_method": "kakao" }
}

// 계정 연결 확인이 필요한 경우 (동일 이메일의 기존 이메일 가입 계정 존재)
{
  "needs_link_confirmation": true,
  "existing_user_masked": { "email_masked": "us***@example.com", "signup_method": "email" },
  "link_token": "lnk_7f2a..."
}
```
- 연결 확인이 필요한 응답을 받으면 클라이언트는 사용자에게 "기존 계정과 연결하시겠습니까?"를 안내 후
  `POST /auth/oauth/{provider}/callback`에 `link_token` + `confirm_link: true`를 재전송해 확정한다.
- 이메일이 provider에서 인증되지 않은 상태(예: Apple private relay)로 왔다면 자동 연결 제안 없이 신규 계정으로 분리 생성(§ERD.md `user_oauth_accounts.email_verified_by_provider`).
- 이미 로그인 상태에서 추가 SNS를 연결하려면 `POST /users/me/oauth-accounts/link`(Bearer 필요, code 교환 흐름 동일).
- `DELETE /users/me/oauth-accounts/{provider}`는 연결 해제 후 로그인 수단이 하나도 남지 않으면(비밀번호 미설정 + SNS 계정 1개뿐) `409 LAST_LOGIN_METHOD` 반환.

### 1.2 유저 정보 수정 (민감 정보 2단계 인증)

이메일·전화번호·비밀번호 변경은 계정 탈취 시 피해가 큰 변경이므로 요청(request)/확정(confirm) 2단계로 분리한다.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Auth API
    participant DB as PostgreSQL

    C->>API: POST /users/me/email-change/request { new_email, current_password }
    API->>DB: 현재 비밀번호 검증, verification_codes INSERT (purpose=email_change)
    API-->>C: 202 { expires_in: 600 }
    Note over API: 신규 이메일로 인증 코드 발송
    C->>API: POST /users/me/email-change/confirm { code }
    API->>DB: code 검증 (attempt_count 체크), users.email UPDATE, user_change_logs INSERT
    API-->>C: 200 { email: "new@example.com" }
```

```json
// POST /users/me/email-change/request
// Request
{ "new_email": "new@example.com", "current_password": "..." }
// 202 Response
{ "expires_in": 600 }

// POST /users/me/email-change/confirm
// Request
{ "code": "482913" }
// 200 Response
{ "email": "new@example.com" }
```
- 실패: `401 INVALID_PASSWORD`, `409 EMAIL_ALREADY_IN_USE`, `422 INVALID_CODE`(5회 초과 시 `429 TOO_MANY_ATTEMPTS`, 코드 재발급 필요).
- `POST /users/me/password` 성공 시 현재 세션을 제외한 모든 `refresh_tokens`를 `revoked_at` 처리(다른 기기 강제 로그아웃).
- 모든 변경은 성공/실패 관계없이 `user_change_logs`에 기록되며, `GET /users/me/change-logs`로 본인이 직접 열람 가능(계정 이상 징후 셀프 체크).

---

## 2. Card Catalog Service (Public)

| Method | Path | Auth | 설명 |
|---|---|---|---|
| GET | `/games` | - | 지원 게임 목록 (Pokemon, One Piece, Yu-Gi-Oh! 등) |
| GET | `/games/{game_code}/sets` | - | 게임별 확장팩/세트 목록 |
| GET | `/cards` | - | 카드 검색 (`?game=&set=&rarity=&q=&cursor=`) |
| GET | `/cards/{id}` | - | 카드 상세 (공식 이미지 + attributes) |
| GET | `/cards/{id}/price-history` | - | 시세 그래프 데이터 (§8 추가 제안 기능) |

**GET /cards/{id} — 응답 예시**
```json
{
  "id": 5821,
  "game": "pokemon",
  "set": { "code": "sv4a", "name": "Shiny Treasure ex" },
  "card_number": "080/190",
  "name": "리자몽 ex",
  "rarity": "SAR",
  "attributes": { "hp": 330, "type": "Fire", "regulation_mark": "H" },
  "images": [
    { "type": "official_front", "url": "https://cdn.../front.webp" }
  ]
}
```

---

## 3. Oripa (뽑기) Service

| Method | Path | Auth | 설명 |
|---|---|---|---|
| GET | `/oripa-packs` | - | 판매중/예정 팩 목록 (`?game=&status=`) |
| GET | `/oripa-packs/{id}` | - | 팩 상세 — 잔여 구수, **등급별 잔여 수량 실시간 공개**(투명성, §8) |
| GET | `/oripa-packs/{id}/proof` | - | Provably Fair 증명 정보 (커밋 해시, sold_out 후 seed reveal) |
| POST | `/oripa-packs/{id}/draws` | Bearer + Idempotency-Key | 1회 뽑기 실행 |
| GET | `/oripa-packs/{id}/draws/{draw_id}` | Bearer | 특정 뽑기 결과 상세 |
| GET | `/users/me/draws` | Bearer | 내 뽑기 이력 |

**GET /oripa-packs/{id} — 응답 예시 (투명성 핵심 필드)**
```json
{
  "id": 42,
  "name": "포켓몬 SAR 이상 확정 오리파",
  "game": "pokemon",
  "price_per_draw": 3000,
  "total_slots": 100,
  "remaining_slots": 37,
  "status": "on_sale",
  "last_one_bonus_enabled": true,
  "grade_summary": [
    { "grade": "SS", "total": 1, "remaining": 1 },
    { "grade": "S", "total": 5, "remaining": 2 },
    { "grade": "A", "total": 20, "remaining": 9 },
    { "grade": "participation", "total": 74, "remaining": 25 }
  ],
  "server_seed_hash": "b7e2...ff01"
}
```

**POST /oripa-packs/{id}/draws — 처리 흐름**
```mermaid
sequenceDiagram
    participant C as Client
    participant API as Oripa API
    participant Redis as Redis (분산락)
    participant DB as PostgreSQL

    C->>API: POST /draws (Idempotency-Key)
    API->>Redis: 락 획득 pack:{id}:lock
    API->>DB: BEGIN; SELECT balance FOR UPDATE (wallet)
    API->>DB: 잔액 검증, ledger_entries INSERT(draw_spend, -price)
    API->>DB: SELECT available slots FOR UPDATE SKIP LOCKED
    API->>API: secrets.randbelow(remaining_count)로 슬롯 선택
    API->>DB: slot.status='drawn', draw_logs INSERT, user_inventory INSERT
    API->>DB: COMMIT
    API->>Redis: 락 해제
    API-->>C: 200 draw 결과 (연출용 payload)
    API--)C: WS 브로드캐스트 (실시간 피드)
```

```json
// Request — 포인트로 뽑기
POST /oripa-packs/42/draws
Idempotency-Key: 7c1b6e2a-...
{ }

// Request — 무료 뽑기권으로 뽑기 (§6 이벤트 보상으로 지급된 티켓)
POST /oripa-packs/42/draws
Idempotency-Key: 9a1f4c3d-...
{ "ticket_id": 5510 }

// 200 Response
{
  "draw_id": 991234,
  "pack_id": 42,
  "slot_no": 63,
  "grade": "A",
  "physical_card": {
    "id": 88213,
    "card_name": "피카츄 V",
    "condition_grade": "S",
    "is_graded": false,
    "images": ["https://cdn.../photo1.webp"]
  },
  "payment_method": "points",
  "points_spent": 3000,
  "bonus_spent": 500,
  "wallet_balance_after": { "paid": 46500, "bonus": 0 },
  "remaining_slots": 36
}
```
- `ticket_id` 생략 시 포인트 차감(무상 적립금 우선 소진 후 유상 잔액, §ERD.md §7), `ticket_id` 지정 시 해당 `free_draw_tickets`를 소비하고 포인트 차감 없음(`payment_method: "free_ticket"`).
- `ticket_id`가 지정한 `pack_scope`와 대상 팩이 맞지 않으면 `422 TICKET_SCOPE_MISMATCH`, 이미 사용/만료된 티켓이면 `409 TICKET_NOT_AVAILABLE`.
- 실패: `409 PACK_SOLD_OUT`, `402 INSUFFICIENT_BALANCE`, `423 PACK_LOCKED`(락 획득 실패, 재시도 유도).

**GET /oripa-packs/{id}/proof — sold_out 이전/이후**
```json
// on_sale 상태
{ "status": "on_sale", "server_seed_hash": "b7e2...ff01", "reveal_available": false }

// settled 상태 (검증 가능)
{
  "status": "settled",
  "server_seed_hash": "b7e2...ff01",
  "server_seed": "f3a9c1...",
  "slot_mapping": [ { "slot_no": 1, "grade": "participation", "physical_card_id": 8811 }, ... ],
  "verify_instruction": "SHA256(server_seed || slot_mapping_json) 계산 결과가 server_seed_hash와 일치하는지 확인하세요."
}
```

---

## 4. Inventory & Shipping Service

| Method | Path | Auth | 설명 |
|---|---|---|---|
| GET | `/users/me/inventory` | Bearer | 보유 카드 목록 (`?status=held`) |
| POST | `/shipping-requests` | Bearer | 배송 신청 (여러 인벤토리 항목 묶음 가능) |
| GET | `/shipping-requests/{id}` | Bearer | 배송 신청 상세/추적 |
| GET | `/users/me/shipping-requests` | Bearer | 내 배송 신청 이력 |
| POST | `/shipping-requests/{id}/confirm-receipt` | Bearer | 수령 확인(조기 확정) — 위탁 카드 정산 보류 해제 트리거 |
| POST | `/shipping-requests/{id}/dispute` | Bearer | 수령 물품 이의제기 (검수기간 내) — 정산 보류 동결 |
| POST | `/refund-requests` | Bearer | 포인트 환급(매입) 신청 |
| GET | `/refund-requests/{id}` | Bearer | 환급 신청 상태 조회 |

**POST /shipping-requests — 요청 예시**
```json
{
  "address_id": 12,
  "user_inventory_ids": [88213, 88214, 88250]
}
```
```json
// 201 Response
{
  "id": 5501,
  "status": "requested",
  "items": [88213, 88214, 88250],
  "requested_at": "2026-07-17T10:00:00Z"
}
```

**POST /refund-requests — 요청/응답 예시**
```json
// Request
{ "user_inventory_id": 88250 }

// 201 Response
{ "id": 771, "status": "pending", "estimated_refund_points": 800 }
```
- 승인(`admin`) 시 `ledger_entries`에 `refund_credit` 자동 기록되며 `status=paid`로 전환.

**POST /shipping-requests/{id}/confirm-receipt / dispute — 수령 확인 & 이의제기**
```json
// 수령 확인
// 200 Response
{ "id": 5501, "status": "confirmed", "confirmed_at": "2026-07-20T09:00:00Z" }

// 이의제기 Request
{ "user_inventory_id": 88213, "reason": "condition_mismatch", "evidence_urls": ["https://cdn.../photo.jpg"] }
// 201 Response
{ "dispute_id": 331, "status": "open" }
```
- `delivered_at`으로부터 `inspection_deadline_at`(예: 7일)이 지나도록 확인/이의제기가 없으면 시스템이 자동으로 `confirmed_at`을 채운다(배치 잡).
- 위탁 카드(§ERD.md §11)의 경우 `confirmed_at`이 채워지는 순간 `settlement_holds`가 해제되어 위탁자 정산이 진행되고, `dispute`가 열리면 해제가 동결된다. 상세 흐름은 §8 참고.

---

## 5. Payment & Wallet Service

### 5.1 PG / 지급대행 벤더 선정

자체 PG·지급대행업 등록은 현실적으로 불가능(전자금융거래법상 자본금 10억원 이상, 전산전문인력 5인 이상 등 요건)하므로,
기존 라이선스를 가진 벤더를 반드시 경유한다. 공개된 수수료율은 매출 규모·거래량에 따라 실제 협상에서 달라지는 경우가
많아 최종 확정 전 직접 견적 비교가 필요하다.

**결제(충전) — 구매자 대금**

| 벤더 | 카드 수수료 | 가입비/관리비 | 비고 |
|---|---|---|---|
| **포트원(PortOne)** — 통합 게이트웨이 | 하위 PG 요율 그대로 적용 | 일부 PG 가입비 면제 패키지 제공 | 하나의 연동으로 하위 PG(KG이니시스/토스/카카오페이 등) 자유 교체·병행 — **1순위 채택** |
| NHN KCP | 3.4% (계좌이체 1.8%, 가상계좌 300원) | 가입비 이벤트 무료, 연관리비 평생 면제 | 정산주기 선택 가능(일일~월1회) |
| KG이니시스 | 2.1~3.3% (매출구간별 우대) | 약 22만원(솔루션 이용 시 면제 가능) | 우대구간 폭넓음 |
| 토스페이먼츠 | 3.4~4.3% (등급/프로모션별 편차) | 연 11만원 | 개발 편의성 우수, 스타트업 프로모션 |
| 카카오페이(간편결제) | 연매출 5억 이하 영세 0.5% | - | 포트원 경유 허브형 연동 권장 |

→ **결정**: PG는 포트원을 게이트웨이로 두고, 초기 하위 PG는 NHN KCP(연관리비 평생면제) 또는 KG이니시스(낮은 우대 수수료)로 시작, 매출 성장 시 재협상.

**지급대행(위탁자 정산) — §8 Consignment & Settlement Service 연동**

| 벤더 | 요금 구조 | 비고 |
|---|---|---|
| **포트원 파트너 정산 자동화** | 견적 필요 | 정산금액 자동계산 + **세금계산서 발행 + 송금까지 자동화** — 개인 위탁자 원천징수 이슈까지 커버, 마켓플레이스/위탁판매 특화. **1순위 채택** |
| 토스페이먼츠 지급대행 | 월 30만원 고정, 1,000건까지 무료 | 거래량이 월 1,000건을 넘어서면 건당 비용 유리 — 스케일 업 시 재비교 대상 |
| 나이스페이먼츠 지급대행 | 월 15일 정산, 요율 비공개 | 오픈마켓형 하위판매자 정산에 특화 |

→ **결정**: 초기엔 위탁 거래량이 적고 세금계산서/원천징수 자동화가 중요하므로 **포트원 파트너 정산 자동화**를 우선 채택, 위탁 거래량이 커지면 토스페이먼츠 지급대행과 손익분기점을 재계산해 전환을 검토한다.

| Method | Path | Auth | 설명 |
|---|---|---|---|
| GET | `/users/me/wallet` | Bearer | 포인트 잔액 조회 (유상/무상 분리) |
| POST | `/wallet/charges` | Bearer + Idempotency-Key | 결제 요청 생성 (PG 리다이렉트/승인용 정보 반환) |
| POST | `/webhooks/payments/{provider}` | 서명 검증(HMAC) | PG 웹훅 콜백 (`provider`: portone 경유가 기본, sub_pg로 실제 PG 구분) — server-to-server |
| GET | `/users/me/ledger` | Bearer | 포인트 증감 원장 이력 |
| GET | `/users/me/bonus-credits` | Bearer | 보유 중인 무상 적립금 내역 (출처별, 만료일 포함) |
| GET | `/charge-bonus-rules/active` | - | 현재 활성화된 충전 수단별 보너스 규칙 조회 (충전 화면 안내용) |

**GET /charge-bonus-rules/active — 응답 예시 (계좌이체 유도)**
```json
{
  "rules": [
    { "method": "transfer", "bonus_rate": 0.03, "min_charge_amount": 10000, "max_bonus_amount": 5000, "description": "계좌이체 충전 시 3% 추가 적립 (최대 5,000P)" },
    { "method": "card", "bonus_rate": 0, "min_charge_amount": 0, "max_bonus_amount": 0, "description": null }
  ]
}
```
- 충전 화면에서 이 API로 수단별 보너스율을 안내해 계좌이체 선택을 유도한다 — 계좌이체 PG 수수료(약 1.8%)가 카드(3.4%+)보다 낮아(§5.1) 절감분 일부를 즉시 무상 적립금으로 되돌려주는 구조.

**POST /wallet/charges — 요청/응답 예시**
```json
// Request
{ "amount": 50000, "method": "transfer", "pg_provider": "portone", "sub_pg": "nhn_kcp" }

// 200 Response
{
  "payment_transaction_id": 33012,
  "status": "pending",
  "pg_checkout_url": "https://checkout.portone.io/...",
  "amount": 50000,
  "applicable_bonus": { "method": "transfer", "bonus_rate": 0.03, "estimated_bonus_amount": 1500 }
}
```
- `pg_provider`는 항상 `portone`(통합 게이트웨이), `sub_pg`로 실제 카드사 라우팅 PG(`nhn_kcp`, `kg_inicis`, `toss`, `kakaopay` 등)를 지정 — 특정 PG 장애/정책 변경 시 코드 변경 없이 `sub_pg` 라우팅만 교체 가능.

**POST /webhooks/payments/{provider} — 검증 절차**
1. `X-Signature` 헤더를 PG 공개 검증 규칙(HMAC-SHA256)으로 재계산 후 대조 — 불일치 시 `401` 즉시 반환.
2. `payment_transactions.pg_tid` 존재 여부로 중복 웹훅 무시(이미 `approved`면 `200 OK`만 반환, 재처리 없음).
3. 서버측 금액 재검증: 웹훅 `amount`가 원 요청 `payment_transactions.amount`와 다르면 `422 AMOUNT_MISMATCH` 처리 후 관리자 알림.
4. 승인 확정 시 트랜잭션(단일 DB 트랜잭션): `payment_transactions.status='approved'` + `ledger_entries` INSERT(`charge`, `balance_type=paid`, `idempotency_key=pg_tid`) + `wallets.paid_balance` 갱신 + (활성 `charge_bonus_rules` 매칭 시) `bonus_credit_grants` INSERT + `ledger_entries` INSERT(`bonus_grant`, `balance_type=bonus`) + `wallets.bonus_balance` 갱신.

```json
// 200 Response (조회용)
GET /users/me/wallet
{
  "paid_balance": 46500,
  "bonus_balance": 2000,
  "total_balance": 48500,
  "updated_at": "2026-07-17T09:55:00Z"
}
```

**GET /users/me/bonus-credits — 응답 예시**
```json
{
  "data": [
    { "id": 881, "source_type": "transfer_bonus", "amount": 1500, "remaining_amount": 1500, "expires_at": "2026-10-17T00:00:00Z" },
    { "id": 879, "source_type": "attendance", "amount": 500, "remaining_amount": 0, "expires_at": "2026-08-15T00:00:00Z" }
  ]
}
```

---

## 6. Event Service

출석/미션/시즌패스/로열티 등급/추천인 등 리텐션 기능. 보상은 대부분 무상 적립금(`bonus_credit_grants`) 또는 무료 뽑기권(`free_draw_tickets`)으로 지급되며 §5/§7의 지갑 구조를 그대로 사용한다.

| Method | Path | Auth | 설명 |
|---|---|---|---|
| POST | `/events/attendance/check-in` | Bearer | 당일 출석 체크 (1일 1회), 캘린더 순번 보상 지급 |
| GET | `/events/attendance/status` | Bearer | 연속 출석 현황 + 캘린더 진행도 |
| GET | `/missions` | Bearer | 진행 가능한 미션 목록과 내 진행도 |
| POST | `/missions/{id}/claim` | Bearer | 완료된 미션 보상 수령 |
| GET | `/season-pass/current` | Bearer | 현재 시즌패스 진행 상태 (XP, 레벨, 트랙별 보상 수령 여부) |
| POST | `/season-pass/premium/purchase` | Bearer + Idempotency-Key | 프리미엄 트랙 구매 (포인트 차감) |
| POST | `/season-pass/rewards/{level}/claim` | Bearer | 레벨 보상 수령 (`?track=free\|premium`) |
| GET | `/users/me/loyalty` | Bearer | 내 로열티 등급, 누적 충전액, 다음 등급까지 남은 금액 |
| GET | `/users/me/draw-tickets` | Bearer | 보유 무료 뽑기권 목록 |
| POST | `/coupons/redeem` | Bearer | 쿠폰 코드 등록 |
| GET | `/rankings` | - | 랭킹 조회 (`?period=weekly`) |
| POST | `/referrals/invite` | Bearer | 초대 코드 발급/조회 |
| GET | `/events` | - | 진행중 이벤트(한정 오리파, 룰렛 등) 목록 |

**POST /events/attendance/check-in — 응답 예시**
```json
{
  "streak_count": 5,
  "calendar_day_number": 5,
  "reward_type": "bonus_credit",
  "reward_value": 100,
  "wallet_balance_after": { "paid": 46500, "bonus": 2100 }
}
```
- 중복 체크인 시 `409 ALREADY_CHECKED_IN`. 캘린더 마지막 날(예: 7일차·30일차)은 `reward_type=free_draw_ticket`으로 무료 뽑기권을 지급하도록 관리자가 구성 가능(§ERD.md §8 `attendance_calendar_configs`).

**GET /missions — 응답 예시**
```json
{
  "data": [
    { "id": 12, "code": "daily_first_draw", "name": "오늘 첫 뽑기", "progress_count": 0, "target_count": 1, "status": "in_progress", "reward_type": "bonus_credit", "reward_value": 300 },
    { "id": 15, "code": "n_draws_in_day_3", "name": "오늘 3회 뽑기", "progress_count": 3, "target_count": 3, "status": "completed", "reward_type": "free_draw_ticket", "reward_value": 1 }
  ]
}
```

**POST /missions/{id}/claim — 응답 예시**
```json
{ "mission_id": 15, "status": "reward_claimed", "granted": { "type": "free_draw_ticket", "ticket_id": 5510, "expires_at": "2026-08-01T00:00:00Z" } }
```
- 미완료 상태에서 청구 시 `409 MISSION_NOT_COMPLETED`, 이미 청구했으면 `409 REWARD_ALREADY_CLAIMED`.

**GET /season-pass/current — 응답 예시**
```json
{
  "season_pass_id": 3,
  "name": "2026 서머 시즌패스",
  "xp": 4200,
  "current_level": 14,
  "premium_purchased": false,
  "next_level_xp": 4500,
  "rewards_preview": [
    { "level": 15, "track": "free", "reward_type": "bonus_credit", "reward_value": 200, "claimable": false },
    { "level": 15, "track": "premium", "reward_type": "free_draw_ticket", "reward_value": 1, "claimable": false, "locked_reason": "premium_not_purchased" }
  ]
}
```

**POST /season-pass/premium/purchase — 요청/응답 예시**
```json
// Request
{ }
// 200 Response
{ "premium_purchased": true, "amount_charged": 9900, "wallet_balance_after": { "paid": 36600, "bonus": 2100 } }
```
- 실패: `409 SEASON_PASS_ALREADY_PURCHASED`, `402 INSUFFICIENT_BALANCE`.

**POST /season-pass/rewards/{level}/claim — 실패 케이스**
- 프리미엄 트랙 미구매 상태에서 청구 시 `403 SEASON_PASS_PREMIUM_REQUIRED`, 아직 도달하지 않은 레벨이면 `409 LEVEL_NOT_REACHED`.

**GET /users/me/loyalty — 응답 예시**
```json
{
  "tier_code": "gold",
  "cumulative_charge_amount": 1250000,
  "next_tier": { "tier_code": "vip", "min_cumulative_charge": 3000000, "remaining_amount": 1750000 },
  "benefits": { "shipping_discount_rate": 0.5, "vip_only_packs": true }
}
```
- `cumulative_charge_amount`는 유상 충전액 기준(§ERD.md §7·§8) — 환불된 금액은 차감되어 등급이 강등될 수 있음.

**GET /users/me/draw-tickets — 응답 예시**
```json
{ "data": [ { "id": 5510, "source_type": "mission", "pack_scope": "any", "status": "available", "expires_at": "2026-08-01T00:00:00Z" } ] }
```

**POST /coupons/redeem — 응답 예시**
```json
// Request
{ "code": "WELCOME2026" }

// 200 Response
{ "coupon_type": "fixed_points", "value": 3000, "wallet_balance_after": { "paid": 46500, "bonus": 5100 } }
```
- 쿠폰 지급분은 `bonus_credit_grants`(`source_type=coupon`)로 적립되어 무상 적립금 취급(환불 대상 아님).
- 실패: `404 COUPON_NOT_FOUND`, `409 COUPON_ALREADY_REDEEMED`, `410 COUPON_EXPIRED`.

---

## 7. Admin Service

모든 엔드포인트 `role=admin` 이상 + 2FA 세션 필수. 성공/실패 관계없이 `audit_logs` 기록.

### 7.1 카드 마스터 관리 & 데이터 수집 파이프라인
| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/cards` | 카드 단건 등록 |
| PATCH | `/admin/cards/{id}` | 카드 수정 (`locked_fields`에 잠글 필드 지정 가능) |
| POST | `/admin/cards/bulk-import` | CSV 대량 업로드 (소스 없는 게임 대응) |
| POST | `/admin/card-sets` | 세트 등록 |
| POST | `/admin/card-sync/{source}/trigger` | 수집 파이프라인 수동 실행 (`source`: pokemontcg_io, ygoprodeck, crawler_onepiece, crawler_ws, crawler_unionarena) |
| GET | `/admin/card-sync/jobs` | 동기화 실행 이력 조회 (`?status=failed`) |
| GET | `/admin/card-sync/review-queue` | 자동 반영 보류된 변경분 조회 |
| POST | `/admin/card-sync/review-queue/{id}/approve` | 리뷰 항목 승인 → `cards`에 반영 |
| POST | `/admin/card-sync/review-queue/{id}/reject` | 리뷰 항목 반려 |

**GET /admin/card-sync/review-queue — 응답 예시**
```json
{
  "data": [
    {
      "id": 501,
      "sync_job_id": 88,
      "card_id": 5821,
      "change_type": "field_update",
      "diff": { "rarity": { "old": "SR", "new": "SAR" } },
      "status": "pending"
    }
  ]
}
```
- 신규 카드 대량 등록(예: 확장팩 발매), 가격/희귀도 급변 등 임계치를 초과하는 변경은 자동 반영되지 않고 이 큐에 쌓인다 (§ERD.md §10 참고). 단순 오탈자·이미지 URL 갱신 등은 즉시 반영되어 큐에 나타나지 않는다.
- `cards.locked_fields`로 잠근 필드는 수집 파이프라인이 애초에 diff 대상에서 제외 — 관리자가 수동 보정한 값이 다음 동기화에서 덮어써지지 않음.

### 7.2 실물 재고 관리
| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/physical-cards` | 실물 카드 개체 입고 등록 (그레이딩/상태 정보 포함) |
| POST | `/admin/physical-cards/{id}/images` | 실물 촬영 이미지 업로드 (자동 리사이즈/워터마크) |
| GET | `/admin/physical-cards?status=in_stock` | 재고 조회 |
| POST | `/admin/physical-cards/stocktake` | 재고 실사 등록 (DB-실물 불일치 보정) |

### 7.3 오리파 팩 관리
| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/oripa-packs` | 팩 생성 (draft) — 등급별 구성, 총 구수, 구당 가격 |
| POST | `/admin/oripa-packs/{id}/slots` | 슬롯-실물 매핑 대량 등록 |
| POST | `/admin/oripa-packs/{id}/publish` | 셔플 실행 + `server_seed_hash` 커밋, `status=on_sale` 전환 |
| POST | `/admin/oripa-packs/{id}/settle` | `server_seed` 공개(reveal), `status=settled` 전환 |

**POST /admin/oripa-packs/{id}/publish — 응답 예시**
```json
{
  "id": 42,
  "status": "on_sale",
  "server_seed_hash": "b7e2...ff01",
  "published_at": "2026-07-17T09:00:00Z"
}
```
- 이 시점 이후 슬롯-실물 매핑은 불변(수정 API 없음, 오직 `settle`에서 reveal만 가능) — 조작 원천 차단.

### 7.4 유저/재무 관리
| Method | Path | 설명 |
|---|---|---|
| GET | `/admin/users` | 유저 목록/검색 |
| GET | `/admin/users/{id}` | 유저 상세 (연결 SNS 계정, 변경 이력 요약 포함) |
| PATCH | `/admin/users/{id}` | 관리자에 의한 유저 정보 강제 수정 (닉네임/이메일/전화번호/배송지) |
| PATCH | `/admin/users/{id}/status` | 계정 정지/복구 |
| GET | `/admin/users/{id}/change-logs` | 해당 유저의 전체 변경 이력(본인+관리자) 조회 |
| GET | `/admin/refund-requests?status=pending` | 환급 신청 대기열 |
| POST | `/admin/refund-requests/{id}/approve` | 환급 승인 (ledger 반영) |
| POST | `/admin/shipping-requests/{id}/ship` | 송장번호 등록, 배송 처리 |

**PATCH /admin/users/{id} — 요청/응답 예시**
```json
// Request
{ "phone": "010-1234-5678", "reason": "CS 요청에 따른 배송지 연락처 정정" }

// 200 Response
{ "id": 205, "phone_masked": "010-****-5678", "updated_at": "2026-07-17T11:00:00Z" }
```
- 본인 확인 절차(이메일/SMS 인증) 없이 즉시 반영되지만, `reason` 필드는 필수이며 `user_change_logs.changed_by='admin'` + `admin_user_id` + `audit_logs`에 이중 기록.

### 7.5 대시보드 & 감사
| Method | Path | 설명 |
|---|---|---|
| GET | `/admin/dashboard/overview` | 전체 매출/가입/뽑기 지표 |
| GET | `/admin/dashboard/packs/{id}/pl` | 팩별 매출-원가(`acquired_cost` 합산)-마진 |
| GET | `/admin/audit-logs` | 관리자 액션 감사 로그 조회 |

### 7.6 위탁 심사 관리
| Method | Path | 설명 |
|---|---|---|
| GET | `/admin/consignment-submissions?status=submitted` | 위탁 심사 대기열 |
| POST | `/admin/consignment-submissions/{id}/approve` | 위탁 승인 — 정산가/수수료율 확정, `physical_cards`에 반영 |
| POST | `/admin/consignment-submissions/{id}/reject` | 위탁 반려 |
| GET | `/admin/settlement-holds?status=held` | 정산 보류 현황 조회 |
| POST | `/admin/settlement-holds/{id}/release` | 정산 보류 수동 해제 (예외 처리용) |
| GET | `/admin/delivery-disputes?status=open` | 배송 이의제기 대기열 |
| POST | `/admin/delivery-disputes/{id}/resolve` | 분쟁 처리 (재배송/환급/무과실/위탁자 귀책) |

### 7.7 리텐션/유인 정책 관리
| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/charge-bonus-rules` | 충전수단별 보너스 규칙 생성/수정 (계좌이체 유도용) |
| PATCH | `/admin/charge-bonus-rules/{id}` | 규칙 활성화/비활성화, 요율 변경 |
| POST | `/admin/attendance-calendar-configs` | 출석 캘린더 N일차 보상 구성 |
| POST | `/admin/mission-definitions` | 미션 템플릿 생성 (트리거/보상 정의) |
| PATCH | `/admin/mission-definitions/{id}` | 미션 활성화/비활성화, 보상 변경 |
| POST | `/admin/season-passes` | 시즌패스 생성 (시즌 기간, 프리미엄 가격, 최대 레벨) |
| POST | `/admin/season-passes/{id}/rewards` | 레벨별 무료/프리미엄 트랙 보상 등록 |
| POST | `/admin/loyalty-tiers` | 로열티 등급(누적 충전 기준) 생성/수정 |
| POST | `/admin/draw-tickets/grant` | 특정 유저에게 무료 뽑기권 수동 지급 (CS 보상용) |

**POST /admin/charge-bonus-rules — 요청 예시**
```json
{
  "method": "transfer",
  "bonus_rate": 0.03,
  "min_charge_amount": 10000,
  "max_bonus_amount": 5000,
  "valid_from": "2026-08-01T00:00:00Z",
  "valid_to": "2026-08-31T23:59:59Z",
  "is_active": true
}
```
- 카드 결제 대비 계좌이체의 PG 수수료 절감분(§5.1)을 재원으로 하므로, `bonus_rate`는 절감폭 이내로 설정해 마진을 해치지 않도록 관리자가 직접 통제한다.

---

## 8. Consignment & Settlement Service

위탁자(개인 판매자)가 카드를 출품하고 정산받는 전체 흐름. 구매자 쪽 배송/검수 API는 §4에 있으며, 이 섹션은
위탁자 온보딩과 정산 파이프라인을 다룬다. 데이터 모델은 `docs/ERD.md` §11 참고.

| Method | Path | Auth | 설명 |
|---|---|---|---|
| POST | `/consignors/apply` | Bearer | 위탁자 신청 (본인인증, 정산 계좌 등록) |
| GET | `/consignors/me` | Bearer(위탁자) | 내 위탁자 프로필/상태 조회 |
| POST | `/consignors/me/consignment-submissions` | Bearer(위탁자) | 카드 위탁 출품 신청 (사진 + 희망가) |
| GET | `/consignors/me/consignment-submissions` | Bearer(위탁자) | 내 출품 이력/심사 상태 |
| GET | `/consignors/me/wallet` | Bearer(위탁자) | 정산 대기(pending) / 출금 가능(available) 잔액 조회 |
| GET | `/consignors/me/ledger` | Bearer(위탁자) | 정산 원장 이력 (보류/해제/지급) |
| POST | `/consignors/me/payouts` | Bearer(위탁자) | 출금(정산) 요청 |
| GET | `/consignors/me/payouts/{id}` | Bearer(위탁자) | 출금 처리 상태 조회 |

**정산 보류 → 해제 → 출금 흐름**
```mermaid
sequenceDiagram
    participant Buyer as 구매자
    participant API as Oripa/Shipping API
    participant DB as PostgreSQL
    participant Consignor as 위탁자

    Buyer->>API: POST /oripa-packs/{id}/draws (위탁 카드가 걸림)
    API->>DB: draw_logs INSERT + settlement_holds INSERT(status=held)
    API->>DB: consignor_ledger_entries INSERT(sale_hold, +hold_amount)
    Note over DB: consignor_wallets.pending_balance 증가, available_balance는 그대로
    Buyer->>API: (수일 후) 배송 완료
    Buyer->>API: POST /shipping-requests/{id}/confirm-receipt (또는 검수기한 자동 경과)
    API->>DB: settlement_holds.status=released
    API->>DB: consignor_ledger_entries INSERT(sale_release)
    Note over DB: pending_balance -> available_balance 이동
    Consignor->>API: POST /consignors/me/payouts
    API->>DB: consignor_payouts INSERT(status=pending)
    API->>Consignor: 포트원 파트너 정산 자동화 API 호출 → 세금계산서 발행 + 계좌 입금
    API->>DB: consignor_payouts.status=paid, ledger에 payout(-amount) 기록
```

**POST /consignors/me/consignment-submissions — 요청/응답 예시**
```json
// Request
{
  "card_id": 5821,
  "requested_price": 15000,
  "images": ["https://cdn.../front.jpg", "https://cdn.../back.jpg"]
}

// 201 Response
{ "id": 902, "status": "submitted" }
```

**GET /consignors/me/wallet — 응답 예시**
```json
{ "pending_balance": 12750, "available_balance": 3400, "updated_at": "2026-07-17T12:00:00Z" }
```

**POST /consignors/me/payouts — 요청/응답 예시**
```json
// Request
{ "amount": 3400 }

// 202 Response
{ "id": 550, "amount": 3400, "withholding_tax_amount": 112, "net_amount": 3288, "payout_status": "pending" }
```
- 실패: `422 INSUFFICIENT_AVAILABLE_BALANCE`(요청 금액이 `available_balance` 초과), `403 PAYOUT_ACCOUNT_NOT_VERIFIED`(정산 계좌 미인증).
- 실제 계좌 이체는 플랫폼이 직접 수행하지 않고 **포트원 파트너 정산 자동화 서비스**(1순위, 세금계산서 자동발행 포함)를 경유한다 — 위탁 거래량이 커지면 토스페이먼츠 지급대행(월 정액 30만원/1,000건)과 손익분기점을 재계산해 전환을 검토한다. 직접 계좌이체 자체 구현은 전자금융거래법상 지급대행업 등록 이슈로 금지(§5.1 참고).

---

## 9. WebSocket — 실시간 뽑기 피드

`wss://api.<domain>/ws/feed`

- 인증 불필요(공개 피드), 연결 시 최근 20건 스냅샷 전송 후 실시간 브로드캐스트.
- 개인정보 보호를 위해 닉네임은 마스킹(`ot***_hunter`), 배송지 등 민감정보 미포함.

```json
// Server → Client push
{
  "type": "draw",
  "pack_name": "포켓몬 SAR 이상 확정 오리파",
  "nickname_masked": "ot***_hunter",
  "grade": "S",
  "card_name": "리자몽 ex SAR",
  "timestamp": "2026-07-17T10:03:21Z"
}
```
- 클라이언트 → 서버 메시지는 없음(단방향 push). 재연결 시 `Last-Event-ID` 유사 커서로 유실 구간 보정(`GET /oripa-packs/{id}/draws?since=`로 폴백 조회 가능).

---

## 10. 에러 코드 표준

| HTTP | code | 설명 |
|---|---|---|
| 400 | `VALIDATION_ERROR` | 요청 스키마 검증 실패 |
| 401 | `INVALID_CREDENTIALS` / `TOKEN_EXPIRED` | 인증 실패 |
| 402 | `INSUFFICIENT_BALANCE` | 포인트 잔액 부족 |
| 403 | `FORBIDDEN` | 권한 부족 (role/2FA 미충족) |
| 404 | `NOT_FOUND` | 리소스 없음 |
| 409 | `PACK_SOLD_OUT` / `ALREADY_CHECKED_IN` / `COUPON_ALREADY_REDEEMED` | 상태 충돌 |
| 410 | `COUPON_EXPIRED` | 만료된 리소스 사용 시도 |
| 422 | `AMOUNT_MISMATCH` | PG 웹훅 금액 불일치 |
| 423 | `PACK_LOCKED` | 동시 뽑기 락 획득 실패 (재시도 유도) |
| 428 | `CAPTCHA_REQUIRED` | 반복 실패로 캡차 요구 |
| 429 | `RATE_LIMITED` | 레이트 리밋 초과 |
| 500 | `INTERNAL_ERROR` | 서버 오류 |
| 401 | `INVALID_PASSWORD` | 민감 정보 변경 시 현재 비밀번호 재확인 실패 |
| 409 | `EMAIL_ALREADY_IN_USE` | 이메일 변경/SNS 연결 시 대상 이메일이 이미 사용 중 |
| 422 | `INVALID_CODE` | 이메일/전화번호 변경 인증 코드 불일치 |
| 429 | `TOO_MANY_ATTEMPTS` | 인증 코드 시도 횟수 초과, 재발급 필요 |
| 409 | `LAST_LOGIN_METHOD` | 마지막 남은 로그인 수단(SNS 계정) 연결 해제 시도 |
| 422 | `INSUFFICIENT_AVAILABLE_BALANCE` | 위탁자 출금 요청 금액이 `available_balance` 초과 |
| 403 | `PAYOUT_ACCOUNT_NOT_VERIFIED` | 위탁자 정산 계좌 미인증 상태에서 출금 시도 |
| 409 | `DISPUTE_ALREADY_OPEN` | 이미 열린 이의제기가 있는 배송 건에 중복 제기 |
| 410 | `INSPECTION_WINDOW_EXPIRED` | 검수 기간 경과 후 이의제기 시도 (자동 확정된 건) |
| 409 | `CONSIGNMENT_NOT_APPROVED` | 심사 승인 전 위탁 카드가 팩에 배치되려는 시도 |
| 409 | `TICKET_NOT_AVAILABLE` | 이미 사용/만료된 무료 뽑기권으로 뽑기 시도 |
| 422 | `TICKET_SCOPE_MISMATCH` | 무료 뽑기권의 `pack_scope`와 대상 팩 불일치 |
| 409 | `MISSION_NOT_COMPLETED` | 미완료 미션 보상 청구 시도 |
| 409 | `REWARD_ALREADY_CLAIMED` | 이미 수령한 보상 재청구 시도 |
| 409 | `SEASON_PASS_ALREADY_PURCHASED` | 이미 구매한 시즌패스 프리미엄 재구매 시도 |
| 403 | `SEASON_PASS_PREMIUM_REQUIRED` | 프리미엄 미구매 상태에서 프리미엄 트랙 보상 청구 |
| 409 | `LEVEL_NOT_REACHED` | 아직 도달하지 않은 시즌패스 레벨 보상 청구 |
