# U6 여행 중 실행·Plan-B — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M18 Trip Execution, M9 Plan-B Detection, M10 Itinerary Recalculation, M11 Weather & Context (+ C1 LLM Gateway·C2 Solver Engine 재사용 계약 수준)
> 정본 관계: [components.md](../../../inception/application-design/components.md) M18·M9·M10·M11 섹션 + **§3.1 여행 상태 머신·§3.3 방문 상태 머신 정본**의 엔티티 개요를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U6 엔티티·불변식·상태 전이의 정본이다. 기술 중립 — 저장 기술·지오펜스 SDK·기상청 어댑터·remote config 수치(상한·임계·폴링 주기·안전계수 G106)는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL 의미 명기), `유니크`=제약 범위 내 유일, `불변`=생성 후 변경 불가, `파생`=저장 안 함(조회 시점 계산). 근거 ID는 requirements.md(D·Δ·N·G·C·ADR)·stories.md(US-E6/E7)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U6-xx)을 참조한다.
> **U6는 CP3(U5→U6)의 소비자이자 CP4(U6→U7)의 공급자다** — 일정 기준선(Itinerary plan/current·DaySchedule·ItinerarySlot·FixedBlock)은 U5([u5-itinerary/functional-design/domain-entities.md](../../u5-itinerary/functional-design/domain-entities.md))가 소유하는 입력이며(§10 소비 계약), 본 유닛의 실행 결과(방문 체크·여행 종료·changelog diff·GPS 폴리라인)가 U7 기록·회고의 원천이다(§11 공급 계약).
> **C1·C2는 U5 완성 재사용 자산**(재계획 사유 해석=C1, 하드 제약 4계열 재검증=C2) — 본 문서는 재사용 계약(입출력·하드 제약 재사용 선언)까지만 정의하고 구현체는 U5·NFR 소유다(§12, FD-U6-03).

## 0. 엔티티 지도

```text
[M18 Trip Execution — 실행 허브·방문 상태 머신·여행 종료 전이 정본]
Trip 1 ──── 1 TripExecutionState        (여행 실행 상태 — ACTIVE 하위 NORMAL/REST·종료 전이 D19)
TripExecutionState 1 ── 0..1 RestMode    (휴식 모드 — 경미 억제·심각 유지·재개 재계산 G54)
Trip 1 ──── 1 ActiveTripHub              (활성 허브 읽기 모델 — 진행률·현재/다음 슬롯·다음 예정지 거리·여유 시간 · 파생 집약)
ActiveTripHub 1 ──── * VisitState        (슬롯별 방문 상태 — §3.3 정본: 다음→도착확인대기→진행중→완료/스킵 D23·체류 측정)
VisitState 1 ──── 0..* ArrivalPromptLog  (도착 프롬프트 노출 이력 — 재프롬프트 억제 근거)

[M9 Plan-B Detection — 자동 트리거 4종·상한/억제·민감도(판정 순수 함수 G116)]
TriggerEvent                             (트리거 수명 DETECTED→FIRED→SHOWN→ACCEPTED/DISMISSED→SUPPRESSED)
SuppressionState                         (여행별 상한 카운터·사유별 무시 누적·당일 억제 목록)
SensitivitySetting                       (민감도 3단계 ±50% — accountId별)
PollingSchedule                          (활성 여행별 다음 평가 시각 — 날씨 1시간·휴무 아침)

[M10 Itinerary Recalculation — 재계획 세션·후보·이월(당일 잔여 C10)]
ReplanSession                            (세션 STARTED→CANDIDATES→COMPARING→CONFIRMED/CANCELLED)
ReplanSession 1 ──── 0..3 Alternative    (대안 후보 2~3개 — 솔버 통과분만)
Trip 1 ──── 0..1 UnplacedList            (이월 미배치 방문지 — 날짜 재배치 대기)

[M11 Weather & Context — 기상청 예보·특보(D10)]
WeatherData = ForecastCache(격자·시간대·강수확률) + WeatherAlert(지역 특보)

[M12 계약 — GPS 발자취(수집=U6·영속·기록 귀속=M12/U7)]
GpsTrail                                 (단순화 폴리라인·옵트인 전제 D34/G55/G73)

[CP3 입력 — U5 소유(본 유닛은 참조·소비만, §10)]
Itinerary(plan 불변/current 가변·상태 INTRIP) + DaySchedule[] + ItinerarySlot[](slotId·LOCK·고정 블록·DistanceRange) + FixedBlock[]

[CP4 출력 — U7 M12·M13 소비(§11)]
VisitChecked · DayClosed · TripEnded · changelog diff(G132) · GPS 폴리라인
[CP5 출력 — U8 M14 소비] TriggerFired
```

---

## 1. TripExecutionState — 여행 실행 상태 (M18, 여행 상태 머신 ACTIVE 하위 정본)

진행 중 여행의 **실행 국면 상태**. components.md §3.1 여행 상태 머신에서 `ACTIVE` 진입(시작일 00:00, plan/current 분리 시작 D14) 이후의 실행 하위 상태(`NORMAL`/`REST`)와 **여행 종료 전이(D19)**를 소유한다. 여행 상태 머신 본체(PLANNED/CONFIRMED/ACTIVE/ENDED)는 M6 소유이며 본 엔티티는 ACTIVE/ENDED 전이의 **트리거 주체**다.

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 · 유니크 | 귀속 여행. **진행 중 여행은 항상 최대 1개**(D21 날짜 겹침 차단으로 구조적 단수) [D21] |
| phase | 열거 {NORMAL, REST} | 필수 · 기본 `NORMAL` | ACTIVE 하위 실행 국면(§1.3). REST=휴식 모드(§4 RestMode 참조) [G54] |
| currentDate | 날짜 | 필수 | 실행 기준 일자(일자 경계 배치가 갱신, `DayClosed` 소스) [D19] |
| currentSlotRef | 식별자(ItinerarySlot 참조) | 선택 | 현재 진행 중(INPROGRESS) 슬롯. NULL=이동 중/미도착 |
| nextSlotRef | 식별자(ItinerarySlot 참조) | 선택 | 다음 예정(UPCOMING) 슬롯. NULL=당일 잔여 없음 |
| endedAt | 시각(UTC) | 선택 | 종료 전이 시각. NULL=진행 중. 값 존재=`ENDED`(M6 상태와 정합) [D19] |
| endMethod | 열거 {AUTO_NEXTDAY, MANUAL} | 선택 | 종료 방식 — 종료일 익일 00:00 자동 배치 / 수동 버튼. NULL=미종료 [D19/Δ4] |

### 1.2 파생 표시값 (저장 안 함)

| 파생값 | 계산 | 근거 |
|---|---|---|
| isActive | `endedAt = NULL`(M6 ACTIVE 정합) | 재계획 진입·트리거 평가 게이트(INV-EXEC2) [PRD 07-1] |
| progress | 완료(COMPLETED)+스킵(SKIPPED) 방문 ÷ 당일 계획 방문(G3) | 홈 카드·여행 목록 진행률 [G3] |

