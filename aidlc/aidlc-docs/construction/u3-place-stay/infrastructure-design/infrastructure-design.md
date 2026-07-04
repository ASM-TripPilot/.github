# U3 숙소·장소 데이터 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U3 (M7 Place Data · M3 탐색 · M4 등록 · M5 OTA 딥링크 + `shared/map` 카카오 지도 SDK)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U3는 SI를 참조만 하고 재정의하지 않는다.** 본 문서는 U3가 SI에 추가하는 **인프라 증분**(외부 아웃바운드·POI 정본 저장·단기 캐시·배치 잡)만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md).

## 0. 요약 — U3 인프라 증분: 신규 AWS 리소스 사실상 0 (외부 아웃바운드·시크릿·스키마 확장)

U3는 외부 의존 최다 유닛이나, **신규 AWS 리소스는 사실상 0**이다 — SI가 이미 NAT 아웃바운드(§2.1)·시크릿 프레임(§6)·외부 쿼터/실패율 알람 프레임(§8.2 A11·A12)·ShedLock 스케줄러 프레임(§1.2)을 U3(TourAPI·카카오·네이버)을 명시적 소비자로 예약해 두었기 때문이다(SI §0 유닛 표). U3 인프라 결정은 넷 — **① 외부 아웃바운드·시크릿 매핑, ② POI 정본 저장(RDS 스키마 확장), ③ 단기 캐시 계층(인메모리 vs ElastiCache), ④ 배치 잡(기존 스케줄러 재사용)** 이며, 전부 기존 리소스 내 확장/설정이다.

**선결 경계**: POI **정본 테이블·스냅샷의 확정 스키마**는 **P2(지도 약관)·P4(TourAPI 캐싱 조건)** 선결 후 Functional Design 소관이다(nfr-requirements.md §7.1). 본 문서는 저장 위치·격리·정책 컬럼 구조까지 확정한다.

---

## 1. 외부 API 아웃바운드 · 시크릿 보관 (SI §2·§6 상속)

- **아웃바운드 경로**: ECS(프라이빗 앱 서브넷) → **NAT Gateway**(SI §2.1, 기존 1개) → 외부 엔드포인트. `sg-app` 아웃바운드 443은 SI §2.2가 이미 허용("목적지 IP 가변 SaaS 예외 문서화") — **SG 변경 0**. U3 아웃바운드 목적지 문서화(egress 허용 목록):

| 목적지 | 어댑터(포트) | 용도 |
|---|---|---|
| 카카오 로컬 API | `PlaceSearchPort`·`GeocodingPort` | 장소 검색·지오코딩(D08) |
| TourAPI(공공데이터포털) | `TourContentPort` | POI·숙소 정적 콘텐츠(P4 선결) |
| 네이버 장소 API | 네이버 폴백 어댑터 | 2차 폴백(D08) |
| OTA 딥링크 도메인 | `DeeplinkBuilder`(외부 호출 없음) | URL 조립만 — 아웃바운드 미발생(크롤링 금지 §6.5) |

- **시크릿(SI §6 프레임)**: 외부 API 키를 **Secrets Manager**에 신규 시크릿 항목으로 등록(신규 서비스 아님 — SI §6 프레임 내 항목 추가): `external/kakao-rest`(카카오 REST 키), `external/tourapi`(서비스 키), `external/naver`(클라이언트 ID/시크릿). TMap 키(`external/tmap`)는 **U5에서 등록**(도로거리 C2 소유). ECS 태스크 정의 `secrets` ARN 참조 주입 — 코드·이미지·Terraform state 평문 0(NFR-U3-SEC-01·NFR-U1-SEC-11). 클라이언트 카카오 지도 SDK 앱 키는 앱 config plugin에 주입(네이티브 키 — 서버 시크릿과 별개, EAS 시크릿/환경).
- **어댑터 타임아웃·서킷**(NFR-U3-RES-01)은 앱 소관(Resilience4j) — 인프라는 SI §8.2 **A12(어댑터 실패율·서킷 open)·A11(쿼터 80%)** 알람을 제공(U3가 첫 소비자).

---

## 2. POI 정본 저장 — RDS 스키마 확장 (Flyway)

