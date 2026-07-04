# U1 기반·계정·온보딩 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U1 (M1 Auth, M2 User Profile, C3 Content Moderation + 스캐폴드·전역 보안)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. 본 문서는 SI를 **U1 워크로드에 매핑**하고, U1이 소유한 인프라 결정(NFR 미결 PD-1~9)을 종결한다. NFR 요건 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md).

## 0. PD 미결 항목 종결표 (nfr-requirements.md §8)

| # | 항목 | 결정 | 위치 |
|---|---|---|---|
| PD-1 | 트랜잭션 메일 | **Amazon SES**(서울 리전) — 샌드박스 해제 절차 포함 | §4, SI §4 |
| PD-2 | 시크릿 매니저 | **AWS Secrets Manager** — ECS 태스크 정의 주입 | SI §6 |
| PD-3 | KMS·JWT 서명 키 | AWS 관리형 KMS 키 + JWK 셋은 Secrets Manager(kid 롤오버 runbook) | SI §6 |
| PD-4 | 중앙 로그·모니터링·알림 | **CloudWatch Logs/Metrics/Alarms + SNS**, APM·크래시=Sentry | SI §8, §7 |
| PD-5 | 관리형 PostgreSQL 세부 | RDS PostgreSQL 16 Multi-AZ·PITR·보존 7일·복원 검증은 Operations 이연 | SI §3 |
| PD-6 | 유출 비밀번호 온라인 조회 | **HIBP Pwned Passwords k-익명 API**(해시 프리픽스 5자만 전송) — 1초 타임박스·fail-open(NFR-U1-SEC-04 기확정), NAT 아웃바운드 허용 목록 문서화 | §5 |
| PD-7 | CI/CD·롤백·배포 | GitHub Actions / 버전 고정 재배포 / 롤링(전역 확정) | [deployment-architecture.md](./deployment-architecture.md) |
| PD-8 | 변경 관리·인시던트 | 경량 프로세스 — **NFR Design 문서 소유**(전역 확정). 인프라는 SNS 알림 라우팅(SI §8.2)까지 제공 | SI §8 |
| PD-9 | 세션·rate-limit 공유 캐시 | **미도입 — PostgreSQL 기반으로 확정**(§6) | §6 |

---

## 1. 인증 API의 ALB 라우팅·헬스체크

- **라우팅**: 단일 모놀리스(D04)이므로 ALB 규칙은 기본 1개 — 443 리스너 → 대상 그룹 `trippilot-api`(전 경로 `/api/*` 및 `/actuator/health/liveness`). 경로 분기 없음(모듈 경계는 앱 내부 Gradle 경계 — NFR-U1-MT-01). U1 공개 엔드포인트 화이트리스트(NFR-U1-SEC-16)는 **Spring Security가 정본**이며 ALB에서 중복 관리하지 않는다(이중 관리 드리프트 방지).
- **헬스체크 경로(NFR-U1-AV-06)**:

| 용도 | 경로 | 판정 | 소비자 |
|---|---|---|---|
| shallow (프로세스 생존) | `/actuator/health/liveness` | 프로세스·스프링 컨텍스트만 — **DB 미포함** | ALB 대상 그룹 + ECS 컨테이너 헬스체크 |
| deep (의존 포함) | `/actuator/health/readiness` | DB 연결(HikariCP)·필수 시크릿 로드 포함 | **관측 전용** — CloudWatch Synthetics 없이 앱 자체 주기 계측 메트릭으로 발행. LB 판단 배제(DB 순단 → 태스크 대량 축출 증폭 차단) |

- ALB 헬스체크 파라미터: interval 15s, timeout 5s, healthy 2회, unhealthy 3회, 성공 코드 200. ECS `healthCheckGracePeriodSeconds=90`(JVM 기동 — SI §1.2).
- 두 경로 모두 인증 예외(공개 화이트리스트)이나 **응답 본문은 상태 코드만**(버전·의존 상세 비노출 — SECURITY-09).

## 2. 규모 매핑 — U1 트래픽과 태스크 사이징 정합

