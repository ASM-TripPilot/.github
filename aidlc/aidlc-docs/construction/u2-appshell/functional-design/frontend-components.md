# U2 앱 셸·홈·내비게이션 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/home` + 내비게이션 컨테이너(5탭 셸) + `shared/ui`(탭바·디자인 시스템 확장)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `home` 행·§6.2 `ui`(공용 탭바·탭바 숨김 규칙)를 컴포넌트 계층으로 상세화한다. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M1.getBootstrapStatus·홈 집약 메서드를 참조하고, 판정 규칙은 [business-rules.md](./business-rules.md) BR-U2-xx(판정 정본은 항상 서버·계약 — D28 원칙 준용)이다.
> 기술 중립: 상태 관리 라이브러리·내비게이션 구현체·스타일 시스템·스토어 SDK 선정은 NFR/Code Generation 소유. 본 문서는 컴포넌트 책임·props/state 개요·상호작용·`data-testid` 규칙·서버 능력 매핑 + Process Flows + Testable Properties를 규정한다.
> `data-testid` 규칙: `appshell-{screen}-{role}` (예: `appshell-splash-progress`, `appshell-tabbar-home`). 스크린리더/자동화 테스트 앵커로 사용.

## 1. 앱 셸 진입 플로우 (정본)

```text
[콜드 스타트]
        │
        ▼
┌─ SplashScreen (탭바 숨김) ─────────────────────────────────┐
│   M1.getBootstrapStatus(accessToken, appVersion) 호출          │
│   진행 표시 유지 · 3초 타임아웃 감시(BR-U2-03)                    │
│   resolveSplashDestination(BootstrapInfo) → 목적지 5종(BR-U2-01) │
└──────────────────────────────────────────────────────────────┘
   │FORCE_UPDATE   │LOGIN            │RECONSENT      │ONBOARDING        │HOME
   ▼               ▼                 ▼               ▼                  ▼
ForceUpdateScreen  [U1 onboarding]   [U1 onboarding] [U1 onboarding]    AppShell
(스토어 이동·      AuthLanding        ReconsentScreen  잔여 단계          (5탭 셸)
 전면 차단)                          (N3)             (nextStep 2차 해소)   │
                                                                          ▼
                                                                    HomeDashboard
                                                                    (홈 탭 루트)

[타임아웃(3초 초과) 폴백 — BR-U2-03]
   로컬 토큰 미만료 → HOME 진입 + 백그라운드 재검증 플래그
   로컬 토큰 만료   → LOGIN
   (온라인 복구 시 동일 멱등 API 재검증 → forceUpdate/재동의 확인 시 해당 게이트 전이)

[딥링크·알림 진입 — BR-U2-06/07]
   세션 없음 → 로그인 게이트(딥링크 목적지 인증 후 이어서 라우팅, D22/Δ5)
   세션 유효 → 대상 탭 활성화 + 그 탭 스택에 push(G7)
```

**전역 규칙**
- 스플래시·강제 업데이트·온보딩 전체는 **탭바 숨김** 화면군(BR-U2-08) [US-E2-04].
- 스플래시 판정은 순수 함수 `resolveSplashDestination`으로 분리 — 어떤 입력 조합에도 정확히 1개 목적지, 우선순위 불변(BR-U2-01·02, PBT 속성 U2-P1).
- 모든 서버 오류는 표준 오류 타입으로 정규화되어 침묵 실패 없이 표면화(shared/api) [ADR-0011 클라이언트측, US-E2-01·02].
- 온보딩·인증·재동의 화면 자체는 U1(`features/onboarding`) 소유 — U2는 스플래시에서 그 진입점으로 **분기**만 한다(FLOW-1).

## 2. 컴포넌트 계층

