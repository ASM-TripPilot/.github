# U5 AI 일정 생성·확정 — Functional Design 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · 유닛 U5 (E5 스토리 12개 — US-E5-01~12) · **1차 최중량 유닛**(M8 Itinerary Generation + C1 LLM Gateway + C2 Solver Engine — 알고리즘 복잡도 1차 최대)
> 정본 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U5 섹션 + [unit-of-work-dependency.md](../../inception/application-design/unit-of-work-dependency.md) CP2(U4→U5 소비)·CP3(U5→U6 공급)·CP5(이벤트 공급 일부), [stories.md](../../inception/user-stories/stories.md) Epic 5, [components.md](../../inception/application-design/components.md) M8·C1·C2·§3.2(일정 상태 머신 정본), [component-methods.md](../../inception/application-design/component-methods.md) M8·C1·C2, [services.md](../../inception/application-design/services.md) S1, [requirements.md](../../inception/requirements/requirements.md) §7.1~7.2·D11·D13·D14·D20·D25/Δ1·D28·D29·D38, docs/PRD/06-AI일정생성.md, [u4-trip/functional-design/domain-entities.md](../u4-trip/functional-design/domain-entities.md)(TripContext=CP2 입력)

## 질문 처리 방침

**사용자 연속 진행 지시에 따라, 본 스테이지의 확인 질문은 requirements.md의 기확정 결정(D01~D38·Δ1~Δ10·G 제안 기본값 — 문서 승인으로 확정)으로 대체한다.** U5 소관 핵심 결정: D11(LLM 단일 벤더·티어), D13(POI 스냅샷), D14(plan 불변+current 가변), D20(확정 해제→재확정), D25/Δ1(소요시간 미표시), D28(솔버 서버/클라 검증 분리), D29(시간창), D38(성능 5초/20초), 그리고 §7.2의 G 기본값(G40·G46·G47·G48·G49·G50·G51·G106·G115·G119·G129·G161). Functional Design에서 추가 확정할 설계 결정은 아래 "설계 결정 목록"(FD-U5-xx)에 기록한다.

> **선결 과제 전제**: P6(LLM 벤더 계약·비용 산정 + 국외 이전 고지)는 **U5 착수 전 완료(C1 설계 입력)** 항목 — 비용 상한이 후보 풀 크기·호출 전략을 결정한다. P2(지도 API 약관·TMap 도로 거리)는 U3에서 기완료 전제(C2 이동시간 추정 입력). 본 설계는 C1·C2를 계약 인터페이스 수준(기술 중립)으로 정의하고, 티어·비용·알고리즘 구현체·remote config 수치(G106)는 NFR/Infrastructure Design으로 이관한다.

## 실행 체크리스트

- [x] 정본 문서 로드·대조 (unit-of-work U5 / dependency CP2·CP3·CP5 / stories E5 / components M8·C1·C2·§3.2 / component-methods M8·C1·C2 / services S1 / requirements §7.1~7.2 / PRD 06 / U4 TripContext)
- [x] 도메인 엔티티 정의 — [domain-entities.md](../u5-itinerary/functional-design/domain-entities.md)
  - [x] Itinerary — plan 불변 스냅샷(D14)·current 가변·상태 머신 DRAFT→EDITING→CONFIRMED→INTRIP(current 분리)·mode·version
  - [x] PlanSnapshot / CurrentItinerary — 확정 시 동결(불변) + 여행 중 가변본 이원 구조(D14)
  - [x] DaySchedule — 일자·적용 거점 참조·시간창 스냅샷(D29)
  - [x] ItinerarySlot — POI 참조(closed-set G115)·시각·체류·LOCK·sourceType·violations·llmReason/solverReason(D25 소요시간 없음)
  - [x] FixedBlock — 숙소 체크인/아웃·시각 고정형 필수 방문지(변경 불가·warm-start 보존 G46)
  - [x] GenerationSession — 상태 COLLECTING→DAY1_READY→GENERATED→FAILED·진행 스트림·부분 초안·취소(G161)
  - [x] SolverProblem / SolverSolution — 하드 제약·목적함수(C2 계약 인터페이스 수준)
  - [x] PoiScore — LLM 선호 점수(C1 계약 인터페이스 수준)
  - [x] TravelEstimate — 거리 기반·소요시간 내부전용(D25/Δ1)
  - [x] StayZoneRecommendation — 숙소 나중 등록 온램프(동선 무게중심)
  - [x] 엔티티별 속성 표(타입·제약·근거 ID)·관계·불변식·상태 전이표
  - [x] CP2 소비 참조 계약(TripContext) + CP3 공급 계약(일정 기준선) 명문화
