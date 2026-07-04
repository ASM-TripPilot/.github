# U6 여행 중 실행·Plan-B — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U6 (M18 Trip Execution · **M9 Plan-B Detection** · **M10 Itinerary Recalculation** · **M11 Weather & Context** + `features/execution`·`features/planb`·`shared/location`)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U6는 SI를 참조만 하고 재정의하지 않는다.** 본 문서는 U6가 SI에 추가하는 **인프라 증분**(기상청 아웃바운드·서버 폴링 잡·GPS/날씨 스키마·위치 법정 로그 기록 개시)만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md). 재계획 연산 인프라: **U5 [infrastructure-design.md](../../u5-itinerary/infrastructure-design/infrastructure-design.md) 재사용(신규 없음)**.

## 0. 요약 — U6 인프라 증분: 신규 AWS 리소스 0 (기상청 아웃바운드·시크릿·서버 폴링 잡·법정 로그 개시)

U6는 실시간·위치 유닛이나, **신규 AWS 리소스는 0**이다 — SI가 이미 U6를 명시적 소비자로 예약해 두었기 때문이다: **NAT 아웃바운드(§2.1) + 스케줄 잡 프레임(§1.2 ShedLock) + 위치 법정 로그 기록 개시(§5.3) + 기상청 시크릿(§6 프레임) + 외부 쿼터·실패율 알람(§8.2 A11·A12) + 아웃박스(§4)**(SI §0 유닛 표: "U6 = NAT 아웃바운드(기상청) + 스케줄 잡 프레임 + 위치 법정 로그 기록 개시(§5.3)"). U6 인프라 결정은 넷 — **① 기상청 벤더 아웃바운드·시크릿, ② 서버 배치 폴링 잡(기존 스케줄러), ③ GPS 발자취·날씨 캐시 저장(RDS 스키마), ④ 위치 법정 로그 기록 개시(§5.3)** 이며, 전부 기존 리소스 내 확장/설정이다. **재계획 연산(C1·C2)은 U5 인프라 재사용**(신규 없음).

**선결 경계**: 기상청 **API 활용 신청·쿼터·서비스 키**는 **P3(기상청 공공데이터포털 API 활용 신청)** 확정 대상이고, **GPS 발자취·위치 기반 트리거의 실서비스 활성**은 **P1(위치기반서비스사업 신고 — 법정 출시 전제)** 완료 전 **비활성 플래그** 대상이다(nfr-requirements.md §8.1). 본 문서는 아웃바운드 경로·시크릿 항목·폴링 잡·저장 스키마·법정 로그까지 벤더 중립으로 확정한다(개발 비차단, 출시는 P1·P3 게이트).

---

## 1. 기상청 벤더 아웃바운드 · 시크릿 보관 (SI §2·§6 상속)

- **아웃바운드 경로**: ECS(프라이빗 앱 서브넷) → **NAT Gateway**(SI §2.1, 기존 1개) → 기상청 공공데이터포털. `sg-app` 아웃바운드 443은 SI §2.2가 이미 허용("소셜 IdP·**기상청**·카카오·TMap·LLM·FCM 등 목적지 IP 가변 SaaS 예외 문서화") — **SG 변경 0**. U6 아웃바운드 목적지 문서화(egress 허용 목록):

| 목적지 | 어댑터(포트 · 소유) | 용도 |
|---|---|---|
| **기상청 공공데이터포털**(단기예보·기상특보, D10) | `WeatherPort` · M11 | Plan-B 트리거 (a) 계열(강수확률·특보) — **P3 신청** |
| (재사용) TourAPI·카카오 지도 | `TourContentPort` 등 · M7(U3) | J2 휴무·영업시간 재조회(정기 변경만 G192) — **U3 어댑터 재사용**(신규 아웃바운드 아님) |
| 외부 지도 앱 딥링크 | 딥링크 빌더(외부 호출 없음) | URL 조립만 — 아웃바운드 미발생(US-E6-03·G66) |

- **시크릿(SI §6 프레임)**: 기상청 API 키를 **Secrets Manager**에 신규 시크릿 항목으로 등록(신규 서비스 아님 — SI §6 프레임 내 항목 추가): **`external/kma`**(기상청 서비스 키). ECS 태스크 정의 `secrets` ARN 참조 주입 — 코드·이미지·Terraform state 평문 0(NFR-U6-SEC-05·NFR-U1-SEC-11). 로테이션은 공공데이터포털 키 갱신 주기 준수(P3 후 확정).
- **어댑터 타임아웃·서킷·격자 변환**(NFR-U6-RES-08)은 앱 소관(Resilience4j·LCC DFS — U6-W1) — 인프라는 SI §8.2 **A12(어댑터 실패율·서킷 open)·A11(기상청 쿼터 80%)** 알람을 제공(§5). TMap 도로 거리(재계획 이동시간)는 **C2 소유(U5) — U6 미호출**, 실패 시 직선거리×우회계수(G106) 폴백(U5 재사용).

