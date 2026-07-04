# U4 여행 생성·필수 방문지 — 기술 스택 결정 (Tech Stack Decisions)

> 2026-07-04 · CONSTRUCTION · U4 NFR Requirements 산출물
> **전제**: 스택 대분류(RN Expo + Spring Boot Kotlin + PostgreSQL, D02)와 서버·클라이언트 라이브러리 정본(U1 [tech-stack-decisions.md](../../u1-foundation/nfr-requirements/tech-stack-decisions.md) S-1~S-13·C-1~C-8)은 **기확정**이다. U4는 **순수 CRUD·검증 유닛**이므로 프레임워크·인증·상태 관리·HTTP·PBT를 전부 U1에서 상속하고, **선택지가 있는 지점은 날짜/기간 표현·동시성 제어·예산 파생 산술 3곳뿐**이다.
> **재결정 금지**: 클라우드·인프라·서버 프레임워크·클라이언트 라이브러리는 SI·U1 정본 상속(nfr-requirements.md §7.1). U4는 외부 벤더가 0(외부 연동 없음)이므로 벤더 결정 자체가 없다. 본 문서는 **기존 스택 내** 선택지만 대안 비교 후 권고한다(과설계·신규 벤더 금지).

---

## 1. 결정 요약표

| # | 영역 | 결정 | 상속/증분 | 근거 ID |
|---|---|---|---|---|
| U4-S1 | 서버 CRUD 프레임워크 | **Spring MVC + JPA/Hibernate + Jakarta Validation** — U1 재사용 | 상속(U1 S-2·S-3·S-8) | requirements §2.3(D04) |
| U4-S2 | 날짜·기간 표현 | **java.time(`LocalDate`)** + PostgreSQL **`daterange` + `btree_gist`** 배제 제약(EXCLUDE) | 증분(기존 PostgreSQL 기능) | D21·D15, G42·G119 |
| U4-S3 | 겹침·비중첩 동시성 제어 | **DB EXCLUDE 제약(정본 가드)** + 낙관적 잠금 `@Version`(보조) | 증분 | D21·D15, PBT |
| U4-S4 | 예산 파생 산술 | **`BigDecimal`(JDK 내장) + 명시 반올림 정책** — 신규 라이브러리 0 | 증분(내장) | D26, PBT |
| U4-S5 | 마이그레이션 | **Flyway** — U1 파이프라인 재사용(신규 도구 0) | 상속(U1 S-9) | unit-of-work.md U4 |
| U4-C1 | 여행 생성·편집 폼 | **React Hook Form + Zod** — U1 재사용(입력 검증·금칙어 선반영) | 상속(U1 C-5) | SECURITY-05, N6 |
| U4-C2 | 날짜 범위·시간창 입력 | **Expo/RN 커뮤니티 날짜 피커**(day range·time) — 증분 최소 | 증분 | G42·G119, US-E4-01 |
| U4-C3 | 상태·HTTP | **Zustand + TanStack Query v5 + axios** — U1·U2 재사용 | 상속(U1 C-4·C-6) | D38 |
| U4-C4 | 클라이언트 PBT | **fast-check + Jest(jest-expo)** — U1 재사용(예산 파생·제목 결정성) | 상속(U1 C-7) | PBT-09 |
| U4-PD | 클라우드·시크릿·로그·알림 | **U1 PD-1~9 종결·SI 상속** — U4 외부 키·비동기 없음 | 상속 | SI, U1 infrastructure-design §0 |

---

## 2. 서버 CRUD — Spring MVC + JPA + Validation (U4-S1, 상속)

U4는 자체 데이터에 대한 표준 CRUD·검증이므로 **U1이 확정한 서버 스택을 그대로 재사용**한다 — 신규 프레임워크·라이브러리 0.

- **재사용**: Spring MVC(REST 컨트롤러)·Spring Data JPA/Hibernate(영속)·Jakarta Validation(요청 스키마 검증)·Spring Security 필터 체인(deny-by-default·소유권)·트랜잭션(`@Transactional`) — 전부 U1 S-2·S-3·S-8 정본.
- **U4 증분(코드 소유, 스택 아님)**: `modules/trip` 도메인(여행·거점 연결·필수 방문지·시간창·속성 애그리게이트), 검증기(겹침·비중첩·한도), CP2 여행 컨텍스트 계약 DTO. 이는 스택 결정이 아니라 도메인 구현이며 Functional Design·Code Generation 소관.
- **선정 사유(1줄)**: U4 서버 로직은 CRUD·검증에 국한(unit-of-work.md U4)되어 U1 스택으로 완전 충족 — 신규 서버 결정을 만들지 않는 것이 규모 정합(G142)·재작업 방지의 정답.

