# U2 앱 셸·홈·내비게이션 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U2 (클라이언트 셸·홈·내비게이션 + server/app 부트스트랩·홈 집약 API)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U2는 SI를 참조만 하고 재정의하지 않는다.** 본 문서는 U2가 SI에 추가하는 **인프라 증분(사실상 없음)**만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md).

## 0. 요약 — U2 인프라 증분: 신규 AWS 리소스 0

U2는 클라이언트 중심 경량 유닛(unit-of-work.md U2 — 신규 엔티티 약 1·외부 연동 0·서버 로직 얕음)이다. 서버측 신규 워크로드는 **부트스트랩 API + 홈 집약 조회 2종**뿐이며, 둘 다 **U1 `server/app` 모놀리스에 흡수**된다(tech-stack §7). 따라서 U2는 **신규 AWS 리소스를 생성하지 않으며**, SI §1(ECS)·§2(ALB·VPC)·§3(RDS)·§6(시크릿)·§8(관측성)·§9(환경)을 그대로 상속한다. U2 인프라 결정은 딱 하나 — **최소 지원 버전 값의 저장·조회 방식**(§2)이고, 이마저 기존 메커니즘 선택이다.

---

## 1. 부트스트랩·홈 API의 ALB·ECS 배치 (신규 리소스 0)

- **ALB 라우팅**: SI §2.3·U1 infrastructure-design.md §1이 확정한 대로 ALB는 **단일 규칙**(443 → 대상 그룹 `trippilot-api`, 전 경로 `/api/*` + `/actuator/health/liveness`)이며 경로 분기가 없다. 부트스트랩(`/api/app/bootstrap` 가칭)·홈 집약 조회(`/api/home` 가칭)는 **기존 `/api/*` 규칙에 그대로 흡수** — **ALB 규칙 변경 0건, 대상 그룹·리스너 변경 0건.** 공개/인증 경계는 ALB가 아니라 **Spring Security 화이트리스트가 정본**(NFR-U1-SEC-16이 "부트스트랩 확인 중 비세션 부분(최소 지원 버전)"을 이미 공개 화이트리스트로 예약) — U2는 그 예약된 화이트리스트 항목을 소비할 뿐 ALB에서 중복 관리하지 않는다(이중 관리 드리프트 방지, U1 §1 원칙 상속).
- **ECS 컴퓨트**: 부트스트랩·홈 조회는 무상태 검증(PAT-PERF-01 상속) + 단일/소량 질의라 CPU·메모리 부하가 낮다. 부트스트랩 피크 ~0.28 RPS(U1 nfr §2.1 산정 — U2 재산정 없음), 홈 조회도 앱 진입당 1회로 유사 빈도. **SI §1.2의 1vCPU/2GB ×2 태스크가 흡수** — 추가 태스크·오토스케일 조정 불요(G142, nfr-requirements.md §6).
- **헬스체크**: SI·U1 §1의 shallow(`/actuator/health/liveness`)·deep(`/actuator/health/readiness`) 상속 — U2 신규 헬스 경로 없음.

---

## 2. 최소 지원 버전 값의 저장·조회 방식 (U2 유일 인프라 결정)

강제 업데이트 게이트(N4·PAT-U2-05)는 서버가 `minSupportedVersion`을 제공해야 하며, **오구성 시 재배포 없이 되돌릴 수 있어야 한다**(unit-of-work.md U2 리스크 완화: "최소 버전은 remote config로 관리+변경 이력, 확인 불가 시 페일오픈"). 저장 위치를 비교한다.

| 기준 | **A. 서버 DB 앱 구성 테이블 (권고)** | B. AWS SSM Parameter Store | C. 앱 빌드 정적 설정 |
|---|---|---|---|
| 재배포 없이 변경 | **가능** — 행 UPDATE, 부트스트랩 응답 즉시 반영 | 가능 — 파라미터 수정 | **불가** — 앱/서버 재배포 필요(게이트 목적 위배) |
| 신규 인프라 표면 | **0** — 부트스트랩이 이미 DB 접근, U1 Flyway·RDS 재사용 | +1 — SSM 읽기 경로·IAM 권한·엔드포인트 추가(SI §2.1 인터페이스 엔드포인트 미도입 기조와 상충) | 0 |
| 변경 이력(경량 변경 관리) | 마이그레이션/운영 UPDATE + append 이력 컬럼으로 기록(OPS-01 연동) | 파라미터 버전 이력 내장 | Git 이력(단 재배포 필요) |
| 조회 성능 | 인메모리 TTL 캐시(수 분) — 부트스트랩 경로 무부하 | API 호출·캐시 별도 | 무조회(정적) |
| 규모 정합(G142) | **적합** — 값 1개에 새 서비스 불요 | 값 1개에 SSM 도입은 과설계 | — |

