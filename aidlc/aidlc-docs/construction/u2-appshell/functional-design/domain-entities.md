# U2 앱 셸·홈·내비게이션 — 도메인 데이터 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 범위: 앱 셸(스플래시·강제 업데이트)·홈 대시보드 프레임·5탭 내비게이션 + 서버 부트스트랩/홈 집약 API(server/app 소유)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `home` feature·§6.2 `ui`(탭바)·§3.1 여행 상태 머신(홈 카드 관련)을 데이터·상태 모델 수준으로 상세화한다. 기술 중립 — 저장 기술·상태 관리 라이브러리·직렬화 포맷은 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL 의미 명기), 근거 ID는 requirements.md(D·Δ·N·G)·stories.md(US)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U2-xx)을 참조.

## 0. 데이터 소유 지도

U2는 **거의 데이터를 소유하지 않는다** — 대부분 클라이언트 세션 상태이거나 타 모듈이 소유한 데이터의 읽기 전용 집계다. 소유·참조 관계를 명시한다.

```text
[서버 영속 — U2 소유]
AppConfig                       (remote config성 앱 구성 — 최소 지원 버전·스토어 링크; 유일 영속 엔티티)

[읽기 전용 집계 DTO — U2가 계약 정의·소비, 원천 데이터 소유는 타 유닛]
BootstrapInfo                   (U1 M1이 집계·공급 — 세션·온보딩·재동의·최소 버전; U2 스플래시가 소비)
HomeDashboardModel              (홈 집약 BFF 응답 — 슬롯별 원천은 U1/U3/U4/U6/U7)
  └─ TrendingPlace  (참조)      (실제 소유 M7/U3 — 인기 장소 집계 결과)

[클라이언트 세션 상태 — 서버 영속 아님]
TabNavigationState              (탭별 스택·스크롤; 세션 메모리 한정, 앱 재시작 초기화 G6)
```

> **U2 소유 서버 영속 엔티티 = `AppConfig` 1종.** (FD-U2-08) 사유: U2의 화면(스플래시·홈·5탭)은 상태를 **생성**하지 않고 조립·표시한다. 스플래시 판정 입력(BootstrapInfo)은 U1 계정·동의 데이터의 집계이고, 홈 카드 데이터는 후행 유닛(여행·인기 장소·활성 일정·기록)이 각각 소유한다. 유일한 서버 영속 데이터는 클라이언트 배포와 독립적으로 조정해야 하는 앱 구성(강제 업데이트 최소 버전 등) 1종이다. unit-of-work U2 산출물("DB 마이그레이션: 앱 구성 1개 내외, 신규 엔티티 약 1")과 정합.

---

## 1. AppConfig — 앱 구성 (server/app, U2 유일 영속 엔티티)

강제 업데이트 게이트·스토어 이동의 서버 정본. 클라이언트 배포와 독립적으로 운영 조정 가능한 remote config성 설정. 오구성 시 전 사용자 차단 리스크가 있어 변경 이력을 남긴다(BR-U2-04) [N4/C6, unit-of-work U2 리스크].

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| configKey | 문자열(설정 키) | 필수 · 유니크 | 설정 식별자. 플랫폼(iOS/Android)별 분리 관리 가능하도록 키에 플랫폼 차원 포함 |
| minSupportedVersion | 버전(SemVer) | 필수 | 강제 업데이트 최소 지원 버전 — 클라이언트 `appVersion < minSupportedVersion`이면 강제 업데이트 게이트(N4). 부트스트랩 응답 계약의 필드 [N4/C6, US-E2-06] |
| storeUrl | 문자열(URL) | 선택 | 강제 업데이트 화면의 스토어 이동 링크. NULL=플랫폼 기본 스토어 스킴 사용(P9 스토어 계정 확정 전 플레이스홀더 허용 — 개발 차단 아님) [unit-of-work U2 §선결] |
| updatedAt | 시각(UTC) | 필수 | 최종 변경 시각(변경 이력의 최신 포인터) |
| updatedBy | 문자열 | 필수 | 변경 주체(운영 감사 — 오구성 추적) |

