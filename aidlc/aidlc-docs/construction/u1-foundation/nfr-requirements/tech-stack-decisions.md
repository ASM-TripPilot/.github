# U1 기반·계정·온보딩 — 기술 스택 결정 (Tech Stack Decisions)

> 2026-07-04 · CONSTRUCTION · U1 NFR Requirements 산출물
> **전제**: 스택의 대분류는 D02(requirements.md §2.3)로 **기확정**이다 — RN(Expo) + Spring Boot(Kotlin) + PostgreSQL 모듈러 모놀리스(D04), PBT는 Kotest(서버)·fast-check(클라이언트). 본 문서는 그 확정 스택을 **U1 구현 수준(버전대·라이브러리·구성)으로 구체화**하고, 선택지가 있는 지점만 대안 비교 후 권고한다.
> **벤더 종속 항목**(메일 발송·시크릿 매니저·KMS·중앙 로그)은 본 문서에서 결정하지 않는다 — **클라우드 결정 대기** ([nfr-requirements.md](./nfr-requirements.md) §8, 질문 파일 `u1-nfr-infra-questions.md`).
> 버전 표기 원칙: "버전대(minor 라인)"까지 확정하고, 정확한 패치 버전은 Code Generation 시점의 최신 안정 패치로 고정(잠금 파일 커밋, SECURITY-10).

---

## 1. 결정 요약표

| # | 영역 | 결정 | 근거 ID |
|---|---|---|---|
| S-1 | 서버 언어·런타임 | Kotlin 2.1.x + JDK 21 (LTS) | D02 |
| S-2 | 프레임워크 | Spring Boot 3.4.x (Spring Framework 6.2, Jakarta EE) | D02 |
| S-3 | 인증 프레임워크 | Spring Security 6.4.x — 커스텀 JWT 필터 체인, deny-by-default | SECURITY-08·12 |
| S-4 | 소셜 OAuth | Spring Security OAuth2 Client 기반 서버측 코드 교환 — Google/Apple=표준 OIDC, 카카오/네이버=커스텀 provider 등록, `SocialOAuthPort` 어댑터 4종 | D22, G20, components.md M1 |
| S-5 | JWT 라이브러리 | Nimbus JOSE+JWT (spring-security-oauth2-jose 경유) | D36, SECURITY-08 |
| S-6 | 비밀번호 해시 | **argon2id 권고** (Spring Security `Argon2PasswordEncoder` + `DelegatingPasswordEncoder`) — bcrypt 비교 §2.5 | SECURITY-12, G22 |
| S-7 | 입력 검증 | Jakarta Validation (Hibernate Validator — spring-boot-starter-validation) | SECURITY-05 |
| S-8 | DB 접근 | **Spring Data JPA(Hibernate) 권고** + append-only 로그는 JDBC 직삽입 — jOOQ 비교 §2.7 | D02, N2 |
| S-9 | 마이그레이션 | Flyway (SQL-first, versioned migration) | D02, SECURITY-14 |
| S-10 | 서버 PBT | **Kotest Property Testing (kotest-property 5.9.x)** — PBT-09 요건 충족 확인 §2.9, jqwik 비채택 §2.10 | D02, PBT-08·09 |
| S-11 | 서버 테스트 보조 | Kotest(러너)+MockK+Testcontainers(PostgreSQL)+Konsist/ArchUnit(모듈 경계) | D37, NFR-U1-MT-01·02 |
| S-12 | 빌드 | Gradle 8.x Kotlin DSL 멀티모듈 + version catalog + 의존성 잠금 | D04, SECURITY-10 |
| S-13 | 로깅 | SLF4J + Logback + logstash-logback-encoder(JSON)·MDC 상관 ID | SECURITY-03 |
| C-1 | 클라이언트 | Expo SDK 54 버전대 (React Native 0.81, development build + prebuild) + TypeScript strict | D02 |
| C-2 | 소셜 인증 UI 흐름 | expo-auth-session(+expo-web-browser) 4종 공통 + iOS Apple은 expo-apple-authentication 병용 | D22, 스토어 심사 |
| C-3 | 토큰 저장 | expo-secure-store (iOS Keychain / Android Keystore) | SECURITY-12, NFR-U1-SEC-28 |
| C-4 | 상태 관리 | Zustand(클라이언트 상태) + TanStack Query v5(서버 상태) | D38(§6.1 300ms) |
| C-5 | 폼·스키마 | React Hook Form + Zod | SECURITY-05(클라이언트측 대응) |
| C-6 | HTTP 클라이언트 | axios + 토큰 회전 인터셉터(갱신 단일 비행 큐) | D36 |
| C-7 | 클라이언트 PBT | **fast-check 3.x + Jest(jest-expo)** — PBT-09 요건 §3.7 | D02, PBT-09 |
| PD | 메일 발송·시크릿·KMS·중앙 로그 | **클라우드 결정 대기** — Port/설정 외부화로 격리 | nfr-requirements.md §8 |

