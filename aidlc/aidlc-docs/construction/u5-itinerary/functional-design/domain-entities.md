# U5 AI 일정 생성·확정 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M8 Itinerary Generation, C1 LLM Gateway(계약 수준), C2 Solver Engine(계약 수준)
> 정본 관계: [components.md](../../../inception/application-design/components.md) M8·C1·C2 섹션 + **§3.2 일정(Itinerary) 상태 머신 정본**의 엔티티 개요를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U5 엔티티·불변식·상태 전이의 정본이다. 기술 중립 — 저장 기술·컬럼 타입·인덱스·LLM 벤더·티어·최적화 알고리즘(OPTW/TOPTW)·remote config 수치(G106)는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL의 의미를 반드시 명기), `유니크`=제약 범위 내 유일, `불변`=생성 후 변경 불가, `파생`=저장 안 함(조회 시점 계산). 근거 ID는 requirements.md(D·Δ·G·ADR)·stories.md(US-E5)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U5-xx)을 참조한다.
> **U5는 CP2(U4→U5)의 소비자이자 CP3(U5→U6)의 공급자다** — 여행 컨텍스트(TripContext)는 U4([u4-trip/functional-design/domain-entities.md](../../u4-trip/functional-design/domain-entities.md) §7)가 소유하는 입력이며(§9 참조 계약), 본 유닛의 일정 기준선(plan/current·고정 블록·확정 상태)이 U6·U7·U9·U11의 공유 척추다(CP3 §10).
> **C1·C2는 재사용 자산**(U6 재계획·U7 회고·U9 어시스턴트·U11 재검증) — 본 문서는 계약 인터페이스(입출력 타입·불변식·실패 타입)까지만 정의하고 구현체는 NFR/Infra로 이관한다(FD-U5-12).

## 0. 엔티티 지도

```text
[M8 Itinerary Generation — 일정 상태 머신·plan/current 이원 정본]
Trip 1 ──── 1 Itinerary                 (여행당 활성 일정 ≤1, G46/G136 — 상태 머신 DRAFT→EDITING→CONFIRMED→INTRIP)
Itinerary 1 ──── 0..1 PlanSnapshot      (확정 시점 동결 불변본, D14 — CONFIRMED 이후 존재)
Itinerary 1 ──── 1 CurrentItinerary     (가변 현재본 — 편집·Plan-B 반영 대상, D14)
CurrentItinerary 1 ──── * DaySchedule   (일자별 일정 — 적용 거점·시간창 스냅샷)
DaySchedule 1 ──── * ItinerarySlot      (슬롯 — POI·시각·체류·LOCK·sourceType·violations)
DaySchedule 1 ──── * FixedBlock         (고정 블록 — 숙소 체크인/아웃·시각 고정형 필수 방문지, 불가침)
Itinerary 1 ──── * GenerationSession    (생성 세션 — COLLECTING→DAY1_READY→GENERATED→FAILED·부분 초안)
(독립) StayZoneRecommendation           (숙소 나중 등록 온램프 — 동선 무게중심, US-E5-11)

[C2 Solver Engine — 상태 비보유 순수 계산 엔진(계약 수준)]
SolverProblem  ──(solve)──▶ SolverSolution   (하드 제약 4종 만족 해 · 위반 배치 구조적 배제)
TravelEstimate                          (거리 기반 이동 추정 — internalDuration 내부전용 D25)
ConstraintSpec                          (버전 있는 검증 규칙 명세 — 클라 경량 검증기 공유 D28)

[C1 LLM Gateway — 상태 비보유 관문(계약 수준)]
PoiScore                                (LLM 선호 점수 — closed-set 후보 ID에만 부여 G115)

[CP2 입력 — U4 소유(본 유닛은 참조·소비만, §9)]
TripContext = Trip + TripDateWindow[] + BaseAssignment[] + MustVisit[] + 여행 속성

[CP3 출력 — U6 M18·M9·M10 소비(§10)]
Itinerary(plan/current·상태) + DaySchedule[] + ItinerarySlot[](slotId·LOCK·고정 블록·DistanceRange)
```

---

## 1. Itinerary — 일정 (M8, 상태 머신·plan/current 이원 정본)

여행 1건에 대한 AI 일정의 정본 컨테이너이자 **일정 상태 머신(§1.3)·plan/current 이원 구조(D14)의 정본 소유자**. 여행당 활성 일정은 항상 1개(G46/G136)이며, 이 엔티티의 상태·plan 불변성이 CP3(일정 기준선) 계약과 U7 회고 대조·U10 공개 스냅샷의 전제다.

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| itineraryId | 식별자 | 필수 · 유니크 · 불변 | **일정 정본 키** — DaySchedule·GenerationSession이 참조. U6 실행 허브·U7 기록의 일정 참조 단일 키 |
| tripId | 식별자(Trip 참조) | 필수 · 유니크 | 귀속 여행(엔티티 정본은 U4 M6). **여행당 활성 일정 ≤1**(INV-ITIN1) [G46/G136] |
| mode | 열거 {FULL_AUTO, TOGETHER, MANUAL} | 필수 | 생성 방식(§1.5) — 완전 AI 자동 / 같이 고르기 / 직접 만들기. 도중 전환 시 최종 mode 기록(진행분 보존) [G48, PRD 06-10] |
| status | 열거 {DRAFT, EDITING, CONFIRMED, INTRIP} | 필수 · 기본 `DRAFT` | 일정 생애 상태(§1.3 상태 머신 정본). 전이 트리거는 사용자 액션(확정·해제)·타 유닛 이벤트(INTRIP=M18 여행 ACTIVE) 수용 |
| version | 정수(단조 증가) | 필수 · 기본 1 | 재생성·확정 왕복 추적 버전. 이전 상태는 changelog(M12)로 추적(G136) — 값 자체는 낙관적 동시성·U11 항목 잠금(D30) 확장 여지 |
| currentRef | 식별자(CurrentItinerary 참조) | 필수 | 가변 현재본 참조(§3) — 항상 존재(생성 시점부터) |
| planRef | 식별자(PlanSnapshot 참조) | 선택 | 확정 시점 동결본 참조(§2). NULL=미확정(DRAFT/EDITING) → CONFIRMED 전이 시 생성(INV-ITIN2) [D14] |
| confirmedAt | 시각(UTC) | 선택 | 최초/재확정 시각. NULL=미확정. 해제 후 재확정 시 갱신 [D20] |
| createdAt | 시각(UTC) | 필수 · 불변 | 생성 일시 |

### 1.2 파생 표시값 (저장 안 함)

