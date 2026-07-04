# U1 기반·계정·온보딩 — Infrastructure Design 실행 계획

> 2026-07-04 · CONSTRUCTION · U1 Infrastructure Design 스테이지 플랜
> **전역 확정 결정 반영**(u1-nfr-infra-questions.md Q1~Q7 사용자 답변, 2026-07-04): ①클라우드=**AWS 서울 리전(ap-northeast-2)** ②변경 관리=경량 프로세스(NFR Design 문서 소유) ③CI/CD=**GitHub Actions** ④롤백=**버전 고정 재배포(ECR 이미지 태그) + DB forward-only** ⑤배포=**롤링(Multi-AZ 2+ 인스턴스)** ⑥인시던트 대응=경량 프로세스(NFR Design 문서 소유) ⑦복원력 테스트=**Operations 이연**. 본 스테이지의 모든 선택은 이 7개 결정과 G142(DAU 1천 — 과설계 금지)를 제약으로 한다.
> U1은 기반 유닛이므로 **프로젝트 공유 인프라 정본(shared-infrastructure.md)을 본 스테이지에서 함께 정의**한다(U2~U8 재사용).

## 입력 검토

- [x] 전역 결정 7건 확인 (`u1-nfr-infra-questions.md` [Answer] 태그 — Q1=A(AWS)·Q2=A·Q3=A·Q4=A·Q5=A·Q6=A·Q7=A)
- [x] `u1-foundation/nfr-requirements/nfr-requirements.md` 검토 — 미결 PD-1~9 인수, 벤더 중립 요건(무상태성·헬스체크·append-only·알람 임계) 승계
- [x] `u1-foundation/nfr-requirements/tech-stack-decisions.md` 검토 — Spring Boot/Kotlin·Flyway·argon2id·Expo 전제 반영
- [x] `inception/application-design/unit-of-work.md` 검토 — 코드 조직(UW-3 모노레포)·U1~U8 인프라 소비 지점 매핑
- [x] `inception/requirements/requirements.md` §6.2(가용성)·§6.3(중요도)·§6.8(규모 G142) 반영
- [x] Security Baseline(`security-baseline.md`) SECURITY-01/02/06/07/09/10/14 검증 기준 대조

## 산출물 1 — 공유 인프라 정본 (`construction/shared-infrastructure.md`)

- [x] 컴퓨트: ECS Fargate vs EC2/EKS 비교 → **Fargate 확정**(운영 부담·규모 근거), 롤링 배포 설정(min 100%/max 200%·Circuit Breaker), min 2 태스크 Multi-AZ, 오토스케일 min 2/max 4·CPU 60%·RPS 트리거
- [x] 네트워크: VPC 2AZ(퍼블릭/프라이빗 앱/프라이빗 DB), 프라이빗 서브넷 IGW 직결 금지·NAT 경유(SECURITY-07), NAT 1개 결정+전환 트리거, S3 게이트웨이 엔드포인트, SG deny-by-default(ALB 443/80만 공개)
- [x] ALB: TLS 1.2+ 정책·80→443 리다이렉트, **액세스 로그 S3(SECURITY-02)**, WAF 이연 근거
- [x] DB: RDS PostgreSQL 16 Multi-AZ, **at-rest KMS·rds.force_ssl(SECURITY-01)**, 자동 백업 7일+PITR(RPO/RTO 시간 단위·CQ4 대비), 파라미터 원칙, DB 롤 3분리
- [x] 스토리지: S3 공통 기준선(버저닝·퍼블릭 차단·암호화·TLS-only), 사진 버킷+CloudFront **U7 예약**, **법정 로그 저장 비교(S3 Object Lock vs DB 테이블+권한 분리) → DB 테이블 확정** + 앱 역할 삭제 불가 IAM 명시 Deny
- [x] 메시징: 브로커 미도입 — DB 아웃박스+ShedLock(규모 정합 근거), 이메일=SES(샌드박스 해제 절차 주석)
- [x] 시크릿/키: Secrets Manager+KMS(AWS 관리형 키), JWT kid 롤오버 구조
- [x] 관측성: CloudWatch Logs(구조화 JSON·보존 90일+·S3 아카이브), 알람 목록 A1~A15(5xx율·p95·DB CPU·연결 수·큐 적체·쿼터 80% — RESILIENCY-05·07·09), SNS 알림 라우팅 2단
- [x] IAM: 태스크 역할 최소 권한·와일드카드 금지(SECURITY-06), 로그 파괴 명시 Deny, **GitHub Actions OIDC(장기 키 금지)**
- [x] 환경: dev/prod 2환경 — **계정 분리 권고** + 근거
- [x] 비용 개요: 월 추정치 표(prod ~$236·dev ~$60-90) + 상한 통제(Budgets·오토스케일 max)
- [x] IaC: Terraform vs CDK 비교 → **Terraform 확정**, 모듈·환경 스택 구성 계획

