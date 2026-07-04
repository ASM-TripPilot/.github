# U1 기반·계정·온보딩 — NFR Design 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U1 (M1 Auth, M2 User Profile, C3 Content Moderation + 스캐폴드·전역 보안 설정)
> 입력: [nfr-requirements.md](../u1-foundation/nfr-requirements/nfr-requirements.md) · [tech-stack-decisions.md](../u1-foundation/nfr-requirements/tech-stack-decisions.md) · [business-logic-model.md](../u1-foundation/functional-design/business-logic-model.md) · requirements.md §6(정본) · Resiliency Baseline(RESILIENCY-03·04·05·06·14·15)
> 산출: `../u1-foundation/nfr-design/nfr-design-patterns.md`, `../u1-foundation/nfr-design/logical-components.md`

---

## 1. 질문 대체 선언

**본 스테이지에서 신규 사용자 질문은 없다.** NFR Design 단계가 물어야 할 사용자 결정(RESILIENCY-03·04·14·15의 "AI가 대신 결정 금지" 항목 + 클라우드 벤더)은 NFR Requirements 단계에서 분리 발행한 질문 파일 [`u1-nfr-infra-questions.md`](./u1-nfr-infra-questions.md)에 대한 **사용자 답변(2026-07-04, 전 7문항 기입 완료)과 기확정 결정(D36·D37·D38·G5·G22, requirements.md §6)으로 대체**한다. 아래 §2가 그 답변의 정본 요약이며, 이후 전 유닛(U2~U8)의 NFR Design·Infrastructure Design은 동일 답변을 재사용한다(재질문 금지 — 프로젝트 전역 결정).

## 2. 전역 확정 결정 7건 (u1-nfr-infra-questions.md 답변 — 프로젝트 전역 정본)

| # | 결정 항목 | 사용자 답변 | 추적 |
|---|---|---|---|
| GD-1 | 클라우드 벤더 | **AWS** (서울 리전 전제 — Multi-AZ·관리형 PostgreSQL·오브젝트 스토리지+CDN·이메일 발송) | Q1=A |
| GD-2 | 변경 관리 프로세스 | 공식 프로세스 없음 — **AI-DLC 경량 프로세스 제안 채택**(변경 기록 + PR 승인 게이트 + 롤백 노트). B안(기존 조직 프로세스) 아님 | Q2=A, RESILIENCY-03 |
| GD-3 | CI/CD 도구 | **GitHub Actions** (저장소 GitHub, 모노레포 워크플로) | Q3=A, RESILIENCY-04 |
| GD-4 | 롤백 메커니즘 | **이전 버전 아티팩트 재배포(버전 고정 롤백)** + DB는 **forward-only 마이그레이션·호환 규칙** 병행(스키마 역방향 롤백 없음) | Q4=A, RESILIENCY-04 |
| GD-5 | 배포 스타일 | **롤링 배포** — Multi-AZ 2+ 인스턴스 점진 교체, 무중단 | Q5=A, RESILIENCY-04 |
| GD-6 | 인시던트 대응 | 공식 프로세스 없음 — **AI-DLC 경량 프로세스 제안 채택**(알림 라우팅 + 대응 절차 + 사후 회고 템플릿) | Q6=A, RESILIENCY-15 |
| GD-7 | 복원력 테스트 | **Operations 단계 이연** — 현 시점에는 테스트 시나리오 문서만 확보, 실행은 Operations | Q7=A, RESILIENCY-14 |

정합성 메모: GD-4/GD-5는 NFR-U1-AV-05(무중단 배포 내성 — 토큰 무상태·세션 정본 DB)와 정합하고, GD-7의 시나리오 대상은 §6.2 확정 토폴로지(Multi-AZ, DR=Backup & Restore, RTO/RPO 시간 단위 — CQ4=A)를 그대로 따른다. 이로써 nfr-requirements.md §8의 PD-7(CI/CD·롤백·배포)·PD-8(변경 관리·인시던트·복원력 테스트)은 **해소**되고, PD-1~6·9는 Infrastructure Design 대기가 유지된다(본 산출물에서는 AWS 매핑을 참조로만 표기).

## 3. 실행 체크리스트

### 3.1 준비

- [x] NFR Requirements 산출물 로드 (nfr-requirements.md — U1 NFR 요구 전수, §8 미결 표)
- [x] 기술 스택 결정 로드 (tech-stack-decisions.md — Spring Security 필터 체인 정본, argon2id, Kotest/fast-check)
- [x] Functional Design 플로우·PBT 속성 로드 (business-logic-model.md — FLOW-1~7, U1-P1~P17)
- [x] NFR 정본 로드 (requirements.md §6 — 성능·가용성·워크로드 분류·보안·관측성·규모)
- [x] Resiliency Baseline 검증 기준 로드 (RESILIENCY-03·04·05·06·14·15 — 사용자 결정 항목 답변 대조)
- [x] 사용자 답변 반영 (u1-nfr-infra-questions.md 7문항 → §2 전역 결정 확정)