| 파생값 | 계산 | 근거 |
|---|---|---|
| isConfirmed | `status ∈ {CONFIRMED, INTRIP} ∧ planRef ≠ NULL` | 확정본 열람·공개 가능 게이트(U10 소비) [D14, D20] |
| planCurrentDiverged | `status = INTRIP ∧ plan ≠ current`(내용 비교) | 회고 대조(U7) 표시 신호 — plan/actual/changelog 3계열 [ADR-0013] |
| activeViolationCount | current 전 슬롯 violations 합 | '그대로 저장' 위반 지속 가시화(BR-U5-27) [PRD 06-7] |

### 1.3 일정 상태 머신 (정본 — components.md §3.2 상세화)

```text
        [generateItinerary — 3방식 공통]
              │
              ▼
          DRAFT ──(userStartsEdit)──▶ EDITING
            │  ▲                         │
            │  └──(unlockForEdit, D20)───┤ (여행 시작 전)
            │        ◀── CONFIRMED ──────┘
            │                 ▲
            │ (confirmItinerary — C2 확정 검증 통과)
            └─────────────────┤
                              │
                    CONFIRMED ──(여행 ACTIVE 전이 · M18 tripBecomesActive)──▶ INTRIP
                                                                                │
                                                              (여행 ENDED · M18)▼
                                                                            (종결)
```

| 전이 | 트리거 | 가드 조건 | 효과 |
|---|---|---|---|
| (없음) → DRAFT | `generateItinerary`(3방식 공통) | 등록 숙소 존재 ∨ 숙소 나중 등록 온램프(NO_BASE 예외, PRD 06-1·11) · 체크인/아웃 날짜 정합·거점 좌표 확정(INV-ITIN3) | 생성 세션 개설(§6). 취소 시 부분 일정을 초안으로 보존 + '이어서 생성' 진입점(G161) |
| DRAFT → EDITING | 사용자 편집 진입(추가·삭제·재정렬·시간 조정) | — | 매 수정마다 클라이언트 경량 검증(D28) + 위반 배지(차단 아님, PRD 06-7) |
| DRAFT/EDITING → CONFIRMED | 사용자 '일정 확정' | **서버 확정 검증(C2 `validate`) 통과** · 위반 잔존 시 'AI 자동 보정 / 그대로 저장' 선택 완료(PRD 06-7 예외) | **PlanSnapshot 동결(D14)** · `ItineraryConfirmed` → M6 여행 CONFIRMED 전이·M14 리마인드 재계산(CP5) |
| CONFIRMED → EDITING | `unlockForEdit`(D20) | 여행 시작 전(status≠INTRIP) | '편집 중' 표시 · 저장 후 재확정 필요 · 기존 공개본(D16 스냅샷)은 유지, 재확정 전 신규 공개 불가 |
| CONFIRMED → INTRIP | 여행 ACTIVE 전이(M18 `tripBecomesActive`) | — | 이후 모든 편집·Plan-B 결과는 **current에만** 반영(D14) · plan은 회고 대조용 불변 유지 |
| any → DRAFT(재생성) | 사용자 '다시 생성'(`regenerate`) | LOCK 슬롯·체류 고정값·수동 추가 POI·시각 고정 필수 방문지는 고정 블록으로 보존(warm-start, G46/G136) | 여행당 활성 일정 1개, 이전 상태는 changelog로 추적, version++ |
| INTRIP → (종결) | 여행 ENDED(M18 `tripEnded`) | — | 편집 종료 — 이후 기록·회고는 U7 소유(plan/actual 대조 마감) |

### 1.4 불변식 (Invariants)

- **INV-ITIN1 (활성 일정 ≤1)**: 한 tripId에 활성(비파기) Itinerary는 정확히 1개 — 재생성은 새 일정 생성이 아니라 동일 일정의 version++·current 교체(이전 상태는 changelog 추적) [G46/G136].
- **INV-ITIN2 (plan 존재 게이트)**: `planRef ≠ NULL ⇔ status가 한 번이라도 CONFIRMED에 도달`. DRAFT/EDITING(최초 확정 전)에는 planRef=NULL. CONFIRMED 전이 시 PlanSnapshot 생성·동결, 이후 해제(EDITING 복귀)해도 기존 plan은 파기되지 않고 유지(재확정 전 신규 공개만 차단) [D14, D20].
- **INV-ITIN3 (생성 전제 무결성 — 하드 제약 입력)**: DRAFT 진입은 (등록 숙소 존재 ∨ 나중 등록 온램프) ∧ 각 거점 좌표 확정(coordConfirmed=true) ∧ 체크인/아웃 날짜 정합을 전제한다 — U3 INV-STAY2·U4 CP2 무결성의 소비 측 이중 방어(계정 무결성·숙소 기준점 하드 제약) [PRD 06-1 예외, CP2 시나리오 3].
- **INV-ITIN4 (상태 전이 가드)**: status는 §1.3 전이표의 (트리거·가드) 조합으로만 변경된다 — 표 밖 전이(예: INTRIP→CONFIRMED 역행, DRAFT→INTRIP 직행)는 구조적 불가. 여행 중(INTRIP)에는 `unlockForEdit`(CONFIRMED→EDITING) 전이가 불가(여행 중 편집은 current에만) [D14, D20, PBT U5-P9].
- **INV-ITIN5 (활성 일정 단일성과 mode)**: mode는 도중 전환(같이 고르기→완전 AI 등)에도 진행분(확정 슬롯)을 고정 블록으로 보존한다 — 전환이 기존 확정 슬롯을 파기하지 않는다 [G48, PRD 06-10].

### 1.5 생성 방식(mode) 요약

| mode | 화면 라벨 | 파이프라인(정본 S1.1) |
|---|---|---|
| FULL_AUTO | '완전 AI 자동' | 전체 일자 일괄 배치(S1 완전 시퀀스) — 초안 슬롯 LOCK·교체 가능 [PRD 06-10] |
| TOGETHER | '같이 고르기' | 슬롯 단위 루프 — 기준점=직전 확정 슬롯(첫 슬롯=등록 숙소 G48), 반경 후보·거리 트레이드오프 [G48] |
| MANUAL | '직접 만들기' | 빈 일정에서 검색·추가 — 추가 시점마다 편집 재검증부(`validateEdit`)만 사용 [PRD 06-10] |

---

## 2. PlanSnapshot — 확정 시점 동결본 (M8, 불변)

