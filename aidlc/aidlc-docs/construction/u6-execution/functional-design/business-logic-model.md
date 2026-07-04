# U6 여행 중 실행·Plan-B — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 5종 + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U6-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) M18·M9·M10·M11, 서비스 정본 [services.md](../../../inception/application-design/services.md) S2(Plan-B 재계획)·S4(여행 종료→회고)
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 전송·저장·지오펜스 SDK·기상청 어댑터·티어·remote config 구현체는 NFR/Infra 단계 소유.
> **재사용 원칙(FD-U6-03)**: 재계획 후보 검증·확정 재검증은 U5 C2(하드 제약 4계열·U5-P1)를 그대로 재사용하고, 사유 해석은 U5 C1을 재사용한다 — U6은 오케스트레이션·상태 머신·트리거 억제·GPS 수집만 신규.

---

## FLOW-1 도착 → 방문 체크 → 체류 측정 (CP4 생산)

**진입**: 지오펜스 진입 감지 ∨ 수동 도착 탭 → `M18.onGeofenceEnter` / `M18.confirmArrival` → `M18.completeVisit`
**관련**: US-E6-01, US-E8-01 · BR-U6-01~05 · D23, ADR-0010, G62, CP4

```text
S1 도착 감지 [BR-U6-01]
   포그라운드 지오펜스 진입 → M18.onGeofenceEnter → '도착하셨나요?' 프롬프트만(UPCOMING→ARRIVALPENDING)
   가드: 위치 3층 동의 충족·GPS 정확도 충분(BR-U6-25 매트릭스) — 자동 기록 없음(INV-VISIT2)
   ├─ E1 위치 권한 없음·저정확도 → 프롬프트 미생성, 수동 체크인 대체(ADR-0010) [BR-U6-02]
   └─ B1 외부 지도앱 복귀 시 다음 예정지 근접 → 도착 프롬프트(handleExternalMapReturn) [BR-U6-30·31]
S2 도착 확정 — M18.confirmArrival(via=Prompt|Manual) [BR-U6-02]
   ARRIVALPENDING→INPROGRESS(∨ 수동 UPCOMING→INPROGRESS) · arrivedAt 기록(기기 시각·수정 가능)
   → VisitChecked(start) 발행(CP4 — M12 actual)
   └─ B2 프롬프트 닫기/무시 → ARRIVALPENDING→UPCOMING, promptSuppressed=true(재프롬프트 억제) [BR-U6-04]
S3 방문 완료 — M18.completeVisit [BR-U6-03]
   INPROGRESS→COMPLETED · departedAt = (a) '방문 완료' 탭 시각(실측) ∨ (b) 다음 장소 도착 체크 시각(추정·departedEstimatedFlag=true)
   actualStay = departedAt − arrivedAt(결정적 산출, INV-VISIT3) → M12 보관(plan 체류 대조)
   → VisitChecked(complete) 발행(CP4) · 체류 초과분(overstayGap) → M9 트리거 (d) 신호
   └─ B3 스킵/취소 → any→SKIPPED, 잔여 영향 시 Plan-B 제안 연결(INV-TRIG3) [BR-U6-05]
S4 진행 상태 반영 [BR-U6-08]
   활성 허브 진행 상태(완료/진행 중/다음) 갱신 · progress(완료+스킵 비율 G3) → 홈 카드·목록
```

**사후 조건**: 방문 상태는 §3.3 전이표만 따르고(INV-VISIT1) 자동 확정 0(INV-VISIT2). actualStay 결정적(INV-VISIT3). VisitChecked·체류 신호가 CP4·M9로 정확히 공급된다.

---

## FLOW-2 자동 트리거 감지 → 억제 → 알림

**진입**: 서버 배치 폴링(날씨·휴무) ∨ 클라 포그라운드 신호(지연·체류) → `M9.evaluateServerTriggers` / `M9.reportClientSignal`
**관련**: US-E7-02 · BR-U6-09~14 · D27, D10, D34, G58/G195, G116 · services S2.1