---

## 2. 서버 배치 폴링 잡 — 기존 ShedLock 스케줄러 재사용 (신규 리소스 0)

U6 폴링 잡은 SI §1.2 확정 모델(앱 프로세스 내 Spring `@Scheduled` + **PostgreSQL ShedLock 분산 락**)을 재사용한다 — **별도 워커 서비스·EventBridge 없음**(진행 중 여행 소량·저빈도, G142·NFR-U6-SC-01).

| 잡 | 소유 | 주기 (원격 구성 G195) | 내용 | 근거 |
|---|---|---|---|---|
| J1 날씨·특보 폴링 | M9 (M11 경유) | **1시간 · 원격 구성** | 진행 중 여행 당일~익일 방문 예정지 좌표(격자 변환·중복 제거) → 날씨 캐시 갱신 + 트리거 평가 → `TriggerFired` | services.md §4 J1, D10 |
| J2 휴무·영업시간 재조회 | M9 (M7 경유) | **당일 아침 1회(기본 07:00) · 원격 구성** | 당일 방문 예정 POI → TourAPI·지도 재조회(정기 변경만 G192) → 변경 감지 시 `TriggerFired` | services.md §4 J2, G192 |
| J3 여행 자동 종료·일자 경계 | M18 | **일 1회 자정(KST)** | 종료일 경과 여행 → `TripEnded` / 일자 경계 → `DayClosed`(당일 회고 트리거 U7) | services.md §4 J3, D19 |

- **중복 실행 방지**: ShedLock 분산 락으로 태스크 2개 중복 실행 배제(SI §1.2). **폴링 커넥션 풀은 실행 허브 API 처리 풀과 분리**(U1 PAT-PERF-02 벌크헤드 상속 — 배치가 여행 중 실시간 요청 지연 방지, NFR-U6-SC-04).
- **원격 구성(G195)**: 폴링 주기·트리거 상한(시간당 2회/하루 8회)·민감도·폴백 순서는 **전부 remote config**(NFR-U6-PRIV-04) — 코드 재배포 없이 조정(OPS-01 변경 관리).
- **기상청 무응답 처리(J1)**: 즉시 1회 재시도 후 **트리거 침묵**(허위 알림 금지) + 실패율 계측(A12) — 잡 자체 실패는 다음 주기 자연 복구(services.md §4 J1, PAT-U6-01).
- **멱등성**: J1·J2는 상태 기반 조회 + 트리거 상한(G58)이 중복 발화 흡수(동일 여행·방문지·사유 1회 노출), J3은 상태 기반 + 조건부 전이 = 자연 멱등(services.md §4).
- **전환 트리거**: 폴링 대상·기상청 호출량이 태스크·쿼터를 상시 압박하면 EventBridge Scheduler + ECS RunTask 분리 재평가(SI §1.2 트리거 상속). **그 전까지 별도 워커 미도입**(과설계 금지).

---

## 3. GPS 발자취·날씨 캐시 저장 — RDS 스키마 확장 + 위치 법정 로그 개시 (Flyway)

- **저장소**: SI §3의 **기존 RDS PostgreSQL**(단일 `trippilot` DB·`public` 스키마) 재사용 — **신규 DB 인스턴스·클러스터 없음**. U1 Flyway 파이프라인으로 U6 마이그레이션 적용.
- **U6 신규 테이블(~8~9개, unit-of-work.md U6 산출물)**: 실행 상태(방문 전이·체류), 여행 실행 상태(활성·휴식·종료), 트리거 이력·억제 상태, 민감도 설정, 재계획 세션, 대안 후보, 미배치 목록(C10), 날씨 캐시(격자 nx·ny), **GPS 폴리라인**(단순화만·원시 파기 G73).
- **위치정보 법정 로그 — U1 append-only 테이블 재사용, U6 기록 개시(N2·SI §5.3)**:
  - **테이블 자체는 U1 생성**(위치정보 수집·이용 사실확인자료 append-only 테이블, `app_user` UPDATE/DELETE **REVOKE** + IAM `s3:DeleteObject`·`logs:Delete*` **Deny** 2중 강제, SI §3·§5.3·§7.1). **U6이 이 테이블에 처음 기록을 개시**(SI §0 U6: "위치 법정 로그 기록 개시").
  - **GPS 이벤트와 동일 트랜잭션 기록**(유실 0, SI §5.3 — 정합성 A안 근거). 앱 역할은 자기 로그 삭제 권한 없음.
  - **월 1회 스냅샷 잡 개시**(SI §5.3): 전월분 법정 로그를 `trippilot-{env}-log-archive` 버킷으로 내보내기(SSE-KMS·별도 프리픽스) — **U6 기능 활성 시점부터 가동**(SI §5.3 명시).
  - **계정 삭제 시**: GPS 폴리라인·위치 파생 데이터는 즉시 파기(D34, services.md §S6 2단계 구독), **법정 로그는 분리 보관 유지**(6개월+, N2).
