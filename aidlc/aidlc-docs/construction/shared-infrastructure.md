# TripPilot 공유 인프라 정본 (Shared Infrastructure)

> 2026-07-04 · CONSTRUCTION · **프로젝트 공유 인프라 정본 — U1 Infrastructure Design에서 확정, U2~U8(및 후속 U9~U11)이 재사용**
> **전역 확정 결정**(u1-nfr-infra-questions.md, 사용자 답변 2026-07-04): 클라우드=**AWS 서울 리전(ap-northeast-2)** / CI/CD=GitHub Actions / 롤백=버전 고정 재배포(ECR 이미지 태그)+DB forward-only / 배포=롤링(Multi-AZ 2+ 인스턴스) / 변경 관리·인시던트 대응=경량 프로세스(NFR Design 문서 소유) / 복원력 테스트=Operations 이연.
> **설계 지배 원칙**: 규모 정합(§6.8 G142 — DAU 1천·동시 일정 생성 10·과설계 금지). 모든 선택에 규모 근거를 병기하고, 확장 트리거(언제 상향하는가)를 함께 기록한다.
> 근거 정본: [requirements.md](../inception/requirements/requirements.md) §6.2·6.3·6.8, [nfr-requirements.md](./u1-foundation/nfr-requirements/nfr-requirements.md) §8(PD-1~9), [tech-stack-decisions.md](./u1-foundation/nfr-requirements/tech-stack-decisions.md), Security Baseline(SECURITY-01~15), Resiliency Baseline(RESILIENCY-01~15).

---

## 0. 전체 아키텍처 개요

단일 배포 모놀리스(D04 — Spring Boot Kotlin) + 모바일 앱(Expo)이므로, 서버 런타임은 **컨테이너 서비스 1개**다. 인프라의 골격:

```text
                        +--------------------------+
  [모바일 앱]  --HTTPS-->|  ALB (퍼블릭, 443/80)     |---> 액세스 로그 S3 (SECURITY-02)
                        +-----------+--------------+
                                    |
                     +--------------v---------------+
                     | ECS Fargate 서비스 (프라이빗)  |  min 2 태스크, AZ-a / AZ-c 분산
                     | server/app 모놀리스 컨테이너   |  (RESILIENCY-08)
                     +---+----------+----------+----+
                         |          |          |
              +----------v--+  +----v-----+  +v-------------------+
              | RDS          |  | S3       |  | NAT GW --> 아웃바운드|
              | PostgreSQL   |  | (사진 U7 |  | (소셜 IdP·기상청·   |
              | Multi-AZ     |  |  예약 등) |  |  카카오·TMap·LLM 등)|
              +--------------+  +----------+  +--------------------+
                         |
              Secrets Manager / KMS / CloudWatch / SNS / SES / (FCM: U8 아웃바운드)
```

**텍스트 대안**: 모바일 앱이 퍼블릭 ALB(443/80)로 접속하고, ALB는 프라이빗 서브넷의 ECS Fargate 태스크(최소 2개, 2개 AZ 분산)로 라우팅한다. 태스크는 RDS PostgreSQL Multi-AZ, S3, Secrets Manager에 접근하며, 외부 API(소셜 IdP·기상청·카카오·TMap·LLM·FCM)는 NAT Gateway를 경유해 아웃바운드 호출한다. ALB 액세스 로그는 S3에, 앱 로그는 CloudWatch Logs에 적재되고 알람은 SNS로 라우팅된다.

| 유닛 | 본 문서에서 소비하는 요소 |
|---|---|
| U1 | 전부(기준 정의 유닛) — 상세 매핑은 [u1-foundation/infrastructure-design/infrastructure-design.md](./u1-foundation/infrastructure-design/infrastructure-design.md) |
| U2 | ALB 라우팅·remote config성 설정(부트스트랩 API) |
| U3 | NAT 아웃바운드(TourAPI·카카오·네이버), 외부 쿼터 알람 프레임(§8) |
| U4 | 추가 없음(기존 컴퓨트·DB) |
| U5 | NAT 아웃바운드(LLM·TMap), LLM 비용 계측 메트릭(§8) |
| U6 | NAT 아웃바운드(기상청), 스케줄 잡 프레임(§6), 위치 법정 로그 기록 개시(§5.3) |
| U7 | **사진 S3 버킷 + CloudFront(§5.2 — 예약분 활성화)** |
| U8 | FCM 아웃바운드, SNS/알림 스케줄링(서버 소유 D32 — DB 기반 잡, §6) |
| U9~U11 | WebSocket(U11 — ALB 지원, 스티키니스 검토), 어드민 웹(U10 — CloudFront+별도 오리진 검토) |

---

## 1. 컴퓨트 — ECS Fargate (확정 권고)

### 1.1 선택 비교: ECS Fargate vs ECS on EC2 vs EKS