```text
S1 감지 (하이브리드, 순수 함수 판정) [BR-U6-09·10·11]
   서버 경로: J1(날씨 1시간)·J2(휴무 아침) → M9.evaluateServerTriggers
      (a) M11.getPrecipitationRisk 강수 60%↑ ∨ getActiveAlerts 특보(SEVERE), (b) M7 휴무·영업시간 변경
   클라 경로(포그라운드 D27): (c) 이동 지연(15분 G106)·(d) 체류 초과(20분·고정 일정 위협) → M9.reportClientSignal
   판정 = clock·외부 데이터 주입 순수 함수(G116, INV-TRIG4)
   ├─ E1 기상청·영업시간 무응답(DataUnavailable) → 트리거 침묵(허위 알림 금지 INV-TRIG6·INV-WX1) + 실패율 계측 [BR-U6-14]
   └─ E2 위치 동의 철회 → (c·d) 미평가, (a·b) 위치 비의존만 지속(INV-TRIG5) [BR-U6-25]
S2 억제 필터 [BR-U6-12·13]
   전역 상한(시간당 2/일 8·±50% 민감도), 동일 {사유·방문지} 1회, 초과·동시 발화 묶음 1회(INV-TRIG1)
   무시 2회 연속 → 당일 동일 사유 억제(학습·멱등 INV-TRIG2) · 휴식 모드 경미 억제(심각 유지 INV-REST1)
   ├─ B1 상한 초과·당일 억제·휴식 경미 → SUPPRESSED(발화 안 함·계측)
   └─ 통과 → DETECTED→FIRED
S3 발화·알림 — outbox TriggerFired(CP5) → M14 [BR-U6-13]
   심각(기상특보·고정 일정 위협)=푸시 / 경미=인앱 칩 · 출처·감지 시각 표기 · 방해금지 예외(진행 중 Plan-B severe G100)
S4 노출·수락 [트리거 수명 §5.4]
   FIRED→SHOWN(배너/칩) → 사용자 '대안 보기' → ACCEPTED → M10 재계획 세션(FLOW-3) — 자동 변경 없음(INV-TRIG3)
   └─ B2 닫기/무시 → DISMISSED → (2회) SUPPRESSED
```

**사후 조건**: 발화 수는 상한 불변식 준수(INV-TRIG1), 억제 멱등(INV-TRIG2), 판정 결정적(INV-TRIG4). 데이터 없음은 침묵(허위 알림 0). 트리거는 제안만(current 미변경).

---

## FLOW-3 재계획 세션 (사유 → 후보 C1/C2 재사용 → 검증 → 확정 → current 갱신 → changelog)

**진입**: 수동 '재계획' ∨ 자동 트리거 '대안 보기' → `M10.startReplan` (정본 서비스 S2)
**관련**: US-E7-01·03·04·05·07·08·09·10·11·12 · BR-U6-15~22 · D14, D19, D25, D38, G53, G56, C10, ADR-0006 · services S2.2·S2.4