## 산출물 2 — U1 매핑 (`u1-foundation/infrastructure-design/infrastructure-design.md`)

- [x] PD-1~9 미결 항목 전건 종결표(SES·Secrets Manager·KMS·CloudWatch·RDS 세부·HIBP·캐시 미도입)
- [x] 인증 API ALB 라우팅·헬스체크 경로(shallow=LB·deep=관측 전용 — NFR-U1-AV-06)
- [x] 규모 매핑 — argon2id CPU 산정과 태스크 사이징 정합(NFR-U1-SC-01)
- [x] RDS 스키마·마이그레이션 실행 방식 — 배포 전 Flyway 일회성 ECS 태스크·앱 기동 마이그레이션 비활성·권한 DDL 포함
- [x] SES 인증 메일 — 도메인 검증·SPF/DKIM/DMARC·Configuration Set·샌드박스 해제(P9 도메인 연계)
- [x] 소셜 IdP 아웃바운드 — NAT 경로·시크릿 4종 보관(Apple p8 동적 서명)·HIBP k-익명
- [x] 레이트리미터 저장 — 인메모리 vs ElastiCache 비교 → **PostgreSQL 확정(MVP)** + 도입 트리거
- [x] 법정 로그·동의 증적 저장 구현 — append-only 2중 강제(DB REVOKE+IAM Deny)·6개월 보존·CI 권한 검증
- [x] U1 알람 셋(U1-S1~S7 — 리프레시 재사용 즉시 P1 포함)
- [x] 컴플라이언스 요약(SECURITY·RESILIENCY·PBT — 차단 소견 없음)

## 산출물 3 — 배포 아키텍처 (`u1-foundation/infrastructure-design/deployment-architecture.md`)

- [x] 모노레포 경로 필터(server/·apps/mobile·infra) 워크플로 분리
- [x] PR 워크플로 — 빌드·테스트·**하드 제약 게이트 100%(D37/G114)**·**PBT 시드 로깅(PBT-08)**·취약점 스캔·SBOM(SECURITY-10)
- [x] main 머지 → ECR **태그 고정(불변) 푸시** → 마이그레이션 태스크 → ECS 롤링 배포(Mermaid 다이어그램 — 영숫자 노드 ID+텍스트 대안 병기)
- [x] DB 마이그레이션 순서 — forward-only·배포 전 실행·N-1 호환 규칙 5개
- [x] 롤백 절차 — 이전 태그 재배포 runbook(자동 Circuit Breaker+수동 workflow_dispatch)·PITR 시나리오(복원 리허설 Operations 이연 명기)
- [x] 환경 승격 흐름 — dev 자동/prod 승인 게이트·동일 아티팩트 재사용·직무 분리(SECURITY-13)
- [x] 모바일 EAS Build 개요(스토어 제출은 Operations)
- [x] 컴플라이언스 요약(파이프라인 관점)

## 검증

- [x] 콘텐츠 검증 — Mermaid 노드 ID 영숫자·텍스트 대안 병기, ASCII 다이어그램은 `+ - | < >` 문자만 사용, 특수문자 이스케이프 확인
- [x] 전 선택에 규모 근거(G142) 또는 전환 트리거 병기 — 과설계 금지 원칙 준수 확인
- [x] 추적 ID 표기(SECURITY-*·RESILIENCY-*·NFR-U1-*·PD-*·D/G/N/P/CQ) 일관성 확인
- [x] nfr-requirements.md §8 '대기' 항목(PD-1~9) 전건 종결 확인