'일정 확정' 시점의 **전체 일정 동결 사본**(day·slot·시각·이유 포함). D14의 불변 축 — 확정 후 여행 중 편집·Plan-B 변경은 current에만 반영되고 plan은 **회고 대조(U7)·공개 스냅샷(U10)의 기준**으로 불변 유지된다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| planSnapshotId | 식별자 | 필수 · 유니크 · 불변 | — |
| itineraryId | 식별자(Itinerary 참조) | 필수 | 귀속 일정 |
| frozenDays | 목록<DaySchedule 사본> | 필수 · 불변 | 확정 시점 전체 일정 동결본(day·slot·시각·고정/추천 구분·이유 포함) — 깊은 사본(current와 물리적 독립) [D14] |
| confirmedAt | 시각(UTC) | 필수 · 불변 | 확정(동결) 시각 |
| confirmVersion | 정수 | 필수 · 불변 | 동결 당시 Itinerary.version — 재확정 시 새 스냅샷 생성(이전 스냅샷은 유지/교체 정책은 D16 공개 스냅샷 계약) |

**불변식**
- **INV-PLAN1 (불변 — 하드 제약 계열)**: PlanSnapshot은 생성 후 어떤 경로로도 수정되지 않는다 — 확정 후 임의 편집·재계획·재확정이 **기존 frozenDays를 바이트 동일하게 유지**한다(회고 대조 전제). 재확정은 새 PlanSnapshot 생성이며 기존 것을 변형하지 않는다 [D14, CP3 시나리오 1, PBT U5-P7].
- **INV-PLAN2 (current 역류 차단)**: current의 변경(편집·Plan-B)은 plan에 절대 역류하지 않는다 — plan↔current는 물리적으로 분리된 사본(참조 공유 금지) [D14, PBT U5-P8].
- **INV-PLAN3 (POI 스냅샷 동반)**: 확정 시 배치된 POI는 `M7.snapshotUserConfirmed`(purpose=확정 일정)로 확정 시점 사본이 영구 저장된다 — plan의 POI 참조가 원본 소실(LOST)에도 잔존(U3 INV-SNAP2 연계) [D13, S1.2 9단계].

---

## 3. CurrentItinerary — 가변 현재본 (M8)

편집·Plan-B가 반영되는 **가변 현재본**. DRAFT/EDITING 단계에서는 유일한 일정 표현이며, INTRIP 진입 후에는 plan과 분리되어 여행 중 변경을 흡수한다(D14). 변경은 changelog(M12)를 경유해 추적된다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| currentItineraryId | 식별자 | 필수 · 유니크 | — |
| itineraryId | 식별자(Itinerary 참조) | 필수 · 유니크 | 귀속 일정(1:1) |
| days | 목록<DaySchedule> | 필수 | 일자별 일정(§4) — 편집·재계산 반영 대상 |
| lastEditedAt | 시각(UTC) | 필수 | 최종 변경 시각 |
| pendingSyncFlag | 불리언(파생) | 필수 · 기본 false | 저장 네트워크 오류 시 클라이언트 로컬 임시 보관 상태 — "저장 대기 중" 표기(서버 반영 전) [PRD 06-8] |

**불변식**
- **INV-CUR1 (항상 존재)**: CurrentItinerary는 Itinerary 생성 시점부터 존재한다(1:1) — 확정 여부와 무관하게 가변본은 상시 유지 [D14].
- **INV-CUR2 (INTRIP 격리)**: status=INTRIP에서 current의 변경은 plan을 건드리지 않는다(INV-PLAN2 대응). 여행 중 편집·Plan-B 확정은 current만 갱신 + changelog 기록 [D14, ADR-0013].
- **INV-CUR3 (직렬화 왕복)**: current는 직렬화→역직렬화 왕복에서 day·slot·시각·고정 블록·violations가 동등하게 보존된다(무손실) — plan/current 저장 왕복 무결성 [D14, PBT U5-P10].

---

## 4. DaySchedule — 일자 일정 (M8)

일정의 한 날짜 단위. 적용 거점(구간 거점)·해당 일자 시간창 스냅샷을 보유하며, 슬롯·고정 블록을 담는다. 자정 초과 활동은 시작한 날짜에 논리적으로 귀속(논리적 하루, D29).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| dayScheduleId | 식별자 | 필수 · 유니크 | — |
| date | 날짜 | 필수 · 유니크(일정 내) | 대상 일자 — 여행 기간 내(체크인 당일~체크아웃 당일 각 1개, PRD 06-1) |
| appliedBaseRef | 참조(BaseAssignment/거점) | 선택 | 그날 적용 거점(구간 거점 — 출발·복귀 기준점). NULL=숙소 미등록일(당일치기·이동일 — 동선만 유지, US-E5-11 예외) [G50, G41] |
| timeWindow | 구조체 {availStart, availEnd} | 필수 | 시간창 스냅샷(U4 TripDateWindow 사본 — 기본 09:00~21:00, 첫날 도착·마지막날 출발 반영) [D29, G119] |
| transitionDayFlag | 불리언 | 필수 · 기본 false | 숙소 전환일(출발=A숙소·복귀=B숙소 편도 동선) 여부 [G50] |
| baseModeMessage | 문자열 | 선택 | 폴백·가정 표기(예: "현재 위치 대신 등록 숙소 기준으로 추천했어요"·"기본 모드로 생성"). NULL=표준 생성 [PRD 06-1·9] |

**불변식**
- **INV-DAY1 (시간창 정합)**: DaySchedule의 모든 슬롯·고정 블록 시각은 `[timeWindow.availStart, timeWindow.availEnd]` 내(자정 초과는 시작일 귀속으로 논리 확장) — C2 하드 제약(시간창) [D29, G119, PBT U5-P1].
- **INV-DAY2 (일자 커버리지)**: 여행 기간 각 날짜에 정확히 1개 DaySchedule(체크인~체크아웃 당일) — 미배정 일자 0(U4 커버리지 완전성 INV-BASE-U4-4 소비) [PRD 06-1, CP2].
- **INV-DAY3 (전환일 편도)**: transitionDayFlag=true인 일자는 출발점≠복귀점(A숙소→B숙소) 편도 동선으로 모델링된다(왕복 강제 금지) [G50, PBT U5-P1(고정 블록 하위)].

---

## 5. ItinerarySlot — 슬롯 (M8)

일정의 최소 항목 단위 — 한 POI의 방문 블록(시각·체류·순서). **slotId는 항목 단위 식별의 키**(U11 항목 잠금 D30의 확장 키)이며, 사용자에게 보이는 모든 시각은 C2 검증값만 담는다(LLM 임의 시각 미노출).

