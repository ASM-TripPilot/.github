# U5 AI 일정 생성·확정 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U5 (M8 Itinerary Generation · **C1 LLM Gateway** · **C2 Solver Engine** + `features/itinerary`·`shared/validation`)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U5는 SI를 참조만 하고 재정의하지 않는다.** 본 문서는 U5가 SI에 추가하는 **인프라 증분**(LLM 벤더 아웃바운드·솔버 CPU 사이징·점진 응답 채널·백그라운드 잡·LLM 비용 계측)만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md).

## 0. 요약 — U5 인프라 증분: 신규 AWS 리소스 사실상 0 (LLM 아웃바운드·시크릿·솔버 CPU 사이징)

U5는 연산 최중량 유닛이나, **신규 AWS 리소스는 사실상 0**이다 — SI가 이미 U5를 명시적 소비자로 예약해 두었기 때문이다: **NAT 아웃바운드(§2.1) + LLM·TMap 시크릿(§6 "LLM API 키(U5) — 예약") + LLM 비용 계측 메트릭(§8.2 대시보드 "LLM 비용 U5 확장 슬롯") + A11 쿼터·A15 비용 예산 알람 + 워커 스레드 실행 모델(§1.2)**(SI §0 유닛 표: "U5 = NAT 아웃바운드(LLM·TMap), LLM 비용 계측 메트릭(§8)"). U5 인프라 결정은 넷 — **① LLM 벤더 아웃바운드·시크릿, ② 솔버 CPU 연산 배치(기존 태스크 vs 별도 워커), ③ 점진 응답 채널(SSE)의 ALB 영향, ④ 백그라운드 생성 잡** 이며, 전부 기존 리소스 내 확장/설정이다.

**선결 경계**: LLM **벤더·모델·단가·후보 풀 크기 최종값·국외 이전 여부**는 **P6(LLM 벤더 계약·비용 산정)** 확정 대상이다(nfr-requirements.md §7.1). 본 문서는 아웃바운드 경로·시크릿 항목·비용 계측·연산 배치까지 벤더 중립으로 확정한다(비차단).

---

## 1. LLM 벤더 아웃바운드 · 시크릿 보관 (SI §2·§6 상속)

- **아웃바운드 경로**: ECS(프라이빗 앱 서브넷) → **NAT Gateway**(SI §2.1, 기존 1개) → LLM 벤더 API·TMap. `sg-app` 아웃바운드 443은 SI §2.2가 이미 허용("목적지 IP 가변 SaaS 예외 문서화") — **SG 변경 0**. U5 아웃바운드 목적지 문서화(egress 허용 목록):

| 목적지 | 어댑터(포트 · 소유) | 용도 |
|---|---|---|
| **LLM 벤더 API**(단일 관리형, D11) | `LlmPort` · C1 | 취향 점수(경량)·배치 설명·회고(상위 티어) — **P6 벤더 확정** |
| TMap(도로 거리) | `RoadDistancePort` · C2 | 이동시간 추정 입력(D08·G106) — 실패 시 직선거리 폴백 |

- **시크릿(SI §6 프레임)**: 외부 API 키를 **Secrets Manager**에 등록(신규 서비스 아님 — SI §6 예약 항목 활성화): **`external/llm`**(LLM 벤더 API 키 — SI §6 "LLM API 키(U5) — 예약" 활성화), **`external/tmap`**(TMap 키 — U3에서 "U5에서 등록"으로 예약). ECS 태스크 정의 `secrets` ARN 참조 주입 — 코드·이미지·Terraform state 평문 0(NFR-U5-SEC-05·NFR-U1-SEC-11). 로테이션은 벤더 콘솔 주기 준수(SI §6, P6 후 확정).
- **어댑터 타임아웃·서킷·rate-limit**(NFR-U5-RES-06·COST-03)은 앱 소관(Resilience4j — U5-S4) — 인프라는 SI §8.2 **A12(어댑터 실패율·서킷 open)·A11(쿼터 80%)** 알람과 **LLM 비용 계측**(§5)을 제공(U5가 LLM 비용 첫 소비자).
- **국외 이전(G181)**: 벤더가 국외 리전인 경우 아웃바운드가 국외로 향한다 — **개인정보 국외 이전·처리위탁 고지**(P7 개인정보처리방침) 대상. 전송 필드는 closed-set 후보 ID·취향 축으로 목적 최소화(D31·PAT-U5-06 — 인프라 밖 앱 강제). 벤더 국내/국외 판별은 **P6 확정**.

