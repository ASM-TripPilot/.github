# U6 여행 중 실행·Plan-B — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/execution`(허브·도착·체류·현장 상세·다음 이동) + `features/planb`(트리거·재계획·휴식·경로 비교) + `shared/location`(위치 권한 3층 상태 G182·포그라운드 수집)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 실행·Plan-B 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M18·M9·M10·M11 메서드를 참조하고, 폼·검증 규칙은 [business-rules.md](./business-rules.md) BR-U6-xx의 클라이언트측 대응(판정 정본은 항상 서버 — 재계획 하드 제약은 서버 C2, 트리거 판정은 서버 M9 순수 함수)이다.
> 기술 중립: 상태 관리·내비게이션·스타일·지오펜스 SDK·포그라운드 수집 구현체는 NFR/Code Generation 소유. **지도 렌더링은 U3 `shared/map`(카카오 지도 SDK 브리지) 재사용 + U5 지도 뷰(순서 핀·동선) 재사용** — U6는 새 지도 브리지를 만들지 않는다(FD-U6-09).
> data-testid 규약: `execution-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 실행·Plan-B 화면 플로우 (정본)

```text
[여행 시작일 00:00 도달(M18 배치) → 홈 활성 카드·'일정' 탭이 실행 허브로 수렴]
        │
        ▼
┌─ ActiveTripHubScreen (단일 허브 — 진행률·현재/다음 슬롯·다음 예정지 거리·여유 시간) ────────────┐
│   오늘 일정(완료/진행 중/다음) · nextLegDistance(거리만 D25) · slack(단순 차이 G67) · 진행률(G3)  │
│   상단: TriggerBannerHost(Plan-B 제안 칩·배너 — 자동 트리거 SHOWN)                              │
└──────────────────────────────────────────────────────────────────────────────────────────┘
   │ 도착/방문          │ 장소 상세        │ 다음 이동           │ '대안 보기'/'재계획'   │ '여행 종료'
   ▼                   ▼                 ▼                    ▼                      ▼
ArrivalPromptSheet  PlaceContextSheet  NextLegBar          ReplanFlow            EndTripConfirm
(도착 확인·수동 도착) (영업·체류·여유·미확인) (거리·외부 지도 시트)  (사유·기준·후보·전후·확정)  (수동 종료 D19)
   │ 방문 체크/스킵                       │ 외부 지도앱 복귀 → 도착 프롬프트   │ 휴식 전환
   ▼                                     (handleExternalMapReturn G66)        ▼
VisitCheckControls ──▶ VisitChecked(CP4)                                    RestModeScreen
                                                                             (진입·재개 재계산 G54)
        │ 계획 vs 실제 경로 토글
        ▼