**불변식**
- **INV-AC1 (버전 형식)**: minSupportedVersion은 유효한 SemVer이며 전순서 비교가 가능하다(BR-U2-04 버전 비교의 전제 — PBT 속성 U2-P3).
- **INV-AC2 (변경 이력 보존)**: minSupportedVersion 변경은 덮어쓰기가 아니라 이력을 남긴다 — 전면 차단 사고의 롤백·원인 추적 근거 [unit-of-work U2 리스크 "강제 업데이트 게이트 오구성"].

---

## 2. BootstrapInfo — 부트스트랩 집계 (읽기 전용 DTO, U1 M1 공급)

스플래시 분기 판정의 **단일 입력**. U1 `M1.getBootstrapStatus`가 세션·동의·버전을 집계해 반환하는 읽기 전용 응답이다. **U2는 이 계약의 소비자**이며, 판정 규칙(BR-U2-01·02)의 정본을 소유한다. 공급자측 판정 우선순위 계약은 U1 BR-U1-44/FD-U1-10.

### 2.1 속성 (component-methods M1.getBootstrapStatus 출력)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| sessionState | 열거 {VALID, EXPIRED, NONE} | 필수 | 세션 토큰 유효성. VALID=유효 세션, EXPIRED=만료(재로그인 필요), NONE=세션 없음(미로그인) [G5, US-E2-01] |
| onboardingComplete | 불리언 | 필수 | 온보딩 완료(=약관 동의 + 닉네임 통과, 취향 무관 — U1 BR-U1-26). false면 잔여 온보딩 단계로 분기(정확 단계는 M2.getOnboardingState 2차 해소, FD-U2-02) [G24/G157] |
| reconsentRequired | 불리언 | 필수 | 중대 개정 재동의 필요(blocking) 여부. true면 재동의 게이트(N3). 경미 변경(nonBlocking)은 이 플래그에 미포함(인앱 공지 채널) [N3, US-E1-18, U1 BR-U1-23] |
| minSupportedVersion | 버전(SemVer) | 필수 | 서버 최소 지원 버전(AppConfig 미러) [N4/C6] |
| forceUpdate | 불리언 | 필수 | `= (clientAppVersion < minSupportedVersion)` 서버 판정 결과. true면 나머지와 무관하게 전면 차단(최우선) [N4, US-E2-06, U1 FD-U1-10] |

**불변식**
- **INV-BI1 (판정 총함수 입력)**: (sessionState × onboardingComplete × reconsentRequired × forceUpdate) 조합은 스플래시 결정 함수의 완전한 입력 도메인이며, 어떤 조합에도 목적지가 정확히 1개 정의된다(§business-rules BR-U2-01·02, PBT 속성 U2-P1) [US-E2-01].
- **INV-BI2 (우선순위 종속)**: forceUpdate=true이면 다른 필드 값과 무관하게 판정은 FORCE_UPDATE — 필드 간 우선순위는 forceUpdate > reconsentRequired > sessionState/onboardingComplete로 고정(INV는 판정 함수가 강제, U1 FD-U1-10 소비자측) [N3, N4, G5].
- **INV-BI3 (멱등 재검증)**: BootstrapInfo 조회는 부작용 없는 멱등 조회 — 타임아웃 후 온라인 복구 시 동일 API 재호출로 상태 수렴(백그라운드 재검증) [G5, components.md §4.M1 폴백].

---

## 3. TabNavigationState — 탭 내비게이션 상태 (클라이언트 세션 상태)

5탭 셸의 탭별 화면 스택·스크롤 위치. **세션 메모리 한정** — 앱 재시작 시 초기화된다(G6). 서버 영속 아님(FD-U2-04·08). 소유 계층은 클라이언트 `shared/ui`(내비게이션)·`shared/storage`(세션 메모리) [components.md §6.2].

