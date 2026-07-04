# U4 여행 생성·필수 방문지 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M6 Trip Creation
> 정본 관계: [components.md](../../../inception/application-design/components.md) §3.1(여행 상태 머신 정본)·M6 섹션의 엔티티 개요를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U4 엔티티·불변식의 정본이다. 기술 중립 — 저장 기술·컬럼 타입·인덱스·라운딩 구현체는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL의 의미를 반드시 명기), `유니크`=제약 범위 내 유일, `파생`=저장 안 함(조회 시점 계산). 근거 ID는 requirements.md(D·G·N·ADR)·stories.md(US-E4)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U4-xx)을 참조한다.
> **U4는 CP1(U3→U4)의 소비자이자 CP2(U4→U5)의 공급자다** — 등록 숙소(SavedStay)·저장 POI(SavedPlace)는 U3([u3-place-stay/functional-design/domain-entities.md](../../u3-place-stay/functional-design/domain-entities.md))이 소유하는 입력이며(§6 참조 계약), 본 유닛의 산출 스키마가 U5 솔버 문제 정의를 결정한다(CP2 §7).

## 0. 엔티티 지도

```text
[M6 Trip Creation — 여행 컨텍스트 정본]
Account 1 ──── * Trip                    (계정 귀속 여행 — 활성 여행 ≤1, D21)
Trip 1 ──── * TripDateWindow             (날짜별 이용 가능 시각, G119/D29)
Trip 1 ──── * BaseAssignment             (거점 배정 — 스키마·API는 U3 M4, 여행 내 비중첩 검증·스마트 기본은 U4 소비 CP1)
Trip 1 ──── * MustVisit                  (필수 방문지 — 저장 POI 사본 투입 G129·한도 G40)
Trip 1 ──── 1 BudgetAllocation           (예산 전체총액→일예산 파생 — 값 객체, 저장은 Trip.budgetTotal)

[CP1 입력 — U3 소유(본 유닛은 참조·소비만)]
BaseAssignment * ──── 1 SavedStay        (U3 M4 — 계정 레벨 풀, D15)
MustVisit.poiSnapshotRef ──▶ PoiSnapshot (U3 M7 — 확정 시점 사본, 저장 POI 투입 시 M6가 사본 복제 G129)

[CP2 출력 — U5 M8·C2 소비]
Trip + TripDateWindow[] + BaseAssignment[] + MustVisit[] + 여행 속성 → TripContext(솔버 문제 정의, §7)
```

---

## 1. Trip — 여행 (M6, 상태 머신 정본)