### 5.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| slotId | 식별자 | 필수 · 유니크 · 불변 | **항목 단위 식별 키** — LOCK·교체·편집·U11 항목 잠금(D30)의 참조 키 [CP3] |
| dayScheduleId | 식별자(DaySchedule FK) | 필수 | 귀속 일자 |
| poiSnapshotRef | 식별자(PoiSnapshot 참조) | 필수 | 배치 POI(확정 시점 사본 — U3 M7 PoiSnapshot). closed-set 후보 풀 canonical POI에서만 유래(INV-SLOT1) [G115, D13] |
| canonicalPoiId | 식별자(Poi 참조) | 필수 | 원본 참조(대조·소실 판정용) — 그라운딩 무결성 키 [G115] |
| startAt | 시각(당일) | 필수 | **C2 검증 시작 시각** — 솔버 소유·검증값만(INV-SLOT2) [ADR-0008, PRD 06-3] |
| endAt | 시각(당일) | 필수 | **C2 검증 종료 시각** — `endAt = startAt + 배치 체류`(솔버 검증값) [ADR-0008] |
| assignedStay | 정수(분) | 필수 | 배치 체류 시간(권장→최소 범위에서 솔버가 선택한 값 또는 사용자 고정값) [PRD 06-3, G51] |
| stayCompressedFlag | 불리언 | 필수 · 기본 false | 권장 체류를 최소 쪽으로 압축 배치했는지 — true면 "권장 90분→최소 50분으로 줄여 배치" 표시(사용자 모르게 깎임 방지) [PRD 06-3] |
| stayFixedFlag | 불리언 | 필수 · 기본 false | 사용자가 체류 시간을 직접 고정했는지 — true면 범위 대신 고정 제약(재생성 warm-start 보존) [PRD 06-3, G46] |
| locked | 불리언 | 필수 · 기본 false | 슬롯 LOCK(사후 고정) — 재생성 warm-start 보존 대상(INV-SLOT4) [G46, PRD 06-10] |
| sourceType | 열거 {AI_RECOMMEND, USER_ADDED, MUST_VISIT, BASE} | 필수 | 슬롯 출처 — AI 추천 / 사용자 수동 추가 / 필수 방문지 / 숙소 거점. USER_ADDED·MUST_VISIT은 재생성 보존(warm-start) [G46, PRD 06-4] |
| orderNo | 정수 | 필수 | 일자 내 방문 순서(지도형 순서 번호 핀) [US-E5-06] |
| violations | 목록<Violation{kind, detail}> | 필수 · 기본 [] | 편집 위반 배지 — `kind ∈ {TRAVEL_TIME, OPENING_HOURS, FIXED_CONFLICT}`. '그대로 저장' 시 지속 가시화(INV-SLOT3) [PRD 06-7, D28] |
| llmReason | 문자열 | 선택 | LLM 추천 이유(취향 근거 — 표시 전용). NULL=LLM 설명 실패("추천 이유를 불러오지 못했어요") [PRD 06-5, ADR-0011] |
| solverReason | 문자열 | 선택 | 솔버 배치 이유(실제 사용 제약 근거 — "영업시간이 오전만→오전 배치"). NULL=표준 배치 [PRD 06-5] |

> **이동 구간 표시(슬롯 간)**: 연속 두 슬롯(A→B) 사이 표시 데이터는 별도 값 객체 `DistanceRange{distance, transportMode, estimatedFlag}` — **거리·이동 수단·추정 표기만**(소요시간 필드 부재, D25/Δ1). 소요시간은 TravelEstimate.internalDuration(§8)에서 솔버 내부 계산 전용으로만 존재.

### 5.2 불변식

- **INV-SLOT1 (POI 그라운딩 — 하드 제약)**: 모든 슬롯의 canonicalPoiId는 생성 시점 후보 풀(closed-set)에 속한다 — **출력 POI ⊆ 후보 풀**(환각 0). LLM 신뢰 없이 C1 출구 ID 화이트리스트 교차 검증으로 구조적 보장 [G115, D37, U5 DoD, PBT U5-P5].
- **INV-SLOT2 (시각은 C2 검증값 — 하드 제약 계열)**: startAt·endAt·orderNo는 C2 솔버가 소유·검증한 값이다 — LLM이 생성한 임의 시각은 슬롯에 저장·노출되지 않는다. 설명(llmReason)과 실제 시각 불일치 시 검증 시각이 정본 [ADR-0008, PRD 06-3·5, PBT U5-P1].
- **INV-SLOT3 (위반 보존 비파괴)**: '그대로 저장'(SaveAsIs) 시 violations는 자동 변경 없이 보존되고 저장 후 활성 일정 화면에서도 지속 가시화된다 — 위반이 있어도 저장 차단하지 않음(수동 수정 폴백) [PRD 06-7, ADR-0011].
- **INV-SLOT4 (LOCK·출처 재생성 보존)**: `locked=true` ∨ `stayFixedFlag=true` ∨ `sourceType ∈ {USER_ADDED, MUST_VISIT}`인 슬롯은 재생성(warm-start) 전후로 POI·시각·체류가 보존된다 — 나머지만 재배치. 보존은 멱등(반복 재생성에도 고정분 불변) [G46/G136, PBT U5-P2].
- **INV-SLOT5 (이동 표시 소요시간 부재)**: 슬롯 간 표시 데이터(DistanceRange)에는 소요시간 필드가 구조적으로 없다 — internalDuration은 표시 DTO에 매핑 금지 [D25/Δ1, ADR-0009].

---

## 6. FixedBlock — 고정 블록 (M8, 불가침)

**변경 불가 고정 블록** — 등록 숙소의 체크인/아웃 시각과 시각 고정형(FIXED) 필수 방문지. 솔버가 이 블록을 불가침으로 놓고 나머지 추천 POI를 충돌 없이 배치하며, 공간적 앵커(주변 추천 POI가 모이는 기준점) 역할을 겸한다. 재생성 warm-start의 1급 보존 대상.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| fixedBlockId | 식별자 | 필수 · 유니크 | — |
| dayScheduleId | 식별자(DaySchedule FK) | 필수 | 귀속 일자 |
| kind | 열거 {STAY_CHECKIN, STAY_CHECKOUT, MUST_VISIT_FIXED} | 필수 | 고정 블록 종류 — 숙소 체크인/아웃 시각 / 시각 고정형 필수 방문지 [PRD 06-4] |
| refId | 식별자 | 필수 | 원천 참조(SavedStay 또는 MustVisit — U3/U4 소유) |
| poiSnapshotRef | 식별자(PoiSnapshot 참조) | 선택 | 방문지 사본(MUST_VISIT_FIXED 시). NULL=숙소 체크인/아웃(장소는 거점) |
| fixedStart | 시각(당일) | 필수 | 고정 시작 시각(불가침) |
| fixedEnd | 시각(당일) | 선택 | 고정 종료 시각. NULL=체류 기본값으로 파생(카테고리 StayTimeTable) [G51] |
| fixedDate | 날짜 | 필수 | 고정 날짜 |

