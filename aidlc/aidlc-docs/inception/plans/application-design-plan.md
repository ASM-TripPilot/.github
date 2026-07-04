# Application Design 계획

> **2026-07-04** · 사용자의 연속 진행 지시("남은거 진행해")에 따라, 질문 게이트를 별도 파일 왕복 없이 **기본값 채택 + 산출물 검토 시 이의 제기** 방식으로 통합한다. 아래 설계 결정은 requirements.md 기확정 사항의 전개이거나 표준 기본값이며, 최종 검토에서 변경 요청 가능하다.

## 실행 체크리스트
- [x] 컨텍스트 분석 (requirements.md D01~D38 + stories.md 128 스토리)
- [x] 설계 결정 확정 (아래)
- [x] components.md 생성
- [x] component-methods.md 생성
- [x] services.md 생성
- [x] component-dependency.md 생성
- [x] application-design.md (통합본) 생성
- [ ] 설계 완결성 검증·승인 (Units Generation과 묶음 검토)

## 설계 결정 (질문 대체 — 기본값 채택)

### AD-1. 컴포넌트 경계
PRD 13의 모듈 17개 + M18 Trip Execution(Δ7)을 그대로 컴포넌트 경계로 채택. 공통(횡단) 컴포넌트 3개 추가: **C1 LLM Gateway**(D11·D31 — 컨텍스트 필터·rate-limit·로깅 일원화), **C2 Solver Engine**(M8·M10 공용 최적화·검증 라이브러리, ADR-0008), **C3 Content Moderation**(금칙어 검증 — 1차엔 닉네임·여행 제목, 후속 커뮤니티 확장).
근거: PRD가 이미 검증한 책임 경계 + 결정 레지스터와 1:1 추적 유지.

### AD-2. 모듈 내부 구조 (백엔드)
각 모듈은 레이어드 구조 통일: `api`(REST 컨트롤러) / `application`(유스케이스 오케스트레이션) / `domain`(엔티티·비즈니스 규칙·상태 머신) / `infra`(영속성·외부 API 어댑터). 외부 API(지도·날씨·OTA·LLM·FCM)는 **어댑터 계약 인터페이스** 뒤에 격리 — D37 테스트 계층 분리·ADR-0009 벤더 비종속의 구조적 구현.

### AD-3. 모듈 간 통신
- 동기: 모듈 공개 인터페이스(퍼사드)를 통한 인프로세스 호출만 허용 — 타 모듈의 domain·repository 직접 접근 금지 (모듈러 모놀리스 경계 강제, D04)
- 비동기: 도메인 이벤트(인프로세스 이벤트 버스) — 예: `TripEnded`(M18→M13·M14), `ItineraryConfirmed`(M8→M14), `VisitChecked`(M18→M9·M12). 후속 워커 분리 시 이벤트 버스만 외부화
- 순환 의존 금지 — component-dependency.md의 매트릭스로 검증

### AD-4. 클라이언트 구조 (RN Expo)
기능(feature) 단위 모듈 구조 + 공용 계층: `features/{onboarding,home,stay,trip,itinerary,execution,planb,archive,notification,settings}` / `shared/{api,ui,map,location,storage}`. 경량 제약 검증기(D28)는 `shared/validation`에 서버와 규칙 명세 공유.

### AD-5. 메서드 상세 수준
이 단계는 시그니처·입출력 수준까지만 정의(핵심 메서드 위주) — 상세 비즈니스 규칙·전체 API 스펙은 유닛별 Functional Design 소유.
