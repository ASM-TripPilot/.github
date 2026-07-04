# U1 기반·계정·온보딩 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M1 Auth, M2 User Profile, C3 Content Moderation
> 정본 관계: [components.md](../../../inception/application-design/components.md) M1·M2·C3의 엔티티 개요를 인터페이스 계약 수준에서 상세화한다. 본 문서가 U1 엔티티·불변식의 정본이다. 기술 중립 — 저장 기술·컬럼 타입·인덱스 전략은 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL의 의미를 반드시 명기), `유니크`=전역 유일 제약. 근거 ID는 requirements.md(D·Δ·N·G)·stories.md(US)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U1-xx)을 참조한다.

## 0. 엔티티 지도

```text
[M1 Auth]
Account 1 ──── * SocialIdentity          (소셜 연결, provider+sub 복합 유니크)
Account 1 ──── * EmailVerification       (인증 토큰 발급 이력)
Account 1 ──── * ConsentRecord           (append-only 동의 증적)
Account 1 ──── 0..1 MarketingConsent     (현재 상태 뷰)
Account 1 ──── 1 LocationConsentState    (3층 동의 현재 상태)
Account 1 ──── * RefreshSession          (기기별 회전 체인)
Account 1 ──── 0..1 DeletionSchedule     (30일 유예)
TermsVersion * ──── * ConsentRecord      (동의 대상 버전 참조)
(독립) LocationLegalLog                  (append-only 법정 로그 — 앱 역할 삭제 불가)

[M2 User Profile]
Account 1 ──── 1 Profile
Profile 1 ──── 1 PreferenceSet           (7축, 축별 NULL=미설정)

[C3 Content Moderation]
(독립) BannedWordDictionary              (버전 관리 사전)
```

---

## 1. Account — 계정 (M1)

계정 생명주기·인증 수단·연령 확인의 정본. 모든 계정은 '여행자' 단일 유형 [US-E1-01].

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자 | 필수 · 유니크 · 불변 | 시스템 전역 계정 식별자. 외부 노출 식별자는 추측 불가 형식(순차 노출 금지 — SECURITY-08 IDOR 방어의 보조선) |
| email | 문자열(이메일 형식) | 선택 · **활성 계정 간 유니크**(§1.3 INV-A3) | 이메일 가입=필수, 소셜 가입=제공자 제공 시 저장. NULL=소셜 제공자가 이메일 미제공(예: 동의 거부). Apple 비공개 릴레이 주소는 릴레이 주소 그대로 저장 [US-E1-01, G20] |
| passwordHash | 문자열(해시) | 선택 | 이메일 계정만 보유. 적응형 해시 결과만 저장 — 평문·가역 암호화 금지. 해시 알고리즘·파라미터 선정은 NFR Design 소유 [G22, §6.4] |
| ageConfirmation | 구조체 {method: BIRTH_DATE\|SELF_DECLARED, birthDate?: 날짜, confirmedAt: 시각} | 필수 | 생년월일 입력 또는 '만 14세 이상' 자기확인. 가입 경로(소셜·이메일) 무관 필수 [N1/D33, US-E1-16] |
| status | 열거 | 필수 | `PENDING_VERIFICATION → ACTIVE → DELETION_PENDING → DELETED` — §1.2 상태 머신 정본 [D18, G22] |
| sanctionStatus | 열거 | 필수 · 기본 `NONE` | 후속 예약 필드(경고→커뮤니티 정지→전체 정지) — 1차는 `NONE` 고정, 스키마만 예약 [G179] |
| createdAt | 시각(UTC) | 필수 · 불변 | 계정 생성 시각 |
| verifiedAt | 시각(UTC) | 선택 | 이메일 인증 완료 시각. 소셜 계정은 생성 시각과 동일. NULL=미인증 |
| deletedAt | 시각(UTC) | 선택 | 소프트 삭제 마킹(DELETION_PENDING 진입 시각). NULL=삭제 아님 [D18] |

### 1.2 계정 상태 머신 (정본 상세 전이도 — requirements.md §9 이관분 확정)

> components.md §3.4의 `UNVERIFIED`는 본 문서의 `PENDING_VERIFICATION`과 동일 상태(FD-U1-01).

```text
                (이메일 가입)                    (인증 완료)
   [*] ────────────────────▶ PENDING_VERIFICATION ────────────▶ ACTIVE
    │                              │                              │  ▲
    │ (소셜 가입 — 연령 통과)        │ (7일 경과 배치 — 정리 삭제)     │  │ (유예 내 철회
    └─────────────────────────────┼──────────▶ [정리 파기]        │  │  cancelAccountDeletion)
                                  ▼                              ▼  │
                                                        DELETION_PENDING
                                                                  │
                                                  (유예 만료 배치 +30일)
                                                                  ▼
                                                              DELETED [최종]
```

