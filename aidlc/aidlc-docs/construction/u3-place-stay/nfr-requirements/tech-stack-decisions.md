# U3 숙소·장소 데이터 — 기술 스택 결정 (Tech Stack Decisions)

> 2026-07-04 · CONSTRUCTION · U3 NFR Requirements 산출물
> **전제**: 스택 대분류(RN Expo + Spring Boot Kotlin + PostgreSQL, D02)와 서버·클라이언트 라이브러리 정본(U1 [tech-stack-decisions.md](../../u1-foundation/nfr-requirements/tech-stack-decisions.md) S-1~S-13·C-1~C-8)은 **기확정**이다. U3는 그 위에 **지도 SDK·서버측 외부 API 클라이언트·POI 캐시·유사도 매칭** 4개 지점만 구체화하며, 나머지(프레임워크·인증·DB 접근·마이그레이션·상태 관리·HTTP·PBT)는 U1 결정을 상속한다.
> **재결정 금지**: 클라우드·인프라·서버 프레임워크는 SI·U1 정본 상속(nfr-requirements.md §7.2). 지도·장소 벤더는 **D08 기확정**(카카오+TMap+네이버 — 재확정 없음). 본 문서는 U3의 선택지가 있는 지점만 대안 비교 후 권고한다.

---

## 1. 결정 요약표

| # | 영역 | 결정 | 상속/증분 | 근거 ID |
|---|---|---|---|---|
| U3-C1 | 클라이언트 지도 SDK | **카카오 지도 네이티브 SDK — Expo config plugin(prebuild 네이티브 브리지)**, 폴백 = WebView JS SDK/목록 뷰 | 증분 | D02·D08, US-E3-03·08 |
| U3-C2 | 지도 상호작용(핀·검색) | 카카오 SDK 브리지(`shared/map`) — 핀 표시·핀 지정·바운즈 | 증분 | US-E3-08, G34 |
| U3-S1 | 서버 외부 API 클라이언트 | **Spring `RestClient`(동기·블로킹) + Resilience4j**(타임아웃·서킷·재시도·벌크헤드) | 증분(U1 Spring Boot 상속) | RESILIENCY-10, D08 |
| U3-S2 | 어댑터 격리 | 포트 4종(`PlaceSearchPort`·`GeocodingPort`·`TourContentPort`·`DeeplinkBuilder`) 뒤 벤더 은닉 | 증분 | component-dependency §6(AD-2) |
| U3-S3 | POI 정본 저장 | **PostgreSQL 정본 테이블(TourAPI 허용 소스 영구)** — U1 RDS·Flyway·JPA 재사용 | 상속(U1 S-8·S-9) | D13, N/A 신규 |
| U3-S4 | 단기 캐시 | **인메모리 Caffeine(단기 TTL)** — 지도 API 소스·핫 탐색. ElastiCache 미도입 | 증분 | D13, G142, NFR-U3-DATA-05 |
| U3-S5 | canonical POI 유사도 매칭 | **좌표 50m 블로킹(Haversine) + 명칭 정규화 + Jaro-Winkler/정규화 Levenshtein**(commons-text) | 증분 | G133/G148, D17 |
| U3-S6 | 거리 계산 | **Haversine 직선거리(로컬·외부 미호출)** — TMap 미사용(도로거리는 C2/U5) | 증분 | G34·D25, NFR-U3-RES-04 |
| U3-PD | 클라우드·시크릿·로그·알림 | **U1 PD-1~9 종결 결과·SI 상속** — 외부 키는 Secrets Manager(SI §6) | 상속 | SI, U1 infrastructure-design §0 |

---

## 2. 클라이언트 지도 SDK — 카카오 config plugin (U3-C1·C2)

D02가 **"Expo development build + prebuild 운용(국내 지도 SDK config plugin 대비)"** 을 U1 스캐폴드에서 이미 확정했다. U3는 그 빌드 체계 위에 **카카오 지도를 실제로 붙인다**. D08이 지도 벤더(카카오)를 확정했으므로 벤더 선택이 아니라 **연동 방식**을 비교한다.