- [x] 비즈니스 규칙 카탈로그 — [business-rules.md](../u5-itinerary/functional-design/business-rules.md) (BR-U5-01~44, 조건/동작/위반 시 처리/근거)
  - [x] 생성 진입·전제(숙소 기준점·NO_BASE 가드·날짜 검증·지오코딩 보류·나중 등록 온램프) / 후보 풀 그라운딩(closed-set G115·환각 0)
  - [x] LLM 점수화(C1 경량 티어·취향 중립 기본값·예산 소프트 가중치 G47) / 하드 제약 4종(영업시간·이동 부등식 G106·고정 블록·시간창 D29)
  - [x] 체류 범위(테이블 G51·min/rec/max·압축 표시) / 점진 노출·폴백(첫날 5초·20초·결정론·취소 부분 초안 G161)
  - [x] 3방식 생성(완전 AI/같이 고르기 G48/직접) / 편집 재검증(클라 경량+서버 확정 D28·자동 보정 최소 G49)
  - [x] LOCK·재생성 보존(warm-start G46) / 확정·해제(plan 동결 D14·해제→재확정 D20) / 이동 표시(거리만 D25/Δ1) / 숙소 전환일 편도(G50) / 숙소 권역 추천 / 추천·배치 이유
- [x] 비즈니스 로직 모델 — [business-logic-model.md](../u5-itinerary/functional-design/business-logic-model.md)
  - [x] 프로세스 플로우 5종(①일정 생성 ②편집 재검증 ③확정/해제 ④재생성 보존 ⑤숙소 권역 추천) + 같이 고르기 서브플로우
  - [x] Testable Properties(PBT-01) — 컴포넌트별 속성 식별 표(U5-P1~P12), U5 DoD 6속성 커버리지 대조, 하드 제약 4계열 대조, C2 oracle 테스트 명시
- [x] 프런트엔드 컴포넌트 설계 — [frontend-components.md](../u5-itinerary/functional-design/frontend-components.md) (생성 방식 선택·생성 진행·시간표/지도 2보기·편집·슬롯 LOCK/교체·같이 고르기·확정·숙소 권역 추천·shared/validation 경량 검증기·shared/map 재사용·M8/C2 매핑·data-testid)
- [x] 추적 ID(US/D/Δ/G/BR/FD/PBT) 일관 표기 검수
- [x] 확장 규칙(SECURITY-11·RESILIENCY-10·PBT-01·D37 하드 제약) 관련 조항 반영 검수 — 기술 중립 유지(알고리즘·티어·비용·remote config 수치는 NFR/Infra 이관)

## 설계 결정 목록 (본 스테이지 확정분)