| 전이 | 트리거 | 가드 조건 | 효과 |
|---|---|---|---|
| (없음) → PENDING_VERIFICATION | `signUpWithEmail` | 연령 확인 통과(N1) · 이메일 활성 계정 중복 없음(BR-U1-01) · 유예 중 동일 식별자 아님(C4) | EmailVerification 발급·메일 발송. 온보딩 진행 차단 상태 [G22, US-E1-01] |
| (없음) → ACTIVE | `signUpWithSocial` (신규) | 연령 확인 통과(N1) · 동일 이메일 충돌 처리 통과(BR-U1-03) · 유예 중 재가입 아님(C4) | 소셜은 제공자가 신원을 보증하므로 인증 단계 생략 — 즉시 활성 [US-E1-01] |
| PENDING_VERIFICATION → ACTIVE | `verifyEmail` (링크 토큰 소비) | 토큰 유효(24h·미소비·최신 발급분) | verifiedAt 기록·자동 로그인·약관 동의 단계로 진행 [G22] |
| PENDING_VERIFICATION → (정리 파기) | 미인증 정리 배치(일 1회) | 생성 후 7일 경과 · 미인증 유지 | 계정·인증 토큰 물리 정리. 동일 이메일 재가입 즉시 허용 [G22] |
| ACTIVE → DELETION_PENDING | `requestAccountDeletion` | 재확인 완료 · 연쇄 삭제 범위 사전 고지(BR-U1-37) | deletedAt 기록·**즉시 비노출**·DeletionSchedule 생성(purgeAt=+30일)·전 세션 무효화·GPS 발자취 즉시 파기(FD-U1-07) [D18, D34] |
| DELETION_PENDING → ACTIVE | 유예 중 로그인 → 복구 확인 → `cancelAccountDeletion` | purgeAt 도래 전 | deletedAt 해제·DeletionSchedule 폐기·노출 복원(GPS 발자취는 미복원 — 기파기) [D18] |
| DELETION_PENDING → DELETED | 유예 만료 배치(일 1회) | purgeAt 경과 | 연쇄 삭제 오케스트레이션(U1은 계정·프로필·취향·세션·소셜 연결 골격, 전 모듈 연쇄 완성은 U8/S6). 법정 보존 데이터(위치 법정 로그·동의 증적)만 분리 보관 [D18, D34, N2] |
| DELETED | — (최종 상태) | 재개 없음 | 동일 식별자 재가입은 신규 계정으로 허용(유예 기간이 지났으므로 C4 제한 해제) |

### 1.3 불변식 (Invariants)

- **INV-A1 (상태 전이 폐쇄)**: status는 위 표의 전이로만 변경된다. 임의 상태 점프(예: DELETED→ACTIVE) 불가 — 하드 제약 '계정 무결성' 계열의 테스트 대상 [D37, U1 DoD].
- **INV-A2 (연령 게이트)**: 어떤 경로로도 ageConfirmation 없이 또는 만 14세 미만으로 계정 행이 존재할 수 없다 — 차단은 계정 생성 **이전** [N1/D33, US-E1-16].
- **INV-A3 (이메일 유일성)**: `status ∈ {PENDING_VERIFICATION, ACTIVE, DELETION_PENDING}`인 계정 집합 안에서 email(NULL 제외)은 유일하다. DELETED·정리 파기된 계정의 이메일은 재사용 가능 [US-E1-01, C4].
- **INV-A4 (인증 수단 존재)**: ACTIVE 계정은 passwordHash 또는 1개 이상의 SocialIdentity 중 최소 하나를 가진다(로그인 불가능 계정 없음).
- **INV-A5 (삭제 마킹 정합)**: `status=DELETION_PENDING ⇔ deletedAt≠NULL ∧ DeletionSchedule 존재`. DELETION_PENDING 계정은 모든 조회·기능에서 비노출(로그인 시 복구 안내 경로만 예외) [D18].
- **INV-A6 (미인증 격리)**: PENDING_VERIFICATION 계정은 약관 동의 이후 온보딩 단계(닉네임·취향)로 진행할 수 없고 토큰 발급 대상이 아니다(인증 완료 시점에 최초 발급) [G22, US-E1-01, U1 DoD "미인증 상태에서 온보딩 진행 차단"].

---

## 2. SocialIdentity — 소셜 연결 (M1, Account 서브엔티티)