```text
features/home + 내비게이션 컨테이너(shared/ui)
├─ SplashScreen                            — 부트스트랩 호출·5분기 판정(탭바 숨김)
│  └─ SplashProgressIndicator              — 검증 진행 표시(3초 타임아웃·침묵 실패 금지)
├─ ForceUpdateScreen                       — 최소 버전 미달 전면 차단·스토어 이동(탭바 숨김)
├─ AppShell                                — 5탭 바텀 내비 컨테이너(탭 상태·딥링크 라우팅 소유)
│  ├─ TabBar                               — 단일 공용 바텀 탭바(5탭·숨김 규칙·선택 시각 구분)
│  ├─ HomeTabRoot → HomeDashboard          — 홈 대시보드(카드 슬롯 렌더·부분 로딩·빈 상태)
│  │  ├─ HomeTopBar                        — 워드마크 + 알림 진입점(읽지 않은 배지)
│  │  ├─ ActiveTripCard                    — 진행 중 여행(D-day·진행률)→execution 허브(U6 데이터)
│  │  ├─ UpcomingTripCard                  — 예정 여행(D-day·등록 숙소 수·일정 유무, U4 데이터)
│  │  ├─ QuickActionCard                   — '여행 만들기' + 첫 사용자 온램프 2종
│  │  ├─ TrendingPlacesCard                — '지금 인기 있는 장소'(U3 M7 데이터·부분 응답)
│  │  ├─ MemoryCard                        — '여행 추억 다시 보기'(U7 데이터)
│  │  ├─ PreferencePromptCard              — 건너뛴 취향 점진 설정(U1 M2 데이터)
│  │  └─ HomeEmptyState                    — 첫 사용자 빈 상태(첫 행동 유도)
│  ├─ ExploreTabRoot                       — 탐색 랜딩('무엇을 찾을까요?' 3카드) — 콘텐츠는 U3
│  │  └─ PlaceSaveOnramp                   — 장소 검색·저장 온램프 셸(US-E2-05, 백엔드 U3)
│  ├─ ItineraryTabRoot                     — 일정 탭 루트·빈 상태 — 콘텐츠는 U5/U6
│  ├─ RecordTabRoot                        — 기록 탭 루트·빈 상태 — 콘텐츠는 U7
│  └─ MyTabRoot                            — 마이 탭 루트·빈 상태 — 콘텐츠는 U8
└─ shared/ui (U2 확장분)
   ├─ TabBar(공용)·TabBarVisibilityRule    — 탭바 숨김 규칙 단일 소스 + 린트 규칙
   ├─ EmptyState / LoadingSkeleton / ErrorRetry  — 표준 패턴(ADR-0011)
   └─ DeepLinkRouter                       — 딥링크·알림 → 대상 탭 활성화 + 스택 push(G7)
```

> **후행 유닛이 채우는 자리**: ExploreTabRoot·ItineraryTabRoot·RecordTabRoot·MyTabRoot와 홈의 데이터 슬롯(ActiveTripCard·UpcomingTripCard·TrendingPlacesCard·MemoryCard)은 U2가 **루트·빈 상태·진입 규칙·슬롯 스키마**까지 완성하고, 실제 콘텐츠·데이터는 후행 유닛(U3~U8)이 점진 공급한다(FD-U2-05) [unit-of-work U2 §제외·산출물].

---

## 3. SplashScreen (스플래시 분기)

- **props**: 없음(앱 진입 루트) · **state**: 부트스트랩 조회 상태(대기/응답/타임아웃), 로컬 토큰 상태(미만료/만료), 백그라운드 재검증 플래그.
- **상호작용**: 콜드 스타트 시 `M1.getBootstrapStatus(accessToken, appVersion)` 호출 → 응답을 `resolveSplashDestination`(순수 함수)에 통과시켜 5목적지로 라우팅(BR-U2-01). ONBOARDING 목적지는 `M2.getOnboardingState.nextStep`로 정확 재개 단계(약관/닉네임) 2차 해소(FD-U2-02). 3초 타임아웃 시 로컬 토큰 폴백(BR-U2-03) — 미만료 HOME+백그라운드 재검증, 만료 LOGIN. 검증 지연 중 `SplashProgressIndicator` 유지(침묵 실패 금지).
- **data-testid**: `appshell-splash-root`, `appshell-splash-progress`.
- **폼 검증(서버 규칙 대응)**: 판정 정본은 서버 계약(BootstrapInfo) — 클라이언트 결정 함수는 그 계약의 소비자 사본(U1 FD-U1-10과 명세 공유, PBT U2-P1로 동치 검증).
- **사용 서버 능력**: `M1.getBootstrapStatus`, `M1.getRequiredConsents`(재동의 상세), `M2.getOnboardingState`(잔여 단계).

## 4. ForceUpdateScreen (강제 업데이트 게이트)