---

## 2. 서버 (Spring Boot + Kotlin)

### 2.1 Kotlin·JDK — Kotlin 2.1.x + JDK 21 LTS

- **선택**: Kotlin 2.1 버전대(K2 컴파일러 안정 라인), JDK 21(LTS — 가상 스레드 사용 가능, Spring Boot 3.x 공식 지원 라인).
- **대안 비교**: JDK 17 LTS는 안정적이나 잔여 지원 기간과 가상 스레드 부재로 21 대비 이점 없음 / Kotlin 1.9는 K2 이전 라인으로 신규 프로젝트 채택 이유 없음.
- **선정 사유**: 그린필드에서 최신 LTS 조합이 5년 유지보수(D01 실서비스)에 가장 유리하고, Spring Boot 3.4가 공식 지원하는 조합이다.

### 2.2 Spring Boot 3.4.x

- **선택**: Spring Boot 3.4 버전대 — Spring Framework 6.2, Jakarta EE 10 네임스페이스.
- **대안 비교**: Ktor는 경량이나 Security·Data·Validation 생태계 통합을 자체 조립해야 하고 D02가 Spring Boot를 기확정 / Boot 3.2·3.3 라인은 지원 종료가 앞당겨져 신규 채택 부적절.
- **선정 사유**: D02 확정 + 모듈러 모놀리스(D04)에 필요한 멀티모듈 부트 조립(`app` 모듈이 조립 소유, components.md)·오토컨피그 생태계.
- **구성 원칙**: `app` 모듈만 Spring Boot 애플리케이션 — `modules/auth`·`modules/profile`·`common/*`은 일반 Gradle 모듈(스프링 컨텍스트 조립은 app, NFR-U1-MT-01).

### 2.3 Spring Security 6.4.x — 인증 필터 체인

- **선택**: Spring Security 6.4 버전대. 필터 체인 구성(U1 스캐폴드 정본):
  1. 상관 ID 필터(MDC 주입, SECURITY-03) → 2. 요청 크기·컨텐츠 타입 가드(SECURITY-05) → 3. **JWT 인증 필터**(Bearer 검증 — 서명·exp·iss·aud, 무상태 `SessionCreationPolicy.STATELESS`) → 4. 브루트포스/rate-limit 필터(공개 인증 엔드포인트 한정, NFR-U1-SEC-07·08) → 5. 인가 규칙: **`anyRequest().authenticated()` 기본 + 공개 화이트리스트 명시**(NFR-U1-SEC-16) → 6. 보안 헤더 라이터(SECURITY-04) → 7. 전역 예외 변환(AuthenticationEntryPoint/AccessDeniedHandler — 일반화 메시지, SECURITY-15).
- **대안 비교**: 자체 인증 미들웨어는 deny-by-default·헤더·세션 정책을 전부 재구현하는 보안 위험 / Keycloak 등 외부 IdP 서버는 DAU 1천 규모(§6.8)에 운영 부담 과잉이고 커스텀 온보딩 여정(연령·약관 증적)과 결이 맞지 않음.
- **선정 사유**: SECURITY-08의 deny-by-default·토큰 매 요청 검증을 프레임워크 표준 기능으로 충족, 감사 이벤트 훅(AuthenticationEventPublisher)이 §4.4.3 감사 목록과 직결.

### 2.4 소셜 OAuth 클라이언트 — 4종 어댑터 전략

- **선택**: 서버측 `SocialOAuthPort` 뒤에 제공자별 어댑터 4종(components.md 어댑터 소유 맵). 클라이언트(expo-auth-session)가 authorization code(+PKCE)를 받아 서버로 전달하고, **토큰 교환·검증은 전부 서버**에서 수행한다(시크릿 비노출).
  - **Google**: 표준 OIDC — discovery 문서 기반, id_token 서명(JWKS)·iss·aud·nonce 검증.
  - **Apple**: OIDC이나 **client_secret이 정적 문자열이 아니라 p8 키로 서명한 ES256 JWT**(수명 6개월 이내) — 커스텀 client authentication 구현 필요. 이메일 비공개 릴레이 대응은 정책 확정(G20 — 별도 계정 허용).
  - **카카오**: OIDC 지원(활성화 필요) — OIDC 모드로 등록하되 사용자 정보 보강은 REST API 병용. 커스텀 provider 등록(discovery 비표준 부분 수동 구성).
  - **네이버**: **OIDC 미지원(순수 OAuth 2.0)** — id_token 없음, 액세스 토큰으로 프로필 API 조회 후 `sub` 상당값(response.id) 매핑. 완전 커스텀 provider.