### 1.3 여행 실행 상태 전이 (정본 — components.md §3.1 ACTIVE 하위 상세화)

```text
   (M6 여행 CONFIRMED/PLANNED)
        │ startDateReached(시작일 00:00 · M18 일자 경계 배치)
        ▼
     ACTIVE / NORMAL ◀──(resumeFromRest: 재개 시각 도달 ∨ 즉시 재개)──┐
        │  ▲                                                          │
        │  └──────────(enterRestMode: 사용자 휴식 전환)──────────▶ ACTIVE / REST
        │ autoEndTrips(종료일 익일 00:00) ∨ endTripManually(수동 버튼)
        ▼
      ENDED (종결) — TripEnded 발행 → 회고 자동 생성·기록 귀속 마감
```

| 전이 | 트리거 | 가드 조건 | 효과 |
|---|---|---|---|
| (M6) → ACTIVE/NORMAL | 시작일 00:00 도달(M18 일자 경계 배치) | 여행 날짜 구간 진입 · 일정 미확정이라도 ACTIVE(PRD 07-1 '여행 중' 정의) · D21로 동시 활성 ≤1 | 홈 활성 카드·일정 탭이 M18 허브(ActiveTripHub)로 수렴, plan/current 분리 작동(D14) |
| NORMAL → REST | `enterRestMode(resumeAt?)` | 진행 중 여행(isActive) | 경미 트리거·일정 알림 억제 신호(M9·M14), resumeAt을 리마인드 큐 등록(§4) [G54] |
| REST → NORMAL | `resumeFromRest`(재개 시각 도달 알림 ∨ 즉시 재개) | REST 상태 | 억제 해제 + **남은 일정 재계산 제안**(M10 위임) [G159] |
| ACTIVE(NORMAL/REST) → ENDED | (a) `autoEndTrips`(종료일 익일 00:00 배치) (b) `endTripManually`(수동 버튼) | isActive · **숙소 유무 무관 단일 규칙**(D19) · 조건부 전이(이미 ENDED면 no-op) | `TripEnded` 발행 → M13 회고 생성·M14 완료 알림·M12 기록 귀속 마감(CP4) [D19/Δ4] |
| 일자 경계(전이 아님) | `autoEndTrips` 배치 동반 일자 경계 | currentDate < 오늘 | `DayClosed` 발행(당일 회고 초안 트리거 M13) + currentDate 갱신 [S4.6] |

### 1.4 불변식 (Invariants)

- **INV-EXEC1 (종료 전이 단일성·멱등 — 하드 계열)**: `ACTIVE→ENDED`는 자동(익일 00:00)·수동(버튼) 어느 경로든 **조건부 전이(이미 ENDED면 no-op)**로 정확히 1회만 발생한다 — 자동/수동 경합에도 `TripEnded`는 여행당 1회. endedAt·endMethod는 첫 전이에서만 확정 [D19/Δ4, services S4.1, PBT U6-P8].
- **INV-EXEC2 (진행 중 단수)**: isActive TripExecutionState는 계정당 최대 1개 — D21(날짜 겹침 차단)의 소비. 재계획·트리거 평가 대상 여행 모호성 0 [D21, PRD 07-1].
- **INV-EXEC3 (하위 상태 가드)**: phase는 §1.3 전이표 조합으로만 변경 — REST 진입은 사용자 명시 선택, 복귀는 재개 시각 도달/즉시 재개만. 표 밖 전이(ENDED→ACTIVE 재개 등) 구조적 불가(종료는 최종 상태, 재개 없음 C11) [G54, PRD 07-6].
- **INV-EXEC4 (종료 = 회고 트리거 유일 소스)**: 회고 자동 생성·기록 귀속 마감은 `TripEnded`(전체)·`DayClosed`(당일)로만 개시된다 — U6이 발행, U7이 구독(CP4). 종료 후 기록 편집은 허용하되 회고 갱신은 수동 재생성만(C11) [D19, C11, CP4].

---

## 2. ActiveTripHub — 활성 여행 허브 (M18, 읽기 모델 집약)

여행 중 화면의 **단일 허브 읽기 모델**. 홈 활성 카드·일정 탭이 수렴하는 하나의 허브(PRD 01-3)로, 오늘 일정·방문 진행 상태·다음 예정지 거리·여유 시간을 집약한다. plan/current 중 **current가 실행 기준선**이며(CP3 소비), 표시 데이터는 전부 거리·시각까지만(**소요시간 부재 D25**).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 · 유니크 | 진행 중 여행(D21 단수) |
| todaySlots | 목록<{slotRef, visitStatus, 시각/체류}> | 필수 | 오늘 일정 슬롯 + 방문 진행 상태(완료/진행 중/다음) — current DaySchedule 파생(CP3) [PRD 08-1] |
| nextSlot | 참조(ItinerarySlot) | 선택 | 다음 예정지. NULL=당일 잔여 없음 |
| nextLegDistance | 구조체 DistanceRange{value, transportMode, estimatedFlag} | 선택 | 현재 위치→다음 예정지 **거리 요약**(추정 표기) — **소요시간 필드 없음**(INV-HUB2). 화면 진입/포커스 1회+수동 새로고침(G65) [D25, G65] |
| slackToNext | Duration(파생·표시) | 선택 | 다음 계획 시작 시각까지 **단순 시간 차이**(이동시간 미반영 G67) — 여유 시간 [G67, D25] |
| progress | 구조체 {completed, total, ratio} | 필수 | 방문 체크 완료 비율(G3) — 홈 카드·목록 공급 [G3] |
| restState | 참조(RestMode) | 선택 | 휴식 모드 진입 시 존재(§4). NULL=NORMAL [G54] |

**불변식**
- **INV-HUB1 (단일 허브 수렴)**: ActiveTripHub는 진행 중 여행당 1개 — 홈 활성 카드·일정 탭이 동일 허브 데이터를 참조한다(단일 데이터 소스). 여행 종료(ENDED) 시 허브 소멸(홈 카드는 추억 카드로 전환) [PRD 01-3, D21].
- **INV-HUB2 (소요시간 표시 부재 — D25 정본 승계)**: nextLegDistance·slackToNext·모든 이동 표시에 **소요시간(internalDuration) 필드가 구조적으로 없다** — 거리(DistanceRange)와 시각 차이(slack)까지만. internalDuration은 C2 이동 부등식·M9 지연 트리거 판정 내부 전용(U5 INV-TRAVEL1 소비) [D25/Δ1, U5 INV-TRAVEL1, PBT U6-P(표시 게이트 → U5-P4 재사용)].
- **INV-HUB3 (온라인 전제)**: 활성 허브 조회는 온라인 전제 — 실패 시 오류·재시도 안내(오프라인 조회 미보장 D24/Δ6). 기록 '입력'만 M12 로컬 큐(U7) [D24/Δ6].
- **INV-HUB4 (current 기준선)**: 허브의 오늘 일정은 U5 current(가변 현재본)에서 파생되며, 재계획 확정 시 current 갱신이 즉시 반영된다 — plan(불변 스냅샷)은 회고 대조용으로만 소비, 실행 기준선 아님(CP3 시나리오 1) [D14, CP3].