---

## 2. 솔버 연산 배치 — 기존 ECS 태스크 내 CPU 집약 워커 (별도 워커 미도입)

C2 Solver Engine은 **외부 무의존 인프로세스 결정론 연산**(component-dependency §2.3)이다 — 아웃바운드·DB 외 인프라 표면이 없다. 유일한 인프라 질문은 **CPU 집약 연산 10개 동시(G142)를 어디서 돌리는가**다.

| 기준 (G142 · 동시 생성 10) | **A. 기존 ECS 태스크 내 워커 스레드 (권고)** | B. 별도 솔버 워커 서비스 + 큐 |
|---|---|---|
| 신규 AWS 리소스 | **0** — 앱 프로세스 내 CPU 벌크헤드 스레드 풀(U5-S2) | +ECS 서비스·큐(SQS)·SG — SI §4 "브로커 미도입" 위반 |
| 진행 스트림(SSE) 연계 | **직접**(동일 프로세스) | 크로스 프로세스 팬아웃 필요 |
| 규모 정합(동시 10·저 DAU) | **적합** — solve당 0.5~0.8초·문제 크기 소, 1 vCPU/2GB(SI §1.2)로 흡수 판단 | 과설계(G142) |
| 격리(무거운 solve ↔ 편집·CRUD) | CPU 스레드 풀 분리(NFR-U5-SC-03 벌크헤드) | 프로세스 분리(과잉) |

- **확정 권고: A(기존 ECS 태스크 내 CPU 집약 워커 스레드) · 별도 솔버 워커 미도입.** 근거: (1) 솔버는 인프로세스 결정론 코드라 **아웃바운드·상태 공유가 없어** 별도 서비스 이점이 없고, (2) 동시 10·solve당 0.5~0.8초는 **기존 태스크 2개(1 vCPU/2GB, SI §1.2)로 흡수 가능한 규모**(G142), (3) 별도 워커+큐는 SI §4(브로커 없음·DB 기반) 결정과 배치되고 SSE 진행 연계가 끊긴다.
- **태스크 CPU 사이징 재검토(SI §1.2 대비)**: SI §1.2 태스크 사이즈(1 vCPU/2GB)는 **JVM 상주 + argon2id 검증**(U1) 기준으로 산정됐다. U5의 **CPU 집약 solve 동시 10**은 새 부하 프로파일이므로, **Functional Design/Code Generation에서 부하 실측 후 판단**한다: 동시 10 solve가 1 vCPU를 상시 포화시키면 (a) **태스크 vCPU 상향(1→2)** 또는 (b) 오토스케일 CPU 임계 하향(SI §1.2 Target Tracking 60%)으로 대응. **솔버 CPU 벌크헤드**(NFR-U5-SC-03): 무거운 solve 스레드 풀 ↔ 편집 재검증·CRUD I/O 풀 분리로 경량 요청 지연 방지(U1 PAT-PERF-02 상속). **동시 생성 세마포어**(NFR-U5-COST-03 rate-limit와 통합)로 동시 10 상한 보호.
- **전환 트리거**: 동시 생성 실측이 태스크 CPU를 상시 포화시키거나 solve가 API p95를 열화(A2·A4 상관 관측)시키면 솔버 워커 분리(EventBridge + ECS RunTask, SI §1.2 트리거 상속) 재평가. **그 전까지 별도 워커 미도입**(과설계 금지).
- **C2 결정론 폴백은 순수 연산**: 규칙 점수 + 결정론 솔버 단독 모드(NFR-U5-RES-04)는 **외부 무의존**(LLM·TMap 없이 배치) — 인프라 표면 0, LLM/외부 장애와 독립 동작.

