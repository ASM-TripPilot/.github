# U1 기반·계정·온보딩 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/onboarding` + `shared/`(api·ui·storage·validation·location) 골격
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `onboarding` 행의 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M1·M2 메서드를 참조하고, 폼 검증 규칙은 [business-rules.md](./business-rules.md) BR-U1-xx의 클라이언트측 대응(UX용 사본 — **판정 정본은 항상 서버**, D28 원칙 준용)이다.
> 기술 중립: 상태 관리 라이브러리·내비게이션 구현체·스타일 시스템 선정은 NFR/Code Generation 소유. 본 문서는 컴포넌트 책임·props/state 개요·상호작용·검증·서버 능력 매핑까지 규정한다.

## 1. 온보딩 화면 플로우 (정본)

```text
[스플래시 분기 — U2 소유, U1은 부트스트랩 계약 공급 BR-U1-44]
        │
        ▼
┌─ AuthLandingScreen (로그인/회원가입) ────────────────────────────┐
│   소셜 4종 버튼 · '회원가입'(수단 비종속) · '이미 계정이 있으신가요? 로그인' │
└──────────────────────────────────────────────────────────────┘
   │ 소셜                     │ 이메일 가입                │ 이메일 로그인
   ▼                          ▼                           ▼
AgeConfirmSheet          EmailSignUpScreen           EmailSignInScreen
(소셜 경로 연령 확인)       (이메일·비밀번호·연령)          (+ 비밀번호 재설정 링크)
   │                          │                           │
   │                          ▼                           │
   │                EmailVerificationPendingScreen        │
   │                (인증 안내·재발송, 24h/분당1·일5)         │
   │                          │ 링크 인증 완료               │
   ▼                          ▼                           ▼
┌─ ConsentScreen (약관 동의 — 최초 1회) ──────────────────────────┐
│   필수 3종 분리 체크(이용약관·개인정보·위치기반서비스) + 선택 2종        │
│   (마케팅·GPS 기록 옵트인) + 전체 동의                            │
└──────────────────────────────────────────────────────────────┘
        │ (약관 개정 시 재진입 지점: ReconsentScreen — 분기 연결은 U2)
        ▼
NicknameScreen (자동 생성 기본값 · 수정 선택)
        │  ← 여기까지가 온보딩 완료 필수 구간 (약관+닉네임, G24/G157)
        ▼
┌─ PreferenceWizard (취향 7단계 — 전 단계 선택적) ────────────────┐
│  ① StyleStep → ② BudgetStep → ③ CompanionStep → ④ ActivityStep │
│  → ⑤ TransportStep → ⑥ FoodStep → ⑦ PaceStep                  │
│  각 단계: 진행률 · 이전 · 건너뛰기 / 전 구간: '나중에 설정하고 시작'    │
└──────────────────────────────────────────────────────────────┘
        │ (완주 또는 일괄 탈출)
        ▼
   홈 대시보드 (U2 — 빈 홈 첫 행동 유도)
```

**전역 규칙**
- 온보딩 전체(로그인·약관·닉네임·취향)는 **탭바 숨김** 화면군 — 전체 폭 단일 CTA 사용 [US-E2-04, components.md §6.2 ui].
- 온보딩에서 OS 위치 권한 다이얼로그를 호출하지 않는다 — just-in-time 원칙(§7 프리프롬프트 프레임) [US-E1-04].
- 모든 서버 오류는 표준 오류 타입으로 정규화되어 침묵 실패 없이 표면화(shared/api) [ADR-0011 클라이언트측].
- 재개: 앱 재진입 시 부트스트랩 `nextStep`/`M2.getOnboardingState`에 따라 미완료 필수 단계(약관·닉네임)로 복귀. 취향 단계 이탈은 완료 처리 [BR-U1-26, FLOW-4].

## 2. 컴포넌트 계층

