# U4 여행 생성·필수 방문지 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U4 (M6 Trip Creation — 여행 CRUD·거점·필수 방문지·시간창·예산)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U4는 SI를 참조만 하고 재정의하지 않는다.** SI §0 유닛 표가 이미 **"U4 = 추가 없음(기존 컴퓨트·DB)"** 로 U4를 신규 인프라 무증분 유닛으로 예약해 두었다. 본 문서는 그 예약을 확인하고 U4의 유일한 변경(**RDS 스키마 확장 + `btree_gist` 확장 활성화**)만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md).

## 0. 요약 — U4 인프라 증분: 신규 AWS 리소스 0

U4는 **순수 CRUD·검증 유닛**(unit-of-work.md U4 — 신규 엔티티 약 5·**외부 연동 0·스케줄러 잡 0·신규 외부 시크릿 0**)이다. 서버측 신규 워크로드는 여행·거점·필수 방문지 CRUD API뿐이며, 전부 **U1 `server/app` 모놀리스와 기존 RDS에 흡수**된다. 따라서 U4는 **신규 AWS 리소스를 생성하지 않으며**, SI §1(ECS)·§2(ALB·VPC)·§3(RDS)·§6(시크릿)·§7(IAM)·§8(관측성)·§9(환경)을 그대로 상속한다. U4의 유일한 인프라 변경은 **RDS 스키마 확장(약 5 테이블) + `btree_gist` 확장 활성화**(Flyway 마이그레이션 — 기존 도구·리소스 내)다.

U4는 외부 API·캐시(ElastiCache)·비동기 브로커·스케줄러·S3·CloudFront 어느 것도 도입하지 않는다(U3와 달리 외부 의존이 0이므로 SI가 U3용으로 활성화한 NAT 아웃바운드·시크릿·외부 쿼터 알람 프레임조차 소비하지 않는다).

---

## 1. 여행·거점·필수 방문지 API의 ALB·ECS 배치 (신규 리소스 0)

- **ALB 라우팅**: SI §2.3·U1 infrastructure-design.md §1이 확정한 **단일 규칙**(443 → 대상 그룹 `trippilot-api`, 전 경로 `/api/*` + `/actuator/health/liveness`, 경로 분기 없음)에 여행 API(`/api/trips`·`/api/trips/{id}/stays`·`/api/trips/{id}/must-visits` 가칭)가 **그대로 흡수** — **ALB 규칙·대상 그룹·리스너 변경 0건.** 공개/인증 경계는 ALB가 아니라 Spring Security(전 U4 엔드포인트 인증 필수, NFR-U4-SEC-01)가 정본(이중 관리 드리프트 방지, U1 §1 원칙 상속).
- **ECS 컴퓨트**: 여행 CRUD·검증은 무상태 토큰 검증(PAT-PERF-01 상속) + 단일/소량 질의라 CPU·메모리 부하가 낮다. 계획 단계 저빈도 쓰기·조회(NFR-U4-SC-02, DAU 1천 정상 피크 수 RPS 미만)로 **SI §1.2의 1vCPU/2GB ×2 태스크가 흡수** — 추가 태스크·오토스케일 조정 불요(G142).
- **헬스체크**: SI·U1 §1의 shallow(`/actuator/health/liveness`)·deep(`/actuator/health/readiness`) 상속 — U4 신규 헬스 경로 없음.

---

## 2. RDS 스키마 확장 + btree_gist 확장 (Flyway — U4 유일 변경)

- **저장소**: SI §3의 **기존 RDS PostgreSQL**(단일 `trippilot` DB·`public` 스키마) 재사용 — **신규 DB 인스턴스·클러스터 없음**. U1 Flyway 파이프라인(배포 전 일회성 `app_migrate` 태스크, U1 infrastructure-design.md §3)으로 U4 마이그레이션 적용(forward-only, N-1 호환 규칙 — 신규 테이블 추가는 "자유", OPS-01).
- **U4 신규 테이블(약 5개, unit-of-work.md U4 산출물)**: 여행, 여행-거점 연결(조인·날짜 구간), 필수 방문지(POI 사본·시각 고정), 일자별 시간창, 여행 속성.
- **`btree_gist` 확장 활성화(U4 DB 스키마 증분의 핵심)**: 겹침 차단(D21)·거점 비중첩(D15) 하드 제약을 DB 레벨 EXCLUDE 배제 제약으로 강제하기 위해 `CREATE EXTENSION IF NOT EXISTS btree_gist`를 Flyway 마이그레이션으로 실행한다. 이는 **관리형 RDS PostgreSQL 기본 제공 확장(요활성 — CREATE EXTENSION 필요)**(추가 요금·신규 AWS 리소스 없음)이며, `app_migrate` 롤의 DDL 권한(SI §3 롤 모델)으로 생성한다.
  - 여행: `EXCLUDE USING gist (account_id WITH =, date_range WITH &&)` — 동일 계정 날짜 겹침 원자 차단(PAT-U4-01).
  - 거점: `EXCLUDE USING gist (trip_id WITH =, stay_range WITH &&)` — 동일 여행 거점 구간 비중첩(PAT-U4-02).
