# U1 NFR Design·Infrastructure Design 사용자 결정 질문

> **2026-07-04** · 아래 7개는 Resiliency Baseline 규칙(RESILIENCY-03·04·14·15)이 "AI가 대신 결정 금지 — 반드시 사용자에게 질문"으로 지정한 항목 + 클라우드 벤더 선택입니다. **프로젝트 전역 결정**이므로 U1에서 한 번 답하면 이후 전 유닛에 재사용됩니다.
> 각 질문의 `[Answer]:` 태그에 선택지 문자를 기입하고 완료를 알려주세요.

## Question 1 — 클라우드 벤더
서버·DB·스토리지를 어느 클라우드에 배포할까요? (Multi-AZ·관리형 PostgreSQL·오브젝트 스토리지+CDN·푸시 제외 이메일 발송이 필요)

A) AWS (권장) — 관리형 서비스 폭·국내 리전(서울)·AI-DLC 참조 아키텍처 풍부. RDS/S3/CloudFront/SES 등

B) GCP — Cloud SQL/GCS, FCM과 같은 생태계

C) Azure

D) 클라우드 미사용·자체 호스팅 (아래 [Answer]: 태그 뒤에 환경 서술)

E) 기타 (아래 [Answer]: 태그 뒤에 직접 서술)

[Answer]: A

## Question 2 — 변경 관리 프로세스 (RESILIENCY-03)
프로덕션 변경을 어떤 절차로 관리할까요?

A) 공식 프로세스 없음 — AI-DLC가 경량 프로세스 제안 (변경 기록 + PR 승인 게이트 + 롤백 노트) (개인·소규모 팀 권장)

B) 조직의 기존 변경 관리 프로세스 사용 (아래 [Answer]: 태그 뒤에 도구/절차명 기입)

C) 면제 — 공식 변경 관리 불요 (사유 기입)

[Answer]: A

## Question 3 — CI/CD 도구 (RESILIENCY-04)
빌드·배포 파이프라인은 무엇으로 할까요?

A) GitHub Actions (권장) — 저장소가 GitHub(Jira 연동 이력 기준), 모노레포 워크플로 용이

B) 기존 파이프라인 사용 (아래 [Answer]: 태그 뒤에 도구명 기입)

C) 기타 (직접 서술)

[Answer]: A

## Question 4 — 롤백 메커니즘 (RESILIENCY-04)
배포 실패 시 어떻게 되돌릴까요?

A) 이전 버전 아티팩트 재배포 (버전 고정 롤백) (권장 — 단일 배포 모놀리스에 적합, DB 마이그레이션은 forward-only+호환 규칙 병행)

B) 블루/그린 스왑백

C) 카나리 자동 롤백 (헬스·지표 회귀 기준)

D) 기타 (직접 서술)

[Answer]: A

## Question 5 — 배포 스타일 (RESILIENCY-04)
릴리스 전략은 무엇으로 할까요?

A) 롤링 배포 (권장) — Multi-AZ 2+ 인스턴스 점진 교체, 무중단·비용 균형

B) 직접(in-place) — 최저 비용, 짧은 다운타임 허용

C) 블루/그린 — 무중단 확실, 비용 2배 구간 존재

D) 카나리 — 점진 트래픽 전환 (초기 규모 대비 과설계 가능)

[Answer]: A

## Question 6 — 인시던트 대응 (RESILIENCY-15)
프로덕션 장애는 어떻게 다룰까요?

A) 공식 프로세스 없음 — AI-DLC가 경량 프로세스 제안 (알림 라우팅 + 대응 절차 문서 + 사후 회고 템플릿) (권장)

B) 조직의 기존 온콜/IR 프로세스 사용 (아래 [Answer]: 태그 뒤에 참조 기입)

[Answer]: A

## Question 7 — 복원력 테스트 (RESILIENCY-14)
장애 조치·복구 검증은 어떻게 할까요?

A) Operations 단계로 이연 — 지금은 테스트 시나리오만 문서로 확보 (권장 — MVP 규모 정합)

B) AI-DLC가 DR 테스트 일정·카오스 실험 계획 제안 (조기 채택)

C) 기존 조직 관행 사용 (참조 기입)

[Answer]: A