- **대안 비교**: 클라이언트 SDK 직접 로그인(카카오/네이버 네이티브 SDK) 후 토큰 서버 검증 방식은 UX가 낫지만 네이티브 모듈 4종 추가로 Expo prebuild 리스크 증가 — 1차는 expo-auth-session 웹 플로 통일, 네이티브 SDK 전환은 어댑터 뒤 격리로 후속 검토.
- **선정 사유**: 4종의 프로토콜 편차(OIDC 2 + 준OIDC 1 + 순수 OAuth 1)를 Port 하나로 흡수해 M1 도메인 로직이 제공자 무지(agnostic)하게 유지 — D37 fake 테스트 지점과 일치.

### 2.5 비밀번호 해시 — bcrypt vs argon2id 비교 후 **argon2id 권고**

| 기준 | bcrypt | argon2id |
|---|---|---|
| 알고리즘 성격 | CPU 비용만(비용 인자 2^n) | **메모리 하드**(메모리·반복·병렬 3축) — GPU/ASIC 크래킹 내성 우수 |
| 표준 지위 | 검증된 구관(1999) | **PHC 우승(2015)·OWASP 1순위 권고·RFC 9106** |
| 입력 제약 | **72바이트 절단**(긴 패스프레이즈 무단 절단 위험) | 길이 제약 없음(자체 상한 128자는 입력 검증으로, NFR-U1-SEC-01) |
| Spring 지원 | `BCryptPasswordEncoder` 내장 | `Argon2PasswordEncoder` 내장(Bouncy Castle 의존 추가) |
| 운영 비용 | 낮음 | 요청당 메모리 점유(권고 파라미터 ~19MiB) — 동시 로그인 병렬도 × 19MiB를 힙 외 산정 필요(§2 규모에서 무리 없음) |
| 파라미터 상향 경로 | cost 상향+재해시 | 3축 상향+재해시 — 동일 |

- **권고**: **argon2id**, 초기 파라미터 OWASP 최소 권고선 `m=19456KiB(19MiB), t=2, p=1` — 검증 지연 실측 100–300ms 범위로 캘리브레이션(NFR-U1-SEC-02의 (c)). `DelegatingPasswordEncoder`(`{argon2}` 프리픽스)로 감싸 해시 자기 기술·**로그인 시 투명 재해시 업그레이드 경로** 확보(NFR-U1-SEC-02의 (a)).
- **bcrypt 비채택 사유(1줄)**: 72바이트 절단과 GPU 내성 열위 — 그린필드에서 구관을 택할 이유가 없다(requirements §6.4의 "bcrypt/argon2" 허용 범위 내에서 상위안 선택).

### 2.6 입력 검증 — Jakarta Validation (Hibernate Validator)

- **선택**: spring-boot-starter-validation — 전 요청 DTO에 선언적 제약(@Email·@Size·@Pattern·커스텀 @Adult/@NicknamePolicy), 컨트롤러 `@Valid` 일괄 적용 + 검증 실패 전역 핸들러에서 구조화 400.
- **대안 비교**: Konform 등 Kotlin 전용 검증 DSL은 어노테이션 메타데이터 기반의 프레임워크 통합(OpenAPI 스키마 파생 등)이 없음.
- **선정 사유**: SECURITY-05의 "전 API 핸들러 검증 라이브러리/스키마 사용" 요건을 프레임워크 관례로 강제 — 누락이 코드 리뷰가 아닌 컨벤션 테스트로 검출 가능.

### 2.7 DB 접근 — JPA vs jOOQ 비교 후 **Spring Data JPA 권고**