| # | 결정 | 내용 | 근거 |
|---|---|---|---|
| FD-U5-01 | plan/current 이원 모델 정본 | Itinerary는 **PlanSnapshot(불변 동결본)**과 **CurrentItinerary(가변본)**를 분리 소유 — 확정 시 plan 동결, 여행 중 편집·Plan-B는 current에만 반영. 회고 대조(U7)·공개 스냅샷(U10)이 이 분리에 의존(CP3 척추) | D14, ADR-0013, CP3 |
| FD-U5-02 | 일정 상태 머신 4상태 | `DRAFT → EDITING → CONFIRMED → INTRIP` 확정 — DRAFT/EDITING↔CONFIRMED 왕복(확정↔해제 D20), CONFIRMED→INTRIP은 여행 ACTIVE 전이(M18) 수용 시점(이후 plan/current 분리 작동). 표 밖 전이 구조적 불가(전이 가드 PBT) | D14, D20, components §3.2 |
| FD-U5-03 | 하드 제약 4종 = C2 단일 소유 | 영업시간 내 배치·이동시간 부등식(G106)·고정 블록 불변·시간창(D29)을 **C2 Solver Engine이 단일 검증**하고, **사용자에게 보이는 모든 시각·순서는 C2 검증값만** — LLM 생성 임의 시각 구조적 미노출. 위반 배치는 결과에서 구조적 배제 | ADR-0008, PRD 06-3, D28, D29 |
| FD-U5-04 | POI 그라운딩 closed-set 구조 보장 | LLM(C1)은 후보 풀 canonical POI ID 목록에서만 선택 — 출력 스키마 검증+ID 화이트리스트 교차 검증을 C1 출구에서 강제해 **환각(후보 밖 POI) 0을 LLM 신뢰 불요로 달성**. U3 canonical ID 무결성(INV-POI1)이 전제 | G115, D37, SECURITY-11 |
| FD-U5-05 | 예산 소프트 가중치(하드 제약 아님) | 예산은 솔버 하드 제약으로 승격하지 않는다 — C1 점수화의 소프트 가중치(카테고리 무료/저/중/고 추정)로만 반영. 예산 초과 배치가 하드 차단되지 않음(단조성 PBT로 방향만 보장) | G37/G47, ADR-0008, INV-BUDGET2(U4) |
| FD-U5-06 | 점진 노출 2단 게이트 | 첫 1일 일정을 **일자별 독립 TX**로 저장·5초 내 우선 반환하고 나머지는 백그라운드 루프. GenerationSession 상태(COLLECTING→DAY1_READY→GENERATED)가 진행 정본. 20초 초과 시 결정론 솔버 단독 모드 전환 | D38, S1.2~S1.4, G161 |
| FD-U5-07 | 폴백 계단(시스템 최심) 6단 | LLM 실패→규칙 점수 결정론 솔버 / 설명만 실패→일정 정상+설명 생략 / 외부 API 실패→캐시·직선거리 추정 / 20초 초과→결정론 단독 / 전 경로 실패→숙소+시각 고정 필수만 최소 일정 / 저장 네트워크 오류→로컬 임시+재동기화. **전 경로 침묵 실패 금지** | ADR-0011, PRD 06-9, RESILIENCY-01/10 |
| FD-U5-08 | 편집 검증 이중화(규칙 단일 정본) | C2가 **버전 있는 검증 규칙 명세**(ConstraintSpec)를 발행 → 클라이언트 경량 검증기(shared/validation)는 즉시 검사(UX용 사본)·서버 C2는 저장 시 확정 검증(정본). 명세 버전 불일치 시 로컬 검사 보수적 비활성화(오판 방지) | D28, G163 |
| FD-U5-09 | warm-start 고정 블록 보존 | 재생성 시 LOCK 슬롯·체류 고정값·수동 추가 POI·시각 고정형 필수 방문지·거점 블록을 고정 블록으로 보존하고 나머지만 재배치 — 여행당 활성 일정 1개, 이전 상태는 changelog 추적. 보존 멱등성 PBT | G46/G136, CP3 시나리오 2 |
| FD-U5-10 | 소요시간 내부전용 캡슐화 | `TravelEstimate.internalDuration`은 솔버 이동 부등식·트리거 판정 내부 계산 전용 — **어떤 표시용 DTO에도 매핑 금지**, 표시 계층엔 `distance`(거리·수단·추정)만 전파. 이동 추정은 결정적(PBT) | D25/Δ1, ADR-0009 |
| FD-U5-11 | 생성 3방식 단일 파이프라인 수렴 | 완전 AI(전체 배치)·같이 고르기(슬롯 루프·기준점=직전 확정 슬롯 G48)·직접 만들기(검증부만)를 단일 생성 파이프라인의 변주로 수렴 — 도중 방식 전환 시 확정 슬롯 고정 블록 보존. 세 방식 모두 최종 시각·순서는 C2 검증값 | G48, D25, S1.1 |
| FD-U5-12 | C1/C2 계약 인터페이스 수준 정의 | C1(LLM Gateway)·C2(Solver Engine)는 U6·U7·U9 재사용 자산이므로 **계약 인터페이스(입출력 타입·불변식·실패 타입)까지만** 본 스테이지에서 정의하고, 벤더·티어·최적화 알고리즘(OPTW/TOPTW)·remote config 수치는 NFR/Infra 소유. C2는 상태 비보유 순수 계산 엔진(oracle 테스트 대상) | D11, D37, unit-of-work U5 §산출물 |