소셜 제공자 신원과 내부 계정의 매핑. 계정 식별의 기준은 이메일이 아니라 **제공자 고유 ID(sub)** [US-E1-01].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| provider | 열거 {GOOGLE, APPLE, KAKAO, NAVER} | 필수 | 소셜 4종 [US-E1-01] |
| providerSub | 문자열 | 필수 | 제공자 발급 고유 사용자 ID(sub). 제공자 토큰 서명·수신자 검증 통과분만 저장 [component-methods M1] |
| accountId | 식별자(Account FK) | 필수 | N:1 — 한 계정에 복수 제공자 연결 가능(구조 허용. 단, **1차에서 수동 연결 기능은 미제공** — 실제 N>1은 발생하지 않고 CS 처리 [G20]) |
| providerEmail | 문자열 | 선택 | 제공자가 알려준 이메일(대조용 스냅샷). NULL=미제공/대조 불가(Apple 비공개 릴레이 포함) [G20] |
| linkedAt | 시각(UTC) | 필수 · 불변 | 최초 연결 시각 |

**불변식**
- **INV-S1 (복합 유니크)**: `(provider, providerSub)` 조합은 전역 유일 — 동일 소셜 신원이 두 계정에 매핑될 수 없다. 하드 제약 '중복 계정 미생성(sub 기준)'의 구조적 강제 [D37, U1 DoD].
- **INV-S2 (댕글링 금지)**: SocialIdentity는 존재하는 Account 없이 존재할 수 없다. 계정 DELETED 시 함께 파기(재가입 시 동일 sub로 신규 계정 생성 가능).

---

## 3. EmailVerification — 이메일 인증 토큰 (M1)

인증 링크의 발급·소비 이력. 재발송 제한 판정의 근거 데이터를 겸한다 [G22].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| verificationId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account FK) | 필수 | 대상 계정(PENDING_VERIFICATION) |
| tokenHash | 문자열(해시) | 필수 · 유니크 | 링크 토큰의 해시만 저장(원문 미저장 — 로그·DB 유출 시 재사용 방지) [SECURITY-03·12 정합] |
| issuedAt | 시각(UTC) | 필수 · 불변 | 발급 시각. 재발송 rate-limit(분당 1회·일 5회)은 계정별 issuedAt 시퀀스로 판정 [G22, BR-U1-13] |
| expiresAt | 시각(UTC) | 필수 | `= issuedAt + 24시간` [G22, US-E1-01] |
| consumedAt | 시각(UTC) | 선택 | 소비(인증 완료) 시각. NULL=미소비 |
| invalidatedAt | 시각(UTC) | 선택 | 무효화 시각(재발송에 의한 구 토큰 무효 — FD-U1-08). NULL=무효화 안 됨 |

**불변식**
- **INV-E1 (1회성)**: consumedAt이 기록된 토큰은 다시 소비될 수 없다(재클릭 시 이미 인증됨 안내 — 오류 아님).
- **INV-E2 (단일 유효)**: 한 계정에 대해 `consumedAt=NULL ∧ invalidatedAt=NULL ∧ expiresAt>now`인 토큰은 항상 최대 1개 — 재발송은 기존 유효 토큰을 먼저 무효화한다(FD-U1-08).
- **INV-E3 (만료 불소비)**: `expiresAt ≤ 소비 시각`인 소비는 불가 — `TokenExpired` + 재발송 경로 안내 [G22].

---

## 4. TermsVersion — 약관 버전 (M1)

약관 3종+선택 동의 문서의 버전 정본. 재동의 판정의 입력 [N3].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| termsType | 열거 {TERMS_OF_SERVICE, PRIVACY_POLICY, LOCATION_TERMS, MARKETING, GPS_RECORDING, PERSONALIZATION} | 필수 | 필수 3종=이용약관·개인정보 처리방침·위치기반서비스 이용약관(분리 항목 N2). MARKETING·GPS_RECORDING은 선택 동의의 문안 버전, PERSONALIZATION은 기록 개인화(US-E8-10 대비 예약) [US-E1-02·17, N8] |
| version | 문자열(버전) | 필수 · (termsType 내) 유니크 | 문안 버전. P7 법무 문안 탑재 전에는 플레이스홀더 버전으로 운영 — 버전 체계 자체는 U1에서 완성 [P7, unit-of-work U1 §선결] |
| body | 텍스트(또는 문서 참조) | 필수 | 상시 재열람 제공(현행+이력) [N5, M1.getPolicyDocuments] |
| effectiveAt | 시각(UTC) | 필수 | 시행 시각 |
| reconsentRequired | 불리언 | 필수 | **재동의 필요 플래그** — true=중대 변경(불리한 변경·수집 항목 확대 → 스플래시 재동의 강제), false=경미 변경(인앱 공지) [N3, US-E1-18] |