| 기준 (MVP 규모 §6.8: 정상 피크 ~0.5 RPS, 헤드룸 10 RPS) | **ECS Fargate** | ECS on EC2 | EKS |
|---|---|---|---|
| 운영 부담(패치·용량 관리) | **없음** — OS·호스트 관리 제로(SECURITY-09 하드닝 표면 최소) | AMI 패치·ASG 관리 필요 | 컨트롤플레인+노드+애드온 운영, 전담 역량 필요 |
| 고정 비용 | 태스크 사용량만 | EC2 예약 대비 저렴 가능하나 관리 인건비 상회 | **컨트롤플레인 $73/월 고정** — 서비스 1개에 순수 낭비 |
| Multi-AZ 롤링 배포(전역 결정) | 서비스 정의로 즉시 충족 | 동일하나 호스트 계층 추가 | Deployment로 충족하나 과잉 계층 |
| 규모 정합(G142) | **적합** — 서비스 1개·태스크 2~4개 | 태스크 밀도 이점이 나올 규모 아님 | K8s 생태계가 필요한 마이크로서비스 아님(D04 모놀리스) |
| 후속 확장(U11 WebSocket 등) | 서비스 추가로 대응 가능 | 동일 | 이 시점 재평가 가능 |

**확정 권고: ECS Fargate.** 서버가 단일 모놀리스 컨테이너 1종(D04)이고 팀에 전담 인프라 인력이 없는 MVP에서, 호스트 운영이 없는 Fargate가 총비용(인프라+인건)이 최소다. EKS는 컨트롤플레인 고정비와 운영 복잡도가 규모 대비 과설계(G142), ECS on EC2는 태스크 밀도 이점이 발생할 규모가 아니다. **전환 트리거**: 상시 태스크 8개 이상 또는 GPU/특수 런타임 필요 시 EC2 용량 공급자 병용 재평가.

### 1.2 서비스 구성 (전역 결정 반영: 롤링·Multi-AZ 2+)

| 항목 | 값 | 근거 |
|---|---|---|
| 클러스터 | `trippilot-{env}` 1개 | 단일 서비스 |
| 서비스 | `trippilot-api` (server/app 모놀리스) | D04 |
| 태스크 사이즈 | **1 vCPU / 2GB** (prod) · 0.5 vCPU / 1GB (dev) | JVM(Spring Boot) 상주 + argon2id 검증(건당 ~19MiB·100–200ms CPU, NFR-U1-SC-01) 동시 처리 헤드룸. 10 RPS 설계 목표를 태스크 2개로 소화 |
| **태스크 수** | **min 2 (AZ-a·AZ-c 강제 분산, spread 배치 전략)** / max 4 | RESILIENCY-08 Multi-AZ, 배포 스타일=롤링(전역 결정) |
| 오토스케일 | Target Tracking: ①평균 CPU 60% ②ALB RequestCountPerTarget 300/분 — 스케일아웃 쿨다운 60초·인 300초 | RESILIENCY-09 min/max 한도. max 4는 비용 상한 겸용(초과 필요 = 용량 재설계 신호) |
| **배포(롤링)** | `minimumHealthyPercent=100`, `maximumPercent=200` — 신 태스크 헬시 확인 후 구 태스크 종료. **Deployment Circuit Breaker + 자동 롤백 활성화**(헬스 실패 시 직전 태스크 정의로 복귀 — 롤백=버전 고정 재배포와 정합) | 전역 결정(롤링), NFR-U1-AV-05 무중단 내성 |
| 헬스체크 | ALB 대상 그룹: **shallow**(`/actuator/health/liveness`)만 사용. deep(DB 포함)은 관측 전용 — LB 판단에서 배제해 DB 순단의 태스크 대량 축출 증폭 차단 | NFR-U1-AV-06 |
| 헬스체크 유예 | `healthCheckGracePeriodSeconds=90` (JVM 기동) | — |
| 이미지 | ECR `trippilot-api:{git-sha}` — **불변 태그(ECR tag immutability ON)**, base 이미지 digest 고정 | SECURITY-10, 롤백=태그 재배포(전역 결정) |
| 컨테이너 하드닝 | non-root USER, read-only root filesystem, 최소 베이스(distroless 또는 ECR Public temurin-jre 21 digest 고정) | SECURITY-09 |
| 런타임 | Fargate Platform 1.4+, ARM64(Graviton — 동급 대비 ~20% 저렴) | 비용(G142) |

**스케줄 잡 실행 모델**: U1~U8의 스케줄러 잡(삭제 유예 만료·미인증 정리·POI 동기화·날씨 폴링·알림 스케줄링 등)은 **별도 워커 서비스를 두지 않고** 앱 프로세스 내 스케줄러(Spring `@Scheduled`) + **PostgreSQL 기반 분산 락(ShedLock 테이블)**으로 실행한다. 근거: 잡 전부가 분 단위 주기·저부하(G142)이며, 태스크 2개 중복 실행은 DB 락으로 배제. **전환 트리거**: 잡 실행 시간이 API 지연에 간섭(p95 열화 상관 관측)하면 EventBridge Scheduler + ECS RunTask 분리.

---

## 2. 네트워크 — VPC (SECURITY-07 deny-by-default)

### 2.1 VPC·서브넷 (2 AZ)