- NFR-U1-SC-01(단일 인스턴스 10 RPS 헤드룸): argon2id(m=19MiB·t=2 — 검증 100–200ms CPU) 동시 실행이 상한 결정 요인. 태스크 1vCPU에서 로그인·가입 동시 ~5건/초 + 일반 API 병행 가능 — 피크 로그인 0.003 RPS(§2.1 nfr) 대비 3자릿수 여유. **SI §1.2의 1vCPU/2GB ×2 태스크로 충족** — 추가 산정 불요(G142).
- JVM 메모리: 힙 1GB(-Xmx1g) + argon2 오프힙(19MiB × 동시 해시 ≤ 8 = ~152MiB) + 메타스페이스 — 2GB 태스크 내 안전.

## 3. RDS 스키마·마이그레이션 실행 방식 (Flyway 잡)

- **DB·롤 모델**: SI §3의 3롤(`마스터`/`app_migrate`/`app_user`). 데이터베이스 `trippilot`, 스키마 `public` 단일(모듈 경계는 Gradle — 스키마 분리는 과설계, G142).
- **마이그레이션 실행 방식 — 배포 전 일회성 ECS 태스크(확정)**:
  - GitHub Actions 배포 잡이 `ecs run-task`로 **Flyway 마이그레이션 태스크**(동일 이미지, 엔트리포인트 `flyway migrate` 프로파일, `sg-migrate`·`app_migrate` 자격 증명)를 실행하고 **종료 코드 0 확인 후에만** 서비스 롤링 업데이트를 개시한다(순서 보장 — forward-only 전제, deployment-architecture.md §4).
  - **앱 기동 시 자동 마이그레이션 비활성**(`flyway.enabled=false` 런타임) — 근거: 롤링 중 신구 태스크 동시 기동 시 경합·부분 적용 위험 제거, `app_user`에 DDL 권한을 주지 않기 위함(SECURITY-06).
  - 로컬·CI(Testcontainers)는 Flyway 자동 적용 유지 — 마이그레이션·권한 검증(append-only REVOKE 포함)이 PR CI에서 실 PostgreSQL로 실행된다(NFR-U1-MT-02).
- **권한 DDL이 마이그레이션에 포함**(tech-stack §2.8 기확정): U1 마이그레이션 시퀀스 말미에 `REVOKE UPDATE, DELETE ON location_legal_log, consent_record FROM app_user` 및 테이블별 GRANT 명세 — 마이그레이션 리뷰로 append-only 검증(U1 DoD, NFR-U1-SEC-23·24).
- U1 테이블(~10~11개: 계정·소셜 연결·이메일 인증 토큰·리프레시 세션·약관 버전·동의 증적·프로필·취향·위치 동의 상태·위치 법정 로그·금칙어 사전 + ShedLock·브루트포스 카운터)은 전부 단일 RDS — 볼륨 근거 NFR-U1-SC-03. 위치 법정 로그는 월 파티션 선언만 U1에서(기록 개시는 U6).

## 4. SES 인증 메일 — 도메인 검증·SPF/DKIM (PD-1)

| 항목 | 구성 |
|---|---|
| Identity | 도메인 identity `trippilot.app`(가칭 — **P9 스토어·도메인 확정과 연계**: 도메인 확정 즉시 Route 53 존과 함께 등록. 확정 전 dev는 개별 이메일 identity로 개발 진행) |
| DKIM | **Easy DKIM(2048비트)** — Route 53 CNAME 3종 자동 등록 |
| SPF | 커스텀 MAIL FROM 서브도메인 `mail.trippilot.app` + MX/TXT(`v=spf1 include:amazonses.com ~all`) — 발신 도메인 정렬(DMARC 통과 요건) |
| DMARC | `_dmarc` TXT `v=DMARC1; p=quarantine; rua=mailto:...` — 도달률·위조 방어(인증 링크 메일의 스팸함 회피가 곧 도달 SLO 5분/95%의 전제) |
| Configuration Set | `trippilot-{env}-transactional` — 바운스·컴플레인·딜리버리 이벤트 → CloudWatch 메트릭(알람 A13 입력) |
| 샌드박스 해제 | SI §4 절차 — **U1 개발 기간 중 prod 계정에서 신청**(승인 리드타임 선반영). dev는 샌드박스+검증 수신자 유지 |
| 발송 경로 | `MailDeliveryPort` → SES SDK v2 어댑터(비동기 잡 — 접수/발송 분리 기확정 §1.2 nfr). IAM `ses:SendEmail` + `ses:FromAddress` 조건(SI §7.1) — SMTP 자격 증명 미발급 |
| 재발송 상한 | 분당 1회·일 5회(G22)는 **서버 강제**(§6 카운터) — SES 쿼터 보호 겸용(NFR-U1-SC-04) |