**불변식**
- **INV-FIX1 (불가침 — 하드 제약)**: 고정 블록의 fixedStart·fixedEnd·refId는 생성·재생성·편집 전후로 불변이다 — 솔버는 고정 블록 시각을 변경할 수 없고, 충돌하는 추천 POI는 자동 배제한다. warm-start 재최적화에서 전후 동일(불변) [PRD 06-4, G46, PBT U5-P1·P2].
- **INV-FIX2 (충돌 시 자동 변경 금지)**: 두 고정 블록이 시간 겹침 또는 이동시간상 불가능하면 솔버는 **자동 변경하지 않고** 충돌 항목을 강조 반환한다 — 어느 쪽을 옮길지 사용자에게 질의(빈 일정·임의 변경 금지) [PRD 06-4 예외].
- **INV-FIX3 (배치 불가 구조화 응답)**: 시각 고정형 필수 방문지를 영업시간·이동시간·기존 고정 블록 제약상 어떤 시간대에도 넣을 수 없으면 빈 일정 대신 불가 사유 + 3안(다른 날짜/시각 고정 해제→ANYTIME 전환/인접 조정) 제안 [PRD 06-4 예외, CP2 시나리오 2].
- **INV-FIX4 (앵커 역할)**: 포함 고정형(ANYTIME) 필수 방문지는 고정 블록이 아니라 필수 노드지만, 주변 추천 POI의 공간적 앵커로 활용된다(§7 SolverProblem.mustInclude) — 포함만 보장·시각/순서는 솔버 위임 [PRD 06-4].

---

## 7. SolverProblem / SolverSolution — 솔버 문제·해 (C2, 계약 수준)

C2 Solver Engine의 입출력 계약(상태 비보유 순수 계산). **C2는 하드 제약 검증·최적화의 단일 소유자**이며, 사용자에게 보이는 모든 시각·순서는 이 엔진이 소유·검증한 값이어야 한다(ADR-0008). 최적화 알고리즘(OPTW/TOPTW) 구현체는 NFR/Infra 소유 — 본 절은 문제 정의·목적함수·하드 제약·해의 불변식만 규정한다.

### 7.1 SolverProblem (입력 계약)

| 요소 | 타입 | 의미·근거 |
|---|---|---|
| days | 목록<DayContext{window, base}> | 일자별 시간창 + 거점(open-ended 전환일 G50) [D29, G119, G50] |
| candidates | CandidatePool | closed-set 후보 풀(canonical POI·영업시간·좌표) + **LLM 선호 점수 보상값**(PoiScore §9) [G115] |
| fixedBlocks | 목록<FixedBlock> | 숙소 체크인/아웃·시각 고정 방문지·LOCK 슬롯·완료 방문지(불가침 §6) [PRD 06-4, G46] |
| mustInclude | 목록<canonicalPoiId> | 포함 고정형(ANYTIME) 필수 방문지 — 목적함수상 필수 노드(반드시 포함) [PRD 06-4] |
| stayDurations | Map<POI, {min, rec, max}> | 체류 범위(권장→최소 유연 변수, 사용자 고정 시 고정 제약) [PRD 06-3, G51] |
| prefs | {pace, transportModes} | 페이스(여유형/빡빡형)·이동 수단 [PRD 06-3, G134] |
| budgetWeight | 소프트 가중치 | 예산 소프트 가중치(하드 제약 아님 — 카테고리 무료/저/중/고) [G37/G47] |
| deterministicFlag | 불리언 | 결정론 폴백 모드(LLM 점수 없이 규칙 점수만) [PRD 06-9] |

### 7.2 하드 제약 4종 (C2 검증 정본)

| # | 하드 제약 | 정의 | 근거 |
|---|---|---|---|
| HC1 | 영업시간 내 배치 | 어떤 POI도 영업시간 밖(종료 후·휴무일 포함)에 배치되지 않는다 | PRD 06-3, ADR-0008 |
| HC2 | 이동시간 부등식 | `직전 슬롯 endAt + 이동시간(안전 마진 G106) ≤ 다음 슬롯 startAt` — 위반 배치는 결과에서 구조적 배제 | PRD 06-3, G106, ADR-0009 |
| HC3 | 고정 블록 불가침 | 고정 블록(§6) 시각 불변, 충돌 추천 POI 자동 배제 | PRD 06-4, INV-FIX1 |
| HC4 | 시간창 귀속 | 슬롯은 일자 시간창 내(자정 초과=시작일 논리 귀속) | D29, G119 |

> **목적함수**: LLM 선호 점수를 보상값으로, 필수 방문지를 필수 노드로, 체류 범위를 유연 변수로 하는 OPTW/TOPTW 최적화(보상 최대화 + 하드 제약 만족). 예산은 소프트 가중치로만 목적함수에 반영(하드 제약 승격 금지, INV-SOLVE3).

### 7.3 SolverSolution (출력 계약)

| 요소 | 타입 | 의미 |
|---|---|---|
| placedSlots | 목록<ItinerarySlot> | 하드 제약 4종 전부 만족하는 배치(시각·순서·체류 검증값) |
| unplaced | 목록<{poi, reason}> | 배치 불가 방문지 + 사유(이월 목록·구조화 응답) |
| infeasibility | 선택<{dominantConstraint, relaxationProposal}> | 해 없음 시 지배 제약 + 완화 제안(빈 결과 금지) [PRD 06-3·4 예외] |
| compressedStays | 목록<slotId> | 권장→최소 압축 슬롯(표시 사유 생성) [PRD 06-3] |