## 3. 날짜·기간 표현 — java.time + PostgreSQL daterange·btree_gist (U4-S2)

U4의 하드 제약(겹침 차단 D21·거점 비중첩 D15)은 **날짜 구간(range)의 중첩 판정**이 본질이다. 표현 방식을 확정한다.

- **애플리케이션**: **java.time `LocalDate`**(JDK 내장, 신규 0) — 여행 시작/종료·거점 체크인/아웃·시간창(`LocalTime`). 로직상 구간은 `[시작, 종료]` 폐구간(체크아웃일 취급 규칙은 Functional Design). 논리적 하루(자정 초과 귀속, D29)는 시간창이 아닌 일정 배치(U5) 소관.
- **저장·제약**: PostgreSQL **`daterange` 타입 + `btree_gist` 확장의 EXCLUDE 배제 제약**을 채택 —
  - 여행: `EXCLUDE USING gist (account_id WITH =, date_range WITH &&)` → **동일 계정 날짜 겹침을 DB가 원자적으로 차단**(D21 동시성 정본 가드, §4).
  - 거점: `EXCLUDE USING gist (trip_id WITH =, stay_range WITH &&)` → **동일 여행 거점 구간 비중첩**(D15).
- **대안 비교(1줄)**: (a) 애플리케이션 SELECT-후-검증만 두면 동시 생성 경합(TOCTOU)에서 둘 다 통과해 겹침 저장 발생 — DB 제약 없이는 불변식이 깨진다 / (b) 정수 컬럼 2개(start·end) + 앱 비교는 인덱스 겹침 조회가 비효율이고 경합에 취약 / (c) 트리거 함수 수동 구현은 EXCLUDE 제약의 재발명(정확성·성능 열위).
- **선정 사유**: `daterange` + `btree_gist` EXCLUDE는 **겹침 불변식을 DB 레벨에서 선언적·원자적으로 강제**해 동시성 경합을 근본 차단하고, GiST 인덱스로 겹침 조회가 O(log n)(NFR-U4-PERF-02). **신규 AWS 리소스 0** — `CREATE EXTENSION btree_gist`는 관리형 RDS PostgreSQL 기본 제공 확장으로 Flyway 마이그레이션 한 줄(app_migrate DDL 권한, SI §3)이면 활성화(infrastructure-design.md §2). 신규 라이브러리도 0(PostgreSQL 내장 타입·확장).

## 4. 겹침·비중첩 동시성 제어 — DB 제약 + 낙관적 잠금 (U4-S3)

D21·D15 불변식을 동시성 경합에서 지키는 계층을 확정한다(설계 상세는 nfr-design PAT-U4-01·02).

| 계층 | 역할 | 대상 |
|---|---|---|
| **① DB EXCLUDE 제약(정본 가드)** | 겹침·비중첩을 원자적으로 차단 — 경합 패배 트랜잭션은 제약 위반 예외 | 동일 계정 여행 겹침(D21), 동일 여행 거점 비중첩(D15) |
| **② 앱 사전 검증(UX)** | 제약 위반 전에 친절 오류·겹치는 여행 명시(unit-of-work.md U4 리스크) | 생성·수정 요청 처리 |
| **③ 낙관적 잠금 `@Version`(보조)** | 여행·거점 애그리게이트 동시 수정 경합(lost update) 방지 | 거점 구간 편집·기간 변경 동시 수정 |

- **선택**: **①이 정본 가드**(무결성), ②는 UX 오류 품질, ③은 애그리게이트 동시 수정 보호. 제약 위반(`org.postgresql` SQLState `23P01` 배제 위반)은 도메인 예외로 변환해 "겹치는 여행이 있어요 + 편집·삭제 바로가기"로 노출(침묵 실패·중복 저장 금지, §NFR-U4-DI-02).
- **대안 비교(1줄)**: 비관적 잠금(`SELECT FOR UPDATE`)으로 계정 단위 직렬화도 가능하나, 저빈도 쓰기(G142)에 락 경합·데드락 리스크를 얹는 과설계 — EXCLUDE 제약이 락 없이 불변식을 보장하므로 불필요.
- **선정 사유**: 하드 제약(D37)은 "테스트 100%·머지 차단"이라 **앱 로직만으로 보증할 수 없는 동시성 경합을 DB가 최종 보증**해야 한다. `@Version` 낙관적 잠금은 JPA 표준(U1 S-3 내)이라 신규 의존 0.

## 5. 예산 파생 산술 — BigDecimal (U4-S4)