```text
features/onboarding
├─ OnboardingNavigator                    — 온보딩 스택 컨테이너(탭바 숨김 구간)
│  ├─ AuthLandingScreen
│  │  ├─ SocialLoginButtonGroup           — Google·Apple·카카오·네이버 4버튼
│  │  ├─ EmailAuthEntry                   — '회원가입'(수단 비종속 라벨)·로그인 전환 링크
│  │  └─ AuthErrorBanner                  — 취소/제공자 오류/재시도
│  ├─ AgeConfirmSheet                     — 생년월일 또는 '만 14세 이상' 확인 (소셜·이메일 공용)
│  ├─ EmailSignUpScreen
│  │  ├─ EmailField / PasswordField(+PasswordPolicyHint)
│  │  └─ (AgeConfirmSheet 포함 호출)
│  ├─ EmailSignInScreen (+PasswordResetLink)
│  ├─ PasswordResetRequestScreen / PasswordResetScreen
│  ├─ EmailVerificationPendingScreen
│  │  └─ ResendButton(쿨다운 카운트다운)
│  ├─ ConsentScreen
│  │  ├─ AllAgreeToggle
│  │  ├─ RequiredConsentItem ×3           — 이용약관·개인정보·위치기반서비스(분리 필수)
│  │  ├─ OptionalConsentItem ×2           — 마케팅 수신 · GPS 기록 옵트인
│  │  └─ PolicyViewerModal                — 문안 본문 열람(버전 표기)
│  ├─ ReconsentScreen                     — 중대 개정 재동의(변경 요약·blocking 목록)
│  ├─ NicknameScreen
│  │  ├─ NicknameField(인라인 검증)
│  │  ├─ RegenerateButton('다른 닉네임 추천')
│  │  └─ SuggestionChips(중복 시 대체 추천 원탭)
│  └─ PreferenceWizard
│     ├─ WizardHeader                     — ProgressBar(7단계)·BackButton
│     ├─ EscapeHatchButton                — '나중에 설정하고 시작'(일괄 탈출)
│     ├─ StepFooter                       — '다음'/'건너뛰기'
│     ├─ StyleStep / ActivityStep / TransportStep / FoodStep   — 복수 선택 칩 그리드
│     ├─ BudgetStep                       — 4구간 카드 + 직접 총액 입력 전환
│     ├─ CompanionStep                    — 단일 선택 + 반려동물 토글(분리)
│     └─ PaceStep                         — 3단계 단일 선택
└─ shared/location
   └─ LocationPrePromptFrame              — 위치 권한 just-in-time 프리프롬프트 프레임(U1은 프레임만)
```

---

## 3. 인증 화면군

### 3.1 AuthLandingScreen
- **props**: 없음(라우트 루트) · **state**: 제출 중 플래그, 마지막 오류(취소/제공자 오류 구분).
- **상호작용**: 소셜 버튼 탭 → 제공자 인증 → `M1.signUpWithSocial` [FLOW-1]. 응답 `nextStep(consent|onboarding|home)`에 따라 라우팅. 취소 시 "로그인이 취소되었습니다" 배너+화면 유지, 제공자 오류 시 재시도 버튼 [US-E1-01]. '회원가입' → EmailSignUpScreen, '로그인' 링크 → EmailSignInScreen 전환.
- **분기 처리**: `EmailAlreadyRegistered`성 충돌 응답(기존 수단 유도 — BR-U1-03) 시 "이미 {수단}으로 가입된 이메일" 안내+해당 수단 버튼 강조. 삭제 유예 계정 응답 시 복구 확인 다이얼로그(BR-U1-39).
- **사용 서버 능력**: `M1.signUpWithSocial`, `M1.signIn`.

### 3.2 AgeConfirmSheet
- **props**: `mode(BIRTH_DATE|SELF_DECLARED 선택 제공)`, `onConfirm(ageConfirmation)` · **state**: 입력값·검증 오류.
- **폼 검증(서버 규칙 대응)**: 생년월일 형식·미래 날짜 불가. 만 14세 미만 판정 시 진행 차단+사유 안내 — 서버 `AgeRestricted`(BR-U1-05)의 UX 선반영(정본은 서버).
- **상호작용**: 가입 요청의 `ageConfirmation` 인자로 동봉 — 소셜·이메일 경로 동일 적용 [US-E1-16]. 확인 통과 전 계정 활성화·온보딩 진행 불가.
- **사용 서버 능력**: 자체 호출 없음(가입 메서드의 인자 공급).

### 3.3 EmailSignUpScreen
- **props**: 없음 · **state**: 이메일·비밀번호 입력, 필드별 인라인 오류, 제출 중.
- **폼 검증(서버 규칙 대응)**: 이메일 형식 / 비밀번호 8자+영문·숫자 포함(PasswordPolicyHint가 조건 충족을 실시간 체크 표시 — BR-U1-08 사본. 유출 목록 검증은 서버 전용) / 연령(AgeConfirmSheet).
- **상호작용**: 제출 → `M1.signUpWithEmail` [FLOW-2]. `WeakPassword(reasons)` → 조건별 인라인 표시. `EmailAlreadyRegistered` → 로그인·비밀번호 재설정 유도(계정 미생성 — BR-U1-01). 성공 → EmailVerificationPendingScreen.
- **사용 서버 능력**: `M1.signUpWithEmail`.