**불변식**
- **INV-T1 (버전 불변)**: 시행된 버전의 body·effectiveAt·reconsentRequired는 수정하지 않는다 — 변경은 항상 새 버전 발행(동의 증적의 참조 무결성 보장) [N2·N3].
- **INV-T2 (현행 유일)**: termsType별로 `effectiveAt ≤ now`인 버전 중 최신 1개가 '현행'이며, 동의 게이트·재동의 판정은 항상 현행 버전 기준.

---

## 5. ConsentRecord — 동의 증적 (M1, append-only)

동의·철회의 법정 증적. **행 단위 불변·추가 전용** — 갱신·삭제 연산이 존재하지 않는다(FD-U1-02) [N2, N3, SECURITY-14].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| recordId | 식별자 | 필수 · 유니크 · 단조 증가 | 증적 순서 보장 |
| accountId | 식별자(Account FK) | 필수 | 동의 주체 |
| termsType | 열거(TermsVersion.termsType 동일) | 필수 | 동의 항목 |
| termsVersion | 문자열(TermsVersion 참조) | 필수 | 동의한 문안 버전 — 항목·일시·버전 3요소가 증적의 최소 구성 [US-E1-02·18] |
| action | 열거 {GRANT, REVOKE} | 필수 | 부여/철회. 철회도 새 행으로 추가(과거 행 수정 없음) |
| occurredAt | 시각(UTC) | 필수 · 불변 | 동의·철회 일시 |
| channel | 열거 {ONBOARDING, RECONSENT, SETTINGS} | 필수 | 수집 지점(증적 맥락) |

**불변식**
- **INV-C1 (append-only)**: 기존 행은 어떤 경로로도 수정·삭제되지 않는다. 계정 DELETED 시에도 법정 보존 대상으로 분리 보관(파기 대상 아님) [D18, N2 — DB 권한 분리 강제는 마이그레이션 리뷰로 검증, U1 DoD].
- **INV-C2 (현재 상태 = 폴드)**: 항목별 현재 동의 상태는 `(termsType별 occurredAt 최신 행).action = GRANT`로 유도된다 — 별도 가변 상태 필드는 파생 캐시일 뿐 정본이 아니다.
- **INV-C3 (필수 동의 완비)**: 온보딩 완료 계정은 필수 3종(TERMS_OF_SERVICE·PRIVACY_POLICY·LOCATION_TERMS) 각각에 대해 현행 계보의 GRANT 증적을 가진다 [US-E1-02·17, BR-U1-21].
- **INV-C4 (REVOKE 전제)**: REVOKE 행은 동일 termsType의 선행 GRANT 없이 존재할 수 없다.

---

## 6. MarketingConsent — 마케팅 수신 동의 현재 상태 (M1)

증적(ConsentRecord)의 파생 뷰를 즉시 조회 가능한 현재 상태로 유지한다. 1차에서 **발송은 하지 않고** 수집·철회만 [N8].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account FK) | 필수 · 유니크 | 1:0..1 |
| optIn | 불리언 | 필수 | 현재 수신 동의 여부. 온보딩 미동의 진행 가능(선택 항목) [US-E1-02, N8] |
| updatedAt | 시각(UTC) | 필수 | 최종 변경 시각 |

**불변식**
- **INV-M1 (증적 정합)**: optIn 값은 ConsentRecord(termsType=MARKETING)의 최신 action과 항상 일치한다 — 변경은 증적 추가와 원자적으로 수행.
- **INV-M2 (발송 부재)**: 1차 범위에는 optIn=true를 소비하는 발송 로직이 존재하지 않는다(발송은 후속 — N8). 동의 데이터만 축적.

---

## 7. LocationConsentState + LocationLegalLog — 위치 동의 3층·법정 로그 (M1)

### 7.1 3층 모델 (G182)

| 층 | 이름 | 정본 위치 | 의미 |
|---|---|---|---|
| L1 | OS 위치 권한 | **클라이언트(단말)** — 서버는 마지막 보고값 미러만 보관 | OS 다이얼로그 허용 여부. just-in-time 발화(실제 발화 지점은 U3 '내 주변'·U6 여행 중 — U1은 프리프롬프트 프레임·상태 관리만) [US-E1-04, unit-of-work U1 §제외] |
| L2 | 앱 내 법정 동의 | 서버(ConsentRecord termsType=LOCATION_TERMS) | 위치기반서비스 이용약관 **필수** 동의 — 온보딩 게이트 대상 [N2/D34, US-E1-17] |
| L3 | GPS 기록 옵트인 | 서버(ConsentRecord termsType=GPS_RECORDING) | GPS 여행 기록(발자취) 보관 **선택** 동의 — 미동의로도 온보딩·서비스 이용 가능 [US-E1-17] |

