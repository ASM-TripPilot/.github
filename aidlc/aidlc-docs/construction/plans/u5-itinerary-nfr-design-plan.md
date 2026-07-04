# U5 AI 일정 생성·확정 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U5 (M8 Itinerary Generation · **C1 LLM Gateway** · **C2 Solver Engine** + `features/itinerary`·`shared/validation`)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U5(리스크·선결 **P6 LLM 비용**) · [stories.md](../../inception/user-stories/stories.md) Epic 5(US-E5-01~12) · [services.md](../../inception/application-design/services.md) **S1 성능 예산(§S1.4 단계별 배분 정본)**·§0(실패 처리·TX·성능 예산 원칙) · [requirements.md](../../inception/requirements/requirements.md) §6.1(D38)·§6.3(워크로드)·§6.4(D31·SECURITY-11)·§6.5(G181)·§6.8(G142)·§7.2(G106·G115·G194) · [component-dependency.md](../../inception/application-design/component-dependency.md) §2.2·2.3·§6(C1·C2 어댑터 격리) · [shared-infrastructure.md](../shared-infrastructure.md)(SI)
> 산출: `../u5-itinerary/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u5-itinerary/nfr-design/nfr-design-patterns.md`, `../u5-itinerary/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U5의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 실행으로 통합한다.** U5는 **LLM·솔버 연산이 무거운 최중량 유닛**(unit-of-work.md U5: 알고리즘 복잡도 1차 최대 · C1·C2는 U6·U7·U9 재사용 공통 자산 · 외부 연동 2 = LLM·TMap)이나, **인프라·플랫폼 골격은 전역 정본을 상속하고 유닛 증분(성능 예산 구체화·복원력 폴백 체인·LLM 비용 통제)만 소유**한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U5에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | SI 헤더 · U1 nfr-design §2 | 재사용(AWS 서울·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — SI §0 유닛 표: "U5 = NAT 아웃바운드(LLM·TMap) + LLM 비용 계측 메트릭(§8)". 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2(S-1~S-13) | Spring Boot Kotlin + PostgreSQL 재사용. U5는 **C1 LLM Gateway·C2 Solver·점진 응답·백그라운드 잡** 4지점만 구체화 |
| LLM 아키텍처 정본 | requirements.md D11·D31, SI §6 시크릿 프레임(§6 `LLM API 키(U5) — 예약`) | **단일 벤더 관리형 API·서버 경유·기능별 티어 분리(D11)** + 서버 재조회 경계(D31) 상속. 벤더 확정은 P6 |
| POI 하이브리드 캐싱 | requirements.md D13 · U3 nfr-requirements §4 | 상속 — U5는 M7 후보 풀(closed-set)의 **소비자**(재정의 없음). 캐시 계층 신규 없음 |
| 성능 예산 정본 | **services.md §S1.4** (첫 1일 5초 = 0.3+1.0+2.5+0.8+0.4 · 전체 20초) | **정본 상속·구체화** — U5 NFR이 이 단계 예산을 타임아웃·폴백 전환 시간과 함께 승격 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 · **동시 일정 생성 10** | 솔버 워커 CPU·태스크 사이징 산정 입력(과설계 금지 — 동시 10 상한) |
| 외부 어댑터 격리 | component-dependency.md §6 (C1 `LlmPort`·C2 `RoadDistancePort`) | 상속 — 어댑터 4요소 계약(타임아웃·재시도·서킷·폴백) 형식 재사용 |

**설계 지배 원칙**: U5 NFR 본질은 **① 성능(핵심 — D38 5초/20초의 단계 예산·타임아웃·점진 노출), ② 복원력(LLM 실패→결정론 솔버 폴백·전 경로 최소 일정), ③ 비용(LLM 호출 통제·티어·국외 이전)** 이다. C2 솔버는 **순수 인프로세스 결정론 연산(외부 무의존)** 이므로 신규 인프라 표면이 아니며, LLM은 SI가 예약한 NAT 아웃바운드·시크릿 프레임 내에서 종결된다(신규 AWS 리소스 사실상 0 — 과설계 금지).

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U5 범위 로드 (unit-of-work.md U5 — 목적·포함·제외·산출물·DoD·리스크·**선결 P6**)
- [x] Epic 5 스토리 로드 (US-E5-01 숙소기준·02 취향 closed-set·03 시간 하드제약·04 고정블록·05 이유설명·06 2보기·07 편집재검증·08 저장연계·09 실패폴백·10 3방식·11 숙소권역·12 확정)
- [x] 성능 예산 정본 로드 (**services.md §S1.4 단계별 배분** — 첫 1일 5초 = 0.3+1.0+**LLM 2.5**+0.8+0.4, 전체 20초, §0.3 타임아웃<예산 원칙)
- [x] 실패 처리 정본 로드 (services.md §0.1 — LLM→규칙점수/솔버 전경로→최소 일정/외부 POI→캐시 축소)
- [x] 확정 결정 로드 (D11 티어·D14 plan/current·D20 상태머신·D28 편집 재검증·D31 서버 재조회·D38 성능·G106 이동시간·G115 closed-set·G161 부분 초안)
- [x] 법률 정본 로드 (§6.5 G181 국외 이전·처리위탁 고지)
- [x] 공유 인프라 정본 로드 (SI §0 U5 표·§2.1 NAT·§6 시크릿 프레임·§8.2 A11·A12·A15 비용·§1.2 워커 모델)
- [x] 컴포넌트 의존 로드 (component-dependency §2.2 M8→C1·§2.3 M8→C2·§6 어댑터 격리 — LLM/TMap 폴백·국외 이전)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] **성능(핵심)** — D38 5초/20초의 U5 구체화: services.md §S1.4 단계 예산 상속, **LLM 타임아웃 2.5초(예산<)**, 솔버 시간(day1 0.8초·잔여 0.5초/일), 점진 노출(첫 1일 게이트·백그라운드 잔여), Plan-B(S2 10초)는 U6 — U5 아님 명시
- [x] **복원력(핵심)** — LLM 실패→규칙 점수 결정론 솔버 폴백(기본 모드 고지) / 솔버 전경로 실패→숙소+시각고정 필수방문지 최소 일정(RESILIENCY-10) / 외부 POI 실패→정본 캐시 축소 풀 / 워크로드 M8 LLM=Medium·저장=Critical(§6.3)
- [x] **비용(핵심)** — LLM 사용자별 rate-limit(SECURITY-11)·기능별 티어 D11·**전 일자 공용 1회 호출(재호출 없음)**·후보 풀 상한 프루닝·국외 이전 G181·SI A15 비용 예산·LLM 비용 계측 메트릭
- [x] 확장성 — 동시 생성 10 G142·솔버 워커 CPU·백그라운드 잔여 일자 처리 모델(요청 단위 비동기)
- [x] 보안 — LLM 컨텍스트 서버 재조회 경계 D31(U5=소비자)·closed-set 그라운딩 구조 보안 G115·SECURITY-11 rate-limit·출력 스키마 검증
- [x] 미결·선결 — **P6(LLM 벤더 계약·비용 산정 + 국외 이전 고지)** 연결, 벤더 중립 요건은 비차단
- [x] 확장 규칙 컴플라이언스 요약(준수/상속/N/A + 근거)

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] **C1 LLM Gateway** — 단일 벤더 클라이언트(D11)·기능별 티어 라우팅·타임아웃·출력 스키마 검증·rate-limit·비용 계측. **벤더는 P6 비용 산정 후, `LlmPort` 추상화 인터페이스·티어 라우팅 구조는 지금 확정**(대안+사유)
- [x] **C2 Solver Engine** — OPTW/TOPTW 구현: 라이브러리(OptaPlanner/Timefold 등) vs 자체 구현 대안 비교·결정론 폴백 모드·클라 경량 검증기 규칙 공유(D28 단일 명세 원천)
- [x] 점진 응답 스트리밍 — SSE vs WebSocket vs 폴링 택1(대안+사유, ALB 영향)
- [x] 백그라운드 잔여 일자 잡 — 요청 단위 비동기 워커(가상 스레드/Executor) vs 별도 워커/큐 대안(과설계 판단)
- [x] 각 선택 "재사용/증분" 명시 + 신규는 대안 1줄 + 사유

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U5-01 점진 생성·첫 1일 우선 반환(5초 게이트·백그라운드 잔여 20초)
- [x] PAT-U5-02 LLM 게이트웨이 폴백 체인(타임아웃 2.5초→규칙 점수→결정론 솔버)
- [x] PAT-U5-03 솔버 warm-start·오라클 검증(고정 블록 불변·확정 재검증)
- [x] PAT-U5-04 트랜잭셔널 처리(일자별 독립 TX·확정 스냅샷 원자성 — services §0.2)
- [x] PAT-U5-05 LLM rate-limit·기능별 티어 라우팅·비용 계측
- [x] PAT-U5-06 closed-set 그라운딩 구조 패턴(출력 스키마 검증·ID 화이트리스트 교차 검증)
- [x] 각 패턴: 문제/적용/U5 적용 지점/검증 기준(규칙 ID) 표기
- [x] 전역 상속 패턴(PAT-SEC-01·05·PAT-RES-01·02·03·PAT-PERF-02·PAT-OBS-01·03·OPS-01~03) "shared/U1 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] LLM 벤더 아웃바운드 — 기존 NAT(SI §2.1)·`sg-app` 443 재사용·시크릿 `external/llm`·`external/tmap`(SI §6 예약분)·타임아웃·**국외 이전 P6/G181**
- [x] 솔버 연산 — **기존 ECS 태스크 내 CPU 집약 연산**(C2 순수 인프로세스)·동시 10 G142 대비 태스크 CPU 사이징 재검토 판단(별도 워커 미도입 근거)
- [x] 점진 응답 채널 — SSE의 ALB 영향(유휴 타임아웃 60초 vs 20초 예산)·스티키니스 불요
- [x] 백그라운드 생성 잡 — 요청 단위 비동기(기존 스케줄러/큐 신규 없음)·C2 결정론 폴백은 순수 연산(외부 무의존)
- [x] LLM 비용 계측 메트릭 — SI §8.2 대시보드 LLM 비용 위젯(U5 확장 슬롯) 활성화·A11 LLM 쿼터·A15 비용 예산
- [x] SI 대비 델타 표(신규 리소스 — LLM 아웃바운드 시크릿·솔버 워커 CPU 사이징)
- [x] 배포 — shared deployment-architecture.md 재사용 갈음(서버 모듈 흡수·순수 JS 화면)
- [x] 컴플라이언스 요약

### 2.6 검증·마감

- [x] 추적 ID(NFR-U5-xx·PAT-U5-xx·SECURITY·RESILIENCY·PBT·D·G·GD·US-E5·CP·S1) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 규모 정합 점검 — 동시 10 기준(과설계 금지: ElastiCache·별도 솔버 워커·큐 미도입) 확인
- [x] 산출물 4종 생성 완료

---

## 3. 선결 과제 연결 (P6 — §8)

| 과제 | 시점 | 본 계획의 비차단 대응 |
|---|---|---|
| **P6 LLM 벤더 계약·비용 산정 + 국외 이전 고지 문안** | **U5 착수 전(벤더·티어 확정이 C1 설계 입력)** | 비용 상한이 **후보 풀 크기·호출 전략을 결정**(unit-of-work.md U5 리스크). 본 산출물은 **벤더 중립**으로 `LlmPort` 추상화·티어 라우팅·전 일자 1회 호출·rate-limit·비용 계측 구조를 확정(비차단). 벤더/모델/단가는 P6 후 remote config·시크릿으로 주입. 국외 이전 고지는 P7 개인정보처리방침 반영(G181) |
| P2 지도 API 약관(TMap 도로 거리) | U5 착수 전(U3에서 기완료 전제) | C2 `RoadDistancePort` 격리 — 실패 시 직선거리×우회계수(G106) 폴백. TMap은 C2 소유(U3는 미호출) |

---

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u5-itinerary/nfr-requirements/nfr-requirements.md` | U5 NFR 요구 — 성능(핵심)·복원력(핵심)·비용(핵심)·확장성·보안 + P6 선결 |
| `../u5-itinerary/nfr-requirements/tech-stack-decisions.md` | C1 LLM Gateway·C2 Solver·점진 응답·백그라운드 잡 스택(대안+사유) |
| `../u5-itinerary/nfr-design/nfr-design-patterns.md` | U5 패턴 6종(점진 생성·폴백 체인·warm-start·TX·rate-limit·closed-set) + 전역 상속 표기 |
| `../u5-itinerary/infrastructure-design/infrastructure-design.md` | U5 인프라 증분(LLM 아웃바운드·솔버 CPU 사이징) + SI 델타 표 + 배포 갈음 |

## 5. 다음 단계

- U5 Functional Design(선행) 또는 Code Generation: 본 산출물의 하드 제약 4계열(숙소 기준점·충돌 무배치·POI 그라운딩·계정 무결성)·성능 예산·폴백 체인·CP2(소비자)·CP3(공급자)·CP5(이벤트) 계약을 구현. PBT 1급 대상(영업시간 배치·이동 부등식·고정 블록 불변·폴백 결정성·상태 머신·직렬화 왕복)은 unit-of-work.md U5 DoD로 실행.
- 본 계획 §1의 전역 결정 상속은 U6 이후에도 동일 재사용(재질문 금지). C1·C2는 U6(재계획)·U7(회고)·U9(어시스턴트)가 재사용하는 공통 자산이므로 본 유닛에서 프로덕션 품질로 확정한다.