여행 단위(목적지·기간·인원·예산·제목·속성)의 정본이자 **여행 상태 머신(§1.4)의 정본 소유자**. 활성 여행은 항상 최대 1개(D21)이며, 이 엔티티의 무결성(겹침 차단·날짜 규칙·국내 범위)이 U5 솔버 "숙소 기준점·충돌 무배치" 하드 제약의 입력 전제다.

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자 | 필수 · 유니크 · 불변 | 여행 정본 키 — BaseAssignment·MustVisit·TripDateWindow가 참조. U5 일정·U6 실행·U7 기록의 여행 참조 단일 키 |
| accountId | 식별자(Account FK) | 필수 | 계정 귀속 — 소유권 검증(SECURITY-08, IDOR 방지) [D22] |
| title | 문자열 | 필수 · 기본 자동생성 | 여행 제목. 미입력 시 `{여행지} N박M일` 자동 생성(§1.3 결정성), 수시 수정, 금칙어 `C3.checkText` [N6] |
| destination | 구조체 {regionRef, centerCoord{lat,lng}} | 필수 | 여행지(명칭 참조 + 중심 좌표). 중심 좌표는 국내 좌표 범위 검증 통과분(INV-TRIP3)·첫날 공백 기본 거점 좌표원(G41) [G120] |
| startDate | 날짜 | 필수 | 시작일 — 오늘 이후만(INV-TRIP2) [G42] |
| endDate | 날짜 | 필수 | 종료일 — `endDate ≥ startDate` ∧ 기간 ≤ 30일(INV-TRIP2) [G42] |
| party | 정수(양수) | 필수 · 기본 1 | 인원. 미입력 시 1명 — 예산 1인 파생·인원 가중 입력 [PRD 05-1] |
| budgetTotal | 구조체(금액){rawTotal, mappedBand?} | 선택 | **여행 전체 총액(항공 제외 — 숙소+식비+활동+교통)**. NULL=예산 미설정. 온보딩 러프 예산(M2)을 기본값 제안하되 정확 총액으로 수정 가능. 1인·1일은 파생(BudgetAllocation §5, 저장 안 함) [D26/Δ2, G36] |
| attributes | 구조체 {companion, transportModes, budgetTier} | 필수 · 기본 계정 취향 | 여행별 속성 — 동행 유형·이동 수단·예산대. 계정 취향(M2)을 기본값 제안한 여행별 오버라이드, C1 점수화 가중치 입력(CP2) [G134] |
| status | 열거 {PLANNED, CONFIRMED, ACTIVE, ENDED} | 필수 · 기본 `PLANNED` | 여행 생애 상태(§1.4 상태 머신 정본). 전이 트리거는 타 유닛 이벤트 수용(CONFIRMED=M8, ACTIVE/ENDED=M18) |
| deletedAt | 시각(UTC) | 선택 | 소프트 삭제 30일 유예(직교 플래그 — 상태와 독립). NULL=미삭제. 값 존재 시 즉시 비노출 [D18] |
| createdAt | 시각(UTC) | 필수 · 불변 | 생성 일시 |

> **여행 속성 세부**: `companion`(동행 유형 — 기본 단일 + '반려동물 동반' 보조 불리언 G19 계열), `transportModes`(이동 수단 다중), `budgetTier`(예산대 열거) — 전부 계정 취향(M2) 기본값 제안 후 여행별 저장 [G134].

### 1.2 파생 표시값 (저장 안 함)

| 파생값 | 계산 | 근거 |
|---|---|---|
| nights / days | `nights = endDate - startDate`, `days = nights + 1` | 제목 자동생성·일예산 분모·한도 분모 |
| dDay | `startDate - today`(여행 전만) | 목록 카드 표시(G3) — 여행 중은 방문 체크 비율(M18 소유) |
| 예산 1인·1일 | BudgetAllocation §5 파생 | D26 — 조회 시점 계산 |

### 1.3 제목 자동생성 규칙 (N6 — 결정성)

- 미입력 제목은 `{destination.regionRef 명칭} {nights}박{days}일`로 결정적 생성(동일 여행지·기간 입력 → 동일 제목, PBT U4-P6).
- 사용자 입력 제목·자동 제목 모두 금칙어 검증(`C3.checkText`) 통과 필수 — 위반 시 인라인 오류·대체 제안(빈 제목 저장 금지) [N6, SECURITY-05].

### 1.4 여행 상태 머신 (정본 — components.md §3.1 상세화)

```text
      [createTrip]
          │
          ▼
      PLANNED ──(M8 ItineraryConfirmed · 여행 시작 전)──▶ CONFIRMED
        │  ▲                                                │  │
        │  └──(M8 unlockForEdit · 여행 시작 전, D20)────────┘  │
        │                                                      │
        │ (시작일 00:00 도달 — M18 일자 경계 배치)              │ (시작일 00:00 도달 — M18)
        ▼                                                      ▼
      ACTIVE ◀──────────────────────────────────────────────┘
        │
        │ (a) 종료일 다음날 00:00 자동(M18 autoEndTrips) / (b) 수동 '여행 종료' 버튼
        ▼
      ENDED (최종)
```