---

## 3. VisitState — 방문 상태 (M18, 방문 상태 머신 정본 §3.3)

일정 슬롯 1건의 **방문 진행 상태 머신**. components.md §3.3의 정본(`다음→도착확인대기→진행중→완료/스킵`)을 계약 수준 상세화한다. **M18이 상태 소유·전이 강제**하며, 실제 방문 기록(actual — 시각·사진·메모)의 **영속은 M12(U7)** 소유다(`VisitRecord`는 M12 엔티티 — 본 엔티티와 혼동 금지, CP4 경계 FD-U6-01). 도착 확정·완료는 **항상 사용자 탭**(자동 기록 없음 D23).

### 3.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| visitStateId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| slotRef | 식별자(ItinerarySlot 참조) | 필수 · 유니크(여행 내) | 대상 슬롯(canonicalPoiId는 슬롯에서 파생 — CP3) [CP3] |
| status | 열거 {UPCOMING, ARRIVALPENDING, INPROGRESS, COMPLETED, SKIPPED} | 필수 · 기본 `UPCOMING` | 방문 생애 상태(§3.3 정본) — 한글: 다음/도착확인대기/진행중/완료/스킵 |
| arrivedAt | 시각(기기 시각) | 선택 | 도착 확정 시각(사용자 탭·수정 가능). NULL=미도착. `VisitChecked(start)`로 M12 actual 공급 [D23] |
| departedAt | 시각 | 선택 | 방문 종료 시각. NULL=미완료. **다음 장소 도착 체크 시각으로 추정 시 estimatedFlag=true**(사용자 완료 탭이면 실측) [D23] |
| departedEstimatedFlag | 불리언 | 필수 · 기본 false | departedAt이 다음 장소 체크로 추정된 값인지(허위 정확성 방지) [D23] |
| promptSuppressed | 불리언 | 필수 · 기본 false | 도착 프롬프트 닫기/무시로 재프롬프트 억제됐는지(§3.4 ArrivalPromptLog 근거) [D23] |

### 3.2 파생값 (저장 안 함)

| 파생값 | 계산 | 근거 |
|---|---|---|
| actualStay | `departedAt - arrivedAt`(둘 다 존재 시) | 실제 체류 — plan 체류와 대조(M12 보관)·M9 체류 초과 트리거 (d) 입력 [D23, PRD 09-1] |
| overstayGap | `actualStay - plan 체류`(양수일 때) | 체류 초과분 — 다음 고정 일정 위협 판정 입력(BR-U6-11) [PRD 07-2d] |

### 3.3 방문 상태 전이 (정본 — components.md §3.3 상세화)

```text
      UPCOMING(다음)
        │  ├──(geofenceEnter · 포그라운드 · 위치 3층 동의 충족)──▶ ARRIVALPENDING(도착확인대기)
        │  ├──(manualArrivalTap · 위치 무관 항상 가능 ADR-0010)────────────────┐
        │  └──(userSkips)──▶ SKIPPED                                          │
        ▼                                                                     ▼
   ARRIVALPENDING ──(userConfirmsArrival: '도착했어요' 탭)──────────────▶ INPROGRESS(진행중)
        ├──(userDismisses: 닫기/무시)──▶ UPCOMING(재프롬프트 억제)             │
        └──(userSkips)──▶ SKIPPED                                            │
                                                                             ▼
                              INPROGRESS ──(userCompletes 탭 ∨ 다음 장소 도착 체크)──▶ COMPLETED(완료)
                                   └──(userCancels)──▶ SKIPPED
   COMPLETED / SKIPPED (종결) — 이후 시각·상태 수정은 기록 편집(상태 재전이 아님 C11)
```

| 전이 | 트리거 | 가드 조건 | 효과 |
|---|---|---|---|
| UPCOMING → ARRIVALPENDING | 지오펜스 진입 감지(포그라운드) ∨ 외부 지도앱 복귀 시 다음 예정지 근접(G66) | 위치 3층 동의 충족(G182) · GPS 정확도 충분(미달 시 이 전이 생략) · 동일 방문지 재프롬프트 억제 | **'도착하셨나요?' 프롬프트만 — 자동 기록 없음**(D23) [D23, G66] |
| ARRIVALPENDING → INPROGRESS | 사용자 '도착했어요' 탭 | — | `arrivedAt` 기록(기기 시각) → M12 actual · `VisitChecked(start)` 발행(CP4) [D23] |
| UPCOMING → INPROGRESS | 사용자 수동 '도착' 탭 | **위치 동의·감지와 무관하게 항상 가능**(ADR-0010) | 동일(수동 체크인 폴백 기본 수단) [D23, ADR-0010] |
| ARRIVALPENDING → UPCOMING | 프롬프트 닫기/무시 | — | promptSuppressed=true(해당 방문지 재프롬프트 억제) [D23] |
| INPROGRESS → COMPLETED | (a) '방문 완료' 탭 (b) 다음 장소 도착 체크 | (b)의 departedAt은 다음 장소 체크 시각 추정(estimatedFlag=true) | 실제 체류 산출 → M12 보관(plan 대조) · **체류 초과 시 M9 신호**(`VisitChecked(complete)`) [D23, PRD 09-1] |
| any → SKIPPED | 사용자 '방문 안 함/스킵/취소' | — | M12 스킵 기록 · 잔여 일정 영향 시 Plan-B 제안 연결(BR-U6-05) [D23, PRD 09-1] |
| COMPLETED/SKIPPED 이후 | 시각·상태 수정 | 기록 편집으로 처리(**상태 재전이 아님** C11) | changelog 기록(M12) [C11, PRD 09-1] |

### 3.4 ArrivalPromptLog — 도착 프롬프트 이력 (M18)

| 속성 | 타입 | 제약 | 의미 |
|---|---|---|---|
| slotRef | 식별자(ItinerarySlot 참조) | 필수 | 대상 슬롯 |
| shownAt | 시각 | 필수 | 프롬프트 노출 시각 |
| outcome | 열거 {CONFIRMED, DISMISSED, SKIPPED} | 필수 | 결과 — 재프롬프트 억제 근거 |

### 3.5 불변식

