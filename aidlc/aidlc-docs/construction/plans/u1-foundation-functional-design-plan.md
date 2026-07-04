# U1 기반·계정·온보딩 — Functional Design 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · 유닛 U1 (E1 스토리 18개 — US-E1-01~18)
> 정본 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U1 섹션, [stories.md](../../inception/user-stories/stories.md) Epic 1, [components.md](../../inception/application-design/components.md) M1·M2·C3·상태 머신 정본, [component-methods.md](../../inception/application-design/component-methods.md) M1·M2·C3, [requirements.md](../../inception/requirements/requirements.md) §4·§5.3·§6.4·§6.6·§7.1

## 질문 처리 방침

**사용자 연속 진행 지시에 따라, 본 스테이지의 확인 질문은 requirements.md의 기확정 결정(D01~D38·Δ1~Δ10·N1~N8·G 제안 기본값 — 문서 승인으로 확정)으로 대체한다.** requirements.md §9가 Functional Design으로 이관한 결정(위치 동의 3층 조합 매트릭스 G182, PBT 속성 식별 PBT-01, 계정 상태 머신 상세 전이도)은 본 스테이지에서 확정하며, 아래 "설계 결정 목록"에 기록한다.

## 실행 체크리스트

- [x] 정본 5개 문서 로드·대조 (unit-of-work U1 / stories E1 / components M1·M2·C3 / component-methods M1·M2·C3 / requirements §4·§5.3·§6.4·§6.6·§7.1)
- [x] 도메인 엔티티 정의 — [domain-entities.md](../u1-foundation/functional-design/domain-entities.md)
  - [x] Account 상태 머신 상세 전이도 (PENDING_VERIFICATION→ACTIVE→DELETION_PENDING→DELETED, D18)
  - [x] SocialIdentity 서브엔티티 (provider+sub 복합 유니크, G20)
  - [x] EmailVerification (24h·재발송 제한, G22)
  - [x] ConsentRecord append-only 증적 모델 (N2·N3) + TermsVersion(재동의 플래그)
  - [x] MarketingConsent (N8)
  - [x] LocationConsent 3층 모델 + G182 조합 매트릭스 8종 명문화 + 위치정보 법정 로그 (N2/D34)
  - [x] RefreshSession 회전 체인 (D36)
  - [x] Profile·PreferenceSet 7축 (값 도메인 = PRD 03 선택지, 미설정 NULL vs 중립 기본값 구분)
  - [x] BannedWordDictionary (C3, P8)
  - [x] 엔티티별 속성 표(타입·제약·근거 ID)·관계·불변식
- [x] 비즈니스 규칙 카탈로그 — [business-rules.md](../u1-foundation/functional-design/business-rules.md) (BR-U1-01~44, 조건/동작/위반 시 처리/근거)
- [x] 비즈니스 로직 모델 — [business-logic-model.md](../u1-foundation/functional-design/business-logic-model.md)
  - [x] 핵심 프로세스 플로우 7종 (소셜 통합 / 이메일 가입·인증 / 약관·재동의 / 취향 위저드 / 토큰 회전·탈취 감지 / 삭제 라이프사이클 / 닉네임)
  - [x] Testable Properties 섹션 (PBT-01) — 컴포넌트별 속성 식별 표, 속성 없는 컴포넌트 사유 명기
- [x] 프런트엔드 컴포넌트 설계 — [frontend-components.md](../u1-foundation/functional-design/frontend-components.md) (온보딩 화면 플로우·컴포넌트 계층·M1/M2 메서드 매핑·위치 프리프롬프트 프레임)
- [x] 추적 ID(US/D/Δ/G/N/BR) 일관 표기 검수
- [x] 확장 규칙(SECURITY·PBT) 관련 조항 반영 검수 — 기술 중립 유지(알고리즘·인프라 명명은 NFR/Infra 단계 이관)

## 설계 결정 목록 (본 스테이지 확정분)

| # | 결정 | 내용 | 근거 |
|---|---|---|---|
| FD-U1-01 | 계정 상태 명칭 | `PENDING_VERIFICATION`을 정본 명칭으로 확정 (components.md의 `UNVERIFIED`와 동일 상태 — 별칭 주석 유지). 소셜 가입은 이 상태를 거치지 않고 즉시 `ACTIVE` | components.md §4.M1, G22 |
| FD-U1-02 | 동의 증적 구조 | ConsentRecord를 GRANT/REVOKE 이벤트 행의 append-only 시퀀스로 확정 — components.md의 `revokedAt?`은 파생 뷰로 재해석 (불변 저장 N2·N3와 정합) | N2, N3, SECURITY-14 |
| FD-U1-03 | 위치 동의 3층 매트릭스 | 8조합 전수 매트릭스 명문화 (§9 이관 결정 이행). 서버 위치 전송·처리 = L1∧L2, GPS 발자취 보존 = L1∧L2∧L3. L1(OS 권한)의 정본은 단말, 서버는 미러 | G182, N2/D34 |
| FD-U1-04 | 브루트포스 수치 제안 | 연속 실패 5회부터 점진 지연(2ⁿ⁻⁵초, 상한 60초, 15분 슬라이딩 윈도), 20회 도달 시 30분 잠금 — 수치는 운영 조정 가능 값으로 제안 | SECURITY-12, §6.4 |
| FD-U1-05 | 닉네임 재추첨 수렴 | 재추첨 최대 10회, 소진 시 숫자 자릿수 확장(2→3자리) 폴백 — 유한 시도 내 유니크 수렴 보장 | G23 |
| FD-U1-06 | 페이스 중립 기본값 | 미설정 페이스의 내부 밀도 파라미터 = '균형있게' 상당(하루 3~4곳) — 저장은 NULL 유지, 파생 시점 주입 | US-E1-14·15 |
| FD-U1-07 | GPS 파기 시점 | 계정 삭제 **요청 시점**에 GPS 발자취 즉시 파기(유예 복구 시에도 미복원 — 법정 요구 우선). 법정 로그는 보존 | D34, N2, component-methods M1.requestAccountDeletion |
| FD-U1-08 | 재발송 시 구 토큰 처리 | 인증 메일 재발송 시 기존 미소비 토큰 즉시 무효화(유효 토큰 항상 최대 1개) | G22 (설계 보강) |
| FD-U1-09 | 온보딩 완료와 취향의 독립 | 완료 판정 함수는 (약관 동의 상태 × 닉네임 통과) 2입력 순수 함수 — 취향 상태는 판정에 미참여, `completeOnboarding` 반복 호출 멱등 | G24/G157 |
| FD-U1-10 | 부트스트랩 판정 우선순위 | 강제 업데이트(N4) > 재동의(N3) > 세션(G5) — U2 스플래시 분기 계약의 공급자측 정본으로 고정 | N3, N4, G5, U1 DoD 계약 포인트 |

## 산출물

| 파일 | 내용 |
|---|---|
| `u1-foundation/functional-design/domain-entities.md` | 엔티티 11종 정의·속성 표·관계·불변식 |
| `u1-foundation/functional-design/business-rules.md` | BR-U1-01~44 규칙 카탈로그 |
| `u1-foundation/functional-design/business-logic-model.md` | 프로세스 플로우 7종 + Testable Properties(PBT-01) |
| `u1-foundation/functional-design/frontend-components.md` | 온보딩 화면 플로우·컴포넌트 계층·서버 능력 매핑 |