---

## 3. 점진 응답 채널 — SSE (ALB 영향 최소)

첫 1일 5초 게이트 후 잔여 일자를 백그라운드로 채우며 진행을 스트리밍한다(PAT-U5-01, US-E5-09). 채널은 **SSE**(tech-stack-decisions.md U5-S1).

- **ALB 영향(SI §2.3)**: SSE는 **롱리브드 HTTP 응답**이다 — ALB 유휴 타임아웃 **60초**(SI §2.3) > 전체 생성 예산 **20초**이므로, 단일 SSE 스트림이 유휴 없이 진행 이벤트를 흘려 타임아웃 내 종료된다. **스티키니스 불요**(WebSocket 인프라는 U11 이연 — SI §0 U9~U11) — SSE는 단일 요청 수명이라 다중 인스턴스 세션 고정이 필요 없다. **ALB 설정 변경 0** — 기존 `/api/*` 대상 그룹 흡수.
- **폴백**: SSE 미지원 프록시·클라이언트는 **GenerationSession 상태 폴링**(services.md §1.3 — 세션 상태가 진행 정본)으로 대체(NFR-U5-PERF-04). 폴링도 기존 `/api/*` 경로 재사용(신규 리소스 0).
- **HTTP/keep-alive**: SSE는 HTTP/1.1 keep-alive로 동작 — Fargate 태스크·ALB 기본 설정 내(별도 프로토콜 설정 불요).

---

## 4. 백그라운드 생성 잡 — 요청 단위 비동기 (스케줄러/큐 신규 없음)

잔여 일자 생성은 **요청 단위 비동기 워커**(unit-of-work.md U5 스케줄러 잡: 없음, tech-stack-decisions.md U5-S2)다.

- **실행 모델**: 생성 요청 수명 내(≤20초) 앱 프로세스 내부 **JDK 21 가상 스레드 / `@Async` Executor**로 잔여 일자 solve 실행. **ShedLock 스케줄 잡·EventBridge·SQS 신규 없음**(SI §1.2 스케줄러 프레임은 주기 잡용 — 요청 구동 생성엔 미사용, SI §4 브로커 없음 정합).
- **격리**: 백그라운드 solve 워커 풀은 요청 처리 풀·편집 재검증 풀과 분리(NFR-U5-SC-03 벌크헤드) — 백그라운드가 신규 요청 수락을 막지 않게.
- **크래시 내성**: 백그라운드 미완료 시 GenerationSession 상태가 `DAY1_READY`/부분 상태로 남아 '생성 중' 식별(services.md §1.3) — 재진입 시 '이어서 생성'(G161). `ItineraryGenerated`는 전체 완료 TX만 발행(부분 미발행, PAT-U5-04) — 크래시가 반쪽 이벤트를 만들지 않는다.

---

## 5. LLM 비용 계측 · 외부 알람 (SI §8.2 구체화)

- **LLM 비용 메트릭(NFR-U5-COST-05)**: C1이 호출별 티어·토큰·추정 비용을 CloudWatch **커스텀 메트릭**(`cloudwatch:PutMetricData` — SI §7.1 태스크 역할 허용)으로 발행 → **SI §8.2 대시보드 `trippilot-{env}-ops`의 "LLM 비용 계측(U5 확장 슬롯)" 위젯 활성화**. U5가 이 슬롯의 첫 소비자다.
- **외부 API 알람(SI §8.2 — U5가 LLM의 첫 소비자)**:

| 알람(SI 프레임) | U5 소스 | 임계 | 단계 |
|---|---|---|---|
| A11 외부 쿼터 80% | LLM 벤더·TMap 호출량 vs 문서화 쿼터 | 80% | P2 |
| A12 어댑터 실패율·서킷 open | LLM·TMap 어댑터 | 실패율 >20%(5분) 또는 서킷 open | P2 |
| **A15 월 비용 예산**(AWS Budgets) | **LLM 호출 비용 포함 월 예산** | 예산 80%·100% | P2 |

- 알람 라우팅은 SNS `trippilot-{env}-alerts` → OPS-02 프로세스(SI §8.2). LLM 쿼터·단가는 **P6 벤더 계약 확정 후 문서화**(NFR-U5-COST-05). A15 비용 예산은 LLM 운영비를 반영해 SI §10 비용 개요에 추가 예산 라인으로 편입(P6 산정 후).
- **관측(§6.7)**: LLM 오류·솔버 검증 실패·타임아웃은 전부 계측(침묵 실패 금지) — §0 PAT-OBS-01·03 상속, 어댑터 로그는 D31 경계로 프롬프트·응답 PII·키 미출현.

---

## 6. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | **태스크 CPU 사이징 재검토**(동시 10 solve 부하 실측 후 vCPU 상향/오토스케일 임계 조정 판단) — 별도 솔버 워커 미도입 | §2, G142 |
| 네트워크 VPC·NAT·ALB (SI §2) | **0** | 0 — NAT 아웃바운드(LLM·TMap)·`sg-app` 443·`/api/*` 재사용. **SSE ALB 설정 변경 0**(유휴 60초>20초·스티키니스 불요). 아웃바운드 허용 목록 문서화만 | §1·§3, SI §2.1·2.3 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 ~6개**(일정 plan/current·일자·슬롯·생성 세션/초안·추천 이유 메타·LLM 호출 로그) — 기존 RDS·Flyway 확장 | nfr-requirements §4.2, unit-of-work.md U5 |
| 캐시 (신규 계층) | **0** | 0 — U5는 M7 후보 풀(closed-set) 소비자, LLM 응답 캐싱 없음(개인화·closed-set별 상이) | nfr-requirements §0.1 |
| 시크릿·KMS (SI §6) | **0** | **Secrets Manager 항목 2개 활성화**(`external/llm`·`external/tmap`) — SI §6 예약분 | §1, SI §6 |
| 관측성 CloudWatch (SI §8) | **0** | **LLM 비용 위젯 활성화**(U5 확장 슬롯)·A11·A12·A15 LLM 소스 편입 | §5, SI §8.2 |
| 메시징·비동기 (SI §4) | **0** | 0 — 백그라운드 생성은 요청 단위 비동기 워커(큐·브로커 없음, SI §4 정합) | §4 |
| 스토리지 S3·CloudFront (SI §5) | **0** | 0 — U5는 사진·정적 자산 없음 | SI §5 |

**총계: 신규 AWS 리소스 0 / 변경 = RDS 스키마 확장(테이블 ~6개) + Secrets Manager 항목 2개 활성화 + LLM 비용·쿼터·예산 알람/위젯 활성화 + 태스크 CPU 사이징 재검토(부하 실측 후).** U5는 연산이 무겁지만 인프라는 SI가 예약한 프레임(NAT·시크릿·비용 계측·워커 모델) 내에서 종결된다 — **솔버는 인프로세스, LLM은 아웃바운드 1종**이라 신규 인프라 표면이 없다(과설계 금지·동시 10 규모 정합).

---

## 7. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U5는 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 재사용한다.

