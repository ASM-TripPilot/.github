# U4 여행 생성·필수 방문지 — Functional Design 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · 유닛 U4 (E4 스토리 11개 — US-E4-01~11)
> 정본 입력: [unit-of-work.md](../../../inception/application-design/unit-of-work.md) U4 섹션 + [unit-of-work-dependency.md](../../../inception/application-design/unit-of-work-dependency.md) CP1(U3→U4 소비)·CP2(U4→U5 공급) 계약, [stories.md](../../../inception/user-stories/stories.md) Epic 4, [components.md](../../../inception/application-design/components.md) §3.1(여행 상태 머신)·M6, [component-methods.md](../../../inception/application-design/component-methods.md) M6, [requirements.md](../../../inception/requirements/requirements.md) D15·D19·D21/Δ3·D26/Δ2·N6·§7.2(G37·G39·G40·G41·G42·G43·G119·G129·G134·G158), docs/PRD/05-여행생성.md, [u3-place-stay/functional-design/domain-entities.md](../../u3-place-stay/functional-design/domain-entities.md)(SavedStay·SavedPlace·BaseAssignment 계약 — CP1 입력)

## 질문 처리 방침

**사용자 연속 진행 지시에 따라, 본 스테이지의 확인 질문은 requirements.md의 기확정 결정(D01~D38·Δ1~Δ10·G 제안 기본값 — 문서 승인으로 확정)으로 대체한다.** U4 소관 핵심 결정: D21/Δ3(날짜 겹침 차단·활성 여행 ≤1), D26/Δ2(예산 전체 총액), D15(계정 레벨 숙소 풀·여행 내 거점 비중첩), D19/Δ4(여행 종료 단일 규칙), N6(여행 제목), 그리고 §7.2의 G 기본값(G37·G39·G40·G41·G42·G43·G119·G129·G134·G158). Functional Design에서 추가 확정할 설계 결정은 아래 "설계 결정 목록"(FD-U4-xx)에 기록한다.

> **선결 과제 전제**: U4는 직접 선결 없음 — U3의 P2(지도 API 약관·D13)·P4(TourAPI 캐싱) 완료를 전제로 하며, 여행 생성은 전부 자체 데이터 위에서 동작한다(unit-of-work U4 §선결). BaseAssignment 스키마·비중첩 판정 순수 함수는 U3(M4)이 공급하고, U4는 여행 컨텍스트 검증 실행·거점 UI를 소비·완성한다.

## 실행 체크리스트

- [x] 정본 문서 로드·대조 (unit-of-work U4 / dependency CP1·CP2 / stories E4 / components §3.1·M6 / component-methods M6 / requirements D15·19·21·26·N6·§7.2 G / PRD 05 / u3 SavedStay·SavedPlace·BaseAssignment)
- [x] 도메인 엔티티 정의 — [domain-entities.md](../functional-design/domain-entities.md)
  - [x] Trip — 상태 머신 정본(PLANNED→CONFIRMED→ACTIVE→ENDED §1.4)·제목 N6·여행지·날짜(겹침 차단 D21·오늘 이후·≤30일 G42)·인원·예산 전체총액 D26·여행 속성(동행/이동/예산대 G134)·국내 범위 G120
  - [x] TripDateWindow — 날짜별 이용 가능 시각(기본 09:00~21:00·첫날·마지막날 G119·전 일자 총함수)
  - [x] BaseAssignment — 거점 배정(계정 숙소 연결 D15·비중첩 검증 소비 U3 판정 함수·스마트 기본 거점 G41·전환일 G50·커버리지)
  - [x] MustVisit — 필수 방문지 2유형(ANYTIME/FIXED)·저장 POI 사본 투입 G129·한도 일수비례 G40·권역 밖 G158·소실 제외 G8
  - [x] BudgetAllocation — 전체총액→1인·1일 파생 G37/G47·소프트 가중치(하드 제약 아님)
  - [x] 속성표·관계·불변식·상태 전이표(트리거·가드) + CP1 소비 참조 계약(§6) + CP2 공급 계약 TripContext(§7)
