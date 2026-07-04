# U6 여행 중 실행·Plan-B — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U6 (M18 Trip Execution · **M9 Plan-B Detection** · **M10 Itinerary Recalculation** · **M11 Weather & Context** + `features/execution`·`features/planb`·`shared/location`)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U6(리스크·선결 **P1 위치정보사업 신고·P3 기상청 API**) · [stories.md](../../inception/user-stories/stories.md) Epic 6(US-E6-01~03)·Epic 7(US-E7-01~13) · [services.md](../../inception/application-design/services.md) **S2 재계획 예산(§S2.4 단계별 배분 정본)**·§0.1(실패 처리 — 기상청 무응답→트리거 침묵)·§0.2(TX·아웃박스)·§4(잡 J1·J2·J3) · [requirements.md](../../inception/requirements/requirements.md) §6(§6.1 D38·§6.3 M9·M10 High·§6.4·§6.5 위치정보법)·D10·D27·D34·D38·§7.3(G58·G195 폴링/상한·G182 위치 3층·G106) · [shared-infrastructure.md](../shared-infrastructure.md)(SI — 스케줄러 §1.2·아웃박스 §4·NAT §2.1·시크릿 §6·법정 로그 §5.3)
> 산출: `../u6-execution/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u6-execution/nfr-design/nfr-design-patterns.md`, `../u6-execution/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U6의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 실행으로 통합한다.** U6는 **실시간성(재계획 10초)·외부 의존(기상청)·배터리/위치·서버 배치 폴링이 핵심인 유닛**(unit-of-work.md U6: 스토리 최다·상태 머신 최다 · 외부 연동 1 = 기상청)이나, **인프라·플랫폼 골격과 재계획 연산(C1·C2)은 전역 정본을 상속하고 유닛 증분(서버 폴링·트리거 침묵·포그라운드 지오펜스·위치 3층 폴백·GPS 법정 로그)만 소유**한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U6에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | SI 헤더 · U1 nfr-design §2 | 재사용(AWS 서울·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — SI §0 유닛 표: "U6 = NAT 아웃바운드(기상청) + 스케줄 잡 프레임 + 위치 법정 로그 기록 개시(§5.3)". 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2 | Spring Boot Kotlin + PostgreSQL + RN Expo 재사용. U6는 **기상청 클라이언트·서버 폴링·포그라운드 위치·GPS 폴리라인** 지점만 구체화 |
| **재계획 연산(C1·C2) 정본** | **U5 [tech-stack-decisions.md](../u5-itinerary/nfr-requirements/tech-stack-decisions.md)·[nfr-design-patterns.md](../u5-itinerary/nfr-design/nfr-design-patterns.md)** | **재사용(신규 없음)** — C1 LLM Gateway·C2 Solver Engine·오라클 검증·warm-start·closed-set은 U5가 프로덕션 품질로 확정한 공통 자산. U6은 재계획(S2)에서 **소비자**로 호출(재정의 금지) |
| 하이브리드 트리거 정본 | requirements.md **D27** · services.md §S2.1 | **날씨·휴무=서버 배치 폴링+푸시, 위치(이동지연·체류초과)=클라이언트 포그라운드 한정** 상속 |
| 날씨 소스 정본 | requirements.md **D10**(기상청 공공데이터포털 단기예보+특보) · services.md §0.1 | 기상청 어댑터(격자 변환)·무응답 시 **트리거 침묵**(허위 알림 금지). 신청은 P3 |
| 성능 예산 정본 | **services.md §S2.4** (재계획 10초 = 1.0+**LLM 1.5**+1.5+4.5+0.5, 확정 재검증 1.0 별도) | **정본 상속·구체화** — U6 NFR이 이 단계 예산을 타임아웃·폴백 전환 시간과 함께 승격 |
| 위치정보법 정본 | requirements.md **D34**·N2·§6.5 · SI §5.3(append-only 법정 로그) | 위치 3층 동의(G182)·GPS 옵트인·법정 로그(**U1 append-only 테이블 재사용**) 상속 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 · 활성 여행 ≤1/사용자(D21) | 서버 폴링 대상 = 진행 중 여행 수(소량)·폴링 상한 G195(과설계 금지) |
| 스케줄러·아웃박스 정본 | SI §1.2(ShedLock)·§4(DB 아웃박스, 브로커 없음) · services.md §4(J1·J2·J3) | 재사용 — **기존 스케줄러·아웃박스 위에 U6 잡(날씨 폴링·휴무 재조회·자동 종료) 추가** |

**설계 지배 원칙**: U6 NFR 본질은 **① 실시간성(핵심 — S2 재계획 10초의 단계 예산·타임아웃·도착 감지 반응), ② 복원력(기상청 무응답→트리거 침묵·재계획 외부 실패→수동 수정 폴백·위치 불가→수동 입력), ③ 위치·배터리·프라이버시(포그라운드 한정 D27·GPS 옵트인 D34·폴링 상한 G195·배터리)** 이다. 재계획 연산(C1·C2)은 **U5 자산 재사용(신규 인프라·벤더 표면 없음)** 이고, 신규 외부는 **기상청 아웃바운드 1종**뿐이며 SI가 예약한 NAT·시크릿·스케줄러·법정 로그 프레임 내에서 종결된다(신규 AWS 리소스 사실상 0 — 과설계 금지).

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U6 범위 로드 (unit-of-work.md U6 — 목적·포함·제외·산출물·DoD·리스크·**선결 P1 위치정보사업 신고·P3 기상청 API·P7 위치기반서비스 약관**)
- [x] Epic 6 스토리 로드 (US-E6-01 도착 확인·방문 전이·02 현장 상세·03 다음 예정지 거리·외부 길찾기)
- [x] Epic 7 스토리 로드 (US-E7-01 수동 재계획·02 자동 트리거·03 영향 분석 입력·04 후보 2~3개·05 후보 정보·06 선택/유지/휴식·07 자동 재정렬·08 전후 비교·확정·09 이력·10 위치 수동 입력·11 외부 API 오류 폴백·12 AI/직접 방식·13 계획 vs 실제 경로)
- [x] 성능 예산 정본 로드 (**services.md §S2.4 단계별 배분** — 재계획 10초 = 1.0+**LLM 1.5**+1.5+4.5+0.5, 확정 재검증 1.0 별도, §0.3 타임아웃<예산 원칙)
- [x] 실패 처리 정본 로드 (services.md §0.1 — 기상청 무응답→**트리거 침묵**(허위 알림 금지)/재계획 외부 API 오류→수동 수정 폴백/위치 불가→수동 입력)
- [x] 확정 결정 로드 (D10 기상청·D14 plan/current·D19 여행 종료·D23 도착 프롬프트·D25 소요시간 미표시·D27 하이브리드 트리거·D34 위치정보법·D38 성능·G53 저장장소 우선·G54 휴식·G56 확정 재검증·G58/G195 상한·G106 이동시간·G116 순수 함수·G182 위치 3층)
- [x] 재계획 연산 정본 로드 (U5 C1 LLM Gateway·C2 Solver Engine — 재사용, 신규 없음)
- [x] 법률 정본 로드 (§6.5 위치정보법·N2 법정 로그 append-only·D34·G182 3층 동의)
- [x] 공유 인프라 정본 로드 (SI §0 U6 표·§2.1 NAT·§6 시크릿 프레임·§1.2 ShedLock 스케줄러·§4 아웃박스·§5.3 법정 로그·§8.2 A11·A12)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] **성능(핵심)** — S2 재계획 10초의 U6 구체화: services.md §S2.4 단계 예산 상속, **C1 LLM 타임아웃 1.5초(예산<)**, C2 후보 검증(후보당 1.5초), 확정 재검증 1.0초(예산 외), 도착 감지 반응(지오펜스 진입→프롬프트)·거리 갱신(진입/포커스 1회+수동 새로고침 G65). AI 일정 생성(S1 5초/20초)은 U5 — U6 아님 명시
- [x] **복원력(핵심)** — 기상청 무응답→**트리거 침묵**(허위 알림 금지 D27, 트리거 (a) 계열만 비활성·나머지 정상) / 재계획 외부 API 실패→수동 수정 폴백(US-E7-11) / 위치 불가→수동 입력(US-E7-10, G182) / 워크로드 M9·M10=High(수동 폴백 존재)·실행 허브 저장=Critical(§6.3)
- [x] **위치·배터리·프라이버시(핵심)** — 포그라운드 한정 감지(D27·G62)·GPS 옵트인(D34·N2)·폴링 주기 상한(G195 원격 구성)·저빈도 수집(1~5분 G55/G73)·배터리 예산
- [x] 가용성 — M9(감지)·M10(재계획) High(수동 수정 폴백 존재, §6.3), M18 실행 허브 저장/조회 Critical(여행 중 일정 접근)
- [x] 보안 — GPS 위치정보 법정 로그(**U1 append-only 테이블 재사용**·N2 append-only)·위치정보법 준수·소유권(accountId 스코핑)
- [x] 규모 — 서버 폴링 대상 = 진행 중 여행 수(활성 ≤1/사용자 D21)·폴링 상한 G195·과설계 금지(별도 워커·큐 미도입)
- [x] 미결·선결 — **P1(위치기반서비스사업 신고 — 법정 출시 전제)·P3(기상청 API 활용 신청)·P7(위치기반서비스 약관)** 연결, 개발은 비활성 플래그로 비차단
- [x] 확장 규칙 컴플라이언스 요약(준수/상속/N/A + 근거)

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] **기상청 API 클라이언트** — 격자 변환(LCC DFS)·단기예보+특보·**Resilience4j 서킷·타임아웃 재사용**·무응답 시 마지막 성공 TTL 후 트리거 침묵 (대안+사유)
- [x] **서버 배치 폴링** — 기존 ShedLock 스케줄러 재사용(J1 날씨 1h·J2 휴무 아침) vs 별도 워커/큐 (과설계 판단·대안+사유)
- [x] **클라이언트 지오펜스/포그라운드 위치** — expo-location(포그라운드 한정)·배터리 vs 백그라운드 지오펜스(D27 위반) (대안+사유)
- [x] **GPS 폴리라인 단순화** — Douglas-Peucker 등 클라/서버 단순화·원시 파기(G73) (대안+사유)
- [x] **재계획 = U5 C1/C2 재사용(신규 없음 명시)** — LLM Gateway·Solver·오라클·warm-start·closed-set은 U5 확정분, U6은 소비자
- [x] 각 선택 "재사용/증분" 명시 + 신규는 대안 1줄 + 사유

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U6-01 서버 배치 폴링 + 트리거 침묵(허위 알림 금지 D27 — 기상청·휴무)
- [x] PAT-U6-02 클라 포그라운드 지오펜스 감지(포그라운드 한정 D27·배터리·앱 재진입 일괄 재평가)
- [x] PAT-U6-03 재계획 확정 시점 재검증(G56 — C1/C2 재사용·오라클·warm-start)
- [x] PAT-U6-04 알림 상한·억제 학습(G58/G195 순수 함수 분리 G116·민감도·무시 억제)
- [x] PAT-U6-05 위치 3층 동의 폴백 체인(G182 — OS 권한 × 법정 동의 × GPS 옵트인 조합 매트릭스)
- [x] PAT-U6-06 GPS 옵트인·법정 로그·폴리라인 단순화(D34/N2 — 원시 파기·append-only 재사용)
- [x] 각 패턴: 문제/적용/U6 적용 지점/검증 기준(규칙 ID) 표기
- [x] 전역 상속 패턴(C1/C2 재사용·PAT-SEC·PAT-RES·PAT-OBS·PAT-PERF) "U5·U1·shared 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] 기상청 아웃바운드 — 기존 NAT(SI §2.1)·`sg-app` 443 재사용·시크릿 **`external/kma`**(SI §6 프레임 내 신규 항목)·타임아웃·격자 변환은 앱 소관
- [x] 서버 폴링 잡 — **기존 ShedLock 스케줄러 재사용**(J1 날씨 1h·J2 휴무 아침 07:00·J3 자동 종료 자정)·**원격 구성 G195**·별도 워커 미도입 근거
- [x] GPS 발자취 저장 — RDS 스키마(폴리라인·날씨 캐시 격자 등 ~8~9 테이블)·**위치정보 법정 로그 U1 append-only 테이블 재사용**·§5.3 월 1회 스냅샷 잡 개시
- [x] 트리거/푸시 M14(U8) 경계 — **U6는 TriggerFired 발행만**(아웃박스), FCM 푸시는 M14(U8) 소비 소관 명시
- [x] SI 대비 델타 표(신규 — 기상청 시크릿·서버 폴링 잡·GPS 폴리라인/날씨 캐시 스키마·법정 로그 기록 개시)
- [x] 배포 — shared deployment-architecture.md 재사용 갈음(서버 모듈 흡수·expo-location 네이티브 권한 델타 검토)
- [x] 컴플라이언스 요약

### 2.6 검증·마감

- [x] 추적 ID(NFR-U6-xx·PAT-U6-xx·SECURITY·RESILIENCY·PBT·D·G·GD·US-E6·US-E7·CP·S2·J) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 규모 정합 점검 — 진행 중 여행 수 기준(과설계 금지: 별도 폴링 워커·큐·ElastiCache 미도입) 확인
- [x] 산출물 4종 생성 완료

---

## 3. 선결 과제 연결 (P1·P3·P7 — §8)

| 과제 | 시점 | 본 계획의 비차단 대응 |
|---|---|---|
| **P1 위치기반서비스사업 신고**(위치정보법 제9조) + 위치정보 전문 법무 자문 | **U6 기능 출시 전 필수(법정 전제)** | GPS 발자취·위치 기반 트리거의 법정 전제. **신고 완료 전 해당 기능 비활성 플래그 운영**(unit-of-work.md U6) — 위치 동의 3층 모델·법정 로그 스키마는 U1에서 자문 반영 완료(append-only 재사용). 개발·테스트는 비활성 플래그로 비차단 |
| **P3 기상청 공공데이터포털 API 활용 신청** | **U6 착수 전** | M11 개발·트리거 (a) 계열(강수·특보)의 전제. 어댑터를 **fake로 개발·테스트 선행**(D37 계층 분리), 실키는 병렬 신청. 무응답 시 트리거 침묵으로 폴백(비차단) |
| P7 약관(위치기반서비스 약관) | U6 출시 전 | U1 동의 체계(3층 모델)에 실문안 탑재 — 버전 체계는 U1 완성, 문안 교체만(비차단) |

---

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u6-execution/nfr-requirements/nfr-requirements.md` | U6 NFR 요구 — 성능(핵심)·복원력(핵심)·위치/배터리/프라이버시(핵심)·가용성·보안·규모 + P1·P3 선결 |
| `../u6-execution/nfr-requirements/tech-stack-decisions.md` | 기상청 클라이언트·서버 폴링·포그라운드 위치·GPS 폴리라인 스택 + 재계획 U5 재사용(대안+사유) |
| `../u6-execution/nfr-design/nfr-design-patterns.md` | U6 패턴 6종(서버 폴링·포그라운드 지오펜스·확정 재검증·알림 억제·위치 3층 폴백·GPS 법정 로그) + 전역 상속(C1/C2·보안·복원력) 표기 |
| `../u6-execution/infrastructure-design/infrastructure-design.md` | U6 인프라 증분(기상청 아웃바운드·서버 폴링 잡·GPS/날씨 스키마·법정 로그 개시) + SI 델타 표 + 배포 갈음 |

## 5. 다음 단계

- U6 Functional Design(선행) 또는 Code Generation: 본 산출물의 위치 3층 조합 매트릭스(G182 테이블 주도 테스트)·트리거 판정 순수 함수(G116)·상태 머신(방문·여행 실행·재계획 세션)·재계획 하드 제약 4계열(C2 재사용)·CP3(소비자)·CP4(공급자)·CP5(TriggerFired) 계약을 구현. **E2E 종단 흐름(숙소 저장→등록→일정 생성→재계획, G118)을 U6 완료 시점에 CI 필수로 완성**(외부는 fake).
- 본 계획 §1의 전역 결정·C1·C2 재사용은 U7 이후에도 동일 재사용(재질문 금지). 위치·법정 로그·트리거 침묵 패턴은 위치 기반 후속(U9 어시스턴트·U11 공동편집)의 참조 정본이다.