### 3.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| activeTab | 열거 {HOME, EXPLORE, ITINERARY, RECORD, MY} | 필수 · 기본 HOME | 현재 활성 탭. 5탭 고정 [US-E2-03] |
| tabStacks | 맵<탭, 화면 스택> | 필수 | 탭별 독립 내비게이션 스택(하위 화면 이력). 탭 전환 시 이전 탭 스택 보존 [G6] |
| scrollPositions | 맵<탭 또는 화면키, 스크롤 오프셋> | 필수 | 탭·화면별 스크롤 위치 보존값 [G6] |
| sessionEpoch | 식별자 | 필수 · 불변(세션 내) | 세션 생명주기 표식 — 앱 재시작으로 세션 교체 시 전체 상태 초기화의 경계(INV-TN2) |

**불변식**
- **INV-TN1 (탭 격리)**: 한 탭의 스택·스크롤 변경은 타 탭의 상태에 영향을 주지 않는다(탭별 독립 보존) [G6, PBT 속성 U2-P2].
- **INV-TN2 (세션 수명)**: 앱 재시작(sessionEpoch 교체) 시 tabStacks·scrollPositions는 전부 초기화된다 — 세션 내에서만 보존, 영속 캐시 아님 [G6, D24: 오프라인 일정 캐시 미제공과 정합].
- **INV-TN3 (재탭 리셋 국소성)**: 현재 활성 탭 재탭 시 해당 탭만 스크롤 탑으로 이동하며 스택은 루트로 복귀 — 타 탭 무영향 [G6, BR-U2-05, PBT 속성 U2-P2].
- **INV-TN4 (딥링크 푸시 정합)**: 딥링크·알림 진입은 대상 탭을 activeTab으로 설정하고 그 탭 스택에 화면을 push한다 — 타 탭 스택은 보존 [G7, BR-U2-06].

---

## 4. HomeDashboardModel — 홈 대시보드 조립 모델 (읽기 전용 집계 BFF)

홈 대시보드의 **카드 슬롯 스키마**. 홈 집약(BFF성) 조회가 가용 슬롯만 담아 반환하는 부분 응답 계약이다(침묵 실패 금지 — 일부 실패해도 가용 카드만). 각 슬롯의 **데이터 출처는 후행 유닛**이며, U2는 슬롯 스키마·부분 응답·빈 상태·미노출 규칙을 고정한다(FD-U2-05) [US-E2-02].

### 4.1 카드 슬롯 스키마

| 슬롯 | 데이터 출처(공급 유닛/메서드) | 페이로드 개요 | 빈/미노출 규칙 | 근거 |
|---|---|---|---|---|
| activeTripCard | U6 `M18.getActiveHub` + `M18.getTripProgress` | 진행 중 여행(D-day·진행률·현재/다음 슬롯 요약)→execution 허브 진입 | 진행 중 여행 없음이면 슬롯 미노출(활성 여행 최대 1개, D21) | US-E2-02, D21, G3 |
| upcomingTripCard | U4 `M6.listTrips` | 예정 여행(목적지·기간·D-day·등록 숙소 수·일정 유무) | 예정 여행 0건이면 emptyState로 대체(첫 행동 유도) | US-E2-02, G3 |
| quickAction | U2(자체) | '여행 만들기' 단일 빠른 액션 + (첫 사용자) 'AI로 먼저 일정 받아보기'·'장소 먼저 저장'(US-E2-05) | 항상 노출(홈의 상시 온램프) | US-E2-02, US-E2-05 |
| trendingPlaces | U3 `M7.getTrendingPlaces` | '지금 인기 있는 장소'(최근 7일 저장+방문 가중합 일 1회 배치, §5) | 집계 부재·조회 실패 시 슬롯 생략(부분 응답), 다른 카드 정상 | US-E2-02, G2/G174 |
| memoryCard | U7 `M12.getVisitStats` | '여행 추억 다시 보기'(과거 기록 진입) | 기록 0건이면 미노출 | US-E2-02 |
| preferencePromptCard | U1 `M2.getPendingPreferencePrompts` | 건너뛴 취향 점진 설정 카드(한두 개씩) | 미설정 취향 없으면 미노출 | US-E2-02, G157, U1 BR-U1-28 |
| communityRecordCard | (U10 — 후속) | '지금 뜨는·내 취향 여행 기록'(커뮤니티) | **U10 출시 전까지 슬롯 자체 미노출**(FD-U2-07) | US-E2-02, unit-of-work U2 §제외 |
| topBar | U1 세션 + U8 `M14.getInbox` | 워드마크 + 알림 진입점(읽지 않은 알림 배지 카운트) | 미읽음 0이면 배지 없이 진입점만 | US-E2-02 |