| 전이 | 트리거 | 가드 조건 | 효과 |
|---|---|---|---|
| (없음) → PLANNED | `createTrip` | 겹침 차단(D21)·날짜 규칙(G42)·국내 범위(G120)·제목 금칙어(N6) 통과 | 여행 생성 — 활성 여행 ≤1 불변식 보전(INV-TRIP1) |
| PLANNED → CONFIRMED | M8 `ItineraryConfirmed` 이벤트 | 여행 시작 전 · 활성 일정 존재 | plan 스냅샷 동결(D14, U5 소유) 참조 · M14 리마인드 스케줄 |
| CONFIRMED → PLANNED | M8 `unlockForEdit`(D20) | 여행 시작 전(여행 중 편집은 current에만 반영되므로 이 전이 불가) | 재확정 전 신규 공개 불가(D20) |
| PLANNED/CONFIRMED → ACTIVE | 시작일 00:00 도달(M18 일자 경계 배치) | D21로 동시 활성 여행은 구조적 ≤1 · 일정 미확정이라도 날짜 진입이면 ACTIVE(PRD 07-1) | 홈 활성 카드·일정 탭이 M18 허브로 수렴, plan/current 분리 시작(D14) |
| ACTIVE → ENDED | (a) 종료일 다음날 00:00 자동(M18 `autoEndTrips`) (b) 사용자 '여행 종료' 수동 버튼 | ACTIVE 상태 · 숙소 유무 무관 단일 규칙(D19, Δ4) | `TripEnded` 발행 → M13 회고, M14 완료 알림, M12 기록 귀속 마감 |
| ENDED (최종) | — | 재개 없음 · 종료 후 기록 편집 허용, 회고 갱신은 수동 재생성만(C11) | — |

> **분업 경계**: U4(M6)는 상태 **엔티티·전이 가드의 정본**을 소유하되, CONFIRMED 전이는 M8(U5) 이벤트를, ACTIVE/ENDED 전이는 M18(U6) 트리거를 수용한다 — U4 구현 범위는 상태 필드·전이 규칙·가드 검증이며, 트리거 발행 주체는 타 유닛(1차 U4 구현 시 이벤트 수신 계약만 확정, INV-TRIP6).

### 1.5 불변식 (Invariants)

- **INV-TRIP1 (겹침 차단·활성 ≤1 — 하드 제약)**: 같은 accountId의 `deletedAt=NULL`인 여행 중 날짜 구간이 겹치는 쌍은 존재하지 않는다 — 생성·기간 변경 단계에서 겹침을 차단(D21). 결과로 동시 ACTIVE 여행은 항상 최대 1개. **하드 제약(충돌 무배치 계열의 입력 무결성)·머지 차단 테스트(D37, §6.6 G114)** [D21/Δ3, U4 DoD, PBT U4-P1].
- **INV-TRIP2 (날짜 규칙)**: `startDate ≥ today` ∧ `endDate ≥ startDate` ∧ `days ≤ 30`. 위반 시 생성·변경 차단·인라인 표시 [G42, PBT U4-P5(전이 가드 일부)].
- **INV-TRIP3 (국내 범위)**: `destination.centerCoord`는 국내 좌표 범위(bounding box/역지오코딩) 내 — 벗어나면 차단+사유(국내 한정 강제) [G120].
- **INV-TRIP4 (제목 존재·결정성)**: title은 항상 비어 있지 않다(미입력은 자동생성으로 채움) — 자동생성은 결정적, 금칙어 통과분만 저장 [N6, PBT U4-P6].
- **INV-TRIP5 (예산 전체총액 정규화)**: budgetTotal은 **전체 총액(항공 제외)**만 저장한다 — 카테고리 분배·1인/1일 값은 저장하지 않고 파생(BudgetAllocation §5). 저장 축과 파생 축의 라운드트립은 라운딩 한계 내 정합(INV-BUDGET1) [D26/Δ2, G37, PBT U4-P2].
- **INV-TRIP6 (상태 전이 가드)**: status는 §1.4 전이표의 (트리거·가드) 조합으로만 변경된다 — 표 밖 전이(예: ENDED→ACTIVE 재개, ACTIVE→CONFIRMED 역행)는 구조적 불가. 각 전이의 가드 미충족 시 전이 거부 [PBT U4-P5].