## 산출물

| 파일 | 내용 |
|---|---|
| `u5-itinerary/functional-design/domain-entities.md` | 엔티티 10종(Itinerary·PlanSnapshot·CurrentItinerary·DaySchedule·ItinerarySlot·FixedBlock·GenerationSession·SolverProblem/Solution·PoiScore·TravelEstimate·StayZoneRecommendation) 정의·속성 표·관계·불변식·상태 전이표 + CP2 소비/CP3 공급 계약 |
| `u5-itinerary/functional-design/business-rules.md` | BR-U5-01~44 규칙 카탈로그 |
| `u5-itinerary/functional-design/business-logic-model.md` | 프로세스 플로우 5종 + 같이 고르기 서브플로우 + Testable Properties(PBT-01 — 속성 12개) |
| `u5-itinerary/functional-design/frontend-components.md` | 일정 화면 플로우·컴포넌트 계층·shared/validation 경량 검증기·shared/map 재사용·M8/C2 매핑·data-testid |

## 확장 규칙 컴플라이언스 요약 (Functional Design 스테이지 적용분)

| 규칙 | 상태 | 근거 |
|---|---|---|
| SECURITY-11 (LLM 경계 — 서버 재조회·rate-limit·내부 지표 미주입) | 준수(설계 명세) | C1 서버 재조회 컨텍스트 주입(D31)·closed-set 검증(BR-U5-08)·내부 지표 구조적 미포함 — 티어·rate-limit 수치는 NFR |
| SECURITY-08 (소유권 검증 — IDOR) | 준수 | 일정·세션 계정/여행 귀속, C1 `resolveContext` 권한 밖 참조 `PermissionDenied`(계정 무결성 하드 제약 D31) |
| RESILIENCY-10 (외부 어댑터 타임아웃·서킷·폴백) | 준수(설계 명세) | LLM 타임아웃(2.5초)→규칙 점수 폴백·거리 API→직선거리 추정·6단 폴백 계단(BR-U5-19~24) — 구현 수치는 NFR |
| RESILIENCY-01 (LLM 경로 Medium — 폴백으로 지속) | 준수 | 결정론 솔버 폴백이 LLM 없이 항상 유효 해 보장(BR-U5-19·PBT U5-P3) |
| PBT-01 (속성 식별) | 준수 | 속성 12개 식별(U5-P1~P12)·U5 DoD 6속성 커버리지 대조·C2 oracle 테스트 명시 |
| 하드 제약(D37) — **본 유닛이 4계열 본체** | 준수(식별) | 숙소 기준점·충돌 무배치(영업시간·이동 부등식·시간창)·POI 그라운딩(closed-set)·계정 무결성(D31) 식별 — 테스트 구현·머지 차단은 Code Generation |
| 그 외 SECURITY/RESILIENCY 세부(알고리즘·티어·비용·remote config) | N/A(본 스테이지) | 기술 중립 — NFR Requirements/Design·Infrastructure Design 소유(C1 벤더·C2 최적화 알고리즘·G106 수치) |