| 항목 | 값 |
|---|---|
| VPC | `10.20.0.0/16` (prod) / `10.10.0.0/16` (dev) — DNS hostnames/support ON |
| AZ | ap-northeast-2a, ap-northeast-2c (2개 — RESILIENCY-08 충족 최소) |
| 퍼블릭 서브넷 ×2 | `10.20.0.0/24`, `10.20.1.0/24` — ALB·NAT GW만 배치 |
| 프라이빗 앱 서브넷 ×2 | `10.20.10.0/24`, `10.20.11.0/24` — ECS 태스크. **IGW 직결 라우트 금지, 아웃바운드는 NAT 경유**(SECURITY-07) |
| 프라이빗 DB 서브넷 ×2 | `10.20.20.0/24`, `10.20.21.0/24` — RDS 전용. **아웃바운드 라우트 자체 없음**(NAT도 미연결 — DB는 외부 호출 불요) |

- **NAT Gateway: 1개(AZ-a)로 시작** — 근거: NAT 장애 영향은 아웃바운드 한정(소셜 신규 로그인·외부 API — 전부 폴백 보유, NFR-U1-AV-04·RESILIENCY-10)이고 RTO 목표가 시간 단위(CQ4)이므로, AZ 장애 시 IaC로 타 AZ NAT 재생성(runbook, ~15분)이 목표 내 복구다. AZ당 NAT 2개(월 +$45)는 MVP 과설계(G142). **전환 트리거**: 아웃바운드 의존 기능이 Critical로 승격되거나 유료 사용자 SLA 도입 시 AZ당 NAT.
- **VPC 엔드포인트**: S3 **Gateway 엔드포인트(무료) 필수 적용**(사진 업로드·ALB 로그 경로가 NAT 요금 우회 + 프라이빗 접근 — SECURITY-07 "가능한 곳 프라이빗 엔드포인트"). ECR·CloudWatch Logs·Secrets Manager Interface 엔드포인트는 **미도입(문서화된 예외)** — 근거: 트래픽 소량(로그 <1GB/일 추정)으로 엔드포인트 고정비(개당·AZ당 ~$7/월)가 NAT 데이터 요금을 상회. 전환 트리거: NAT 데이터 처리량 월 100GB 초과.

### 2.2 보안 그룹 (deny-by-default — SECURITY-07)

| SG | 인바운드 | 아웃바운드 |
|---|---|---|
| `sg-alb` | **443, 80 from 0.0.0.0/0** (공개 LB 허용 예외 — SECURITY-07 명시 허용 범위) | 8080 → `sg-app`만 |
| `sg-app` (ECS) | 8080 from `sg-alb`만 | 443 → 0.0.0.0/0 (외부 API·AWS API — NAT 경유. **정당화**: 소셜 IdP 4종·기상청·카카오·TMap·LLM·FCM·OTA 등 목적지 IP가 가변인 SaaS 다수로 IP 고정 불가. 도메인 수준 통제는 Operations에서 egress 프록시 검토), 5432 → `sg-db` |
| `sg-db` (RDS) | **5432 from `sg-app`, `sg-migrate`만** (CIDR 아닌 SG 참조) | 없음 |
| `sg-migrate` (Flyway 마이그레이션 태스크) | 없음 | 5432 → `sg-db`, 443(ECR·로그) |

NACL은 서브넷 기본값 유지(SG로 통제 일원화 — 이중 관리 비용 회피, G142). 0.0.0.0/0 인바운드는 ALB 80/443이 유일하다.

### 2.3 ALB

| 항목 | 값 | 근거 |
|---|---|---|
| 리스너 | 443(HTTPS, ACM 인증서·TLS 정책 `ELBSecurityPolicy-TLS13-1-2-2021-06` — TLS 1.2+) / 80은 443 리다이렉트 전용 | SECURITY-01 전송 암호화 |
| **액세스 로그** | **S3 `trippilot-{env}-alb-logs` 활성화**(SSE-S3 — ALB 로그는 SSE-S3만 지원), 보존 90일+ 수명주기(§5.4) | **SECURITY-02** |
| 대상 그룹 | ECS `trippilot-api`, 헬스체크 §1.2 | — |
| 보호 | 삭제 보호 ON, 유휴 타임아웃 60초 | — |
| WAF | 1차 미도입 — 근거: 앱 레벨 rate-limit·브루트포스 방어(NFR-U1-SEC-07·08)가 선행 구현되고 공개 표면이 JSON API 한정. **전환 트리거**: 자동화 공격 관측(CAPTCHA 도입 트리거와 동일 지표) 시 AWS WAF 관리 규칙 도입 | G142 |
| 도메인 | Route 53 `api.trippilot.app`(가칭 — P9 도메인 확정 연계) → ALB alias | — |

---

## 3. 데이터베이스 — RDS PostgreSQL Multi-AZ