**불변식**
- **INV-SOLVE1 (하드 제약 만족 — 하드 제약 본체)**: placedSlots의 모든 배치는 HC1~HC4를 만족한다 — 위반 배치는 solution에 존재할 수 없다(구조적 배제). 임의 입력에 대해 무차별 대입(oracle) 대조로 검증 [ADR-0008, U5 DoD, PBT U5-P1(oracle)].
- **INV-SOLVE2 (결정론 폴백 결정성)**: deterministicFlag=true(또는 LLM 점수 부재)일 때 동일 입력은 동일 출력을 낸다 — 무작위성 제거·시드 고정. "덜 최적이어도 유효한 해" 항상 보장 [PRD 06-9, PBT U5-P3].
- **INV-SOLVE3 (예산 비하드화)**: budgetWeight는 목적함수 보상에만 영향하고 하드 제약으로 승격되지 않는다 — 예산 초과 배치가 HC로 차단되지 않으며, 예산대 낮을수록 저비용 POI 보상이 단조 증가(방향만 보장) [G37/G47, PBT U5-P6].
- **INV-SOLVE4 (해 없음 구조화)**: 제약 만족 배치가 불가하면 빈 결과 대신 infeasibility(지배 제약 + 완화 제안)를 반환한다 — 침묵 실패·빈 화면 금지 [ADR-0011, PRD 06-3·4 예외].
- **INV-SOLVE5 (warm-start 보존)**: fixedBlocks·LOCK·완료 방문지 입력분은 solution에서 전후 동일(불변) — 나머지만 재배치 [G46, PBT U5-P2].

> **ConstraintSpec(C2 서브 계약, D28)**: `{version, rules[HC1~HC4]}` — 클라이언트 경량 검증기(shared/validation)가 소비하는 **버전 있는 규칙 명세**. 편집 중 즉시 검사=클라이언트, 저장 확정 검증=서버(C2). 버전 불일치 시 클라 검사 보수적 비활성화(오판 방지 D28). 클라↔서버 규칙 동치는 이중 실행 계약 테스트로 보장(PBT U5-P12).

---

## 8. TravelEstimate — 이동 추정 (C2, 거리 기반 · 소요시간 내부전용)

거리 기반 이동 추정 값 객체. **`internalDuration`(소요시간)은 솔버 이동 부등식(HC2)·트리거 판정 내부 계산 전용**이며 어떤 표시용 DTO에도 매핑되지 않는다(D25/Δ1) — 표시 계층엔 `distance`만 전파한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| distance | 구조체 DistanceRange{value, transportMode, estimatedFlag} | 필수 | **표시 허용값** — 거리·이동 수단·추정 표기(예: "약 850m · 도보, 추정") [D25/Δ1] |
| internalDuration | Duration | 필수 · **표시 금지** | 소요시간 — HC2 이동 부등식·M9 트리거 판정 내부 계산 전용. **표시 DTO 매핑 구조적 금지**(INV-TRAVEL1) [D25/Δ1, ADR-0009] |
| estimationBasis | 열거 {ROAD_TMAP, ROAD_NAVER, STRAIGHT_LINE} | 필수 | 추정 근거 — TMap 도로 거리(1차)·네이버(2차)·직선거리×우회계수(최종 폴백) [D08] |

**불변식**
- **INV-TRAVEL1 (소요시간 표시 금지 — D25 정본)**: internalDuration은 표시용 응답 DTO에 포함되지 않는다 — 어떤 퍼사드도 UI 표시 목적의 소요시간 필드를 반환하지 않으며, 표시 계약은 DistanceRange까지만이다 [D25/Δ1, component-methods 전역 규칙 3, PBT U5-P4].
- **INV-TRAVEL2 (결정성)**: 동일 (from, to, mode, 파라미터)에 대해 estimateTravel은 결정적(동일 입력 동일 출력) — clock·외부 데이터 주입 가능(테스트 결정성). 라우팅 API 실패 시 직선거리 폴백+estimatedFlag=true [G106, G116, PBT U5-P4].
- **INV-TRAVEL3 (추정 플래그 전파)**: estimationBasis=STRAIGHT_LINE(폴백)이면 estimatedFlag=true로 전파되어 UI '추정' 표기 근거가 된다(허위 정확성 금지) [ADR-0009, ADR-0011].

---

## 9. PoiScore — LLM 선호 점수 (C1, 계약 수준)

C1 LLM Gateway가 산출하는 **후보 POI별 선호 점수** 값 객체. 사용자 자유 입력(취향 문장 포함)을 해석해 각 후보에 점수를 매기며, 이 점수가 SolverProblem 목적함수의 보상값이 된다. **closed-set(후보 ID 목록)에서만 부여** — 후보 밖 POI에는 점수가 존재할 수 없다(G115).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| canonicalPoiId | 식별자(Poi 참조) | 필수 | 점수 대상 — **후보 풀 canonical POI에만**(closed-set, INV-SCORE1) [G115] |
| score | 수치 | 필수 | 선호 점수(목적함수 보상값) — 취향 축·자유 입력 해석 반영 [PRD 06-2] |
| preferenceBasis | 목록<문자열> | 선택 | 반영된 선호 근거(예: "한식 선호", "도보 이동", "예산대") — 설명(llmReason) 원천. NULL=규칙 점수 폴백 [PRD 06-5] |
| tier | 열거 {LIGHT, PREMIUM} | 필수 | 호출 티어 — 취향 해석=경량(LIGHT). 회고·요약은 상위(PREMIUM, U7 소비) [D11] |
| isFallback | 불리언 | 필수 · 기본 false | 규칙 점수 폴백 여부 — LLM 실패 시 true("기본 모드" 고지) [PRD 06-9, ADR-0011] |

**불변식**
- **INV-SCORE1 (closed-set — 하드 제약 계열)**: PoiScore는 후보 풀에 속한 canonicalPoiId에만 부여된다 — LLM이 후보 밖 ID를 반환하면 C1 출구에서 구조적으로 드롭·계측(환각 0). 출력 스키마 검증 + ID 화이트리스트 교차 검증 [G115, SECURITY-11, U5 DoD, PBT U5-P5].
- **INV-SCORE2 (전 일자 공용 1회)**: 취향 점수는 전 일자 공용으로 1회 산출(일자별 재호출 없음) — 백그라운드 잔여 일자 solve는 동일 점수 재사용 [S1.2 4단계, S1.4].
- **INV-SCORE3 (무실패 보장)**: LLM 실패·타임아웃(2.5초) 시 규칙 점수(카테고리-취향 매핑 + 인기 집계 가중)로 폴백해 isFallback=true — 취향 없음·LLM 실패가 생성 실패가 되지 않는다 [PRD 06-9, US-E1-14, RESILIENCY-01].
- **INV-SCORE4 (내부 지표 미주입 — 계정 무결성)**: C1 입력은 리소스 ID 참조만이며 게이트웨이가 요청자 권한으로 재조회 주입(D31) — 타 사용자 비공개 데이터·내부 지표(제휴 수수료 등)는 구조적으로 LLM 입력에 미포함(프롬프트 주입 우회 불가) [SECURITY-11, D31, 하드 제약 계정 무결성].

---

## 10. GenerationSession — 생성 세션 (M8)

