# U4 여행 생성·필수 방문지 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/trip`(여행 생성·상세·거점·필수 방문지·기간 조정)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `trip` 행의 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M6(+M4·M7 소비) 메서드를 참조하고, 폼 검증 규칙은 [business-rules.md](./business-rules.md) BR-U4-xx의 클라이언트측 대응(UX용 사본 — **판정 정본은 항상 서버**, D28 원칙 준용)이다.
> 기술 중립: 상태 관리 라이브러리·내비게이션 구현체·스타일 시스템 선정은 NFR/Code Generation 소유.
> **지도 핀·핀 지정·현위치는 U3 `shared/map`(카카오 지도 SDK 브리지)을 재사용**한다 — U4는 신규 지도 컴포넌트를 만들지 않고 좌표 확인·핀 지정 진입점만 연결한다.
> data-testid 규약: `trip-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 여행 화면 플로우 (정본)

```text
[홈/온램프 '여행 만들기' — U2 소유 진입점 (숙소 우선 / '숙소 없이 일정부터' 1급 온램프)]
        │
        ▼
┌─ TripCreateScreen (여행지·날짜·인원·예산·시간창·점진 취향 카드) ──────────┐
│   여행지·날짜 필수(공백 시 생성 비활성) · 겹침/규칙 오류 · 예산 총액 입력   │
└──────────────────────────────────────────────────────────────────┘
   │ 생성 완료                                              │ 등록 숙소 있음
   ▼                                                        ▼
TripListScreen ──탭──▶ TripDetailScreen ───────────┬─ BaseAssignmentScreen (거점 지정·구간 편집·겹침 UI)
(예정/진행/종료 구분)   (기간·거점·필수방문지·상태)   │     │ 숙소 등록 진입점(→ U3 StayRegisterScreen)
                           │                          │     ▼
                           │ '필수 방문지'             │  TripDateChangeDialog (숙소 날짜 자동 반영·기간 변경 영향 확인)
                           ▼                          │
                   MustVisitScreen (검색 추가 / 저장 POI 체크박스·권역 경고 / 시각 고정)
                           │ 준비 완료
                           ▼
                   [AI 일정 생성하기] (U5 진입 — CP2 TripContext 공급)