| 항목 | 값 | 근거 |
|---|---|---|
| 엔진 | RDS PostgreSQL 16.x (관리형 — 기확정) | requirements §2.3, PD-5 |
| 인스턴스 | **db.t4g.small, Multi-AZ 인스턴스 배포**(prod) / db.t4g.micro Single-AZ(dev) | 첫해 계정 ~2만·피크 0.5 RPS(NFR-U1-SC-03)에 충분. Aurora는 최소 비용 구조가 규모 대비 과잉(G142). 전환 트리거: 연결 수 상시 60% 초과 또는 CPU 상시 50% 초과 시 small→medium |
| 스토리지 | gp3 20GB, 오토스케일 상한 100GB, **저장 암호화 KMS**(§7) | **SECURITY-01 at-rest** |
| **TLS 강제** | 파라미터 그룹 `rds.force_ssl=1` — 비TLS 연결 거부, 앱 JDBC `sslmode=verify-full` | **SECURITY-01 in-transit** |
| **백업** | 자동 백업 보존 **7일 + PITR(5분 단위)** — RPO 시간 단위 목표(CQ4) 대비 상회 충족. 백업 암호화(스냅샷 KMS 상속). 월 1회 수동 스냅샷(분기 보존) | RESILIENCY-02·11·12, requirements §6.2 G147 |
| 복원 검증 | 복원 리허설 절차는 **Operations 이연(전역 결정 — 복원력 테스트)** — 시나리오 문서는 deployment-architecture.md 롤백 runbook에 병기 | RESILIENCY-12·14 |
| Multi-AZ 페일오버 | 관리형 자동(통상 60~120초) — 앱은 JDBC 재연결(HikariCP `maxLifetime` < DNS TTL 고려) | RESILIENCY-08 |
| 파라미터 원칙 | `rds.force_ssl=1`, `log_min_duration_statement=1000ms`(느린 쿼리 관측), `max_connections` 기본(t4g.small ~170 — 태스크 4×풀 20=80 여유), 타임존 UTC 고정 | §8 관측 |
| 삭제 보호 | ON + 최종 스냅샷 필수 | — |
| Performance Insights | ON(7일 무료 구간) | RESILIENCY-05 |

**DB 계정·권한 분리(SECURITY-06 + N2)** — 스키마 정본은 U1 마이그레이션이 소유(상세: u1 infrastructure-design.md §3):

| DB 롤 | 권한 | 사용처 |
|---|---|---|
| 마스터 | 전체(비상용 — Secrets Manager 보관, 평시 미사용) | 운영자 breakglass |
| `app_migrate` | DDL + 권한 부여 | Flyway 마이그레이션 태스크 전용 |
| `app_user` | 테이블별 DML — **법정 로그·동의 증적 테이블은 INSERT/SELECT만(UPDATE/DELETE REVOKE)** | 앱 런타임(append-only의 DB 레벨 강제 — NFR-U1-SEC-23·24) |

---

## 4. 메시징·비동기 — 브로커 없음 (DB 기반 확정)

- **도메인 이벤트**: 모듈러 모놀리스 내 인프로세스 이벤트 버스(`common/core` — U1 산출)가 정본. **크로스-인스턴스 브로커(SQS·Kafka 등) 미도입** — 근거: 발행·구독이 전부 단일 프로세스 내 모듈 간 계약(D04)이고, 유일한 지속 요구(알림 스케줄링 D32·아웃박스성 재시도)는 **PostgreSQL 테이블 기반 아웃박스 + ShedLock 폴링 잡**으로 충족된다. 피크 0.5 RPS·DAU 1천(G142)에서 브로커는 운영 표면만 늘린다.
- **전환 트리거**: U11(WebSocket 다중 인스턴스 팬아웃) 착수 시점에 경량 pub/sub(ElastiCache Redis 또는 SNS) 재평가 — 그 전까지 예약만.
- **이메일 = Amazon SES (PD-1 확정)**: 트랜잭션 메일(인증 링크·비밀번호 재설정) 발송. `MailDeliveryPort` 어댑터로 격리(기확정), 도달 SLO 5분/95%. 도메인 검증·DKIM 등 상세는 U1 문서 §4.
  - **샌드박스 해제 절차(주석)**: 신규 SES 계정은 샌드박스(검증된 수신자만·일 200통 제한) 상태다. **프로덕션 승격 요청**(AWS 콘솔 → SES → Account dashboard → Request production access: 사용 사례=트랜잭션, 발송량 추정 일 1천 미만, 바운스·컴플레인 처리 방식 기술)을 U1 개발 기간 중 착수(승인 1~2영업일). dev 환경은 샌드박스 유지 + 검증된 테스트 수신자로 충분.
- **푸시 = FCM(U8)**: 서버 아웃바운드(NAT 경유) — 인프라 신규 요소 없음, 서비스 계정 키는 Secrets Manager(§7).

---

## 5. 스토리지 — S3 · CloudFront · 법정 로그

### 5.1 S3 공통 기준선 (전 버킷)

- **퍼블릭 액세스 전면 차단**(계정 레벨 Block Public Access ON — SECURITY-09), **버저닝 ON**, **기본 암호화 SSE-KMS**(ALB 로그 버킷만 SSE-S3 — 서비스 제약), **TLS-only 버킷 정책**(`aws:SecureTransport=false` Deny — SECURITY-01), 수명주기 규칙 필수.

### 5.2 버킷 목록

