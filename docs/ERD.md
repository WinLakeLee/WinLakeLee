# TCG 오리파 플랫폼 — ERD (데이터 모델 설계)

프로젝트 계획서(§2, §3, §4, §5, §7)를 기반으로 한 엔티티 관계 설계. DB는 PostgreSQL 16을 기준으로 하며,
게임별 가변 속성은 JSONB로, 상태값은 Postgres ENUM 타입으로 표현한다. 모든 테이블은 `id BIGINT
GENERATED ALWAYS AS IDENTITY PK`, `created_at timestamptz DEFAULT now()`를 공통으로 가진다(표에서는
생략하지 않고 명시).

## 목차

1. [전체 도메인 개요](#1-전체-도메인-개요)
2. [Auth & User](#2-auth--user)
3. [Card Master Data](#3-card-master-data)
4. [Physical Inventory & Grading](#4-physical-inventory--grading)
5. [Oripa (뽑기) Engine](#5-oripa-뽑기-engine)
6. [User Inventory & Shipping](#6-user-inventory--shipping)
7. [Payment & Wallet (복식 원장)](#7-payment--wallet-복식-원장)
8. [Events & Retention](#8-events--retention)
9. [Admin & Audit](#9-admin--audit)
10. [Card Data Collection Pipeline (운영 데이터 수집)](#10-card-data-collection-pipeline-운영-데이터-수집)
11. [Consignment & Marketplace Escrow (위탁 판매 & 정산 에스크로)](#11-consignment--marketplace-escrow-위탁-판매--정산-에스크로)
12. [Offline Store Check-in & Anti-Farming (오프라인 매장 체크인 & 채굴 방지)](#12-offline-store-check-in--anti-farming-오프라인-매장-체크인--채굴-방지)
13. [Marketplace — 단품/묶음 판매 & 경매](#13-marketplace--단품묶음-판매--경매)
14. [설계 노트](#14-설계-노트)

---

## 1. 전체 도메인 개요

```mermaid
graph LR
    A[Auth & User] -->|user_id FK| F[Oripa Engine]
    A -->|user_id FK| G[Inventory & Shipping]
    A -->|user_id FK| H[Payment & Wallet]
    A -->|user_id FK| I[Events & Retention]
    A -->|admin_user_id FK| J[Admin & Audit]

    B[Card Master Data] -->|card_id FK| C[Physical Inventory & Grading]
    C -->|physical_card_id FK| F
    F -->|draw_log| G
    H -->|ledger debit| F
    G -->|refund_request| H
    I -->|coupon/attendance credit| H

    K[Consignment & Escrow] -->|consignor_id FK, settlement_price| C
    F -->|draw_log triggers hold| K
    G -->|delivery confirmed / dispute| K
    K -->|payout| L[External Bank Account]
```

- **Auth & User**: 전 도메인의 기준(anchor) 엔티티인 `users`를 소유.
- **Card Master Data → Physical Inventory**: 카드 마스터 1건에 실물 개체(`physical_cards`)가 N건 매핑(동일 카드라도 그레이딩/상태가 다른 개체가 여러 장 존재).
- **Physical Inventory → Oripa Engine**: 팩의 각 슬롯(`oripa_slots`)은 실물 카드 1장에 1:1로 고정 매핑되어 오버셀을 원천 차단.
- **Oripa Engine → Inventory & Shipping**: 뽑기 결과가 `user_inventory`로 이동, 이후 배송 또는 환급 분기.
- **Payment & Wallet**: 모든 포인트 증감(충전/뽑기 차감/환급/이벤트 지급)은 `ledger_entries`에 원자적으로 기록되며 `wallets.balance`는 파생 캐시.
- **Consignment & Escrow**: 위탁받은 실물 카드가 뽑히면(`draw_log`) 정산금이 즉시 지급되지 않고 `settlement_holds`에 보류되며, 배송 완료+검수 기간 통과 또는 구매자 분쟁 여부에 따라 위탁자에게 지급(payout)되거나 보류가 취소된다.

---

## 2. Auth & User

```mermaid
erDiagram
    USERS ||--o{ USER_ADDRESSES : has
    USERS ||--o{ REFRESH_TOKENS : has
    USERS ||--o| USER_2FA : has
    USERS ||--o{ USER_OAUTH_ACCOUNTS : links
    USERS ||--o{ USER_CHANGE_LOGS : "audited by"
    USERS ||--o{ VERIFICATION_CODES : requests

    USERS {
        bigint id PK
        varchar email UK "SNS 전용 가입은 provider 이메일로 채움, nullable 아님"
        varchar password_hash "nullable, SNS 전용 계정은 NULL, Argon2id"
        varchar nickname UK
        varchar phone_encrypted "AES-256 (KMS), nullable"
        varchar signup_method "email, kakao, naver, google, apple — 최초 가입 경로(통계/정책용, 불변)"
        varchar role "user, admin, super_admin"
        varchar status "active, suspended, withdrawn"
        timestamptz last_login_at
        timestamptz created_at
        timestamptz updated_at
    }

    USER_ADDRESSES {
        bigint id PK
        bigint user_id FK
        varchar recipient_name
        varchar phone_encrypted
        varchar zipcode
        varchar address1_encrypted
        varchar address2_encrypted
        boolean is_default
        timestamptz created_at
    }

    REFRESH_TOKENS {
        bigint id PK
        bigint user_id FK
        varchar token_hash UK "SHA-256(token), raw token never stored"
        varchar device_info
        varchar ip_address
        timestamptz expires_at
        timestamptz revoked_at
        timestamptz created_at
    }

    USER_2FA {
        bigint id PK
        bigint user_id FK
        varchar secret_encrypted "TOTP seed, KMS-encrypted"
        boolean enabled
        timestamptz enabled_at
    }

    USER_OAUTH_ACCOUNTS {
        bigint id PK
        bigint user_id FK
        varchar provider "kakao, naver, google, apple"
        varchar provider_user_id UK "provider+provider_user_id 복합 UNIQUE"
        varchar email_from_provider
        boolean email_verified_by_provider
        timestamptz connected_at
        timestamptz last_used_at
    }

    USER_CHANGE_LOGS {
        bigint id PK
        bigint user_id FK
        varchar field "email, phone, password, nickname, address, status, role ..."
        varchar old_value_masked
        varchar new_value_masked
        varchar changed_by "self, admin"
        bigint admin_user_id FK "changed_by=admin일 때만"
        varchar ip_address
        timestamptz created_at
    }

    VERIFICATION_CODES {
        bigint id PK
        bigint user_id FK
        varchar purpose "email_change, phone_change, signup_email, password_reset"
        varchar target "변경 대상 이메일/전화번호"
        varchar code_hash "SHA-256(code), 평문 미저장"
        int attempt_count
        timestamptz expires_at "발급 후 5~10분"
        timestamptz verified_at
        timestamptz created_at
    }
```

**설계 포인트**
- Refresh Token Rotation: 매 갱신 시 기존 토큰 `revoked_at` 세팅 후 신규 행 삽입. 탈취 감지를 위해 재사용 시도 시 해당 유저의 전체 세션 강제 만료.
- 배송지는 컬럼 단위 암호화(AES-256, KMS). 인덱싱이 필요한 zipcode만 평문 유지.
- `role`은 admin 전용 라우트 게이트에 사용, 세분화된 권한은 §9 `admin_permissions` 참고.
- **회원 탈퇴(`status=withdrawn`) 처리 규칙**: (a) `wallets.paid_balance > 0`이면 탈퇴 전 §7 현금 환불(`payment_refunds`) 절차를 먼저 안내하고, 유저가 명시적으로 포기 동의한 잔액만 소멸 처리(동의 기록을 `user_change_logs`에 보존). (b) 미배송 `user_inventory`(`status=held`)가 있으면 배송 신청 또는 즉시환급 완료 전까지 탈퇴 차단. (c) 위탁자(`consignors`)로서 `settlement_holds.status=held`가 남아 있으면 정산 완료 전까지 탈퇴 차단. (d) 탈퇴 확정 시 개인정보(이메일·전화·주소·닉네임)는 즉시 비식별화(해시 대체)하되, 전자상거래법상 거래기록(결제·원장·뽑기 로그)은 5년 보존 — 로그의 `user_id` FK는 유지하고 식별 컬럼만 파기하는 가명처리 방식.
- **SNS 계정은 `user_oauth_accounts`로 분리**(1:N) — 기존 `users.oauth_provider/oauth_id` 단일 컬럼 설계로는 한 유저가 카카오+네이버를 동시에 연결할 수 없어 확장. `UNIQUE(provider, provider_user_id)`로 동일 SNS 계정의 중복 유저 생성을 방지.
- **계정 연결(Account Linking)**: SNS 최초 로그인 시 `email_from_provider`가 기존 `users.email`과 일치하고 `email_verified_by_provider = true`인 경우에만 자동 연결 제안. 그렇지 않으면 신규 계정으로 분리 생성(이메일 스푸핑 방지) — §API 명세 3.항 참고.
- 이메일/전화번호 변경, 비밀번호 변경 등 민감 정보 수정은 `verification_codes`로 2단계 인증 후 확정하며, 성공 여부와 무관하게 `user_change_logs`에 감사 기록(본인 변경 포함 — 계정 탈취 대응 근거자료).

---

## 3. Card Master Data

```mermaid
erDiagram
    GAMES ||--o{ CARD_SETS : contains
    CARD_SETS ||--o{ CARDS : contains
    CARDS ||--o{ CARD_IMAGES : has
    CARDS ||--o{ CARD_MARKET_PRICES : "priced by"
    GAMES ||--o{ PRODUCTS : offers

    GAMES {
        bigint id PK
        varchar code UK "pokemon, onepiece, yugioh, ws, unionarena"
        varchar name
        text description
        boolean is_active
        timestamptz created_at
    }

    CARD_SETS {
        bigint id PK
        bigint game_id FK
        varchar set_code
        varchar name
        varchar language "ko, ja, en"
        date release_date
        timestamptz created_at
    }

    CARDS {
        bigint id PK
        bigint card_set_id FK
        varchar card_number
        varchar name
        varchar rarity
        jsonb attributes "게임별 가변 속성, §14 참고"
        varchar source "api_sync, crawl, manual"
        timestamptz synced_at
        timestamptz created_at
    }

    CARD_IMAGES {
        bigint id PK
        bigint card_id FK
        varchar image_type "official_front, official_back"
        varchar url
        int sort_order
    }

    PRODUCTS {
        bigint id PK
        bigint game_id FK
        varchar type "box, starter_deck, sleeve, accessory"
        varchar name
        text description
        timestamptz created_at
    }

    CARD_MARKET_PRICES {
        bigint id PK
        bigint card_id FK
        numeric reference_price "그레이딩 PSA10/최상급 기준 시세"
        varchar source "tcgplayer, cardrush_ja, manual_admin"
        timestamptz fetched_at
    }
```

**설계 포인트**
- `cards.attributes`(JSONB)로 게임별 스키마 차이를 흡수 — 별도 마이그레이션 없이 신규 게임 추가 가능.
- `(card_set_id, card_number)` 복합 UNIQUE로 동일 세트 내 중복 등록 방지.
- `source`로 API 동기화/크롤링/수동 입력 이력을 구분해 배치 재동기화 시 수동 보정 데이터 덮어쓰기 방지.
- `card_market_prices`는 TCGPlayer·일본 토레카 시세 등 외부 시세를 주기적으로 동기화(§10 데이터 수집 파이프라인과 유사한 배치)해 카드 단위 최신 시세를 유지 — §6 즉시 시세 환급, 원래 계획서 §8 "카드 시세 연동" 기능의 기반 데이터가 된다. 뽑기 확정 시점(§5)에 `fetched_at`이 오래된(신뢰 만료) 시세만 존재하면 §6 `instant_exchange_policies.max_price_age_hours`에 의해 스냅샷을 남기지 않고, 그 카드는 이후 환급 요청 시 자동으로 수동 심사로 넘어간다.

---

## 4. Physical Inventory & Grading

```mermaid
erDiagram
    CARDS ||--o{ PHYSICAL_CARDS : "instantiated as"
    PHYSICAL_CARDS ||--o{ PHYSICAL_CARD_IMAGES : has

    PHYSICAL_CARDS {
        bigint id PK
        bigint card_id FK
        varchar serial_no UK "내부 재고 관리 번호"
        boolean is_graded
        varchar grading_company "PSA, BGS, CGC, ARS, null"
        varchar grading_score "10, 9.5 ..."
        varchar grading_cert_no
        varchar condition_grade "S/A/B/C 또는 NM/LP/MP/HP/DMG"
        text condition_note
        varchar status "in_stock, allocated, shipped, refunded, lost, cleared"
        varchar ownership_type "platform_owned, consigned — §11 참고"
        bigint consignor_id FK "ownership_type=consigned일 때만, §11 CONSIGNORS 참조"
        numeric settlement_price "위탁 카드가 뽑혀 확정될 때 위탁자에게 지급할 금액, consigned 전용"
        numeric acquired_cost "매입 원가(platform_owned), 마진 계산용"
        boolean clearance_eligible "장기 체화/비인기 재고 — §8 출석 럭키박스 실물 보상 풀에 편입 가능"
        timestamptz created_at
    }

    PHYSICAL_CARD_IMAGES {
        bigint id PK
        bigint physical_card_id FK
        varchar image_type "front, back, defect_closeup, grading_label"
        varchar url
        int sort_order
        timestamptz created_at
    }
```

**설계 포인트**
- `physical_cards`가 실물 재고의 **단일 진실 공급원(SSOT)**. `oripa_slots.physical_card_id`가 여기를 참조하므로 슬롯:실물 = 1:1이 DB 레벨(UNIQUE 제약)로 강제됨.
- `status = allocated`는 팩에 배치되었지만 아직 안 뽑힌 상태, `shipped`/`refunded`는 §6 인벤토리 처리 결과와 동기화.
- 실물 촬영 이미지는 관리자 업로드 시 리사이즈·워터마크 파이프라인(Celery)을 거쳐 `physical_card_images`에 다건 저장.
- `ownership_type=consigned` 카드는 `consignor_id`/`settlement_price`가 채워지며, 뽑힌 뒤 정산 보류(에스크로) 흐름을 탄다 — 상세 설계는 §11.
- `clearance_eligible=true`(관리자가 장기 미판매·비인기 카드에 수동 표시)인 `in_stock` 카드는 오리파 팩에 배치하는 대신 §8 출석 럭키박스의 실물 보상 풀로 돌려 재고를 소진할 수 있다 — 지급 시 `status=cleared`로 전환.

---

## 5. Oripa (뽑기) Engine

```mermaid
erDiagram
    GAMES ||--o{ ORIPA_PACKS : "themed for"
    ORIPA_PACKS ||--o{ ORIPA_SLOTS : contains
    PHYSICAL_CARDS ||--|| ORIPA_SLOTS : "allocated 1:1"
    USERS ||--o{ DRAW_LOGS : performs
    ORIPA_PACKS ||--o{ DRAW_LOGS : records
    ORIPA_SLOTS ||--|| DRAW_LOGS : "consumed as"

    ORIPA_PACKS {
        bigint id PK
        bigint game_id FK
        varchar name
        text description
        int total_slots
        numeric price_per_draw
        varchar status "draft, on_sale, sold_out, settled"
        boolean last_one_bonus_enabled
        bigint last_one_bonus_physical_card_id FK "라스트원 상 실물(§4), enabled=true면 필수 — 슬롯과 별도로 사전 확보"
        int max_draws_per_request "1회 요청 최대 연차 수, 예: 10 (다연차 뽑기 상한)"
        varchar server_seed_hash "SHA-256(server_seed), on_sale 전 공개(commit)"
        varchar server_seed "sold_out 후 공개(reveal), 그 전까지 NULL"
        timestamptz published_at
        timestamptz revealed_at
        bigint created_by FK
        timestamptz created_at
    }

    ORIPA_SLOTS {
        bigint id PK
        bigint pack_id FK
        int slot_no UK "pack_id + slot_no 복합 UNIQUE"
        varchar grade "SS, S, A, participation"
        bigint physical_card_id FK UK "실물 1:1, 슬롯 재사용 불가"
        varchar status "available, drawn"
        bigint drawn_by_user_id FK
        timestamptz drawn_at
    }

    DRAW_LOGS {
        bigint id PK
        bigint user_id FK
        bigint pack_id FK
        bigint slot_id FK UK
        varchar draw_group_id "다연차 뽑기 묶음 UUID — 한 요청으로 N회 뽑으면 N행이 동일값 공유, 단건이면 단독값"
        varchar payment_method "points, free_ticket"
        numeric points_spent "payment_method=points일 때만"
        bigint free_draw_ticket_id FK "payment_method=free_ticket일 때만, §8 참조"
        varchar random_proof "CSPRNG 선택 인덱스 + 잔여슬롯수, 감사용"
        varchar ledger_entry_ref FK "ledger_entries.id, nullable(무료뽑기권 소비 시)"
        timestamptz drawn_at
    }
```

**설계 포인트 — Provably Fair**
1. **커밋(Commit)**: 팩 생성 시 `secrets` 기반 Fisher-Yates로 슬롯↔등급↔실물 매핑을 셔플하고, 전체 매핑 데이터 + `server_seed`의 해시(`server_seed_hash = SHA256(server_seed || mapping)`)를 `on_sale` 전환 시점에 선공개.
2. **소비**: 개별 뽑기는 `SELECT ... FOR UPDATE SKIP LOCKED` + Redis 분산 락(`pack:{id}:lock`)으로 동시성 제어 후, 잔여 `available` 슬롯 중 `secrets.randbelow(remaining_count)`로 균등 추출 → 오버셀·중복 뽑기 불가.
3. **리빌(Reveal)**: `status = settled` 전환 시 `server_seed` 공개 → 누구나 최초 커밋 해시와 대조해 매핑 조작 여부 검증 가능 (API: `GET /oripa-packs/{id}/proof`).
4. `draw_logs`는 append-only(UPDATE/DELETE 금지, DB 권한으로 강제) 감사 로그.
5. `payment_method=free_ticket`인 뽑기는 `free_draw_tickets.status`를 `used`로 전환하고 포인트 차감이 없다(§8 이벤트 보상 연동).
6. `user_inventory` 생성과 같은 트랜잭션 안에서 `card_market_prices`의 그 시점 최신 시세를 `locked_reference_price`로 스냅샷한다(§6) — 즉시 시세 환급 금액이 이후 시세 변동과 무관하게 뽑힌 순간의 값으로 고정되도록 하는 지점이 바로 여기다.
7. **다연차 뽑기**: 한 요청으로 최대 `max_draws_per_request`(예: 10)회를 뽑을 수 있다. 총액 잔액 검증 → N개 슬롯 소비 → N건의 `draw_logs`(동일 `draw_group_id`) 생성을 **전부 하나의 트랜잭션**으로 처리 — 부분 성공 없음(잔여 슬롯이 요청 수보다 적으면 전체 거부, `409 INSUFFICIENT_REMAINING_SLOTS`). 슬롯 선택은 단건과 동일하게 잔여 슬롯에서 `secrets` 기반 비복원 추출이므로 Provably Fair 검증 방식도 동일하다.
8. **라스트원 상**: `last_one_bonus_enabled=true`인 팩의 **마지막 슬롯**을 소진한 뽑기 트랜잭션 안에서, `last_one_bonus_physical_card_id`의 실물을 같은 유저의 `user_inventory`에 추가로 삽입(`source=last_one_bonus`)하고 팩을 `sold_out`으로 전환한다 — 별도 배치가 아닌 동일 트랜잭션이라 "마지막 구 당첨자 판정" 경합이 존재하지 않는다. 라스트원 실물은 슬롯 밖에서 사전 확보(`physical_cards.status=allocated`)되어 커밋 해시에는 포함되지 않되 팩 상세에 공개된다.

---

## 6. User Inventory & Shipping

```mermaid
erDiagram
    USERS ||--o{ USER_INVENTORY : owns
    PHYSICAL_CARDS ||--|| USER_INVENTORY : "held as"
    USERS ||--o{ SHIPPING_REQUESTS : requests
    USER_ADDRESSES ||--o{ SHIPPING_REQUESTS : "ships to"
    SHIPPING_REQUESTS ||--o{ SHIPPING_REQUEST_ITEMS : contains
    USER_INVENTORY ||--o| SHIPPING_REQUEST_ITEMS : included
    USER_INVENTORY ||--o| REFUND_REQUESTS : "refunded via"
    USER_INVENTORY ||--o| DELIVERY_DISPUTES : "disputed via"
    CARD_MARKET_PRICES ||--o{ USER_INVENTORY : "snapshotted into"

    USER_INVENTORY {
        bigint id PK
        bigint user_id FK
        bigint physical_card_id FK UK
        varchar source "draw, event, attendance_lucky_box, last_one_bonus, listing_purchase, auction_won"
        bigint source_ref_id "draw_logs.id, listing_orders.id 등"
        varchar status "held, shipping_requested, shipped, delivered, confirmed, disputed, refunded"
        numeric locked_reference_price "획득(뽑기) 시점 card_market_prices 스냅샷 — 즉시환급 계산은 이 값만 사용, 이후 시세변동 미반영"
        bigint locked_price_source_id FK "스냅샷 당시 참조한 CARD_MARKET_PRICES 레코드, 감사용"
        timestamptz locked_price_fetched_at "스냅샷된 시세의 원 fetched_at"
        timestamptz acquired_at
    }

    SHIPPING_REQUESTS {
        bigint id PK
        bigint user_id FK
        bigint address_id FK
        varchar status "requested, preparing, shipped, delivered, confirmed, cancelled"
        varchar carrier
        varchar tracking_number
        timestamptz requested_at
        timestamptz shipped_at
        timestamptz delivered_at
        timestamptz inspection_deadline_at "delivered_at + N일, 위탁 카드 정산 보류 해제 기준(§11)"
        timestamptz confirmed_at "구매자 명시적 수령 확인 또는 기한 경과 자동 확정"
    }

    DELIVERY_DISPUTES {
        bigint id PK
        bigint user_inventory_id FK
        bigint user_id FK
        varchar reason "wrong_item, condition_mismatch, damaged, not_received"
        jsonb evidence "구매자 제출 사진/설명"
        varchar status "open, resolved_reship, resolved_refund, resolved_no_fault, resolved_penalize_consignor"
        bigint resolved_by FK
        timestamptz resolved_at
        timestamptz created_at
    }

    SHIPPING_REQUEST_ITEMS {
        bigint id PK
        bigint shipping_request_id FK
        bigint user_inventory_id FK UK
    }

    REFUND_REQUESTS {
        bigint id PK
        bigint user_inventory_id FK "부분 UNIQUE: 활성 상태(pending/approved/paid)만 — rejected 후 재신청 허용"
        bigint user_id FK
        numeric refund_points
        varchar valuation_method "instant_market_price, manual_review"
        numeric priced_from_locked_reference_price "계산에 쓰인 user_inventory.locked_reference_price 그대로 복사(감사용) — 요청 시점 시세를 다시 조회하지 않음"
        numeric buy_rate_applied "적용된 매입률, 예: 0.8"
        varchar status "pending, approved, rejected, paid"
        timestamptz requested_at
        timestamptz processed_at
    }

    INSTANT_EXCHANGE_POLICIES {
        bigint id PK
        numeric buy_rate "시세 대비 매입률, 예: 0.8 = 시세의 80%"
        jsonb condition_multipliers "그레이딩/컨디션별 배율, 예: {\"PSA10\":1.0,\"PSA9\":0.7,\"NM\":0.5,\"DMG\":0.05}"
        int max_price_age_hours "뽑기 확정 시점 기준 이 시간 초과한 시세는 스냅샷하지 않고 manual_review로 전환"
        boolean is_active
    }
```

**설계 포인트**
- `user_inventory.physical_card_id`에 UNIQUE 제약 → 동일 실물 카드가 두 유저 인벤토리에 동시 존재 불가.
- 배송 신청은 여러 인벤토리 항목을 하나의 `shipping_requests`로 묶어(§8 "배송 통합") 배송비 절감.
- **수령 확인/검수 기간**: `delivered_at` 이후 `inspection_deadline_at`까지 구매자가 이의를 제기하지 않으면 자동으로 `confirmed_at`이 채워진다(또는 구매자가 직접 조기 확정). 이 시점이 위탁 카드의 정산 보류(`settlement_holds`) 해제 트리거다 — 상세는 §11.
- `delivery_disputes`는 위탁/플랫폼 소유 카드 모두에 적용되는 공통 이의제기 창구이며, `consignor_id`가 있는 케이스는 처리 결과에 따라 §11의 정산 보류가 취소·차감될 수 있다.

**설계 포인트 — 즉시 시세 환급 (뽑은 카드를 배송 대신 바로 현금화)**
- **기준가는 뽑힌 시점에 고정(스냅샷)한다.** 오리파 뽑기가 확정되는 순간(§5) `card_market_prices`의 그 시점 최신값을 `user_inventory.locked_reference_price`로 즉시 복사해둔다 — 이후 실제 시세가 오르든 내리든, 그 카드의 즉시환급 계산은 **오직 이 스냅샷 값만** 사용한다. 요청 시점에 시세를 다시 조회하지 않으므로, 유저가 카드를 오래 들고 있다가 "시세가 올랐으니" 더 높은 금액을 받아가는 차익거래가 원천적으로 불가능하다(반대로 시세가 내려도 손해를 보지 않는다 — 결과적으로 플랫폼도 유저도 시세 변동에 노출되지 않는 중립적 구조).
- 환급 요청 시엔 `locked_reference_price`에 `instant_exchange_policies.condition_multipliers`(§4 `condition_grade`/`grading_score` 기준)와 `buy_rate`(예: 0.8 — KREAM·스니커즈덩크류 리셀 플랫폼이 시세보다 낮게 매입하는 것과 같은 원리로 플랫폼 마진 확보)를 곱해 최종 지급액을 산출한다. `refund_requests.priced_from_locked_reference_price`에 계산에 쓰인 스냅샷 값을 그대로 복사해 감사 가능하게 남긴다.
- 뽑기 확정 시점에 참조할 시세 데이터 자체가 없으면(`card_market_prices` 미존재) `locked_reference_price`는 NULL로 남고, 이후 환급 요청은 자동으로 `valuation_method=manual_review`(관리자 심사)로 전환된다 — 스냅샷이 없는 카드는 즉시 처리 대상이 아니다.
- `valuation_method=instant_market_price`인 요청은 생성과 동시에 `status=paid`로 확정되고 `ledger_entries`에 `refund_credit` 항목이 즉시 기록된다(§7) — 관리자 승인 대기 없이 단일 DB 트랜잭션으로 종료.
- `refund_credit`은 **환급 포인트(`balance_type=exchange`)로 적립된다 — 뽑기·마켓 구매에는 자유롭게 쓸 수 있지만 현금 인출은 불가**(§7). 뽑기 결과물이 현금으로 되돌아가는 환전 경로를 차단해 사행성 판단 리스크를 낮추는 구조적 장치이며, 일본 오리파 플랫폼들의 표준 관행과도 일치한다. 무상 적립금과도 구분(만료 없음, §12 무료 채굴 방지 상한 계산에 미포함).

---

## 7. Payment & Wallet (복식 원장)

포인트는 **유상(paid) / 환급(exchange) / 무상(bonus)** 세 버킷으로 분리한다.
- **paid**: 실제 충전한 돈 — 전자상거래법상 청약철회·현금 환불 대상.
- **exchange**: 뽑은 카드를 즉시 시세 환급(§6)해 받은 포인트 — **뽑기·마켓 구매에는 자유롭게 사용 가능하지만 현금 인출은 불가**. 뽑기 결과물이 현금으로 되돌아가는 경로(환금성)를 끊어 사행행위 판단 리스크를 구조적으로 낮추는 핵심 장치(§13 법적 리스크 노트 참고). 일본 오리파 플랫폼들의 "환급 포인트는 재뽑기 전용" 표준과 동일한 구조.
- **bonus**: 무상 적립금(출석/미션/이벤트/계좌이체 보너스) — 환불 대상 아니며 만료 있음.

이 구분이 없으면 "이벤트로 받은 적립금까지 현금 환불해줘야 하는" 법적 리스크와, "뽑기→환급→현금인출"이라는 사실상의
환전 구조(사행성 핵심 징표)가 동시에 생긴다.

```mermaid
erDiagram
    USERS ||--|| WALLETS : owns
    WALLETS ||--o{ LEDGER_ENTRIES : records
    WALLETS ||--o{ BONUS_CREDIT_GRANTS : "accrues"
    WALLETS ||--o{ WALLET_HOLDS : "locked by"
    USERS ||--o{ PAYMENT_TRANSACTIONS : makes
    PAYMENT_TRANSACTIONS ||--o| CHARGE_BONUS_RULES : "may apply"
    PAYMENT_TRANSACTIONS ||--o{ PAYMENT_REFUNDS : "cancelled via"
    USERS ||--o{ PAYMENT_REFUNDS : requests

    WALLETS {
        bigint id PK
        bigint user_id FK UK
        numeric paid_balance "유상 잔액(충전액), 현금 환불 대상, 파생 캐시"
        numeric exchange_balance "즉시환급 포인트 — 사용 가능·현금 인출 불가, 파생 캐시"
        numeric bonus_balance "무상 적립금, 환불 불가, 파생 캐시"
        timestamptz updated_at
    }

    LEDGER_ENTRIES {
        bigint id PK
        bigint wallet_id FK
        varchar entry_type "charge, draw_spend, listing_purchase, refund_credit, bonus_grant, bonus_expired, adjustment, withdrawal, hold_capture"
        varchar balance_type "paid, exchange, bonus — 어느 버킷에 영향을 주는지"
        numeric amount "부호: +적립 / -차감"
        numeric balance_after
        varchar ref_type "payment_transaction, draw_log, listing_order, refund_request, coupon_redemption, bonus_credit_grant"
        bigint ref_id
        varchar idempotency_key UK
        timestamptz created_at
    }

    WALLET_HOLDS {
        bigint id PK
        bigint wallet_id FK
        varchar hold_type "auction_bid"
        bigint ref_id "auction_bids.id"
        numeric amount "홀드 금액 — 사용가능잔액 = 잔액합 - 활성 홀드합"
        varchar status "held, released, captured"
        timestamptz created_at
        timestamptz resolved_at
    }

    BONUS_CREDIT_GRANTS {
        bigint id PK
        bigint wallet_id FK
        varchar source_type "attendance, mission, transfer_bonus, coupon, offline_checkin, admin_grant"
        bigint source_ref_id
        numeric amount
        numeric remaining_amount "소진 추적, 뽑기 시 무상 우선 차감(FIFO by expires_at)"
        timestamptz expires_at
        timestamptz created_at
    }

    CHARGE_BONUS_RULES {
        bigint id PK
        varchar method "transfer, card, kakaopay, naverpay"
        numeric bonus_rate "예: 0.03 = 충전액의 3% 적립"
        numeric min_charge_amount
        numeric max_bonus_amount "1회 최대 적립 한도"
        timestamptz valid_from
        timestamptz valid_to
        boolean is_active
        timestamptz created_at
    }

    PAYMENT_TRANSACTIONS {
        bigint id PK
        bigint user_id FK
        varchar pg_provider "portone(통합 게이트웨이, 기본값)"
        varchar sub_pg "nhn_kcp, kg_inicis, toss, kakaopay — 실제 라우팅된 하위 PG, API_SPEC.md §5.1 참고"
        varchar pg_tid UK
        numeric amount
        numeric refunded_amount "누적 부분취소 합계, 기본 0 — amount 초과 불가"
        varchar method "card, transfer, kakaopay, naverpay"
        varchar status "pending, approved, failed, cancelled, partially_refunded, refunded"
        bigint applied_bonus_rule_id FK
        jsonb webhook_payload
        timestamptz requested_at
        timestamptz approved_at
    }

    PAYMENT_REFUNDS {
        bigint id PK
        bigint user_id FK
        bigint payment_transaction_id FK "취소 대상 원거래"
        numeric refund_amount "이 거래에서 취소하는 금액 (부분취소 가능)"
        numeric clawed_back_bonus "원거래에 딸려 지급됐던 충전 보너스 중 회수분"
        varchar pg_refund_tid UK "PG 취소 거래번호 — 중복 취소 방지"
        varchar status "requested, processing, completed, failed"
        varchar idempotency_key UK
        timestamptz requested_at
        timestamptz completed_at
    }
```

**설계 포인트**
- `wallets.paid_balance`/`bonus_balance`는 성능을 위한 캐시이며, 정합성 검증 배치가 주기적으로 `SUM(ledger_entries.amount) GROUP BY balance_type`과 대조.
- 모든 포인트 증감은 `ledger_entries` 삽입 + `wallets` 갱신을 **하나의 DB 트랜잭션**으로 처리.
- `idempotency_key` UNIQUE 제약으로 PG 웹훅 재전송/중복 충전 요청을 DB 레벨에서 차단.
- `payment_transactions.pg_tid` UNIQUE로 동일 PG 거래 중복 승인 방지.
- **계좌이체 유도**: `charge_bonus_rules`에 `method=transfer` 규칙을 활성화하면, 계좌이체 충전 승인 시 `bonus_rate`만큼 `bonus_credit_grants`(`source_type=transfer_bonus`)가 자동 지급된다. 계좌이체 PG 수수료(약 1.8%)가 카드(3.4%+)보다 낮으므로(§API_SPEC §5.1), 절감분 일부를 즉시 무상 적립금으로 돌려주는 구조 — 환불 시에는 이 무상분은 회수되고 유상 충전액만 환불된다.
- **소진 우선순위**: 뽑기(`draw_spend`)·마켓 구매 시 `bonus_balance` → `exchange_balance` → `paid_balance` 순으로 차감(bonus는 만료 임박 `bonus_credit_grants`부터 FIFO) — 유저 입장엔 만료·인출불가 재화가 먼저 없어지는 게 유리하고, 플랫폼 입장엔 현금 환불 의무가 있는 유상 잔액을 최대한 보존할 수 있다.
- **경매 입찰 홀드**: 입찰 시 입찰액만큼 `wallet_holds`(`status=held`)로 잠가 사용가능잔액에서 제외 — 상회 입찰이 나오면 `released`, 낙찰 확정 시 `captured`로 전환하며 실제 원장 차감(`hold_capture`)이 일어난다(§13). 이중 입찰·잔액 초과 입찰을 구조적으로 차단.
- `bonus_credit_grants.expires_at` 경과분은 배치가 `remaining_amount`를 0으로 만들며 `ledger_entries`에 `bonus_expired` 기록.

**설계 포인트 — 유상 잔액 현금 환불(청약철회, `payment_refunds`)**
- **환불 가능액 = 현재 `paid_balance`만** — 이미 뽑기에 소진된 금액은 물론, `exchange_balance`(뽑기 결과 환급 포인트)와 `bonus_balance`도 현금 환불 대상에서 제외된다. exchange를 제외하는 것이 환금성 차단(§7 서두, §13 법적 노트)의 핵심이다. 환불 요청이 오면 미취소 잔액이 남은 충전 건들 중 **최신 건부터 역순(LIFO)**으로 PG 부분취소를 배분한다 — 오래된 거래일수록 PG 취소 가능 기한(카드사 정책상 통상 수개월)을 넘겼을 확률이 높기 때문.
- **보너스 회수(clawback)**: 취소되는 충전 건에 딸려 지급됐던 `charge_bonus_rules` 보너스(`bonus_credit_grants`)는 취소 비율만큼 회수한다. 이미 소진해서 회수할 무상 잔액이 부족하면 그 부족분을 환불액에서 차감(`clawed_back_bonus`) — "충전 보너스만 챙기고 원금은 환불"하는 어뷰징 차단.
- 처리 순서(단일 트랜잭션 + PG 호출): `ledger_entries`에 `withdrawal`(-refund_amount, `balance_type=paid`) 선기록 → PG 취소 API 호출 → 성공 시 `payment_refunds.status=completed` / 실패 시 원장 역분개(보상 트랜잭션) 후 `failed`. `pg_refund_tid` UNIQUE + `idempotency_key`로 중복 취소 방지.
- 계좌이체 충전 건 등 PG 취소가 불가한 결제수단은 지급대행 벤더(§API_SPEC §5.1)를 통한 계좌 환불로 폴백하며, 이 경우 처리 SLA가 다름을 유저에게 고지.
- 관리자 대시보드에서 환불률·환불 사유를 모니터링(§API_SPEC §7.4) — 단기간 대량 충전 후 환불 반복은 §계획서 §5.2 이상거래 탐지 룰의 입력이 된다.

---

## 8. Events & Retention

```mermaid
erDiagram
    USERS ||--o{ ATTENDANCE_LOGS : checks_in
    ATTENDANCE_REWARD_POLICIES ||--o{ ATTENDANCE_LOGS : "generates reward via"
    ATTENDANCE_REWARD_POLICIES ||--o{ ATTENDANCE_LUCKY_BOX_OUTCOMES : defines
    USERS ||--o{ COUPON_REDEMPTIONS : redeems
    COUPONS ||--o{ COUPON_REDEMPTIONS : "redeemed by"
    USERS ||--o{ REFERRALS : refers
    USERS ||--o{ RANKING_SNAPSHOTS : ranked
    USERS ||--o{ USER_MISSIONS : progresses
    MISSION_DEFINITIONS ||--o{ USER_MISSIONS : defines
    USERS ||--|| USER_LOYALTY_STATUS : has
    LOYALTY_TIERS ||--o{ USER_LOYALTY_STATUS : "achieved by"
    USERS ||--o{ FREE_DRAW_TICKETS : holds
    USERS ||--o{ WISHLISTS : keeps
    CARDS ||--o{ WISHLISTS : "wished as"
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--|| USER_NOTIFICATION_SETTINGS : configures
    USERS ||--o{ USER_COLLECTION_ENTRIES : collects
    CARDS ||--o{ USER_COLLECTION_ENTRIES : "recorded in"
    LOYALTY_TIERS ||--o{ COSMETIC_ITEMS : "unlocks at"
    USERS ||--o{ USER_COSMETIC_UNLOCKS : owns
    COSMETIC_ITEMS ||--o{ USER_COSMETIC_UNLOCKS : "unlocked as"
    USERS ||--|| USER_PROFILE_EQUIPMENT : has

    ATTENDANCE_LOGS {
        bigint id PK
        bigint user_id FK
        bigint policy_id FK "보상 계산에 사용된 attendance_reward_policies 버전 — 정책 변경 후에도 과거 지급 근거 재현 가능"
        date check_in_date UK "user_id+date 복합 UNIQUE"
        int streak_count
        int attendance_day_number "가입/최근 리셋 이후 누적 출석일수 — 마일스톤(10/20일차 등) 판정 기준"
        varchar reward_source "dynamic, milestone, lucky_box"
        varchar reward_type "bonus_credit, free_draw_ticket"
        numeric reward_value
        numeric activity_score_snapshot "보상 계산에 쓰인 가중합 점수, 재현/감사용"
        timestamptz created_at
    }

    ATTENDANCE_REWARD_POLICIES {
        bigint id PK
        numeric base_reward_amount "활동 없는 최소 기준 보상"
        numeric payment_recency_weight "최근 결제 이력 반영 가중치"
        numeric streak_weight "연속 출석일수 반영 가중치"
        numeric engagement_weight "뽑기 횟수/미션 완료 등 기타 이용 반영 가중치"
        int decay_start_days "무결제 상태 지속 시 감소가 시작되는 시점"
        int decay_window_days "감소가 바닥값까지 도달하는 기간 — 길게 잡아 하루 변화폭을 체감 못하게 함"
        numeric decay_floor_multiplier "바닥값, 예: 0.7 — 뚝 떨어지지 않고 은근히만 줄어듦"
        jsonb milestone_days "예: [7, 10, 20, 30, 50, 100]"
        numeric milestone_bonus_multiplier "마일스톤 날 보상 배율"
        int long_term_free_threshold_days "이 일수 이상 무결제 지속 시 럭키박스 방식으로 전환"
        boolean long_term_free_lucky_box_enabled
        numeric lucky_box_target_ev_ratio "럭키박스 기댓값 목표 비율, 예: 0.85 = 그 시점 decay 적용 동적 보상의 85%. 반드시 1.0 미만"
        boolean is_active
        timestamptz valid_from
        timestamptz valid_to
    }

    ATTENDANCE_LUCKY_BOX_OUTCOMES {
        bigint id PK
        bigint policy_id FK
        varchar outcome_code "small, medium, jackpot"
        numeric probability "policy_id 내 합계 1.0"
        varchar reward_type "bonus_credit, free_draw_ticket, physical_card_clearance"
        numeric reward_value "reward_type=physical_card_clearance일 땐 EV 계산용 참고가(§4 clearance 풀의 평균 시세)"
        boolean is_active
    }

    COUPONS {
        bigint id PK
        varchar code UK
        varchar type "fixed_points, percent_bonus, free_draw"
        numeric value
        timestamptz valid_from
        timestamptz valid_to
        int max_uses
        int per_user_limit
        timestamptz created_at
    }

    COUPON_REDEMPTIONS {
        bigint id PK
        bigint coupon_id FK
        bigint user_id FK
        timestamptz redeemed_at
    }

    REFERRALS {
        bigint id PK
        bigint referrer_user_id FK
        bigint referred_user_id FK UK
        varchar status "pending, rewarded"
        timestamptz created_at
        timestamptz rewarded_at
    }

    RANKING_SNAPSHOTS {
        bigint id PK
        bigint user_id FK
        varchar period_type "daily, weekly, monthly"
        varchar period_key "예: 2026-W29"
        numeric score
        int rank
        timestamptz computed_at
    }

    MISSION_DEFINITIONS {
        bigint id PK
        varchar code UK
        varchar name
        varchar trigger_type "first_charge, first_draw, daily_login, n_draws_in_day, invite_friend, transfer_charge"
        int target_count "예: n_draws_in_day=3"
        varchar reward_type "bonus_credit, free_draw_ticket"
        numeric reward_value
        timestamptz valid_from
        timestamptz valid_to
        boolean is_active
    }

    USER_MISSIONS {
        bigint id PK
        bigint user_id FK
        bigint mission_id FK
        int progress_count
        varchar status "in_progress, completed, reward_claimed"
        timestamptz completed_at
        timestamptz reward_claimed_at
    }

    LOYALTY_TIERS {
        bigint id PK
        varchar tier_code UK "basic, bronze, silver, gold, vip — 순수 결제 등급 사다리"
        numeric min_cumulative_charge "누적 유상 충전액 기준, basic=0"
        jsonb benefits "배송비 할인율, 전용 이벤트, 등급전용 팩 접근 등 — 결제 이력과 무관한 출석 보상과는 별개"
    }

    USER_LOYALTY_STATUS {
        bigint id PK
        bigint user_id FK UK
        bigint tier_id FK
        numeric cumulative_charge_amount "누적 유상 충전액 — 결제등급과 출석 동적 보상 계산 양쪽의 입력값"
        numeric cumulative_free_offline_bonus "오프라인 체크인으로 받은 무상 적립금 누적, §12 채굴방지 규칙에 사용"
        timestamptz last_charge_at "가장 최근 유상 충전 시각 — 출석 보상 decay 계산 기준"
        timestamptz tier_achieved_at
        timestamptz updated_at
    }

    COSMETIC_ITEMS {
        bigint id PK
        varchar code UK
        varchar name
        varchar item_type "profile_frame, profile_badge, animated_deco"
        varchar asset_format "webp, gif, png"
        varchar asset_url
        bigint min_tier_required FK "이 등급 이상 달성 시 자동 unlock, nullable(이벤트 전용 아이템)"
        boolean is_active
        timestamptz created_at
    }

    USER_COSMETIC_UNLOCKS {
        bigint id PK
        bigint user_id FK
        bigint cosmetic_item_id FK
        varchar source "tier_achieved, event, admin_grant — 출석 경로 없음(§8 참고)"
        bigint source_ref_id
        timestamptz unlocked_at
    }

    USER_PROFILE_EQUIPMENT {
        bigint id PK
        bigint user_id FK UK
        bigint equipped_frame_id FK "COSMETIC_ITEMS 참조, nullable"
        bigint equipped_badge_id FK "nullable"
        bigint equipped_avatar_deco_id FK "nullable"
        timestamptz updated_at
    }

    FREE_DRAW_TICKETS {
        bigint id PK
        bigint user_id FK
        varchar source_type "attendance, mission, coupon, admin_grant"
        bigint source_ref_id
        varchar pack_scope "any, game:pokemon, pack:{id} — 사용 가능 범위"
        varchar status "available, used, expired"
        timestamptz expires_at
        timestamptz used_at
        timestamptz created_at
    }

    WISHLISTS {
        bigint id PK
        bigint user_id FK
        bigint card_id FK "user_id+card_id 복합 UNIQUE"
        boolean notify_on_pack_open "이 카드 포함 오리파 오픈 시 알림"
        timestamptz created_at
    }

    NOTIFICATIONS {
        bigint id PK
        bigint user_id FK
        varchar type "wishlist_pack_open, restock, shipping_update, settlement_released, dispute_resolved, marketing"
        varchar title
        text body
        varchar deep_link "앱/웹 이동 경로"
        varchar channel "in_app, push, email"
        timestamptz read_at
        timestamptz created_at
    }

    USER_NOTIFICATION_SETTINGS {
        bigint id PK
        bigint user_id FK UK
        boolean push_enabled
        boolean email_enabled
        boolean marketing_opt_in "수신동의 — 정보통신망법상 명시적 동의 필수, 동의/철회 시각 기록"
        timestamptz marketing_opt_in_at
        timestamptz updated_at
    }

    USER_COLLECTION_ENTRIES {
        bigint id PK
        bigint user_id FK
        bigint card_id FK "user_id+card_id 복합 UNIQUE — 도감 등록"
        int acquired_count "누적 획득 횟수 (환급/배송과 무관하게 유지)"
        timestamptz first_acquired_at
        timestamptz created_at
    }
```

**설계 포인트**
- `attendance_logs`는 `(user_id, check_in_date)` UNIQUE로 하루 중복 출석 방지, `streak_count`는 전일 미출석 시 배치/트리거로 리셋.
- `ranking_snapshots`는 실시간 집계 대신 주기적 배치 스냅샷(랭킹 조작·부하 방지), 실시간 뽑기 피드는 별도 WebSocket 채널(§API 명세 참고)로 처리하며 영속 저장하지 않음.
- **미션 시스템**: `mission_definitions`는 관리자가 트리거 조건과 보상을 정의하는 템플릿, `user_missions`가 유저별 진행도를 추적. 뽑기/충전/친구초대 등 이벤트 발생 시 해당 유저의 `user_missions.progress_count`를 증가시키고 `target_count` 도달 시 `completed`로 전환(보상은 별도 청구 API로 명시적 수령 — 자동 지급하지 않아 "받았는지 모르고 방치" 방지 및 UX 상 보상 연출 여지 확보).
- **무료 뽑기권**: `free_draw_tickets`는 출석/미션/쿠폰으로 지급되는 "뽑기 1회 무료" 아이템. `pack_scope`로 특정 게임/팩에만 쓰게 제한 가능. 뽑기 API(`POST /oripa-packs/{id}/draws`)가 `ticket_id`를 받으면 포인트 차감 대신 이 티켓을 소비 — 상세는 API_SPEC.md §3, §6 참고.
- **위시리스트 & 알림**: 유저가 `wishlists`에 담은 카드가 신규 오리파 팩 슬롯에 포함되어 `on_sale` 전환되는 순간, 팩 발행 트랜잭션 후속 Celery 잡이 해당 카드를 위시한 유저 전원에게 `notifications`(`type=wishlist_pack_open`)를 생성하고 푸시 발송 — 원 계획서 §8 "위시리스트 & 재입고 알림". `user_notification_settings.marketing_opt_in`이 false인 유저에게는 `marketing` 타입만 차단하고 거래성 알림(배송/정산)은 항상 발송(정보통신망법상 광고성/거래성 정보 구분).
- **컬렉션 도감**: 뽑기 확정 트랜잭션에서 `user_collection_entries`를 upsert(`acquired_count += 1`) — 카드를 환급·배송해도 도감 기록은 남는다(수집 이력이므로). 세트별 완성률(`도감 등록 수 / 세트 총 카드 수`)은 조회 시 계산하며, 완성 시 미션(`mission_definitions`) 트리거로 뱃지·보상 연계 가능 — 원 계획서 §7 "컬렉션 도감".

**설계 포인트 — 활동 기반 동적 출석 보상 & "눈치채지 못하게" 줄어드는 무결제 보상**
- 고정된 날짜별 보상 테이블 대신, 매 체크인마다 `attendance_reward_policies`의 가중치로 그 유저의 **결제 이력**(`last_charge_at`/`cumulative_charge_amount`), **연속 출석일수**(`streak_count`), **기타 이용도**(뽑기 횟수, 완료한 미션 수 등)를 합산한 `activity_score`를 계산해 보상을 그때그때 산출한다. `attendance_logs.activity_score_snapshot`에 계산근거를 남겨 재현·감사가 가능하다.
- **서서히, 감지 못하게 감소**: 무결제 상태가 `decay_start_days`(예: 15일)를 넘기면 그 즉시 확 줄이는 게 아니라, `decay_window_days`(예: 90일)에 걸쳐 서서히 `decay_floor_multiplier`(예: 0.7)까지 하락시킨다 — 하루 변화폭이 1% 미만이라 유저가 "오늘 갑자기 줄었다"고 체감하지 못한다. 절벽형 등급 강등은 두지 않는다.
- **마일스톤 스파이크**: `milestone_days`(예: 7·10·20·30·50·100일차)에는 결제 여부와 무관하게 `milestone_bonus_multiplier`를 곱한 고가치 보상을 지급해 장기 이탈을 방지하고 "다음 마일스톤까지 채워야지" 하는 동기를 유지시킨다.
- **장기 무결제자 = 확정형 → 확률형(럭키박스) 전환**: `long_term_free_threshold_days`를 넘긴 무결제 유저는 보상 방식이 `attendance_lucky_box_outcomes`의 확률 테이블로 전환된다. 예: 88% 확률로 소액, 10% 확률로 중간값, 2% 확률로 잭팟(무료뽑기권) — **기댓값 자체도 그 시점의 decay 적용 동적 보상보다 살짝 낮게**(`lucky_box_target_ev_ratio`, 예: 0.85) 설계한다. 즉 평균적으로는 이전보다 조금 덜 받지만, 분산이 커서 "가끔 크게 받을 수도 있다"는 기대감만 남기고 매일 조금씩 깎이는 느낌은 주지 않는다 — 기댓값을 부풀리지 않으면서도 체감상의 지루한 하락 대신 변동성으로 흥미를 유지하는 것이 핵심.
- **비인기 재고 소진 채널**: `reward_type=physical_card_clearance`를 쓰면 잭팟/중간 등급 보상을 현금성 재화 대신 `physical_cards.clearance_eligible=true`(§4, 장기 체화·비인기 카드)인 실물로 지급할 수 있다 — 유저에게는 "실물 카드 당첨"이라는 체감가치 큰 이벤트지만, 플랫폼 입장에서는 팔리지 않던 재고를 원가 이상 지출 없이 정리하는 효과가 있어 `lucky_box_target_ev_ratio` 제약을 지키면서도 매력적인 보상을 구성하기 쉽다. 당첨 시점에 `clearance_eligible` 재고가 소진되어 없으면 동일 EV의 `bonus_credit`으로 자동 대체 지급한다(지급 실패를 노출하지 않음).
- **결제 관련 혜택은 그대로**: 위 감소/럭키박스 전환은 오직 **출석 보상**에만 적용된다. 배송비 할인, 충전 보너스(§7 `charge_bonus_rules`), PG 결제 조건 등 결제와 관련된 혜택·조건은 결제 이력과 무관하게 모든 유저에게 동일하게 적용한다 — 장기 무결제자를 다른 영역에서까지 차별하지 않는다.

**설계 포인트 — 로열티 등급 & 코스메틱 (출석 보상과는 완전히 분리)**
- `loyalty_tiers`는 순수하게 **누적 유상 충전액** 기준의 결제 등급 사다리(`basic → bronze → silver → gold → vip`)이며, 위 출석 보상의 증감 로직과는 무관하다.
- **코스메틱은 등급 달성 시에만 자동 지급**: `cosmetic_items.min_tier_required` 등급에 도달하면 배치/트리거가 즉시 `user_cosmetic_unlocks`를 생성한다(`source=tier_achieved`) — 출석 체크인을 통한 코스메틱 지급 경로는 없다. 애니메이션(webp/gif) 프로필 장식처럼 원가 없는 "과시형" 보상을 결제 등급에만 묶어 결제 유도 효과를 낸다.
- `user_profile_equipment`는 보유(`user_cosmetic_unlocks`) 중 실제 착용 중인 것만 가리킨다.
- `user_cosmetic_unlocks`는 `(user_id, cosmetic_item_id)` UNIQUE로 중복 해금 방지(§14.3 참고).

---

## 9. Admin & Audit

```mermaid
erDiagram
    USERS ||--o{ AUDIT_LOGS : performs
    USERS ||--o| ADMIN_PERMISSIONS : has

    ADMIN_PERMISSIONS {
        bigint id PK
        bigint user_id FK UK
        varchar scope "card_admin, pack_admin, finance_admin, super_admin"
        timestamptz granted_at
    }

    AUDIT_LOGS {
        bigint id PK
        bigint admin_user_id FK
        varchar action "card.create, pack.settle, user.suspend, ..."
        varchar target_type
        bigint target_id
        jsonb before_state
        jsonb after_state
        varchar ip_address
        timestamptz created_at
    }
```

**설계 포인트**
- 관리자 액션(카드 등록, 팩 발행/정산, 유저 제재, 환급 승인 등)은 예외 없이 `audit_logs`에 before/after 스냅샷 기록.
- `audit_logs`도 append-only. 관리자 페이지는 별도 도메인/IP 화이트리스트 + 2FA 필수(§6 보안)이며 `admin_permissions.scope`로 세분화된 RBAC 적용.

---

## 10. Card Data Collection Pipeline (운영 데이터 수집)

카드 마스터 데이터(§3)를 최신 상태로 유지하기 위한 수집 파이프라인의 실행 이력·리뷰 큐 모델.
소스별 흐름과 API는 계획서 §3.1 및 `API_SPEC.md` §7.1을 참고.

```mermaid
erDiagram
    GAMES ||--o{ CARD_SYNC_JOBS : "synced by"
    CARD_SYNC_JOBS ||--o{ CARD_SYNC_REVIEW_ITEMS : produces
    CARDS ||--o| CARD_SYNC_REVIEW_ITEMS : "affects (nullable, 신규 카드는 NULL)"

    CARD_SYNC_JOBS {
        bigint id PK
        bigint game_id FK
        varchar source "pokemontcg_io, ygoprodeck, crawler_onepiece, crawler_ws, crawler_unionarena, manual_csv"
        varchar status "running, succeeded, failed, partially_failed"
        int records_fetched
        int records_created
        int records_updated
        int records_flagged "리뷰 큐로 넘어간 건수"
        text error_message
        timestamptz started_at
        timestamptz finished_at
    }

    CARD_SYNC_REVIEW_ITEMS {
        bigint id PK
        bigint sync_job_id FK
        bigint card_id FK "기존 카드 갱신 시에만, 신규 카드는 NULL"
        varchar change_type "new_card, field_update, price_spike, possible_duplicate"
        jsonb diff "필드별 old/new 값"
        varchar status "pending, approved, rejected"
        bigint reviewed_by FK
        timestamptz reviewed_at
        timestamptz created_at
    }
```

`cards` 테이블에는 필드 단위 수동 잠금을 위한 컬럼을 추가한다 (§3 원본 정의에 대한 보강):

```
CARDS.locked_fields  jsonb   -- 예: {"rarity": true, "attributes.hp": true} — true인 필드는 자동 동기화가 덮어쓰지 않음
```

**소스별 수집 전략**

| 게임 | 수집 방식 | 주기 | 비고 |
|---|---|---|---|
| Pokemon | PokemonTCG.io REST API | 매일 03:00 (Celery beat) | 공식 API, 페이지네이션 + rate limit 준수 |
| Yu-Gi-Oh! | YGOPRODeck API | 매일 03:10 | 공식 API |
| One Piece | 공식 카드리스트 크롤러 + 관리자 수동 보정 | 주 1회 + 신규 세트 발매 시 수동 트리거 | robots.txt/ToS 확인, 요청 간격 제한 |
| Weiss Schwarz | 공식 DB 크롤러 + 수동 | 주 1회 | 상동 |
| Union Arena | 공식 DB 크롤러 + 수동 | 주 1회 | 상동 |

**파이프라인 동작 원칙**
1. 소스 fetch → 정규화(공통 스키마 매핑) → 기존 `cards`와 필드 단위 diff 계산.
2. `locked_fields`에 잠긴 필드는 diff에서 제외(관리자 수동 보정값 보호).
3. diff 규모가 임계치(예: 신규 카드 50건 이상, 특정 필드 가격/희귀도 급변)를 넘으면 **자동 반영하지 않고** `card_sync_review_items`에 적재 후 관리자 승인 대기.
4. 임계치 이하의 단순 갱신(오탈자 수정, 이미지 URL 갱신 등)은 즉시 반영, `card_sync_jobs.records_updated`에 카운트.
5. 동기화 실패/rate-limit 초과 시 `status=failed` + 관리자 알림(Slack/이메일), 재시도는 다음 스케줄 또는 수동 트리거.

---

## 11. Consignment & Marketplace Escrow (위탁 판매 & 정산 에스크로)

개인/외부 판매자가 실물 카드를 위탁 출품하는 마켓플레이스형 거래를 지원하기 위한 도메인. 구매자 보호(§6 검수 기간)와
**위탁자(판매자) 보호**를 동시에 만족시키는 양방향 에스크로 구조 — "뽑히면 즉시 지급"이 아니라 "배송·검수 통과 후 지급"이
핵심이다.

```mermaid
erDiagram
    CONSIGNORS ||--o{ CONSIGNMENT_SUBMISSIONS : submits
    CONSIGNORS ||--o{ PHYSICAL_CARDS : "owns (consigned)"
    PHYSICAL_CARDS ||--o| SETTLEMENT_HOLDS : "triggers on draw"
    DRAW_LOGS ||--o| SETTLEMENT_HOLDS : creates
    DELIVERY_DISPUTES ||--o| SETTLEMENT_HOLDS : "may reverse"
    CONSIGNORS ||--|| CONSIGNOR_WALLETS : has
    CONSIGNOR_WALLETS ||--o{ CONSIGNOR_LEDGER_ENTRIES : records
    CONSIGNORS ||--o{ CONSIGNOR_PAYOUTS : "cashed out via"

    CONSIGNORS {
        bigint id PK
        bigint user_id FK UK "위탁자도 플랫폼 회원이어야 함"
        varchar display_name
        varchar real_name_encrypted "AES-256 (KMS), 본인확인용"
        varchar id_verification_status "unverified, verified"
        varchar bank_account_encrypted "정산 계좌, AES-256 (KMS)"
        varchar tax_id_encrypted "원천징수 신고용 식별정보, AES-256 (KMS)"
        varchar status "pending, active, suspended"
        timestamptz created_at
    }

    CONSIGNMENT_SUBMISSIONS {
        bigint id PK
        bigint consignor_id FK
        bigint card_id FK "카드 마스터, §3 참조"
        jsonb submitted_images "위탁자 제출 사진"
        numeric requested_price "위탁자 희망가"
        numeric admin_assessed_settlement_price "관리자 심사 후 확정 정산가"
        numeric commission_rate "플랫폼 수수료율, 예: 0.15"
        varchar status "submitted, under_review, approved, rejected, withdrawn"
        bigint reviewed_by FK
        timestamptz reviewed_at
        timestamptz created_at
    }

    SETTLEMENT_HOLDS {
        bigint id PK
        bigint physical_card_id FK UK
        varchar source_type "draw, listing_order — 오리파 뽑힘 또는 마켓 판매(§13) 모두 동일 에스크로"
        bigint source_ref_id "draw_logs.id 또는 listing_orders.id"
        bigint consignor_id FK
        numeric hold_amount "= admin_assessed_settlement_price × (1 - commission_rate); 경매 낙찰 시 낙찰가 기준"
        varchar status "held, released, disputed, reversed"
        timestamptz hold_expires_at "배송완료(delivered_at) + 검수기간, §6 inspection_deadline_at와 동기화"
        timestamptz released_at
        timestamptz created_at
    }

    CONSIGNOR_WALLETS {
        bigint id PK
        bigint consignor_id FK UK
        numeric pending_balance "정산 보류 중 합계, 파생 캐시"
        numeric available_balance "출금 가능 합계, 파생 캐시"
        timestamptz updated_at
    }

    CONSIGNOR_LEDGER_ENTRIES {
        bigint id PK
        bigint consignor_id FK
        varchar entry_type "sale_hold, sale_release, payout, dispute_reversal, commission_fee"
        numeric amount "부호 있음"
        varchar ref_type "settlement_hold, delivery_dispute, consignor_payout"
        bigint ref_id
        timestamptz created_at
    }

    CONSIGNOR_PAYOUTS {
        bigint id PK
        bigint consignor_id FK
        numeric amount
        numeric withholding_tax_amount "기타소득 원천징수 등, 세무 규정에 따름"
        numeric net_amount
        varchar payout_status "pending, processing, paid, failed"
        varchar pg_payout_ref "지급대행 벤더 참조번호 (포트원 파트너 정산 자동화 우선, API_SPEC.md §5.1 참고)"
        timestamptz requested_at
        timestamptz paid_at
    }
```

**설계 포인트 — 정산 에스크로 흐름**
1. **위탁 심사**: 위탁자가 `consignment_submissions`로 카드+사진+희망가 제출 → 관리자가 실물 확인/그레이딩 후 `admin_assessed_settlement_price`, `commission_rate` 확정 → 승인 시 `physical_cards`에 `ownership_type=consigned`, `consignor_id`, `settlement_price`로 반영(§4).
2. **보류 생성**: 위탁 카드가 오리파에서 뽑히는 순간(`draw_logs` 생성 시점) `settlement_holds`가 `status=held`로 즉시 생성되고, 동시에 `consignor_ledger_entries`에 `sale_hold`(+`hold_amount`, pending) 기록 → `consignor_wallets.pending_balance` 증가. **이 시점엔 위탁자에게 실제 지급되지 않는다** — 구매자가 아직 실물을 받지 못했기 때문.
3. **보류 해제**: 배송 완료 후 `shipping_requests.confirmed_at`이 채워지면(구매자 명시적 확인 또는 §6 `inspection_deadline_at` 경과 자동 확정) `settlement_holds.status=released` 전환, `consignor_ledger_entries`에 `sale_release` 기록 → `pending_balance`에서 `available_balance`로 이동.
4. **분쟁 시 처리**: 검수 기간 내 `delivery_disputes`가 열리면 보류 해제가 정지된다. 위탁자 귀책(그레이딩/사진과 실물 불일치 등)으로 판정되면 `settlement_holds.status=reversed` + `dispute_reversal` 기록으로 보류 취소, 구매자는 재배송/환급을 플랫폼 자체 재고 또는 별도 보상 재원으로 처리(위탁자에게 책임을 전가하되 구매자 피해를 위탁자 귀책 확정까지 기다리게 하지 않음).
5. **출금**: `available_balance`가 쌓인 위탁자는 `consignor_payouts` 요청 → **플랫폼이 직접 계좌 이체하지 않고 지급대행 벤더(`pg_payout_ref`)를 경유** — 1순위는 세금계산서 자동발행까지 지원하는 포트원 파트너 정산 자동화, 위탁 거래량이 커지면 토스페이먼츠 지급대행(월 정액제)과 재비교(API_SPEC.md §5.1). 플랫폼이 임의 계좌로 직접 송금하면 전자금융거래법상 지급대행업 등록 이슈가 발생할 수 있어 반드시 라이선스가 있는 벤더를 경유.
6. 개인 위탁자 대상 지급은 세법상 원천징수(예: 기타소득 3.3%) 대상일 가능성이 높아 `consignor_payouts.withholding_tax_amount`로 분리 계산 — 정확한 세율/신고 의무는 세무사 자문 필요.
7. **위탁 철회(반환) 규칙**: 승인된 위탁 실물은 `physical_cards.status=in_stock`(팩 미배치) 상태에서만 위탁자가 철회·반환 요청 가능. 슬롯에 배치(`allocated`)된 뒤에는 커밋 해시에 포함되어 매핑 불변이므로 철회 불가 — 해당 팩이 `settled`될 때까지 기다렸다가 미뽑힘으로 남은 경우에만 반환된다(반환 배송비는 위탁자 부담). 이 규칙은 위탁 신청 약관에 명시.

**법적 유의사항 (요약)**
- 위탁 판매 자체가 중고/위탁물품 취급 관련 별도 신고·등록 대상인지 사업 개시 전 법률 검토 필요(§리스크, 오리지널 계획서 §10).
- 위탁자 정산은 "지급대행 벤더 경유"가 원칙(포트원 파트너 정산 자동화 우선) — 직접 계좌이체 자체 구현 금지.
- 위탁자 개인정보(실명, 계좌, 세금 식별정보)는 §2와 동일하게 컬럼 단위 암호화(AES-256, KMS).

---

## 12. Offline Store Check-in & Anti-Farming (오프라인 매장 체크인 & 채굴 방지)

제휴 오프라인 카드샵 방문 시 무상 적립금을 지급해 오프라인 매장 유입을 유도하는 기능. **매장 측 QR 발급 기록**과
**고객 측 스캔 제출 기록**을 서로 다른 주체가 남기고 서버가 대조하는 이중 장부 구조로 실제 방문을 검증하며,
"돈은 안 쓰고 무료 적립금만 계속 채굴"하는 어뷰징을 막기 위한 누적 상한을 둔다.

```mermaid
erDiagram
    PARTNER_STORES ||--o{ STORE_QR_ISSUANCES : issues
    PARTNER_STORES ||--o{ STORE_CHECKINS : "visited via"
    STORE_QR_ISSUANCES ||--o| STORE_CHECKINS : "consumed by"
    USERS ||--o{ STORE_CHECKINS : performs
    STORE_CHECKINS ||--o{ OFFLINE_CHECKIN_FRAUD_FLAGS : "may flag"

    PARTNER_STORES {
        bigint id PK
        varchar name
        varchar address
        numeric latitude
        numeric longitude
        int geofence_radius_m "매장 반경, 예: 50m"
        varchar qr_secret_key "회전 QR 생성용 HMAC 시드"
        varchar status "pending, active, suspended, terminated"
        timestamptz created_at
    }

    STORE_QR_ISSUANCES {
        bigint id PK
        bigint store_id FK
        varchar qr_token UK "매장 태블릿/PC 화면에 표시되는 회전 토큰 — '장부 1' (매장측 기록)"
        timestamptz issued_at
        timestamptz expires_at "issued_at + 60초 등 짧은 주기로 재발급"
    }

    STORE_CHECKINS {
        bigint id PK
        bigint user_id FK
        bigint store_id FK
        bigint qr_issuance_id FK UK "스캔한 토큰, 1회만 소비(replay 방지) — '장부 2' (고객측 기록)"
        varchar device_id "디바이스 지문"
        numeric client_lat
        numeric client_lng
        numeric distance_from_store_m
        boolean device_integrity_verified "Android Play Integrity / iOS DeviceCheck 통과 여부"
        varchar status "verified, rejected, flagged"
        varchar rejected_reason
        timestamptz created_at
    }

    OFFLINE_CHECKIN_REWARD_RULES {
        bigint id PK
        int daily_limit_per_user
        int weekly_limit_per_user
        numeric reward_amount
        int cooldown_minutes "연속 체크인 간 최소 간격"
        numeric lifetime_free_cap_without_paid_charge "유상 충전 이력 없는 유저의 누적 무료지급 상한 — 채굴 방지 핵심"
        boolean is_active
        timestamptz valid_from
        timestamptz valid_to
    }

    OFFLINE_CHECKIN_FRAUD_FLAGS {
        bigint id PK
        bigint checkin_id FK
        varchar flag_type "impossible_travel, device_multi_account, gps_mismatch, shop_anomaly"
        jsonb detail "탐지 근거(이전 체크인과의 거리/시간, 동일 device_id 목록 등)"
        varchar status "open, confirmed_fraud, cleared"
        bigint reviewed_by FK
        timestamptz reviewed_at
        timestamptz created_at
    }
```

**설계 포인트 — 이중 출석부(Dual-Ledger) 검증**
1. 매장에 설치된 태블릿/PC가 60초 주기로 서버로부터 새 QR을 발급받아 화면에 표시 — 이 발급 자체가 `store_qr_issuances`에 독립적으로 기록된다("장부 1", 매장측 실체 증명 — 그 시각 그 매장에 실제로 화면이 존재해야만 생성 가능).
2. 고객은 앱으로 그 QR을 스캔하고, 위치(GPS) + 디바이스 무결성 증명과 함께 서버에 제출 — `store_checkins`에 기록된다("장부 2", 고객측 제출).
3. 서버는 두 장부를 대조: `qr_issuance_id`가 실재하고 미만료·미사용이며, `client_lat/lng`가 매장 반경(`geofence_radius_m`) 내인 경우에만 `status=verified`로 확정 — 한쪽 장부만 조작(GPS 스푸핑만, 또는 QR 스크린샷 공유만)해서는 보상이 나가지 않는다.
4. `qr_issuance_id` UNIQUE로 동일 QR 토큰 재사용(스크린샷 돌려쓰기, 여러 계정이 같은 QR 제출) 원천 차단 — 토큰은 발급 후 짧은 주기 내 1인 1회만 소비 가능.
5. GPS는 그 자체로 모킹 가능하므로 OS 레벨 무결성 증명(Android Play Integrity API / iOS DeviceCheck)을 병행 — `device_integrity_verified=false`면 자동으로 `flagged` 처리, 보상은 검토 전까지 보류.
6. 검증 통과 + 일/주 한도 이내인 경우에만 `bonus_credit_grants`(`source_type=offline_checkin`, §7)로 무상 적립금 지급.

**설계 포인트 — 무료 적립금 채굴(Farming) 방지**
- **핵심 장치**: `offline_checkin_reward_rules.lifetime_free_cap_without_paid_charge` — `user_loyalty_status.cumulative_charge_amount`(§8, 유상 충전 누적)가 0인 유저의 `cumulative_free_offline_bonus`(오프라인 체크인 무상 적립금 누적)가 이 상한에 도달하면 더 이상 지급하지 않는다(`403 FREE_CREDIT_CAP_REACHED_REQUIRES_CHARGE`). 즉 "최소 1회 유상 충전"을 해야 그 이후로는 정상적인 일/주 한도로 계속 받을 수 있다 — 카드샵 방문만 반복해서 무한정 무료 재화를 채굴하는 경로를 구조적으로 막는다.
- **이상 탐지(비동기)**: 체크인 직후 Celery 잡이 (a) 직전 체크인과의 이동거리/시간으로 물리적으로 불가능한 이동인지(`impossible_travel`), (b) 동일 `device_id`가 여러 `user_id`로 체크인하는지(`device_multi_account`), (c) 동일 매장에서 짧은 시간 내 비정상적으로 많은 서로 다른 신규 계정이 몰리는지(`shop_anomaly`, 매장 결탁 의심)를 검사해 `offline_checkin_fraud_flags`에 적재. 의심 건은 지급을 보류하고 관리자 검토 후 `confirmed_fraud`면 회수(및 해당 유저/매장 체크인 자격 정지 검토), `cleared`면 지급 재개.
- 매장 등록(`partner_stores`) 자체도 관리자 승인제(`status=pending→active`)로 운영해, 아무 장소나 셀프 등록해 스스로 방문 처리하는 것을 방지.

---

## 13. Marketplace — 단품/묶음 판매 & 경매

오리파(확률형) 외에 **정가 단품 판매, 묶음(번들) 판매, 경매**까지 지원하는 확장 도메인. 판매 재고는 §4
`physical_cards`(플랫폼 소유 + 위탁)를 그대로 재사용하고, 결제는 §7 지갑, 배송/검수/분쟁은 §6, 위탁 정산은 §11
에스크로(`settlement_holds.source_type=listing_order`)를 공유한다 — 새 도메인이지만 돈·물건의 흐름은 기존 레일 위에서 돈다.

```mermaid
erDiagram
    STORE_LISTINGS ||--o{ LISTING_ITEMS : contains
    PHYSICAL_CARDS ||--o| LISTING_ITEMS : "listed as"
    STORE_LISTINGS ||--o| LISTING_AUCTIONS : "auctioned via"
    LISTING_AUCTIONS ||--o{ AUCTION_BIDS : receives
    USERS ||--o{ AUCTION_BIDS : places
    STORE_LISTINGS ||--o| LISTING_ORDERS : "sold via"
    USERS ||--o{ LISTING_ORDERS : buys
    CONSIGNORS ||--o{ STORE_LISTINGS : "consigns (nullable)"

    STORE_LISTINGS {
        bigint id PK
        varchar listing_type "fixed_price, bundle, auction"
        varchar seller_type "platform, consignor"
        bigint consignor_id FK "seller_type=consignor일 때만 — 위탁 심사(§11) 통과 재고만"
        varchar title
        text description
        numeric price "fixed_price/bundle 판매가, auction이면 NULL"
        varchar status "draft, active, sold, cancelled, expired"
        bigint created_by FK "발행 관리자"
        timestamptz published_at
        timestamptz created_at
    }

    LISTING_ITEMS {
        bigint id PK
        bigint listing_id FK
        bigint physical_card_id FK UK "리스팅 활성 중 실물 잠금 — 오리파 슬롯과 동일한 1:1 원칙"
    }

    LISTING_AUCTIONS {
        bigint id PK
        bigint listing_id FK UK
        numeric start_price
        numeric buy_now_price "즉시구매가, nullable"
        numeric min_bid_increment
        timestamptz starts_at
        timestamptz ends_at
        int soft_close_extension_sec "마감 직전 입찰 시 연장 초 — 스나이핑 방지, 예: 120"
        bigint highest_bid_id FK "현재 최고 입찰, 파생 캐시"
        varchar status "scheduled, live, ended, settled, cancelled"
    }

    AUCTION_BIDS {
        bigint id PK
        bigint auction_id FK
        bigint user_id FK
        numeric bid_amount
        bigint wallet_hold_id FK "입찰액 홀드(§7 wallet_holds) — 잔액 없는 허수 입찰 차단"
        varchar status "active, outbid, won, cancelled"
        timestamptz created_at
    }

    LISTING_ORDERS {
        bigint id PK
        bigint listing_id FK UK
        bigint buyer_user_id FK
        varchar order_type "fixed_price, bundle, auction_won, buy_now"
        numeric total_price "경매는 낙찰가"
        varchar ledger_entry_ref "결제 원장 참조"
        varchar status "paid, preparing, shipped, delivered, confirmed, disputed"
        timestamptz created_at
    }
```

**설계 포인트**
- **실물 잠금**: 리스팅이 `active`로 발행되면 포함된 `physical_cards`를 `status=allocated`로 전환(오리파 슬롯 배치와 동일 상태) — 같은 실물이 오리파 팩과 마켓 리스팅에 동시에 올라가는 것을 `listing_items.physical_card_id` UNIQUE + `oripa_slots.physical_card_id` UNIQUE + 상태머신으로 삼중 차단. `cancelled/expired` 시 `in_stock`으로 복귀.
- **묶음(번들)**: `listing_items`에 N개 실물을 담으면 그대로 번들 — 구매 확정 시 N건의 `user_inventory`가 일괄 생성된다(`source=listing_purchase`).
- **경매 진행**: 입찰은 `현재 최고가 + min_bid_increment` 이상만 허용. 입찰 성공 시 입찰액을 `wallet_holds`로 잠그고 직전 최고 입찰자의 홀드는 즉시 해제(`outbid`) — 잔액이 없는 허수 입찰과 낙찰 후 미납이 구조적으로 불가능하다. `ends_at` 직전 `soft_close_extension_sec` 이내 입찰이 들어오면 마감을 연장(스나이핑 방지).
- **낙찰 정산**: 마감 배치가 최고 입찰을 `won` 처리 → 홀드를 `captured`로 전환(원장 차감) → `listing_orders`(`order_type=auction_won`) 생성 → 이후 배송/검수/확정 흐름은 §6과 동일. `buy_now_price` 즉시구매 시 경매를 조기 종료하고 동일 흐름.
- **위탁 재고 판매 시 에스크로 재사용**: `seller_type=consignor` 리스팅이 판매되면 `settlement_holds`(`source_type=listing_order`)가 생성되어 §11과 동일하게 배송 확정 후 정산 — 오리파에서 뽑히든 마켓에서 팔리든 위탁자 보호 구조는 하나다.
- 마켓 구매 결제도 §7 소진 우선순위(bonus → exchange → paid)를 따른다 — **환급 포인트(exchange)를 마켓 구매에 쓸 수 있게 함**으로써 "환급받은 가치를 플랫폼 안에서 소비"하는 선순환을 만들되 현금 인출은 계속 차단.

**법적 리스크 노트 — 개인 오리파 개설 & 판매 유형별 정리**

| 판매 유형 | 주체 | 법적 성격 | 리스크 평가 |
|---|---|---|---|
| 오리파(확률형) | **플랫폼 직영만 허용** | 랜덤박스 판매 자체는 현행법상 금지 아님. 단 사행성 판단 회피 요건 필수 | 통제 하에 중간 리스크 |
| 오리파(확률형) | ~~개인(위탁자) 주최~~ | 개인이 영리 목적으로 다수로부터 금전을 모아 우연으로 재산상 득실을 결정하면 사행행위규제법상 **무허가 사행행위영업** 소지, 플랫폼은 방조·공동책임 리스크 | **고위험 — 지원하지 않음** |
| 단품/묶음 정가 판매 | 플랫폼 직영 + 위탁 | 일반 통신판매(전자상거래법) — 우연성 없음 | 저위험 |
| 경매 | 플랫폼 직영 + 위탁 | 가격 경쟁 방식으로 우연성 없음 — 사행행위 아님. 통신판매의 한 형태(기존 중고거래 플랫폼 경매와 동일) | 저위험 |

- **개인 오리파를 지원하지 않는 이유**: 사행행위규제법상 "사행행위영업"(복표발행업·추첨업 등)은 경찰청 허가제이며, 개인이 허가 없이 반복·영리적으로 유료 뽑기를 주최하면 무허가 사행행위영업이 될 소지가 크다. 이 경우 플랫폼도 장소·수단 제공자로서 방조 책임을 질 수 있다. 따라서 **오리파의 구성·발행·판매 주체는 항상 플랫폼(사업자)이고, 개인은 §11 위탁으로 카드를 공급하는 역할까지만 허용**한다 — `oripa_packs.created_by`는 관리자 계정만 가능(DB 레벨로는 `admin_permissions` 보유자 검증). 개인의 판매 욕구는 저위험인 마켓플레이스(단품/묶음/경매 위탁)로 흡수한다.
- **플랫폼 직영 오리파의 사행성 완화 요건**(원 계획서 §10 + 본 설계로 구현): ① 전 구 당첨(꽝 없음) + 최소 보장 가치, ② 등급별 잔여 수량 실시간 공개(§API §3), ③ Provably Fair 커밋-리빌(§5), ④ **환급 포인트의 현금 인출 차단**(§7 exchange 버킷 — 환금성 차단), ⑤ 확률형 아이템 확률 공개 의무(게임산업법 개정 취지 준용). 이 요건들을 갖춰도 규제기관 해석 변동 가능성이 있으므로 **출시 전 사행성 전문 변호사 자문은 필수**다.
- 위탁자(개인)가 마켓 판매를 반복하면 통신판매업 신고 의무(전년도 50회 이상 등 기준)가 발생할 수 있다 — 플랫폼은 통신판매**중개**자로서 위탁자 신원정보 열람 제공 의무 등을 지며, 위탁자 온보딩 시 거래량 기준 신고 안내를 자동화한다(§11 `consignors` 온보딩 플로우에 고지 단계 추가).

---

## 14. 설계 노트

### 14.1 `cards.attributes` JSONB 스키마 예시 (게임별)

```jsonc
// Pokemon
{ "hp": 120, "type": "Fire", "evolves_from": "Charmeleon", "regulation_mark": "H" }

// Yu-Gi-Oh!
{ "attribute": "DARK", "level": 8, "atk": 2500, "def": 2100, "card_type": "Effect Monster" }

// One Piece Card Game
{ "cost": 5, "power": 6000, "counter": 1000, "color": ["Red"], "card_type": "Character" }

// Weiss Schwarz
{ "level": 2, "cost": 1, "power": 9500, "soul": 2, "trigger": "Soul" }

// Union Arena
{ "energy_cost": 3, "bp": 5000, "ap": 1, "affinity": "Attack" }
```
공통 컬럼(`rarity`, `card_number`, `name`)만 정규화하고 나머지는 JSONB로 흡수 → 신규 게임 추가 시 스키마 마이그레이션 불필요, 대신 애플리케이션 레벨(Pydantic discriminated union)에서 게임별 검증.

### 14.2 인덱스 전략(주요)
- `cards`: `(card_set_id, card_number)` UNIQUE, `attributes` GIN 인덱스(검색/필터용).
- `physical_cards`: `serial_no` UNIQUE, `status` 부분 인덱스(`WHERE status = 'in_stock'`) — 재고 조회 최적화.
- `oripa_slots`: `(pack_id, slot_no)` UNIQUE, `physical_card_id` UNIQUE, `(pack_id, status)` 부분 인덱스(`WHERE status='available'`) — 뽑기 시 잔여 슬롯 스캔 최적화.
- `ledger_entries`: `(wallet_id, created_at)` 복합 인덱스, `idempotency_key` UNIQUE.
- `draw_logs`, `audit_logs`: `created_at` BRIN 인덱스(append-only 대용량 로그).
- `settlement_holds`: `physical_card_id` UNIQUE, `(status, hold_expires_at)` 부분 인덱스(`WHERE status='held'`) — 검수기한 만료 배치 스캔용.
- `consignor_ledger_entries`: `(consignor_id, created_at)` 복합 인덱스.
- `bonus_credit_grants`: `(wallet_id, expires_at)` 부분 인덱스(`WHERE remaining_amount > 0`) — 만료 배치/FIFO 소진 스캔용.
- `user_missions`: `(user_id, mission_id)` UNIQUE.
- `free_draw_tickets`: `(user_id, status)` 부분 인덱스(`WHERE status='available'`), `expires_at` 만료 배치용.
- `store_qr_issuances`: `qr_token` UNIQUE, `expires_at` 만료 배치용.
- `store_checkins`: `qr_issuance_id` UNIQUE, `(user_id, created_at)` 복합 인덱스 — 일/주 한도 집계 및 이동거리 이상탐지용.
- `attendance_lucky_box_outcomes`: `policy_id` 인덱스, `SUM(probability) = 1.0`은 애플리케이션 레벨 검증(저장 전).
- `user_cosmetic_unlocks`: `(user_id, cosmetic_item_id)` UNIQUE.
- `card_market_prices`: `(card_id, fetched_at)` 복합 인덱스 — 최신 시세 조회 최적화.
- `physical_cards`: `clearance_eligible` 부분 인덱스(`WHERE status='in_stock' AND clearance_eligible=true`) — 럭키박스 실물 보상 풀 스캔용.
- `refund_requests`: `user_inventory_id` 부분 UNIQUE(`WHERE status IN ('pending','approved','paid')`) — 반려 후 재신청 허용하되 활성 요청은 1건만.
- `payment_refunds`: `pg_refund_tid` UNIQUE, `idempotency_key` UNIQUE, `(payment_transaction_id, status)` 복합 인덱스.
- `draw_logs`: `draw_group_id` 인덱스 — 다연차 묶음 조회용.
- `wishlists`: `(user_id, card_id)` UNIQUE, `card_id` 인덱스(팩 오픈 시 역방향 알림 대상 조회).
- `user_collection_entries`: `(user_id, card_id)` UNIQUE.
- `notifications`: `(user_id, read_at)` 부분 인덱스(`WHERE read_at IS NULL`) — 미읽음 카운트 최적화.
- `listing_items`: `physical_card_id` UNIQUE — 동일 실물 이중 리스팅 차단.
- `auction_bids`: `(auction_id, bid_amount DESC)` 복합 인덱스 — 최고가 조회, `wallet_hold_id` UNIQUE.
- `wallet_holds`: `(wallet_id, status)` 부분 인덱스(`WHERE status='held'`) — 사용가능잔액 계산.
- `listing_auctions`: `(status, ends_at)` 부분 인덱스(`WHERE status='live'`) — 마감 배치 스캔.

### 14.3 동시성 & 정합성 강제 요약

| 요구사항 | 강제 방법 |
|---|---|
| 슬롯:실물 1:1 | `oripa_slots.physical_card_id` UNIQUE |
| 동일 슬롯 중복 뽑기 방지 | `SELECT FOR UPDATE SKIP LOCKED` + Redis 분산 락 + `draw_logs.slot_id` UNIQUE |
| 중복 충전/웹훅 재처리 방지 | `payment_transactions.pg_tid` UNIQUE, `ledger_entries.idempotency_key` UNIQUE |
| 포인트 잔액 위변조 방지 | `wallets.paid_balance`/`bonus_balance`는 트리거로만 갱신, 애플리케이션 직접 UPDATE 금지(뷰/함수 경유) |
| 무상 적립금 현금 환불 방지 | 환불(`refund_credit`)은 `balance_type='paid'` 항목에서만 계산, 환불 시 `bonus_balance`는 대상에서 제외 |
| 무료 뽑기권 중복 사용 방지 | `free_draw_tickets.status`는 상태머신(`available→used`)으로만 전이, `draw_logs.free_draw_ticket_id` UNIQUE |
| 동일 미션 중복 완료 처리 방지 | `user_missions.(user_id, mission_id)` UNIQUE |
| 감사 로그 불변성 | `draw_logs`, `audit_logs`, `user_change_logs`에 UPDATE/DELETE 권한 미부여 (append-only role) |
| 동일 SNS 계정 중복 가입 방지 | `user_oauth_accounts.(provider, provider_user_id)` UNIQUE |
| 인증 코드 재사용/무한 시도 방지 | `verification_codes.attempt_count` 임계치 초과 시 코드 폐기, `expires_at` 경과 시 무효 |
| 수동 보정 카드 데이터 덮어쓰기 방지 | `cards.locked_fields`에 잠긴 필드는 동기화 diff 계산에서 제외 |
| 위탁 카드 정산금 조기 유출 방지 | `settlement_holds`는 배송 확정(`confirmed_at`) 전까지 `released`로 전환 불가 (애플리케이션 레벨 상태머신 + 배치 검증) |
| 위탁자 정산 이중 지급 방지 | `consignor_payouts`는 `available_balance` 범위 내에서만 생성, 지급 완료 후 `consignor_ledger_entries`에 `payout` 차감 기록 |
| 동일 QR 토큰 재사용(리플레이) 방지 | `store_checkins.qr_issuance_id` UNIQUE, `store_qr_issuances.expires_at` 경과 토큰은 소비 불가 |
| 무료 적립금 무한 채굴 방지 | 유상 충전 이력(`cumulative_charge_amount`) 없는 유저는 `cumulative_free_offline_bonus`가 `lifetime_free_cap_without_paid_charge` 도달 시 오프라인 체크인 보상 지급 중단 |
| 장기 무결제자 출석 보상 감소가 결제 혜택까지 침범하지 않도록 격리 | `attendance_reward_policies`/`attendance_lucky_box_outcomes`는 `bonus_credit_grants`(§7)만 갱신, `loyalty_tiers.benefits`·`charge_bonus_rules`(결제 혜택)는 별도 경로로 절대 참조하지 않음 |
| 럭키박스 확률 합계 오류 방지 | `attendance_lucky_box_outcomes` 저장 시 `SUM(probability) = 1.0` 애플리케이션 검증, 불일치 시 저장 거부 |
| 럭키박스 기댓값이 일반 보상보다 높아지는 것 방지 | 저장 시 `Σ(probability × reward_value) <= 그 시점 decay 적용 동적 보상 × lucky_box_target_ev_ratio`(1.0 미만)를 검증, 위반 시 저장 거부 |
| 코스메틱 중복 해금 방지 | `user_cosmetic_unlocks.(user_id, cosmetic_item_id)` UNIQUE |
| 신뢰할 수 없는 시세로 즉시 환급되는 것 방지 | 뽑기 확정 시점 `card_market_prices.fetched_at`이 `instant_exchange_policies.max_price_age_hours`를 초과하면 `user_inventory.locked_reference_price`를 NULL로 두어 이후 환급이 자동으로 `manual_review`로 전환되고, 즉시 지급(`status=paid`) 경로가 차단됨 |
| 시세 변동 차익거래(뽑은 뒤 오른 가격으로 환급) 방지 | 즉시환급 계산은 항상 `user_inventory.locked_reference_price`(뽑기 확정 시점 스냅샷)만 사용, 환급 요청 시점에 `card_market_prices`를 재조회하지 않음(애플리케이션 레벨 — 조회 자체를 하지 않도록 구현) |
| 현금 환불 과다 지급 방지 | `payment_refunds.refund_amount` 합계가 `paid_balance` 및 원거래별 `amount - refunded_amount`를 초과할 수 없음(트랜잭션 내 검증), `pg_refund_tid`/`idempotency_key` UNIQUE로 중복 취소 차단 |
| 충전 보너스 먹튀(보너스만 소진 후 원금 환불) 방지 | 환불 시 해당 충전 건에 지급된 `bonus_credit_grants`를 비율대로 회수, 회수 불가분은 환불액에서 차감(`clawed_back_bonus`) |
| 환급 재신청 시 중복 활성 요청 방지 | `refund_requests.user_inventory_id` 부분 UNIQUE(`WHERE status IN ('pending','approved','paid')`) |
| 다연차 뽑기 부분 성공 방지 | N연차는 단일 트랜잭션 — 잔여 슬롯 부족 시 전체 롤백(`INSUFFICIENT_REMAINING_SLOTS`), 절대 일부만 커밋되지 않음 |
| 라스트원 상 중복/경합 지급 방지 | 마지막 슬롯을 소진한 뽑기 트랜잭션 내에서만 지급 — 팩당 마지막 슬롯은 물리적으로 1개이므로 경합 자체가 불가능 |
| 도감 중복 등록 방지 | `user_collection_entries.(user_id, card_id)` UNIQUE + upsert |
| 뽑기 환급 포인트의 현금 인출 차단(환금성 차단) | `refund_credit`은 `balance_type=exchange`로만 적립, `payment_refunds` 환불 가능액 계산은 `paid_balance`만 참조 — 애플리케이션·원장 이중 강제 |
| 동일 실물의 오리파/마켓 이중 판매 방지 | `oripa_slots.physical_card_id` UNIQUE + `listing_items.physical_card_id` UNIQUE + `physical_cards.status` 상태머신(`in_stock→allocated`) 삼중 차단 |
| 허수 입찰/낙찰 미납 방지 | 입찰 시 `wallet_holds`로 입찰액 선점(잔액 부족 시 입찰 자체 불가), 낙찰 시 `captured` 전환으로만 결제 |
| 경매 동시 입찰 경합 | 경매별 Redis 락 + `bid_amount > 현재최고가 + min_bid_increment` 검증을 단일 트랜잭션에서 수행 |
| 개인의 오리파 발행 차단 | `oripa_packs.created_by`는 `admin_permissions` 보유 계정만 허용(애플리케이션 + DB 트리거 검증) — §13 법적 리스크 노트 |