- **INV-VISIT1 (전이 가드 — §3.3 정본)**: status는 §3.3 전이표 (트리거·가드) 조합으로만 변경 — 표 밖 전이(COMPLETED→INPROGRESS 역행·UPCOMING→COMPLETED 직행) 구조적 불가. COMPLETED/SKIPPED는 종결(이후는 기록 편집) [D23, C11, PBT U6-P1(stateful)].
- **INV-VISIT2 (자동 기록 없음 — 하드 계열)**: 지오펜스 진입은 **프롬프트만** 생성하고 어떤 상태도 자동 확정하지 않는다 — INPROGRESS·COMPLETED 전이는 항상 사용자 탭이 트리거. 위치 감지 실패가 방문 처리를 막지 않음(수동 경로 항상 존재 ADR-0010) [D23, ADR-0010, PBT U6-P1].
- **INV-VISIT3 (체류 측정 결정성)**: actualStay = departedAt − arrivedAt은 동일 (arrivedAt, departedAt) 입력에 결정적 산출 — departedAt이 추정(다음 장소 체크)이면 departedEstimatedFlag=true로 전파(허위 정확성 금지). arrivedAt 부재 시 actualStay 미산출(NULL) [D23, PBT U6-P2].
- **INV-VISIT4 (완료 방문 불변 — 재계획 제외)**: COMPLETED/SKIPPED 방문은 재계획 재정렬 대상에서 제외되고 warm-start로 보존된다 — 재계획은 "현재 시각 이후 남은 일정만"에 적용(INV-REPLAN4 대응) [PRD 07-3, C10].
- **INV-VISIT5 (actual 영속 경계 — CP4)**: 방문 시각·사진·메모의 **영속 정본은 M12(U7) VisitRecord**이며, M18 VisitState는 실행 중 상태 머신만 소유한다 — M18은 `VisitChecked` 이벤트로 M12에 공급(생산까지, 보관·대조는 U7) [FD-U6-01, CP4, D23].

---

## 4. RestMode — 휴식 모드 (M18, ACTIVE 하위 상태 G54)

'여행 중'의 하위 상태(TripExecutionState.phase=REST). 진입 시 **경미 트리거·일정 알림을 억제**하고 심각 사유(기상특보·고정 일정 위협)만 유지하며, 재개 시각 도달 시 알림 + 남은 일정 재계산을 제안한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 · 유니크 | 귀속 여행(진행 중 단수) |
| enteredAt | 시각 | 필수 | 휴식 진입 시각 |
| resumeAt | 시각 | 선택 | 재개(휴식 종료) 예정 시각. NULL=즉시 재개(사용자가 수동 재개) [PRD 07-6] |
| suppressionScope | 열거 {MINOR_ONLY} | 필수 · 기본 `MINOR_ONLY` | 억제 범위 — **경미 트리거·일정 알림만 억제, 심각 사유는 억제 불가**(INV-REST1) [G54] |

**불변식**
- **INV-REST1 (심각 억제 불가 — 안전 불변식)**: 휴식 모드는 severity=MINOR 트리거·일정 알림만 억제하고 **severity=SEVERE(기상특보·고정 일정 도착 위협)는 억제하지 못한다** — 휴식 중에도 안전 위협 알림은 통과(방해금지 예외 G100과 정합) [G54, G100, PBT U6-P3(억제 규칙)].
- **INV-REST2 (재개 재계산 제안)**: resumeAt 도달(∨ 즉시 재개) 시 알림 + `resumeFromRest`가 **남은 일정 재계산 제안**(ReplanProposal)을 반환한다 — 자동 재계획은 아니며 사용자가 대안 보기를 눌러야 M10 세션 시작 [G159, INV-TRIG3 정합].
- **INV-REST3 (하위 상태 종속)**: RestMode는 TripExecutionState.phase=REST일 때만 존재하며, 여행 종료(ENDED)·즉시 재개 시 소멸 — 독립 생명주기 없음 [G54, INV-EXEC3].

---

## 5. TriggerEvent — 자동 트리거 (M9, 트리거 수명 정본)

여행 중 재계획 필요 상황 4종(날씨·휴무·이동 지연·체류 초과)의 **감지·발화·억제 단위**. **제안만** 발화하고 자동으로 일정을 바꾸지 않는다(INV-TRIG3). 판정 로직은 clock·외부 데이터 주입 순수 함수로 분리(G116).

### 5.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| triggerId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 진행 중 여행 |
| type | 열거 {WEATHER, CLOSURE, DELAY, OVERSTAY} | 필수 | 사유 4종 — (a)날씨·(b)휴무·(c)이동 지연·(d)체류 초과 [PRD 07-2] |
| severity | 열거 {SEVERE, MINOR} | 필수 | 심각도 — SEVERE(기상특보·고정 일정 도착 위협)=푸시 / MINOR=인앱 칩 [PRD 07-2] |
| affectedSlots | 목록<slotRef> | 필수 | 영향 받는 슬롯(다음 방문지·고정 일정) |
| source | 구조체 {provider, kind} | 필수 | 외부 데이터 출처(기상청·영업시간)·감지 계열 — **알림 출처 표기 입력**(허위 알림 방지) [PRD 07-2] |
| detectedAt | 시각 | 필수 | 감지 시각 — 알림 표기 |
| lifecycle | 열거 {DETECTED, FIRED, SHOWN, ACCEPTED, DISMISSED, SUPPRESSED} | 필수 · 기본 `DETECTED` | 트리거 수명(§5.4 상태 머신) |
| detectionMode | 열거 {SERVER_POLL, CLIENT_FOREGROUND} | 필수 | 하이브리드 경로 — 날씨·휴무=서버 폴링 / 지연·체류=클라 포그라운드(D27) [D27, G62] |

### 5.2 SuppressionState — 억제 상태 (M9)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자 | 필수 · 유니크 | 여행별 억제 상태 |
| hourlyCount / dailyCount | 정수 | 필수 · 기본 0 | 전역 상한 카운터(초기값 시간당 2/일 8 — remote config G58) [G58/G195] |
| ignoreStreakByReason | Map<type, 정수> | 필수 · 기본 ∅ | 사유별 연속 무시 카운트 — **2회 연속 시 당일 동일 사유 억제(학습)** [G58, PRD 07-2] |
| dailySuppressedReasons | 집합<type> | 필수 · 기본 ∅ | 당일 억제된 사유(학습 결과)·휴식 모드 경미 억제 반영 [G54, G58] |
| shownKeys | 집합<{type, slotRef}> | 필수 · 기본 ∅ | 동일 사유·동일 방문지 1회 노출 보장 키 [PRD 07-2] |

### 5.3 SensitivitySetting / PollingSchedule (M9)

| 엔티티 | 속성 | 의미·근거 |
|---|---|---|
| SensitivitySetting | accountId, level {적게/보통/많이} | 민감도 3단계 — 상한 ±50% 조정(remote config) [G58/G195] |
| PollingSchedule | 대상 활성 여행, nextEvalAt | 배치 폴링 스케줄 — 날씨 1시간·휴무 당일 아침 1회(remote config) [G58/G195] |

### 5.4 트리거 수명 상태 머신 (정본)

```text
   [evaluateServerTriggers 배치 ∨ reportClientSignal 포그라운드]
        │
        ▼
    DETECTED ──(상한·억제·민감도·위치 게이트 통과)──▶ FIRED ──▶ SHOWN(배너/칩·푸시는 심각만)
        │  └──(상한 초과·당일 억제·휴식 경미·위치 미동의)──▶ SUPPRESSED(발화 안 함·계측)
        │                                                       ▲
        ▼                                                       │
   SHOWN ──(사용자 '대안 보기')──▶ ACCEPTED(→ M10 재계획 세션 진입)
        └──(닫기/무시)──▶ DISMISSED ──(동일 사유 2회 무시)──▶ SUPPRESSED(당일 학습)
```