---

## 2. TripDateWindow — 날짜별 이용 가능 시각 (M6)

여행 각 날짜의 '이용 가능 시작/종료 시각'(솔버 시간 예산). 첫날 도착·마지막날 출발 시각을 반영해 일정 생성 시간 예산에 직결(CP2 공급). 정확 시각이 아닌 '이용 가능 창'이며, 자정 초과 활동은 해당 일자에 논리적으로 귀속(논리적 하루).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip FK) | 필수 | 귀속 여행 |
| date | 날짜 | 필수 · 유니크(tripId 내) | 대상 일자 — 여행 기간 내(INV-WIN2) |
| availStart | 시각(당일) | 필수 · 기본 09:00 | 이용 가능 시작 — 첫날은 도착 시각 반영 [G119, D29] |
| availEnd | 시각(당일) | 필수 · 기본 21:00 | 이용 가능 종료 — 마지막날은 출발 시각 반영 [G119, D29] |
| isUserOverridden | 불리언 | 필수 · 기본 false | 사용자 조정 여부(기본 시간창 vs 명시 조정 구분) |

**불변식**
- **INV-WIN1 (시각 순서)**: `availStart < availEnd`. 위반 시 저장 차단·인라인 [G119].
- **INV-WIN2 (기간 정합)**: date는 `[startDate, endDate]` 내에 존재하며, 기간 변경 시 창 밖 일자는 파괴적 변경 확인 대상(G39, BR-U4-10) [G119].
- **INV-WIN3 (기본값 완전성)**: 사용자가 조정하지 않은 일자도 기본 09:00~21:00 창을 갖는다 — CP2로 전달되는 시간창은 전 일자에 대해 총함수(공백 일자 0) [D29, G119].

---

## 3. BaseAssignment — 여행 거점 배정 (스키마 U3 M4 · 여행 내 검증은 U4 M6 소비)

SavedStay(U3 계정 풀)를 여행 구간 거점으로 연결하는 조인. **데이터·API·조인 스키마·비중첩 판정 순수 함수·스마트 기본 거점 산출(getBaseTimeline)은 U3(M4)이 소유·공급**하고(U3 domain-entities §7·BR-U3-17, INV-BASE1·CP1), **여행 컨텍스트가 필요한 여행 내 거점 비중첩 검증 실행·전환일 처리·커버리지 완성·거점 UI는 U4(M6)가 소비·완성**한다. 본 절은 U4가 완성하는 여행 측 규칙·불변식을 정의한다(엔티티 속성 정본은 U3 §7).

### 3.1 U4가 완성하는 파생·산출

| 산출 | 타입 | 의미·근거 |
|---|---|---|
| 비중첩 검증 결과 | `ok \| ConflictDetected(overlapDates)` | 같은 tripId의 active BaseAssignment dateRange 집합에 U3 비중첩 판정 순수 함수를 적용해 **검증을 실행**(계정 풀 미연결 숙소는 대상 아님). 겹침 시 겹치는 날짜 표시+거점 선택 유도 [D15, BR-U4-06] |
| isSmartDefault | 불리언(U3 산출·저장, U4 소비) | 공백일 스마트 기본 거점 — **U3 getBaseTimeline이 산출·저장**(공백일=직전 숙소, **첫날 공백=여행지 중심 좌표** Trip.destination.centerCoord, isSmartDefault=true). U4는 이를 소비해 비차단 안내(사후 수정 배지) 표시 [G41, BR-U3-17, BR-U4-12] |
| transitionDay | 파생 식별 | 전환일 — A 숙소 체크아웃일 = B 숙소 체크인일. "출발점=A·복귀점=B" 편도 동선 모델링 입력(CP2, U5 C2 소비) [G50] |
| coverage | 파생 | 여행 전 일자에 대해 (실거점 ∪ 스마트 기본)이 빈틈없이 배정 — CP2 거점 목록의 완전성 [D15, PBT U4-P3] |