| 기준 (U3 요구) | **A. 카카오 네이티브 SDK + config plugin (권고)** | B. WebView + 카카오 지도 JS SDK | C. react-native-maps(구글/기본) |
|---|---|---|---|
| 벤더 정합(D08 카카오) | **직접** — 카카오 네이티브 지도 | JS SDK로 카카오 사용 가능 | 카카오 아님(구글) — D08 위배 |
| 핀·바운즈·핀 지정(US-E3-08) | 네이티브 성능·제스처 | WebView 브리지 왕복 지연 | 네이티브이나 벤더 불일치 |
| Expo prebuild 정합 | config plugin으로 네이티브 모듈 주입(D02 확정 이유) | 네이티브 모듈 최소(WebView만) — 리스크 낮음 | config plugin 필요 |
| 리스크(unit-of-work.md U3) | **config plugin 불안정 리스크** — 초기 스파이크 선행 필요 | 낮음(폴백 후보로 적합) | 벤더 교체 비용 |
| 출처 표기·약관(D13) | SDK 약관 준수 | 동일 | — |

- **권고: A(카카오 네이티브 SDK + Expo config plugin)**, **폴백 = B(WebView JS SDK) 또는 목록/좌표 텍스트 뷰**. 근거: D08(카카오) + D02(config plugin 대비 확정)의 정합, 핀 지정·바운즈 등 US-E3-08 상호작용의 네이티브 품질. **리스크 완화(unit-of-work.md U3)**: config plugin 검증을 **U3 초기 스파이크로 선행**하고, 실패 시 **지도 표시 범위 축소 폴백**(목록 뷰·WebView)을 설계에 상비(NFR-U3-RES-01 클라이언트 행).
- **C 비채택 사유(1줄)**: react-native-maps는 구글 지도 기반으로 D08(카카오) 확정과 벤더가 불일치 — 국내 POI·지오코딩 정합이 떨어진다.
- **B는 폴백으로 채택**: config plugin 네이티브 모듈 리스크의 안전판(지도 로드 실패 시 목록 뷰와 함께).

## 3. 서버 외부 API 클라이언트 — RestClient + Resilience4j (U3-S1·S2)

U3는 외부 API 4종을 서버에서 호출한다(카카오·TourAPI·네이버 + OTA는 URL 조립만). U1의 Spring Boot 3.4(S-2)를 상속하되, **HTTP 클라이언트 + 복원력 라이브러리**를 확정한다.

### 3.1 HTTP 클라이언트 — Spring RestClient

- **선택**: Spring `RestClient`(Spring Framework 6.1+ 동기 클라이언트) — 어댑터별 베이스 URL·타임아웃·직렬화 구성. 저빈도·순차 소량 호출(계획 단계 Medium §6.3)이라 동기 블로킹으로 충분.
- **대안 비교(1줄)**: `WebClient`(리액티브)는 대량 병렬 팬아웃에 강점이나 U3 규모(수 RPS·소스 2종 병렬)에서 리액티브 스택 도입은 과설계(G142) — 탐색 2소스 병렬은 `RestClient` + 가상 스레드(JDK 21, U1 S-1)/`CompletableFuture`로 충분. 구식 `RestTemplate`은 유지보수 모드라 신규 채택 부적절.
- **선정 사유**: U1 스택(JDK 21·Spring Boot 3.4) 내 표준 클라이언트로 신규 의존 최소, 어댑터 격리(§3.2)와 정합.

### 3.2 복원력 라이브러리 — Resilience4j

- **선택**: **Resilience4j**(Spring Boot 3 통합 `resilience4j-spring-boot3`) — 어댑터 포트별 **타임아웃 + 서킷 브레이커 + (제한적) 재시도 + 벌크헤드**를 선언적 데코레이터로 적용(NFR-U3-RES-01의 4요소 계약 구현체).
  - 서킷: 카카오·TourAPI·네이버 **벤더별 독립 인스턴스**(1종 장애 격리 — 벌크헤드).
  - 폴백: 서킷 open·타임아웃 시 폴백 함수로 벤더 폴백 체인(카카오→네이버 D08→캐시)을 코드로 구성(nfr-design-patterns PAT-U3-01).
  - 설정 외부화: 타임아웃·실패율 임계·폴백 순서는 remote config(NFR-U1-MT-04 상속).
