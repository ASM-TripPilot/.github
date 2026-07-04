# U2 앱 셸·홈·내비게이션 — 기술 스택 결정 (Tech Stack Decisions)

> 2026-07-04 · CONSTRUCTION · U2 NFR Requirements 산출물
> **전제**: 스택 대분류(RN Expo + Spring Boot Kotlin + PostgreSQL, D02)와 클라이언트 라이브러리 정본(U1 [tech-stack-decisions.md](../../u1-foundation/nfr-requirements/tech-stack-decisions.md) C-1~C-8)은 **기확정**이다. U2는 그중 **내비게이션 셸 계층**만 구체화하며, 나머지(Expo SDK·상태 관리·토큰 저장·HTTP·PBT)는 U1 결정을 상속한다. 서버 스택은 U1 app 모듈을 재사용하고 신규 없음.
> **재결정 금지**: 클라우드·인프라·서버 프레임워크는 SI·U1 정본 상속(nfr-requirements.md §7). 본 문서는 U2 클라이언트 셸의 선택지가 있는 지점만 대안 비교 후 권고한다.

---

## 1. 결정 요약표

| # | 영역 | 결정 | 상속/증분 | 근거 ID |
|---|---|---|---|---|
| U2-C1 | 내비게이션 라이브러리 | **Expo Router**(파일 기반) — 탭 셸·딥링크 라우팅 | U1 C-scaffold 확정(§3.8) 상속 + U2 구체화 | G6·G7, US-E2-03 |
| U2-C2 | 탭 셸 구조 | Expo Router **탭 레이아웃 그룹 + 탭별 독립 스택**(상태 보존) | 증분 | G6, US-E2-03 |
| U2-C3 | 딥링크·URL 스킴 | Expo Router linking config(앱 스킴 + universal/app links) | 증분 | G7, US-E2-06(스토어 링크) |
| U2-C4 | 상태 관리(세션·탭) | **Zustand**(전역 세션·앱 버전·탭 UI 상태) + **TanStack Query v5**(부트스트랩·홈 서버 상태) | U1 C-4 상속 | D38 |
| U2-C5 | 스플래시 | **expo-splash-screen**(부트스트랩 게이트까지 유지·수동 hide) | 증분 | G5, US-E2-01 |
| U2-C6 | 앱 버전 취득 | **expo-constants**(`nativeappVersion`/`expoConfig.version`) — 강제 업데이트 비교 입력 | 증분 | N4, US-E2-06 |
| U2-C7 | HTTP(부트스트랩·홈) | axios 인스턴스 + 토큰 회전 인터셉터 | U1 C-6 상속 | D36 |
| U2-C8 | 클라이언트 PBT | fast-check 3.x + Jest(jest-expo) — 스플래시 분기·탭 보존 속성 | U1 C-7 상속 | PBT-09, US-E2-01 |
| U2-S1 | 부트스트랩·홈 서버 | **U1 `server/app` 모듈 재사용 — 신규 서버 스택 0** | 상속 | unit-of-work.md U2 |

---

## 2. 내비게이션 — Expo Router vs React Navigation (비교 후 Expo Router 권고)

U1 tech-stack §3.8이 이미 **"내비게이션: Expo Router(파일 기반) — 셸·탭 구조는 U2 소관이나 스캐폴드 골격은 U1이 확정"** 이라고 정했다. U2는 이 확정을 **재검토가 아니라 U2 요구(탭 상태 보존 G6·딥링크 G7) 충족 여부 확인 후 채택**한다.

| 기준 (U2 요구) | **Expo Router (권고)** | React Navigation(순정) |
|---|---|---|
| U1 확정 정합 | **U1 C-scaffold가 이미 확정**(§3.8) — 재결정 불요 | U1 결정 뒤집기 필요(비용) |
| 탭 스택 상태 보존(G6) | 탭 레이아웃(`Tabs`) + 탭별 스택(`Stack`)에서 화면 언마운트 없이 상태 유지 — 세션 내 메모리 보존 직접 지원 | 동일 기능(내부적으로 React Navigation 사용) — Expo Router가 이를 래핑 |
| 딥링크 라우팅(G7) | **파일 경로 = URL 경로** 규약으로 딥링크→탭 활성화+스택 푸시가 linking config로 선언적 구성 | 수동 `linking` 매핑 테이블 유지 필요(경로 증가 시 드리프트) |
| 몰입 화면 탭바 숨김(US-E2-04) | 레이아웃 그룹(`(tabs)` 밖 스택)으로 탭바 노출/숨김을 파일 구조로 강제 | `screenOptions`로 화면별 수동 제어 |
| 후행 유닛 화면 편입 | 파일 추가 = 라우트 추가 — U3~U8 화면이 규약으로 꽂힘(unit-of-work.md U2 "화면이 꽂힐 자리") | 라우터 등록 코드 수정 필요 |
| Expo 정합 | Expo SDK 54 공식 라우터(prebuild·EAS 무마찰) | Expo에서 동작하나 라우팅은 별도 관리 |