### 3.4 EmailVerificationPendingScreen
- **props**: `email`, `verificationSentAt` · **state**: 재발송 쿨다운(초), 일 한도 도달 여부.
- **상호작용**: 발송 이메일 안내+"메일이 안 왔나요?" 재발송 버튼. 재발송 → `M1.resendVerificationEmail` — `RateLimited(retryAfter)` 시 카운트다운 비활성(BR-U1-13). 인증 링크는 메일 앱에서 열림 → 딥링크 복귀 시 `M1.verifyEmail` 결과에 따라 ConsentScreen 진행 또는 `TokenExpired` 재발송 안내 [FLOW-2 S8]. 미인증 상태에서 다음 단계 진입 시도는 이 화면으로 회수(BR-U1-14).
- **사용 서버 능력**: `M1.resendVerificationEmail`, `M1.verifyEmail`.

### 3.5 EmailSignInScreen / PasswordReset 2화면
- **state**: 자격 증명 입력·오류(단일 문구 — 존재 여부 비노출 BR-U1-11)·잠금 안내(`AccountLocked(until)` 남은 시간).
- **상호작용**: `M1.signIn` — 미인증 계정이면 `nextStep=verifyEmail` → 3.4로. 삭제 유예 계정이면 복구 확인 경로(BR-U1-39). 재설정: `M1.requestPasswordReset`(존재 여부 무관 동일 성공 안내) → 메일 링크 → `M1.resetPassword`(새 비밀번호 — 3.3과 동일 정책 검증, 성공 시 전 기기 재로그인 안내 BR-U1-10).
- **사용 서버 능력**: `M1.signIn`, `M1.requestPasswordReset`, `M1.resetPassword`.

---

## 4. 동의 화면군

### 4.1 ConsentScreen
- **props**: `termsVersions`(현행 버전 메타 — 문안 버전 표기용) · **state**: 5개 체크 상태(필수 3+선택 2), 제출 중.
- **폼 검증(서버 규칙 대응)**: '다음' 활성 ⇔ 필수 3종 전부 체크(선택 무관) — BR-U1-21의 클라이언트 게이트(PBT U1-P17). 미동의 시 미동의 항목 하이라이트 안내. **위치기반서비스 약관은 이용약관·개인정보와 분리된 별도 필수 행** [N2, US-E1-17]. GPS 기록 옵트인은 "여행 발자취 보관" 용도 설명+미동의 진행 가능 명시.
- **상호작용**: AllAgreeToggle=5항목 전체 설정/해제 동치. 각 항목 '보기' → PolicyViewerModal(`M1.getPolicyDocuments` — 버전 본문). 제출 → `M1.recordConsent(ConsentBundle)` [FLOW-3] → 성공 시 NicknameScreen.
- **사용 서버 능력**: `M1.recordConsent`, `M1.getPolicyDocuments`.

### 4.2 ReconsentScreen (N3 — 스플래시 분기 연결은 U2)
- **props**: `requirement: {blocking[], nonBlocking[]}`(`M1.getRequiredConsents` 응답) · **state**: blocking 항목 체크 상태.
- **상호작용**: 중대 변경 요약·전문 열람 제시, blocking 전체 동의 시 `M1.reconsent` → 게이트 해제·원래 목적지 진행. 미완료 시 서비스 진입 불가 상태 유지(BR-U1-24). 경미 변경(nonBlocking)은 이 화면이 아닌 인앱 공지 채널.
- **사용 서버 능력**: `M1.getRequiredConsents`, `M1.reconsent`.

---

## 5. NicknameScreen