```

**전역 규칙**
- 여행 화면군은 5탭 셸(U2) 내 '일정' 탭 스택 또는 홈 진입점에서 열린다(탭 규칙 G6·G7은 U2 shared/ui 소유).
- 숙소 등록 폼은 U3 `StayRegisterScreen`(US-E3-08 정본)을 재사용 — 여행 화면은 **등록 진입점만** 제공하고 등록 폼을 중복 정의하지 않는다(BR-U4-08).
- 외부 OTA 이동·제휴 고지는 U3 US-E3-05 정본 — 여행 화면에서 중복 정의하지 않는다(US-E4-10).
- 모든 서버 오류는 표준 오류 타입으로 정규화되어 침묵 실패 없이 표면화(shared/api) [ADR-0011].
- 파괴적 변경(기간 축소·거점 삭제·확정 후 필수 방문지 변경)은 항상 영향 미리보기·확인을 거친다(G39·G43).

## 2. 컴포넌트 계층

```text
features/trip
├─ TripNavigator                          — 여행 스택 컨테이너('일정' 탭·홈 진입점 하위)
│  ├─ TripCreateScreen
│  │  ├─ DestinationField                 — 여행지 입력(국내 한정·중심 좌표, 공백 시 생성 비활성)
│  │  ├─ TripDateRangePicker              — 시작·종료일(오늘 이후·≤30일·겹침 오류)
│  │  ├─ ImportDatesFromStayButton        — '등록 숙소에서 날짜 가져오기'(등록 숙소 있을 때) / 없으면 대안 노출
│  │  ├─ PartyField                       — 인원(선택·기본 1)
│  │  ├─ BudgetTotalField                 — 예산 전체 총액(항공 제외·러프 예산 기본값·1인/1일 파생 표기)
│  │  ├─ TripWindowEditor                 — 날짜별 이용 가능 시각(기본 09:00~21:00·첫날·마지막날)
│  │  ├─ TripTitleField                   — 제목(선택·자동 생성 미리보기·금칙어)
│  │  ├─ ProgressivePreferenceCard        — 온보딩 미설정 취향 점진 권유(건너뛰기 가능)
│  │  ├─ DateOverlapError                 — 겹치는 기존 여행 표시+편집·삭제 바로가기
│  │  └─ NoStayOnrampBanner               — "숙소 미등록" 상태·'숙소 없이 일정부터' 1급 온램프 안내
│  ├─ TripListScreen
│  │  └─ TripCard                         — 목적지·기간·D-day·등록 숙소 수·일정 유무·상태(예정/진행/종료)
│  ├─ TripDetailScreen
│  │  ├─ TripSummarySection               — 기간·인원·예산 파생·상태 배지
│  │  ├─ BaseTimelineSummary              — 거점 타임라인(스마트 기본 거점 비차단 배지)
│  │  ├─ MustVisitSummary                 — 필수 방문지 목록(유형 배지·'확인 불가' 배지)
│  │  └─ GenerateItineraryOnramp          — [AI 일정 생성하기](U5 진입) / 숙소 0개 시 비활성+등록 유도
│  ├─ BaseAssignmentScreen
│  │  ├─ RegisteredStayPicker             — 계정 풀 등록 숙소 선택(coordConfirmed 전제)
│  │  ├─ StayRegisterEntryButton          — 숙소 등록 진입점(→ U3 StayRegisterScreen)
│  │  ├─ BaseSegmentEditor                — 일자별 구간 거점 편집(다박 'N박 체류' 묶음)
│  │  ├─ BaseOverlapConflict              — 겹침 날짜 표시+그날 기준 거점 선택
│  │  ├─ SmartDefaultBaseBadge            — "이 날은 ○○ 기준 — 바꾸기"(비차단·첫날 공백=여행지 중심)
│  │  └─ CoordConfirmGuard                — 좌표 미확정 숙소 차단→"지도에서 위치 확인"(U3 shared/map)
│  ├─ MustVisitScreen
│  │  ├─ PoiSearchAddField                — 장소 검색 추가(M7.searchPoi)
│  │  ├─ SavedPlaceCheckboxList             — 저장 POI 체크박스 투입(사본 복제)
│  │  │  ├─ RegionOutsideWarnBadge        — 권역 밖 경고+기본 해제(G158)
│  │  │  └─ LostPoiExcludeBadge           — 소실 '확인 불가' 투입 제외(G8)
│  │  ├─ TimingToggle                     — '아무 때나 꼭 가기'(ANYTIME) / '시간 정해두기'(FIXED)
│  │  ├─ FixedTimingEditor                — 날짜·시각·체류(FIXED, 미입력 시 자동 배치)
│  │  ├─ MustVisitLimitGuard              — 한도(3×일수) 초과 차단+사유
│  │  └─ MustVisitChangePreview           — 확정 후 변경 전/후 비교(승인 후 적용 G43)
│  └─ TripDateChangeDialog
│     ├─ DateConflictSideBySide           — 여행 기간 vs 숙소 날짜 충돌 선택(자동 덮어쓰기 없음)
│     ├─ PeriodExtendConfirm              — 여행 기간 확장 여부 확인(자동 변경 없음)
│     └─ DestructiveChangeImpactList      — 기간 축소·거점 삭제 영향 블록 나열+차단형 확인(G39)
└─ (지도) shared/map (U3 재사용 — 신규 없음)
   ├─ MapView / PinDropController         — 거점 좌표 확인·필수 방문지 핀 지정(U3 브리지)