일정 생성의 진행 상태·부분 결과·취소를 소유하는 세션. **일자별 독립 TX로 부분 노출**을 허용하며, 세션 상태가 진행 정본이다(부분 일정은 세션 상태로 '생성 중'임이 식별됨). 첫 1일 5초 우선 노출·전체 20초 한계·취소 시 초안 보존(G161)을 관리한다.

### 10.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| sessionId | 식별자 | 필수 · 유니크 | — |
| itineraryId | 식별자(Itinerary 참조) | 필수 | 귀속 일정 |
| mode | 열거 {FULL_AUTO, TOGETHER, MANUAL} | 필수 | 생성 방식(§1.5) — 도중 전환 반영 [G48] |
| status | 열거 {COLLECTING, DAY1_READY, GENERATED, FAILED} | 필수 · 기본 `COLLECTING` | 세션 생애 상태(§10.2 상태 머신) |
| progress | 구조체 {stepText, ratio, elapsed} | 필수 | 진행 스트림 — 단계 텍스트("장소 고르는 중→동선 맞추는 중→시간표 확정 중")·진행률·경과 시간 [PRD 06-9, D38] |
| firstDayEta | Duration | 선택 | 첫 1일 예상 시간(5초 게이트) [D38] |
| partialDraft | 참조(부분 DaySchedule[]) | 선택 | 취소·부분 완료 시 초안 보존분. NULL=미보존 [G161] |
| cancelState | 열거 {NONE, CANCELLED_KEPT, BACKGROUND} | 필수 · 기본 `NONE` | 취소 → 초안 보존(CANCELLED_KEPT) / '백그라운드로 계속'(BACKGROUND) [G161, PRD 06-9] |
| fallbackFlags | 집합<열거> | 필수 · 기본 ∅ | 폴백 발생 표기 — LLM_FALLBACK(기본 모드)·EXPLANATION_MISSING·DETERMINISTIC_ONLY(20초 초과)·MINIMAL_ONLY(전 경로 실패) [PRD 06-9] |

### 10.2 세션 상태 머신

```text
   [generateItinerary]
        │
        ▼
    COLLECTING ──(컨텍스트 로드·후보 풀·LLM 점수 실패)──▶ FAILED
        │                                                  ▲
        │ (day1 solve 완료 · 5초 게이트)                     │ (M6·M4 로드 실패 등 치명)
        ▼                                                  │
    DAY1_READY ──(백그라운드 잔여 일자 완료)──▶ GENERATED     │
        │  │                                               │
        │  └──(사용자 취소 keepDraft)──▶ 부분 초안 보존 ──────┘
        │        (Itinerary는 DRAFT 유지 · '이어서 생성' 진입점 G161)
        ▼
    (20초 초과 → 잔여 일자 결정론 단독 모드로 GENERATED + DETERMINISTIC_ONLY 플래그)
```

| 전이 | 트리거 | 가드/효과 |
|---|---|---|
| (없음) → COLLECTING | `generateItinerary` | 진행 스트림 채널 개설(단계 텍스트·진행률) [S1.2 1단계] |
| COLLECTING → DAY1_READY | day1 solve 완료(5초 게이트) | day1 초안 저장(독립 TX) → 첫 1일 응답 반환. 이후 백그라운드 [S1.2 6단계, D38] |
| DAY1_READY → GENERATED | 백그라운드 잔여 일자 완료 | 세션 GENERATED + outbox `ItineraryGenerated`(전체 완료 TX에서만) [S1.2 8단계] |
| COLLECTING → FAILED | 컨텍스트 로드 치명 실패(M6·M4) | 세션 FAILED + 재시도 노출(M2 실패는 중립 기본값으로 대체 — FAILED 아님) [S1.2 2단계] |
| DAY1_READY → (초안 보존) | 사용자 '취소'(keepDraft=true) | partialDraft 저장·cancelState=CANCELLED_KEPT, Itinerary DRAFT 유지, '이어서 생성'(`resumeGeneration`) 진입점 [G161] |

**불변식**
- **INV-SESS1 (진행 정본)**: GenerationSession.status가 생성 진행의 정본 — 부분 일정(day1만 저장)은 세션 상태로 '생성 중'임이 식별된다. `ItineraryGenerated`는 전체 완료 TX에서만 발행(부분 완료는 이벤트 없음) [S1.3].
- **INV-SESS2 (일자별 독립 TX)**: 일자별 저장은 독립 TX — 부분 결과 노출 허용. LLM·솔버 호출은 TX 밖 [S1.3].
- **INV-SESS3 (취소 초안 보존)**: 취소(keepDraft=true) 시 이미 만들어진 부분 일정이 초안으로 보존되고 '이어서 생성' 진입점이 남는다 — 폐기는 keepDraft=false 명시 시에만 [G161, PRD 06-9].
- **INV-SESS4 (최소 일정 최후 폴백)**: 전 생성 경로 실패 시 빈 화면 대신 숙소+시각 고정형 필수 방문지만 배치한 최소 일정 + "추천을 다시 시도"(MINIMAL_ONLY) [PRD 06-9 예외, INV-SOLVE4].

---

## 11. StayZoneRecommendation — 숙소 권역 추천 (M8, US-E5-11 온램프)

숙소 나중 등록 경로 — 완성 일정의 동선 무게중심 기반 숙소 권역 추천. 방문지 분포·평균 이동 거리를 입력으로 '이 동선에 묵으면 이동이 최단'인 권역을 지도로 추천한다. **소요시간 미표시**(before/after는 '추정 이동 거리'만, D25).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 | 대상 여행 |
| centroid | 구조체 {lat, lng} | 필수 | 동선 무게중심(방문지 분포 기반) [US-E5-11] |
| candidates | 목록<{stayZone, avgDistanceBefore, avgDistanceAfter}> | 필수 | 추천 권역 후보 — 전체 일정 평균 이동 거리 순, before/after **추정 이동 거리**(소요시간 없음) [US-E5-11, D25] |
| approximationNote | 문자열 | 필수 | "무게중심 기반 추정 — 개별 동선의 전역 최적을 보장하지 않음" 표기 [US-E5-11] |