- **props**: `initialNickname`(가입 직후 자동 생성값 — `M2.generateNickname`) · **state**: 입력값, 검증 상태(대기/검증 중/통과/거절 사유), 대체 추천 목록.
- **폼 검증(서버 규칙 대응)**: 길이 2~20자 즉시 인라인(BR-U1-17 사본). 금칙어·중복은 서버 판정(`M2.updateNickname`)의 `Rejected(LENGTH|FORBIDDEN|DUPLICATE, suggestions)`를 인라인 오류로 매핑(BR-U1-18·19) — 위반 시 진행 차단.
- **상호작용**: 자동 생성값이 기본 채움 — 무수정 '다음'으로 통과 가능(BR-U1-20), "닉네임은 나중에 설정에서 바꿀 수 있어요" 캡션 상시 노출. RegenerateButton → `M2.generateNickname` 재호출. DUPLICATE 시 SuggestionChips 원탭 적용. 통과 시 PreferenceWizard로(이 시점에 온보딩 완료 조건 충족 — BR-U1-26).
- **사용 서버 능력**: `M2.generateNickname`, `M2.updateNickname`.

---

## 6. PreferenceWizard (취향 7단계)

### 6.1 컨테이너 규칙
- **state**: 현재 단계 인덱스(1~7), 단계별 임시 선택값, 저장 중 플래그.
- **상호작용**: 단계 저장은 축 단위 patch — `M2.updatePreferences` 즉시 호출(저장 즉시 반영 BR-U1-31). '이전'=직전 단계 복귀·수정 가능, '건너뛰기'=해당 축 미변경(NULL 유지 — 중립값 전송 금지, BR-U1-27·INV-PR2), EscapeHatchButton=잔여 단계 일괄 건너뛰기→`M2.completeOnboarding`→홈 [FLOW-4]. 7단계 완주 시에도 동일하게 completeOnboarding→홈. WizardHeader 진행률은 7단계 기준 표시 [US-E1-11].
- **재개**: 재진입 시 `M2.getOnboardingState`로 위치 복원(단, 약관+닉네임 통과 상태면 이탈=완료 — 재진입 시 홈).
- **사용 서버 능력**: `M2.updatePreferences`, `M2.getOnboardingState`, `M2.completeOnboarding`.

### 6.2 단계별 컴포넌트 (값 도메인 = [domain-entities.md](./domain-entities.md) §11.1 정본)

| 단계 | 컴포넌트 | 선택 UI | 클라이언트 검증(서버 BR-U1-29 대응) | 근거 |
|---|---|---|---|---|
| ① 스타일 | StyleStep | 복수 선택 칩(휴양·관광·액티비티·미식·쇼핑·자연·문화예술) | '다음' 활성=1개 이상 선택(건너뛰기는 별도 경로) | US-E1-05 |
| ② 예산 | BudgetStep | 4구간 카드(저가/중간/고급/럭셔리) ↔ 직접 총액 입력 전환 | 총액=양수 숫자만. 카피='대략적인 여행 씀씀이', '항공 제외·합산' 정의는 직접 입력 모드에서만 노출. 원값 입력 시 매핑 구간 미리보기 표시(서버와 동일 매핑 함수 사본 — BR-U1-32) | US-E1-06, D26/Δ2, G26 |
| ③ 동행 | CompanionStep | 기본 유형 **단일 선택**(혼자·커플·친구·가족(아동 동반)·부모님) + '반려동물 동반' **분리 토글** | 단일 선택 강제(라디오 성격), 토글은 유형 미선택과 독립 조작 가능(INV-PR4) | US-E1-07, G19 |
| ④ 활동 | ActivityStep | 복수 선택 칩(자연·역사문화·테마파크·맛집투어·카페·전시·야경·쇼핑·스포츠) | — | US-E1-08 |
| ⑤ 이동 | TransportStep | 복수 선택 칩(도보·대중교통·렌터카·택시·자전거) | 안내 문구에 소요시간 미표시 원칙 반영 금지 항목 없음 — 단 결과 화면 어디에도 소요시간 노출 금지(D25/Δ1)는 소비 화면(U5)의 계약 | US-E1-09, D25/Δ1 |
| ⑥ 음식 | FoodStep | 복수 선택 칩(한식·양식·일식·중식·아시안 등) | — | US-E1-10 |
| ⑦ 페이스 | PaceStep | 단일 선택 카드(느긋하게 1~2곳 / 균형있게 3~4곳 / 빡빡하게 5곳+) | 단일 선택 강제 | US-E1-15 |

**공통**: 각 단계 '건너뛰기' 시 해당 축 미설정 저장 — 미설정 동작(중립 기본값·필터 미적용)의 안내는 강요 없이 캡션 수준 [US-E1-14]. 설정 화면(U8 통합)에서 동일 Step 컴포넌트를 재사용해 온보딩과 동일 선택지 보장 [US-E1-12 — 온보딩 흐름 내 수정은 U1, 마이페이지 재진입은 U8].