| 전이 | 트리거 | 가드/효과 |
|---|---|---|
| (없음) → DETECTED | 서버 배치 판정 ∨ 클라 신호 판정 | 순수 함수 판정(clock·외부 데이터 주입 G116) |
| DETECTED → FIRED | 상한·억제·민감도·위치 게이트 통과 | outbox `TriggerFired`(CP5) → M14(심각=푸시·경미=인앱 칩) [PRD 07-2] |
| DETECTED → SUPPRESSED | 상한 초과·당일 억제·휴식 경미·위치 미동의·외부 데이터 없음 | 발화 안 함 + 실패/억제 계측(침묵이지만 관측) [ADR-0011] |
| FIRED → SHOWN | 배너/칩 노출 | 출처·감지 시각 표기 [PRD 07-2] |
| SHOWN → ACCEPTED | 사용자 '대안 보기' 탭 | M10 `startReplan`(자동 진입) — **자동 일정 변경 없음** [PRD 07-2, INV-TRIG3] |
| SHOWN → DISMISSED | 닫기/무시 | 동일 방문지·사유 재노출 방지 · ignoreStreak++ |
| DISMISSED → SUPPRESSED | 동일 사유 2회 연속 무시 | 당일 동일 사유 억제(학습) [G58] |

### 5.5 불변식

- **INV-TRIG1 (상한 불변식 — G58)**: 임의 (clock·트리거 시퀀스) 입력에 대해 FIRED 발화 수는 **전역 상한(시간당 2/일 8, 민감도 ±50%)을 초과하지 않는다** — 초과분은 묶어 1회로 정리·SUPPRESSED. 동일 {type, slotRef}는 1회만 SHOWN [G58/G195, PBT U6-P3].
- **INV-TRIG2 (억제 멱등·학습)**: 동일 사유 2회 연속 무시 후 당일 동일 사유는 SUPPRESSED로 억제되고, 이 억제는 멱등(반복 판정에도 동일 사유 재발화 0) — 학습 상태는 당일 리셋(익일 재평가) [G58, PBT U6-P3].
- **INV-TRIG3 (제안만 — 자동 변경 없음)**: TriggerEvent는 **어떤 경로로도 일정을 자동 변경하지 않는다** — ACCEPTED(대안 보기) 후에도 M10 세션의 사용자 확정(§6 INV-REPLAN)이 있어야 current 갱신. 발화=제안 [PRD 07-2, INV-REPLAN2].
- **INV-TRIG4 (판정 순수 함수 — G116)**: 트리거 판정은 clock·날씨·체류·이동 데이터를 파라미터로 주입받는 순수 함수 — 동일 입력 동일 판정(결정적 시뮬레이션 테스트). 민감도 레벨 상향 시 발화 임계 완화(단조성) [G116, PBT U6-P4].
- **INV-TRIG5 (위치 동의 게이트 — D34)**: 위치 의존 트리거(c 이동 지연·d 체류 초과)는 **위치정보 동의 철회 상태에서 미평가**되고, 위치 비의존(a 날씨·b 휴무)만 지속 평가된다 — 동의 3층(G182) 종속 [D34, PRD 12-3, INV-GPS1 정합].
- **INV-TRIG6 (허위 알림 금지)**: 외부 데이터 부재(`DataUnavailable`)는 '맑음/정상'과 **구분**되어 자동 알림을 발화하지 않는다(침묵) — 수동 재계획 경로는 항상 유지, 실패는 관측 계측 [ADR-0011, PRD 07-2 예외, INV-WX1 연계].

---

## 6. ReplanSession — 재계획 세션 (M10, 세션 상태 머신 정본)

수동·자동 트리거로 시작된 **재계획 세션**. 사유 해석(C1)→후보 2~3개 생성(M7 소싱+C2 검증)→전/후 비교→확정(current 반영)까지를 소유한다. 등록 숙소는 **항상 불변 고정 제약**(ADR-0006 — Plan-B는 예약 변경이 아닌 실행 보조).

### 6.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| sessionId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 대상 진행 중 여행(D21 단수) |
| reason | 열거 {Weather, Closure, Delay, Cancelled, Fatigue, None} | 선택 | 재계획 사유(5종 + '사유 없음'). NULL 불가·None=명시적 무사유 [PRD 07-1] |
| method | 열거 {DelegateToAi, ManualEdit} | 필수 | 대안 획득 방식 — AI에게 맡기기 / 직접 수정(수동 편집 직행) [PRD 07-12] |
| triggerRef | 식별자(TriggerEvent 참조) | 선택 | 자동 진입 시 원천 트리거. NULL=수동 진입 |
| basePosition | 구조체 {coord, kind} | 선택 | 기준 위치 · kind∈{GPS, MANUAL, LAST_VISIT, STAY} — **가정 표기 대상**(추정 출발지). NULL+GPS 불가 시 `positionRequired` 상태 [PRD 07-10] |
| positionAssumption | 문자열 | 선택 | 폴백 가정 표기("추정 출발지 — 지도 핀 입력"·"마지막 완료 방문지 기준"). NULL=GPS 실측 [PRD 07-10] |
| status | 열거 {STARTED, CANDIDATES, COMPARING, CONFIRMED, CANCELLED} | 필수 · 기본 `STARTED` | 세션 생애(§6.3) |
| candidates | 목록<Alternative> | 선택 | 대안 후보(§7) — CANDIDATES 이후 존재 |

### 6.2 파생·게이트값

| 값 | 의미 | 근거 |
|---|---|---|
| replanScope | 재정렬 범위 = **당일 잔여만**(완료 방문·시각 고정 제외) | C10, PRD 07-3·7 |
| fixedConstraints | 숙소 체크인/아웃(불변)·시각 고정형 필수 방문지·완료 방문지(warm-start 보존) | ADR-0006, PRD 07-3 |

### 6.3 재계획 세션 상태 머신 (정본)

```text
   [startReplan(tripId, reason?, method, position?)]  — 가드: '여행 중'(날짜 구간 D19)·숙소 0개도 허용
        │
        ▼
    STARTED ──(method=ManualEdit)──▶ (수동 편집 화면 직행 · applyManualEdit)
        │ method=DelegateToAi
        │ getAlternatives (10초 예산 · C1 사유 해석 + M7 소싱 G53 + C2 검증)
        ▼
    CANDIDATES ──(후보 0건/전부 검증 실패/10초 초과)──▶ Fallback(3종: 1개 건너뛰기/휴식 전환/수동 수정)
        │ previewAlternative(choiceId)
        ▼
    COMPARING ──(confirmAlternative: 확정 버튼)──▶ [C2 재검증 1회 G56]
        │                                              ├─ 무효화 → CANDIDATES(재산출 안내)
        │                                              └─ 통과 → CONFIRMED
        └──(keepCurrentPlan '기존 유지')──▶ CANCELLED(어떤 변경도 미저장)
   CONFIRMED — current만 갱신(D14) + changelog(M12) + ItineraryChanged(CP5)
```

