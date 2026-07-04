# U1 기반·계정·온보딩 — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 7종 + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U1-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) M1·M2·C3
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 전송·저장 기술은 NFR/Infra 단계 소유.

---

## FLOW-1 소셜 가입/로그인 통합 플로우 (신규/기존/이메일 충돌 3분기)

**진입**: 로그인/회원가입 화면의 소셜 버튼 4종(Google·Apple·카카오·네이버) → 제공자 인증 → `M1.signUpWithSocial(provider, providerToken, ageConfirmation)`
**관련**: US-E1-01, US-E1-16 · BR-U1-02~07 · D22, N1/D33, G20, C4, D36

```text
S1 제공자 인증 결과 수신
   ├─ E1 사용자 취소/동의 거부 → "로그인이 취소되었습니다" 안내 + 로그인 화면 복귀 (계정 미생성)
   └─ E2 제공자 오류(타임아웃·5xx) → ProviderAuthFailed(retryable) → "잠시 후 다시 시도" + 재시도 버튼 (계정 미생성 보장)
S2 제공자 토큰 서명·수신자 검증 (서버) — 실패 시 E2와 동일 처리(비신뢰 토큰 거부)
S3 (provider, sub)로 SocialIdentity 조회 [BR-U1-02]
   ├─ B1 존재 → 【기존 로그인 분기】 S4로
   └─ 부재 → S5로
S4 【기존 로그인】 계정 상태 확인
   ├─ ACTIVE → 토큰 발급(기기 체인 신설, BR-U1-33·35) → nextStep 판정: 재동의 blocking 있으면 consent,
   │           온보딩 미완료면 onboarding(잔여 단계), 완료면 home [BR-U1-23·26]
   ├─ DELETION_PENDING → 복구 안내 경로 (FLOW-6 S5) [BR-U1-39]
   └─ (DELETED는 SocialIdentity가 함께 파기되므로 이 분기 미도달 — 신규 분기로 진행)
S5 【신규 후보】 연령 확인 게이트 [BR-U1-05]
   └─ E3 만 14세 미만 → AgeRestricted, 사유 안내, 계정 미생성 (INV-A2)
S6 유예 중 재가입 차단 확인 — 동일 식별자(sub·이메일)가 DELETION_PENDING 귀속이면 복구 안내 [BR-U1-06]
S7 이메일 충돌 대조 [BR-U1-03·04]
   ├─ B2 제공자 이메일이 기존 계정과 일치(대조 가능) → 【이메일 충돌 분기】 계정 미생성,
   │      "이미 {기존 수단}으로 가입된 이메일입니다" — 기존 수단 로그인 유도 (수동 연결 미제공, CS 안내 [G20])
   ├─ B3 이메일 미제공/대조 불가(Apple 비공개 릴레이 등) → 별도 계정 생성 허용 [BR-U1-04] → S8
   └─ 충돌 없음 → S8
S8 【신규 생성 분기】 Account 생성(status=ACTIVE — 소셜은 인증 단계 생략) + SocialIdentity 연결(INV-S1)
S9 토큰 발급 → AuthResult{isNewAccount=true, nextStep=consent} → 약관 동의 화면(FLOW-3)으로
```

**사후 조건**: 어떤 경로로도 (a) 동일 (provider, sub) 이중 계정 없음, (b) 대조 가능한 동일 이메일 이중 계정 없음, (c) 연령 미확인 계정 없음 — 하드 제약 '계정 무결성' [D37].

---

## FLOW-2 이메일 가입 → 인증 → 온보딩 진입

**진입**: '회원가입'(수단 비종속 라벨) → 이메일 폼 → `M1.signUpWithEmail(email, password, ageConfirmation)`
**관련**: US-E1-01, US-E1-16 · BR-U1-01, 05, 08~09, 12~15 · G22