- **저장소**: SI §3의 **기존 RDS PostgreSQL**(단일 `trippilot` DB·`public` 스키마) 재사용 — **신규 DB 인스턴스·클러스터 없음**. U1 Flyway 파이프라인(U1 infrastructure-design.md §3 — 배포 전 일회성 `app_migrate` 태스크)으로 U3 마이그레이션 적용.
- **U3 신규 테이블(~8~9개, unit-of-work.md U3 산출물)**: POI 정본, POI 소스 매핑(place_id N:1 D17), POI 스냅샷(사용자 확정), 탐색 캐시(허용 소스 파생 — §3 참조), 위시리스트, 등록 숙소, 숙소 ID 소스 매핑, 클릭 로그, 인기 장소 집계.
- **소스별 저장 정책(D13 — nfr-requirements.md §4)을 스키마로 강제**:
  - **정본 테이블**: TourAPI 등 **허용 소스만** 영구 저장(D13 (a)) + **보존 정책 컬럼**(`source`·`retention_policy`·`cached_at`) — P4 캐싱 조건 확정 시 소급 축소.
  - **지도 API 소스**: 정본 테이블에 **영구 저장하지 않음**(D13 (b)·P2) — 인메모리 TTL(§3). 아키텍처 테스트로 "지도 소스 → 정본 테이블 쓰기" 부재 검증.
  - **스냅샷**: 최소 필드 + 정책 컬럼(NFR-U3-DATA-03) — P2 확정 전 지도 소스 스냅샷 보류.
- **볼륨**: 지역당 POI 5천 × 국내 지역 수십 = 수십만 행(NFR-U3-SC-01) — SI §3 gp3 20GB(오토스케일 100GB 상한) 내 충분. 정본 테이블은 **지역·카테고리 인덱스**(탐색·closed-set 후보 조회 G115), 파티셔닝 여지 확보(NFR-U1-SC-03 상속).
- **소유권·권한**: U3 테이블은 `app_user` 표준 DML(SI §3 롤 모델) — 법정 로그·동의 증적 같은 append-only REVOKE 대상 아님(U3는 일반 CRUD). 위시리스트·등록 숙소 소유권은 앱 레벨 accountId 스코핑(NFR-U3-SEC-05).

---

## 3. 단기 캐시 계층 결정 — 인메모리 vs ElastiCache (규모 대비 권고)

지도 API 소스·핫 탐색 결과의 단기 TTL 캐시(D13 (b)·nfr-requirements.md §4.2)의 저장 위치를 비교한다.

| 기준 | **A. 인메모리 Caffeine (권고)** | B. ElastiCache Redis | C. DB TTL 테이블 |
|---|---|---|---|
| 정본 적합성 | 캐시=최적화(미스=외부 재호출, NFR-U1-AV-03 상속) — 태스크 간 불일치 무해 | 충족(공유) | 충족 |
| 지도 약관(P2) | **영구 저장 표면 0** — 프로세스 메모리·TTL만(리스크 최소) | 외부 저장소 상주(TTL이나 표면↑) | DB 영구 스토리지 상주(약관 경계 근접) |
| 신규 AWS 리소스 | **0** | **+1 서비스·SG·서브넷·월 ~$25** | 0(RDS 내, 정리 잡 추가) |
| 규모 정합(G142) | **적합** — 수 RPS·POI 5천/지역, 태스크 2GB 힙 여유(SI §1.2) | 이득 0(운영 표면만) | RDS I/O·정리 부하 추가 |
| 쿼터 보호(G196) | 태스크당 캐시 → 최대 태스크 수배 호출(0.5 RPS에서 무의미) | 공유로 최소화 | 공유 |

- **확정 권고: A(인메모리 Caffeine · 단기 TTL) · ElastiCache 미도입.** 근거: (1) 지도 API 소스는 **영구 저장 금지**(D13·P2)라 인메모리 TTL이 약관 리스크를 최소화, (2) 캐시는 정본이 아니므로 태스크 간 불일치는 재호출로 무해(NFR-U1-AV-03), (3) **신규 AWS 리소스 0**(G142). TourAPI 허용분은 §2 정본 테이블이 폴백 소스를 겸한다(별도 캐시 불요).
- **B(ElastiCache) 비채택 사유(1줄)**: 값·규모 대비 월 비용·운영 표면 증가가 이득을 상회 — SI §4와 **동일 트리거(인증 계열 포함 실측 50 RPS 초과 또는 U11 WebSocket 팬아웃)** 로 통합 재평가. **C(DB TTL) 비채택 사유(1줄)**: 지도 API 데이터를 DB 영구 스토리지에 두는 것이 P2 약관 경계에 근접.

---

## 4. 배치 잡 — 기존 ShedLock 스케줄러 재사용 (신규 리소스 0)