PlanVsActualMapScreen (계획 동선 흐리게 + GPS 발자취 점선 · 누적 거리·걸음 수 추정 — U3/U5 map 재사용)
```

**전역 규칙**
- 실행·Plan-B 화면군은 5탭 셸(U2) 내 '일정' 탭 스택 + 홈 활성 카드에서 진입하며 **단일 허브(ActiveTripHubScreen)로 수렴**한다(INV-HUB1). 여행 종료(ENDED) 시 허브 소멸(추억 카드 전환).
- **사용자에게 보이는 모든 시각·거리·여유 시간은 서버 검증값·거리만** 렌더한다 — **소요시간 필드는 어떤 화면에도 없다**(D25/Δ1, INV-HUB2/INV-ALT2, U5 INV-TRAVEL1 재사용). 이동 구간·다음 예정지는 `DistanceRange`(거리·수단·추정)까지만.
- **도착 확정·방문 완료는 항상 사용자 탭** — 지오펜스는 프롬프트만(자동 기록 없음 D23). 위치 실패 시 수동 체크인이 기본 수단(ADR-0010).
- **자동 트리거는 제안만** — 배너/칩은 '대안 보기'를 눌러야 재계획 진입(자동 일정 변경 없음, INV-TRIG3). 심각=푸시·경미=인앱 칩.
- **재계획 결과는 current에만** 반영·plan 불변(D14). 확정 전 어떤 후보도 일정 미반영(확정 재검증 G56 통과 후에만).
- 위치 의존 UI는 **`shared/location` 3층 상태(G182)에 게이트**되며 전 조합에 폴백 존재(BR-U6-25 매트릭스). 지도 실패 시 목록/거리 요약 폴백.

## 2. 컴포넌트 계층

```text
features/execution
├─ ExecutionNavigator                      — 실행 허브 스택('일정' 탭·홈 활성 카드 진입)
│  ├─ ActiveTripHubScreen
│  │  ├─ HubProgressHeader                 — 진행률(완료/진행 중/다음 G3)·D-day 맥락
│  │  ├─ TodayTimeline                     — 오늘 일정 슬롯(방문 상태 배지)
│  │  │  └─ VisitSlotRow                   — 슬롯(POI·시각·체류·방문 상태·현재/다음 하이라이트)
│  │  ├─ NextLegBar                        — 다음 예정지 거리(거리만 D25)·'외부 지도로 열기'·수동 새로고침(G65)
│  │  ├─ TriggerBannerHost                 — Plan-B 제안 배너/칩(SHOWN)·출처·감지 시각·'대안 보기'
│  │  └─ EndTripButton                     — '여행 종료' 수동 버튼(D19)
│  ├─ ArrivalPromptSheet                   — '도착하셨나요?'(지오펜스·자동 기록 없음 D23)
│  │  ├─ ConfirmArrivalButton              — '도착했어요'(사용자 탭 확정)
│  │  └─ DismissPromptButton               — 닫기(재프롬프트 억제)
│  ├─ VisitCheckControls                   — 방문 시작/완료/스킵/취소 액션+기록 진입점(사진·메모=U7)
│  │  ├─ CompleteVisitButton               — 방문 완료(실제 체류 측정)
│  │  ├─ SkipVisitButton / UndoVisitButton — 스킵·취소
│  │  └─ ManualArrivalButton               — 수동 도착(위치 무관 ADR-0010)
│  ├─ PlaceContextSheet                    — 현장 장소 상세(영업시간·예상 체류·주변 추천·여유 시간 G67)
│  │  ├─ OpeningHoursRow / StayEstimateRow — 영업시간·예상 체류(추정 표기·출처)
│  │  ├─ CrowdednessRow                    — 혼잡도 '미확인' 고정(G199)
│  │  ├─ SlackToNextRow                    — 다음 일정까지 여유(단순 차이·소요시간 없음 G67/D25)
│  │  └─ NearbyRecommendList               — 주변 추천(탭 → Plan-B 수동 재계획 G64)
│  ├─ ExternalMapHandoffSheet              — 설치 지도 앱 목록 시트(외부 위임 G66)
│  └─ EndTripConfirmDialog                 — 여행 종료 확인(수동 종료 → 회고 생성 트리거)
└─ (진입점) HomeActiveTripCard             — 홈 활성 일정 카드(U2 슬롯·허브로 수렴 — 데이터는 M18)

