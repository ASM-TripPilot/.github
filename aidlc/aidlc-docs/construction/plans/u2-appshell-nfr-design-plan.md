# U2 앱 셸·홈·내비게이션 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U2 (클라이언트 중심: 스플래시 분기·5탭 셸·홈 대시보드·장소 우선 온램프 + server/app 부트스트랩 API)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U2 · [stories.md](../../inception/user-stories/stories.md) Epic 2(US-E2-01~06) · [requirements.md](../../inception/requirements/requirements.md) §6·§7.1(G5·G6·G7)·§5.3(N4) · [shared-infrastructure.md](../shared-infrastructure.md)(SI)
> 산출: `../u2-appshell/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u2-appshell/nfr-design/nfr-design-patterns.md`, `../u2-appshell/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U2의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 경량 실행으로 통합한다.** U2는 클라이언트 중심·경량 유닛(unit-of-work.md U2 예상 규모: 신규 엔티티 약 1·외부 연동 0)이므로, 인프라·플랫폼 결정은 **전부 전역 정본을 상속하고 유닛 증분만 기술**한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U2에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | [u1-foundation-nfr-design-plan.md](./u1-foundation-nfr-design-plan.md) §2 | 재사용(AWS·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — U2는 SI를 재정의하지 않음. 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §3(C-1~C-8) | RN Expo + Spring Boot Kotlin 재사용. U2는 **내비게이션 셸 계층**만 구체화 |
| 성능 정본 | requirements.md §6.1(D38) — 일반 화면 전환 300ms·스플래시 세션 검증 3초(G5) | U2 관점 예산 배분으로 전개 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 | U2 신규 컴퓨트·DB 산정 없음(부트스트랩은 U1 app 재사용) |
| 운영 프로세스 | U1 nfr-design-patterns.md OPS-01~03 | 상속 — U2 신규 프로세스 없음 |

**설계 지배 원칙**: 과설계 금지(경량 유닛) — U2 산출물은 전역 정본 참조 + 유닛 증분(스플래시 부트스트랩·탭 셸·홈 카드 로딩·강제 업데이트·딥링크)에 국한한다.

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U2 범위 로드 (unit-of-work.md U2 — 목적·포함·제외·산출물·DoD·리스크)
- [x] Epic 2 스토리 로드 (stories.md US-E2-01 스플래시·02 홈·03 5탭·04 탭바 숨김·05 장소 온램프·06 강제 업데이트)
- [x] NFR 정본 로드 (requirements.md §6.1 성능·§6.2 복원력·§6.3 워크로드 분류·§6.4 보안·§6.7 관측성·§6.8 규모)
- [x] 제안 기본값 로드 (§7.1 G5 스플래시 타임아웃·G6 탭 보존·G7 딥링크·§5.3 N4 강제 업데이트)
- [x] 공유 인프라 정본 로드 (SI §1 컴퓨트·§2.3 ALB·§8 관측성·§9 환경 — U2 소비 요소 확인)
- [x] U1 정본 로드 (U1 nfr-requirements §1.2·§1.3 부트스트랩 API·§4.3 deny-by-default, nfr-design PAT-RES-02·03·PAT-PERF-01, infrastructure-design §1 ALB 라우팅)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] 성능 — 스플래시 부트스트랩 3초 예산 배분(세션 검증+최소 버전+재동의 3검사 1왕복 집계, G5)
- [x] 성능 — 탭 전환 300ms·탭 상태 메모리 보존(G6, D38)
- [x] 성능 — 홈 카드 병렬 로딩·부분 실패 허용(가용 카드 우선·침묵 실패 금지)
- [x] 가용성 — 부트스트랩 API 경로(U1 Auth 의존·Critical 상속) + 오프라인 스플래시 로컬 토큰 폴백(G5)
- [x] 보안 — 부트스트랩도 인증 규약(SECURITY-08) + 최소 버전 비인증 허용 경계 명시(NFR-U1-SEC-16 상속)
- [x] 보안 — 딥링크 진입 검증(비로그인 진입은 초대·공유 딥링크 한정 D22, 라우팅 가드)
- [x] 클라이언트 NFR — 탭 상태 보존 수명(G6)·오프라인 조회 미보장(D24) 캐시 정책 경계
- [x] 미결 표 — 전역 결정으로 U2 신규 인프라 결정 없음(대기 항목 0) 확인
- [x] 확장 규칙 컴플라이언스 요약(준수/N/A + 근거)

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] 내비게이션 라이브러리 — Expo Router vs React Navigation 비교(탭 스택 상태 보존·딥링크 G7) → 권고
- [x] 상태 관리 — 전역 세션·탭 상태 구체화(U1 Zustand+TanStack Query 상속)
- [x] 스플래시 — expo-splash-screen(부트스트랩 게이트 유지)
- [x] 버전 체크 — 앱 빌드 버전 취득(expo-constants/app config) + expo-updates 관계 정리
- [x] 부트스트랩 서버측 — U1 app 모듈 재사용(신규 서버 스택 없음) 명시
- [x] 각 선택에 대안 1줄 + 사유 표기

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U2-01 스플래시 부트스트랩 단일 왕복 집계
- [x] PAT-U2-02 클라이언트 오프라인 우선 세션 폴백(G5)
- [x] PAT-U2-03 탭별 독립 내비 스택 + 상태 보존(G6)
- [x] PAT-U2-04 홈 카드 병렬 로딩·부분 실패 격리(스켈레톤·가용 카드 우선)
- [x] PAT-U2-05 강제 업데이트 게이트(N4)
- [x] PAT-U2-06 딥링크 라우팅(G7)
- [x] 각 패턴: 문제/적용/U2 적용 지점/검증 기준 표기
- [x] 전역 재사용 패턴(보안 PAT-SEC-01·복원력 PAT-RES-01·02·03·관측성 PAT-OBS-01) "shared/U1 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] 부트스트랩 API 배치 — 기존 ALB `/api/*`·ECS 대상 그룹 흡수(신규 리소스 0)
- [x] 최소 지원 버전 값 저장·조회 방식 — 원격 구성(앱 config vs 서버 파라미터) 비교 후 권고
- [x] CloudFront·정적 자산 해당 없음(모바일 EAS/스토어 배포) 명시
- [x] 홈 인기 장소 집계 잡 — U3/M7 소유(U2 아님 — 참조만)
- [x] SI 대비 델타 표(신규 0 / 변경 극소)
- [x] 배포 — shared deployment-architecture.md 재사용·U2 델타 없음(EAS 빌드에 U2 화면 포함) 절
- [x] 컴플라이언스 요약

### 2.6 검증·마감

- [x] 추적 ID(NFR-U2-xx·PAT-U2-xx·SECURITY·RESILIENCY·D·G·N·GD·US-E2) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 과설계 점검 — U2 증분이 전역 정본 참조를 넘지 않음 확인
- [x] 산출물 4종 생성 완료

---

## 3. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u2-appshell/nfr-requirements/nfr-requirements.md` | U2 NFR 요구 — 성능·가용성·보안·클라이언트 NFR + 미결 0 확인 |
| `../u2-appshell/nfr-requirements/tech-stack-decisions.md` | U2 클라이언트 셸 스택 — 내비게이션·상태·스플래시·버전 체크(서버는 U1 재사용) |
| `../u2-appshell/nfr-design/nfr-design-patterns.md` | U2 패턴 6종 + 전역 상속 표기 |
| `../u2-appshell/infrastructure-design/infrastructure-design.md` | U2 인프라 증분(신규 0) + SI 델타 표 + 배포 갈음 절 |

## 4. 다음 단계

- U2 Code Generation: 본 산출물의 부트스트랩 API 계약·스플래시 분기 결정 함수(PBT 대상)·5탭 셸·홈 카드 슬롯 스키마를 구현.
- 본 계획 §1의 전역 결정 상속은 U3 이후에도 동일 재사용(재질문 금지).