- **props**: `minSupportedVersion`, `storeUrl?` · **state**: 없음(전면 차단 정적 화면).
- **상호작용**: 최소 버전 미달 시 진입 — 서비스 진입(홈·로그인 포함)을 전면 차단하고 스토어 이동 액션만 제공(BR-U2-04). storeUrl 부재 시 플랫폼 기본 스토어 스킴. 뒤로가기·우회 진입 불가(하드 제약 라우팅 가드).
- **data-testid**: `appshell-forceupdate-root`, `appshell-forceupdate-store`.
- **사용 서버 능력**: `M1.getBootstrapStatus`(minSupportedVersion·forceUpdate — 판정 입력, 화면 자체 호출 없음).

## 5. AppShell + TabBar (5탭 바텀 내비)

### 5.1 AppShell (내비게이션 컨테이너)
- **props**: `initialTab(딥링크 진입 시 대상 탭)` · **state**: TabNavigationState([domain-entities.md](./domain-entities.md) §3 — activeTab·tabStacks·scrollPositions·sessionEpoch).
- **상호작용**: 탭별 독립 스택 유지·세션 보존(BR-U2-05, INV-TN1). 딥링크·알림 진입은 `DeepLinkRouter`가 대상 탭 활성화 + 그 탭 스택 push(BR-U2-06, G7). 비로그인 딥링크 진입은 로그인 게이트 경유(BR-U2-07, D22). 앱 재시작 시 sessionEpoch 교체로 전체 초기화(INV-TN2).
- **data-testid**: `appshell-shell-root`.
- **사용 서버 능력**: (탭 라우팅·상태 보존은 클라이언트 전용 — US-E2-03·04 서버 메서드 불요).

### 5.2 TabBar (단일 공용 컴포넌트)
- **props**: `activeTab`, `unreadBadge?(마이/알림 관련)` · **state**: 없음(제어 컴포넌트).
- **상호작용**: 5탭 고정(홈·탐색·일정·기록·마이) 동일 형태 노출·선택 탭 시각 구분(US-E2-03). 현재 탭 재탭 시 해당 탭만 스크롤 탑+스택 루트 복귀(BR-U2-05, INV-TN3). 탭바 숨김 규칙(BR-U2-08)은 `TabBarVisibilityRule` 단일 소스 + 린트 규칙으로 강제 — 몰입 화면군(스플래시·강제 업데이트·온보딩·입력 폼·일정 생성/편집·여행 중 실행·Plan-B·바텀시트)에서 숨김.
- **data-testid**: `appshell-tabbar-{home|explore|itinerary|record|my}`.
- **사용 서버 능력**: 없음(클라이언트 전용).
- **각 탭 루트 진입 규칙**: 홈=HomeDashboard / 탐색=탐색 랜딩(3카드) / 일정=진행 중 여행 있으면 execution 허브(M18.getActiveHub), 없으면 최근 여행 일정, 일정 없으면 등록·생성 유도 / 기록=여행 기록·요약 / 마이=마이페이지. 각 탭 데이터 0건은 해당 섹션 정본 빈 상태(BR-U2-13).

## 6. HomeDashboard (홈 대시보드)

- **props**: 없음(홈 탭 루트) · **state**: 슬롯별 로딩/성공/실패 상태(부분 응답 — INV-HD1).
- **상호작용**: 홈 집약 조회로 HomeDashboardModel 슬롯([domain-entities.md](./domain-entities.md) §4.1) 조립·렌더(BR-U2-09). 슬롯별 로딩 스켈레톤·부분 실패 격리(가용 카드 우선, 침묵 실패 금지). ActiveTripCard 탭 → execution 허브(U6). QuickActionCard '여행 만들기' → 여행 생성(U4). 첫 사용자 빈 상태(HomeEmptyState)는 첫 행동 유도(BR-U2-13). 진행률 표시 규칙(BR-U2-11): 여행 중=방문 체크 비율, 여행 전=D-day만.
- **data-testid**: `appshell-home-root`, `appshell-home-activetrip`, `appshell-home-trending`, `appshell-home-empty`.
- **슬롯별 사용 서버 능력**(US-E2-02 매핑):
  - ActiveTripCard: `M18.getActiveHub` · `M18.getTripProgress` (U6)
  - UpcomingTripCard: `M6.listTrips` (U4)
  - TrendingPlacesCard: `M7.getTrendingPlaces` (U3)
  - MemoryCard: `M12.getVisitStats` (U7)
  - PreferencePromptCard: `M2.getPendingPreferencePrompts` (U1)
  - HomeTopBar 배지: `M14.getInbox`(미읽음 카운트, U8)
  - communityRecordCard: **미노출**(U10 전까지 — FD-U2-07)