### 3.2 불변식 (U4 소비 측)

- **INV-BASE-U4-1 (여행 내 비중첩 — 하드 제약)**: 같은 tripId에 연결된 active BaseAssignment들의 dateRange는 서로 겹치지 않는다 — **검증 실행은 U4**(U3은 판정 순수 함수 공급). 겹침을 허용해 거점 둘인 상태로 일정 생성 진입 금지. **하드 제약(충돌 무배치 계열 입력 무결성)·머지 차단 테스트(D37, §6.6 G114)** [D15, U4 DoD, PBT U4-P3, CP1].
- **INV-BASE-U4-2 (좌표 확정 전제)**: coordConfirmed=true인 SavedStay만 거점 배정 대상(U3 INV-STAY2 이중 방어) — 미확정 좌표는 "지도에서 위치 확인" 유도 [CP1 시나리오 2].
- **INV-BASE-U4-3 (스마트 기본 비차단)**: 공백일 스마트 기본 거점은 사용자 흐름을 차단하지 않는다(isSmartDefault=true·사후 수정 가능) — 첫날 공백은 여행지 중심 좌표로 채움(U3 getBaseTimeline 산출·저장, U4 소비) [G41, BR-U3-17, PRD 05-6].
- **INV-BASE-U4-4 (커버리지 완전성)**: CP2로 공급되는 거점 목록은 여행 전 일자를 빈틈없이 커버한다(실거점 + 스마트 기본) — 미배정 일자 0 [D15, G41, PBT U4-P3].

---

## 4. MustVisit — 필수 방문지 (M6)

AI 일정 생성의 기준점(앵커)으로 사용자가 미리 지정한 '꼭 가고 싶은 장소'. 저장 POI 투입 시 **사본 복제**(원본 저장 목록과 독립, G129)하며, 일수 비례 한도(G40) 내에서만 추가된다. 2유형(포함 고정형 기본 / 시각 고정형)으로 일정 반영 규칙이 나뉜다(정본은 U5 PRD 06-4).

### 4.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| mustVisitId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip FK) | 필수 | 귀속 여행 |
| poiSnapshotRef | 식별자(PoiSnapshot 참조) | 필수 | **확정 시점 사본 참조**(U3 M7 PoiSnapshot, purpose=MUST_VISIT). 저장 POI 투입은 M6가 사본 복제 — 원본 SavedPlace 삭제와 독립(INV-MV1) [D13, G129] |
| canonicalPoiId | 식별자(Poi 참조) | 필수 | 원본 참조(대조·소실 판정용) — 사본은 독립 잔존 |
| type | 열거 {ANYTIME, FIXED} | 필수 · 기본 `ANYTIME` | 유형 — `ANYTIME`(포함만 보장, 시각·순서는 AI 배치 — 화면 라벨 '아무 때나 꼭 가기') / `FIXED`(시각 고정 — 라벨 '시간 정해두기'). 내부 용어(§4.2) [PRD 05-8] |
| fixedDate | 날짜 | 선택 | FIXED 시 지정 날짜. NULL=미입력 → 영업시간·동선 맞춰 자동 배치(U5) [PRD 05-8 예외] |
| fixedStart | 시각 | 선택 | FIXED 시 시작 시각. NULL=미입력 → 자동 배치 |
| stayDuration | 정수(분) | 선택 | FIXED 시 체류 시간. NULL=미입력 → 카테고리 기본값(StayTimeTable, U3 M7) |
| regionOutsideFlag | 불리언 | 필수 · 기본 false | 여행지 권역 밖 항목 표시(경고 배지+기본 해제 신호) [G158] |
| staleFlag | 불리언(파생) | 필수 · 기본 false | 원본 Poi가 `LOST`면 true → '확인 불가' 배지, 시드 투입 제외(사본은 잔존) [G8] |
| createdAt | 시각(UTC) | 필수 | 지정 일시 |