```text
S1 세션 시작 — M10.startReplan(tripId, reason?, method, position?) [BR-U6-15]
   가드: '여행 중'(날짜 구간 D19)·숙소 0개 허용(INV-REPLAN1) — 위반 시 PreconditionFailed(NOT_ACTIVE)
   방식 분기: method=ManualEdit → S6 직행 / DelegateToAi → S2
   ├─ E1 GPS 불가 → positionRequired → M10.setManualPosition(핀·검색·추정 출발지 표기 INV-REPLAN7) [BR-U6-26]
   │     └─ 생략 → 최후 완료 방문지 ∨ 등록 숙소 기준(가정 명시)
S2 영향 분석 입력 수집 (1.0초 예산) [BR-U6-16]
   현재 위치·시각, 남은 일정(완료 방문 제외 — 현재 시각 이후만 INV-REPLAN4), 고정 제약(숙소 불변 ADR-0006·시각 고정), 당일 영업시간(M7), C2.estimateTravel(거리만 D25)
S3 C1 사유 해석 (1.5초 타임아웃) [BR-U6-17] — U5 C1 재사용
   C1.call(reason, sessionContext) 경량 티어 — 날씨→실내·체력→축소 정렬 힌트
   └─ E2 타임아웃·오류 → 규칙 기반 사유-카테고리 매핑 폴백(설명 축약)
S4 후보 소싱 (1.5초) [BR-U6-17]
   M7 저장 장소 우선 → 부족 시 주변 신규 확장(G53) · closed-set 그라운딩(G115, U5 INV-SLOT1 재사용)
S5 솔버 검증·재정렬 (4.5초·후보당 1.5초) [BR-U6-17] — U5 C2 재사용
   C2.solve/validate 후보별 하드 제약 4계열(HC1~HC4) 검증(U5-P1 재사용), warm-start(완료·시각 고정 보존), 당일 잔여만(C10)
   → Alternative 2~3개(솔버 통과분만 INV-ALT1), 사유 부합 정렬(INV-ALT3)
   ├─ E3 후보 0건/전부 실패/10초 초과 → Fallback 3종(1개 건너뛰기/휴식 전환/수동 수정)+사유 한 줄(INV-ALT4) [BR-U6-18]
   └─ E4 외부 API 오류 → S6 수동 수정 폴백(누락 데이터 표기·숙소 고정 위반 차단) [BR-U6-19]
S6 (분기) 수동 편집 — M10.applyManualEdit [BR-U6-19]
   순서 재배열·삭제·시각 직접 입력 · 숙소 고정 제약 위반 차단(입력 차단) · API 복구 시 자동 검증 재활성
S7 대안 표시·전후 비교 (0.5초) — M10.previewAlternative [BR-U6-20]
   후보별 추천 이유·거리·수단·체류·다음 고정 제약까지 여유(소요시간 없음 D25/INV-ALT2)
   전/후 diff(추가/삭제/시간 이동)·지표 요약(총 이동 거리·방문지 수·숙소 복귀 시각 증감)
   이월·제외 방문지 확정 전 명시 동의(INV-ALT5) [BR-U6-22]
S8 확정 — M10.confirmAlternative (별도 상호작용·1.0초) [BR-U6-21]
   확정 시점 C2 재검증 1회(G56, INV-REPLAN3)
   ├─ E5 재검증 무효화 → RevalidationFailed → S5 후보 재산출 안내(확정 차단)
   └─ 통과 → TX: current만 갱신(plan 불변 D14 INV-REPLAN2) + M12.appendChangeLog(사유·전후·트리거 유형 G132) + outbox ItineraryChanged
S9 사후 [BR-U6-21]
   M14 리마인드 재계산(D32) · 이월분 UnplacedList(사용자 날짜 선택 시 rescheduleUnplaced) [C10]
   └─ B1 '기존 유지'(keepCurrentPlan) → 어떤 변경도 미저장(CANCELLED)
```

**성능 예산(D38 — services S2.4)**: S2~S7 합계 9.0초(여유 1.0초), S8 확정 재검증 1.0초는 10초 예산 외(별도 상호작용). 초과 시 산출 후보만 제시·0건이면 3종 폴백.
**사후 조건**: current만 변경·plan 바이트 동일 불변(INV-REPLAN2), 확정 재검증 통과 없이 반영 0(INV-REPLAN3), 재정렬 범위 당일 잔여만(INV-REPLAN4). changelog diff가 CP4로 U7 공급.

---

## FLOW-4 여행 종료 전이 (자동/수동 D19 · DayClosed)

**진입**: J3 배치(종료일 익일 00:00 ∨ 일자 경계) ∨ '여행 종료' 수동 버튼 → `M18.autoEndTrips` / `M18.endTripManually`
**관련**: US-E7-01, US-E8-08 · BR-U6-06·07 · D19/Δ4 · services S4

```text
S1 종료 전이 [BR-U6-06]
   J3 M18.autoEndTrips(종료일 익일 00:00·숙소 무관 단일 규칙) ∨ M18.endTripManually
   TX: 여행 상태 조건부 전이(ACTIVE→ENDED, 이미 ENDED면 no-op — 자동/수동 경합 안전 INV-EXEC1) + outbox TripEnded
   → M13 회고 자동 생성·M14 완료 알림·M12 기록 귀속 마감(CP4)
S2 일자 경계 [BR-U6-07]
   J3 일자 경계 → DayClosed 발행(당일 방문 요약) → M13 당일 회고 초안 트리거 · currentDate 갱신
```

**멱등성**: `TripEnded`·`DayClosed` 핸들러는 (tripId, kind) 기준 중복 수신 안전(INV-EXEC1). 종료 후 기록 편집은 허용하되 회고 갱신은 수동 재생성만(C11).
**사후 조건**: 여행당 `TripEnded` 정확히 1회(PBT U6-P8), 회고 트리거는 종료 전이로만 개시(INV-EXEC4).

---

## FLOW-5 휴식 모드 (진입 → 억제 → 재개 재계산)