```text
S1 입력 검증 — 이메일 형식, 비밀번호 정책(8자+영문/숫자+유출 목록) [BR-U1-08]
   └─ E1 WeakPassword(reasons) → 미충족 조건 인라인 표시
S2 연령 확인 게이트 [BR-U1-05] — E2 미만 차단(계정 미생성)
S3 중복 이메일 확인 [BR-U1-01]
   └─ E3 EmailAlreadyRegistered → 계정 미생성, 로그인/비밀번호 재설정 유도(재설정 링크 발송 경로 제공)
S4 유예 중 재가입 확인 [BR-U1-06] — 해당 시 복구 안내
S5 Account 생성(status=PENDING_VERIFICATION, passwordHash 저장 [BR-U1-09])
S6 EmailVerification 발급(expiresAt=+24h, INV-E2 단일 유효) + 인증 메일 발송
   └─ E4 메일 발송 실패 → 미인증 상태 유지 + 재발송 동작 제공, 온보딩 진행 차단(침묵 실패 금지)
S7 인증 안내 화면 대기 — 재발송 버튼 [BR-U1-13: 분당 1회·일 5회, 재발송 시 구 토큰 무효화(FD-U1-08)]
   ├─ E5 RateLimited(retryAfter) → 카운트다운 표시
   └─ E6 7일 경과 미인증 → 정리 배치가 계정·토큰 파기 [BR-U1-15] (재가입 즉시 허용)
S8 사용자 링크 클릭 → M1.verifyEmail(token)
   ├─ E7 TokenExpired → 재발송 경로 안내
   ├─ E8 기소비 토큰 → "이미 인증되었습니다" (멱등 UX, INV-E1)
   └─ 성공 → PENDING_VERIFICATION→ACTIVE(verifiedAt 기록) + 자동 로그인(토큰 발급) [BR-U1-12]
S9 nextStep=consent → 약관 동의(FLOW-3) → 닉네임(FLOW-7) → 취향(FLOW-4)
```

**가드**: S8 성공 전에는 약관 동의 이후 온보딩 단계 진행·정식 토큰 체인 발급 불가 [BR-U1-14, INV-A6] — 하드 제약 '미인증 온보딩 진행 차단' [D37].

---

## FLOW-3 약관 동의·재동의 플로우

**진입**: (최초) 최초 로그인 직후 온보딩 진입 전 1회 / (재동의) blocking 판정 시 — 스플래시 분기 연결은 U2
**관련**: US-E1-02, US-E1-17, US-E1-18 · BR-U1-21~25 · N2/D34, N3, N8

```text
【최초 동의】
S1 약관 동의 화면 구성 — 필수 3종 분리 체크(이용약관·개인정보 처리방침·위치기반서비스 이용약관 [N2])
   + 선택 2종(마케팅 수신 [N8], GPS 여행 기록 옵트인 [N2]) + 전체 동의 토글 + 항목별 본문 열람
S2 클라이언트 게이트 — 필수 3종 전부 체크 ⇔ '다음' 활성화, 미동의 항목 안내 [BR-U1-21]
S3 M1.recordConsent(ConsentBundle{termsVersion, privacyVersion, locationTermsVersion, marketingOptIn, gpsRecordingOptIn?})
   ├─ E1 서버 검증: 필수 3종 미완비 → ValidationFailed (클라이언트 게이트 불신뢰 — SECURITY-05)
   └─ 성공 → ConsentRecord GRANT 행 append(항목·버전·일시·채널, INV-C1·C2) + MarketingConsent·LocationConsentState 파생 갱신
S4 ConsentReceipt 반환 → 닉네임 단계(FLOW-7)로 진행

【재동의 (약관 개정, N3)】
S5 개정 발행 — 새 TermsVersion(reconsentRequired: 중대=true/경미=false) [INV-T1: 기존 버전 불변]
S6 판정 — M1.getRequiredConsents: 항목별 최신 GRANT 버전 vs 현행 버전 비교 [BR-U1-23]
   ├─ B1 blocking 존재(중대 변경) → 재동의 게이트: 재동의 완료 전 일반 퍼사드 접근 제한 [BR-U1-24]
   │      → 재동의 화면(변경 요약·전문 열람) → M1.reconsent → GRANT 증적 append → 게이트 해제
   ├─ B2 nonBlocking만(경미 변경) → 인앱 공지 처리, 진입 무차단
   └─ B3 없음 → 통과
S7 재동의 거부 지속 → 서비스 진입 불가 상태 유지(강제 없음 — 사용자 선택 존중, 진입만 제한)
```

**사후 조건**: 온보딩 완료 계정은 항상 필수 3종 현행 계보 GRANT 보유(INV-C3). 증적은 append-only(INV-C1) — 어떤 재동의·철회도 과거 행을 변경하지 않는다.