## 5. 소셜 IdP 아웃바운드·시크릿 보관

- **경로**: ECS(프라이빗) → NAT → IdP 토큰 엔드포인트 4종(Google·Apple·카카오·네이버) + JWKS 조회 + HIBP(PD-6). `sg-app` 아웃바운드 443(SI §2.2 — 목적지 IP 가변 SaaS 예외 문서화). 어댑터별 타임아웃 3초+서킷(NFR-U1-AV-04 — 앱 소관, 인프라는 A12 실패율 알람 제공).
- **시크릿(SI §6)**: `social/google`(client_secret), `social/kakao`, `social/naver`, `social/apple-p8`(서명 키 원문 — Apple client_secret JWT는 앱이 p8로 ES256 동적 서명, 수명 6개월 미만 재생성 runbook). 전부 Secrets Manager ARN 참조로 태스크 주입 — 코드·이미지·Terraform state 평문 0(NFR-U1-SEC-11).
- JWKS·discovery 응답은 앱 인메모리 캐시(TTL) — 인프라 캐시 불요(G142).
- **HIBP(PD-6 종결)**: k-익명 프리픽스 5자만 아웃바운드(평문·전체 해시 미전송 — NFR-U1-SEC-04 기확정 구조), 타임아웃 1초·fail-open·실패 계측. 별도 계약 불요(무료 공개 API — 쿼터 정책 문서화, A11 대상 외 저빈도).

## 6. 레이트리미터·브루트포스 카운터 저장 — 인메모리 vs ElastiCache (PD-9 종결)

| 기준 | 인스턴스 인메모리 | **PostgreSQL 기반 (확정)** | ElastiCache Redis |
|---|---|---|---|
| 정본 적합성 | **금지** — NFR-U1-AV-02(인스턴스 로컬 정본 금지·2태스크 라우팅 불일치) | 충족 — 어느 태스크든 동일 판정 | 충족 |
| 규모 정합(G142) | — | 인증 계열 피크 0.5 RPS — 카운터 UPDATE는 무부하 | **월 ~$25 + 운영 표면 추가가 이득 0** |
| 장애 결합 | — | DB 다운 시 로그인 자체가 불가(Critical 본질)라 추가 결합 없음 | 캐시 장애 시 폴백 설계 별도 필요(NFR-U1-AV-03) |

**확정: PostgreSQL 단일** — 브루트포스 카운터(계정·출처 단위, NFR-U1-SEC-07·08)·메일 재발송 상한(G22)을 DB 테이블(원자적 UPSERT)로 구현. 보조로 **인스턴스 로컬 토큰버킷을 1차 방어(coarse limit)로 허용**하되 정본 판정은 DB(허용 오차 = 태스크 수 × 상한 — 문서화). **ElastiCache 도입 트리거**: 인증 계열 실측 50 RPS 초과 또는 U11 WebSocket 팬아웃 요구 시(SI §4와 동일 트리거로 통합 재평가).

## 7. 법정 로그·동의 증적 저장 구현 (N2·N3·SECURITY-14)