- **권고: Expo Router.** U1이 확정한 선택이며, U2의 3대 내비 요구(G6 상태 보존·G7 딥링크·US-E2-04 탭바 숨김 규칙)를 **파일 구조 규약으로 선언적 강제**해 후행 유닛의 내비게이션 파편화(unit-of-work.md U2 리스크)를 구조적으로 차단한다.
- **React Navigation 비채택 사유(1줄)**: Expo Router가 React Navigation을 래핑하므로 저수준 API가 필요하면 언제든 접근 가능 — 순정 단독 채택은 U1 확정을 뒤집을 뿐 이득이 없다.

## 3. 탭 셸 구조 — 탭별 독립 스택 + 상태 보존 (U2-C2)

- **선택**: Expo Router 레이아웃 — `app/(tabs)/_layout.tsx`에 5탭(홈·탐색·일정·기록·마이) `Tabs`, 각 탭은 자체 `Stack`. 탭 전환 시 비활성 탭 스택을 **언마운트하지 않고 유지**(G6 세션 내 메모리 보존), 재탭 시 스크롤 탑 훅. 몰입 화면(온보딩·입력 폼·생성·실행)은 `(tabs)` **밖의 별도 스택 그룹**에 배치해 탭바가 구조적으로 숨겨지게 한다(US-E2-04 — `screenOptions` 산발 제어 대신 파일 구조로 강제).
- **대안 비교(1줄)**: 단일 스택에 탭을 화면으로 두는 방식은 상태 보존·탭바 노출 규칙을 코드로 일일이 관리해야 해 파편화 위험(unit-of-work.md U2 리스크 "내비게이션 규칙 파편화").
- **선정 사유**: 탭바 단일 공용 컴포넌트(`shared/ui`) + 린트 규칙으로 규칙을 강제하라는 U2 리스크 완화(unit-of-work.md)와 파일 구조 규약이 정확히 합치.

## 4. 딥링크·URL 스킴 (U2-C3)

- **선택**: Expo Router `linking`(app scheme `trippilot://` + Universal Links/App Links). 딥링크 → **대상 화면이 속한 탭 활성화 + 그 탭 스택에 푸시**(G7)를 라우터 규약으로 처리. 비로그인 딥링크 진입은 초대·공유 딥링크에 한정(D22) — 인증 가드 라우트(로그인 리다이렉트)를 레이아웃 레벨에서 적용. 강제 업데이트 화면의 스토어 이동(US-E2-06)은 아웃바운드 링크(`Linking.openURL`)로 앱 스토어/플레이 스토어 URL 오픈(신규 외부 연동 아님).
- **대안 비교(1줄)**: 수동 딥링크 파서 + 라우팅 디스패치는 경로 증가 시 매핑 드리프트 — Expo Router의 경로=URL 규약이 이를 제거.
- **선정 사유**: G7의 "대상 탭 활성화 + 스택 푸시"가 파일 경로 규약으로 자동 충족되어 후행 유닛 화면 추가 시 딥링크 자동 커버.

## 5. 상태 관리 — Zustand + TanStack Query (U2-C4, U1 상속)

- **선택**: U1 C-4를 상속 — **서버 상태(부트스트랩 결과·홈 카드)는 TanStack Query v5**(병렬 쿼리·부분 실패 격리·stale-while-revalidate가 NFR-U2-PERF-06·07·CL-02의 구현 수단), **클라이언트 상태(전역 세션 플래그·앱 버전·탭 UI·온보딩 진행)는 Zustand**. 홈 카드는 카드별 독립 쿼리로 두어 한 쿼리 실패가 타 카드를 막지 않게 한다(NFR-U2-PERF-07).
- **대안 비교(1줄)**: 홈 카드를 단일 집약 쿼리로 두면 부분 실패 격리가 어려움 — 카드별 쿼리 + 서버 부분 응답(BFF)의 조합이 침묵 실패 금지에 부합.
- **선정 사유**: U1이 확정한 "서버 상태=Query, UI 상태=경량 스토어" 분리가 스플래시 백그라운드 재검증(G5)·홈 부분 로딩 패턴과 그대로 합치 — U2 신규 라이브러리 도입 불요(과설계 금지).

