# U2 앱 셸·홈·내비게이션 — Functional Design 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · 유닛 U2 (E2 스토리 6개 — US-E2-01~06)
> 정본 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U2 섹션, [stories.md](../../inception/user-stories/stories.md) Epic 2, [components.md](../../inception/application-design/components.md) §3.1 여행 상태 머신·§6 클라이언트(RN Expo) 구조, [component-methods.md](../../inception/application-design/component-methods.md) M1.getBootstrapStatus·홈 집약 메서드, [requirements.md](../../inception/requirements/requirements.md) §7.1(G2·G3·G5·G6·G7)·§5.3(N4/C6)·D21·D22/Δ5·D24/Δ6
> 선행 유닛 계약: U1 부트스트랩 공급자측(BR-U1-44·FD-U1-10) — 본 유닛은 그 소비자측(스플래시 분기) 정본이다.

## 질문 처리 방침

**사용자 연속 진행 지시에 따라, 본 스테이지의 확인 질문은 requirements.md의 기확정 결정(D01~D38·Δ1~Δ10·N1~N8·G 제안 기본값 — 문서 승인으로 확정)으로 대체한다.** U2는 클라이언트 중심·경량 유닛으로, 서버 로직이 얕고 신규 영속 엔티티가 약 1종(AppConfig)에 그친다. 스플래시 분기 결정 함수(순수 함수·5목적지·우선순위)와 탭 내비게이션 상태 모델은 본 스테이지에서 확정하며 아래 "설계 결정 목록"에 기록한다.

## 실행 체크리스트

- [x] 정본 6개 입력 로드·대조 (unit-of-work U2 / stories E2 / components §3.1·§6 / component-methods 부트스트랩·홈 집약 / requirements §7.1·§5.3·D21·D22·D24 / U1 부트스트랩 공급자 계약)
- [x] 도메인 데이터 정의 — [domain-entities.md](../u2-appshell/functional-design/domain-entities.md)
  - [x] AppConfig — U2 소유 유일 서버 영속 엔티티(최소 지원 버전 등 remote config성 설정)
  - [x] BootstrapInfo — 읽기 전용 집계 DTO(세션 상태·온보딩 완료·재동의 플래그·최소 지원 버전·forceUpdate, U1에서 집계)
  - [x] TabNavigationState — 클라이언트 상태(탭별 스택·스크롤 위치·보존 수명 G6)
  - [x] HomeDashboardModel — 카드 슬롯 스키마(슬롯별 데이터 출처 유닛·부분 응답·미노출 규칙 표기)
  - [x] TrendingPlace — 인기 장소 집계 결과 참조(실제 소유 M7/U3)
  - [x] "U2 소유 영속 엔티티 = AppConfig 1종" 명시·사유
- [x] 비즈니스 규칙 카탈로그 — [business-rules.md](../u2-appshell/functional-design/business-rules.md) (BR-U2-01~14, 조건/동작/근거)
  - [x] 스플래시 5분기 판정(BR-U2-01) + 우선순위(BR-U2-02) + 타임아웃 폴백(BR-U2-03)
  - [x] 강제 업데이트 게이트(BR-U2-04, N4)
  - [x] 탭 상태 보존·재탭 리셋(BR-U2-05, G6) + 딥링크 내비(BR-U2-06, G7) + 비로그인 가드(BR-U2-07, D22)
  - [x] 탭바 숨김(BR-U2-08) · 홈 카드 조립·진행률·인기 장소(BR-U2-09~12) · 빈 상태(BR-U2-13) · 오프라인 미보장(BR-U2-14)
- [x] 프런트엔드 컴포넌트 설계 — [frontend-components.md](../u2-appshell/functional-design/frontend-components.md)
  - [x] 컴포넌트 계층(SplashScreen·AppShell·TabBar·HomeDashboard·ForceUpdateScreen·5탭 루트 자리)
  - [x] 컴포넌트별 props/state·상호작용·data-testid 규칙(appshell-{screen}-{role})·사용 백엔드 능력
  - [x] Process Flows(스플래시 부트스트랩·탭 전환·딥링크 진입)
  - [x] Testable Properties(PBT-U2) — 스플래시 분기 결정표·탭 라운드트립·버전 단조성·홈 조립 멱등성 + 속성 없는 부분 사유
- [x] 추적 ID(US/D/Δ/G/N/BR) 일관 표기 검수
- [x] 확장 규칙(RESILIENCY-10·SECURITY-08) 관련 조항 반영 검수 — 기술 중립 유지(상태 관리 라이브러리·내비게이션 구현체는 NFR/Code Generation 이관)

## 설계 결정 목록 (본 스테이지 확정분)