**진입**: 재계획 대안 화면 '휴식 전환' ∨ 허브 휴식 버튼 → `M10.enterRestMode`
**관련**: US-E7-06 · BR-U6-23·24 · G54, G159, G100

```text
S1 진입 — M10.enterRestMode(tripId, resumeAt?) [BR-U6-23]
   TripExecutionState phase NORMAL→REST · RestMode{resumeAt?, suppressionScope=MINOR_ONLY}
   경미 트리거·일정 알림 억제 신호(M9·M14) · 심각 사유는 억제 불가(INV-REST1)
   resumeAt을 리마인드 큐(J4) 등록(NULL=즉시 재개 대기)
S2 재개 — M10.resumeFromRest [BR-U6-24]
   재개 시각 도달 알림 ∨ 즉시 재개 → phase REST→NORMAL(억제 해제)
   → ReplanProposal(남은 일정 재계산 제안) — 자동 재계획 아님(사용자 대안 보기 필요 INV-REST2·INV-TRIG3)
```

**사후 조건**: 휴식 중 심각 사유 억제 0(INV-REST1), 재개 시 재계산 제안(자동 변경 없음).

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장, D37 계층 분리). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check. 시드 로깅·수축(shrinking) 필수(PBT-08). **트리거 판정·상태 머신은 실코드 테스트, 외부(기상청·거리 API·LLM)만 어댑터 fake**(D37). 아래 속성은 U6 Code Generation의 DoD 항목이며, **U6 DoD 명시 속성**((a)트리거 판정 순수 함수 상한·억제·민감도·(b)방문 상태 머신 전이·(c)warm-start 고정 블록 보존·(d)휴식 억제 규칙·(e)폴리라인 단순화)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**M18 Trip Execution (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P1 | 방문 상태 머신 (stateful) | **방문 전이 가드**: 임의 (상태·트리거) 시퀀스에 대해 (a) §3.3 전이표 조합만 성공, (b) 표 밖 전이(COMPLETED→INPROGRESS 역행·UPCOMING→COMPLETED 직행·SKIPPED 재전이) 전부 거부(INV-VISIT1), (c) **지오펜스 진입은 프롬프트만 — 어떤 상태도 자동 확정 안 함**(INV-VISIT2), (d) 수동 도착은 위치 무관 항상 가능(ADR-0010) | 상태·트리거 시퀀스 생성기(유효/무효 혼합) + 위치 동의·GPS 정확도 주입 |
| U6-P2 | 체류 측정 결정성 | **체류 산출 결정성**: actualStay = departedAt − arrivedAt이 (a) 동일 (arrivedAt, departedAt)에 동일 산출(결정적·clock 주입), (b) departedAt이 다음 장소 체크 추정이면 departedEstimatedFlag=true 전파(허위 정확성 0), (c) arrivedAt 부재 시 미산출(NULL) | 방문 시각 쌍 생성기(실측/추정 혼합) + 다음 장소 체크 시각 주입 |
| U6-P8 | 여행 종료 전이 단일성 | **종료 전이 멱등·단일**: 임의의 (자동 익일 00:00 배치 · 수동 버튼) 발화 순서·중복에 대해 `TripEnded`가 여행당 **정확히 1회** 발행되고(조건부 전이·이미 ENDED면 no-op, INV-EXEC1), endedAt·endMethod는 첫 전이에서만 확정 — 숙소 유무 무관 단일 규칙 | 종료 트리거 시퀀스 생성기(자동/수동 경합·중복·순서 셔플) + 숙소 유무 주입 |

**M9 Plan-B Detection (서버 — Kotest · 순수 함수 시뮬레이션)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P3 | 트리거 상한·억제 멱등 (순수 함수) | **상한 불변식·억제 멱등**: 임의 clock·트리거 시퀀스에 대해 (a) FIRED 발화 수가 **전역 상한(시간당 2/일 8·민감도 ±50%)을 초과하지 않음**(INV-TRIG1), (b) 동일 {type, slotRef} 1회만 SHOWN, (c) 동일 사유 2회 무시 후 당일 동일 사유 억제(멱등 — 반복 판정에 재발화 0, INV-TRIG2), (d) **휴식 모드 경미 억제·심각 사유 억제 불가**(INV-REST1) | clock·트리거 시퀀스 생성기(사유·방문지·시각 분포) + 무시 이벤트 + 휴식 상태·심각도 주입 |
| U6-P4 | 트리거 판정 결정성·민감도 단조 | **판정 순수 함수**: (a) 동일 (clock·날씨·체류·이동) 입력에 동일 판정(결정적·주입, INV-TRIG4), (b) 민감도 상향(적게→많이) 시 발화 임계 완화로 발화 집합이 단조 확대(비축소), (c) 위치 동의 철회 시 (c·d) 트리거 미평가·(a·b) 유지(INV-TRIG5) | 판정 입력 생성기(임계 경계 분포) + 민감도 레벨 시퀀스 + 위치 동의 상태 주입 |

**M10 Itinerary Recalculation (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P5 | plan 불변·current만 변경 (하드 계열) | **재계획 격리**: 확정된 INTRIP 일정에 임의의 재계획 연산 시퀀스(사유·후보·확정·기존 유지)를 적용해도 (a) U5 PlanSnapshot.frozenDays가 **바이트 동일**하게 유지되고(plan 변경 0, INV-REPLAN2 · U5 INV-PLAN1 소비), (b) 변경은 current에만 반영, (c) '기존 유지'·CANCELLED는 current도 불변 | INTRIP 일정 + 재계획 연산 시퀀스 생성기(확정/취소/기존유지 혼합) |
| U6-P6 | 당일 잔여 재정렬 범위 불변식 (C10) | **재정렬 범위**: 임의 재계획에 대해 (a) 재배치 대상은 **현재 시각 이후 당일 잔여 방문지만**(완료·시각 고정·숙소 고정 제외, INV-REPLAN4), (b) 타 일자 슬롯 불변, (c) 넣지 못한 방문지는 UnplacedList로 이월(침묵 드롭 0)·확정 전 명시(INV-ALT5) | 일정 생성기(완료/잔여/고정 혼합·다일자) + 현재 시각·재계획 사유 주입 |
| U6-P7 | 확정 재검증 게이트 (G56) | **확정 재검증**: 확정 버튼 시점 C2 재검증(G56)에 대해 (a) 재검증 무효화 시 CONFIRMED 전이 거부·CANDIDATES 복귀(반영 0, INV-REPLAN3), (b) 통과 후보만 current 반영, (c) 재정렬 결과가 하드 제약 4계열 통과(U5-P1 재사용) | 후보 + 확정 시점 상황 변화(영업 종료·시간 경과) 주입 |

**M12 계약 — GpsTrail (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P9 | 폴리라인 단순화 라운드트립 | **GPS 폴리라인**: 임의 원시 좌표열에 대해 (a) 단순화 폴리라인이 **경로 위상(방문 순서·구간 연결) 보존**(INV-GPS3), (b) 단순화→직렬화→역직렬화 라운드트립에서 위상 동등, (c) **원시 좌표 파기 후 재구성 불가**(파기 검증)·표시·거리 산출엔 충분, (d) 옵트인 미동의 입력은 수집 거부(INV-GPS1) | 원시 좌표열 생성기(밀도·노이즈 분포) + 옵트인 동의 상태 주입 |

**M11 Weather & Context (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P10 | 격자 변환 결정성 | **격자 변환**: 위경도→기상청 격자 변환이 (a) 동일 좌표 동일 격자(결정적 순수 함수, INV-WX2), (b) API 실패 시 `DataUnavailable` 명시('맑음' 위장 0 — INV-WX1·INV-TRIG6 입력) | 좌표 생성기(국내 범위 분포) + API 성공/실패 주입 |

**C2 재사용 검증 (서버 — U5 자산·계약 테스트)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U6-P11 | 하드 제약 재사용 (하드 제약 재사용·U5-P1) | **재계획 하드 제약 = 생성 하드 제약**: 재계획 후보·확정 재정렬 결과가 **U5 C2 하드 제약 4계열(HC1 영업시간·HC2 이동 부등식·HC3 고정 블록·HC4 시간창)을 그대로 재사용 검증**해 위반 배치가 결과에 부재(INV-ALT1). **U5-P1(oracle 포함)을 재계획 입력으로 재실행** — U6은 하드 제약을 재구현하지 않고 U5 C2를 호출(재구현 0 확인) | 재계획 문제 인스턴스 생성기(당일 잔여·고정 블록·이동 파라미터) → U5 C2.solve/validate 재호출 |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| M18 허브 조립(getActiveHub·getPlaceContext) | No PBT properties identified | 읽기 모델 집약·표시 DTO — 소요시간 미표시는 U5-P4(표시 게이트) 재사용, 진행률은 카운트 집계. 예시 기반+온라인 전제 오류 처리 통합 테스트 |
| M18 외부 지도 핸드오프(buildExternalMapHandoff·handleExternalMapReturn) | No PBT properties identified | 좌표 페이로드 조립·복귀 라우팅 — 폴백 계단(외부 앱 없음→웹→거리 요약)은 예시 기반 |
| M10 오케스트레이션(startReplan 시퀀스·10초 예산) | No PBT properties identified | 외부 모듈 조립·TX 경계·타임박스 — 보편 양화 도메인 로직은 하위 C2(U6-P11)·C1(U5 재사용)·상태 머신(U6-P5~7)으로 분해. 폴백 계단·성능 예산은 예시 기반 통합 테스트+어댑터 fake |
| M10 위치 폴백 체인(setManualPosition) | No PBT properties identified | 폴백 우선순위(GPS→수동→최후 완료→숙소)·가정 표기 — 3층 매트릭스(BR-U6-25)의 테이블 주도 테스트 |
| M11 캐시·쿼터 보호 | No PBT properties identified | TTL·서킷 브레이커 정책(수치 NFR) — 결정성(U6-P10)과 달리 정책 테이블. 예시 기반+계약 테스트 |

### 커버리지 대조 (U6 DoD → 속성)

| U6 DoD 명시 속성 | 대응 |
|---|---|
| (a) 트리거 판정 순수 함수(상한 초과 발화 0·무시 2회 억제·민감도 단조) | U6-P3 · U6-P4 |
| (b) 방문 상태 머신(예정→도착→시작→완료/스킵) 전이 불변식 | U6-P1 |
| (c) warm-start 재계획 고정 블록 보존 | U6-P7 · U6-P11(C2 재사용) |
| (d) 휴식 모드 억제 규칙(심각 사유 억제 불가) | U6-P3(d) |
| (e) 폴리라인 단순화(원시 파기 후 재구성 불가·위상 보존) | U6-P9 |
| (설계 확장) 체류 측정 결정성 | U6-P2 |
| (설계 확장) 재계획 plan 불변·current만 변경 | U6-P5 |
| (설계 확장) 당일 잔여 재정렬 범위 불변식(C10) | U6-P6 |
| (설계 확장) 여행 종료 전이 단일성(D19) | U6-P8 |
| (설계 확장) 격자 변환 결정성·데이터 없음 명시 | U6-P10 |
| (설계 확장) 재계획 하드 제약 = U5 재사용 | U6-P11 |

**속성 합계: 11개** (M18: 3 · M9: 2 · M10: 3 · M12(GPS): 1 · M11: 1 · C2 재사용: 1)

### 하드 제약(D37) 대조 — U6은 재사용·경계 방어

| 하드 제약 | U6 방어 속성·불변식 |
|---|---|
| **충돌 무배치**(재계획 결과도 영업시간·이동 부등식·시간창 위반 0) | **U6-P11(U5-P1 재사용·oracle)** · INV-ALT1 · BR-U6-17·21 |
| **숙소 기준점**(숙소 고정 제약 불가침 — 예약 미변경 ADR-0006) | INV-REPLAN5 · BR-U6-16·19 · U6-P6·P7 |
| **POI 그라운딩**(재계획 후보 ⊆ closed-set) | U5 INV-SLOT1 재사용 · INV-ALT1 · BR-U6-17 |
| **계정 무결성**(C1 재조회 컨텍스트 — 재계획 사유 해석) | U5 INV-SCORE4 재사용(SECURITY-11) · §12 재사용 계약 |
| **회고 대조 무결성**(재계획 후 plan 불변·current만) | U6-P5 · INV-REPLAN2 · U5 INV-PLAN1 소비 |

> **C2 재사용 명시(FD-U6-03)**: U6은 하드 제약 검증을 재구현하지 않는다 — 재계획 후보·확정 재검증은 **U5 C2.solve/validate를 호출**하고, PBT는 U5-P1(무차별 대입 oracle 포함)을 재계획 입력으로 재실행해 "재계획 하드 제약 = 생성 하드 제약"을 통합 검증한다(U6-P11). 이로써 "재계획 결과도 생성과 동일한 하드 제약 4계열 검증 100%"(U6 DoD)가 이중 개발 없이 성립한다.