---

## FLOW-4 온보딩 취향 수집 (건너뛰기·일괄 탈출·재개)

**진입**: 닉네임 통과 직후 — 취향 7단계 위저드(스타일→예산→동행→활동→이동→음식→페이스)
**관련**: US-E1-05~11, US-E1-15 · BR-U1-26~32 · G24/G157, G19, G26, D26/Δ2

```text
S1 위저드 진입 — M2.getOnboardingState로 진행 위치 조회(재진입 시 남은 단계부터 재개)
S2 각 단계 공통 동작
   ├─ 선택 저장 → M2.updatePreferences(patch — 해당 축만) [BR-U1-29 값 도메인 검증, BR-U1-31 즉시 반영]
   ├─ B1 '건너뛰기' → 해당 축 NULL(미설정) 유지 — 중립 기본값을 저장하지 않음 [BR-U1-27, INV-PR2]
   ├─ B2 '이전' → 직전 단계로 — 이전 응답 수정 후 재진행 가능(진행률 표시 유지)
   └─ B3 '나중에 설정하고 시작'(일괄 탈출구) → 잔여 취향 단계 전부 건너뛰기 → S4
   └─ E1 ValidationFailed(도메인 밖 값) → 축 단위 인라인 오류, 저장 안 함
S3 7단계 완주 → S4
S4 완료 처리 — M2.completeOnboarding [BR-U1-26: 판정=약관+닉네임, 취향 무관·멱등]
   → onboardingCompletedAt 기록 → 홈 대시보드 즉시 진입(빈 홈 첫 행동 유도 — U2 소관)
S5 (사후) 미설정 축 점진 회수 — M2.getPendingPreferencePrompts가 한두 개씩 카드 데이터 공급 [BR-U1-28]
   (노출 지점: 홈 대시보드=U2, 첫 여행 생성 직전=U4)
S6 (사후) 설정 화면 상시 수정 — 온보딩과 동일 선택지·동일 검증, 미설정 축 신규 설정 가능 [US-E1-12]
```

**중단·재개 규칙**: 취향 단계 중 앱 이탈은 실패가 아니다 — 약관+닉네임을 통과했다면 완료로 처리하고(FD-U1-09), 미완료(약관 또는 닉네임 미통과)면 부트스트랩 onboardingComplete=false → 해당 필수 단계로 재개 [BR-U1-26·44].

---

## FLOW-5 토큰 갱신·회전·탈취 감지

**진입**: 액세스 토큰 만료(1시간) 시 클라이언트 자동 갱신 — `M1.refreshTokens(refreshToken)`
**관련**: US-E1-01 (+U2 US-E2-01 세션 기반) · BR-U1-33~36 · D36, SECURITY-03·12

```text
S1 제시된 리프레시 토큰 해시로 RefreshSession 조회
   └─ E1 미존재/만료(expiresAt≤now)/revoked → AuthenticationRequired → 재로그인
S2 회전 상태 판정
   ├─ B1 정상(rotatedAt=NULL, 체인 현행 토큰) → S3 회전 수행
   └─ B2 재사용 탐지(rotatedAt≠NULL — 이미 회전 소비된 토큰) → 【탈취 신호】
        → 해당 chainId 체인 전체 즉시 무효화(현행 토큰 포함) [BR-U1-34, INV-R2]
        → 보안 이벤트 기록·알림(SECURITY-03) → AuthenticationRequired (해당 기기 재로그인 필요)
        → 타 기기 체인은 유지 [BR-U1-35, INV-R3]
S3 회전 — 구 토큰 rotatedAt 마킹 + 신규 RefreshSession(동일 chainId 연결) + 신규 액세스 토큰(1h) 발급
   → 체인 현행 유일 유지(INV-R1)
S4 TokenPair 반환 — 클라이언트는 OS 보안 저장소에 교체 저장(SECURITY-12), 동시 갱신은 직렬화(shared/api)

【체인 무효화 이벤트 소스】
- 로그아웃(logout): 해당 기기 체인 revoke + FCM 토큰 연결 해제
- 비밀번호 재설정: 전 기기 체인 revoke [BR-U1-10]
- 계정 삭제 요청: 전 기기 체인 revoke [BR-U1-37]
- 재사용 탐지: 해당 체인 revoke [BR-U1-34]
```