### 4.2 불변식

- **INV-HD1 (부분 응답 완전성)**: HomeDashboardModel은 항상 렌더 가능한 상태를 반환한다 — 일부 슬롯 조회가 실패해도 가용 슬롯만 담아 반환하고, 실패 슬롯은 로딩/재시도 또는 생략으로 표면화한다(침묵 실패 금지) [US-E2-02, RESILIENCY-10, PBT 속성 U2-P4].
- **INV-HD2 (조립 멱등성)**: 동일 슬롯 데이터 집합에 대해 조립 결과(카드 목록·정렬·노출/미노출 판정)는 동일하다 — 조립은 순수 변환 [PBT 속성 U2-P4].
- **INV-HD3 (활성 여행 단수)**: activeTripCard는 최대 1개 여행만 참조한다(D21로 진행 중 여행 구조적 단수) — 복수 활성 전환 슬롯 없음 [D21/Δ3, FD-U2-06].
- **INV-HD4 (미노출 게이트)**: communityRecordCard 슬롯은 U10 출시 게이트 이전에는 스키마에 예약만 되고 페이로드가 존재하지 않는다(미노출) [FD-U2-07, US-E2-02].
- **INV-HD5 (진행률 표시 종속)**: activeTripCard의 진행률은 여행 중일 때만 방문 체크 비율로 표기하고, 여행 전(예정)에는 진행률 없이 D-day만 표기한다 — 계산 정본은 M18(U6), U2는 표시 규칙 [G3, FD-U2-09].

---

## 5. TrendingPlace — 인기 장소 (참조 — 실제 소유 M7/U3)

홈 `trendingPlaces` 슬롯이 참조하는 인기 장소 집계 결과. **U2는 소유하지 않는다** — 실제 데이터 소유·집계 배치는 M7 Place Data(U3)이며(`M7.getTrendingPlaces`), U2는 표시용 최소 필드 계약만 소비한다. 여기 정의는 홈 슬롯이 기대하는 읽기 전용 표시 계약이며 정본은 U3 Functional Design [G2/G174, components.md §4.M7].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| poiRef | 식별자(canonical POI ID 참조) | 필수 | 인기 장소의 정본 POI 참조(canonical ID — 정본 소유 M7) [G133, components.md §4.M7] |
| displayName | 문자열 | 필수 | 표시 이름(스냅샷) |
| category | 문자열 | 선택 | 카테고리 라벨. NULL=미확인('미확인' 표기 규칙 준용) |
| region | 문자열 | 선택 | 지역 라벨 |
| rank | 정수 | 필수 | 집계 순위(최근 7일 저장+방문 가중합, 일 1회 배치 결과 — 집계 로직 정본 U3) [G2] |

**불변식**
- **INV-TP1 (참조 무결성 위임)**: TrendingPlace의 실제 정본·집계 무결성은 U3(M7)가 소유한다 — U2는 표시 계약만 고정하고, 소실 POI는 '확인 불가' 배지 규칙(G8)에 위임한다 [G2, G8, unit-of-work U3].

---

## 6. 데이터-스토리 추적 요약

| 데이터 | 소유/참조 | 주 근거 스토리 | 주 결정 |
|---|---|---|---|
| AppConfig | U2 서버 영속(유일) | US-E2-06 | N4/C6, unit-of-work U2 |
| BootstrapInfo | U1 공급·U2 소비(DTO) | US-E2-01, US-E2-06 | G5, N3, N4, D22, U1 FD-U1-10 |
| TabNavigationState | 클라이언트 세션 상태 | US-E2-03 | G6, G7, D24/Δ6 |
| HomeDashboardModel | 읽기 전용 집계 BFF | US-E2-02 | D21, G2, G3, G157 |
| TrendingPlace | 참조(소유 M7/U3) | US-E2-02 | G2/G174, G8 |