U3 스케줄 잡은 SI §1.2 확정 모델(앱 프로세스 내 Spring `@Scheduled` + **PostgreSQL ShedLock 분산 락**)을 재사용한다 — **별도 워커 서비스·EventBridge 없음**(전부 일 1회 저부하, G142).

| 잡 | 주기 | 내용 | 근거 |
|---|---|---|---|
| POI 정본 동기화 | **일 1회** | TourAPI 정적 콘텐츠 수집·정규화·canonical 발급(G133) | G196, unit-of-work.md U3 |
| 인기 장소 집계 | **일 1회** | 최근 7일 저장+방문 가중합(홈 '인기 장소' 카드 공급 — U2 슬롯) | G2 |
| OTA 정적 콘텐츠 갱신 | **일 1회** | 정적 콘텐츠 갱신(라이브 가격 제외 — 온디맨드 PAT-U3-03) | G196 |
| 커버리지 게이트 측정 | 주기(일 1회) | 지역별 좌표 확보율·영업시간 채움률 산출 → 커스텀 메트릭(§5) | G192 |

- ShedLock 락으로 태스크 2개 중복 실행 배제(SI §1.2). 배치 커넥션 풀은 탐색 처리 풀과 분리(U1 PAT-PERF-02 벌크헤드 상속 — 배치가 탐색 커넥션 고갈 방지). **전환 트리거**: 배치가 탐색 API p95에 간섭 시 EventBridge Scheduler + ECS RunTask 분리(SI §1.2 트리거 상속).

---

## 5. 커버리지 게이트 측정·외부 알람 (SI §8.2 구체화)

- **커버리지 메트릭(NFR-U3-GATE-02)**: 측정 잡이 지역별 좌표 확보율·영업시간 채움률을 CloudWatch **커스텀 메트릭**(`cloudwatch:PutMetricData` — SI §7.1 태스크 역할 허용)으로 발행 → 대시보드(SI §8.2 `trippilot-{env}-ops` 위젯 추가) + 출시 게이트 판정. 임계 미달(좌표 <95%·영업시간 <70%, G192)은 P3 주의 알람.
- **외부 API 알람(SI §8.2 — U3가 첫 소비자)**:

| 알람(SI 프레임) | U3 소스 | 임계 | 단계 |
|---|---|---|---|
| A11 외부 쿼터 80% | 카카오·TourAPI·네이버 호출량 vs 문서화 쿼터 | 80% | P2 |
| A12 어댑터 실패율·서킷 open | 카카오·TourAPI·네이버 어댑터 | 실패율 >20%(5분) 또는 서킷 open | P2 |
| U3 커버리지 게이트 미달 | 지역별 채움률 | 좌표 <95% / 영업시간 <70% | P3 |

- 알람 라우팅은 SNS `trippilot-{env}-alerts` → OPS-02 프로세스(SI §8.2). 쿼터 한도는 P4(TourAPI)·벤더 콘솔 확인 후 문서화(NFR-U3-SC-05).

---

## 6. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 탐색·외부 호출·배치는 기존 태스크 2개 흡수(I/O 바운드·저빈도) | §1·§4, G142 |
| 네트워크 VPC·NAT·ALB (SI §2) | **0** | 0 — NAT 아웃바운드·`sg-app` 443·`/api/*` 규칙 재사용. 아웃바운드 허용 목록 **문서화만** | §1, SI §2.1·2.2 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 ~8~9개**(POI 정본·매핑·스냅샷·탐색 캐시·위시리스트·등록 숙소·클릭 로그·인기 집계) — 기존 RDS·Flyway 확장 | §2, unit-of-work.md U3 |
| 캐시 (신규 계층) | **0** | **인메모리 Caffeine**(앱 내) — ElastiCache 미도입 | §3, NFR-U3-DATA-05 |
| 시크릿·KMS (SI §6) | **0** | **Secrets Manager 시크릿 항목 3개**(kakao·tourapi·naver) — 기존 서비스 내 항목 | §1, SI §6 프레임 |
| 관측성 CloudWatch (SI §8) | **0** | 커버리지 커스텀 메트릭·A11·A12 활성화·대시보드 위젯 — 기존 프레임 편입 | §5, SI §8.2 A11·A12 |
| 스토리지 S3·CloudFront (SI §5) | **0** | 0 — U3는 사진 없음(사진 U7 예약분 무관) | SI §5 |