## 7. 각 탭 루트 자리 (후행 유닛이 채움)

| 탭 루트 | U2 완성 범위 | 후행 유닛 콘텐츠 | data-testid |
|---|---|---|---|
| ExploreTabRoot | 탐색 랜딩 3카드(숙소/장소/여행자 일정 진입) + 장소 저장 온램프 셸(US-E2-05) | 숙소 탐색·장소 검색 백엔드(M7) = U3 | `appshell-explore-root` |
| ItineraryTabRoot | 루트·진입 규칙·빈 상태(일정 없음 → 등록·생성 유도) | 일정 콘텐츠 = U5, 활성 허브 = U6 | `appshell-itinerary-root` |
| RecordTabRoot | 루트·빈 상태(기록 0건) | 기록·회고 = U7 | `appshell-record-root` |
| MyTabRoot | 루트·빈 상태 | 마이페이지·설정 = U8 | `appshell-my-root` |

> 탐색 랜딩의 '여행자 일정(커뮤니티 둘러보기)' 카드는 진입점만 두되 커뮤니티 콘텐츠는 U10 — 전용 커뮤니티 탭은 두지 않는다(US-E2-03).

---

## Process Flows

> 참조: 규칙 [business-rules.md](./business-rules.md) BR-U2-xx, 데이터 [domain-entities.md](./domain-entities.md) INV-xx. 표기: 단계 `Sx`, 분기 `Bx`, 예외 `Ex`.

### FLOW-1 스플래시 부트스트랩 분기

**진입**: 콜드 스타트 → SplashScreen · **관련**: US-E2-01, US-E2-06 · BR-U2-01~03 · G5, N3, N4, D22

```text
S1 스플래시 표시(탭바 숨김) + M1.getBootstrapStatus(accessToken, appVersion) 호출, 3초 타임아웃 감시
   └─ E1 3초 초과(타임아웃) → 【페일오픈 폴백 BR-U2-03】
        ├─ 로컬 토큰 미만료 → HOME 진입 + 백그라운드 재검증 플래그(온라인 복구 시 S3 재수렴)
        └─ 로컬 토큰 만료   → LOGIN
S2 BootstrapInfo 수신 → resolveSplashDestination(순수 함수, BR-U2-01)
S3 우선순위 판정(BR-U2-02):
   ├─ B1 forceUpdate=true → FORCE_UPDATE(ForceUpdateScreen — 전면 차단·스토어 이동)
   ├─ B2 sessionState ∈ {NONE, EXPIRED} → LOGIN(U1 AuthLandingScreen)
   ├─ B3 reconsentRequired=true → RECONSENT(U1 ReconsentScreen, N3)
   ├─ B4 onboardingComplete=false → ONBOARDING
   │      └─ S4 M2.getOnboardingState.nextStep로 정확 재개 단계 2차 해소(약관/닉네임, FD-U2-02)
   └─ B5 그 외 → HOME(AppShell → HomeDashboard)
```

**사후 조건**: 어떤 BootstrapInfo 조합에도 목적지가 정확히 1개(완전성·상호배타) + 우선순위 불변(PBT 속성 U2-P1). 확인 불가(타임아웃) 시 페일오픈으로 전면 차단 회피(리스크 완화).

### FLOW-2 탭 전환·상태 보존·재탭

**진입**: AppShell 내 탭 조작 · **관련**: US-E2-03 · BR-U2-05 · G6

```text
S1 탭 A에서 하위 화면 탐색·스크롤 → tabStacks[A]·scrollPositions[A] 갱신(세션 메모리)
S2 탭 B로 전환 → activeTab=B, 탭 A 상태 보존(INV-TN1)
S3 다시 탭 A로 전환 → tabStacks[A]·scrollPositions[A] 복원(라운드트립 동일 — PBT 속성 U2-P2)
   ├─ B1 현재 활성 탭 재탭 → 해당 탭만 스크롤 탑 + 스택 루트 복귀(INV-TN3), 타 탭 무영향
   └─ B2 앱 재시작(sessionEpoch 교체) → 전체 초기화(INV-TN2)
```

