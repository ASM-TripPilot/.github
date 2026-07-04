# U1 기반·계정·온보딩 — NFR Requirements 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · Per-Unit Loop: U1 → NFR Requirements 스테이지
> 근거 정본: [requirements.md](../../inception/requirements/requirements.md) §2.3(D02)·§6 전체, [unit-of-work.md](../../inception/application-design/unit-of-work.md) U1 섹션, [components.md](../../inception/application-design/components.md) M1·M2·C3·공통 설계 규약, `security-baseline.md`(SECURITY-01~15), `property-based-testing.md`(PBT-09)

## 질문 처리 방침

**본 스테이지의 NFR 질문은 신규 질문 없이 기확정 결정으로 대체한다.** Requirements Analysis에서 NFR이 Comprehensive 깊이로 이미 확정되었기 때문이다(requirements.md §6 — 성능 D38, 가용성·복원력 CQ4/RESILIENCY, 워크로드 분류 §6.3, 보안 CQ1=Security Baseline 전체 강제, 테스트 D37/PBT, 관측성 §6.7, 규모 §6.8). 기술 스택도 D02(RN Expo + Spring Boot Kotlin + PostgreSQL)로 기확정이므로 tech-stack-decisions.md는 **확정 스택의 U1 구체화**(버전대·라이브러리 선정)만 수행한다.

**단, 아래 항목은 사용자 결정이 필요하므로 별도 질문 파일 `u1-nfr-infra-questions.md`로 분리해 사용자 결정을 대기한다** (requirements.md §9 이관 결정과 정합):

| 대기 항목 | 이관 근거 |
|---|---|
| 클라우드 벤더·관리형 서비스 선정(메일 발송·시크릿 매니저·KMS·중앙 로그 포함) | §9 → Infrastructure Design |
| CI/CD 도구·롤백 메커니즘·배포 스타일 | §9 → NFR Design (RESILIENCY-04) |
| 변경 관리 프로세스 | §9 → NFR Design (RESILIENCY-03) |
| 인시던트 대응 프로세스 | §9 → NFR Design (RESILIENCY-15) |
| 복원력 테스트 접근 | §9 → NFR Design (RESILIENCY-14) |

위 항목이 결정될 때까지 nfr-requirements.md의 해당 부분은 **'미결 — 대기'로 명시 표기**하고, 벤더 중립 요구사항(Port 격리·요건 명세)까지만 확정한다.

## 실행 체크리스트

### 1단계 — 입력 로드
- [x] requirements.md §2.3(스택 D02)·§3(확장 구성)·§6.1~6.8(NFR 전체)·§9(이관 결정) 로드
- [x] unit-of-work.md U1 섹션(목적·범위·산출물·DoD·리스크·선결 과제) 로드
- [x] components.md 공통 설계 규약(§2)·M1 Auth·M2 User Profile·C3 Content Moderation 로드
- [x] security-baseline.md SECURITY-01~15 로드 — U1 적용 지점 식별
- [x] property-based-testing.md PBT-09(프레임워크 요건) 로드

### 2단계 — U1 NFR 전개 (nfr-requirements.md)
- [x] 성능: 로그인·토큰 갱신·취향 저장 등 U1 API 응답 목표 제안 — 화면 전환 300ms(D38) 지원 관점, 스플래시 세션 검증 3초 타임아웃(G5)과의 관계 정리
- [x] 규모: DAU 1천(G142/§6.8) 기준 인증 처리량 산정(부트스트랩·토큰 갱신·로그인·가입 스파이크 헤드룸)
- [x] 가용성: M1=Critical(§6.3)·Multi-AZ 전제(§6.2)·인증 불가 시 전 기능 마비 → 토큰 검증 무상태성·세션 캐시 요구사항 도출
- [x] 보안(핵심): SECURITY-12 전 항목 U1 구체화(적응형 해시 요건, 유출 비밀번호 검증 방식, 세션 무효화 매트릭스, 브루트포스 점진 지연 수치, MFA=어드민 후속)
- [x] 보안: SECURITY-05(입력 검증)·SECURITY-08(deny-by-default 인증 미들웨어) U1 구체화
- [x] 보안: SECURITY-03/14(로깅 — PII 금지, 동의 증적·위치 법정 로그 append-only 6개월+, 감사 이벤트 목록), 토큰 클라이언트 저장(OS 보안 저장소)
- [x] 컴플라이언스: N1(연령)·N2(위치정보법) 중 U1 소유분(동의·증적·법정 로그) 정리
- [x] 유지보수성: 모듈 경계 컴파일 타임 강제·테스트 계층 분리(D37)
- [x] 미결 항목 표 작성: 클라우드 벤더 종속 결정(이메일 발송·시크릿 매니저·KMS 등) — Infrastructure Design 질문 답변 후 확정, '대기' 명시

### 3단계 — 기술 스택 U1 구체화 (tech-stack-decisions.md)
- [x] 서버: Kotlin 버전대·Spring Boot 3.x·Spring Security 필터 체인·OAuth 클라이언트(Google/Apple OIDC + 카카오/네이버 커스텀 provider) 확정
- [x] 서버: JWT 라이브러리·비밀번호 해시(bcrypt vs argon2id 비교 후 권고)·Jakarta Validation·DB 접근(JPA vs jOOQ 비교 후 권고)·Flyway 확정
- [x] 서버 PBT: Kotest Property Testing — PBT-09 요건(커스텀 제너레이터·수축·시드 재현·러너 통합) 충족 확인, jqwik 비채택 사유 기록
- [x] 클라이언트: Expo SDK 버전대·expo-auth-session·expo-secure-store·상태 관리·폼·fast-check(PBT-09) 확정
- [x] 각 선택에 대안 비교 1줄 + 선정 사유 기록
- [x] 벤더 종속 항목(메일·시크릿·KMS)은 '클라우드 결정 대기' 표기

### 4단계 — 검증·마감
- [x] 추적 ID(D/Δ/N/G/CQ/SECURITY/RESILIENCY/PBT) 전 항목 표기 확인
- [x] 확장 규칙 컴플라이언스 요약(준수/비준수/N/A + N/A 근거)을 nfr-requirements.md에 포함
- [x] 콘텐츠 검증(common/content-validation.md): 표·ASCII 다이어그램 구문 확인
- [x] 산출물 2종 생성 완료 — 스테이지 완료 메시지·사용자 승인은 메인 워크플로에서 수행

## 산출물

| 파일 | 내용 |
|---|---|
| `../u1-foundation/nfr-requirements/nfr-requirements.md` | U1 관점 NFR 전개(성능·규모·가용성·보안·컴플라이언스·유지보수성·미결 표) |
| `../u1-foundation/nfr-requirements/tech-stack-decisions.md` | 확정 스택(D02)의 U1 구체화 — 버전대·라이브러리·비교 근거 |
| `u1-nfr-infra-questions.md` (본 플랜이 예고 — 별도 생성) | 클라우드 벤더·CI/CD·롤백·배포·인시던트 대응 질문 — 사용자 결정 대기 |