- **선택**: **`java.math.BigDecimal`(JDK 내장)** — 전체 총액 정본에서 1인·1일 파생값 계산(D26). 부동소수점(`double`) 금지(반올림 누적 오차 → 왕복 손실 초과). **표시 반올림 정책 명문화**(예: `HALF_UP`, 원 단위) 및 파생값 비저장(조회 시 계산) 또는 저장 시 정본과 정합 검증(NFR-U4-DI-06·07).
- **대안 비교(1줄)**: 정수(원 단위) 산술도 가능하나 1인·1일 나눗셈의 나머지 처리를 수동 관리해야 해 파생 왕복 속성 테스트가 복잡 — BigDecimal + 명시 스케일·반올림이 왕복 손실 한계(PBT)를 선언적으로 만든다.
- **선정 사유**: 신규 라이브러리 0(JDK 내장), 파생 왕복·반올림 손실 한계(unit-of-work.md U4 PBT)를 정밀 산술로 보증.

## 6. 클라이언트 — 폼·날짜 입력·상태 (U4-C1~C4)

U4 클라이언트는 여행 생성·편집 폼과 거점·필수 방문지 편집 화면으로, **U1·U2가 확정한 폼·상태·HTTP 스택을 재사용**한다.

- **폼·검증 — React Hook Form + Zod(재사용, U1 C-5)**: 여행 생성 폼(제목·날짜·인원·예산·시간창·점진 취향 카드)·거점 편집·필수 방문지 선택. Zod 스키마로 클라이언트 즉시 검증(날짜 오늘 이후·≤30일 G42·시간창 시작<종료·예산 음수 차단)을 UX 선반영하되, **하드 제약은 서버 정본**(제출 시 서버 검증 신뢰, NFR-U4-PERF-03).
- **날짜 범위·시간창 입력 — Expo/RN 날짜 피커(증분 최소)**: 여행 기간(day range)·날짜별 시간창(time) 입력 컴포넌트. Expo 공식 또는 검증된 RN 커뮤니티 피커를 U1 version catalog에 추가(잠금 파일 커밋, SECURITY-10 상속). **대안 비교(1줄)**: 자체 캘린더 구현은 접근성·엣지(월 경계·최대 30일 클램프) 재발명 — 표준 피커로 마찰 최소화. 이 항목이 U4의 **유일한 신규 클라이언트 의존**이며 네이티브 모듈 여부는 순수 JS 피커 우선(EAS 재빌드 회피 — infrastructure-design.md §5).
- **상태·HTTP — Zustand + TanStack Query v5 + axios(재사용, U1 C-4·C-6)**: 여행 서버 상태는 TanStack Query(생성·수정 뮤테이션·낙관적 업데이트·목록 무효화), 폼 로컬 상태는 RHF, 전역 취향 기본값은 Zustand(계정 취향 → 여행 속성 기본값 제안 G134). **대안 비교(1줄)**: 신규 상태 라이브러리 도입은 U1 확정을 뒤집는 과설계 — 재사용이 정답.
- **PBT — fast-check(재사용, U1 C-7)**: 예산 총액↔파생 왕복·제목 자동 생성 결정성의 클라이언트 속성 테스트(§NFR-U4-DI-07·11). 서버측 하드 제약 PBT는 Kotest(백엔드, U1 S 정본).

---

## 7. 상속·비결정 재확인

| 항목 | U4 처리 | 정본 위치 |
|---|---|---|
| 서버 프레임워크·인증·DB 접근·트랜잭션·마이그레이션 | U1 S-2·S-3·S-8·S-9 상속(신규 0) | U1 tech-stack §2 |
| Expo SDK·TypeScript strict·prebuild·폼·상태·HTTP·PBT | U1 C-1·C-4·C-5·C-6·C-7 상속 | U1 tech-stack §3 |
| 클라우드·CI/CD·배포·시크릿·로그 스택 | GD-1~7·SI·U1 PD-1~9 종결 상속(재정의 금지) | SI, u1-foundation-nfr-design-plan.md §2 |
| 외부 벤더(지도·장소·OTA) | **U4 무관 — 외부 연동 0**(장소 검색은 M7/U3 재사용) | unit-of-work.md U4 |
| 등록 숙소·저장 POI 스키마 | U3(M4·M7) 소유 — U4는 CP1 소비자 | unit-of-work.md U4 CP1 |

**U4 신규 벤더·클라우드 결정 0 — 기존 스택 내 3개 지점(날짜/기간 `daterange`·동시성 EXCLUDE 제약·예산 BigDecimal)의 구체화와 클라이언트 날짜 피커 1종 추가만 유닛 증분이며, 전부 기존 PostgreSQL·JDK·Expo 범위 내다(과설계·신규 벤더 금지).** 확정 스키마(여행·거점·필수 방문지 테이블)는 Functional Design/Code Generation 소관이다.