| 기준 | Spring Data JPA (Hibernate) | jOOQ | Spring Data JDBC |
|---|---|---|---|
| U1 워크로드 적합성 | **CRUD·상태 전이·연관 소량 — 최적** | 타입세이프 복잡 SQL·집계에 강점(U1엔 과잉) | 단순하나 생태계·감사 지원 얇음 |
| 파라미터 바인딩(SECURITY-05) | 기본 바인딩 | 기본 바인딩 | 기본 바인딩 |
| 감사 필드·낙관적 잠금 | @CreatedDate·@Version 표준 제공 | 수동 | 부분 |
| 학습·운영 | 사실상 표준, 인력 풀 최대 | 라이선스(상용 DB 아님—OSS 무료)·코드젠 파이프라인 추가 | — |
| 위험 | N+1·지연 로딩 실수 | — | — |

- **권고**: **Spring Data JPA** — U1 엔티티(계정·동의·세션·프로필 등 10~11개)는 전형적 CRUD+상태 머신이며 복잡 질의가 없다. N+1 위험은 U1 규모에서 fetch 전략 리뷰로 통제.
- **단서 2건**: (1) **append-only 법정 로그·동의 증적은 JPA 엔티티 UPDATE 경로를 원천 배제**하기 위해 INSERT 전용 리포지토리(JdbcTemplate 직삽입 또는 @Immutable) + DB 권한 분리로 이중 강제(NFR-U1-SEC-23·24). (2) 후속 유닛(U3 POI 집계·U7 통계)에서 복잡 질의가 늘면 해당 모듈 한정 jOOQ 병용을 재평가 — 모듈 경계(D04) 덕분에 모듈별 선택 가능.
- **jOOQ 단독 비채택 사유(1줄)**: U1의 도메인 CRUD 중심 워크로드에서 코드젠 파이프라인·수동 감사 구현 비용이 이득을 상회.

### 2.8 마이그레이션 — Flyway

- **선택**: Flyway(버전드 SQL 마이그레이션) — 마이그레이션에 **DB 역할·권한 분리 DDL 포함**(법정 로그 테이블에 앱 역할 DELETE/UPDATE 회수 — N2, 마이그레이션 리뷰로 검증하는 U1 DoD 항목).
- **대안 비교(1줄)**: Liquibase는 XML/멀티포맷 추상화가 강점이나 PostgreSQL 단일(D02) 환경에선 SQL-first Flyway가 리뷰 가독성·단순성 우위.
- **선정 사유**: append-only 제약을 "DB 레벨(권한 분리)로 강제"(unit-of-work.md U1 리스크 완화)하려면 권한 DDL이 형상 관리·리뷰 대상이어야 한다.

### 2.9 서버 PBT — Kotest Property Testing (PBT-09 요건 확인)

- **선택**: kotest-property 5.9 버전대(kotest-runner-junit5와 동일 버전 정렬) — D02 기확정.
- **PBT-09 요건 충족 확인**:

| PBT-09 요건 | Kotest 지원 확인 |
|---|---|
| 커스텀 제너레이터(도메인 타입) | `Arb` 컴비네이터(`Arb.bind`·`arbitrary { }`) — Account 상태·ConsentRecord·토큰 회전 체인용 도메인 Arb 정의 가능 |
| 자동 수축(shrinking) | 내장 shrinker + 커스텀 Arb에 shrink 함수 부착 가능 — PBT-08 필수 요건 |
| 시드 기반 재현 | 실패 시 시드 출력 + `PropTestConfig(seed = …)` 고정 재현, `PropertyTesting` 전역 설정으로 CI 시드 로깅 — PBT-08 시드 로깅 요건 |
| 기존 러너 통합 | JUnit Platform(Gradle `useJUnitPlatform()`) 위에서 Kotest 러너로 예제 테스트와 동일 파이프라인 실행 |

- **U1 적용 대상**(unit-of-work.md U1 DoD): 계정 상태 머신(UNVERIFIED→ACTIVE→DELETION_PENDING→DELETED) 불변식, 리프레시 회전(재사용 토큰 무효) 속성, 닉네임 생성기(패턴 준수·충돌 재추첨 수렴, G23), 동의 증적·프로필 직렬화 왕복 — 속성 최종 식별은 U1 Functional Design(PBT-01).

### 2.10 jqwik 비채택 사유

- jqwik은 JUnit 5 통합·상태 기반 테스트가 훌륭한 Java 진영 PBT이나, (1) **D02가 Kotest를 이미 확정**(requirements.md §2.3 "PBT 프레임워크: 백엔드 Kotest Property Testing"), (2) Java 어노테이션·리플렉션 지향이라 Kotlin 관용구(널러블·data class·코루틴)용 제너레이터를 별도 브리지로 구성해야 하며, (3) 테스트 프레임워크를 Kotest로 통일할 때 러너·assertion·PBT가 단일 스택인 이점(학습·설정·리포트 일원화)을 잃는다. 기능적 결격이 아니라 **스택 일관성에 따른 비채택**이다.