- **확정 권고: A(서버 DB 앱 구성 테이블).** 근거: (1) unit-of-work.md U2 산출물이 이미 "앱 구성(최소 지원 버전 등 remote config성 설정) 1개 내외" DB 마이그레이션을 예정, (2) 부트스트랩 API가 이미 DB에 접근하므로 **신규 인프라 리소스 0**(U1 RDS·Flyway 재사용), (3) 값 1개를 위해 SSM Parameter Store를 도입하는 것은 인터페이스 엔드포인트·IAM 표면을 늘려 SI §2.1의 "엔드포인트 고정비 회피" 기조와 상충하는 과설계. **인메모리 TTL 캐시(수 분)**로 부트스트랩 경로의 DB 질의를 무부하화(NFR-U2-PERF-01 p95 500ms 여유).
- **B 비채택 사유(1줄)**: 값 하나에 SSM 읽기 경로·권한·엔드포인트를 추가하는 인프라 표면 증가가 이득을 상회(G142). **C 비채택 사유(1줄)**: 재배포 없이 되돌릴 수 없어 게이트 오구성 대응(전 사용자 차단 리스크)이 불가능.
- **운영 안전**: 값 변경은 경량 변경 관리(OPS-01 — PR/운영 승인 + 변경 이력) 대상. dev/prod 환경별 독립 값(SI §9 환경 분리 상속). 강제 업데이트 게이트는 **확인 불가 시 페일오픈**(PAT-U2-02·05)이라 이 값 조회 실패가 전 사용자 차단으로 번지지 않는다.

---

## 3. 해당 없음·타 유닛 소유 (참조만)

| 항목 | U2 처리 | 소유 |
|---|---|---|
| **CloudFront·정적 자산 호스팅** | **해당 없음** — 모바일 앱 화면은 CDN이 아니라 **EAS 빌드 → 앱/플레이 스토어 배포**(deployment-architecture.md §7). U2 웹 정적 오리진 없음 | SI §5.4 CloudFront는 U7(사진)·U10(어드민 웹) 예약분 — U2 무관 |
| **홈 '인기 장소' 집계 잡** | **U2 아님 — 참조만.** 홈 카드 슬롯(레이아웃·빈 상태)만 U2, 데이터 공급(최근 7일 저장+방문 가중합 일 1회 배치, G2)은 U3/M7 소유 | U3 Place Data(M7), SI §1.2 ShedLock 스케줄 잡 프레임 |
| 홈 여행·활성 일정·추억 카드 데이터 | 슬롯 계약(스키마)만 U2 고정, 데이터는 후행 유닛 | U4(여행)·U6(활성 일정)·U7(추억)·U8(최종 통합) |
| 스토어 이동(강제 업데이트) | 아웃바운드 링크(`Linking.openURL`) — 인프라 신규 요소 아님 | U2 클라이언트(NAT·서버 무관) |
| 스케줄러 잡 | **없음**(unit-of-work.md U2 — 스케줄러 잡 없음) | — |

---

## 4. SI 대비 델타 표

| 계층 (SI 절) | 신규 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 부트스트랩·홈은 기존 태스크 2개 흡수 | §1, G142 |
| 네트워크 VPC·ALB (SI §2) | **0** | 0 — 부트스트랩·홈은 기존 `/api/*` 규칙 흡수(경로 분기 없음). 공개 경계는 Spring Security 화이트리스트(U1 기예약) | §1, NFR-U1-SEC-16 |
| 데이터베이스 RDS (SI §3) | **0** | **앱 구성 테이블 1개**(최소 지원 버전 등) — U1 Flyway·RDS 재사용, 신규 AWS 리소스 아님 | §2, unit-of-work.md U2 |
| 스토리지 S3·CloudFront (SI §5) | **0** | 0 — 모바일 EAS/스토어 배포, CDN 무관 | §3 |
| 시크릿·KMS (SI §6) | **0** | 0 — U2 신규 시크릿 없음(외부 연동 0) | unit-of-work.md U2 |
| 관측성 CloudWatch (SI §8) | **0** | 0 — 클라이언트 계측(3초 타임아웃 발동률·크래시율)은 기존 알람 프레임(PAT-OBS-03 클라이언트 행) 편입, 신규 알람 리소스 없음 | nfr-requirements.md §4, PAT-OBS-03 상속 |