| 버킷 | 용도 | 소비 유닛 | 수명주기 |
|---|---|---|---|
| `trippilot-{env}-alb-logs` | ALB 액세스 로그(SECURITY-02) | 전 유닛 | 90일 후 Glacier IR, 400일 파기 |
| `trippilot-{env}-photos` | **사진 원본·썸네일 — U7 예약**(버킷·정책만 선생성, 파이프라인은 U7) | U7 | 버저닝 + 이전 버전 30일 파기 |
| `trippilot-{env}-log-archive` | CloudWatch Logs 장기 아카이브 내보내기(90일 초과 보존분) | 전 유닛 | 1년 후 Glacier, 3년 파기(운영 정책) |
| `trippilot-{env}-artifacts` | SBOM·릴리스 부속물(SECURITY-10) | CI | 1년 |

### 5.3 위치정보 법정 로그 저장 방식 — 비교 후 확정 (N2·NFR-U1-SEC-24)

| 기준 | **A. PostgreSQL append-only 테이블 + DB 권한 분리 (권고)** | B. S3 Object Lock(Compliance 모드) |
|---|---|---|
| 삭제·변조 방지 | `app_user`에서 UPDATE/DELETE REVOKE — DB 레벨 강제. 마이그레이션 리뷰로 검증(U1 DoD 기확정) | 보존 기간 내 누구도 삭제 불가(최강) |
| 법정 대응(6개월+ 조회·제출) | **SQL 조회·계정별 추출 즉시 가능** — 사실확인자료 제출 요건에 직합 | Athena/수집 파이프라인 별도 구축 필요 |
| 기록 정합성 | GPS 이벤트와 **동일 트랜잭션 기록** 가능(유실 0) | 비동기 적재 — 유실·중복 처리 계층 추가 필요 |
| 규모(G142) | U6 활성 후에도 일 수만 행 수준 — 단일 DB로 충분(NFR-U1-SC-03, 파티셔닝 여지 확보) | 과잉 구성 요소 추가 |
| 백업·내구성 | RDS 자동 백업+PITR에 포함(백업이 곧 2차 사본) | S3 자체 내구성 |
| 잔여 리스크 | DB 마스터 권한 보유자는 이론상 변조 가능 | — |

**확정 권고: A(PostgreSQL append-only 테이블 + 권한 분리)** — NFR-U1-SEC-24가 이미 요구한 구조이며, 법정 로그의 본질 요구(6개월+ 보존·제출 가능성·기록 누락 0)에 트랜잭션 정합·조회성이 직접 부합한다. B의 잔여 리스크 보완으로 **월 1회 스냅샷 잡이 전월분을 `log-archive` 버킷에 내보내기(SSE-KMS·별도 프리픽스)** 를 U6 기능 활성 시점부터 가동한다(Object Lock 없이 IAM으로 충분 — 아래). Object Lock Compliance 모드 전면 도입은 운영 실수 시 복구 불능 비용 대비 이득이 없어 비채택(G142).

**앱 역할 삭제 불가 — 2중 강제**: ① DB 레벨: `app_user` REVOKE(§3) ② IAM 레벨: ECS 태스크 역할에 `s3:DeleteObject`(log-archive·alb-logs 버킷)·`logs:DeleteLogGroup`·`logs:DeleteLogStream`·`logs:PutRetentionPolicy` **명시적 Deny** 문 부착(SECURITY-14 "앱이 자기 감사 로그를 삭제·수정 불가").

### 5.4 CloudFront — U7 예약

- 사진 서빙 CDN: **CloudFront + S3 OAC(Origin Access Control)** — 버킷 직접 접근 차단, 서명 URL 정책·캐시 정책은 **U7 Infrastructure Design에서 확정**(사진 파이프라인 소유 유닛). 공유 정본에는 배포 자리와 원칙만 예약: 표준 로깅 활성(SECURITY-02), TLS 1.2+ viewer 정책, 기본 루트 객체 없음(디렉터리 리스팅 불가 — SECURITY-09).
- U10 어드민 웹 정적 호스팅도 동일 패턴 재사용 예정(후속 게이트).

---

## 6. 시크릿·키 관리 — Secrets Manager + KMS (PD-2·PD-3 확정)

| 시크릿 | 보관 | 로테이션 |
|---|---|---|
| DB 자격 증명(app_user·app_migrate·마스터) | Secrets Manager — RDS 연동 시크릿 | 마스터는 관리형 로테이션 후속 검토, 앱 계정은 반기 수동(경량 프로세스) |
| JWT 서명 키(JWK 셋 — kid 롤오버 구조, PD-3) | Secrets Manager(JSON JWK 셋) | 신 kid 추가 → 배포 → 구 kid 검증 유예(액세스 1h) 후 제거 — runbook화 |
| 소셜 IdP 시크릿 4종(Google·카카오·네이버 client_secret, **Apple p8 서명 키**) | Secrets Manager | IdP 콘솔 갱신 주기 준수(Apple client_secret JWT는 앱이 p8로 동적 서명 — 6개월 미만 수명) |
| SES는 SDK+IAM 역할 사용 — SMTP 자격 증명 미발급 | — | — |
| FCM 서비스 계정 키(U8) | Secrets Manager(예약) | U8에서 확정 |
| LLM API 키(U5) | Secrets Manager(예약) | U5에서 확정 |