- **대안 비교(1줄)**: Spring Retry는 재시도에 특화되나 서킷·벌크헤드가 없어 RESILIENCY-10 전체를 자체 조립해야 함 / Spring Cloud Circuit Breaker 추상화는 벤더별 세밀 튜닝(독립 서킷)에 오히려 우회 계층 / 수동 구현은 서킷 상태 머신 재구현의 보안·정확성 위험.
- **선정 사유**: RESILIENCY-10의 4요소(타임아웃·재시도·서킷·폴백)를 단일 라이브러리로 표준 충족하고, 메트릭(Micrometer 연동)이 SI §8.2 A12(어댑터 실패율·서킷 open)와 직결. U1 PAT-RES-01이 "구현 라이브러리 Resilience4j 상당은 Code Generation 확정"으로 이미 예고 — 본 문서에서 확정.

## 4. POI 캐시 저장 — PostgreSQL 정본 + 인메모리 TTL (U3-S3·S4)

D13 하이브리드(nfr-requirements.md §4)를 스택으로 매핑한다.

| 계층 | 저장소 | 대상 | 근거 |
|---|---|---|---|
| **정본(영구)** | **PostgreSQL 정본 테이블**(U1 RDS·Flyway·JPA 재사용) | TourAPI 등 **캐싱 허용 소스**(D13 (a)) | 신규 인프라 0 — 스키마 확장만(infrastructure-design.md §2) |
| **단기 TTL 캐시** | **인메모리 Caffeine** | 지도 API(카카오·네이버) 소스·핫 탐색 결과(D13 (b)) | 영구 캐싱 금지·정본 아님·미스=재호출 |

- **캐시 계층 결정 (인메모리 vs ElastiCache — 규모 대비 권고)**:

| 기준 | **인메모리 Caffeine (권고)** | ElastiCache Redis | DB TTL 테이블 |
|---|---|---|---|
| 정본 적합성 | 캐시는 최적화(미스=외부 재호출, NFR-U1-AV-03 상속) — 태스크 간 불일치 무해 | 충족(공유) | 충족 |
| 지도 약관(P2) | **영구 저장 표면 0** — 프로세스 메모리·TTL만(리스크 최소) | 외부 저장소에 지도 데이터 상주(TTL이나 표면 증가) | DB 영구 스토리지에 상주(약관 경계 근접) |
| 규모 정합(G142) | **적합** — 수 RPS·POI 5천/지역, 태스크당 힙 여유(SI §1.2 2GB) | **월 ~$25 + 운영 표면 — 이득 0** | RDS I/O·정리 잡 추가 |
| 신규 인프라 | **0** | +1 서비스·SG·서브넷 | 0(RDS 내) |
| 쿼터 보호(G196) | 태스크당 캐시 → 최대 태스크 수배 호출(0.5 RPS에서 무의미) | 공유로 호출 최소화 | 공유 |

- **확정 권고: 인메모리 Caffeine(단기 TTL).** 근거: (1) 지도 API 소스는 **영구 저장 금지**(D13·P2)라 인메모리 TTL이 약관 리스크를 가장 줄인다, (2) 캐시는 정본이 아니므로 태스크 간 불일치는 외부 재호출로 무해(NFR-U1-AV-03), (3) 신규 인프라 0(G142). **ElastiCache 비채택 사유(1줄)**: 값·규모 대비 월 비용·운영 표면 증가가 이득을 상회 — SI §4와 동일 트리거(실측 50 RPS 초과 또는 U11 팬아웃)로 통합 재평가. **DB TTL 테이블 비채택 사유(1줄)**: 지도 API 데이터를 DB 영구 스토리지에 두는 것이 P2 약관 경계에 근접하고, TourAPI 허용분은 이미 정본 테이블이 있어 중복.

## 5. canonical POI 유사도 매칭 — 좌표 블로킹 + 문자열 유사도 (U3-S5)