| # | 결정 | 내용 | 근거 |
|---|---|---|---|
| FD-U2-01 | 스플래시 분기 결정 함수 정본 | `resolveSplashDestination`을 순수 함수로 확정 — 목적지 5종 enum {FORCE_UPDATE, LOGIN, RECONSENT, ONBOARDING, HOME}, 우선순위 **강제 업데이트 > 재동의 > 세션/온보딩** 고정. U1 공급자 계약(FD-U1-10)의 소비자측 정본 | US-E2-01, US-E2-06, N3, N4, G5, U1 BR-U1-44 |
| FD-U2-02 | 온보딩 하위 목적지 해소 | 스플래시 결정 함수는 5목적지까지만 판정(완전성·상호배타 유지). ONBOARDING 진입 후 정확한 재개 단계(약관 미동의 vs 닉네임 미통과)는 `M2.getOnboardingState.nextStep`로 2차 해소 — 분기 함수 순수성 보존 | US-E2-01, component-methods M2.getOnboardingState, U1 BR-U1-26 |
| FD-U2-03 | 타임아웃 페일오픈 폴백 | 부트스트랩 3초 초과 시 버전 게이트 **페일오픈** + 로컬 토큰 폴백(미만료→HOME+백그라운드 재검증 플래그, 만료→LOGIN). 확인 불가 시 페일오픈은 강제 업데이트 오구성의 전면 차단 리스크 완화 | G5, US-E2-06, unit-of-work U2 리스크 |
| FD-U2-04 | 탭 상태 보존 범위 | TabNavigationState는 **세션 메모리 한정**(앱 재시작 시 초기화), 탭별 독립 스택·스크롤 위치. 재탭 시 활성 탭만 스크롤 탑. 오프라인 일정 캐시 미제공(D24)과 정합 — 조회는 온라인 전제 | G6, D24/Δ6 |
| FD-U2-05 | 홈 카드 슬롯 스키마·부분 응답 | HomeDashboardModel은 **가용 슬롯만 담는 부분 응답 계약**(침묵 실패 금지). 후행 유닛이 채울 슬롯(인기 장소=U3, 여행 카드=U4, 활성 일정=U6, 추억=U7)의 필드·데이터 출처·빈 상태를 U2에서 스키마로 고정 | US-E2-02, unit-of-work U2 DoD 계약 포인트 |
| FD-U2-06 | 활성 여행 단수 전제 | D21로 진행 중 여행 최대 1개 — 홈 활성 카드·일정 탭이 M18 허브로 수렴, 복수 활성 여행 전환 UI 없음 | D21/Δ3, US-E2-02, US-E2-03, components §3.1 |
| FD-U2-07 | 커뮤니티 카드 미노출 게이트 | '지금 뜨는·내 취향 여행 기록'(커뮤니티) 카드는 U10 출시 전까지 슬롯 자체 미노출 | US-E2-02, unit-of-work U2 §제외 |
| FD-U2-08 | U2 소유 서버 영속 엔티티 = AppConfig 1종 | 최소 지원 버전·스토어 링크 등 remote config성 설정 1종만 서버 영속. 그 외(BootstrapInfo·HomeDashboardModel·TrendingPlace)는 읽기 전용 집계 DTO, TabNavigationState는 클라이언트 세션 상태 | unit-of-work U2 산출물(신규 엔티티 약 1) |
| FD-U2-09 | 진행률 표시 규칙 소비 | 홈 여행 카드 진행률은 G3 — 여행 중=방문 체크 완료 비율(`M18.getTripProgress`), 여행 전=진행률 없이 D-day만. 계산 정본은 M18/U6, U2는 **표시 규칙**만 소유 | G3, US-E2-02, component-methods M18.getTripProgress |
| FD-U2-10 | 딥링크 라우팅 가드 | 비로그인 진입은 초대·공유 딥링크 수신에 한정(D22/Δ5), 진입 즉시 로그인 게이트. 인증 후 대상 화면이 속한 탭 활성화 + 그 탭 스택에 푸시(G7) | D22/Δ5, G7, US-E2-03 |

## 산출물

| 파일 | 내용 |
|---|---|
| `u2-appshell/functional-design/domain-entities.md` | AppConfig(유일 영속)·BootstrapInfo·TabNavigationState·HomeDashboardModel·TrendingPlace 정의·속성 표·불변식 |
| `u2-appshell/functional-design/business-rules.md` | BR-U2-01~14 규칙 카탈로그 |
| `u2-appshell/functional-design/frontend-components.md` | 컴포넌트 계층·서버 능력 매핑 + Process Flows 3종 + Testable Properties(PBT-U2) |
