# Unit of Work 계획

> **2026-07-04** · 사용자의 연속 진행 지시에 따라 질문 게이트를 기본값 채택 + 산출물 묶음 검토로 통합 (application-design-plan.md와 동일 방식). 아래 결정은 최종 검토에서 변경 요청 가능.

## 실행 체크리스트
- [x] 분해 결정 확정 (아래 UW-1~4)
- [x] unit-of-work.md 생성 (유닛 정의 + 코드 조직 전략)
- [x] unit-of-work-dependency.md 생성 (의존 매트릭스·순서)
- [x] unit-of-work-story-map.md 생성 (128 스토리 전수 매핑)
- [x] 유닛 경계·의존 검증, 전 스토리 배정 확인
- [ ] 승인 (Application Design과 묶음 검토)

## 분해 결정 (질문 대체 — 기본값 채택)

### UW-1. 그룹핑 전략
**여정 에픽 = 유닛** 기본 정렬 (에픽이 이미 모듈 경계·의존 순서와 정합). 예외: E6+E7을 U6으로 통합(여행 중 실행과 Plan-B는 상태 머신·위치 신호를 공유 — 분리 시 유닛 간 왕복 과다), E1에 프로젝트 스캐폴드·공통 인프라 포함(첫 유닛이 기반 부담).

### UW-2. 배포 모델
단일 배포 서버(모듈러 모놀리스, D04) + 모바일 앱 1개. 유닛은 배포 단위가 아니라 **개발 순서 단위** — 유닛 완료마다 서버·앱에 기능이 누적된다.

### UW-3. 코드 조직 (그린필드)
워크스페이스 루트에 애플리케이션 코드 배치 (aidlc-docs/ 밖 — CLAUDE.md 규칙):
```text
apps/mobile/                  # React Native Expo (TypeScript)
  features/... shared/...     # AD-4 구조
server/                       # Spring Boot Kotlin, Gradle 멀티모듈
  app/                        # 부트 진입점·모듈 조립·전역 설정(보안 헤더·에러 핸들러)
  common/{core,llm-gateway,solver,moderation}
  modules/{auth,profile,stay-search,saved-stay,affiliate,trip,place-data,
           itinerary,planb-detection,recalculation,weather,archive,
           reflection,notification,execution}
docs/                         # 기존 PRD
aidlc-docs/                   # 문서 (기존)
```

### UW-4. 팀·시퀀스
단독(AI-DLC) 순차 개발 전제 — 유닛 시퀀스는 의존 순서 그대로 U1→U8. 후속 게이트 3종(U9~U11)은 유닛 정의만 예약, 1차 컨스트럭션 루프에서 제외.