- **GPS 옵트인 게이트**: GPS 폴리라인 저장·법정 로그 기록은 **옵트인(3층 동의 3층) 전제**(D34·NFR-U6-PRIV-05) + **P1 신고 완료 전 비활성 플래그**(NFR-U6-SEC-02).
- **볼륨**: DAU 1천·활성 여행 ≤1/사용자(D21)·저빈도(1~5분) 수집 → GPS 폴리라인·법정 로그 일 수만 행 수준(NFR-U6-SC-03·SI §5.3) — SI §3 gp3 20GB(오토스케일 100GB 상한) 내 충분. 날씨 캐시는 격자 단위 소량(인메모리 캐시 병행 — U6-W1). 트리거 이력·실행 상태는 여행별 인덱스.
- **소유권·권한**: U6 일반 테이블(실행 상태·트리거·폴리라인)은 `app_user` 표준 DML(accountId 스코핑, NFR-U6-SEC-04). **위치 법정 로그만 append-only REVOKE 대상**(§5.3 — U1 롤 모델 재사용).

---

## 4. 트리거·푸시 경계 — M14(U8) 발행/구독 분리 (U6는 발행만)

U6의 트리거 처리는 **아웃박스 발행까지**이며, 실제 FCM 푸시 발송은 **M14(U8) 소관**이다 — 경계를 명시한다.

- **U6 소유(발행)**: M9가 트리거 상한·억제 필터(G58) 통과분을 **TX 트리거 기록 + outbox `TriggerFired`**(services.md §S2.1 4단계). 재계획 확정은 **current 갱신 + changelog + outbox `ItineraryChanged`**(PAT-U5-04 아웃박스 재사용, SI §4). 여행 종료·일자 경계는 `TripEnded`·`DayClosed` 발행(J3).
- **M14(U8) 소유(구독·푸시)**: `TriggerFired` 구독 → **심각(severe)만 FCM 푸시, 경미(minor)는 인앱 칩**(services.md §S5). FCM 아웃바운드·SNS/알림 스케줄링은 **U8 인프라**(SI §0 U8: "FCM 아웃바운드"). **U6은 FCM 인프라를 만들지 않는다.**
- **아웃박스 인프라**: 전역 아웃박스(PostgreSQL 테이블 + J10 릴레이, SI §4)는 U1 산출 — U6은 신규 아웃박스 리소스 없이 **이벤트 타입만 추가**(at-least-once·구독자 멱등, services.md §0.2). 아웃박스 적체는 A10 알람(SI §8.2)으로 관측.

---

## 5. 외부 알람 · 관측 (SI §8.2 구체화)

- **외부 API 알람(SI §8.2 — U6가 기상청의 첫 소비자)**:

| 알람(SI 프레임) | U6 소스 | 임계 | 단계 |
|---|---|---|---|
| A11 외부 쿼터 80% | **기상청** 호출량 vs 문서화 쿼터(P3) | 80% | P2 |
| A12 어댑터 실패율·서킷 open | **기상청** 어댑터 | 실패율 >20%(5분) 또는 서킷 open | P2 |
| A10 잡·큐 적체 | 폴링 잡·아웃박스(TriggerFired) 지연 | > 15분 | P2 |

- 알람 라우팅은 SNS `trippilot-{env}-alerts` → OPS-02 프로세스(SI §8.2). 기상청 쿼터 한도는 **P3 API 활용 신청 후 문서화**(NFR-U6-SC-02). 커스텀 메트릭은 SI §7.1 태스크 역할 `cloudwatch:PutMetricData` 허용 내.
- **트리거 침묵 관측(§6.7 · NFR-U6-RES-01)**: 기상청 무응답으로 트리거 (a) 계열이 침묵 전환되면 **자동 알림은 조용하되 실패율·침묵 전환은 계측**(침묵 실패 금지 관측) — A12·커스텀 메트릭으로 대시보드 노출. GPS·위치 처리 로그는 **PII·정밀 좌표 마스킹**(§0 PAT-OBS-01 상속, NFR-U6-SEC-03), 법정 로그와 운영 로그 분리.