### 7.2 LocationConsentState 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account FK) | 필수 · 유니크 | 1:1 |
| osPermissionMirror | 열거 {GRANTED, DENIED, NOT_DETERMINED} | 필수 · 기본 NOT_DETERMINED | 단말 보고값 미러(참고용 — 판정 정본은 단말 런타임) [G182] |
| legalConsent | 불리언(파생 — ConsentRecord 폴드) | 필수 | L2 현재 상태 |
| gpsRecordingOptIn | 불리언(파생 — ConsentRecord 폴드) | 필수 | L3 현재 상태 |
| updatedAt | 시각(UTC) | 필수 | — |

### 7.3 조합별 기능 동작 매트릭스 (G182 — requirements.md §9 이관분 확정, 정본)

> 판정 함수: `effectiveCapabilities(L1, L2, L3) → {localLocationUse, serverLocationService, gpsTrackRetention}` — 총함수(8조합 전부 정의), 클라이언트 shared/location 런타임 판정기와 서버 M1이 동일 명세를 공유한다.

| # | L1 OS 권한 | L2 법정 동의 | L3 GPS 옵트인 | 단말 로컬 위치 사용¹ | 서버 위치 서비스² | GPS 발자취 보존³ | 대표 사용자 상황·폴백 |
|---|---|---|---|---|---|---|---|
| 1 | ✕ | ✕ | ✕ | 불가 | 불가 | 불가 | 온보딩 전/재동의 대기 + 권한 거부. 위치 기능 전부 비활성 — 등록 숙소·여행지 중심 좌표 폴백 [US-E1-04] |
| 2 | ✕ | ✕ | ○ | 불가 | 불가 | 불가 | 과도 상태(약관 재동의 대기 중 옵트인 잔존). L3은 L2 없이 무효 — 수집 0 |
| 3 | ✕ | ○ | ✕ | 불가 | 불가(입력 없음) | 불가 | 법정 동의는 있으나 OS 거부. 위치 의존 기능은 수동 입력·등록 숙소 기준 폴백, "현재 위치 기능은 설정에서 켤 수 있습니다" 안내 [US-E1-04] |
| 4 | ✕ | ○ | ○ | 불가 | 불가(입력 없음) | 불가(입력 없음) | OS 거부로 단말 위치 없음 — 옵트인은 유지되나 수집 실입력 0. OS 허용 시 즉시 5→8행 동작 |
| 5 | ○ | ✕ | ✕ | 가능 | **금지** | 금지 | 재동의 대기 등 예외 상태. 위치는 단말 내 편의(지도 현위치 표시)까지만 — 서버 전송·처리 금지 [components.md shared/location] |
| 6 | ○ | ✕ | ○ | 가능 | **금지** | 금지 | 5와 동일 — L3 단독은 무효 |
| 7 | ○ | ○ | ✕ | 가능 | 가능 | **불가** | 표준 상태(옵트인 안 함). 내 주변 탐색·도착 프롬프트·위치 트리거 동작, 발자취는 미수집(기록 비교 화면은 추정 표시 — U6) |
| 8 | ○ | ○ | ○ | 가능 | 가능 | 가능 | 전체 활성 — GPS 발자취 수집·보존(수집 자체는 U6, 동의 모델은 U1) |

¹ 단말 로컬 위치 사용: 지도 현위치 표시 등 단말 내 처리·서버 미전송 편의 기능.
² 서버 위치 서비스: 위치의 서버 전송·처리(내 주변 탐색 기준점, 도착 확인 프롬프트, 이동 지연·체류 초과 트리거 — 실제 기능은 U3·U6).
³ GPS 발자취 보존: 서버측 폴리라인 영구 보존(G55/G73 — U6).

**매트릭스 불변식**
- **INV-L1 (서버 전송 게이트)**: `serverLocationService = L1 ∧ L2`. L2 없는 서버 전송은 어떤 조합에서도 불가(법정 요건) [N2/D34].
- **INV-L2 (보존 게이트)**: `gpsTrackRetention = L1 ∧ L2 ∧ L3`.
- **INV-L3 (단조성)**: 층을 하나 추가로 허용해도 기존 능력이 축소되지 않는다(능력 집합은 층 집합에 대해 단조 증가).
- **INV-L4 (철회 즉시성)**: L3 REVOKE 시 → 수집 즉시 중단 + 기보존 GPS 발자취 즉시 파기 트리거(M12.purgeLocationData — U1은 트리거 계약, 대상 데이터는 U6 이후 존재). 법정 로그는 파기와 무관하게 보존 [N2, US-E1-17].
- **INV-L5 (온보딩 독립)**: L1·L3의 어떤 값도 온보딩 완료를 막지 않는다(L2만 필수 게이트) [US-E1-04·17].