**불변식**
- **INV-ZONE1 (동선 부족 게이트)**: 방문지 2개 미만 등 동선 부족 시 권역 추천을 제공하지 않고 일반 탐색 유도(S3 실패 분기) — 빈 추천 금지 [services S3.146].
- **INV-ZONE2 (추정 표기 강제)**: before/after 개선치는 '추정 이동 거리'로만 표시하고 전역 최적 비보장을 명기한다(소요시간 표시 0) [US-E5-11, D25].
- **INV-ZONE3 (등록 온램프 연계)**: 추천 숙소 등록 시 그 숙소를 출발·복귀 기준점으로 US-E5-01 생성·재정렬(warm-start)을 정상 수행한다 — 미등록일(당일치기·이동일)은 숙소 없이 동선만 유지(DaySchedule.appliedBaseRef=NULL) [US-E5-11 예외, INV-DAY... appliedBaseRef].

---

## 12. CP2 소비 참조 계약 (U4 → U5 입력) — TripContext

U5는 U4가 공급하는 **여행 컨텍스트(TripContext)**를 SolverProblem 입력으로 소비한다(스키마 정본은 U4, 본 유닛은 재정의하지 않음). **1차 유닛 체인 정밀도 최고 계약** — U5는 이를 신뢰하되 D21·D15·G40을 방어적으로 재검증한다(하드 제약 입력 무결성 이중 방어, CP2 시나리오 3).

| 입력 객체(U4 소유) | U5 소비 용도 | 소비 시 방어 검증 |
|---|---|---|
| Trip(destination·start/end·party·budget·attributes) | 일수별 DaySchedule 생성·예산 소프트 가중치·C1 점수화 가중치 | 국내 범위(G120)·오늘 이후·≤30일(G42)·겹침 없음(D21) 재검증 |
| TripDateWindow[](일자별 availStart/End) | DaySchedule.timeWindow(솔버 시간 예산 HC4) | 전 일자 총함수(공백 0)·availStart<availEnd(INV-WIN 소비) [D29, G119] |
| BaseAssignment[](구간·숙소 참조·전환일) | DaySchedule.appliedBaseRef·전환일 편도 모델링(G50)·숙소 기준점 하드 제약 | coordConfirmed=true 전제·비중첩(D15)·커버리지 완전성 재검증 [INV-ITIN3] |
| MustVisit[](사본·시각 고정·한도) | FixedBlock(FIXED)·SolverProblem.mustInclude(ANYTIME 앵커)·한도(G40) | 사본 독립(G129)·한도 ≤3×days(G40)·소실 제외(G8)·권역 밖 경고(G158) 재검증 |
| 여행 속성(동행·이동·예산대) | C1 점수화 가중치(PoiScore)·prefs(페이스·이동 수단) | 계정 취향 기본 오버라이드(G134) |

> **계약 변경 통제**: 시간창·거점 구간 표현 변경은 C2 제약 모델 재작성을 유발 — U5 착수 후에는 필드 **추가만** 허용(파괴적 변경 금지, CP2 §125). 시각 고정 필수 방문지가 시간창과 충돌하는 입력(예: 21시 이후 고정)은 침묵 실패 없이 완화 제안(INV-SOLVE4·INV-FIX3)을 구조화 응답으로 반환(CP2 시나리오 2).

---

## 13. CP3 공급 계약 (U5 → U6 출력) — 일정 기준선

U5 산출(plan/current·고정 블록·확정 상태)이 U6 실행 허브·재계획의 입력 기준선이며, **5개 유닛(U6·U7·U8·U10·U11)이 공유하는 척추**다(plan/current 이중 구조·슬롯 식별 체계 변경은 ADR급). U5(M8)가 공급하고 U6(M18·M9·M10)이 소비한다.

| 공급 객체 | 필드(개요) | 경계 무결성 보장 |
|---|---|---|
| Itinerary | itineraryId·tripId·상태(DRAFT→EDITING→CONFIRMED→INTRIP D20)·**plan 불변 스냅샷**(확정 시 동결)+**current 가변본**(D14)·확정 일시·mode | INV-ITIN2·4·INV-PLAN1·2 |
| DaySchedule | 일자·적용 거점 참조(구간 거점)·시간창 스냅샷·전환일 | INV-DAY1·2·3 |
| ItinerarySlot | **slotId**(항목 단위 식별 — U11 항목 잠금 D30의 키)·canonicalPoiId(closed-set 보장분 G115)·start/end(C2 검증값)·LOCK·고정 블록 플래그(warm-start 보존 G46)·추천/배치 이유·이동 구간 DistanceRange(거리·수단만 — 소요시간 없음 D25) | INV-SLOT1·2·4·5 |
| 이벤트(CP5 일부) | `ItineraryConfirmed`(확정·리마인드 스케줄 소스)·`ItineraryChanged`(변경·재계산 D32) | CP5 envelope(common/core) |

**검증 방법 — 통합 테스트 시나리오(CP3)**
1. **기준선 로드 불변성**: 확정 일정을 U6 실행 허브가 로드하면 current가 실행 기준선이 되고, 임의 재계획·현장 편집 후에도 plan 스냅샷이 바이트 동일하게 유지된다(INV-PLAN1) [PBT U5-P7].
2. **warm-start 고정 블록 보존**: LOCK 슬롯·시각 고정 필수 방문지·거점 블록이 포함된 일정에 재계획을 실행하면 고정 블록이 전후 동일하고 나머지만 재배치되며 하드 제약 4계열(C2 재사용) 통과(INV-SLOT4·INV-SOLVE5) [PBT U5-P2].
3. **확정 해제 경합**: 소유자가 확정 해제·편집 중 상태로 전환한 일정에 대해 U6 트리거 발화·재계획 진입이 상태 머신 규칙(D20)에 따라 차단/안내(INV-ITIN4) [PBT U5-P9].

> **계약 변경 통제**: plan/current 이중 구조와 slotId 식별 체계는 5개 유닛이 공유하는 척추 — 변경은 ADR급 결정으로 통제(CP3 §145).

---

## 14. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| Itinerary | US-E5-01·10·12 | D14, D20, G46/G136, components §3.2 |
| PlanSnapshot / CurrentItinerary | US-E5-12, US-E5-08 | D14, ADR-0013, D16 |
| DaySchedule | US-E5-01·03 | D29, G119, G50, G41 |
| ItinerarySlot | US-E5-03·05·06·07·10 | G115, D25/Δ1, G46, G51, D28 |
| FixedBlock | US-E5-04 | PRD 06-4, G46, D29 |
| SolverProblem / SolverSolution | US-E5-02·03·04·09 | ADR-0008, D28, D29, G47, G50, G106, G115 |
| TravelEstimate | US-E5-06 | D25/Δ1, ADR-0009, D08, G106 |
| PoiScore | US-E5-02·05 | D11, D31, G115, SECURITY-11 |
| GenerationSession | US-E5-09·10 | D38, G161, G136 |
| StayZoneRecommendation | US-E5-11 | D25 |