---

## 6. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 폴링·재계획·실행 API는 기존 태스크 2개 흡수(진행 중 여행 소량·저빈도) | §2, G142·NFR-U6-SC-01 |
| 네트워크 VPC·NAT·ALB (SI §2) | **0** | 0 — **기상청 NAT 아웃바운드**·`sg-app` 443·`/api/*` 재사용. 아웃바운드 허용 목록 문서화만(기상청 추가) | §1, SI §2.1·2.2 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 ~8~9개**(실행 상태·여행 실행 상태·트리거 이력/억제·민감도·재계획 세션·대안 후보·미배치·날씨 캐시(격자)·GPS 폴리라인) — 기존 RDS·Flyway 확장 | §3, unit-of-work.md U6 |
| 위치 법정 로그 (SI §5.3) | **0** | **U1 append-only 테이블에 기록 개시** + **월 1회 스냅샷 잡 가동 개시**(log-archive) — 테이블·버킷은 기존 | §3, SI §5.3 |
| 캐시 (신규 계층) | **0** | **인메모리 Caffeine**(격자 날씨 캐시 — U3 재사용) — ElastiCache 미도입 | §3, U6-W1 |
| 시크릿·KMS (SI §6) | **0** | **Secrets Manager 항목 1개 추가**(`external/kma`) — SI §6 프레임 내 | §1, SI §6 |
| 관측성 CloudWatch (SI §8) | **0** | 기상청 A11·A12·A10 소스 편입·트리거 침묵 커스텀 메트릭·대시보드 위젯 — 기존 프레임 | §5, SI §8.2 |
| 메시징·비동기 (SI §4) | **0** | 0 — 서버 폴링은 기존 ShedLock, TriggerFired/ItineraryChanged는 기존 아웃박스(이벤트 타입만 추가) | §2·§4, SI §1.2·§4 |
| 스토리지 S3·CloudFront (SI §5) | **0** | 0 — U6는 사진 없음(사진 U7). log-archive 버킷은 §5.3 스냅샷 대상(기존) | SI §5 |

**총계: 신규 AWS 리소스 0 / 변경 = RDS 스키마 확장(테이블 ~8~9개) + Secrets Manager 항목 1개(`external/kma`) + 위치 법정 로그 기록·월 스냅샷 잡 개시 + 기상청 알람/메트릭 활성화 + 폴링 잡 3종(기존 스케줄러).** U6는 실시간·위치 유닛이지만 인프라는 SI가 예약한 프레임(NAT·스케줄러·시크릿·법정 로그·아웃박스) 내에서 종결된다 — **재계획 연산은 U5 재사용, 신규 외부는 기상청 아웃바운드 1종**이라 신규 인프라 표면이 없다(과설계 금지·진행 중 여행 소량 규모 정합).

---

## 7. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U6는 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 재사용한다.