### 2.11 테스트 보조·빌드·로깅

- **테스트**: Kotest(러너·assertion) + MockK(Kotlin 네이티브 목킹 — Mockito 대비 코틀린 final class 친화) + **Testcontainers PostgreSQL**(실 엔진으로 마이그레이션·권한 분리·append-only 검증 — H2 비채택: 권한·DDL 방언 차이로 법정 로그 검증 불가) + Konsist 또는 ArchUnit(모듈 경계·"인증 가드 없는 핸들러" 검출, NFR-U1-SEC-20·MT-01).
- **빌드**: Gradle 8.x Kotlin DSL, version catalog(`libs.versions.toml`) 단일 정본, 의존성 잠금·검증 메타데이터 커밋(SECURITY-10), 멀티모듈 = 컴포넌트 경계(D04).
- **로깅**: SLF4J+Logback+logstash-logback-encoder(stdout JSON) — 마스킹 컨버터로 PII·토큰 패턴 구조적 차단(NFR-U1-SEC-22). 중앙 수집 대상은 클라우드 결정 대기(PD-4).

---

## 3. 클라이언트 (React Native + Expo)

### 3.1 Expo SDK 54 버전대 (React Native 0.81)

- **선택**: Expo SDK 54 라인 — **development build + prebuild** 운용(D02 확정: 국내 지도 SDK config plugin 대비 — 지도는 U3이나 빌드 체계는 U1 스캐폴드가 확정). TypeScript strict 모드.
- **대안 비교(1줄)**: bare React Native는 지도 네이티브 모듈 자유도가 높으나 OTA 업데이트·빌드 서비스·권한 모듈 관리 비용이 커서 D02가 Expo를 기확정.
- **선정 사유**: U1 시점 최신 안정 SDK 라인에서 시작해야 U8까지의 개발 기간 중 SDK 상향 횟수를 최소화. New Architecture 기본 라인.

### 3.2 소셜 인증 — expo-auth-session (+ expo-apple-authentication)

- **선택**: expo-auth-session + expo-web-browser로 4종 공통 authorization code + PKCE 플로(카카오·네이버는 discovery 수동 구성) → 코드를 서버에 전달, 교환은 서버(§2.4). **iOS의 Apple 로그인은 expo-apple-authentication(네이티브 시트) 병용** — 애플 심사 가이드라인상 소셜 로그인 제공 시 Apple 로그인 요구 + 네이티브 경험 요건 대응.
- **대안 비교(1줄)**: @react-native-seoul/kakao-login 등 제공자 네이티브 SDK 4종 도입은 UX 이점 대비 prebuild 네이티브 모듈 리스크·유지비가 1차 범위에 과잉(§2.4와 동일 판단 — 어댑터 뒤 후속 전환 여지).
- **선정 사유**: Expo 관리 플로 내에서 시크릿 없는 클라이언트(PKCE)·서버측 교환 구조(SECURITY-12 하드코딩 금지)와 정확히 합치.

### 3.3 토큰 저장 — expo-secure-store

- **선택**: expo-secure-store — iOS Keychain / Android Keystore 백엔드(NFR-U1-SEC-28, requirements §6.4 "OS 보안 저장소" 확정의 구현체). 저장 항목은 액세스·리프레시 토큰 2건으로 최소화(항목당 크기 제한 대비 — 값 2KB 상한 유의), 읽기는 기동 시 1회(§1.4).
- **대안 비교(1줄)**: AsyncStorage/MMKV는 평문 저장으로 SECURITY-12 위반 — 부적합. react-native-keychain은 기능 동등하나 Expo config plugin 표준 경로 대비 이점 없음.
- **선정 사유**: Expo 공식 모듈로 prebuild 무마찰, OS 보안 저장소 요건 직접 충족.

### 3.4 상태 관리 — Zustand + TanStack Query v5

- **선택**: 서버 상태(세션·프로필·취향)는 TanStack Query가 소유(캐싱·재검증·낙관적 업데이트 — §1.2 취향 저장 낙관적 UI의 구현 수단), 순수 클라이언트 상태(온보딩 진행 스텝·폼 임시값)는 Zustand.
- **대안 비교(1줄)**: Redux Toolkit(+RTK Query)은 동등 기능이나 보일러플레이트·러닝커브가 1차 팀 규모에 과잉, Recoil/Jotai는 서버 상태 계층이 없어 Query 조합이 어차피 필요.
- **선정 사유**: "서버 상태 = Query, UI 상태 = 경량 스토어" 분리가 화면 전환 300ms(D38) 하의 낙관적 UI·백그라운드 재검증(G5) 패턴과 합치.

