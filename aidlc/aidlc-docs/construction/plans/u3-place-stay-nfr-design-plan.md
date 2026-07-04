# U3 숙소·장소 데이터 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U3 (M7 Place Data 기반 · M3 숙소 탐색 · M4 등록 숙소 · M5 OTA 딥링크 + `shared/map` 카카오 지도 SDK)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U3(리스크·선결 P2·P4) · [stories.md](../../inception/user-stories/stories.md) Epic 3(US-E3-01~11) · [requirements.md](../../inception/requirements/requirements.md) §6·§7.2(G31·G33·G34·G196·G192·G133/G148·G32·G106) · [component-dependency.md](../../inception/application-design/component-dependency.md) §6 외부 어댑터 격리 표 · [shared-infrastructure.md](../shared-infrastructure.md)(SI)
> 산출: `../u3-place-stay/nfr-requirements/{nfr-requirements.md, tech-stack-decisions.md}`, `../u3-place-stay/nfr-design/nfr-design-patterns.md`, `../u3-place-stay/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U3의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 실행으로 통합한다.** U3는 **외부 의존이 1차 유닛 중 최다(외부 연동 4 + 지도 SDK, unit-of-work.md U3 예상 규모)** 인 데이터 척추 유닛이다. 인프라·플랫폼의 전역 결정은 **전부 정본을 상속하고 재결정하지 않으며**, U3 증분(외부 어댑터 복원력·POI 하이브리드 캐싱·POI 정본 저장·외부 아웃바운드·커버리지 게이트)에 집중한다. **다만 외부 API 의존이 크므로 복원력·캐싱 설계는 충실히 기술한다.**

| 정본 | 출처 | U3에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | [u1-foundation-nfr-design-plan.md](./u1-foundation-nfr-design-plan.md) §2 | 재사용(AWS 서울 리전·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — U3는 SI를 재정의하지 않음. 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2·§3 | RN Expo + Spring Boot Kotlin + PostgreSQL 재사용. U3는 **지도 SDK·외부 API 클라이언트·POI 캐시·유사도 매칭**만 구체화 |
| 지도·장소 벤더 | requirements.md §2.3·D08 | **카카오(검색·지오코딩) + TMap(도로 거리) + 네이버(2차 폴백)** — 재확정 금지. **TMap(도로거리)은 C2 소유로 U5 범위**(§ 아래 경계 주) |
| POI 저장 정책 | requirements.md D13 · component-dependency.md §3.1 | 하이브리드(TourAPI 등 허용 소스 정본 영구 + 지도 API TTL) 재사용 — NFR화·인프라 매핑만 U3 |
| 외부 어댑터 격리 | component-dependency.md §6 | 포트 뒤 격리(AD-2)·RESILIENCY-10 4요소 계약(U1 PAT-RES-01 정본) 상속 — U3 어댑터가 자기 계약 표 작성 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천·**지역당 POI 5천** | U3 신규 컴퓨트·오토스케일 산정 없음(외부 호출은 I/O 바운드·기존 태스크 흡수) |
| 운영 프로세스·관측성 | U1 nfr-design-patterns.md OPS-01~03·PAT-OBS-01~03 | 상속 — 외부 쿼터 알람(SI §8.2 A11)·어댑터 실패율 알람(A12)의 첫 소비자로 편입 |

**설계 지배 원칙**: 규모 정합(과설계 금지) — 값 1개·테이블 몇 개에 새 서비스를 만들지 않되(SI §2.1·§4 기조), 외부 의존이 큰 유닛이므로 **복원력(폴백 체인)·캐싱(소스별 정책)·커버리지 게이트**는 충실히 확정한다.

---

## 2. 선결 과제 연결 — P2·P4 (설계 차단 요인 · unit-of-work.md U3 §8)

U3는 **비개발 선결 과제 2건이 설계 착수를 차단**한다. 본 산출물은 벤더·정책 중립 요건까지 확정하고, 두 과제의 답이 데이터 모델·캐싱 정책을 확정하는 지점을 명시한다(과제 미완 시 해당 결정은 '차단' 표기).

| 과제 | 시점 | 차단하는 설계 결정 | U3 산출물의 대응(중립 설계) |
|---|---|---|---|
| **P2 지도 API 약관 검토·계약**(카카오·TMap·네이버 — 특히 D13 **사용자 확정 스냅샷 영구 저장의 적법성** 법무 확인) | **U3 Functional Design 착수 전(설계 차단)** | (a) 지도 API 소스의 캐싱 허용 범위·TTL 상한 (b) 사용자 확정 스냅샷 영구 저장 가부·필드 범위 (c) 출처 표기 의무 | 스냅샷 **필드 최소화** + **소스별 보존 정책 컬럼**(source·retention_policy·cached_at)으로 정책 변경 흡수. 지도 API 소스는 **영구 저장 표면 0(인메모리 TTL만)** 로 P2 리스크 격리. 약관 확정 전에는 지도 API 소스 스냅샷을 **보류**(TourAPI 허용분만 영구) |
| **P4 TourAPI 활용 신청·캐싱 조건 확인** | **U3 착수 전(설계 차단)** | 캐싱 허용 범위가 M7 정본 저장 정책(D13 (a))을 좌우 — **어느 필드를 정본 영구 저장할지** | `TourContentPort` 어댑터 뒤 격리. 캐싱 허용분만 정본 테이블 영구, 불허분은 조회 시점 소비. 캐싱 조건 미확정 시 정본 테이블 스키마의 보존 정책 컬럼으로 **소급 축소 가능**하게 설계 |
| P5 OTA 제휴 계약 | U3 출시 전(개발 무차단) | 파트너별 딥링크 정책·수수료 고지 문안 | 딥링크 템플릿·화이트리스트 **remote config화**(G31·G196) — 개발은 검색 딥링크로 진행, 계약 후 값만 교체 |

**차단 경계 명시**: P2·P4는 **비개발(법무·제휴) 선결**이며 본 NFR·Infra 산출물(중립 요건)은 차단되지 않는다. 차단되는 것은 **U3 Functional Design의 스냅샷·정본 테이블 확정 스키마**뿐이다 — 본 산출물은 그 확정을 정책 컬럼·어댑터 격리로 **소급 대응 가능**하게 준비한다.

---

## 3. 통합 실행 체크리스트

### 3.1 준비 — 정본 로드

- [x] U3 범위 로드 (unit-of-work.md U3 — 목적·포함·제외·산출물·DoD·리스크·선결 P2·P4)
- [x] Epic 3 스토리 로드 (stories.md US-E3-01 탐색·02 필터/정렬·03 상세·04 위시리스트·05 OTA 딥링크·06 등록·07 다중 거점·08 직접등록 3경로·09 목록·10 결과없음·11 부분실패)
- [x] NFR 정본 로드 (requirements.md §6.1 성능·§6.2 복원력·§6.3 워크로드 분류(M3·M5=Medium)·§6.4 보안·§6.7 관측성·§6.8 규모(POI 5천))
- [x] 제안 기본값 로드 (§7.2 G31·G33·G34·G106·G129·G133/G148·G32, §7.4 G196·G192)
- [x] 외부 어댑터 격리 표 로드 (component-dependency.md §6 — 카카오·TMap·네이버·TourAPI·OTA + §3.1 POI 생명주기 D13)
- [x] 공유 인프라 정본 로드 (SI §1 컴퓨트·§2 NAT 아웃바운드·§3 RDS·§6 시크릿·§8.2 A11·A12 외부 알람 프레임·§9 환경)
- [x] U1 정본 로드 (nfr-design PAT-RES-01 4요소 계약·PAT-RES-03 우아한 저하·PAT-OBS-03 알람, tech-stack §2 서버 스택)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 3.2 NFR Requirements (nfr-requirements.md)

- [x] 성능 — 탐색 응답(캐시 히트/미스·부분 응답), 라이브 가격 온디맨드 단건 타임아웃(D09 일괄 금지), POI 후보 풀 조회(closed-set)
- [x] 복원력(핵심) — 외부 API별 타임아웃·재시도·서킷·폴백 체인(카카오→네이버 D08→TTL 캐시→'확인 불가' G8 / TourAPI→정본 잔존→'미확인' / 전체 실패→수동 등록 우회 US-E3-08·11) RESILIENCY-10
- [x] 복원력 — TMap 도로거리(C2/U5) 경계 명시, U3 거리는 직선거리(G34) 로컬 계산 정본
- [x] 캐싱·데이터 — D13 소스별 캐싱 정책 NFR화(지도 API 영구 캐싱 금지·TTL·정적 콘텐츠 일 1회 갱신 G196, POI 정본은 TourAPI 등 허용 소스)
- [x] 규모 — 지역당 POI 5천(G142)·POI 정본 볼륨·라이브 호출 rate-limit(G196)·외부 쿼터 80% 알람
- [x] 보안 — 외부 API 키 시크릿(NFR-U1-SEC-11 상속)·URL 파싱 SSRF 방어(G31 페이지 fetch 없음)·크롤링 금지 준수·위시리스트/등록 숙소 소유권(SECURITY-08)
- [x] 커버리지 게이트 — G192 출시 판정(좌표 95%·영업시간 70%) 측정·문서화·'미확인' 표기
- [x] 미결/선결 — P2·P4 표기 + U3 신규 벤더 종속 미결 0 확인(GD·SI 상속)
- [x] 확장 규칙 컴플라이언스 요약(준수/N/A + 근거)

### 3.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] 카카오맵 SDK — RN 네이티브 브리지 config plugin(prebuild) vs WebView JS SDK vs 목록 뷰 폴백 → 권고 + 대안·사유
- [x] 서버측 외부 API 클라이언트 — HTTP 클라이언트(Spring RestClient/WebClient) + 서킷브레이커(Resilience4j) → 권고 + 대안·사유
- [x] POI 캐시 저장 — PostgreSQL 정본 + 인메모리 단기 TTL(Caffeine) / Redis(ElastiCache) 도입 여부 규모 대비 권고
- [x] 유사도 매칭 — 문자열 유사도 알고리즘(canonical POI ID G133/G148) → 권고 + 대안·사유
- [x] 각 선택에 대안 + 사유 표기, 서버 대분류는 U1 재사용 명시

### 3.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U3-01 외부 어댑터 서킷브레이커·폴백 체인(벤더 폴백 D08)
- [x] PAT-U3-02 POI 하이브리드 캐싱(정본 영구 vs 실시간 TTL — 소스별 D13)
- [x] PAT-U3-03 라이브 가격 온디맨드 호출(목록 일괄 금지 D09)
- [x] PAT-U3-04 URL 파싱 SSRF 가드(화이트리스트·fetch 없음 G31)
- [x] PAT-U3-05 복귀 핸드오프 상태 머신(G32)
- [x] PAT-U3-06 커버리지 게이트 데이터 품질 검증(G192)
- [x] 각 패턴: 문제/적용/검증 기준 표기
- [x] 전역 상속(보안 기본 PAT-SEC-01·05·관측성 PAT-OBS-01·03·복원력 PAT-RES-01·03) "shared/U1 정본 상속" 표기

### 3.5 Infrastructure Design (infrastructure-design.md)

- [x] 외부 API 아웃바운드(NAT 경유·SG 아웃바운드 상속) + 시크릿 보관(카카오·TMap·TourAPI·네이버 키 — Secrets Manager)
- [x] POI 정본 저장(기존 RDS 스키마 확장·Flyway — 테이블 ~8~9개)
- [x] 단기 캐시 계층 결정(인메모리 vs ElastiCache — 규모 대비 권고)
- [x] POI 동기화 배치 잡(TourAPI 일 1회) + 인기 장소 집계·OTA 정적 콘텐츠 갱신 — 기존 ShedLock 스케줄러 재사용
- [x] 커버리지 게이트 측정 잡
- [x] SI 대비 델타 표(신규 리소스 — 있으면 캐시/없으면 0)
- [x] 배포 — shared deployment-architecture.md 재사용 갈음 절(서버 흡수 + 지도 SDK config plugin의 EAS 빌드 델타)
- [x] 컴플라이언스 요약

### 3.6 검증·마감

- [x] 추적 ID(NFR-U3-xx·PAT-U3-xx·SECURITY·RESILIENCY·D·G·N·GD·US-E3·CP1·P2·P4) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 규모 정합 점검 — 과설계 금지(값·테이블에 새 서비스 금지), 단 외부 의존 복원력·캐싱은 충실
- [x] 선결 과제(P2·P4) 설계 차단 요인 연결 명시 확인
- [x] 산출물 4종 생성 완료

---

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u3-place-stay/nfr-requirements/nfr-requirements.md` | U3 NFR 요구 — 성능·복원력(핵심)·캐싱/데이터·규모·보안·커버리지 게이트 + 선결 P2·P4 |
| `../u3-place-stay/nfr-requirements/tech-stack-decisions.md` | U3 스택 — 지도 SDK·외부 API 클라이언트·서킷브레이커·POI 캐시·유사도 매칭(서버 대분류는 U1 재사용) |
| `../u3-place-stay/nfr-design/nfr-design-patterns.md` | U3 패턴 6종 + 전역 상속 표기 |
| `../u3-place-stay/infrastructure-design/infrastructure-design.md` | U3 인프라 증분 + SI 델타 표 + 배포 갈음 절 |

## 5. 다음 단계

- **P2·P4 선결 완료 확인** 후 U3 Functional Design 착수(스냅샷·정본 테이블 확정 스키마).
- U3 Code Generation: 외부 어댑터 4종(fake 계약 테스트 D37)·서킷브레이커·POI 캐시·유사도 매칭·커버리지 게이트 측정 잡·CP1(U3→U4) 공급자 계약 구현.
- 본 계획 §1의 전역 결정 상속은 U4 이후에도 동일 재사용(재질문 금지).