**총계: 신규 AWS 리소스 0 / 변경 1건(앱 구성 테이블 — 기존 RDS·Flyway 내).** U2는 SI에 인프라를 사실상 추가하지 않는다(경량 유닛·과설계 금지).

---

## 5. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U2는 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드가 생성한 프로젝트 공통 정본)을 그대로 재사용하며, U2 델타가 없기 때문이다.

- **서버(부트스트랩·홈 API·앱 구성 마이그레이션)**: `server/**` 경로 안이므로 `server-ci.yml`·`server-deploy.yml`(deployment-architecture.md §1·§2·§3) **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 파이프라인을 완성한 근거). 앱 구성 테이블 추가는 Flyway 마이그레이션(§4 forward-only·N-1 호환 규칙 — 신규 테이블 추가는 규칙 1 "자유")로 배포 전 일회성 태스크 실행. 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속.
- **모바일(U2 화면 = 스플래시·5탭 셸·홈·온램프)**: `apps/mobile/**` 경로이므로 `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **U2 화면은 EAS Build(deployment-architecture.md §7)의 development/preview 프로파일에 자연 포함** — 별도 빌드 파이프라인 델타 없음. Expo Router 라우트 추가(U2-C1~C3)는 JS 번들 변경이라 대부분 OTA 채널 대상이나, **강제 업데이트 게이트(N4)는 네이티브 스토어 빌드 교체 기준**이므로 expo-updates OTA와 구분(tech-stack §6) — OTA 정책은 Operations 소관(참조만).
- **환경**: dev(자동)·prod(승인 게이트) 승격 흐름(deployment-architecture.md §6) 상속. 최소 지원 버전 값은 환경별 독립(SI §9).

**U2 배포 델타: 없음.** 서버는 기존 워크플로 흡수, 모바일은 EAS 빌드에 U2 화면 포함.

---

## 6. 컴플라이언스 요약 (U2 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-01·02·06·07·09·10 | 상속 | SI·U1 정본 — U2 신규 인프라 리소스 0(§0·§4), 특이사항 없음 |
| SECURITY-08 | 준수(상속) | 부트스트랩·홈 공개/인증 경계는 Spring Security 화이트리스트(NFR-U1-SEC-16), 홈 소유권 스코핑(nfr-requirements.md §4) |
| SECURITY-03·14 | 상속 | 부트스트랩·홈 로그도 U1 마스킹 컨버터·알람 프레임 편입 — 신규 로그 그룹·보안 이벤트 없음 |
| RESILIENCY-01·02·08·11·12 | 상속 | Multi-AZ·백업·워크로드 분류는 SI·U1 — U2 신규 워크로드 없음(부트스트랩은 M1 Critical 상속) |
| RESILIENCY-05·06·07·09 | 상속 | 관측성·헬스체크·알람은 SI §8·U1 정본 — 클라이언트 타임아웃 발동률 계측만 편입 |
| RESILIENCY-10 | 준수 | 부트스트랩 타임아웃·G5 폴백·홈 부분 실패(nfr-design PAT-U2-01·02·04) — 외부 의존 0이라 신규 서킷 없음 |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속. 최소 버전 값 변경은 OPS-01 대상(§2) |
| SECURITY-04·05·11·12·13·15 | 상속/N/A | 헤더·입력 검증·자격 증명·예외 처리는 U1 스캐폴드 상속, LLM 경계(11)는 U2 무관 |
| PBT-01~10 | 상속 | CI 실행 요건은 deployment-architecture.md §2(시드 로깅) 상속 — U2 스플래시 분기·탭 보존 속성은 Code Generation에서 실행 |

**차단 소견 없음.** U2 신규 AWS 리소스 0·미결 항목 0(nfr-requirements.md §7) — SI·U1 정본 상속으로 인프라 설계가 종결된다. 유일한 유닛 증분(최소 지원 버전 저장 = 서버 DB 앱 구성 테이블, §2)은 기존 RDS·Flyway 내 변경으로 신규 인프라가 아니다.