- **인덱스·볼륨**: 여행은 `account_id` 스코핑 조회 인덱스 + daterange GiST(겹침 조회 O(log n), NFR-U4-PERF-02). 볼륨은 계정당 활성 ≤1·이력 소량 → 여행 테이블 수만 행(NFR-U4-SC-01) — SI §3 gp3 20GB(오토스케일 100GB 상한) 내 충분, 파티셔닝 불요.
- **소유권·권한**: U4 테이블은 `app_user` 표준 DML(SI §3 롤 모델) — 법정 로그·동의 증적 같은 append-only REVOKE 대상 아님(U4는 일반 CRUD). 여행·거점·필수 방문지 소유권은 앱 레벨 accountId 스코핑(NFR-U4-SEC-01, IDOR 차단).

---

## 3. 외부 연동·시크릿·캐시·스케줄러 — U4 신규 없음 (참조만)

| 항목 | U4 처리 | 소유·근거 |
|---|---|---|
| **외부 API 아웃바운드(NAT)** | **해당 없음** — U4 외부 연동 0(장소 검색은 M7/U3 재사용). SI §2.1 NAT·`sg-app` 443 아웃바운드 미소비 | unit-of-work.md U4 |
| **Secrets Manager 항목** | **신규 0** — U4 외부 API 키·서비스 계정 키 없음 | SI §6 |
| **캐시(ElastiCache·인메모리)** | **해당 없음** — U4는 자체 데이터 조회, 외부 캐시 대상 없음(여행 조회는 인덱스 질의로 충분) | SI §4(메시징 — 캐시 전용 절 없음; ElastiCache는 U11 전환 트리거로만 언급), NFR-U4-SC-02 |
| **스케줄러·비동기 잡** | **없음**(unit-of-work.md U4 — 스케줄러 잡 없음). ShedLock·아웃박스 미소비 | SI §1.2 |
| **S3·CloudFront** | **해당 없음** — U4는 사진·정적 자산 없음(사진 U7 예약분 무관) | SI §5 |
| **등록 숙소·저장 POI 데이터** | U4 아님 — U3(M4·M7) 소유, U4는 CP1 소비자(스키마 계약) | unit-of-work.md U4 CP1 |

---

## 4. 관측성 — 기존 프레임 편입 (SI §8 상속, 신규 알람 0)

- U4는 신규 CloudWatch 알람·메트릭 리소스를 생성하지 않는다. 여행 CRUD 로그는 U1 구조화 로깅·PII 마스킹 컨버터·상관 ID(SI §8.1·PAT-OBS-01)를 상속하고, 여행 저장은 Critical 경로(§6.3)로서 기존 ALB p95·5xx·RDS 알람(SI §8.2 A1·A2·A6~A8)이 그대로 커버한다.
- U4 도메인 계측(하드 제약 위반율·파괴적 변경 확인 이탈률 등)이 필요하면 기존 대시보드(`trippilot-{env}-ops`) 위젯·로그 메트릭 필터로 편입(신규 알람 리소스 없음, PAT-OBS-03 상속). 외부 쿼터·서킷 알람(A11·A12)은 U4 무소비(외부 의존 0).

---

## 5. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 여행 CRUD는 기존 태스크 2개 흡수(저빈도·저부하) | §1, G142 |
| 네트워크 VPC·ALB·NAT (SI §2) | **0** | 0 — 기존 `/api/*` 규칙 흡수(경로 분기 없음). **NAT 아웃바운드 미소비**(외부 연동 0) | §1·§3 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 약 5개**(여행·거점 연결·필수 방문지·시간창·속성) + **`btree_gist` 확장 활성화**(EXCLUDE 제약) — 기존 RDS·Flyway 확장 | §2, unit-of-work.md U4 |
| 캐시 (SI §4 메시징 — 캐시 전용 절 없음) | **0** | 0 — U4 캐시 대상 없음(자체 데이터 인덱스 질의) | §3 |
| 스토리지 S3·CloudFront (SI §5) | **0** | 0 — U4 사진·정적 자산 없음 | §3 |
| 시크릿·KMS (SI §6) | **0** | 0 — U4 외부 키 없음(외부 연동 0) | §3 |
| IAM (SI §7) | **0** | 0 — 앱 태스크 역할 기존 권한 내(RDS 접근·로그·메트릭), 신규 API 호출 없음 | §1·§4 |
| 관측성 CloudWatch (SI §8) | **0** | 0 — 여행 저장 Critical은 기존 ALB·RDS 알람 커버, 도메인 계측은 기존 대시보드 편입 | §4 |
| 스케줄러 (SI §1.2) | **0** | 0 — U4 스케줄러 잡 없음 | §3 |