### 4.2 2유형 요약 (내부 용어 ↔ 화면 라벨)

| 내부 type | 화면 라벨 | 일정 반영(정본 U5 PRD 06-4) |
|---|---|---|
| ANYTIME | '아무 때나 꼭 가기' | 포함만 보장 — 시각·순서 재계산 가능하나 일정에서 누락 금지(US-E4-09) |
| FIXED | '시간 정해두기' | 시각 고정 — 생성·재생성 후에도 날짜·시간 불변(고정 블록·warm-start 보존 G46) |

### 4.3 불변식

- **INV-MV1 (사본 독립 — G129)**: MustVisit는 poiSnapshotRef 사본을 보유하며, 원본 SavedPlace/PoiSnapshot 삭제·갱신이 MustVisit에 역류하지 않는다(투입 시점 동결) — "사본" 의미는 UI·설계 양쪽에 명시 [G129, U4 리스크].
- **INV-MV2 (한도 — 하드 제약 G40)**: 한 여행의 MustVisit 개수 ≤ `3 × days`. 초과 추가는 `LimitExceeded`로 차단(추가 순서·일수 무관 상한 불변). **하드 제약(충돌 무배치 계열 입력 무결성)·머지 차단 테스트(D37, §6.6 G114)** [G40, U4 DoD, PBT U4-P4].
- **INV-MV3 (유형 필드 정합)**: type=ANYTIME이면 fixedDate·fixedStart·stayDuration는 무의미(무시). type=FIXED이면 세 필드는 선택 — 미입력은 오류가 아니라 자동 배치 신호(빈 일정 금지) [PRD 05-8 예외].
- **INV-MV4 (권역 밖 기본 해제 — G158)**: regionOutsideFlag=true 항목은 체크박스 화면에서 경고 배지+기본 해제 상태로 표시된다(사용자 명시 선택 시에만 투입) [G158].
- **INV-MV5 (소실 제외 — G8)**: staleFlag=true(원본 LOST) 항목은 시드 투입 제외 안내되되 이미 지정된 사본은 파기되지 않는다('확인 불가' 배지 병기) [G8, CP1 시나리오 3].

---

## 5. BudgetAllocation — 예산 배분 파생 (M6, 값 객체 — 저장 안 함)

여행 전체 총액(Trip.budgetTotal)에서 **파생 계산**되는 1인·1일·1인1일 예산과 솔버 소프트 가중치. v1은 카테고리 분배 없이 총액÷일수÷인원만 산출한다(G37/G47). 저장 실체는 Trip.budgetTotal뿐이며, 본 값 객체는 조회 시점 파생(components.md "1인·1일 재계산은 조회 시점 파생").

| 파생값 | 계산 | 의미·근거 |
|---|---|---|
| rawTotal | `Trip.budgetTotal.rawTotal` | 전체 총액(항공 제외 — 정본 저장 축) [D26/Δ2] |
| perPerson | `rawTotal / party` | 1인 파생 표기 |
| perDay | `rawTotal / days` | 1일 파생 표기 |
| perPersonPerDay | `rawTotal / (party × days)` | 1인 1일 파생 — 숙소 가격대 환산·솔버 소프트 가중치 입력(G26/G37) |
| softWeight | perPersonPerDay 기반 카테고리 무료/저/중/고 추정 | **솔버 하드 제약 아님** — LLM 추천 소프트 가중치 + 숙소 필터 상한 환산만(INV-BUDGET2) [G37/G47] |