**상태 머신 요약(기기 체인 단위)**: `발급됨(현행) → 회전됨(rotatedAt) → …` 체인 연쇄, 종결 이벤트=revoke(4소스) 또는 만료. 임의 이벤트 시퀀스에 대한 불변식은 PBT 속성 U1-P2(stateful)로 검증.

---

## FLOW-6 계정 삭제 라이프사이클 (요청 → 유예 → 복구/만료 연쇄)

**진입**: 설정 > 계정 삭제(화면은 U8 — U1은 API·데이터 모델·배치 골격) — `M1.requestAccountDeletion(confirmation)`
**관련**: US-E09-09(소비) · BR-U1-37~40 · D18, D34, N2, C4

```text
S1 사전 고지 — cascadeSummary 구성·표시 [BR-U1-37]:
   삭제: 숙소(위시리스트·등록)·여행·일정·기록·회고·사진 / (후속) 커뮤니티 게시물 삭제·댓글 '삭제된 사용자' 익명화
   분리 보존: 위치정보 법정 로그(INV-LL1·LL2)·동의 증적(INV-C1)
   즉시 파기: GPS 발자취(복구해도 미복원 — 명시 고지, FD-U1-07)
S2 재확인(DeletionConfirmation) — 미확인 시 전이 불가
S3 전이 실행 — ACTIVE→DELETION_PENDING (INV-A5)
   ├─ deletedAt 기록 + DeletionSchedule{requestedAt, purgeAt=+30일, cascadeSummary 스냅샷} 생성
   ├─ 전 기기 세션 무효화(INV-R4)
   ├─ 즉시 비노출 — 전 조회·기능에서 배제 [BR-U1-38]
   ├─ GPS 발자취 즉시 파기 트리거(M12.purgeLocationData) + 법정 로그 PURGE 기록 [BR-U1-42·43]
   └─ DeletionSchedule 반환(effectiveAt·purgeAt·cascadeSummary)
S4 【유예 기간 30일】
   ├─ B1 유예 중 동일 식별자 로그인 시도 → 일반 로그인 대신 "삭제 예정 계정" 안내 + 복구 확인 [BR-U1-39]
   │     └─ 복구 확인 → M1.cancelAccountDeletion → DELETION_PENDING→ACTIVE, 스케줄 폐기(cancelledAt),
   │        노출 복원(GPS 발자취 제외 — 기파기) → 정상 로그인 재개
   ├─ B2 유예 중 동일 식별자 신규 가입 시도 → 차단 + 복구 경로 안내 [BR-U1-06]
   └─ E1 purgeAt 경과 후 철회 시도 → ResourceNotFound (INV-D2) → 신규 가입 안내
S5 【유예 만료 배치 — 일 1회】 purgeAt 경과분 처리 [BR-U1-40]
   ├─ DELETION_PENDING→DELETED (최종 상태)
   ├─ U1 소관 파기: 계정·소셜 연결·프로필·취향·세션·인증 토큰·마케팅 상태 완전 삭제·익명화
   ├─ 모듈별 파기 훅 호출(U1은 훅 계약 골격 — 전 모듈 연쇄 S6 완성은 U8)
   ├─ 법정 보존 분리: 위치 법정 로그·동의 증적은 파기 제외
   └─ 멱등 처리 — 재실행 안전, 부분 실패 계정 단위 재시도 + 계측(침묵 실패 금지)
S6 DELETED 이후 — 동일 식별자 신규 가입 허용(신규 계정, 과거 데이터 무연결)
```

---

## FLOW-7 닉네임 생성·검증

**진입**: (생성) 계정 생성 직후 온보딩 닉네임 단계 기본값 / (검증) 온보딩·설정의 수정 저장
**관련**: US-E1-03, US-E1-12 · BR-U1-16~20 · G23, C3, P8