features/planb
├─ PlanbNavigator                          — Plan-B 스택(트리거 랜딩·재계획·휴식·경로 비교)
│  ├─ ReplanFlow
│  │  ├─ ReplanReasonPicker               — 사유 5종+'사유 없음'(수동) · 방식(AI에게 맡기기/직접 수정)
│  │  ├─ ReplanPositionInput              — 기준 위치(GPS 실패 시 지도 핀·검색·추정 출발지 표기 US-E7-10)
│  │  ├─ AlternativeCompareList           — 대안 후보 2~3개 비교 카드(추천 이유·거리·체류·여유 — 소요시간 없음)
│  │  ├─ AlternativeFallbackGuide         — 대안 0개 폴백 3종(1개 건너뛰기/휴식 전환/수동 수정)+사유 한 줄
│  │  ├─ BeforeAfterCompare               — 전/후 비교(추가·삭제·시간 이동 구분·지표 요약)
│  │  ├─ UnplacedConsentDialog            — 이월·제외 방문지 명시+확정 전 동의(C10)
│  │  ├─ ConfirmReplanButton              — 확정(확정 시점 재검증 G56 통과 후 current 반영)
│  │  ├─ KeepCurrentButton                — '기존 유지'(어떤 변경도 미저장)
│  │  └─ ManualEditFallback               — 수동 일정 수정(순서·삭제·시각 — 숙소 고정 위반 차단 US-E7-11)
│  ├─ TriggerBannerHost (features/execution와 공유 렌더 계약) — 배너/칩·심각/경미 구분
│  ├─ RestModeScreen
│  │  ├─ RestResumeInput                  — 재개 시각 입력 ∨ 즉시 재개
│  │  └─ ResumeRecalcPrompt               — 재개 시 남은 일정 재계산 제안(G159)
│  ├─ UnplacedListScreen                  — 미배치(이월) 목록·날짜 선택 재배치(C10)
│  ├─ SensitivitySettingControl           — 트리거 민감도 3단계(적게/보통/많이 G58 — 설정 연동 U8)
│  └─ PlanVsActualMapScreen (U3 shared/map·U5 map 재사용)
│     ├─ PlannedRouteLayer                — 계획 동선(흐리게)
│     ├─ ActualTrailLayer                 — 실제 GPS 발자취(점선·점·옵트인 게이트)
│     └─ TrailStatsBar                    — 누적 실제 거리·걸음 수(추정 표기 G59)
└─ shared/location                         — 위치 권한 3층 상태·포그라운드 수집(G182)
   ├─ LocationConsentState                — OS 권한 × 법정 동의 × GPS 옵트인 3층 상태(N2)
   ├─ GeofenceForegroundWatcher           — 포그라운드 지오펜스 진입 감지(백그라운드 미사용 G62)
   ├─ GpsTrailCollector                   — 포그라운드 저빈도(1~5분) 수집·단순화(옵트인 전제 D34)
   └─ LocationFallbackResolver            — 3층 조합별 동작 매트릭스 적용(BR-U6-25)