- [x] 비즈니스 규칙 카탈로그 — [business-rules.md](../functional-design/business-rules.md) (BR-U4-01~25, 조건/동작/위반 시 처리/근거)
  - [x] 여행 생성(필수 입력·날짜 검증·겹침 차단 D21·국내 G120·제목 자동생성 N6) / 예산(전체 총액 정규화 D26·일예산 파생·소프트 가중치 G37)
  - [x] 거점(계정 풀 연결 D15·날짜 자동 반영·기간 이탈 확인·다중 비중첩·첫날 공백 기본거점 G41·다박·전환일)
  - [x] 필수 방문지(2유형·사본 G129·권역 밖 G158·한도 G40·소실 제외 G8·확정 후 미리보기 G43) / 시간창(G119)
  - [x] 파괴적 변경 차단형 확인(G39)·소프트 삭제 연쇄(D18) / 상태 머신(전이 가드·종료 단일 규칙 D19) / CP2 공급·경계 무결성 이중 방어
- [x] 비즈니스 로직 모델 — [business-logic-model.md](../functional-design/business-logic-model.md)
  - [x] 프로세스 플로우 4종(여행 생성·겹침/제목/예산 / 숙소 거점 연결 CP1 소비·비중첩·자동 반영 / 필수 방문지 사본·권역·한도 / 여행 컨텍스트 조립 CP2 공급)
  - [x] Testable Properties(PBT-01) — 컴포넌트별 속성 식별 표(U4-P1~P6), U4 DoD 5속성 커버리지 대조, 하드 제약 3계열 대조
- [x] 프런트엔드 컴포넌트 설계 — [frontend-components.md](../functional-design/frontend-components.md) (여행 생성 폼·목록·상세·거점 연결·필수 방문지·기간 조정·지도 핀 U3 shared/map 재사용·M6 매핑·data-testid `trip-{screen}-{role}`)
- [x] 추적 ID(US/D/G/N/BR/FD/PBT) 일관 표기 검수
- [x] 확장 규칙(SECURITY·PBT) 관련 조항 반영 검수 — 기술 중립 유지(알고리즘·인프라·라운딩 구현체는 NFR/Infra 이관)

## 설계 결정 목록 (본 스테이지 확정분)

| # | 결정 | 내용 | 근거 |
|---|---|---|---|
| FD-U4-01 | 여행 상태 머신 정본 소유·트리거 분업 | Trip은 PLANNED→CONFIRMED→ACTIVE→ENDED 상태 머신의 정본을 소유. U4는 상태 엔티티·전이 가드 정본이되 CONFIRMED 전이는 M8(U5) 이벤트, ACTIVE/ENDED 전이는 M18(U6) 트리거를 수용(1차는 수신 계약만) | components §3.1, D14, D19, D20 |
| FD-U4-02 | 날짜 겹침 생성 단계 차단 | 활성 여행 ≤1을 생성·기간 변경 단계 겹침 차단으로 구조적 보장 — 소프트 삭제 여행은 판정 제외. 겹침 시 겹치는 여행 편집·삭제 바로가기 제공(UX 마찰 완화). 하드 제약 | D21/Δ3, U4 리스크 |
| FD-U4-03 | 예산 전체 총액 단일 정규화 | budgetTotal은 전체 총액(항공 제외)만 저장 — 1인·1일은 파생(BudgetAllocation, 조회 시점 계산), 카테고리 분배 없음. 예산은 솔버 하드 제약 아닌 LLM 소프트 가중치+숙소 필터 상한 | D26/Δ2, G37/G47, ADR-0008 |
| FD-U4-04 | 필수 방문지 2유형·사본 투입 | ANYTIME(포함만 보장·기본)/FIXED(시각 고정) 2유형. 저장 POI 투입은 PoiSnapshot 사본 복제 — 원본 SavedPlace 삭제와 독립(투입 시점 동결). 내부 용어 ↔ 화면 라벨 분리 | G129, D13, PRD 05-8 |
| FD-U4-05 | 필수 방문지 한도 일수 비례 | 상한 = 하루 3곳 × 여행 일수 — 초과 추가 차단. 추가 순서·유형 무관 상한 불변. 하드 제약 | G40 |
| FD-U4-06 | 거점 검증 분업 소비 | BaseAssignment 스키마·비중첩 판정 순수 함수는 U3(M4) 공급, 여행 내 비중첩 검증 실행·스마트 기본 거점 생성·전환일 식별·커버리지·거점 UI는 U4 소비. 계정 풀 미연결 숙소는 검증 대상 아님 | D15, unit-of-work U3 §제외·U4, CP1 |
| FD-U4-07 | 시간창 전 일자 총함수 | 날짜별 이용 가능 시각(기본 09:00~21:00, 첫날 도착·마지막날 출발 반영) — 미조정 일자도 기본 창 보유해 CP2로 전 일자 총함수 공급(공백 0) | G119, D29 |
| FD-U4-08 | 파괴적 변경 차단형 확인 | 기간 축소·거점 삭제·연결 숙소 변경은 서버가 영향 블록 산출·나열 후 확인(ImpactConfirmation) 없이 미적용. 날짜 변경 시 거점 비중첩·시간창 재검증 | G39, D15 |
| FD-U4-09 | 확정 후 필수 방문지 변경 미리보기 | 확정 일정의 필수 방문지 변경은 전/후 비교 미리보기(previewMustVisitChange) 승인 후 적용 — 생성 전 변경은 즉시 반영+재계산 신호 | G43, D28 |
| FD-U4-10 | CP2 공급 계약·경계 무결성 이중 방어 | TripContext(Trip·시간창·거점·필수 방문지·속성) 조립·공급. D21·D15·G40 위반 컨텍스트는 CP2 경계로 미공급 — U4 1차 보장+U5 방어적 재검증. U5 착수 후 필드 파괴적 변경 금지(추가만) | CP2, D14 |
| FD-U4-11 | 제목 자동 생성 결정성 | 미입력 제목은 '{여행지} N박M일' 결정적 생성(멱등)·금칙어 C3 검증 통과분만 저장 — 빈 제목 금지 | N6, C2 |