---

## 7. LocationPrePromptFrame (shared/location — U1은 프레임만)

- **책임**: 위치 권한 just-in-time 프리프롬프트의 **재사용 프레임** — 사용 목적 설명 카드 + '계속'(→OS 다이얼로그 호출) + '나중에' 2액션. **U1에서는 온보딩 중 어떤 화면도 이 프레임을 발화하지 않는다** — 실제 발화 지점은 U3('내 주변 숙소 탐색' 직전)·U6(여행 중 실행 첫 진입 직전) [US-E1-04, unit-of-work U1 §제외].
- **props**: `purposeContext(NEARBY_SEARCH|TRIP_EXECUTION)`(발화 맥락별 목적 문구), `onProceed`, `onDefer` · **state**: 1회 노출 이력(맥락별 — 동일 맥락 재노출 억제).
- **상호작용·상태 관리(U1 소유분)**:
  - OS 권한 상태(L1: GRANTED/DENIED/NOT_DETERMINED)의 단말 정본 관리 + 변경 시 서버 미러 동기화(`M1.updateLocationConsent(layer=OS_MIRROR)`).
  - 3층 조합 판정기 — [domain-entities.md](./domain-entities.md) §7.3 매트릭스의 런타임 구현(서버와 명세 공유). 소비자(U3·U6 화면)는 `effectiveCapabilities`만 질의.
  - 거부 시 폴백 안내: "현재 위치 기능은 설정에서 켤 수 있습니다" + 등록 숙소/여행지 중심 기준 동작 — 거부만으로 어떤 플로우도 중단하지 않음 [US-E1-04].
  - GPS 옵트인(L3)은 이 프레임의 소관이 아님 — 약관 화면(4.1)·설정(U8)에서 관리. 철회 시 즉시 수집 중단은 판정기 상태 반영으로 보장(BR-U1-42).
- **사용 서버 능력**: `M1.getLocationConsentState`, `M1.updateLocationConsent`.

---

## 8. shared/ 골격 중 U1 완성분 (요약)

| 계층 | U1 완성 범위 | 근거 |
|---|---|---|
| shared/api | 액세스 토큰 첨부·만료 시 자동 리프레시 회전(동시 갱신 직렬화 — FLOW-5 소비자)·표준 오류 정규화·부트스트랩 계약 소비 자리 | D36, G5, ADR-0011 |
| shared/storage | 토큰 OS 보안 저장소 보관(SECURITY-12) — 그 외 오프라인 큐 등은 후속 유닛 | §6.4 |
| shared/ui | 온보딩 화면군의 디자인 시스템 기초(입력 필드·체크 행·칩·진행률·전체 폭 CTA·탭바 숨김 규칙) | US-E2-04, G143(온보딩=KWCAG 우선 대상) |
| shared/validation | 폼 수준 공통 유틸(길이·형식·날짜) — C2 규칙 명세 소비부는 U5 | D28 |

## 9. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M1·M2) |
|---|---|---|
| AuthLandingScreen | US-E1-01 | signUpWithSocial · signIn |
| AgeConfirmSheet | US-E1-16 | (가입 메서드 ageConfirmation 인자) |
| EmailSignUpScreen | US-E1-01, US-E1-16 | signUpWithEmail |
| EmailVerificationPendingScreen | US-E1-01 | verifyEmail · resendVerificationEmail |
| EmailSignInScreen / PasswordReset | US-E1-01 | signIn · requestPasswordReset · resetPassword |
| ConsentScreen | US-E1-02, US-E1-17 | recordConsent · getPolicyDocuments |
| ReconsentScreen | US-E1-18 | getRequiredConsents · reconsent |
| NicknameScreen | US-E1-03 | generateNickname · updateNickname |
| PreferenceWizard(7 Step) | US-E1-05~11, US-E1-15 | updatePreferences · getOnboardingState · completeOnboarding |
| (점진 카드 데이터 — 노출은 U2/U4) | US-E1-11 | getPendingPreferencePrompts |
| LocationPrePromptFrame | US-E1-04, US-E1-17 | getLocationConsentState · updateLocationConsent |
| (취향 소비 — U3/U5 화면) | US-E1-13, US-E1-14 | getPreferences · getPersonalizationInput |