### 3.5 폼·스키마 — React Hook Form + Zod

- **선택**: React Hook Form(비제어 입력 — 취향 7단계·가입 폼 리렌더 최소화) + Zod 스키마(이메일·비밀번호·닉네임·생년월일 규칙을 스키마로 선언). 클라이언트 검증은 UX용이며 **서버 검증(SECURITY-05)이 정본** — 규칙 수치(길이·패턴)는 서버와 대응표 유지.
- **대안 비교(1줄)**: Formik은 제어 컴포넌트 중심으로 다단계 폼 성능 열위, Yup은 TS 추론이 Zod 대비 약함.
- **선정 사유**: Zod 스키마가 API 응답 런타임 검증(계약 테스트 보조)에도 재사용되어 U1 계약 포인트(세션·온보딩 판정 API — U2 소비) 검증과 연결.

### 3.6 HTTP — axios + 토큰 회전 인터셉터

- **선택**: axios 인스턴스 + 인터셉터: (a) Authorization 자동 부착, (b) 401 시 **리프레시 회전을 단일 비행(single-flight) 큐로 직렬화**(동시 401 다발 시 회전 1회만 — 회전 체인 재사용 오탐(NFR-U1-SEC-05) 방지에 필수), (c) 갱신 실패 확정 시 토큰 삭제+로그인 이동.
- **대안 비교(1줄)**: fetch+자체 래퍼는 의존성이 없지만 인터셉터·타임아웃·재시도를 재구현해야 하고, ky는 RN 환경 검증 사례가 상대적으로 얇음.
- **선정 사유**: 토큰 회전(D36)의 클라이언트측 정합(단일 비행)이 검증된 패턴으로 존재.

### 3.7 클라이언트 PBT — fast-check (PBT-09 요건 확인)

- **선택**: fast-check 3.x + Jest(jest-expo 프리셋) — D02 기확정.
- **PBT-09 요건 충족 확인**: 커스텀 Arbitrary(`fc.record`·`fc.letrec` — 온보딩 스텝 상태·폼 입력 도메인 생성기) / 자동 shrinking 내장 / 시드 재현(실패 리포트에 seed·path 출력, `fc.assert(…, { seed, path })` 고정 재현 — PBT-08) / Jest 통합(별도 러너 불요).
- **U1 적용 대상**: 온보딩 진행 판정 순수 함수(약관+닉네임=완료, 취향 건너뛰기 무관 — G24/G157), 클라이언트 폼 스키마(서버 규칙 대응 불변), 토큰 저장/복원 왕복.

### 3.8 기타 클라이언트 확정

- **내비게이션**: Expo Router(파일 기반) — 셸·탭 구조는 U2 소관이나 스캐폴드 골격(`features/onboarding`·`shared/*` 배치)은 U1이 확정(unit-of-work.md U1).
- **린트·포맷**: ESLint + Prettier + TypeScript strict — 모노레포 공통 설정(서버 ktlint/detekt와 함께 U1 CI 게이트).

---

## 4. 클라우드 결정 대기 항목 (본 문서 비결정 — 재확인)

| 항목 | U1에서의 처리 | 결정 위치 |
|---|---|---|
| 트랜잭션 메일 발송(인증 링크·재설정) | `MailDeliveryPort` + 로컬/CI는 fake(개발용 캡처 도구 허용) | Infrastructure Design — **대기** (PD-1) |
| 시크릿 매니저 | 외부화 설정 주입 인터페이스만 확정 | Infrastructure Design — **대기** (PD-2) |
| KMS·JWT 서명 키 보관(kid 롤오버) | 키 교체 가능한 JWK 셋 구조만 확정(Nimbus 지원) | Infrastructure Design — **대기** (PD-3) |
| 중앙 로그·APM·알림 | stdout JSON까지 U1 보장 | Infrastructure Design — **대기** (PD-4) |
| 유출 비밀번호 온라인 조회 서비스 | 2단 검증 구조·타임박스만 확정(NFR-U1-SEC-04) | Infrastructure Design — **대기** (PD-6) |

상세 요건·질문은 [nfr-requirements.md](./nfr-requirements.md) §8과 `../../plans/u1-nfr-infra-questions.md` 참조.