**총계: 신규 AWS 리소스 0 / 변경 = RDS 스키마 확장(테이블 약 5개) + `btree_gist` 확장 활성화 — 전부 기존 RDS·Flyway 내.** U4는 SI에 인프라를 사실상 추가하지 않으며(순수 CRUD 유닛·과설계 금지), SI가 예약한 "U4 = 추가 없음"을 그대로 이행한다. U4의 인프라 의의는 새 리소스가 아니라 **DB 레벨 무결성 제약(EXCLUDE)으로 하드 제약을 인프라 차원에서 강제**하는 것이다.

---

## 6. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U4는 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 그대로 재사용하며 U4 델타가 사실상 없다.

- **서버(M6 `modules/trip` + 여행·거점·필수 방문지 마이그레이션)**: `server/**` 경로이므로 `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). 스키마 확장·`btree_gist` 활성화·EXCLUDE 제약은 Flyway forward-only(신규 테이블·확장 추가 = N-1 호환 규칙 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속.
- **모바일(U4 화면 = 여행 생성 폼·목록·상세·거점 편집·필수 방문지 선택·기간 변경 다이얼로그)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **U4 화면은 순수 JS(React Hook Form·Zod·날짜 피커)이며 신규 네이티브 모듈이 없으므로 EAS 재빌드 델타 없음** — 대부분 expo-updates OTA 채널 대상(U3의 카카오 지도 SDK 같은 네이티브 델타가 U4에는 없다, tech-stack §6). 클라이언트 날짜 피커는 순수 JS 우선 채택으로 네이티브 재빌드를 회피(tech-stack U4-C2).
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. 여행 속성 기본값·시간창 기본(D29) 등 remote config성 값은 환경별 독립(SI §9).

**U4 배포 델타: 없음.** 서버는 기존 워크플로 흡수(스키마 확장은 Flyway), 모바일은 순수 JS 화면으로 EAS 재빌드 불요.

---

## 7. 컴플라이언스 요약 (U4 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-13 무결성 | **준수** | §2 — 겹침·비중첩 하드 제약을 DB EXCLUDE(`btree_gist`)로 인프라 차원 강제(PAT-U4-01·02) |
| SECURITY-08 접근 제어 | 준수(상속) | §1·§2 — 인증 경계 Spring Security, 여행·거점 소유권 앱 레벨 accountId 스코핑(nfr-requirements §4) |
| SECURITY-06·07 | 준수(상속) | §2 — `app_user` 표준 DML(최소 권한), 신규 아웃바운드·SG 변경 없음(외부 연동 0) |
| SECURITY-05 | 준수(상속) | 입력 검증·금칙어는 nfr-design(앱) — 인프라 신규 표면 없음 |
| SECURITY-10·12 | 상속 | 공급망·자격 증명은 U1 CI 상속 — U4 신규 시크릿 0(§3) |
| SECURITY-01·02·03·04·09·14·15 | 상속 | 암호화·중개 로깅·앱 로깅·헤더·하드닝·알림·예외 처리는 SI·U1 정본 — U4 신규 인프라 표면 없음 |
| RESILIENCY-01·02·08·11·12 | 상속 | 여행 저장 Critical(§6.3)·Multi-AZ·백업·PITR는 SI §3·U1 정본 — U4 신규 워크로드 없음 |
| RESILIENCY-10 | N/A | **U4 외부 의존 0** — 타임아웃·서킷·폴백 대상 없음(§3) |
| RESILIENCY-05·07·09 | 상속 | 관측성·헬스체크·오토스케일·쿼터는 SI §8·§1 정본 — U4 외부 쿼터·큐·신규 알람 없음(§4) |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — 겹침·비중첩·예산 왕복·한도·제목 결정성은 Code Generation 실행 |

**차단 소견 없음.** U4 신규 AWS 리소스 0·미결 항목 0(nfr-requirements.md §7) — SI가 예약한 "U4 = 추가 없음"을 그대로 이행하며, 유일한 변경(RDS 스키마 확장 + `btree_gist` EXCLUDE 제약)은 기존 RDS·Flyway 내에서 하드 제약을 인프라 차원 강제하는 것이다. U4는 외부 연동·스케줄러·시크릿·캐시가 없어 SI의 대다수 프레임(NAT·시크릿·외부 알람·CDN)을 소비조차 하지 않는다(과설계 금지).