| 전이 | 트리거 | 가드/효과 |
|---|---|---|
| (없음) → STARTED | `startReplan` | **'여행 중' 상태만**(날짜 구간 D19, 숙소 0개 허용) — 위반 시 `PreconditionFailed(NOT_ACTIVE)` [PRD 07-1] |
| STARTED → (수동 편집) | method=ManualEdit ∨ 외부 API 오류 폴백 | `applyManualEdit` — 숙소 고정 제약 위반 차단(BR-U6-19) [PRD 07-11·12] |
| STARTED → CANDIDATES | `getAlternatives` 성공 | C1 사유 해석 + M7 저장 장소 우선 소싱(G53) + C2 하드 제약 검증 → 2~3개(10초 D38) [G53, D38] |
| CANDIDATES → Fallback | 후보 0건·전부 실패·10초 초과 | 3종 폴백 + 사유 한 줄(빈 화면 금지) [PRD 07-4·12 예외] |
| CANDIDATES → COMPARING | `previewAlternative(choiceId)` | 전/후 비교(추가·삭제·시간 이동 diff, 지표 요약 — 소요시간 없음) [PRD 07-8] |
| COMPARING → CONFIRMED | `confirmAlternative` + **C2 재검증 1회 통과(G56)** | current 갱신(plan 불변 D14) + changelog + `ItineraryChanged` [G56, D14] |
| COMPARING → CANDIDATES | 확정 재검증 무효화 | `RevalidationFailed` — 후보 재산출 안내(확정 차단) [G56] |
| any → CANCELLED | `keepCurrentPlan` '기존 유지' ∨ 세션 폐기 | 어떤 변경도 미저장 [PRD 07-6] |

### 6.4 불변식

- **INV-REPLAN1 (여행 중 게이트)**: startReplan은 '여행 중'(날짜 구간 진입 D19) 상태에서만 성공 — **등록 숙소 0개여도 날짜 구간 안이면 허용**(숙소=생성 기준일 뿐, 상태 진입 조건 아님). 여행 중 아님이면 `PreconditionFailed(NOT_ACTIVE)` [PRD 07-1, D19, INV-EXEC2].
- **INV-REPLAN2 (current만 갱신·plan 불변 — 하드 계열)**: 확정(CONFIRMED)은 **current에만** 반영되고 U5 plan 스냅샷은 바이트 동일하게 불변 유지된다 — 재계획 연산 시퀀스 후에도 plan 변경 0(U5 INV-PLAN1 소비). current 갱신은 확정 TX(S2 11단계)에서만, 그 전 어떤 후보도 일정 미반영 [D14, CP3, PBT U6-P5].
- **INV-REPLAN3 (확정 시점 재검증 — G56)**: CONFIRMED 전이는 **확정 버튼 시점 C2 재검증 1회 통과** 없이 불가 — 그 사이 상황 변화로 무효화되면 CANDIDATES 복귀·재산출 안내(반영 차단). 재검증은 U5 하드 제약 4계열 재사용(§12, U5-P1) [G56, PBT U6-P7].
- **INV-REPLAN4 (당일 잔여 범위 — C10)**: 재정렬 대상은 **현재 시각 이후 당일 잔여 방문지만** — 완료(COMPLETED/SKIPPED) 방문·시각 고정형 필수 방문지·숙소 고정 블록은 재배치 대상에서 제외(warm-start 보존). 넣을 수 없게 된 방문지는 UnplacedList로 이월(§7) [C10, PRD 07-3·7, PBT U6-P6].
- **INV-REPLAN5 (숙소 고정 불가침 — ADR-0006)**: 등록 숙소 체크인/체크아웃 일시·위치는 **변경 불가 고정 제약** — Plan-B는 숙소 예약을 바꾸지 않고 출발·복귀 기준점으로 고정한 채 방문 일정만 재구성. AI/직접 수정 어느 방식에서도 위반 불가(수동은 입력 차단) [ADR-0006, PRD 07-3·12].
- **INV-REPLAN6 (10초 예산 — D38)**: getAlternatives는 10초 목표(services S2.4 단계 예산) — 초과 시 산출된 후보만이라도 제시, 0건이면 3종 폴백. 확정 재검증(11단계)은 별도 상호작용으로 10초 예산 외 [D38, services S2.4].
- **INV-REPLAN7 (기준 위치 가정 표기)**: basePosition이 GPS 실측이 아니면(MANUAL/LAST_VISIT/STAY) positionAssumption이 필수로 채워져 화면에 표기된다 — 추정 출발지 가정 명시(허위 정확성 금지) [PRD 07-10].

---

## 7. Alternative / UnplacedList — 대안 후보·이월 (M10)

### 7.1 Alternative — 대안 후보 (2~3개, 솔버 통과분만)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| choiceId | 식별자 | 필수 · 유니크(세션 내) | 후보 택일 키 |
| slots | 목록<ItinerarySlot diff> | 필수 | 재정렬 후 당일 잔여 슬롯(C2 검증값) [C10] |
| rationale | 문자열 | 선택 | 추천 이유(사유 연결 LLM 설명 — 구체 근거·모호어 금지). NULL=템플릿 대체("추천 이유를 불러오지 못했어요") [PRD 07-5] |
| distanceFromHere | 구조체 DistanceRange{value, transportMode, estimatedFlag} | 필수 | 현재 위치→후보 **거리·수단만**(추정 표기) — **소요시간 없음**(INV-ALT2) [D25/Δ1] |
| stayEstimate | 정수(분) | 필수 | 예상 체류(추정 표기) [PRD 07-5] |
| slackToNextFixed | Duration | 필수 | 다음 고정 제약(숙소 복귀·시각 고정 방문지)까지 여유 시간 [PRD 07-5] |
| validationResult | 구조체 {passed, hardConstraintsChecked} | 필수 | C2 하드 제약 4계열 통과분만 보관(INV-ALT1) [PRD 07-4] |

### 7.2 UnplacedList — 이월 미배치 목록 (M10)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자 | 필수 · 유니크 | 대상 여행 |
| unplacedPois | 목록<{poiRef, reason}> | 필수 | 당일 잔여에 못 넣은 방문지 + 사유(이월) [C10] |
| status | 열거 {PENDING_RESCHEDULE} | 필수 | 날짜 재배치 대기 — 사용자가 날짜 선택 시 해당 일 재계산 세션 시작 [C10] |

### 7.3 불변식