- **서버(M18·M9·M10·M11 모듈 + 실행/트리거/재계획/GPS 마이그레이션)**: `server/**` 경로이므로 `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). 실행 상태·트리거·재계획·날씨 캐시·GPS 폴리라인 테이블 추가는 Flyway forward-only(신규 테이블 = N-1 호환 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속. **기상청 시크릿(`external/kma`, §1)은 배포 전 Secrets Manager 등록**. 폴링 잡은 기존 ShedLock 스케줄러 흡수(신규 워크플로 없음).
- **모바일(`features/execution`·`features/planb`·`shared/location`)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **화면(활성 허브·도착 프롬프트·재계획 비교·계획 vs 실제 경로 지도)은 대부분 순수 JS/TS로 OTA 반영 가능**. 단 **`shared/location`(expo-location)은 위치 권한 문자열을 config plugin으로 주입하는 네이티브 모듈**이다 — U1('내 주변' 프리프롬프트)·U3(카카오 지도 SDK) 시점에 expo-location·권한 문자열이 이미 도입됐으면 순수 JS OTA, **여행 중 포그라운드 위치 권한 문자열을 신규 추가하면 EAS Build 재빌드 델타**(OTA 불가 — U3 지도 SDK와 동일 성격). config plugin·권한 상태를 U6 초기 스파이크로 확인(tech-stack-decisions.md U6-C1 네이티브 델타 주석). 지도(계획 vs 실제 경로)는 U3 `shared/map` 상속.
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. **폴링 주기·트리거 상한·민감도·폴백 순서·기상청 키는 remote config·시크릿으로 환경별 독립**(SI §9, G195) — P3 확정 후 값 주입(코드 재배포 없이 조정). **위치 기능은 P1(위치정보사업 신고) 완료 전 비활성 플래그**(remote config) 운영 — 신고 완료 시 플래그 활성(NFR-U6-SEC-02).

**U6 배포 델타**: 서버는 기존 워크플로 흡수(0 델타). 모바일은 **expo-location 위치 권한 config plugin으로 인한 네이티브 재빌드 가능성**(U3 지도 SDK와 동일 — 기존 권한 재사용이면 OTA)이 유일한 델타. 유일한 배포 전 조치는 **기상청 시크릿 등록(P3 후)** + **위치 기능 비활성 플래그(P1 게이트)** 이며, 인프라 파이프라인 변경은 없다.

---

## 8. 컴플라이언스 요약 (U6 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-10 | **준수** | §1·§5 — 기상청 어댑터 타임아웃·서킷은 앱(Resilience4j), 인프라는 A12·A11 알람·트리거 침묵 관측. 폴백은 nfr-design PAT-U6-01·05 |
| RESILIENCY-09 | 준수 | §5 — 기상청 쿼터 80%(A11)·폴링 상한 원격 구성(G195)·오토스케일 SI §1 상속 |
| RESILIENCY-05·07 | 준수(증분) | §5 — 기상청 실패율·서킷·트리거 침묵·잡/아웃박스 적체(A10·A12) 메트릭·대시보드 편입 |
| RESILIENCY-01·02·08·11·12 | 상속 | Multi-AZ·백업·워크로드 분류(M9·M10 High·실행 Critical·기상청 Medium)는 SI·U1 — 폴링은 기존 스케줄러 흡수 |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속. 폴링·트리거·폴백 순서 변경은 OPS-01(§7) |
| SECURITY-14+N2 위치 로그 | **준수(핵심)** | §3 — 위치 법정 로그 U1 append-only 재사용·GPS 동일 TX 기록·앱 삭제 권한 없음(REVOKE+IAM Deny)·월 스냅샷·계정 삭제 시 분리 보관 |
| SECURITY-06·07 | 준수 | §1 — 최소 권한(어댑터 시크릿 ARN)·NAT 경유·SG deny-by-default(SI §2.2·§7.1) |
| SECURITY-11 | 상속(U5) | 재계획 LLM 경계는 U5 인프라·nfr-design 재사용(서버 재조회·필드 최소화) |
| SECURITY-10·12 | 준수(상속) | §1·§7 — 기상청 키 Secrets Manager·평문 0·시크릿 스캔(U1 CI 상속) |
| SECURITY-05 | 준수 | 수동 수정 입력 검증·숙소 고정 제약 차단은 nfr-design PAT-U6-03(앱) — 인프라는 아웃바운드 미발생(딥링크) 정합 |
| SECURITY-01·02·03·04 | 상속 | 암호화·중개 로깅·앱 로깅(GPS·PII 마스킹)·헤더는 SI·U1 정본 — 위치 로그도 U1 마스킹·상관 ID·법정/운영 분리 |
| SECURITY-08·09·13·15 | 상속 | 소유권(앱 nfr §5 accountId)·하드닝·무결성·예외 처리 U1 스캐폴드 상속 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — 트리거 순수 함수·상태 머신·warm-start·폴리라인 위상은 Code Generation 실행(U6 DoD) |

**차단 소견 없음.** U6 신규 AWS 리소스 0(§6) — SI가 예약한 NAT·스케줄러·시크릿·법정 로그·아웃박스 프레임 내에서 인프라가 종결된다. 재계획 연산은 U5 재사용, 신규 외부는 기상청 아웃바운드 1종이다. **비개발 선결 P1(위치정보사업 신고 — 법정 출시 전제)·P3(기상청 API 활용 신청)** 는 위치 기능 활성·기상청 키/쿼터를 확정하는 대상이며(§0·nfr-requirements.md §8.1), 본 인프라 산출물(아웃바운드 경로·시크릿 항목·폴링 잡·저장 스키마·법정 로그 개시)은 벤더 중립·비활성 플래그로 준비되어 **개발 비차단**(출시는 P1·P3 게이트)이다.
