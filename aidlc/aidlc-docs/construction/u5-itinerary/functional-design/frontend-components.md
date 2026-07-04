# U5 AI 일정 생성·확정 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/itinerary`(생성 방식·진행·2보기·편집·같이 고르기·확정·권역 추천) + `shared/validation`(경량 제약 검증기, D28)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `itinerary` 행의 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M8·C2 메서드를 참조하고, 폼·편집 검증 규칙은 [business-rules.md](./business-rules.md) BR-U5-xx의 클라이언트측 대응(UX용 사본 — **판정 정본은 항상 서버 C2**, D28 원칙 준용)이다.
> 기술 중립: 상태 관리 라이브러리·내비게이션 구현체·스타일 시스템·경량 검증기 코드 생성 방식은 NFR/Code Generation 소유. **지도 렌더링은 U3 `shared/map`(카카오 지도 SDK 브리지) 재사용** — U5는 새 지도 브리지를 만들지 않고 U3 브리지 인터페이스를 소비한다.
> data-testid 규약: `itinerary-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 일정 화면 플로우 (정본)

```text
[탐색 랜딩 '여행자 일정' 진입(U2) 또는 여행 상세 [AI 일정 생성하기] 온램프(U4)]
        │
        ▼
┌─ GenerationModeSelectScreen (3방식 카드 — 결과 예고·예상 소요·인터랙션 양) ──────┐
│   완전 AI 자동 / 같이 고르기 / 직접 만들기 · 방식별 한 줄 결과 예고(BR-U5-34)      │
│   가드: 등록 숙소 0개 → NO_BASE 안내(단 숙소 나중 등록 온램프 예외 BR-U5-02)      │
└──────────────────────────────────────────────────────────────────────────┘
   │ FULL_AUTO / MANUAL          │ TOGETHER                    │ 나중 등록
   ▼                            ▼                             ▼
GenerationProgressScreen    TogetherPickFlow            StayZoneRecommendScreen
(스트림·1일 우선·취소/백그라운드) (슬롯 루프·반경·회색)         (동선 무게중심·before/after 추정 거리)
   │ 첫 1일 5초 노출·전체 완료                                    │ 추천 숙소 등록 → U4 registerStay → 재정렬
   ▼
┌─ ItineraryViewScreen (시간표/지도 2보기 토글 — 이동 구간 거리·수단만 D25) ────────┐
│   TimelineView / MapView(U3 shared/map 재사용) · 슬롯 탭 → SlotDetailSheet         │
└──────────────────────────────────────────────────────────────────────────┘
   │ 편집                        │ 슬롯 LOCK/교체            │ 확정
   ▼                            ▼                          ▼
ItineraryEditScreen          SlotDetailSheet           ConfirmScreen
(경량 검증 배지·자동보정/그대로)  (추천/배치 이유·LOCK·교체)    (읽기 전용·확정 의미·D-day)
   │ 저장(위반 분기)                                         │ 확정 해제 → '편집 중' 재진입(D20)
   ▼                                                        ▼