- **INV-ALT1 (솔버 통과분만)**: candidates의 모든 Alternative는 C2 하드 제약 4계열(숙소 시간창·영업시간·이동 부등식·시각 고정) 통과분만 — 환각·폐업·좌표 불명 POI는 후보에서 구조적 제외(closed-set 그라운딩 G115, U5 INV-SLOT1 재사용) [PRD 07-4, G115].
- **INV-ALT2 (소요시간 부재 — D25)**: distanceFromHere·전후 비교 지표(총 이동 거리 증감·숙소 복귀 시각 변화)에 소요시간 필드 없음 — 거리·시각까지만(INV-HUB2 동일 규칙) [D25/Δ1, U5 INV-TRAVEL1].
- **INV-ALT3 (사유 부합)**: 후보는 사유에 부합 정렬 — 날씨→실내 우선, 체력(Fatigue)→이동거리·방문 수 축소. C1 사유 해석 실패 시 규칙 매핑 폴백(정렬 힌트 대체) [PRD 07-4, services S2.7 폴백].
- **INV-ALT4 (2~3개 또는 폴백)**: 유효 후보는 2~3개 제시 — 0건이면 빈 화면 대신 **3종 폴백(1개 건너뛰기/휴식 전환/수동 수정) + 사유 한 줄**. AI/직접 수정 두 분기 대칭 [PRD 07-4·12 예외].
- **INV-ALT5 (이월 명시 동의)**: 재정렬로 제외·이월되는 방문지는 확정 전 명시적으로 표시되고 사용자 동의 후에만 UnplacedList로 보관된다(침묵 드롭 금지) [PRD 07-7].

---

## 8. WeatherData — 기상 데이터 (M11, 기상청 D10)

기상청 공공데이터포털 단기예보(강수확률)·기상특보를 수집·캐싱해 M9(트리거 (a) 판정)·M8(생성 맥락)에 공급하는 값 객체. **데이터 부재를 명시**해 소비자가 '데이터 없음'과 '맑음'을 구분하게 한다.

### 8.1 ForecastCache / WeatherAlert

| 엔티티 | 속성 | 제약 | 의미·근거 |
|---|---|---|---|
| ForecastCache | gridXY | 필수 | 기상청 격자 좌표(좌표→격자 변환은 M11 정본, 결정적) [D10] |
| | timeSlot | 필수 | 예보 시간대(도착 시간대 강수확률 조회) |
| | pop | 필수 | 강수확률(%) — **60% 이상이 트리거 (a) 판정 기준** [PRD 07-2a] |
| | sky / announcedAt | 필수 | 하늘 상태·발표 시각(출처 표기 입력) |
| | fetchedAt + TTL | 필수 | 폴링 주기 정합 캐시(1시간)·동일 격자 중복 호출 억제(쿼터 보호) [G195, RESILIENCY-09] |
| WeatherAlert | region, alertType, effectiveAt, source | 필수 | 지역 특보 — **Plan-B 심각 사유·푸시 대상 입력** [PRD 07-2] |

### 8.2 불변식

- **INV-WX1 (데이터 없음 명시 — 허위 알림 방지)**: API 실패 시 `DataUnavailable`을 명시 반환하고 **추정치를 조작하지 않는다** — M9는 이를 받아 자동 알림 침묵(INV-TRIG6), M8은 날씨 무반영 생성 지속. '데이터 없음'≠'맑음' [ADR-0011, PRD 07-2 예외].
- **INV-WX2 (격자 변환 결정성)**: 위경도→기상청 격자 변환은 결정적 순수 함수(동일 좌표 동일 격자) — 테스트 결정성 [D10, PBT U6-P10].
- **INV-WX3 (TTL 캐시·쿼터 보호)**: ForecastCache는 폴링 주기(1시간)와 정합하는 TTL로 동일 격자 중복 호출을 억제 — 쿼터 80% 알람·서킷 브레이커(RESILIENCY-09·10). 폴링 실패 시 마지막 성공 데이터 TTL 내 사용 후 트리거 (a) 계열 일시 중지 [G195, RESILIENCY-09].

---

## 9. GpsTrail — GPS 발자취 (M12 소유·U6 수집 계약)

계획 동선 vs 실제 경로 비교의 원천. **수집은 U6**(포그라운드 저빈도·단순화 폴리라인 생산), **기록 귀속·영속은 M12(U7)**(CP4). GPS 기록 옵트인(N2 3층 동의의 3층) 전제이며, 원시 좌표는 가공 후 파기한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| date | 날짜 | 필수 | 수집 일자(날짜별 귀속) |
| simplifiedPolyline | 폴리라인(단순화) | 필수 · 불변 | **단순화 폴리라인만 서버 보존 — 원시 좌표는 가공 후 파기**(INV-GPS3) [G55/G73] |
| consentRef | 식별자(ConsentRecord 참조) | 필수 | GPS 기록 옵트인 동의 참조 — **미동의 수집 불가**(INV-GPS1) [D34/N2] |
| collectionMeta | 구조체 {interval, foregroundOnly} | 필수 | 수집 구간 메타 — 포그라운드 저빈도(1~5분) [G55, G62] |

**파생 표시값**: cumulativeDistance(누적 실제 이동 거리·추정), estimatedSteps(걸음 수=GPS 거리 환산 추정·추가 권한 없음 G59) — 둘 다 **추정 표기 강제**.

**불변식**
- **INV-GPS1 (옵트인 전제 — D34 하드 계열)**: GpsTrail 수집·저장은 GPS 기록 옵트인 동의(consentRef)를 전제한다 — 미동의 수집은 `PermissionDenied`(구조적 차단). 수집·이용 사실은 append-only 법정 로그로 기록(N2, 앱은 자기 로그 삭제 권한 없음) [D34/N2, SECURITY-14].
- **INV-GPS2 (포그라운드 저빈도 — G55/G62)**: 수집은 포그라운드 한정 저빈도(1~5분) — 백그라운드 위치 권한 미요청(G62). 포그라운드 밖 구간 미수집(제품 동작으로 수용, 앱 재진입 시 재평가) [G55, G62, D27].
- **INV-GPS3 (단순화 라운드트립 — 파기 후 위상 보존)**: 원시 좌표 파기 후 남는 단순화 폴리라인은 **경로 위상(방문 순서·구간 연결)을 보존**하며, 단순화→직렬화→역직렬화 라운드트립에서 위상 동등 — 원시 재구성은 불가(파기)하되 표시·거리 산출에 충분 [G73, PBT U6-P9].
- **INV-GPS4 (철회·탈퇴 즉시 파기 — N2)**: 옵트인 철회·계정 탈퇴 시 GpsTrail·위치 파생 데이터는 **즉시 파기**(30일 유예 없음 D34) — 단 법정 로그(수집·이용 사실)는 분리 보관 유지(M12 `purgeLocationData`, S6.2) [D34/N2, S6.2].
- **INV-GPS5 (수집·영속 경계 — CP4)**: U6은 수집·폴리라인 생산까지, **기록 귀속·경로 비교 데이터·영속 정본은 M12(U7)** — CP4로 공급(GPS 폴리라인 이벤트) [FD-U6-09, CP4, M12].

---

## 10. CP3 소비 참조 계약 (U5 → U6 입력) — 일정 기준선