- **서버(M8·C1·C2 모듈 + 일정 마이그레이션)**: `server/**` 경로이므로 `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). 일정·세션·슬롯·LLM 로그 테이블 추가는 Flyway forward-only(신규 테이블 = N-1 호환 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속. **LLM·TMap 시크릿(§1)은 배포 전 Secrets Manager 등록**. 솔버 라이브러리·commons-math는 Gradle version catalog + 잠금 파일(SECURITY-10 상속).
- **모바일(`features/itinerary`·`shared/validation`)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **순수 JS/TS 화면**(생성 진행·시간표/지도 2보기·편집·확정)이라 **네이티브 델타 없음** — expo-updates OTA 반영 가능(U3 카카오 지도 SDK 같은 네이티브 재빌드 불요). 지도 표시는 U3 `shared/map` 상속. 클라 경량 검증기(`shared/validation`)는 서버 규칙 단일 원천 파생(D28).
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. **LLM 티어→모델 매핑·후보 프루닝 상한·타임아웃·폴백 순서·rate-limit·벤더 단가는 remote config·시크릿으로 환경별 독립**(SI §9) — P6 확정 후 값 주입(코드 재배포 없이 조정, OPS-01 변경 관리).

**U5 배포 델타**: 서버·모바일 모두 기존 워크플로 흡수(0 델타). 유일한 배포 전 조치는 **LLM·TMap 시크릿 등록(P6 후)** 과 **LLM 파라미터 remote config 주입**이며, 인프라 파이프라인 변경은 없다.

---

## 8. 컴플라이언스 요약 (U5 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-10 | **준수** | §1·§5 — LLM·TMap 어댑터 타임아웃·서킷은 앱(Resilience4j), 인프라는 A12·A11 알람. 폴백은 nfr-design PAT-U5-02 |
| RESILIENCY-09 | 준수 | §5 — LLM 쿼터 80%(A11)·**월 비용 예산(A15)**·오토스케일 SI §1 상속 |
| RESILIENCY-05·07 | 준수(증분) | §5 — LLM 비용·실패율·서킷 커스텀 메트릭·대시보드 편입(U5 LLM 첫 소비자) |
| RESILIENCY-01·02·08·11·12 | 상속 | Multi-AZ·백업·워크로드 분류(LLM Medium·저장 Critical)는 SI·U1 — 솔버는 인프로세스(신규 워크로드 없음) |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속. LLM 파라미터·폴백 순서 변경은 OPS-01(§7) |
| SECURITY-11 | **준수** | §1 — LLM 아웃바운드·필드 최소화(G181)·서버 재조회(D31)는 nfr-design PAT-U5-06(앱), 인프라는 국외 이전 고지 경계 제공 |
| SECURITY-06·07 | 준수 | §1 — 최소 권한(어댑터별 시크릿 ARN)·NAT 경유·SG deny-by-default(SI §2.2·§7.1) |
| SECURITY-10·12 | 준수(상속) | §1·§7 — LLM·TMap 키 Secrets Manager·평문 0·시크릿 스캔·솔버 라이브러리 잠금 파일(U1 CI 상속) |
| SECURITY-05 | 준수 | LLM 출력 스키마 검증·closed-set은 nfr-design PAT-U5-06(앱) — 인프라는 아웃바운드 목적 최소화 정합 |
| SECURITY-01·02·03·14 | 상속 | 암호화·중개 로깅·앱 로깅·알림은 SI·U1 정본 — LLM 어댑터 로그도 U1 마스킹·상관 ID·D31 경계 |
| SECURITY-08·09·13·15 | 상속 | 소유권(앱 nfr §5)·하드닝·무결성·예외 처리 U1 스캐폴드 상속 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — 영업시간·이동 부등식·고정 블록·폴백 결정성·상태 머신·직렬화는 Code Generation 실행(U5 DoD 1급 대상) |

**차단 소견 없음.** U5 신규 AWS 리소스 0(§6) — SI가 예약한 NAT·시크릿·비용 계측·워커 프레임 내에서 인프라가 종결된다. 솔버는 인프로세스 CPU 연산(태스크 사이징 재검토만), LLM은 아웃바운드 1종이다. **비개발 선결 P6(LLM 벤더·비용 산정)** 는 벤더·단가·후보 크기 최종값·국외 이전 여부를 확정하는 대상이며(§0·nfr-requirements.md §7.1), 본 인프라 산출물(아웃바운드 경로·시크릿 항목·비용 계측·연산 배치)은 벤더 중립으로 준비되어 비차단이다.