- 주입 방식: ECS 태스크 정의 `secrets`(Secrets Manager ARN 참조) → 환경 변수 — 코드·이미지·IaC에 평문 0(SECURITY-12 하드코딩 금지, NFR-U1-SEC-11). Terraform state에도 시크릿 값 미기록(참조만).
- **KMS**: **AWS 관리형 키**(`aws/rds`·`aws/s3`·`aws/secretsmanager`)로 시작 — SECURITY-01의 "관리형 키 서비스" 요건 충족. CMK(고객 관리형)는 키 정책 분리·크로스 계정 공유 요구 발생 시 도입(월 $1/키 + 운영 — G142 판단). 예외: 사진 버킷은 U7에서 CMK 필요성(서명 URL·다운스트림 통제) 재평가.

---

## 7. IAM — 최소 권한 (SECURITY-06) · GitHub Actions OIDC

### 7.1 역할 설계 원칙

- **와일드카드 금지**: 액션·리소스 전부 명시(SECURITY-06). 예외는 리소스 수준 권한 미지원 API뿐이며 문서화 필수.
- **태스크 실행 역할(execution role) ≠ 태스크 역할(task role) 분리**: 실행 역할은 ECR pull·로그 생성·시크릿 읽기만, 태스크 역할은 앱이 실제 호출하는 API만.

| 역할 | 허용(명시 리소스 한정) | 명시 Deny |
|---|---|---|
| `trippilot-{env}-task-exec` | `ecr:GetDownloadUrlForLayer/BatchGetImage`(해당 리포), `logs:CreateLogStream/PutLogEvents`(해당 로그 그룹), `secretsmanager:GetSecretValue`(명시 ARN 목록) | — |
| `trippilot-{env}-task-app` | `s3:PutObject/GetObject`(photos·log-archive 특정 프리픽스), `ses:SendEmail`(검증 identity 한정 + `ses:FromAddress` 조건), `sns:Publish`(알람 토픽 아님 — U8 예약), `cloudwatch:PutMetricData`(네임스페이스 조건) | **`s3:DeleteObject`(로그 버킷), `logs:DeleteLogGroup/DeleteLogStream/PutRetentionPolicy`, `rds:*`, `secretsmanager:*`(쓰기)** — §5.3 로그 무결성 |
| `trippilot-{env}-migrate` | 로그 기록 + Secrets(마이그레이션 자격 증명 ARN) | — |
| `gha-deploy-{env}` (§7.2) | ECR push, ECS 서비스 업데이트·태스크 정의 등록·RunTask(마이그레이션), `iam:PassRole`(위 역할 ARN 한정 조건) | 그 외 전부(기본 거부) |

### 7.2 GitHub Actions OIDC 연동 — 장기 키 금지

- IAM OIDC Identity Provider: `token.actions.githubusercontent.com` 등록. **IAM 사용자 액세스 키(장기 자격 증명) 발급 금지**(SECURITY-10 CI/CD 무결성·SECURITY-12).
- 신뢰 정책 조건: `sub`를 `repo:{org}/TripPilot:ref:refs/heads/main`(dev 배포)·`repo:{org}/TripPilot:environment:prod`(prod 배포 — GitHub Environment 보호 규칙 결합)으로 한정, `aud=sts.amazonaws.com`.
- 워크플로는 `aws-actions/configure-aws-credentials`로 15분 단기 토큰 취득 — 상세 파이프라인은 [deployment-architecture.md](./u1-foundation/infrastructure-design/deployment-architecture.md).

---

## 8. 관측성 — CloudWatch (PD-4 확정)

### 8.1 로그

- **CloudWatch Logs** 확정: ECS awslogs 드라이버 → 로그 그룹 `/trippilot/{env}/api`. 앱은 stdout **구조화 JSON**(logstash-logback-encoder — 타임스탬프·상관 ID·레벨·모듈 태그, PII·토큰 마스킹 컨버터, SECURITY-03·NFR-U1-SEC-21·22 기확정)만 책임지고 수집은 인프라가 소유.
- **보존**: 로그 그룹 보존 **90일**(SECURITY-14 최소 충족) + 90일 초과 보존 필요분은 S3 `log-archive` 내보내기(§5.2). 위치 법정 로그는 DB 소유(§5.3 — CloudWatch 비의존).
- 로그 그룹 삭제·보존 변경 권한은 앱 역할에서 명시 Deny(§7.1).

### 8.2 메트릭·알람 목록 (RESILIENCY-05·07·09 + SECURITY-14)

라우팅: **전 알람 → SNS 토픽 `trippilot-{env}-alerts` → 운영자 이메일(+Slack 웹훅은 Chatbot으로 후속)**. 심각(P1)/주의(P2) 2단 — 경량 인시던트 프로세스(NFR Design 소유) 입력.