### 3.2 패턴 카탈로그 설계 (nfr-design-patterns.md)

- [x] 보안 패턴 — 인증 필터 체인(deny-by-default, SECURITY-08) 설계
- [x] 보안 패턴 — 토큰 회전·재사용 감지(D36, FLOW-5) 설계
- [x] 보안 패턴 — 브루트포스 점진 지연·레이트리미터(NFR-U1-SEC-07·08·09) 설계
- [x] 보안 패턴 — 비밀번호 해시 전략(argon2id + DelegatingPasswordEncoder 재해시 경로) 설계
- [x] 보안 패턴 — 시크릿 관리(하드코딩 금지·주입 인터페이스, AWS Secrets Manager 참조) 설계
- [x] 복원력 패턴 — 외부 의존 타임아웃+재시도+서킷 브레이커+폴백 계약(RESILIENCY-10) 설계
- [x] 복원력 패턴 — 헬스체크 shallow `/health` + deep `/health/deep`(RESILIENCY-06) 설계
- [x] 복원력 패턴 — 우아한 성능 저하(소셜 IdP 장애 시 이메일 로그인 유지 등, NFR-U1-AV-04) 설계
- [x] 성능 패턴 — 토큰 검증 무상태(NFR-U1-AV-01)·커넥션 풀 원칙 설계
- [x] 관측성 패턴 — 구조화 로깅(상관 ID·PII 마스킹, SECURITY-03) 설계
- [x] 관측성 패턴 — 보안 이벤트 감사 로그(SECURITY-14 이벤트 목록) 설계
- [x] 관측성 패턴 — 메트릭·알람 대상 정의(RESILIENCY-05) 설계
- [x] 운영 프로세스(전역 정본) — 경량 변경 관리 프로세스 제안(GD-2, RESILIENCY-03) 작성
- [x] 운영 프로세스(전역 정본) — 경량 인시던트 대응 프로세스 제안(GD-6, RESILIENCY-15) 작성
- [x] 운영 프로세스(전역 정본) — 복원력 테스트 시나리오 문서(GD-7, RESILIENCY-14 — Operations 이연 명시) 작성
- [x] 각 패턴에 문제/적용/U1 적용 지점/검증 기준(SECURITY·RESILIENCY 규칙 ID) 표기 + 전역 재사용 여부 표시

### 3.3 논리 컴포넌트 설계 (logical-components.md)

- [x] API 게이트웨이/로드밸런서 계층 정의 (Multi-AZ·롤링 배포 정합)
- [x] 인증 필터 정의 (deny-by-default 화이트리스트)
- [x] 레이트리미터 정의 (로그인·재발송·닉네임 확인)
- [x] 세션/토큰 저장 정의 (무상태 액세스 + 리프레시 저장소 — PostgreSQL 정본)
- [x] 이메일 발송 큐 정의 (비동기·재시도, MailDeliveryPort 뒤)
- [x] 시크릿 저장소 정의 (주입 인터페이스, AWS 참조)
- [x] 법정 로그 저장소 정의 (append-only — 앱 역할 DELETE/UPDATE 불가, N2)
- [x] 감사 이벤트 파이프라인 정의 (SECURITY-14 이벤트 → 저장·알림 라우팅)
- [x] 메트릭·알람 정의 (RESILIENCY-05·09 임계)
- [x] 트랜잭셔널 아웃박스 릴레이 정의 (common/core — services.md §0.2 at-least-once·멱등 정합)
- [x] 각 컴포넌트에 책임/통합 지점/장애 모드·대응 표기 + AWS 매핑은 참조만(확정은 Infrastructure Design) 명시

### 3.4 검증·마감

- [x] RESILIENCY-03·04·14·15 검증 기준 대조 — 사용자 답변 기반 충족 확인(§2)
- [x] RESILIENCY-05·06 검증 기준 대조 — 패턴 카탈로그 관측성·헬스체크 절 충족 확인
- [x] SECURITY 규칙 커버리지 대조 — nfr-requirements.md §7 표의 '대기' 항목 해소분 반영
- [x] 추적 ID(SECURITY-xx·RESILIENCY-xx·NFR-U1-xx·D·G·FLOW·U1-P) 전 항목 표기 확인
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 산출물 2종 생성 완료

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u1-foundation/nfr-design/nfr-design-patterns.md` | 패턴 카탈로그 — 보안 5 + 복원력 3 + 성능 2 + 관측성 3 + 운영 프로세스(전역 정본) 3 |
| `../u1-foundation/nfr-design/logical-components.md` | 논리 컴포넌트 10종 — 책임/통합 지점/장애 모드·대응, AWS 매핑 참조 |

## 5. 다음 단계

- U1 Infrastructure Design: PD-1~6·9(메일 제품·시크릿 매니저 제품·KMS·로그 스택·RDS 세부·유출 조회 서비스·캐시 여부)를 GD-1(AWS) 전제로 확정
- 본 문서 §2의 전역 결정은 U2 이후 전 유닛 NFR Design에서 재사용(재질문 금지)