### 7.4 LocationLegalLog — 위치정보 수집·이용·제공 사실 확인자료 (법정 로그)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| logId | 식별자 | 필수 · 유니크 · 단조 증가 | — |
| accountId | 식별자 | 필수 | 대상(파기 후에도 로그는 잔존하므로 FK 강제가 아닌 값 보존) |
| eventType | 열거 {CONSENT_GRANTED, CONSENT_REVOKED, COLLECTION, USE, PROVISION, PURGE} | 필수 | 수집·이용·제공·파기 사실 유형. U1 발생분=동의·철회·파기 트리거, 수집·이용 행은 U6부터 적재 |
| detail | 구조체(사유·범위 요약) | 필수 | 원시 좌표 미포함(사실 확인자료이지 위치 데이터가 아님) |
| occurredAt | 시각(UTC) | 필수 · 불변 | — |

**불변식**
- **INV-LL1 (append-only·권한 분리)**: 애플리케이션 역할은 본 로그의 갱신·삭제 권한을 갖지 않는다(추가만 가능) — DB 레벨 권한 분리로 강제, 마이그레이션 리뷰 검증 항목 [N2, SECURITY-14, unit-of-work U1 리스크 완화].
- **INV-LL2 (보존 기간)**: 최소 6개월 보존 — 계정 삭제·GPS 파기와 독립 [N2, US-E1-17].

---

## 8. RefreshSession — 리프레시 토큰 회전 체인 (M1)

기기별 세션의 정본. 토큰 회전·재사용 감지의 데이터 기반 [D36].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| sessionId | 식별자 | 필수 · 유니크 | 회전 체인 항목 식별 |
| accountId | 식별자(Account FK) | 필수 | — |
| deviceId | 문자열 | 필수 | 기기 식별 — 다기기 동시 로그인은 기기별 체인 분리로 허용 [D36] |
| tokenHash | 문자열(해시) | 필수 · 유니크 | 리프레시 토큰 해시(원문 미저장). 토큰 자체는 단말 OS 보안 저장소 보관(클라이언트 책무 — SECURITY-12) |
| chainId | 식별자 | 필수 | 동일 기기 로그인 1회가 만드는 회전 체인의 묶음 식별자 |
| issuedAt | 시각(UTC) | 필수 | 발급 시각. 리프레시 수명 90일 [D36] |
| expiresAt | 시각(UTC) | 필수 | `= issuedAt + 90일` |
| rotatedAt | 시각(UTC) | 선택 | 회전 소비 시각(후속 토큰 발급됨). NULL=체인의 현행 토큰 |
| revokedAt | 시각(UTC) | 선택 | 무효화 시각(로그아웃·재사용 감지·비밀번호 재설정·계정 삭제). NULL=유효 |

> **TokenPair(액세스+리프레시)**: 액세스 토큰(수명 1시간)은 자기 서명 검증형으로 서버 상태를 갖지 않는 값 객체 — 엔티티가 아니며, 영속 정본은 RefreshSession뿐이다 [D36].

**불변식**
- **INV-R1 (체인 현행 유일)**: 체인(chainId)마다 `rotatedAt=NULL ∧ revokedAt=NULL ∧ expiresAt>now`인 토큰은 항상 최대 1개.
- **INV-R2 (재사용 = 탈취 신호)**: `rotatedAt≠NULL`(이미 회전 소비된) 토큰의 재사용 시도는 인증 실패이자 **해당 체인 전체 즉시 무효화 + 보안 알림** 트리거 [D36, SECURITY-03, U1 DoD PBT "재사용 토큰 무효"].
- **INV-R3 (기기 격리)**: 체인 무효화의 기본 범위는 해당 deviceId 체인 — 타 기기 세션은 유지(다기기 허용). 단 비밀번호 재설정은 **전 기기** 체인 무효화 [component-methods M1.resetPassword].
- **INV-R4 (상태 종속)**: ACTIVE가 아닌 계정(PENDING_VERIFICATION·DELETION_PENDING·DELETED)에 유효 체인이 존재하지 않는다 — 삭제 요청 시 전 체인 revoke(복구 확인용 임시 인증은 별도 단명 흐름).

---