(current 갱신 → 실행 기준선 CP3)                        (plan 동결 → U6 실행 허브 진입)
```

**전역 규칙**
- 일정 화면군은 5탭 셸(U2) 내 '일정' 탭 스택에 쌓인다(탭 규칙 G6·G7은 U2 shared/ui 소유). 여행 중은 실행 허브(U6)로 수렴.
- **사용자에게 보이는 모든 시각·순서·이동 거리는 서버 C2 검증값만** 렌더한다(LLM 임의 시각 미노출, INV-SLOT2) — 클라이언트는 값을 생성하지 않는다.
- **소요시간 필드는 어떤 화면에도 없다** — 이동 구간은 `DistanceRange`(거리·수단·추정)까지만(D25/Δ1, INV-TRAVEL1).
- 모든 서버 오류는 표준 오류 타입으로 정규화되어 침묵 실패 없이 표면화(shared/api), 폴백 고지는 화면에 명시(기본 모드·추정 표기) [ADR-0011].
- 지도 렌더는 U3 `shared/map` 브리지 재사용 — 지도 타일/경로 실패 시 시간표형 폴백("지도를 불러오지 못했어요")+일정 데이터 정상 제공(BR-U5-38).

## 2. 컴포넌트 계층

```text
features/itinerary
├─ ItineraryNavigator                     — 일정 스택 컨테이너('일정' 탭 하위)
│  ├─ GenerationModeSelectScreen
│  │  ├─ ModeCard × 3                      — 완전 AI/같이 고르기/직접 — 결과 예고·예상 소요·인터랙션 양
│  │  └─ NoBaseGuide                       — 숙소 0개 안내(나중 등록 온램프 CTA 예외)
│  ├─ GenerationProgressScreen
│  │  ├─ ProgressStepIndicator            — 단계 텍스트("장소 고르는 중→동선 맞추는 중→시간표 확정 중")·진행률·경과
│  │  ├─ FirstDayPreview                   — 첫 1일 5초 우선 노출 블록
│  │  ├─ GenerationControlBar             — '취소'(부분 초안 저장)·'백그라운드로 계속'
│  │  └─ FallbackNoticeBanner             — "일부 추천이 기본 모드로 생성"·"최소 일정"·최소 일정 재시도
│  ├─ ItineraryViewScreen
│  │  ├─ ViewToggleSegment                — 시간표/지도 세그먼트 전환(단일 데이터 소스)
│  │  ├─ TimelineView                     — 시간순 타임라인·시각/체류 블록·위반 배지
│  │  │  └─ SlotBlock                     — 슬롯(POI·시각·체류·LOCK 아이콘·압축 표시·위반)
│  │  ├─ MapView (shared/map 재사용)       — 순서 번호 핀+이동 동선(U3 KakaoMapBridge)
│  │  ├─ TransitSegmentLabel              — 이동 구간 "약 850m · 도보, 추정"(거리·수단만 — 소요시간 없음)
│  │  └─ DirectionsHandoffButton          — '길찾기' → 외부 지도앱 좌표 위임
│  ├─ SlotDetailSheet
│  │  ├─ RecommendReason                  — LLM 추천 이유(취향 근거·표시 전용)
│  │  ├─ PlacementReason                  — 솔버 배치 이유("영업시간 오전만→오전 배치")
│  │  ├─ StayDurationControl              — 체류 조정(고정 시 고정 제약)·압축 배치 고지
│  │  ├─ LockToggle                       — 슬롯 LOCK/해제(warm-start 보존)
│  │  └─ ReplaceSlotButton                — 슬롯 교체 후보(솔버 검증 통과분만)
│  ├─ ItineraryEditScreen
│  │  ├─ EditableSlotList                 — 추가·삭제·재정렬·시간 조정
│  │  ├─ LightValidationBadge             — 경량 검증 즉시 배지("이동시간 부족"/"영업시간 외")
│  │  ├─ InsertableSlotPicker             — POI 추가 시 삽입 가능 시간대만 후보
│  │  └─ SaveViolationDialog              — "○곳에서 시간이 안 맞아요" → 'AI 자동 보정 / 그대로 저장'
│  ├─ TogetherPickFlow
│  │  ├─ ConceptPicker                    — 슬롯별 컨셉/테마 선택
│  │  ├─ RadiusControl                    — 반경 조정(도보 15분/차량 15분 — 넓히면 이동 거리 표기)
│  │  ├─ SlotCandidateList                — 반경 내 후보(반경 밖 disabled·회색)
│  │  ├─ ZeroCandidateGuide               — 반경 0건 → 반경 확대/컨셉 변경 제안
│  │  └─ ModeSwitchButton                 — 도중 방식 전환(→완전 AI, 진행분 보존)
│  ├─ ConfirmScreen
│  │  ├─ ReadonlyItineraryView            — 확정 일정(시간표/지도 토글·읽기 전용)
│  │  ├─ ConfirmMeaningNote               — 확정 의미·이점 한 줄 설명
│  │  ├─ DDayContext                      — D-day·출발일 여행 시작 맥락
│  │  └─ UnlockForEditButton              — '일정 수정' → '편집 중' 전환(재확정 필요 D20)
│  └─ StayZoneRecommendScreen
│     ├─ CentroidMapView (shared/map 재사용) — 동선 무게중심·권역 지도
│     ├─ StayZoneCandidateList            — 평균 이동 거리 순·before/after '추정 이동 거리'
│     ├─ ApproximationNote                — 추정·전역 최적 비보장 표기
│     └─ RegisterRecommendedStayButton    — 추천 숙소 등록(U4 registerStay 연계)
└─ shared/validation                       — 경량 제약 검증기(D28 — C2 ConstraintSpec 소비)
   ├─ ConstraintSpecClient                — 서버 발행 규칙 명세(버전) 소비·캐시
   ├─ LightValidator                      — 영업시간·이동시간·고정 충돌 즉시 검사(UX용 사본)
   └─ SpecVersionGuard                    — 명세 버전 불일치 시 로컬 검사 보수적 비활성화(오판 방지)