### FLOW-3 딥링크·알림 진입

**진입**: 알림 탭·딥링크·공유 링크 · **관련**: US-E2-03 · BR-U2-06·07 · G7, D22/Δ5

```text
S1 딥링크 수신 → 세션 상태 확인
   ├─ B1 세션 없음 → 로그인 게이트(딥링크 목적지 보관 → 인증 후 이어서 라우팅, BR-U2-07)
   └─ B2 세션 유효 → S2
S2 DeepLinkRouter: 대상 화면이 속한 탭 활성화(activeTab 설정) + 그 탭 스택에 push(INV-TN4, G7)
   └─ B3 대상이 몰입 화면군 → 탭바 숨김 규칙 적용(BR-U2-08)
S3 타 탭 스택 보존(무영향) — 뒤로가기·닫기로 직전 탭 컨텍스트 복귀
```

---

## Testable Properties (PBT-U2 — 속성 식별 정본)

> PBT 전체 강제(PBT-01~10). 클라이언트 프레임워크: fast-check(§ U1 PBT-01 정합 — 시드 로깅·수축 필수). 아래 속성은 U2 Code Generation의 DoD 항목이다. unit-of-work U2 DoD가 명시한 속성(스플래시 분기 결정 함수·탭 상태 보존/복원 왕복)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**스플래시 분기 (클라이언트 순수 함수 — fast-check)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U2-P1 | 총함수·논리 게이트 (결정표) | **스플래시 분기 결정**: 임의의 BootstrapInfo(sessionState 3값 × onboardingComplete × reconsentRequired × forceUpdate) 전 조합에 대해 `resolveSplashDestination`은 (a) 정확히 1개 목적지 반환(총함수·완전성), (b) 목적지 집합 {FORCE_UPDATE, LOGIN, RECONSENT, ONBOARDING, HOME}이 입력 도메인을 상호배타 분할, (c) 우선순위 불변: forceUpdate=true ⇒ 항상 FORCE_UPDATE(타 필드 무관), reconsent가 온보딩/세션-완료 분기보다 우선(BR-U2-02), (d) 동일 입력→동일 출력(결정성), (e) 서버 공급자 판정(U1 FD-U1-10)과 동치 | BootstrapInfo 조합 생성기(sessionState 열거 3 × 불리언 3축 전수 24조합 exhaustive + SemVer 쌍) |
| U2-P3 | 단조성·전순서 | **강제 업데이트 버전 비교**: 임의의 (clientVersion, minSupportedVersion) SemVer 쌍에 대해 (a) forceUpdate = (clientVersion < minSupportedVersion)이 SemVer 전순서와 일치, (b) 버전 비교 단조성 — clientVersion을 올리면 forceUpdate가 false에서 true로 되지 않음(비차단 방향 단조), minSupportedVersion을 올리면 차단이 완화되지 않음, (c) 동일 버전은 미달 아님(경계 포함 판정) | SemVer 생성기(major/minor/patch 경계·프리릴리스 태그·동일 버전 케이스 포함) |

**탭 내비게이션 (클라이언트 상태 — fast-check)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U2-P2 | 라운드트립·상태 보존 | **탭 상태 보존/복원**: 임의의 탭 조작 시퀀스(전환·하위 화면 push/pop·스크롤·재탭)에 대해 (a) 탭 A 이탈 후 복귀 시 tabStacks[A]·scrollPositions[A] 라운드트립 동일(INV-TN1), (b) 재탭은 활성 탭만 스크롤 탑+스택 루트 복귀, 타 탭 상태 불변(INV-TN3 국소성), (c) sessionEpoch 교체(앱 재시작)는 전체 초기화(INV-TN2), (d) 딥링크 push는 대상 탭에만 반영, 타 탭 스택 보존(INV-TN4) | 탭 조작 명령 시퀀스 생성기(5탭 × {switch, push, pop, scroll, retap, restart, deeplink}, 길이 0~50) |