| # | 알람 | 임계(초기값 — remote 조정) | 단계 | 근거 |
|---|---|---|---|---|
| A1 | ALB 5xx 비율 | > 5% (5분) | P1 | RESILIENCY-05 |
| A2 | ALB 대상 p95 지연 | > 1초 (5분 지속 — U1 API 예산 §1.2 초과) | P2 | D38, RESILIENCY-05 |
| A3 | ALB 헬시 호스트 수 | < 2 (Multi-AZ 최소 붕괴) | P1 | RESILIENCY-08 |
| A4 | ECS CPU / 메모리 | 평균 > 80% (10분) | P2 | RESILIENCY-09 |
| A5 | ECS 실행 태스크 수 | < desired (5분) | P1 | — |
| A6 | RDS CPU | > 70% (10분) | P2 | RESILIENCY-05 |
| A7 | RDS 연결 수 | > 120 (max_connections의 ~70%) | P2 | §3 |
| A8 | RDS 여유 스토리지 | < 4GB | P1 | — |
| A9 | RDS 페일오버·가용성 이벤트 | RDS 이벤트 구독(failover·failure 카테고리) | P1 | RESILIENCY-08 |
| A10 | **잡·큐 적체** — 아웃박스/스케줄 잡 지연(커스텀 메트릭: 최고 대기 항목 age) | > 15분 | P2 | RESILIENCY-07 (U8 알림 스케줄링 포함) |
| A11 | **외부 API 쿼터 80%** — 어댑터별 호출량 커스텀 메트릭(카카오·TMap·TourAPI·기상청·LLM·SES) 대비 문서화된 쿼터 | 80% | P2 | RESILIENCY-09, NFR-U1-SC-04 |
| A12 | 외부 어댑터 실패율·서킷 오픈 | 실패율 > 20%(5분) 또는 서킷 오픈 이벤트 | P2 | RESILIENCY-10, ADR-0011 |
| A13 | SES 바운스율 / 발송 실패율 | 바운스 > 5%·실패율 임계(U1 문서 §4) | P2 | PD-1, NFR-U1-SEC-27(e) |
| A14 | 보안 이벤트(인증 실패 급증·리프레시 재사용·403 급증·잠금 급증) | 로그 메트릭 필터 — 상세 U1 문서 §7 | P1/P2 | SECURITY-14, NFR-U1-SEC-27 |
| A15 | **월 비용 예산** — AWS Budgets | 예산(§10) 80%·100% | P2 | RESILIENCY-09 비용 상한 |

- 대시보드: CloudWatch 대시보드 1장(`trippilot-{env}-ops`) — A1~A13 위젯 + LLM 비용 계측(U5 확장 슬롯). APM·크래시 리포팅(클라이언트): **Sentry(무료 티어) 권고** — RN·Spring 양쪽 커버, MVP 규모 무과금 구간. X-Ray는 분산 추적 대상이 모놀리스 1개라 미도입(G142).

---

## 9. 환경 — dev / prod 2환경

- **계정 분리 권고: AWS 계정 2개(dev·prod)** — 근거: ①폭발 반경 분리(dev 실수·키 유출이 prod 데이터에 도달 불가 — SECURITY-06을 계정 경계로 상위 강제) ②비용 계정 단위 가시화 ③GitHub OIDC 역할을 계정별로 신뢰 분리(§7.2). 오버헤드는 IAM Identity Center + Terraform 변수화로 흡수 — Organizations 2계정은 Control Tower 없이도 경량 운영 가능(G142 내). 단일 계정+태그 분리는 차선(수용 가능하나 권고하지 않음 — IAM 경계 실수 여지).
- 환경별 차등: dev = Fargate 1태스크(0.5vCPU/1GB)·RDS t4g.micro Single-AZ·Multi-AZ 미적용·야간 정지 스케줄 허용 / prod = 본 문서 §1~§8 전 사양. **보안 기준선(SECURITY-01·07·태그 불변·시크릿 관리)은 양 환경 동일** — dev에서만 완화되는 보안 설정 금지.
- 승격 흐름(main→dev 자동, prod 수동 승인)은 deployment-architecture.md §6.

## 10. 비용 개요 — 월 추정 (prod, 서울 리전, 온디맨드, USD)

| 항목 | 구성 | 월 추정 |
|---|---|---|
| ECS Fargate | 1vCPU/2GB ×2 태스크 ×730h (ARM64) | ~$83 |
| ALB | 고정 + LCU 소량 | ~$22 |
| NAT Gateway | 1개 + 데이터 ~30GB | ~$47 |
| RDS PostgreSQL | db.t4g.small Multi-AZ + gp3 20GB×2 | ~$66 |
| S3 + CloudWatch(로그·알람·대시보드) | 로그 ~20GB/월·보존 90일 | ~$12 |
| Secrets Manager | 시크릿 ~7개 | ~$3 |
| Route 53 + ACM + ECR + SES | 존 1 + 인증서 무료 + 이미지 ~5GB + 메일 소량 | ~$3 |
| **합계 (prod)** | | **~$236/월** |
| dev (축소 사양·야간 정지 시) | | ~$60–90/월 |

전제: DAU 1천(G142) 트래픽. 상한 통제: AWS Budgets 알람(A15) + 오토스케일 max 4(§1.2). 최대 절감 여지: dev 야간 정지(−40%), Fargate 태스크 0.5vCPU 하향(−$40, 부하 실측 후), Compute Savings Plan(안정화 후).

## 11. IaC — 도구 선택 비교와 스택 구성 (확정 권고: Terraform)