**불변식**
- **INV-BUDGET1 (라운드트립 정합)**: budgetTotal 존재 시 파생 4값은 결정적이며 `perPersonPerDay × party × days`는 rawTotal과 라운딩 한계 내 정합(왕복 손실 ≤ 라운딩 단위) — 전체총액↔파생 라운드트립 [D26, PBT U4-P2].
- **INV-BUDGET2 (소프트 가중치 게이트)**: 예산은 솔버 하드 제약으로 승격되지 않는다 — softWeight는 C1 점수화 가중치·숙소 필터 상한으로만 전파(예산 초과 배치가 하드 차단되지 않음) [G37/G47, ADR-0008].
- **INV-BUDGET3 (미설정 중립)**: budgetTotal=NULL이면 BudgetAllocation은 미산출(중립) — softWeight 없이 취향 기반 점수화만(빈 값 강요 금지) [PRD 05-1].

---

## 6. CP1 소비 참조 계약 (U3 → U4 입력)

U4는 다음 U3 소유 엔티티를 **참조·소비만** 한다(스키마 정본은 U3, 본 유닛은 재정의하지 않음). 계약 무결성은 CP1 통합 테스트로 검증.

| 입력 엔티티(U3 소유) | U4 소비 용도 | 소비 시 검증 |
|---|---|---|
| SavedStay(내부 숙소 ID·좌표·체크인/아웃·등록 경로) | BaseAssignment 거점 배정 입력, 숙소 날짜→여행 기간 자동 반영(US-E4-05) | coordConfirmed=true 전제(INV-BASE-U4-2), 체크아웃>체크인 완료 상태 |
| SavedPlace(canonical POI·스냅샷·소실 플래그) | MustVisit 체크박스 시드 투입(사본 복제 G129) | 소실('확인 불가') 항목 투입 제외(INV-MV5), 권역 밖 경고(G158) |
| M7 장소 검색 API(`searchPoi`) | 필수 방문지 검색 추가 재사용 | canonical POI·좌표 미확정 시 핀 지정 요구 |

---

## 7. CP2 공급 계약 (U4 → U5 출력) — TripContext

U4 산출 스키마가 U5 솔버 문제 정의를 결정한다(**1차 유닛 체인 정밀도 최고 계약**). 아래 조립 결과가 CP2 경계로 공급되며, U5는 이를 신뢰하되 D21·D15·G40을 방어적으로 재검증한다(이중 방어).

| 공급 객체 | 필드(개요) | 경계 무결성 보장 |
|---|---|---|
| Trip | tripId·accountId·destination(국내 범위 G120)·start/end(오늘 이후·≤30일 G42·겹침 없음 D21)·party·budget(총액+파생 D26)·title·status | INV-TRIP1·2·3·5 |
| TripDateWindow[] | 일자별 availStart/End(기본 09:00~21:00 D29, 첫날·마지막날 반영 G119) — 전 일자 총함수 | INV-WIN3 |
| BaseAssignment[] | 구간(날짜 range — 비중첩 D15)·숙소 참조(내부 ID+스냅샷 좌표)·다박 표현·첫날 공백 기본 거점(G41)·전환일 식별(G50) | INV-BASE-U4-1·4 |
| MustVisit[] | POI 사본(원본 독립 G129)·시각 고정(선택)·한도 내 보장(G40)·권역 밖 경고 이력(G158) | INV-MV1·2 |
| 여행 속성 | 동행·이동·예산대(계정 취향 기본 오버라이드 G134) — C1 점수화 가중치 | — |

> **계약 변경 통제**: 시간창·거점 구간 표현 변경은 C2 제약 모델 재작성을 유발 — U5 착수 후에는 필드 **추가만** 허용(파괴적 변경 금지, CP2(계약 변경 시 영향 범위)).

---

## 8. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| Trip | US-E4-01·02·11 | D21/Δ3, D26/Δ2, D29, N6, G42, G120, G134 |
| TripDateWindow | US-E4-01 | G119, D29 |
| BaseAssignment(U4 소비) | US-E4-03·04·06·07 | D15, G41, G50, CP1 |
| MustVisit | US-E4-08·09, US-E2-05 | G40, G129, G158, G43, G8, G46 |
| BudgetAllocation(파생) | US-E4-01 | D26/Δ2, G37/G47, G26 |