U6는 U5가 공급하는 **일정 기준선**(plan/current·고정 블록·확정 상태)을 실행 허브·재계획의 입력으로 소비한다(스키마 정본은 U5, 본 유닛 재정의 금지). **5개 유닛(U6·U7·U8·U10·U11) 공유 척추** — U6는 이를 신뢰하되 상태 머신 규칙(D20)을 방어적으로 재검증한다.

| 입력 객체(U5 소유) | U6 소비 용도 | 소비 시 방어 검증 |
|---|---|---|
| Itinerary(plan 불변/current 가변·상태 INTRIP·확정 D20) | 실행 기준선(current) 로드·재계획 결과 반영 대상(current만) | 상태 INTRIP 정합·확정 해제 경합 차단(D20)·plan 불변 유지 [INV-HUB4, INV-REPLAN2] |
| DaySchedule[](일자·거점·시간창) | 오늘 일정 허브 구성·재계획 당일 범위 | 시간창 정합·거점 기준(하드 제약 재검증 시 C2) [CP3] |
| ItinerarySlot[](slotId·LOCK·고정 블록·DistanceRange) | 방문 상태 머신 부착(slotRef)·재계획 재정렬·거리 표시(소요시간 없음) | slotId 무결성·소요시간 미매핑(D25)·closed-set 보존 [INV-VISIT, INV-ALT2] |
| FixedBlock[](숙소 체크인/아웃·시각 고정 방문지) | 재계획 고정 제약(불가침 ADR-0006)·warm-start 보존 | 고정 블록 불변·충돌 자동 변경 금지(U5 INV-FIX 재사용) [INV-REPLAN4·5] |

> **계약 변경 통제**: plan/current 이중 구조·slotId 식별 체계는 ADR급(CP3 §145). U6 착수 후 U5 계약은 필드 추가만 허용(파괴적 변경 금지).

---

## 11. CP4 공급 계약 (U6 → U7 출력) — actual·changelog·GPS

U6 산출(방문 체크·여행 종료·changelog diff·GPS 폴리라인)이 U7 기록·회고 생산의 트리거·원천 데이터다. U6(M18·M10)이 발행하고 U7(M12·M13)이 구독한다(`common/core` 이벤트 버스 위).

| 이벤트/객체 | 필드(개요) | 경계 무결성 |
|---|---|---|
| `VisitChecked` | 여행·일정·slotRef, canonicalPoiId, 도착 확인 시각(사용자 탭 D23), 전이 유형(start/complete/skip), 실제 체류(departedEstimatedFlag) | INV-VISIT2·3·5 |
| `DayClosed` | 여행·일자, 당일 방문 요약(완료·스킵 카운트) — 당일 회고 초안 트리거 | INV-EXEC1·4 |
| `TripEnded` | 여행, 종료 방식(AUTO_NEXTDAY/MANUAL D19/Δ4), 종료 시각 — 전체 요약·스타일 분석 트리거 | INV-EXEC1·4 |
| changelog diff(G132 통합 스키마) | 항목 단위: 대상(slotRef), 행위자, 출처 유형(**PlanB** — 후속 공용 enum), 사유, 전/후 값(POI 내부 ID), 발생 시각, 스키마 버전 | INV-REPLAN2, G57/G132 |
| GPS 폴리라인 | 여행·일자, 단순화 폴리라인(원시 파기 후 G55/G73), consentRef | INV-GPS3·4·5 |

**검증 방법 — 통합 테스트 시나리오(CP4)**
1. **actual 생산 파이프라인**: 방문 완료 체크(U6) → `VisitChecked` 발행 → U7 actual 방문 기록 생성 → plan/actual/changelog 3계열 대조(US-E8-04) 정확 표시.
2. **회고 트리거 연쇄**: `TripEnded`(자동·수동 각각) 수신 시 전체 요약 생성 기동, LLM 실패 시 기본 카드 폴백까지 도달(침묵 실패 금지).
3. **changelog 재생 동등성**: U6 재계획 생산 changelog diff 시퀀스를 U7이 순서대로 재생한 결과가 current 스냅샷과 일치(U7 PBT 1급 속성을 U6 실생산 데이터로 통합 검증).

> **계약 변경 통제**: changelog 통합 스키마(G132)는 후속 3유닛(U9·U10·U11) 공용 — 스키마 버전 필드로 전방 호환(CP4 §167).

---

## 12. C1·C2 재사용 계약 (U5 자산 — 계약 수준)

U6은 U5가 프로덕션 품질로 완성한 **C1 LLM Gateway·C2 Solver Engine을 재사용**한다(중복 재구현 금지, FD-U6-03).

| 자산 | U6 재사용 용도 | 재사용 계약(입출력) | 하드 제약 재사용 |
|---|---|---|---|
| C1 LLM Gateway | 재계획 **사유 해석·후보 정렬 힌트**(날씨→실내·체력→축소) | `C1.call(reason, sessionContext)` 경량 티어(D11) · 서버 재조회 컨텍스트 주입(D31) · 타임아웃 1.5초 규칙 매핑 폴백 | closed-set 그라운딩(G115)·계정 무결성(D31·SECURITY-11) = U5 INV-SCORE1·4 재사용 |
| C2 Solver Engine | 후보 검증·재정렬·**확정 시점 재검증**(G56) | `C2.solve/validate`(warm-start·당일 잔여) · `C2.estimateTravel`(거리만 반환·internalDuration 내부) | **하드 제약 4계열(HC1~HC4) = U5-P1 그대로 재사용**(생성과 동일 검증, 재구현 0) [DoD, U5-P1] |
| ConstraintSpec(C2 D28) | (수동 편집 폴백) 클라 경량 검증 | U5 shared/validation 재사용 | U5 INV-SOLVE1 재사용 |

> **재사용 원칙(FD-U6-03)**: U6은 C1·C2의 **계약 인터페이스만 소비**하고 도메인 로직(최적화·하드 제약 검증·closed-set)을 재구현하지 않는다 — "재계획 결과도 생성과 동일한 하드 제약 4계열 검증 100%"(U6 DoD)는 **U5 C2·U5-P1 재사용**으로 달성한다(PBT U6-P11).

---

## 13. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| TripExecutionState | US-E7-01, US-E8-08 | D19/Δ4, D21, G54, components §3.1 |
| ActiveTripHub | US-E2-02·03, US-E5-08, US-E6-03 | D21, D24/Δ6, D25, G3 |
| VisitState / ArrivalPromptLog | US-E6-01, US-E8-01 | D23, ADR-0010, components §3.3, CP4 |
| RestMode | US-E7-06 | G54, G159, G100 |
| TriggerEvent / SuppressionState / SensitivitySetting | US-E7-02 | D27, D10, D34, G58/G195, G116 |
| ReplanSession | US-E7-01·03·10·11·12 | D19, D38, C10, D14, G53, G56, ADR-0006 |
| Alternative / UnplacedList | US-E7-04·05·07·08 | G53, D25/Δ1, C10, G115 |
| WeatherData | US-E7-02 | D10, G195, RESILIENCY-09 |
| GpsTrail | US-E7-13, US-E8-03 | G55/G73, D34/N2, G59, G62, CP4 |
