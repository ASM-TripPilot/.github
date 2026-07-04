# U4 여행 생성·필수 방문지 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U4 (M6 Trip Creation — 여행 CRUD·거점 연결·필수 방문지·시간창·예산 컨텍스트)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U4 · [stories.md](../../inception/user-stories/stories.md) Epic 4(US-E4-01~11) · [requirements.md](../../inception/requirements/requirements.md) §6·§7.2(G39·G40·G42·G43·G119)·§4(D15·D21·D26·D29·N6) · [shared-infrastructure.md](../shared-infrastructure.md)(SI)
> 산출: `../u4-trip/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u4-trip/nfr-design/nfr-design-patterns.md`, `../u4-trip/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U4의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 경량 실행으로 통합한다.** U4는 **순수 CRUD·검증 중심 유닛**(unit-of-work.md U4 예상 규모: 신규 엔티티 약 5·**외부 연동 0·스케줄러 잡 0·신규 인프라 0**)이므로, 인프라·플랫폼 결정은 **전부 전역 정본을 상속하고 유닛 증분만 기술**한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U4에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | [u1-foundation-nfr-design-plan.md](./u1-foundation-nfr-design-plan.md) §2 | 재사용(AWS·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — U4는 SI를 재정의하지 않음(SI §0 유닛 표: "U4 = 추가 없음, 기존 컴퓨트·DB"). 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2(S-1~S-13) | Spring Boot Kotlin + PostgreSQL 재사용. U4는 **서버 CRUD·검증 계층**만 구체화(JPA·Validation·Flyway 상속) |
| 성능 정본 | requirements.md §6.1(D38) — 일반 화면 전환 300ms | U4 CRUD API 예산은 U1 §1.2 CRUD 성능 규약(p95 200~300ms급) 상속 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 | U4 신규 컴퓨트·DB 산정 없음(여행·거점·필수방문지는 U1 app·RDS 흡수) |
| 운영 프로세스 | U1 nfr-design-patterns.md OPS-01~03 | 상속 — U4 신규 프로세스 없음 |

**설계 지배 원칙**: 과설계 금지(경량 CRUD 유닛) — U4 산출물은 전역 정본 참조 + 유닛 증분(**데이터 무결성 하드 제약** D21·D15·G40, 예산 파생 일관성 D26, 파괴적 변경 차단형 확인 G39·G43)에 국한한다. U4의 NFR 본질은 **U5 솔버 입력 무결성**이며, 신규 인프라 표면이 아니다.

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U4 범위 로드 (unit-of-work.md U4 — 목적·포함·제외·산출물·DoD·리스크)
- [x] Epic 4 스토리 로드 (stories.md US-E4-01 생성·02 숙소없이·03 거점·04 저장숙소·05 날짜반영·06 다중거점·07 다박·08 필수방문지·09 고정블록·10 OTA연결·11 제목)
- [x] NFR 정본 로드 (requirements.md §6.1 성능·§6.2 복원력·§6.3 워크로드 분류(여행 저장 Critical)·§6.4 보안·§6.6 PBT·§6.8 규모)
- [x] 확정 결정 로드 (D15 계정 레벨 풀·D21 겹침 차단·D26 전체 총액 예산·D29 시간창·N6 제목)
- [x] 제안 기본값 로드 (§7.2 G39·G40·G41·G42·G43·G119·G120·G129·G134·G158)
- [x] 공유 인프라 정본 로드 (SI §0 U4 표·§1 컴퓨트·§3 RDS·§7 IAM·§8 관측성·§9 환경)
- [x] U1 정본 로드 (U1 nfr §1.2 CRUD 성능·§3 가용성, nfr-design PAT-SEC-01·PAT-PERF-01·02·PAT-OBS-01·03, infrastructure-design §3 RDS 롤·Flyway)
- [x] U3 계약 로드 (CP1 소비자 측 — 등록 숙소·저장 POI 스키마)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] 성능 — 여행 생성·거점 연결·필수 방문지 변경 응답(CRUD 300ms급 — U1 §1.2 상속)
- [x] 데이터 무결성(핵심) — 날짜 겹침 차단 D21 동시성(동일 계정 여행 생성 경합), 거점 비중첩 트랜잭션 경계 D15, 예산 파생 일관성 D26
- [x] 데이터 무결성 — 필수 방문지 사본 독립(G129)·일수 비례 한도(G40)·국내 좌표 범위(G120)
- [x] 가용성 — M6 여행 저장 Critical 경로(§6.3) — RDS 의존, 오프라인 조회 미보장(D24)
- [x] 보안 — 전 API 인증·소유권 검증(SECURITY-08 IDOR)·여행/숙소 계정 귀속(D15·D22), 금칙어·입력 검증(SECURITY-05·N6)
- [x] 규모 — 계정당 여행 수 가정(활성 ≤1·이력 소량, G142)
- [x] 미결 표 — 신규 인프라 없음(대기 항목 0) 확인
- [x] 확장 규칙 컴플라이언스 요약(준수/상속/N/A + 근거)

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] 서버 CRUD — Spring MVC + JPA/Hibernate + Jakarta Validation(U1 상속) 명시
- [x] 날짜·기간 처리 — java.time(LocalDate·daterange) + PostgreSQL `daterange` + `btree_gist` EXCLUDE 제약(증분)
- [x] 동시성 제어 — DB EXCLUDE 제약(정본 가드) + 낙관적 잠금 `@Version`(보조) 비교 후 권고
- [x] 예산 파생 계산 — BigDecimal 정밀·반올림 정책(신규 라이브러리 0)
- [x] 클라이언트 폼 — React Hook Form + Zod(U1 C-5 상속) + 날짜 범위·시간창 입력(증분 최소)
- [x] 마이그레이션·상태·HTTP·PBT — U1 Flyway·Zustand·TanStack Query·fast-check 상속
- [x] 각 항목 "재사용" 명시, 신규 있으면 사유(대안 1줄)

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U4-01 여행 날짜 겹침 차단 동시성 제어(D21 — 활성 여행 ≤1)
- [x] PAT-U4-02 거점 구간 비중첩 검증 트랜잭션(D15)
- [x] PAT-U4-03 예산 단일 정본(전체 총액)→파생 일관성(D26)
- [x] PAT-U4-04 필수 방문지 사본 복제·일수 비례 한도 불변식(G129·G40·G158)
- [x] PAT-U4-05 파괴적 변경 차단형 확인(기간 축소·거점 삭제 G39)
- [x] PAT-U4-06 필수 방문지 변경 재계산 미리보기 핸드오프(G43 — U5 경계)
- [x] 각 패턴: 문제/적용/U4 적용 지점/검증 기준 표기
- [x] 전역 재사용 패턴(보안 PAT-SEC-01·복원력 PAT-RES-01·02·03·성능 PAT-PERF-01·02·관측성 PAT-OBS-01·03) "shared/U1 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] 여행·거점·필수 방문지 API 배치 — 기존 ALB `/api/*`·ECS 대상 그룹 흡수(신규 리소스 0)
- [x] RDS 스키마 확장(약 5 테이블) + `btree_gist` 확장 활성화(Flyway) — 기존 RDS·Flyway 재사용
- [x] 외부 연동·스케줄러 잡·시크릿·캐시 — U4 신규 없음(참조만)
- [x] SI 대비 델타 표(신규 AWS 0 / 변경: RDS 스키마·확장·ALB 경로 흡수)
- [x] 배포 — shared deployment-architecture.md 재사용·U4 델타 없음(순수 JS 화면+서버 모듈 흡수) 절
- [x] 컴플라이언스 요약

### 2.6 검증·마감

- [x] 추적 ID(NFR-U4-xx·PAT-U4-xx·SECURITY·RESILIENCY·PBT·D·G·N·GD·US-E4·CP) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 과설계 점검 — U4 증분이 데이터 무결성·검증에 국한, 전역 정본 참조를 넘지 않음 확인
- [x] 산출물 4종 생성 완료

---

## 3. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u4-trip/nfr-requirements/nfr-requirements.md` | U4 NFR 요구 — 성능·데이터 무결성(핵심)·가용성·보안·규모 + 미결 0 확인 |
| `../u4-trip/nfr-requirements/tech-stack-decisions.md` | U4 서버 CRUD·날짜/기간·동시성·예산 파생 스택(대부분 U1·U3 상속, 증분 최소) |
| `../u4-trip/nfr-design/nfr-design-patterns.md` | U4 패턴 6종(무결성·검증 중심) + 전역 상속 표기 |
| `../u4-trip/infrastructure-design/infrastructure-design.md` | U4 인프라 증분(신규 AWS 0) + SI 델타 표 + 배포 갈음 절 |

## 4. 다음 단계

- U4 Functional Design(선행) 또는 Code Generation: 본 산출물의 하드 제약(D21·D15·G40)·예산 파생·CP2(U4→U5) 여행 컨텍스트 계약을 구현. PBT 속성(겹침 후 활성 ≤1·거점 비중첩·예산 왕복·한도 불변·제목 결정성)은 unit-of-work.md U4 DoD로 실행.
- 본 계획 §1의 전역 결정 상속은 U5 이후에도 동일 재사용(재질문 금지).