- **방식**: SI §5.3 확정안 — PostgreSQL append-only 테이블 + 권한 분리(비교·권고 근거는 SI 소유).
- U1 구현 체크리스트:
  1. `consent_record`(동의 증적 — NFR-U1-SEC-23)·`location_legal_log`(위치 법정 로그 — NFR-U1-SEC-24): INSERT 전용 리포지토리(JdbcTemplate 직삽입, JPA UPDATE 경로 구조 배제 — tech-stack §2.7 단서) + `app_user` REVOKE UPDATE/DELETE(§3 마이그레이션 DDL).
  2. 위치 법정 로그 보존 6개월+(N2): 파기 잡도 `app_user`로는 **불가능한 구조** — 보존 만료 파기는 별도 운영 절차(월 파티션 DROP을 `app_migrate` 권한의 수동 승인 잡으로, U6 활성 후 가동). 6개월 미만 데이터 삭제는 어떤 경로로도 불가.
  3. IAM 이중 강제: 태스크 역할에 로그 파괴 계열 명시 Deny(SI §7.1).
  4. 백업 포함: RDS 자동 백업+PITR이 증적의 2차 사본(RPO 시간 단위 — CQ4). 월 1회 S3 `log-archive` 내보내기는 U6 기록 개시 시점부터(SI §5.3).
  5. 검증: PR CI Testcontainers에서 `app_user`로 UPDATE/DELETE 시도 → 권한 오류 단언 테스트(NFR-U1-MT-02 — append-only의 회귀 방지).

## 8. U1 알람 셋 (SI §8.2 A14의 구체화 — NFR-U1-SEC-27)

전부 CloudWatch Logs **메트릭 필터**(구조화 JSON 보안 이벤트 — NFR-U1-SEC-26 감사 이벤트가 소스) → 알람 → SNS `trippilot-{env}-alerts`.

| 알람 | 필터 대상 이벤트 | 임계(초기) | 단계 |
|---|---|---|---|
| U1-S1 인증 실패 급증 | `auth.login.failed` | > 50건/5분 (정상 피크 로그인 0.003 RPS 대비 이상치) | P2 |
| U1-S2 **리프레시 재사용 감지** | `auth.token.reuse_detected` | **≥ 1건 즉시** (탈취 신호 — NFR-U1-SEC-05) | P1 |
| U1-S3 권한 위반(403) 급증 | `authz.denied` | > 30건/5분 | P2 |
| U1-S4 브루트포스 잠금 급증 | `auth.lockout` | > 10건/15분 | P2 |
| U1-S5 메일 발송 실패율 | `mail.delivery.failed` / 시도 | > 10%/15분 (+ A13 SES 바운스와 이중) | P2 |
| U1-S6 가입 스파이크 | `account.created` | > 500건/시간 (NFR-U1-SC-01 헤드룸 소진 경보 — 축하할 일이어도 알림) | P2 |
| U1-S7 C3 사전 로딩 실패 | `moderation.dictionary.load_failed` | ≥ 1건 | P2 (fail-closed 상태 — 가입 흐름 보류 중 신호) |

- U1은 공통 알람 A1~A9·A13·A15(SI)의 **첫 소비자**이며 임계 초기값의 실측 보정 책임을 가진다(운영 2주 후 재조정 — 경량 변경 관리 기록).

## 9. 컴플라이언스 요약 (U1 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-01·02·06·07·09 | 준수 | SI §2·3·5·7 매핑 — U1 특이사항 없음 |
| SECURITY-03·14 | 준수 | §7 append-only 2중 강제·§8 알람 셋·보존(SI §8.1) |
| SECURITY-04·05·08·11·12·15 | 준수(설계 기확정) | nfr-requirements.md §4 — 코드 증빙은 Code Generation |
| SECURITY-10 | 준수 | deployment-architecture.md(스캔·SBOM·태그 고정·OIDC) |
| SECURITY-13 | 부분 반영 유지 | 파이프라인 통제는 deployment-architecture.md §2·§6, 역직렬화는 Code Generation |
| RESILIENCY-01·02·08·09·10·11·12 | 준수 | SI §1~3·8 — M1 Critical의 Multi-AZ·백업·알람 충족 |
| RESILIENCY-03·04·15 | 준수(프로세스 위임) | 전역 결정 — 경량 프로세스는 NFR Design 문서 소유, 파이프라인은 PD-7 종결 |
| RESILIENCY-14 | 준수(이연 확정) | 복원력 테스트 Operations 이연(전역 결정) — 시나리오 문서는 deployment-architecture.md §5 runbook |
| PBT-01~10 | 해당 유지 | 인프라 무관 — CI 실행 요건은 deployment-architecture.md §2(PBT-08 시드 로깅) |

**차단 소견 없음.** PD-1~9 전 항목 종결(§0) — nfr-requirements.md §8의 '대기' 상태 해소.