**홈 대시보드 조립 (클라이언트 순수 변환 — fast-check)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U2-P4 | 멱등성·부분 응답 완전성 | **홈 카드 조립**: 임의의 슬롯 데이터 집합(각 슬롯 {성공·데이터 있음 / 성공·0건 / 실패 / 로딩})에 대해 조립 결과는 (a) 항상 렌더 가능(어떤 슬롯 실패에도 전체 실패 없음 — INV-HD1 부분 응답 완전성), (b) 동일 입력 집합 재조립 시 동일 결과(멱등 — INV-HD2), (c) activeTripCard는 최대 1개 여행 참조(INV-HD3, D21), (d) communityRecordCard는 항상 미노출(INV-HD4), (e) 여행 전 카드는 진행률 미표시·여행 중 카드는 진행률 표시(INV-HD5) | 슬롯 상태 조합 생성기(8슬롯 × 4상태 + 여행 상태 {없음/예정/진행중} + 진행률 값) |

**딥링크 라우팅 게이트 (클라이언트 — fast-check)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U2-P5 | 게이트 불변식 | **비로그인 진입 가드**: 임의의 (진입 유형 {일반, 초대·공유 딥링크} × 세션 상태)에 대해 (a) 비로그인 + 딥링크 아닌 진입은 반드시 로그인 화면으로 귀결(홈·탐색 등 직접 도달 0 — BR-U2-07, D22), (b) 비로그인 + 딥링크 진입은 로그인 게이트 경유 후 목적지 이어서 라우팅, (c) 인증 완료 후에만 대상 탭 스택 push | 진입 유형 × 세션 상태 × 딥링크 목적지 조합 생성기 |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| 서버 홈 집약 조회(BFF성 — server/app) | No PBT properties identified | 후행 유닛 퍼사드(M6·M7·M12·M18·M14·M2)를 병렬 집계·부분 응답 조립하는 오케스트레이션 코드 — 보편 양화 가능한 도메인 불변식은 클라이언트 조립(U2-P4)이 소유. 서버측 부분 실패 격리·소유권 검증(SECURITY-08)은 예시 기반 통합 테스트로 검증. 슬롯 스키마 계약은 계약 테스트(후행 유닛이 채울 필드 고정) |
| AppConfig 조회(remote config 미러) | No PBT properties identified | 단일 설정 값 조회 어댑터 — 자체 도메인 로직 없음. 버전 비교 로직 자체의 속성은 U2-P3(스플래시측)에서 커버. 조회·캐시는 계약 테스트로 충분 |
| ForceUpdateScreen·각 탭 루트 자리(정적/후행 콘텐츠) | No PBT properties identified | 전면 차단 정적 화면·후행 유닛이 채우는 빈 껍데기 — 분기 판정은 U2-P1·U2-P3에 귀속, 렌더는 예시 기반 컴포넌트 테스트 |

### 커버리지 대조 (U2 DoD → 속성)

| U2 DoD 명시 속성 | 대응 |
|---|---|
| 스플래시 분기 결정 함수(세션 × 약관 버전 × 앱 버전 → 목적지, 정확히 1개·우선순위 불변) | U2-P1 (+ 버전 비교 U2-P3) |
| 탭 상태 보존·복원 왕복 | U2-P2 |
| (하드 제약 연계) 강제 업데이트 미달 버전 차단 | U2-P3 + 라우팅 가드 계약 테스트 |
| (하드 제약 연계) 비로그인 진입 딥링크 한정 | U2-P5 |
| (설계 보강) 홈 카드 부분 응답·조립 멱등 | U2-P4 |

**속성 합계: 5개** (스플래시 분기 2 · 탭 내비 1 · 홈 조립 1 · 딥링크 가드 1)

---

## 8. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력 |
|---|---|---|
| SplashScreen | US-E2-01 | M1.getBootstrapStatus · M1.getRequiredConsents · M2.getOnboardingState |
| ForceUpdateScreen | US-E2-06 | M1.getBootstrapStatus(minSupportedVersion·forceUpdate) |
| AppShell / TabBar | US-E2-03, US-E2-04 | (클라이언트 전용 — 탭 상태·딥링크 라우팅) |
| HomeDashboard | US-E2-02 | M6.listTrips · M18.getActiveHub/getTripProgress · M7.getTrendingPlaces · M12.getVisitStats · M14.getInbox · M2.getPendingPreferencePrompts |
| ExploreTabRoot / PlaceSaveOnramp | US-E2-05 | (셸만 — M7.searchPoi/savePoi/listSavedPois 백엔드는 U3) |
| ItineraryTabRoot | US-E2-03 | M18.getActiveHub · M8.getItinerary · M6.listTrips (진입 규칙, 콘텐츠 U5/U6) |