```text
【자동 생성 — M2.generateNickname】
S1 후보 생성 — 형용사 어휘풀 × 여행명사 어휘풀 × 2자리 숫자(00~99) [BR-U1-16]
   (어휘풀은 사전 검증분 — 금칙어·길이 위반 원소 없음)
S2 유니크 확인 — 충돌 시 재추첨(최대 10회)
   └─ B1 10회 소진 → 숫자 3자리 확장 폴백 → 재추첨 (유한 수렴 — FD-U1-05)
S3 온보딩 화면 기본값으로 채움 — "닉네임은 나중에 설정에서 바꿀 수 있어요" 명시
   └─ B2 사용자 무수정 '다음' → 자동 생성값 그대로 확정·통과 [BR-U1-20]

【수정 검증 — M2.updateNickname】
S4 검증 체인(순서 고정 — 실패 시 즉시 해당 사유 반환):
   ① 길이 2~20자(공백 정리 후) [BR-U1-17] → Rejected(LENGTH)
   ② 금칙어 — C3.checkText(context=Nickname) [BR-U1-18] → Rejected(FORBIDDEN)
      └─ E1 사전 로드 실패 → fail-closed: 저장 보류 + 재시도 안내 (Pass 처리 금지, INV-B2)
   ③ 중복 — 활성 계정 유니크 [BR-U1-19] → Rejected(DUPLICATE, suggestions: 원탭 적용 가능 대체 추천)
S5 저장 — 유니크 제약이 검증-저장 경합의 최종 방어(경합 패배 → 동일 Rejected 응답)
S6 저장 즉시 반영 [BR-U1-31] — 설정 경유 수정도 동일 규칙(US-E1-12 동일 적용)
```

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장 구성). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check [§2.3]. 시드 로깅·수축(shrinking) 필수(PBT-08). 아래 속성은 U1 Code Generation의 DoD 항목(속성 테스트 존재·통과)이다. U1 DoD가 명시한 4속성(계정 상태 머신·토큰 회전·닉네임 생성기·동의/프로필 직렬화 왕복)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**M1 Auth (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U1-P1 | 상태 머신 불변식 (stateful) | 임의의 이벤트 시퀀스(가입·인증·삭제 요청·철회·배치 틱)에 대해 Account.status는 정본 전이표의 경로만 밟는다: PENDING_VERIFICATION→{ACTIVE, 정리}, ACTIVE→DELETION_PENDING, DELETION_PENDING→{ACTIVE, DELETED}, DELETED=최종(후속 이벤트 전부 무효). 소셜 생성은 PENDING_VERIFICATION 미경유 | 이벤트 열거 시퀀스 생성기(길이 0~50, 유효·무효 이벤트 혼합) + 가입 경로(소셜/이메일) 선택 + 가상 clock(7일·30일 경계 포함 시각 점프) |
| U1-P2 | 상태 머신 불변식 (stateful) | **토큰 회전 상태 머신**: 임의의 (갱신·재사용·로그아웃·재설정·시각 경과) 시퀀스에 대해 (a) 체인당 유효 현행 토큰 ≤1(INV-R1), (b) 회전 소비된 토큰 재사용 시 신규 발급 0 + 해당 체인 전체 무효(INV-R2), (c) 타 기기 체인 불간섭(INV-R3 — 재설정·삭제 이벤트 제외), (d) 만료(90일) 후 갱신 불가 | 다기기(1~5) × 명령 시퀀스 생성기(정상 갱신/구 토큰 재제시/로그아웃/재설정 혼합) + 가상 clock |
| U1-P3 | append-only 불변식 | **동의 증적**: 임의의 GRANT/REVOKE 시퀀스 적용 후 (a) 기존 행 불변·행 수 단조 증가(수정·삭제 연산 부재), (b) 항목별 현재 상태 = 시퀀스 마지막 이벤트의 폴드와 일치(INV-C2), (c) REVOKE는 선행 GRANT 없이 미출현(INV-C4) | 동의 이벤트 시퀀스 생성기(6개 termsType × GRANT/REVOKE × 버전, 길이 0~100) |
| U1-P4 | 순수 함수 판정 | **재동의 판정**: 임의의 (동의 이력 × TermsVersion 집합)에 대해 (a) blocking ⊆ {현행이 reconsentRequired=true이고 그 버전 GRANT가 없는 항목}, (b) 전 항목 현행 동의 시 blocking=∅, (c) 동일 입력→동일 출력(결정성) | 동의 이력 생성기(U1-P3 재사용) + 버전 계보 생성기(중대/경미 개정 혼합) |
| U1-P12 | 멱등성·1회성 | **이메일 인증 토큰**: 임의 발급·소비·재발송·시각 경과 시퀀스에 대해 (a) 소비는 계정당 최대 1회 성공, (b) 만료(24h) 후 소비 성공 0, (c) 재발송 후 구 토큰 소비 성공 0(INV-E2), (d) 유효 토큰 동시 존재 ≤1 | 토큰 조작 시퀀스 생성기 + 가상 clock(24h 경계) |
| U1-P13 | rate-limit 상한 | **재발송 제한**: 임의의 재발송 요청 타임스탬프 시퀀스에 대해 실발송 횟수가 어떤 60초 창에서도 ≤1, 어떤 24시간 롤링 창에서도 ≤5 | 타임스탬프 시퀀스 생성기(버스트·균등 혼합, 0~200회) |
| U1-P14 | 단조성 | **브루트포스 지연**: 연속 실패 횟수 n에 대해 지연(n)은 비감소·상한(60초) 준수, n<5에서 0, 성공 이벤트 후 리셋(지연(성공 후 첫 실패)=0) | 실패/성공 혼합 시퀀스 생성기 |
| U1-P11 | 총함수·논리 게이트 | **위치 동의 매트릭스(G182)**: 8조합 전수에 대해 (a) 판정 함수는 정확히 1개 능력 집합 반환(총함수), (b) serverLocationService=L1∧L2, gpsTrackRetention=L1∧L2∧L3 (INV-L1·L2), (c) 층 추가 시 능력 비축소(INV-L3 단조성) | 불리언 3튜플 전수(exhaustive) + 층 전이 쌍 생성기 |
| U1-P16 | 멱등성·연쇄 정확성 | **GPS 옵트인 철회 연쇄**: 임의의 동의 토글 시퀀스에 대해 (a) 최종 상태 L3=REVOKE면 수집 능력 false, (b) 파기 트리거는 GRANT→REVOKE 하강 에지당 정확히 1회(중복 REVOKE에 멱등), (c) 법정 로그 행 수는 이벤트 수 이상(누락 없음) | L3 토글 시퀀스 생성기(중복 이벤트 포함) |

**M2 User Profile (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U1-P5 | 형식 불변식 + 수렴 | **닉네임 생성기**: 임의의 기존 닉네임 집합(충돌 밀도 0~높음)에 대해 생성 결과는 (a) 항상 '형용사+여행명사+숫자' 패턴·2~20자·금칙어 미포함, (b) 기존 집합과 미충돌(유니크), (c) 유한 시도 내 종료(재추첨 10회+자릿수 확장 폴백 수렴 — FD-U1-05) | 기존 닉네임 집합 생성기(어휘풀 조합 고밀도 점유 케이스 포함) + 어휘풀 축소 케이스 |
| U1-P6 | 검증 함수 동치 | **닉네임 검증**: 임의 문자열에 대해 통과 ⇔ (길이∧금칙어∧중복) 3조건 전부 충족이며, Rejected 사유는 첫 위반 조건과 일치, DUPLICATE 시 suggestions는 전부 3검증 통과분 | 유니코드 문자열 생성기(경계 길이 1·2·20·21자, 공백 패딩, 금칙어 삽입 변형) + 기존 닉네임 집합 |
| U1-P7 | 라운드트립 | **취향 직렬화**: 임의 PreferenceSet(7축, NULL 혼합)에 대해 직렬화→역직렬화 왕복 동일. 특히 NULL(미설정)과 명시값(예: 이동={대중교통} 직접 선택)의 구분 보존(INV-PR2) | 7축 조합 생성기 — 축별 {NULL, 도메인 내 임의값} + 예산 (구간만/원값+구간) 변형 + 동행 (NULL, petFlag=true) 조합 |
| U1-P8 | 파생 함수 정확성·멱등 | **중립 기본값 채움**: 임의 PreferenceSet에 대해 getPreferences 결과는 (a) 전 축 값 존재(INV-PR5 완전성), (b) isNeutralDefault=true ⇔ 저장값 NULL, (c) 명시값 축은 원값 그대로, (d) 채움 결과에 재적용해도 동일(멱등 — 채움이 저장에 역류하지 않음) | U1-P7 생성기 재사용 |
| U1-P9 | 매핑 정합·전역성 | **예산 원값↔구간**: 임의 양수 총액에 대해 매핑 구간은 정확히 1개(전역·중복 없음), 구간 경계에서 단조(금액↑⇒구간 비하강), 원값+구간 동시 저장 후 재조회 시 INV-PR3 정합 유지 | 금액 생성기(0 근방·구간 경계값·극대값 포함 로그 스케일) |
| U1-P10 | 멱등성·독립성 | **온보딩 완료 판정**: (a) 판정 함수는 (동의 상태, 닉네임 상태) 2입력 순수 함수 — 취향 7축 임의 변형에 판정 불변(독립성), (b) completeOnboarding 반복 호출 멱등(onboardingCompletedAt 최초값 유지), (c) 완료 ⇒ INV-C3∧닉네임 존재 | (동의 상태 × 닉네임 상태 × 취향 7축) 직교 조합 생성기 + 반복 호출 횟수 생성기 |

**C3 Content Moderation (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U1-P15 | 결정성 + fail-closed | **금칙어 검증**: (a) 동일 (사전 버전, 입력)에 대해 결과 결정적, (b) 사전 원소를 부분 문자열로 포함하는 입력은 Blocked·미포함 정상 입력은 Pass(매칭 정의 일관성), (c) Blocked 응답에 매칭 원문 미포함, (d) 사전 로드 실패 상태에서 Pass 반환 0(fail-closed — 저장 보류만, INV-B2) | 사전 생성기(임의 어휘 5~500) × 입력 생성기(사전 원소 삽입/비삽입·유니코드 혼합) + 로드 실패 상태 주입 |

**클라이언트 features/onboarding + shared (fast-check)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U1-P17 | 게이트 판정 동치 | **약관 동의 게이트(클라이언트)**: 임의의 체크 상태 조합(필수 3 × 선택 2 × 전체 토글 조작 시퀀스)에 대해 (a) '다음' 활성 ⇔ 필수 3종 전부 체크(선택 항목 무관), (b) 전체 동의 토글은 5항목 전체 설정/해제와 동치, (c) 서버 규칙(BR-U1-21)과 동일 판정(명세 공유) | 체크박스 조작 시퀀스 생성기(토글 순서 임의) |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| server/app (부트 조립·전역 보안 구성) | No PBT properties identified | 구성·조립 코드 — 보편 양화 가능한 도메인 로직이 없다. 보안 헤더·에러 핸들러·deny-by-default는 예시 기반 통합 테스트+아키텍처 테스트(ArchUnit류)로 검증(SECURITY-04·08·15). 부트스트랩 판정 우선순위(BR-U1-44)의 순수 함수 PBT는 분기 결정 함수를 소유하는 U2 스코프(unit-of-work U2 DoD)로 이관 |
| server/common/core (이벤트 버스·공통 타입·에러 모델) | No PBT properties identified (U1 시점) | U1에서는 계약(타입·발행 인터페이스) 정의가 중심 — 발행·구독 로직에 도메인 불변식이 아직 없다. 이벤트 전달 보장 속성은 최초 실소비 유닛(U3 StayRegistered)에서 식별 |
| 클라이언트 shared/storage (토큰 보안 저장) | No PBT properties identified | OS 보안 저장소 위임 어댑터 — 자체 로직 없음. 저장·복원은 계약 테스트로 충분 |

### 커버리지 대조 (U1 DoD → 속성)

| U1 DoD 명시 속성 | 대응 |
|---|---|
| 계정 상태 머신(미인증→활성→삭제 유예→파기) 불변식 | U1-P1 |
| 리프레시 토큰 회전(재사용 토큰 무효) | U1-P2 (stateful) |
| 닉네임 생성기(패턴 준수·충돌 시 재추첨 수렴) | U1-P5 |
| 동의 증적·프로필 직렬화 왕복 | U1-P3(증적 불변식) + U1-P7(직렬화 왕복) |
| (과제 지정 추가) 온보딩 완료 판정 멱등성 | U1-P10 |
| (과제 지정 추가) 위치 동의 매트릭스 | U1-P11 |

**속성 합계: 17개** (M1: 9 · M2: 6 · C3: 1 · 클라이언트: 1)