동일 실세계 장소를 1개 canonical POI ID로 병합하는 매칭(G133/G148·D17, 하드 제약 NFR-U3-DATA-02)의 알고리즘을 확정한다.

- **선택**: **2단 매칭** —
  1. **좌표 블로킹**: 후보를 좌표 근접 50m(Haversine) 이내로 1차 축소(G133 — 전수 비교 회피, POI 5천/지역 성능).
  2. **명칭 정규화 + 문자열 유사도**: 한국어명 정규화(공백·특수문자·법인격 접미어 제거, 한국어명 정본 + 영문 alias) 후 **Jaro-Winkler(접두 가중) 또는 정규화 Levenshtein**(apache **commons-text** `similarity` 패키지)으로 유사도 산출 → **보수적 임계값** 이상만 병합.
- **대안 비교(1줄)**: 코사인 유사도(토큰 벡터)는 긴 문장에 강하나 짧은 상호명(2~5음절)에는 Jaro-Winkler가 적합 / 완전 일치만 사용하면 표기 변형(호텔/HOTEL·띄어쓰기)을 놓쳐 중복 canonical 발생 / ML 임베딩 매칭은 규모(G142)·운영 대비 과설계.
- **선정 사유**: 좌표 블로킹으로 계산량을 억제하고(50m + 유사도), **보수적 임계 + 운영자 보정 여지**(D17·unit-of-work.md U3 리스크)로 오매칭(그라운딩 오염 §2.1)을 격리. 매칭 멱등성·대칭성(같은 입력 재수집 시 canonical 불변)은 PBT + 골든 데이터셋 회귀로 상시 검증(unit-of-work.md U3 DoD).
- **라이브러리**: apache commons-text(유사도) — U1 Gradle version catalog에 추가, 잠금 파일 커밋(SECURITY-10 상속).

## 6. 거리 계산 — Haversine 직선거리 (U3-S6, TMap 미사용)

- **선택**: 거리 필터·정렬은 **Haversine 직선거리(로컬 계산 — 외부 미호출)**, 기준점은 검색 여행지 중심 좌표(G34). 소요시간 미표시(D25).
- **TMap 경계**: TMap(도로 거리) `RoadDistancePort`는 **C2 소유로 U5 범위**(component-dependency §6·§2.3) — U3는 TMap을 호출하지 않으며 직선거리가 1차 정본이다(NFR-U3-RES-04). 라우팅 실패 시 직선거리×우회계수(G106) 폴백도 C2/U5 소관.
- **선정 사유**: 직선거리 로컬 계산은 외부 라우팅 의존을 없애 가용성 우위(폴백 불요) + G34/D25 정합.

---

## 7. 상속·비결정 재확인

| 항목 | U3 처리 | 정본 위치 |
|---|---|---|
| 서버 프레임워크·인증·DB 접근·마이그레이션 | U1 S-2·S-3·S-8·S-9 상속(신규 0) | U1 tech-stack §2 |
| Expo SDK·TypeScript strict·prebuild·상태 관리·HTTP·PBT | U1 C-1·C-4·C-6·C-7 상속 | U1 tech-stack §3 |
| 클라우드·CI/CD·배포·시크릿·로그 스택 | GD-1~7·SI·U1 PD-1~9 종결 상속(재정의 금지) | SI, u1-foundation-nfr-design-plan.md §2 |
| 지도·장소 벤더(카카오·TMap·네이버) | **D08 기확정 — 재확정 없음**. U3는 연동 방식만(§2·§3·§6) | requirements.md §2.3·D08 |
| POI 저장 정책(하이브리드) | D13 기확정 — U3는 스택 매핑(§4) | requirements.md D13 |

**U3 신규 벤더·클라우드 결정 0 — 기존 스택 내 라이브러리 구체화(지도 SDK 연동·Resilience4j·Caffeine·유사도 알고리즘)만 유닛 증분이며, 지도 벤더·저장 정책은 D08·D13 확정 범위 내다(과설계 금지·재결정 금지).** 확정 스키마(스냅샷·정본 테이블)는 P2·P4 선결 후 Functional Design 소관(nfr-requirements.md §7.1).