## 9. DeletionSchedule — 삭제 유예 (M1)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account FK) | 필수 · 유니크 | 계정당 최대 1개(활성 기준) |
| requestedAt | 시각(UTC) | 필수 · 불변 | 삭제 요청 시각 |
| purgeAt | 시각(UTC) | 필수 | `= requestedAt + 30일` — 유예 만료 배치의 판정 기준 [D18] |
| cascadeSummary | 구조체(연쇄 삭제 대상 요약) | 필수 | 요청 시점 고지 내용의 스냅샷: 숙소(위시리스트·등록)·여행·일정·기록·회고·사진, (후속) 커뮤니티 게시물 삭제·댓글 '삭제된 사용자' 익명화. 법정 보존 분리 항목(위치 법정 로그·동의 증적) 명시 [D18, D34] |
| cancelledAt | 시각(UTC) | 선택 | 철회 시각. NULL=진행 중 |

**불변식**
- **INV-D1 (유예 정합)**: `cancelledAt=NULL ∧ purgeAt>now`인 DeletionSchedule 존재 ⇔ Account.status=DELETION_PENDING.
- **INV-D2 (만료 후 철회 불가)**: purgeAt 경과 후 철회 시도는 `ResourceNotFound` [component-methods M1.cancelAccountDeletion].

---

## 10. Profile — 프로필 (M2)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account FK) | 필수 · 유니크 | 1:1 |
| nickname | 문자열 | 필수 · **활성 계정 간 유니크** · 2~20자 | 자동 생성 기본값('형용사+여행명사+2자리 숫자')으로 시작 — 항상 값 존재(NULL 불가) [G23, US-E1-03] |
| nicknameUpdatedAt | 시각(UTC) | 필수 | 최종 변경 시각(자동 생성 시각 포함) |
| onboardingCompletedAt | 시각(UTC) | 선택 | 온보딩 완료 판정 시각. NULL=미완료. 판정=약관 동의+닉네임 통과(취향 무관) [G24/G157] |

**불변식**
- **INV-P1 (닉네임 검증 통과)**: 저장된 nickname은 길이(2~20)·금칙어(C3)·유니크 3검증을 모두 통과한 값이다 — 자동 생성값 포함(생성기는 사전 검증된 어휘풀 사용) [G23, BR-U1-16~19].
- **INV-P2 (완료 판정 정합)**: `onboardingCompletedAt≠NULL ⇒ INV-C3(필수 동의 완비) ∧ nickname 존재`. 역방향 기록은 `completeOnboarding` 멱등 호출로만 [G24, FD-U1-09].
- **INV-P3 (닉네임 전파 규칙)**: 표시(추후 댓글·게시물)는 accountId 라이브 참조, 기록·복제 출처는 시점 스냅샷 [C3(§7.4), components.md M2].

---

## 11. PreferenceSet — 취향 7축 (M2)

축별 NULL=**미설정**(사용자가 선택하지 않음)이며, **중립 기본값과 명확히 구분**한다: 중립 기본값은 저장되지 않고 조회 시점에 파생 주입되며 `isNeutralDefault=true` 플래그로 표시된다 [US-E1-14, component-methods M2.getPreferences].

### 11.1 속성 — 값 도메인은 PRD 03 선택지 그대로 (stories.md E1 정본)

| 축 | 속성 | 타입·값 도메인 | 선택 방식 | NULL(미설정)의 의미 | 중립 기본값(파생 — 저장 안 함) | 근거 |
|---|---|---|---|---|---|---|
| 1 스타일 | styles | 집합 ⊆ {휴양, 관광, 액티비티, 미식, 쇼핑, 자연, 문화예술} | 복수(1개 이상) | 스타일 축 무가중 | 무가중치(빈 가중 벡터) | US-E1-05 |
| 2 예산 | budgetTier / budgetRawAmount | 열거 {저가, 중간, 고급, 럭셔리} / 금액(원, 양수) | 4구간 선택 **또는** 직접 총액 입력(원값+매핑 구간 동시 저장) | 가격 필터 미적용·전체 가격대 노출 신호 | 필터 미적용 | US-E1-06, D26/Δ2, G26 |
| 3 동행 | companionType + petFlag | 열거 {혼자, 커플, 친구, 가족(아동 동반), 부모님} + 불리언 | 기본 유형 **단일** 선택 + 반려동물 **분리 불리언** | 동행 축 무가중 | 무가중치 · petFlag 파생 기본 false | US-E1-07, G19 |
| 4 활동 | activities | 집합 ⊆ {자연, 역사문화, 테마파크, 맛집투어, 카페, 전시, 야경, 쇼핑, 스포츠} | 복수 | 활동 축 무가중 | 무가중치 | US-E1-08 |
| 5 이동 | transportModes | 집합 ⊆ {도보, 대중교통, 렌터카, 택시, 자전거} | 복수 | 동선 기본 수단 미지정 | **대중교통** + 안전계수 보수 추정(내부 계산 한정 — 화면은 거리만) | US-E1-09·14, D25/Δ1 |
| 6 음식 | foodTastes | 집합 ⊆ {한식, 양식, 일식, 중식, 아시안, 기타} | 복수 | 음식 축 무가중 | 무가중치 | US-E1-10 |
| 7 페이스 | pace | 열거 {느긋하게(하루 1~2곳), 균형있게(하루 3~4곳), 빡빡하게(하루 5곳 이상)} | 단일 | 밀도 지정 없음 | 내부 밀도 파라미터='균형있게' 상당(FD-U1-06) | US-E1-15 |