```

---

## 3. 여행 생성 화면

### 3.1 TripCreateScreen
- **props**: `prefill?: TripPrefill`(온램프·숙소 날짜 가져오기 결과) · **state**: 여행지·날짜·인원·예산·시간창·제목 입력값, 겹침/규칙 오류, 점진 취향 카드 상태.
- **폼 검증(서버 규칙 대응)**:
  - 여행지·날짜 공백 ⇔ "여행 생성" 버튼 비활성(BR-U4-01). 날짜는 오늘 이후·≤30일·종료≥시작(BR-U4-02). 국내 좌표 범위(BR-U4-04).
  - 겹침 시 `DateOverlapError`가 겹치는 기존 여행 명시+편집·삭제 바로가기(BR-U4-03 — 정본은 서버 `DateOverlap`).
  - 예산은 전체 총액 입력, 1인·1일은 파생 표기(BR-U4-06). 제목 미입력 시 자동 생성 미리보기+금칙어(BR-U4-05).
  - 등록 숙소 있으면 `ImportDatesFromStayButton`, 없으면 '숙소 등록하기 / 날짜 직접 입력' 대안(US-E4-01).
- **상호작용**: `M6.createTrip` [FLOW-1]. 숙소 미등록 생성 → `NoStayOnrampBanner`(Case B 온램프 US-E4-02). 미설정 취향 → `ProgressivePreferenceCard`(건너뛰기 가능).
- **data-testid**: `trip-create-destination`, `trip-create-daterange`, `trip-create-budget`, `trip-create-title`, `trip-create-overlaperror`, `trip-create-submit`, `trip-create-nostayonramp`.
- **사용 서버 능력**: `M6.createTrip`, `M6.updateTripWindow`, `M2.getPendingPreferencePrompts`.

---

## 4. 여행 목록·상세 화면

### 4.1 TripListScreen
- **props**: `filter: TripStatusFilter` · **state**: 예정/진행/종료 구분 목록.
- **폼 검증(서버 규칙 대응)**: '진행 중'은 항상 최대 1건(D21). 카드 진행률은 여행 전=D-day, 여행 중=방문 체크 비율(M18 소유, G3).
- **상호작용**: `M6.listTrips`. 카드 탭 → TripDetailScreen.
- **data-testid**: `trip-list-card`, `trip-list-status`.
- **사용 서버 능력**: `M6.listTrips`.

### 4.2 TripDetailScreen
- **props**: `tripId: TripId` · **state**: 기간·거점·필수 방문지·일정 유무·상태 로드.
- **폼 검증(서버 규칙 대응)**: 상태 배지(계획중/확정/진행중/종료). 숙소 0개면 `GenerateItineraryOnramp` 비활성+"숙소를 먼저 등록하세요"(US-E4-02 정본은 U5 스토리 1). 예산 미설정·선택 미입력은 명시값 표기.
- **상호작용**: `M6.getTrip`. 거점 → BaseAssignmentScreen, 필수 방문지 → MustVisitScreen, [AI 일정 생성하기] → U5(CP2 TripContext 공급).
- **data-testid**: `trip-detail-status`, `trip-detail-basetimeline`, `trip-detail-mustvisit`, `trip-detail-generateonramp`.
- **사용 서버 능력**: `M6.getTrip`, `M6.deleteTrip`.

---

## 5. 거점 연결 화면 (CP1 소비)

### 5.1 BaseAssignmentScreen
- **props**: `tripId` · **state**: 등록 숙소 선택, 구간 거점 편집값, 겹침 충돌, 스마트 기본 거점.
- **폼 검증(서버 규칙 대응)**:
  - `RegisteredStayPicker`는 coordConfirmed=true 숙소만 거점 자격 — `CoordConfirmGuard`가 미확정 시 "지도에서 위치 확인"(U3 shared/map, BR-U4-08, INV-BASE-U4-2).
  - `BaseOverlapConflict`: 구간 겹침 시 겹치는 날짜 표시+그날 기준 거점 선택(BR-U4-11 — 정본은 서버 `ConflictDetected`).
  - `SmartDefaultBaseBadge`: 공백일 비차단 안내(첫날 공백=여행지 중심 좌표 G41, BR-U4-12). 다박은 "N박 체류" 묶음(US-E4-07).
  - `StayRegisterEntryButton`은 U3 등록 폼 진입점만(중복 정의 금지).
- **상호작용**: `M4.getBaseTimeline`·`M4.linkToTrip`. 숙소 날짜 자동 반영 → `M4.proposeTripDatesFromStay` → TripDateChangeDialog. 거점 삭제 → `DestructiveChangeImpactList`(차단형 확인 G39).
- **data-testid**: `trip-base-staypicker`, `trip-base-registerentry`, `trip-base-segmenteditor`, `trip-base-overlapconflict`, `trip-base-smartdefaultbadge`, `trip-base-coordguard`.
- **사용 서버 능력**: `M4.getBaseTimeline`, `M4.linkToTrip`, `M4.proposeTripDatesFromStay`, `M6.updateTripDates`.

### 5.2 TripDateChangeDialog
- **props**: `tripId`, `proposedDates` · **state**: 충돌 값 비교, 확장 확인, 영향 블록.
- **폼 검증(서버 규칙 대응)**: `DateConflictSideBySide`는 여행 기간 vs 숙소 날짜 충돌 시 두 값 나란히·사용자 선택(자동 덮어쓰기 없음, BR-U4-09). `PeriodExtendConfirm`은 기간 이탈 시 확장 여부 확인(자동 변경 없음, BR-U4-10). `DestructiveChangeImpactList`는 기간 축소 영향 블록 나열+차단형 확인(BR-U4-20, G39). 자동 반영 후 D21·G42 재검증.
- **상호작용**: `M6.updateTripDates(dates, confirmation)`.
- **data-testid**: `trip-datechange-sidebyside`, `trip-datechange-extendconfirm`, `trip-datechange-impactlist`.
- **사용 서버 능력**: `M6.updateTripDates`.

---

## 6. 필수 방문지 화면

### 6.1 MustVisitScreen
- **props**: `tripId` · **state**: 검색 결과, 저장 POI 체크 상태, 유형 토글, FIXED 편집값, 한도 카운트, 변경 미리보기.
- **폼 검증(서버 규칙 대응)**:
  - `PoiSearchAddField`(M7.searchPoi) 또는 `SavedPlaceCheckboxList`로 추가(BR-U4-13). 저장 POI 투입은 사본 복제(BR-U4-14, G129).
  - `RegionOutsideWarnBadge`: 권역 밖 경고+기본 해제(BR-U4-15, G158). `LostPoiExcludeBadge`: 소실 '확인 불가' 투입 제외(BR-U4-17, G8).
  - `TimingToggle`: 기본 '아무 때나 꼭 가기'(ANYTIME), 토글 '시간 정해두기'(FIXED). `FixedTimingEditor` 필드 미입력은 자동 배치(INV-MV3).
  - `MustVisitLimitGuard`: 개수 > 3×일수 시 추가 차단+사유(BR-U4-16 — 정본은 서버 `LimitExceeded`). 좌표 미확정 → 핀 지정(U3 shared/map).
  - `MustVisitChangePreview`: 확정 일정 변경 시 전/후 비교·승인 후 적용(BR-U4-18, G43).
- **상호작용**: `M6.addMustVisit`·`M6.updateMustVisit`·`M6.removeMustVisit`·`M6.previewMustVisitChange`. 검색·저장 목록은 `M7.searchPoi`·`M7.listSavedPlaces`.
- **data-testid**: `trip-mustvisit-searchadd`, `trip-mustvisit-savedcheckbox`, `trip-mustvisit-regionwarn`, `trip-mustvisit-lostbadge`, `trip-mustvisit-timingtoggle`, `trip-mustvisit-limitguard`, `trip-mustvisit-changepreview`.
- **사용 서버 능력**: `M6.addMustVisit`, `M6.updateMustVisit`, `M6.removeMustVisit`, `M6.previewMustVisitChange`, `M7.searchPoi`, `M7.listSavedPlaces`.

---

## 7. shared/ U4 재사용·완성분 (요약)

| 계층 | U4 범위 | 근거 |
|---|---|---|
| shared/map | **U3 재사용**(신규 없음) — 거점 좌표 확인·필수 방문지 핀 지정 진입점만 연결 | U3 §7, BR-U4-08·17 |
| shared/api | 겹침·규칙·한도 오류 타입 소비·차단형 확인 토큰 흐름·CP2 컨텍스트 조립 | ADR-0011, CP2 |
| shared/ui | 여행 카드·상태 배지·비차단 스마트 기본 배지·명시값('미설정'·자동 제목 미리보기) | PRD 05-1·6 |
| features/onboarding(재사용) | 점진 설정 카드 데이터 소스(M2 미설정 취향) | US-E4-01, G24/G157 |

## 8. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M6 + M4·M7·M2 소비) |
|---|---|---|
| TripCreateScreen | US-E4-01, US-E4-02, US-E4-11 | M6.createTrip · M6.updateTripWindow · M2.getPendingPreferencePrompts |
| TripListScreen | US-E09-07, US-E2-02 | M6.listTrips |
| TripDetailScreen | US-E4-01~11 횡단 | M6.getTrip · M6.deleteTrip |
| BaseAssignmentScreen | US-E4-03, US-E4-04, US-E4-06, US-E4-07 | M4.getBaseTimeline · M4.linkToTrip · M4.proposeTripDatesFromStay |
| TripDateChangeDialog | US-E4-05, US-E4-03 | M6.updateTripDates |
| MustVisitScreen | US-E4-08, US-E4-09, US-E2-05 | M6.addMustVisit · updateMustVisit · removeMustVisit · previewMustVisitChange · M7.searchPoi · listSavedPlaces |
| GenerateItineraryOnramp(진입점) | US-E4-02(정본 U5) | M8.generateItinerary(NO_BASE 가드 — U5 소유) |
| (지도 핀·핀 지정) | US-E4-03·08 | U3 shared/map(M7.searchPoi·getPoi 재사용) |