**총계: 신규 AWS 리소스 0 / 변경 = RDS 스키마 확장(테이블 ~8~9개) + Secrets Manager 항목 3개 + 알람·메트릭 활성화 + 인메모리 캐시(앱 내).** U3는 외부 의존이 크지만 인프라는 SI가 예약한 프레임(NAT·시크릿·알람·스케줄러) 내에서 종결된다(과설계 금지).

---

## 7. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U3는 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 재사용한다. 단, **클라이언트 지도 SDK는 네이티브 모듈이므로 EAS 빌드 델타가 있다**(U2와 달리 순수 JS가 아님).

- **서버(M3·M4·M5·M7 모듈 + POI 마이그레이션)**: `server/**` 경로이므로 `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). POI 정본·매핑·캐시 테이블 추가는 Flyway forward-only(신규 테이블 추가 = N-1 호환 규칙 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속. 외부 시크릿(§1)은 배포 전 Secrets Manager 등록.
- **모바일(U3 화면 + `shared/map` 카카오 지도 SDK)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **단, 카카오 지도 네이티브 SDK config plugin은 네이티브 모듈 추가라 development/preview EAS Build 재빌드가 필요**하고 **expo-updates OTA로는 반영 불가**(네이티브 변경 — tech-stack §2·SI §5.4/U2 §5의 OTA vs 네이티브 구분 상속). **리스크 완화(unit-of-work.md U3)**: config plugin 검증을 U3 초기 **스파이크로 선행**, 실패 시 목록/WebView 지도 폴백 빌드. 지도 앱 키는 EAS 시크릿 주입(§1).
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. 외부 API 키·딥링크 화이트리스트·폴백 순서 remote config는 환경별 독립(SI §9).

**U3 배포 델타**: 서버는 기존 워크플로 흡수(0 델타). 모바일은 **카카오 지도 SDK config plugin으로 인한 네이티브 재빌드**(EAS)가 유일한 델타 — OTA 아닌 스토어/개발 빌드 경로.

---

## 8. 컴플라이언스 요약 (U3 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-10 | **준수** | §1·§5 — 어댑터 타임아웃·서킷은 앱(Resilience4j), 인프라는 A12·A11 알람 제공. 폴백은 nfr-design PAT-U3-01 |
| RESILIENCY-09 | 준수 | §5 — 외부 쿼터 80% 알람(A11)·라이브 가격 억제, 오토스케일 SI §1 상속 |
| RESILIENCY-01·02·08·11·12 | 상속 | Multi-AZ·백업·워크로드 분류(M3·M5=Medium)는 SI·U1 — U3 신규 워크로드 없음 |
| RESILIENCY-05·07 | 준수(증분) | §5 — 외부 실패율·서킷·커버리지 커스텀 메트릭·대시보드 편입 |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속. 화이트리스트·폴백 순서 변경은 OPS-01(§7) |
| SECURITY-05 | 준수 | 외부 응답 스키마 검증·URL 파싱 SSRF 가드는 nfr-design PAT-U3-04(앱) — 인프라는 아웃바운드 미발생(fetch 없음) 정합 |
| SECURITY-06·07 | 준수 | §1 — 최소 권한(어댑터별 시크릿 ARN)·NAT 경유·SG deny-by-default(SI §2.2·§7.1) |
| SECURITY-10·12 | 준수(상속) | §1 — 외부 키 Secrets Manager·평문 0·시크릿 스캔 게이트(U1 CI 상속) |
| SECURITY-01·02·03·14 | 상속 | 암호화·중개 로깅·앱 로깅·알림은 SI·U1 정본 — 외부 어댑터 로그도 U1 마스킹·상관 ID |
| SECURITY-08·09·11·13·15 | 상속/N/A | 소유권(앱 §5 nfr)·하드닝·LLM(U5 무관)·무결성·예외 처리 U1 스캐폴드 상속 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — POI 매칭·TTL·URL 파서·복귀 결정성은 Code Generation 실행 |

**차단 소견 없음.** U3 신규 AWS 리소스 0(§6) — SI가 예약한 NAT·시크릿·알람·스케줄러 프레임 내에서 인프라가 종결된다. **비개발 선결 P2(지도 약관)·P4(TourAPI 캐싱 조건)** 는 POI 정본·스냅샷의 **확정 스키마**를 차단하는 Functional Design 대상이며(§0·nfr-requirements.md §7.1), 본 인프라 산출물(저장 위치·격리·정책 컬럼)은 소급 대응 가능하게 준비되어 비차단이다.