```

---

## 3. 실행 허브 화면군 (features/execution)

### 3.1 ActiveTripHubScreen (활성 일정 허브)
- **props**: `accountId` · **state**: 허브 데이터(오늘 일정·진행 상태·다음 예정지·여유·진행률), 트리거 배너 목록, 휴식 상태.
- **폼 검증(서버 규칙 대응)**: 진행 중 여행당 단일 허브로 수렴(홈 카드·일정 탭 단일 소스 INV-HUB1). 다음 예정지는 `NextLegBar`가 **거리만**("약 850m · 도보, 추정") — 소요시간 없음(BR-U6-30, INV-HUB2). 온라인 전제 — 실패 시 오류·재시도(오프라인 조회 미보장 D24). 상단 `TriggerBannerHost`는 자동 트리거 제안(제안만, '대안 보기' 필요 INV-TRIG3).
- **상호작용**: `M18.getActiveHub`(허브 데이터) · `M18.getTripProgress`(진행률) · `M9.getActiveTriggerBanners`(배너) · '여행 종료' → `EndTripConfirmDialog`.
- **data-testid**: `execution-hub-timeline`, `execution-hub-nextleg`, `execution-hub-progress`, `execution-hub-triggerbanner`, `execution-hub-endtrip`.
- **사용 서버 능력**: `M18.getActiveHub`, `M18.getTripProgress`, `M18.getNextLegDistance`, `M9.getActiveTriggerBanners`.

### 3.2 ArrivalPromptSheet + VisitCheckControls (도착·방문 체크)
- **props**: `tripId`, `slotId` · **state**: 방문 상태(§3.3), 프롬프트 노출/억제, 도착 방식(Prompt/Manual).
- **폼 검증(서버 규칙 대응)**: 지오펜스 진입 시 **'도착하셨나요?' 프롬프트만 — 자동 기록 없음**(BR-U6-01, INV-VISIT2). 도착 확정·완료는 **항상 사용자 탭**(D23). 위치 없음/저정확도면 프롬프트 미노출 — **수동 도착 경로 항상 제공**(BR-U6-02, ADR-0010). 닫기 시 재프롬프트 억제(BR-U6-04). 완료 시 실제 체류 측정(BR-U6-03). 스킵·취소 지원.
- **상호작용**: `M18.onGeofenceEnter`(프롬프트) · `M18.confirmArrival(via)` · `M18.completeVisit`(→ VisitChecked·M12.checkVisit 위임) · `M18.skipVisit`/`M18.undoVisitAction`. 사진·메모 진입점은 U7(M12.attachMedia).
- **data-testid**: `execution-arrival-prompt`, `execution-arrival-confirm`, `execution-arrival-dismiss`, `execution-visit-complete`, `execution-visit-skip`, `execution-visit-manual`.
- **사용 서버 능력**: `M18.onGeofenceEnter`, `M18.confirmArrival`, `M18.completeVisit`, `M18.skipVisit`, `M18.undoVisitAction`, (U7)`M12.checkVisit`.

### 3.3 PlaceContextSheet (현장 장소 상세)
- **props**: `tripId`, `slotId` · **state**: 장소 카드(영업시간·예상 체류·주변 추천·여유 시간).
- **폼 검증(서버 규칙 대응)**: 영업시간·예상 체류(추정 표기·출처)·주변 추천·여유 시간 표시. **여유 시간=다음 계획 시작 시각까지 단순 차이(이동시간 미반영)**(BR-U6-29, G67/D25). **혼잡도 '미확인' 고정**(G199), 누락 데이터 '미확인'. 주변 추천 탭 → Plan-B 수동 재계획(G64).
- **상호작용**: `M18.getPlaceContext` · 주변 추천 탭 → `M10.startReplan`(수동 재계획 연결).
- **data-testid**: `execution-place-hours`, `execution-place-stay`, `execution-place-crowd`, `execution-place-slack`, `execution-place-nearby`.
- **사용 서버 능력**: `M18.getPlaceContext`, `M10.startReplan`(주변 추천 편입 경유).

### 3.4 NextLegBar + ExternalMapHandoffSheet (다음 이동·외부 길찾기)
- **props**: `tripId`, `slotId` · **state**: 다음 예정지 거리, 외부 지도 앱 목록, 복귀 처리.
- **폼 검증(서버 규칙 대응)**: **거리만**(거리 기반 추정·추정 표기)·소요시간 미표시(BR-U6-30, D25). 화면 진입/포커스 1회 산출+수동 새로고침(G65). '외부 지도로 열기' → 설치 앱 시트(G66). **복귀 시 다음 예정지 근접이면 도착 확인 프롬프트**(BR-U6-31·04). 외부 앱 없음 → 웹 지도 → 거리 요약 폴백(소요시간 없음).
- **상호작용**: `M18.getNextLegDistance`(거리만) · `M18.buildExternalMapHandoff`(좌표 위임) · `M18.handleExternalMapReturn`(복귀→도착 프롬프트).
- **data-testid**: `execution-nextleg-distance`, `execution-nextleg-refresh`, `execution-nextleg-openmap`, `execution-nextleg-return`.
- **사용 서버 능력**: `M18.getNextLegDistance`, `M18.buildExternalMapHandoff`, `M18.handleExternalMapReturn`.

### 3.5 EndTripConfirmDialog (여행 종료)
- **props**: `tripId` · **state**: 종료 확인.
- **폼 검증(서버 규칙 대응)**: 수동 '여행 종료' 버튼 → 종료 전이(자동 익일 00:00과 병행 단일 규칙 D19). 종료 = 회고 자동 생성 트리거(자동/수동 경합 멱등 INV-EXEC1). 숙소 유무 무관.
- **상호작용**: `M18.endTripManually`(→ TripEnded·M13 회고).
- **data-testid**: `execution-endtrip-confirm`, `execution-endtrip-cancel`.
- **사용 서버 능력**: `M18.endTripManually`.

---

## 4. Plan-B 화면군 (features/planb)

### 4.1 ReplanFlow (재계획 — 사유·기준·후보 비교·전후·확정)
- **props**: `tripId`, `triggerRef?`(자동 진입 시) · **state**: 세션(사유·방식·기준 위치·후보·비교·확정), 폴백 상태.
- **폼 검증(서버 규칙 대응)**:
  - 진입: '여행 중' 게이트(숙소 0개 허용 BR-U6-15). 사유 5종+'사유 없음', 방식 **AI에게 맡기기/직접 수정**(직접 수정=수동 편집 직행 US-E7-12).
  - 기준 위치: GPS 실패 시 `ReplanPositionInput`(핀·검색·**추정 출발지 표기** BR-U6-26). 생략 시 최후 완료 방문지/숙소(가정 명시).
  - 후보: `AlternativeCompareList` 2~3개 — 추천 이유·거리·수단·체류·여유(소요시간 없음 BR-U6-17·20, INV-ALT2). 솔버 통과분만.
  - 0건 폴백: `AlternativeFallbackGuide` 3종(1개 건너뛰기/휴식 전환/수동 수정)+사유 한 줄(BR-U6-18).
  - 전후 비교: `BeforeAfterCompare`(추가/삭제/시간 이동 구분·지표 요약 — 총 이동 거리·방문지 수·숙소 복귀 시각 증감, 소요시간 없음 BR-U6-20). 이월 방문지 `UnplacedConsentDialog`(확정 전 동의 C10).
  - 확정: **확정 시점 재검증(G56)** 통과 후 current 반영(plan 불변 D14, BR-U6-21). 무효화 시 재산출 안내. '기존 유지' → 미저장.
  - 수동 편집 폴백: `ManualEditFallback`(순서·삭제·시각 — **숙소 고정 위반 차단** BR-U6-19, US-E7-11).
- **상호작용**: `M10.startReplan` · `M10.setManualPosition` · `M10.getAlternatives` · `M10.previewAlternative` · `M10.confirmAlternative`(G56 재검증) · `M10.keepCurrentPlan` · `M10.applyManualEdit`.
- **data-testid**: `execution-replan-reason`, `execution-replan-method`, `execution-replan-position`, `execution-replan-candidate`, `execution-replan-fallback`, `execution-replan-compare`, `execution-replan-unplaced`, `execution-replan-confirm`, `execution-replan-keep`, `execution-replan-manualedit`.
- **사용 서버 능력**: `M10.startReplan`, `M10.setManualPosition`, `M10.getAlternatives`, `M10.previewAlternative`, `M10.confirmAlternative`, `M10.keepCurrentPlan`, `M10.applyManualEdit`, (U7)`M12.appendChangeLog`.

### 4.2 TriggerBannerHost (자동 트리거 제안 랜딩)
- **props**: `tripId` · **state**: 활성 배너/칩(유형·영향 일정·출처·감지 시각), 심각/경미 구분.
- **폼 검증(서버 규칙 대응)**: 비차단 배너/칩 — **제안일 뿐, '대안 보기'를 눌러야 재계획 시작**(자동 변경 없음 INV-TRIG3). 심각=푸시 랜딩·경미=인앱 칩(BR-U6-13). **출처·감지 시각 표기**. 닫기 시 재노출 억제(2회 무시→당일 억제 BR-U6-13). 외부 데이터 없으면 배너 미노출(허위 알림 금지 BR-U6-14).
- **상호작용**: `M9.getActiveTriggerBanners` · `M9.dismissTrigger` · '대안 보기' → `ReplanFlow`(triggerRef 전달).
- **data-testid**: `execution-trigger-banner`, `execution-trigger-source`, `execution-trigger-seealternatives`, `execution-trigger-dismiss`.
- **사용 서버 능력**: `M9.getActiveTriggerBanners`, `M9.dismissTrigger`, `M9.getSuppressionState`.

### 4.3 RestModeScreen (휴식 모드)
- **props**: `tripId` · **state**: 재개 시각, 억제 상태.
- **폼 검증(서버 규칙 대응)**: 휴식 진입 시 **경미 트리거·일정 알림 억제, 심각 사유만 유지**(BR-U6-23, INV-REST1). 재개 시각 입력 ∨ 즉시 재개. **재개 시 남은 일정 재계산 제안**(`ResumeRecalcPrompt` — 자동 재계획 아님 BR-U6-24).
- **상호작용**: `M10.enterRestMode(resumeAt?)` · `M10.resumeFromRest`(→ 재계산 제안) · `M9.getSuppressionState`.
- **data-testid**: `execution-rest-resumeinput`, `execution-rest-resumenow`, `execution-rest-recalcprompt`.
- **사용 서버 능력**: `M10.enterRestMode`, `M10.resumeFromRest`, `M9.getSuppressionState`.

### 4.4 PlanVsActualMapScreen (계획 vs 실제 경로 — U3/U5 map 재사용)
- **props**: `tripId`, `date` · **state**: 계획 동선·실제 발자취 레이어, 누적 거리·걸음 수, 옵트인/권한 상태.
- **폼 검증(서버 규칙 대응)**: 계획 동선(흐리게)+실제 GPS 발자취(점선·점) 함께/토글(BR-U6-28). **누적 실제 거리·걸음 수 추정 표기**(걸음 수=거리 환산 G59). **옵트인/권한 없으면 실제 레이어 비활성화+사유 표기**(계획 동선만 BR-U6-25 매트릭스). 지도 실패 시 방문 목록 폴백.
- **상호작용**: `M12.getRouteComparison`(계획·실제·거리·걸음) · `CentroidMapView`/순서 핀은 U3 `shared/map`·U5 map 재사용.
- **data-testid**: `execution-route-planned`, `execution-route-actual`, `execution-route-stats`, `execution-route-disabled`.
- **사용 서버 능력**: (U7)`M12.getRouteComparison`, (U3)`shared/map` 브리지.

### 4.5 UnplacedListScreen (이월 미배치 목록)
- **props**: `tripId` · **state**: 이월 방문지 목록, 날짜 선택.
- **폼 검증(서버 규칙 대응)**: 당일 잔여 재정렬로 이월된 방문지 목록(C10). 사용자가 날짜 선택 시 해당 일 재계산 세션 시작(BR-U6-22).
- **상호작용**: `M10.getUnplacedPois` · `M10.rescheduleUnplaced(targetDate)`.
- **data-testid**: `execution-unplaced-list`, `execution-unplaced-reschedule`.
- **사용 서버 능력**: `M10.getUnplacedPois`, `M10.rescheduleUnplaced`.

---

## 5. shared/location — 위치 권한 3층 상태·포그라운드 수집 (U6 신규, G182)

- **책임**: 위치 동의 **3층(① OS 권한 × ② 위치기반서비스 법정 동의 × ③ GPS 기록 옵트인)** 상태를 관리하고, 조합별 기능 동작 매트릭스(BR-U6-25)를 클라이언트에서 적용한다. 포그라운드 지오펜스 감지·GPS 저빈도 수집을 담당하되 **판정 정본은 서버**(도착=사용자 탭·트리거=서버 M9). **수집 로직·SDK 브리지는 NFR/Code Generation 소유**, 본 문서는 계층 계약만 규정한다.
- **props**: `LocationConsentState{ osPermission, legalConsent, gpsOptIn }` · **state**: 3층 상태, 포그라운드 활성 여부, GPS 정확도.
- **상호작용·폴백**:
  - `GeofenceForegroundWatcher`: 포그라운드 지오펜스 진입 → `M18.onGeofenceEnter`(프롬프트만). **백그라운드 위치 미사용**(G62). 저정확도/권한 없음 → 감지 생략(수동 체크인).
  - `GpsTrailCollector`: **GPS 옵트인 전제**(미동의 미수집 INV-GPS1) 포그라운드 저빈도(1~5분) 수집·단순화 → `M12.appendGpsTrack`(원시 파기 후 폴리라인만). 철회 시 즉시 중단·`M12.purgeLocationData`.
  - `LocationFallbackResolver`: 3층 조합별 동작 매트릭스 적용(BR-U6-25) — 권한 거부→재계획 수동 위치 입력(US-E7-10)·옵트인 거부→발자취 레이어 disabled. 전 조합 폴백 존재.
  - 위치 just-in-time 발화 2번째 지점(여행 중 첫 진입 — US-E1-04 연동).
- **data-testid**: `execution-location-consent`, `execution-location-fallback`, `execution-location-geofence`.
- **사용 서버 능력**: `M18.onGeofenceEnter`, (U7)`M12.appendGpsTrack`, (U7)`M12.purgeLocationData`, (U1)위치 동의 증적.

## 6. shared/ 재사용·U6 완성분 (요약)

| 계층 | U6 범위 | 근거 |
|---|---|---|
| shared/map (U3 소유) | **재사용** — 계획 vs 실제 경로 지도·순서 핀·이동 동선·발자취 레이어(카카오 브리지). U6는 신규 지도 브리지 미생성 | components §6.1, U3 shared/map, FD-U6-09 |
| U5 일정 지도 뷰 | **재사용** — 순서 핀·계획 동선 렌더(재계획 전후 비교·경로 비교) | U5 frontend-components MapView |
| shared/location | **신규** — 위치 3층 상태·포그라운드 지오펜스·GPS 수집·폴백 매트릭스 | G182, N2, D34, G62 |
| shared/api | 트리거 배너 스트림·폴백 고지(허위 알림 금지)·재계획 10초 진행·오프라인 조회 미보장 오류 안내 | ADR-0011, D38, D24 |
| shared/ui | 거리·'추정' 표기·'미확인' 표기·전후 비교 diff 시각화·심각/경미 배너 | D25, G199, PRD 07-8 |

## 7. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M18·M9·M10·M11·M12) |
|---|---|---|
| ActiveTripHubScreen | US-E2-02·03, US-E5-08, US-E6-03 | M18.getActiveHub · getTripProgress · getNextLegDistance · M9.getActiveTriggerBanners |
| ArrivalPromptSheet + VisitCheckControls | US-E6-01, US-E8-01 | M18.onGeofenceEnter · confirmArrival · completeVisit · skipVisit · undoVisitAction · (U7)M12.checkVisit |
| PlaceContextSheet | US-E6-02 | M18.getPlaceContext · M10.startReplan(주변 추천 편입) |
| NextLegBar + ExternalMapHandoffSheet | US-E6-03 | M18.getNextLegDistance · buildExternalMapHandoff · handleExternalMapReturn |
| EndTripConfirmDialog | US-E7-01, US-E8-08 | M18.endTripManually |
| TriggerBannerHost | US-E7-02 | M9.getActiveTriggerBanners · dismissTrigger · getSuppressionState · updateSensitivity |
| ReplanFlow | US-E7-01·03·04·05·07·08·09·10·11·12 | M10.startReplan · setManualPosition · getAlternatives · previewAlternative · confirmAlternative · keepCurrentPlan · applyManualEdit · (U7)M12.appendChangeLog |
| RestModeScreen | US-E7-06 | M10.enterRestMode · resumeFromRest |
| PlanVsActualMapScreen | US-E7-13 | (U7)M12.getRouteComparison · (U3)shared/map |
| UnplacedListScreen | US-E7-07 | M10.getUnplacedPois · rescheduleUnplaced |
| shared/location | US-E1-04, US-E6-01, US-E7-10·13 | M18.onGeofenceEnter · (U7)M12.appendGpsTrack · purgeLocationData |