```

---

## 3. 생성 화면군

### 3.1 GenerationModeSelectScreen
- **props**: `tripId` · **state**: 선택 방식, 등록 숙소 존재 여부.
- **폼 검증(서버 규칙 대응)**: 각 방식 카드에 결과 예고·예상 소요·인터랙션 양 표시(BR-U5-34). 등록 숙소 0개면 `NoBaseGuide`("숙소를 먼저 등록하면…") — 단 '숙소 나중 등록' 온램프 CTA는 예외 노출(BR-U5-02).
- **상호작용**: 방식 선택 → `M8.generateItinerary(tripId, mode)`(FULL_AUTO/MANUAL→진행 화면, TOGETHER→같이 고르기) · 나중 등록 → `StayZoneRecommendScreen`.
- **data-testid**: `itinerary-modeselect-card`, `itinerary-modeselect-nobase`, `itinerary-modeselect-onramp`.
- **사용 서버 능력**: `M8.generateItinerary`.

### 3.2 GenerationProgressScreen
- **props**: `sessionId` · **state**: 진행 스트림(단계·진행률·경과), 첫 1일 노출 상태, 취소/백그라운드 상태, 폴백 플래그.
- **폼 검증(서버 규칙 대응)**: 첫 1일 5초 우선 노출(BR-U5-21), 진행 스트림 표시(BR-U5-22). '취소' 시 부분 초안 저장+'이어서 생성'(BR-U5-24). 폴백 고지 배너("일부 추천이 기본 모드로 생성"·최소 일정 재시도).
- **상호작용**: `M8.streamGenerationProgress`(스트림) · '취소' → `M8.cancelGeneration(keepDraft=true)` · 완료 → `ItineraryViewScreen`.
- **data-testid**: `itinerary-progress-step`, `itinerary-progress-firstday`, `itinerary-progress-cancel`, `itinerary-progress-background`, `itinerary-progress-fallback`.
- **사용 서버 능력**: `M8.streamGenerationProgress`, `M8.cancelGeneration`, `M8.resumeGeneration`.

### 3.3 TogetherPickFlow (같이 고르기 — mode=TOGETHER)
- **props**: `sessionId` · **state**: 현재 슬롯 커서, 컨셉, 반경, 기준점(직전 확정 슬롯).
- **폼 검증(서버 규칙 대응)**: 기준점=직전 확정 슬롯(첫 슬롯=등록 숙소, BR-U5-35). 반경 밖 후보 disabled·회색. 반경 확대 시 예상 이동 거리 표기(소요시간 없음). 0건 → 반경 확대/컨셉 변경 제안(`ZeroCandidateGuide`). 도중 방식 전환 시 진행분 보존(BR-U5-37).
- **상호작용**: `M8.getSlotCandidates(slotCursor, concept, radius)` → `M8.chooseSlotCandidate` → 다음 슬롯 · `M8.switchGenerationMode`.
- **data-testid**: `itinerary-together-concept`, `itinerary-together-radius`, `itinerary-together-candidate`, `itinerary-together-disabled`, `itinerary-together-zeroguide`, `itinerary-together-modeswitch`.
- **사용 서버 능력**: `M8.getSlotCandidates`, `M8.chooseSlotCandidate`, `M8.switchGenerationMode`.

---

## 4. 조회·상세 화면군

### 4.1 ItineraryViewScreen (시간표/지도 2보기)
- **props**: `itineraryId` · **state**: 활성 보기(시간표/지도), 일정 데이터(단일 소스).
- **폼 검증(서버 규칙 대응)**: 한 보기 수정이 다른 보기에 즉시 반영(단일 데이터 소스, BR-U5-38). 이동 구간은 `TransitSegmentLabel`이 **거리·수단만**("약 850m · 도보, 추정") — 소요시간 필드 없음(BR-U5-39, INV-TRAVEL1). 지도 실패 시 시간표형 폴백. '길찾기'는 외부 지도앱 좌표 위임(`DirectionsHandoffButton`).
- **상호작용**: `M8.getItinerary`(2보기 공용 데이터) · `MapView`는 U3 `shared/map` 재사용 · `DirectionsHandoffButton` → `M18.buildExternalMapHandoff`(길찾기 위임, 거리만) · 슬롯 탭 → `SlotDetailSheet`.
- **data-testid**: `itinerary-view-toggle`, `itinerary-view-timeline`, `itinerary-view-map`, `itinerary-view-transit`, `itinerary-view-directions`.
- **사용 서버 능력**: `M8.getItinerary`, `M18.buildExternalMapHandoff`, (U3)`shared/map` 브리지.

### 4.2 SlotDetailSheet
- **props**: `itineraryId`, `slotId` · **state**: 추천/배치 이유 로드, LOCK 상태, 체류 값.
- **폼 검증(서버 규칙 대응)**: `RecommendReason`(LLM 취향 근거)·`PlacementReason`(솔버 제약 근거) 병기 — **표시 전용, 검증값 우선**(설명-시각 불일치 시 검증 시각 표시, BR-U5-40). LLM 설명 실패 시 "추천 이유를 불러오지 못했어요"(일정 영향 없음). 체류 압축 시 고지("권장 90분→최소 50분"). LOCK·교체는 솔버 검증 통과분만.
- **상호작용**: `M8.getExplanations` · `M8.lockSlot`/`M8.unlockSlotLock`(LOCK) · `M8.replaceSlot`(교체 후보).
- **data-testid**: `itinerary-slot-recommendreason`, `itinerary-slot-placementreason`, `itinerary-slot-lock`, `itinerary-slot-replace`, `itinerary-slot-staycompressed`.
- **사용 서버 능력**: `M8.getExplanations`, `M8.lockSlot`, `M8.unlockSlotLock`, `M8.replaceSlot`.

---

## 5. 편집 화면

### 5.1 ItineraryEditScreen
- **props**: `itineraryId` · **state**: 편집 조작 목록, 클라 경량 검증 결과(위반), 저장 위반 분기.
- **폼 검증(서버 규칙 대응)**:
  - 편집 시 `shared/validation` 경량 검증기가 영업시간·이동시간·고정 충돌 **즉시 재검증**(UX용 사본 — 정본은 서버 C2, BR-U5-25). 명세 버전 불일치 시 로컬 검사 보수적 비활성화(`SpecVersionGuard`).
  - 위반은 **비차단** — `LightValidationBadge`("이동시간 부족"/"영업시간 외")+사유(BR-U5-26). POI 추가 시 `InsertableSlotPicker`(삽입 가능 시간대만).
  - 저장 시 위반 있으면 `SaveViolationDialog` → 'AI 자동 보정'(시각·순서만 G49) / '그대로 저장'(위반 보존·지속 가시화, BR-U5-27·28).
  - 확정 상태 일정 편집 진입은 `unlockForEdit` 선행(D20, BR-U5-33).
- **상호작용**: `M8.validateEdit`(서버 확정 검증) · `M8.getInsertableSlots` · `M8.applyEdit(onViolation)` · `C2.repair`(자동 보정 경유는 M8) · 저장 네트워크 오류 → 로컬 임시("저장 대기 중").
- **data-testid**: `itinerary-edit-slotlist`, `itinerary-edit-validationbadge`, `itinerary-edit-insertpicker`, `itinerary-edit-violationdialog`, `itinerary-edit-autofix`, `itinerary-edit-saveasis`.
- **사용 서버 능력**: `M8.validateEdit`, `M8.getInsertableSlots`, `M8.applyEdit`.

---

## 6. 확정·권역 추천 화면군

### 6.1 ConfirmScreen (일정 확정·확정본 열람)
- **props**: `itineraryId` · **state**: 확정 검증 결과, 읽기 전용 보기, 확정/해제 상태.
- **폼 검증(서버 규칙 대응)**: 확정 전 서버 확정 검증(C2 `validate`) 통과 게이트 — 위반 잔존 시 'AI 자동 보정/그대로 저장' 선택 완료 후(BR-U5-31). 확정 후 읽기 전용(`ReadonlyItineraryView` — 시간표/지도 토글)+D-day/출발일(`DDayContext`)+확정 의미 한 줄(`ConfirmMeaningNote`, BR-U5-32). '일정 수정' → `unlockForEdit`('편집 중' 전환, 재확정 필요·재확정 전 신규 공개 불가 D20, BR-U5-33).
- **상호작용**: `M8.confirmItinerary`(plan 동결 D14) · `M8.unlockForEdit` · `M8.getItinerary`(확정본 열람).
- **data-testid**: `itinerary-confirm-readonly`, `itinerary-confirm-meaning`, `itinerary-confirm-dday`, `itinerary-confirm-unlock`.
- **사용 서버 능력**: `M8.confirmItinerary`, `M8.unlockForEdit`, `M8.getItinerary`.

### 6.2 StayZoneRecommendScreen (숙소 나중 등록 온램프)
- **props**: `tripId` · **state**: 무게중심·권역 후보, before/after 추정 거리.
- **폼 검증(서버 규칙 대응)**: 방문지 2개 미만 등 동선 부족 시 추천 미제공·일반 탐색 유도(INV-ZONE1). 후보는 평균 이동 거리 순, before/after **'추정 이동 거리'만**(소요시간 없음 D25). 추정·전역 최적 비보장 표기(`ApproximationNote`, BR-U5-44).
- **상호작용**: `M8.recommendStayZone` · `CentroidMapView`는 U3 `shared/map` 재사용 · 추천 숙소 등록 → `M4.registerStay`(U3/U4 연계) → 기준점 삼아 재정렬(FLOW-1 warm-start).
- **data-testid**: `itinerary-stayzone-map`, `itinerary-stayzone-candidate`, `itinerary-stayzone-approxnote`, `itinerary-stayzone-register`.
- **사용 서버 능력**: `M8.recommendStayZone`, (U3/U4)`M4.registerStay`, (U3)`shared/map` 브리지.

---

## 7. shared/validation — 경량 제약 검증기 (U5 신규, D28)

- **책임**: 편집 중 즉시 제약 검사(영업시간·이동시간·고정 블록 충돌)를 클라이언트에서 수행하는 **UX용 사본** — **판정 정본은 항상 서버 C2**(저장 시 확정 검증). 규칙 명세(ConstraintSpec)는 C2가 버전 있는 계약으로 발행하고 본 계층이 소비한다(단일 규칙 정본화). **검증 로직 코드 생성/파생 방식은 NFR/Code Generation 소유**, 본 문서는 계층 계약만 규정한다.
- **props**: `ConstraintSpecClient{ version, rules[HC1~HC4] }` · **state**: 명세 버전, 로컬 검사 활성 여부.
- **상호작용·폴백**:
  - `LightValidator`: 편집 조작마다 영업시간·이동 부등식·고정 충돌 즉시 검사 → `LightValidationBadge` 렌더(비차단, BR-U5-26).
  - `SpecVersionGuard`: 서버 명세 버전과 로컬 버전 불일치 시 **로컬 검사 보수적 비활성화**(통과→저장 거부 UX 방지, 서버 검증 위임 — D28 오판 방지).
  - 클라↔서버 규칙 동치는 동일 케이스 이중 실행 계약 테스트로 보장(PBT U5-P12).
- **data-testid**: `itinerary-validation-badge`, `itinerary-validation-specguard`.
- **사용 서버 능력**: `C2.exportClientValidationSpec`(규칙 명세 발행), `M8.validateEdit`(서버 확정 검증 위임).

## 8. shared/ 재사용·U5 완성분 (요약)

| 계층 | U5 범위 | 근거 |
|---|---|---|
| shared/map (U3 소유) | **재사용** — 일정 지도형 뷰·권역 추천 지도·순서 핀·이동 동선(카카오 브리지). U5는 신규 지도 브리지 미생성 | components §6.1, U3 shared/map |
| shared/validation | **신규** — 경량 제약 검증기(C2 명세 소비·즉시 검사·버전 가드) | D28, G163 |
| shared/api | 생성 진행 스트림 소비·라이브 폴백 고지·저장 네트워크 오류 로컬 임시("저장 대기 중") | ADR-0011, PRD 06-8 |
| shared/ui | 위반 배지·폴백 고지 배너·'추정' 표기·시간표/지도 세그먼트 컴포넌트 | D25, PRD 06-9 |

## 9. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M8·C2) |
|---|---|---|
| GenerationModeSelectScreen | US-E5-01, US-E5-10 | M8.generateItinerary |
| GenerationProgressScreen | US-E5-09 | M8.streamGenerationProgress · cancelGeneration · resumeGeneration |
| TogetherPickFlow | US-E5-10 | M8.getSlotCandidates · chooseSlotCandidate · switchGenerationMode |
| ItineraryViewScreen | US-E5-06, US-E5-08 | M8.getItinerary · M18.buildExternalMapHandoff · (U3)shared/map |
| SlotDetailSheet | US-E5-05, US-E5-10 | M8.getExplanations · lockSlot · unlockSlotLock · replaceSlot |
| ItineraryEditScreen | US-E5-07, US-E5-08 | M8.validateEdit · getInsertableSlots · applyEdit |
| ConfirmScreen | US-E5-12 | M8.confirmItinerary · unlockForEdit · getItinerary |
| StayZoneRecommendScreen | US-E5-11 | M8.recommendStayZone · (U3/U4)M4.registerStay · (U3)shared/map |
| shared/validation | US-E5-07 | C2.exportClientValidationSpec · M8.validateEdit |