## 6. 스플래시·버전 체크 (U2-C5·C6)

- **스플래시 — expo-splash-screen**: `preventAutoHideAsync()`로 네이티브 스플래시를 유지하다가 **부트스트랩 왕복 + 분기 결정 완료 후 `hideAsync()`**. 3초 타임아웃(G5) 발동 시에도 폴백 목적지로 전환 후 hide(무한 스플래시 금지). 대안(자체 오버레이 스플래시)은 네이티브 스플래시와 이중 깜빡임 — 비채택.
- **버전 체크 — expo-constants**: 현재 앱 버전(`Constants.expoConfig.version` / `nativeAppVersion`)을 부트스트랩 응답의 `minSupportedVersion`과 semver 비교(N4). **비교·게이트 판정은 클라이언트 순수 함수**(PBT 대상, nfr-requirements.md §5). 
  - **expo-updates 관계 정리**: expo-updates(OTA)는 JS 번들 업데이트로 **네이티브 강제 업데이트(N4)를 대체하지 않는다** — N4는 스토어 네이티브 빌드 교체가 필요한 경우(SDK·네이티브 모듈 변경)의 게이트다. OTA 정책 자체는 Operations 소관(deployment-architecture.md §7). U2는 **서버 `minSupportedVersion` 기반 스토어 이동 게이트**만 소유하고, OTA 채널 운용은 참조만.
- **선정 사유**: expo-splash-screen·expo-constants는 Expo 공식 모듈로 prebuild 무마찰(U1 C-1 Expo SDK 54 정합), 신규 네이티브 모듈 리스크 0.

## 7. 서버측 — U1 app 모듈 재사용 (U2-S1, 신규 0)

- **부트스트랩 API**(`server/app` 소유, unit-of-work.md U2): 세션 유효성·최소 지원 버전·재동의 플래그를 **1왕복 집계 응답**으로 반환(nfr-requirements.md §1.1). 세션·재동의 판정은 **U1 M1·동의 도메인 재조회**, 최소 버전은 앱 구성 값 조회(infrastructure-design.md §2) — **신규 서버 프레임워크·모듈 없음**. Spring Security 필터 체인(U1 tech-stack §2.3)·무상태 검증(PAT-PERF-01)·deny-by-default(NFR-U1-SEC-16, 최소 버전 부분만 공개 화이트리스트)를 상속.
- **홈 대시보드 집약 조회**(BFF성, `server/app`): 가용 카드만 담은 부분 응답(nfr-requirements.md §1.3). U2 시점 카드 슬롯은 대부분 후행 유닛이 채우므로 응답 스키마(슬롯 계약)만 U2가 고정하고 실제 데이터는 U3~U8이 공급(unit-of-work.md U2 DoD 계약 포인트).
- **DB**: 앱 구성(최소 지원 버전 등 remote config성 설정) 테이블 1개 내외(unit-of-work.md U2) — 마이그레이션은 U1 Flyway 파이프라인 재사용(신규 도구 0).
- **선정 사유(1줄)**: U2 서버 로직은 얕고(unit-of-work.md U2) 전부 U1 스택으로 충족 — 신규 서버 결정을 만들지 않는 것이 규모 정합(G142)·재작업 방지의 정답.

---

## 8. 상속·비결정 재확인

| 항목 | U2 처리 | 정본 위치 |
|---|---|---|
| Expo SDK·TypeScript strict·prebuild | U1 C-1 상속 | U1 tech-stack §3.1 |
| 토큰 저장(expo-secure-store) | U1 C-3 상속 — U2는 기동 시 1회 읽어 세션 스토어에 로드(§1.4) | U1 tech-stack §3.3, NFR-U1-SEC-28 |
| 폼·스키마(RHF+Zod) | U1 C-5 상속 — U2 입력 폼 없음(온램프 셸은 U3에서 완성) | U1 tech-stack §3.5 |
| 클라우드·CI/CD·배포 | GD-1~7 상속 | u1-foundation-nfr-design-plan.md §2 |
| 서버 프레임워크·시크릿·로그 스택 | SI·U1 정본 상속(재정의 금지) | SI, U1 tech-stack §2 |

**U2 신규 벤더·인프라 결정 0 — 클라이언트 셸 라이브러리 구체화(U2-C1~C6)만 유닛 증분이며, 그마저 U1 확정(Expo Router)·Expo 공식 모듈 범위 내다(과설계 금지).**