### 11.1 Terraform vs AWS CDK

| 기준 | **Terraform (권고)** | AWS CDK (TypeScript) |
|---|---|---|
| 변경 안전성 | **`plan` diff가 PR 리뷰 산출물** — 경량 변경 관리 프로세스(전역 결정: PR 승인 게이트+변경 기록)와 직결 | `cdk diff` 가능하나 CloudFormation 중간층 해석 필요 |
| 상태·드리프트 | S3 backend + 상태 잠금, drift 감지 단순 | CloudFormation 스택 드리프트·롤백 상태 꼬임 대응 비용 |
| 선언성·리뷰 가독성 | HCL 선언 — 리소스=코드 1:1, 비인프라 직군도 리뷰 가능 | 추상화 이점이 있으나 합성 결과 검증 부담 |
| 팀 스택 정합 | 별도 언어(HCL)이나 학습 곡선 낮음 | TS는 앱과 공유되나 서버 팀은 Kotlin — 이점 반감 |
| 생태계 | AWS 프로바이더·모듈 사실상 표준, 멀티 클라우드 이식성 | AWS 한정 |
| 규모 정합(G142) | 리소스 ~60개 수준 — 모듈 몇 개로 충분 | L2/L3 컨스트럭트 이점이 나올 규모 아님 |

**확정 권고: Terraform**(HCL, S3 원격 상태 + 상태 잠금, 계정·환경별 분리). CDK 비채택 사유(1줄): CloudFormation 중간층의 운영 비용 대비, 이 규모에서 컨스트럭트 추상화 이득이 없다.

### 11.2 스택(루트 모듈) 구성 계획

```text
infra/                          # 워크스페이스 루트(코드 — aidlc-docs 밖)
  modules/
    network/     # VPC·서브넷·NAT·엔드포인트·SG (§2)
    compute/     # ECS 클러스터·서비스·태스크 정의·오토스케일·ALB (§1·§2.3)
    database/    # RDS·파라미터 그룹·서브넷 그룹 (§3)
    storage/     # S3 버킷 4종·수명주기·(U7) CloudFront 예약 (§5)
    security/    # KMS 참조·Secrets Manager·IAM 역할·OIDC (§6·§7)
    observability/ # 로그 그룹·메트릭 필터·알람 A1~A15·SNS·대시보드 (§8)
    mail/        # SES 도메인 identity·DKIM·configuration set (§4)
  envs/
    dev/  { main.tf, terraform.tfvars, backend.tf }   # dev 계정
    prod/ { main.tf, terraform.tfvars, backend.tf }   # prod 계정
```

- 원칙: 환경 차이는 tfvars 변수만(모듈 코드 동일), 프로바이더 버전·모듈 버전 고정(SECURITY-10 공급망 원칙의 IaC 적용), `terraform plan`을 CI에서 실행해 PR에 diff 첨부(변경 관리), apply는 수동 승인 잡.
- 적용 순서(U1 시점): network → security → database → mail → compute → observability. storage의 photos/CloudFront는 자리만 생성(U7 활성화).

---

## 12. 확장 규칙 컴플라이언스 (공유 인프라 설계 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-01 | 준수 | §3(RDS KMS·force_ssl)·§5.1(S3 SSE·TLS-only)·§2.3(ALB TLS1.2+) |
| SECURITY-02 | 준수 | §2.3 ALB 액세스 로그 S3, §5.4 CloudFront 로깅(U7 예약분 원칙 고정) |
| SECURITY-06 | 준수 | §7.1 와일드카드 금지·역할 분리·명시 Deny |
| SECURITY-07 | 준수 | §2 deny-by-default SG·프라이빗 서브넷 IGW 직결 금지·NAT 경유·S3 게이트웨이 엔드포인트(인터페이스 엔드포인트 미도입은 문서화 예외) |
| SECURITY-09 | 준수 | §1.2 컨테이너 하드닝·§5.1 퍼블릭 차단·§2.3 WAF 이연 근거 명기 |
| SECURITY-10 | 준수 | §1.2 불변 태그·digest 고정, §7.2 OIDC(장기 키 금지), §11 IaC 버전 고정 — 파이프라인 상세는 deployment-architecture.md |
| SECURITY-14 | 준수 | §8 보존 90일+·로그 삭제 Deny·알람 A14·§5.3 append-only |
| RESILIENCY-02·08·11·12 | 준수 | §3 Multi-AZ·백업 7일+PITR(RPO 시간 단위 상회)·§1.2 Multi-AZ 태스크. 복원 리허설은 Operations 이연(전역 결정 — 계획된 지연) |
| RESILIENCY-05·07·09 | 준수 | §8 알람 A1~A15(5xx·p95·DB·큐 적체·쿼터 80%·비용) |
| RESILIENCY-10 | 준수(인프라분) | 앱 어댑터 타임아웃·서킷은 각 유닛 소관 — 인프라는 A12 관측 제공 |
| SECURITY-03·04·05·08·11·12·13·15 / RESILIENCY-01·03·04·14·15 / PBT | 유닛·프로세스 문서 소관 | U1 매핑·NFR Design·deployment-architecture.md 참조 |