## 산출물

| 파일 | 내용 |
|---|---|
| `u4-trip/functional-design/domain-entities.md` | 엔티티 5종(Trip·TripDateWindow·BaseAssignment·MustVisit·BudgetAllocation) 정의·속성 표·관계·불변식·상태 전이표 + CP1 소비 참조(§6)·CP2 공급 TripContext(§7) |
| `u4-trip/functional-design/business-rules.md` | BR-U4-01~25 규칙 카탈로그 |
| `u4-trip/functional-design/business-logic-model.md` | 프로세스 플로우 4종 + Testable Properties(PBT-01 — 속성 6개) |
| `u4-trip/functional-design/frontend-components.md` | 여행 생성·목록·상세·거점·필수 방문지·기간 조정 화면 플로우·컴포넌트 계층·지도 U3 shared/map 재사용·M6 매핑·data-testid |

## 확장 규칙 컴플라이언스 요약 (Functional Design 스테이지 적용분)

| 규칙 | 상태 | 근거 |
|---|---|---|
| SECURITY-05 (입력 스키마 검증·금칙어) | 준수 | 여행 생성 필수 입력·날짜·국내 범위 검증(BR-U4-01·02·04)·제목 금칙어 C3(BR-U4-05)·필수 방문지 한도(BR-U4-16) |
| SECURITY-08 (소유권 검증 — IDOR) | 준수 | Trip·MustVisit·BaseAssignment 계정 귀속(INV-TRIP1 accountId·BR-U4-21) |
| PBT-01 (속성 식별) | 준수 | 속성 6개 식별(U4-P1~P6)·U4 DoD 5속성 커버리지 대조·하드 제약 3계열 대조 |
| RESILIENCY-10 (외부 어댑터 폴백) | N/A(본 유닛·스테이지) | U4는 신규 외부 연동 없음(장소 검색은 M7 재사용) — 솔버 미리보기 비가용 시 UpstreamUnavailable 보류 안내(BR-U4-18)만 명세, 수치는 NFR |
| 그 외 SECURITY/RESILIENCY 세부(알고리즘·수치·인프라) | N/A(본 스테이지) | 기술 중립 — NFR Requirements/Design·Infrastructure Design 소유 |
| 하드 제약(D37) | 준수(식별) | 날짜 겹침 차단(INV-TRIP1·U4-P1)·거점 구간 비중첩(INV-BASE-U4-1·U4-P3)·필수 방문지 한도(INV-MV2·U4-P4) 3계열 식별 — U5 솔버 하드 제약의 입력 무결성, 테스트 구현은 Code Generation |
| 계약 포인트 | 준수(식별) | CP1 소비자 측(SavedStay·SavedPlace 스키마 §6)·CP2 공급자 측(TripContext §7) 계약 테스트 대상 식별 |