공통 속성: `accountId`(필수·유니크, Profile 1:1), `updatedAt`(필수 — 저장 즉시 반영의 기준 시각 [US-E1-12]).

### 11.2 불변식

- **INV-PR1 (값 도메인 폐쇄)**: 각 축의 저장값은 위 도메인 집합의 원소(집합 축은 부분집합)만 허용 — 도메인 밖 값 저장 불가 [BR-U1-29].
- **INV-PR2 (미설정≠중립)**: 저장 계층에는 중립 기본값이 존재하지 않는다. `NULL`(미설정)과 사용자가 명시 선택한 값(예: 이동=대중교통을 직접 고름)은 항상 구분 가능해야 한다 — 직렬화 왕복에서도 보존 [US-E1-14, PBT 속성 U1-P7].
- **INV-PR3 (예산 쌍 정합)**: `budgetRawAmount≠NULL ⇒ budgetTier = 매핑함수(budgetRawAmount)` — 원값과 구간은 항상 동시 저장·상호 정합. 1박 가격대 환산은 저장하지 않는다(여행 생성 시점 파생 — U4 소관) [G26].
- **INV-PR4 (동행 구조)**: companionType은 단일값, petFlag는 독립 불리언 — petFlag만 true이고 companionType=NULL인 상태 허용(동행 미설정+반려동물 동반) [G19].
- **INV-PR5 (무실패 보장)**: 7축 전부 NULL이어도 getPreferences는 항상 완전한(모든 축이 값 또는 중립 기본값으로 채워진) 응답을 반환한다 — 미설정만으로 일정 생성이 실패하지 않는다 [US-E1-14].

---

## 12. BannedWordDictionary — 금칙어 사전 (C3)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| dictVersion | 문자열(버전) | 필수 · 유니크 | 사전 버전 — 데이터 주도 교체 가능 구조(P8 기본 사전 확보 전 임시 최소 사전으로 개발) [P8, unit-of-work U1 리스크] |
| entries | 목록<{word: 문자열, category: 열거(욕설·차별·성인·사칭·기타)}> | 필수 | 매칭 어휘. 매칭 원문은 검증 응답에 미포함(우회 학습 방지) [component-methods C3.checkText] |
| deployedAt | 시각(UTC) | 필수 | 배포 시각 |
| active | 불리언 | 필수 | 활성 버전 1개 |

**불변식**
- **INV-B1 (활성 유일)**: active=true인 버전은 항상 정확히 1개.
- **INV-B2 (fail-closed)**: 활성 사전을 로드할 수 없는 상태에서 검증 요청은 통과(Pass)로 처리되지 않는다 — 저장 보류+재시도 안내 [components.md C3, BR-U1-18].
- **INV-B3 (일관 적용)**: 닉네임(M2)·여행 제목(M6 — U4)·후속 UGC 전부 동일 사전·동일 기준 [G23, N6, PRD 10-4].

---

## 13. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| Account·SocialIdentity | US-E1-01, US-E1-16 | D18, D22, D33/N1, G20, G22, C4 |
| EmailVerification | US-E1-01 | G22 |
| TermsVersion·ConsentRecord | US-E1-02, US-E1-17, US-E1-18 | N2/D34, N3, N8 |
| MarketingConsent | US-E1-02 | N8 |
| LocationConsentState·LocationLegalLog | US-E1-04, US-E1-17 | N2/D34, G182 |
| RefreshSession | US-E1-01 (+U2 US-E2-01 세션 기반) | D36, SECURITY-12 |
| DeletionSchedule | (US-E09-09 소비 — U1은 데이터 모델·골격) | D18, D34, C4 |
| Profile | US-E1-03, US-E1-11 | G23, G24/G157 |
| PreferenceSet | US-E1-05~10, US-E1-12~15 | D26/Δ2, G19, G26, D25/Δ1 |
| BannedWordDictionary | US-E1-03, US-E1-12 | G23, P8, N6 |
