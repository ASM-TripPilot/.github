# 컴포넌트 메서드 (Component Methods)

> **2026-07-04 전면 심화판** · 21개 컴포넌트(M1~M18, C1~C3)의 공개 퍼사드 메서드를 인터페이스 계약 수준으로 정의한다 (Kotlin 유사 표기).
> **소유 경계**: 상세 비즈니스 규칙·상태 전이도·전체 REST/API 스펙·예외 계약 상세는 유닛별 Functional Design 소유 (AD-5). 본 문서는 "어떤 메서드가 존재하고, 무엇을 받고 주며, 어떤 가드와 폴백을 갖는가"까지만 규정한다.
> **커버리지**: stories.md 128개 스토리 전수 매핑 — 1차 102개는 메서드 단위, 후속 26개는 모듈 단위 (문서 말미 매트릭스).

---

## 0. 공통 규약 (전 메서드 적용)

1. **인증·권한 (D22, SECURITY-08)**: 모든 퍼사드 메서드는 인증된 호출자 `caller: Principal`을 암묵 첫 인자로 받는다(시그니처에서 생략). 전 엔드포인트 deny-by-default — 비로그인 접근은 초대·공유 딥링크 수신 진입의 로그인 게이트 이전 화면에 한정하고, 서버 퍼사드는 예외 없이 인증을 요구한다. 리소스 ID(`tripId`, `itineraryId`, `visitId` 등)를 받는 모든 메서드는 실행 전 **호출자 소유권/참여 권한을 서버 측에서 검증**한다(IDOR 방지). 위반 시 `PermissionDenied`.
2. **공통 예외 (ADR-0011 침묵 실패 금지)**: 모든 실패는 타입화된 예외/Result로 표면화한다. 공통 타입 — `AuthenticationRequired`, `PermissionDenied`, `ResourceNotFound`, `ValidationFailed(fieldErrors)`, `ConflictDetected(current)`, `UpstreamUnavailable(source, fallbackApplied)`, `RateLimited(retryAfter)`. 외부 의존(지도·OTA·날씨·LLM·FCM) 호출은 명시적 타임아웃+서킷 브레이커를 전제로 하며(RESILIENCY-10), 각 메서드의 "실패·폴백"은 대표 케이스만 명시한다.
3. **소요시간 비노출 (D25/Δ1)**: 이동시간 값은 `C2.estimateTravel` 내부 계산 전용이다. **어떤 퍼사드도 UI 표시 목적의 소요시간 필드를 반환하지 않는다** — 표시용 응답 DTO는 `DistanceRange`(거리·이동 수단·추정 표기)까지만 포함한다.
4. **여행 겹침 차단 (D21/Δ3)**: 여행 날짜 구간을 생성·변경하는 모든 메서드는 계정 내 날짜 구간 비중첩을 하드 가드로 검증한다. 따라서 "진행 중 여행"은 시스템 전역에서 항상 최대 1개이며, `getActiveHub` 등 활성 여행 조회는 단수를 반환한다.
5. **UGC 검증 (SECURITY-05)**: 사용자 입력 문자열(닉네임·여행 제목·메모·캡션)은 스키마 검증(길이·형식)+`C3.checkText` 금칙어 검증을 통과해야 저장된다.
6. **감사·이력**: 일정을 변경하는 모든 확정 메서드는 `M12.appendChangeLog`(G132 통일 diff 스키마)에 기록을 남긴다. 위치정보 수집·이용은 append-only 법정 로그에 별도 기록한다(N2, SECURITY-14).
7. **표기**: `Page<T>`=페이지네이션 응답, `Flow<T>`=진행 스트림, `T?`=없을 수 있음, `A | B`=합 타입(성공 변형).

---

## M1 Auth — 계정·세션·동의

계정 생성/로그인, 토큰 생명주기(D36), 연령 확인(N1), 약관·위치·마케팅 동의 증적(N2·N3·N8), 계정 삭제(D18), 스플래시 부트스트랩(G5·N4)을 소유한다.

- **`signUpWithSocial(provider: SocialProvider, providerToken: String, ageConfirmation: AgeConfirmation): AuthResult`** — 소셜 4종(Google·Apple·카카오·네이버) 가입/로그인을 제공자 고유 ID(sub)로 통합 처리한다.
  - 입력: 제공자 식별자, 제공자 발급 토큰, 연령 확인(생년월일 또는 '만 14세 이상' 확인). 출력: `AuthResult{accountId, tokenPair, isNewAccount, nextStep(consent|onboarding|home)}`.
  - 가드: 제공자 토큰 서명·수신자 검증. 만 14세 미만이면 계정 미생성 차단(N1). 동일 이메일이 기존 계정에 확인되면 신규 생성 대신 기존 수단 로그인 유도(Apple 비공개 릴레이 등 대조 불가 시에만 별도 계정, G20). 탈퇴 후 30일 유예 중 동일 식별자 재가입 차단(C4).
  - 실패·폴백: `ProviderAuthFailed(provider, retryable)` — 취소/거부는 "로그인이 취소되었습니다" 복귀, 제공자 오류는 재시도 노출. `AgeRestricted` — 사유 안내.
  - 스토리: US-E1-01, US-E1-16 · 결정: D22, D33/N1, G20, C4
- **`signUpWithEmail(email: String, password: String, ageConfirmation: AgeConfirmation): PendingAccount`** — 이메일 가입을 미인증 상태로 생성하고 인증 메일을 발송한다.
  - 입력: 이메일, 비밀번호, 연령 확인. 출력: `PendingAccount{accountId, verificationSentAt, expiresAt(24h)}`.
  - 가드: 비밀번호 8자+영문/숫자+유출 목록 검증, 적응형 해시 저장(G22, SECURITY-12). 중복 이메일이면 계정 미생성·로그인/재설정 유도. 연령 가드 동일(N1). 미인증 계정 7일 후 정리(G22).
  - 실패·폴백: `EmailAlreadyRegistered` — 로그인·비밀번호 재설정 경로 안내. `WeakPassword(reasons)`.
  - 스토리: US-E1-01, US-E1-16 · 결정: G22, D33/N1
- **`verifyEmail(token: String): AuthResult`** — 인증 링크 토큰을 검증해 계정을 활성화하고 로그인시킨다.
  - 가드: 토큰 유효 24시간·1회성. 실패·폴백: `TokenExpired` — 재발송 경로 안내.
  - 스토리: US-E1-01 · 결정: G22
- **`resendVerificationEmail(email: String): ResendResult`** — 인증 메일을 재발송한다.
  - 가드: 분당 1회·일 5회 rate-limit(G22). 실패: `RateLimited(retryAfter)`.
  - 스토리: US-E1-01 · 결정: G22
- **`signIn(credentials: EmailCredentials | SocialCredentials): AuthResult`** — 기존 계정 로그인(수단 비종속).
  - 가드: 브루트포스 방어 — 실패 누적 시 점진 지연/잠금(SECURITY-12). 미인증 이메일 계정은 `nextStep=verifyEmail`.
  - 실패·폴백: `InvalidCredentials`(존재 여부 비노출 문구), `AccountLocked(until)`, 삭제 유예 중 계정은 `cancelAccountDeletion` 경로 안내.
  - 스토리: US-E1-01 · 결정: D22, D36
- **`requestPasswordReset(email: String): Unit` / `resetPassword(resetToken: String, newPassword: String): Unit`** — 등록 이메일로 재설정 링크 발송 후 새 비밀번호 저장.
  - 가드: 계정 존재 여부를 응답으로 노출하지 않음. 새 비밀번호는 G22 정책 동일 적용. 재설정 성공 시 전 기기 리프레시 토큰 무효화.
  - 스토리: US-E1-01 · 결정: G22, SECURITY-12
- **`refreshTokens(refreshToken: String): TokenPair`** — 액세스 1시간/리프레시 90일 토큰 재발급(회전 적용).
  - 가드: 회전된 구 토큰 재사용 감지 시 해당 기기 세션 전체 무효화+보안 알림(SECURITY-03). 다기기 동시 로그인 허용 — 기기별 토큰 체인 분리.
  - 스토리: US-E2-01(세션 유지 기반) · 결정: D36
- **`logout(deviceId: String): Unit`** — 해당 기기의 리프레시 토큰 체인과 FCM 토큰 연결을 무효화한다.
  - 스토리: US-E09-09(계정 관리 일반) · 결정: D36, D12
- **`getBootstrapStatus(accessToken: String?, appVersion: SemVer): BootstrapStatus`** — 스플래시 분기 판정 입력(세션 유효성·약관 재동의 필요·온보딩 완료·최소 지원 버전)을 단일 호출로 반환한다.
  - 출력: `BootstrapStatus{sessionState(valid|expired|none), reconsentRequired: Boolean, onboardingComplete: Boolean, minSupportedVersion: SemVer, forceUpdate: Boolean}` — 응답 계약에 최소 버전 필드 포함(N4).
  - 가드: `forceUpdate=true`면 클라이언트는 서비스 진입(홈·로그인 포함) 차단.
  - 실패·폴백: 타임아웃 3초 — 클라이언트는 로컬 토큰 미만료 시 홈 진입+백그라운드 재검증, 만료 시 로그인 분기(G5).
  - 스토리: US-E2-01, US-E2-06 · 결정: G5, N3, N4/C6, D22
- **`recordConsent(accountId: AccountId, consents: ConsentBundle): ConsentReceipt`** — 최초 약관 동의를 버전·일시와 함께 증적 저장한다.
  - 입력: `ConsentBundle{termsVersion, privacyVersion, locationTermsVersion, marketingOptIn: Boolean, gpsRecordingOptIn: Boolean?}` — 위치기반서비스 약관은 분리된 필수 항목(N2), 마케팅·GPS 기록은 선택(N8·N2). 출력: 동의 항목·일시·버전이 담긴 영수증.
  - 가드: 필수 3종 전부 동의해야 성공. 증적은 불변 저장(N2·N3).
  - 스토리: US-E1-02, US-E1-17, US-E09-14 · 결정: N2/D34, N3, N8
- **`getRequiredConsents(accountId: AccountId): ConsentRequirement` / `reconsent(accountId: AccountId, consents: ConsentBundle): ConsentReceipt`** — 약관 개정 시 재동의 필요 여부 판정과 재동의 수행.
  - 출력: `ConsentRequirement{blocking: List<TermsVersion>, nonBlocking: List<TermsVersion>}` — 중대 변경('재동의 필요' 플래그)은 blocking, 경미 변경은 인앱 공지용.
  - 가드: blocking 미해소 시 다른 퍼사드 접근을 재동의 게이트로 제한.
  - 스토리: US-E1-18, US-E2-01 · 결정: N3
- **`updateMarketingConsent(accountId: AccountId, optIn: Boolean): ConsentReceipt`** — 마케팅 수신 동의 부여·철회(즉시 저장, 증적 갱신).
  - 스토리: US-E09-14, US-E1-02 · 결정: N8
- **`updateLocationConsent(accountId: AccountId, layer: LocationConsentLayer, granted: Boolean): LocationConsentState` / `getLocationConsentState(accountId: AccountId): LocationConsentState`** — 위치 동의 3층(OS 권한 미러 × 앱 내 법정 동의 × GPS 기록 옵트인)을 층별로 갱신·조회한다.
  - 출력: `LocationConsentState{osPermission, legalConsent, gpsRecordingOptIn, effectiveCapabilities}` — 조합별 기능 동작 매트릭스는 Functional Design 소유(G182).
  - 가드: GPS 기록 옵트인 철회 시 `M12.purgeLocationData` 즉시 파기를 트리거하고, 수집·이용 사실은 append-only 법정 로그에 기록(파기와 무관하게 보존, N2). 철회 직전 영향 기능 고지는 클라이언트 책무.
  - 스토리: US-E1-04, US-E1-17, US-E8-03, US-E09-11 · 결정: D34/N2, G182
- **`getPolicyDocuments(): List<PolicyDoc>`** — 이용약관·개인정보처리방침·위치기반서비스 약관의 현행(및 이력) 버전 본문을 상시 열람용으로 반환한다.
  - 스토리: US-E09-13 · 결정: N5 (오픈소스 라이선스 목록은 클라이언트 번들 소유)
- **`requestAccountDeletion(accountId: AccountId, confirmation: DeletionConfirmation): DeletionSchedule`** — 소프트 삭제+30일 유예를 개시한다(즉시 비노출).
  - 출력: `DeletionSchedule{effectiveAt, purgeAt(+30d), cascadeSummary}` — 연쇄 삭제 대상(숙소·여행·기록·회고, 후속: 커뮤니티 콘텐츠·댓글 익명화) 요약 포함.
  - 가드: 사전 고지·재확인 필수. 법정 보존 데이터(위치 로그 등)는 분리 보관. GPS 기록은 즉시 파기(D34).
  - 스토리: US-E09-09 · 결정: D18, D34
- **`cancelAccountDeletion(accountId: AccountId): Unit`** — 30일 유예 내 삭제 철회·계정 복원.
  - 가드: 유예 만료 후 호출 시 `ResourceNotFound`.
  - 스토리: US-E09-09 · 결정: D18, C4

## M2 User Profile — 닉네임·취향·개인화

닉네임 생명주기(G23), 취향 7종 CRUD와 중립 기본값(E1), 온보딩 완료 판정·점진 회수(G24/G157), 개인화 입력 공급(E8-10), 기본 정보·데이터 내보내기(E09-09)를 소유한다.

- **`generateNickname(): String`** — '형용사+여행명사+2자리 숫자' 패턴으로 중복 없는 닉네임을 생성한다(충돌 시 내부 재추첨).
  - 스토리: US-E1-03 · 결정: G23
- **`updateNickname(accountId: AccountId, nickname: String): NicknameResult`** — 닉네임 검증·저장. 실패 시 대체 후보를 동봉한다.
  - 출력: `NicknameResult = Ok(profile) | Rejected(reason: LENGTH|FORBIDDEN|DUPLICATE, suggestions: List<String>)` — 중복 시 원탭 적용 가능한 대체 닉네임 추천 포함.
  - 가드: 2~20자, `C3.checkText(context=nickname)` 금칙어, 유니크 제약(가입 시점부터 적용).
  - 스토리: US-E1-03, US-E1-12 · 결정: G23, C3(닉네임 표시=라이브 참조)
- **`getPreferences(accountId: AccountId): Preferences`** — 취향 7종(스타일·페이스·예산·동행·활동·이동·음식)을 반환하며, 미설정 축은 **중립 기본값**으로 채워 일정 생성이 무실패임을 보장한다.
  - 출력: 축별 `{value?, isNeutralDefault: Boolean}` — 이동 방식 중립 기본=대중교통(보수 추정, 내부 계산 한정), 예산 미설정=가격 필터 미적용 신호.
  - 스토리: US-E1-14, US-E1-13 · 결정: D25/Δ1, D26
- **`updatePreferences(accountId: AccountId, patch: PreferencePatch): Preferences`** — 취향 축 단위 부분 갱신(설정·해제 모두). 저장 즉시 탐색 필터·일정 생성·추천에 반영.
  - 가드: 동행 유형=단일 선택+반려동물 불리언 분리(G19). 예산=전체 총액 기준 원값+매핑 구간 동시 저장, 1박 환산은 여행 생성 시점 수행(G26, D26).
  - 스토리: US-E1-05~10, US-E1-12, US-E1-15, US-E09-10 · 결정: G19, G26, D26
- **`getOnboardingState(accountId: AccountId): OnboardingState` / `completeOnboarding(accountId: AccountId): OnboardingState`** — 온보딩 진행 위치 조회와 완료 판정 기록.
  - 가드: 완료 판정 = **약관 동의+닉네임 통과** (취향 단계 이탈도 완료 처리, G24/G157). 필수 단계(약관·닉네임)는 탈출구 대상 아님.
  - 스토리: US-E1-11, US-E2-01 · 결정: G24/G157
- **`getPendingPreferencePrompts(accountId: AccountId): List<PreferencePrompt>`** — 건너뛴 취향의 점진 설정 카드(한두 개씩)를 홈·첫 여행 생성 직전 노출용으로 산출한다.
  - 스토리: US-E1-11, US-E2-02, US-E4-01 · 결정: G157
- **`getPersonalizationInput(accountId: AccountId): PersonalizationProfile`** — 취향+누적 기록(방문 카테고리·선호 지역·체류 패턴)을 M8/M10 개인화 입력으로 공급한다.
  - 가드: 기록 개인화 동의 OFF면 과거 기록을 제외하고 취향만 반환(기본 추천 신호 동봉).
  - 스토리: US-E1-13, US-E8-10 · 결정: D26
- **`updatePersonalizationConsent(accountId: AccountId, enabled: Boolean): Unit`** — 여행 기록의 추천 활용 동의 토글.
  - 스토리: US-E8-10
- **`getProfileSummary(accountId: AccountId): ProfileSummary` / `updateBasicInfo(accountId: AccountId, patch: BasicInfoPatch): ProfileSummary`** — 마이페이지 기본 정보(닉네임·이메일 등) 조회·수정.
  - 가드: 닉네임 변경은 `updateNickname` 규칙 경유.
  - 스토리: US-E09-09 · 결정: D18
- **`exportUserData(accountId: AccountId): ExportArchive`** — 텍스트 데이터(여행·일정·기록·설정)의 JSON 즉시 다운로드를 생성한다(사진 제외). 각 모듈 퍼사드를 집계하는 오케스트레이션.
  - 스토리: US-E09-09 · 결정: G101/G186

## M3 Accommodation Search — 숙소 탐색·위시리스트

TourAPI+지도 소스 탐색(D09), 필터·정렬(G34), 상세(정적), 라이브 가격(G33), 위시리스트를 소유한다.

- **`searchStays(query: StayQuery, page: PageReq): StaySearchResult`** — 여행지(또는 '내 주변') 기반 숙소 탐색. 날짜·인원 없이 동작한다.
  - 입력: `StayQuery{destination | nearMe(coord?), filters{priceBand, stayTypes, amenities, distanceKm}, sort}` — 거리 기준점=검색 여행지 중심 좌표 직선거리(G34), 가격순 정렬 없음('대표 가격대순' 대체). 출력: `StaySearchResult{cards: Page<StayCard>, degraded: List<SourceFlag>, zeroResultDiagnosis?: FilterRelaxation}` — 카드는 숙소명·위치·대표 가격대(정적, D09), 0건 시 원인 필터·완화 제안 동봉.
  - 가드: 위치 권한 거부 시 '내 주변'은 등록 숙소/여행지 중심 좌표 대체. 편의시설 필터는 채움률 검증 항목만(핵심 3종부터).
  - 실패·폴백: 일부 소스 실패 → 나머지로 구성+`degraded` 표기(침묵 실패 금지). 전체 실패 → `UpstreamUnavailable`+재시도, 직접 등록(M4) 우회 경로는 클라이언트 안내.
  - 스토리: US-E3-01, US-E3-02, US-E3-10, US-E3-11 · 결정: D09, G33, G34, D25
- **`getStayDetail(stayId: CanonicalStayId): StayDetail`** — 위치(핀)·사진·대표 가격대·편의시설·체크인/아웃 시각 등 정적 콘텐츠 상세.
  - 가드: 리뷰·평점 필드 없음(외부 OTA 위임). 누락 필드는 `UNKNOWN`('미확인') 명시값.
  - 스토리: US-E3-03 · 결정: D09
- **`getLivePrice(stayId: CanonicalStayId, dates: DateRange, party: PartySize): LivePriceResult`** — '가격 보기' 탭 시 날짜·인원 입력 후 정확 가격 라이브 조회. 입력값은 세션 내 재사용(클라이언트).
  - 가드: 정확 가격 비캐싱(D09), 파트너 약관 최소 호출 준수(G196).
  - 실패·폴백: `UpstreamUnavailable` — "외부 OTA에서 확인" 딥링크 대체 제시.
  - 스토리: US-E3-01 · 결정: G33, D09, G196
- **`addToWishlist(accountId: AccountId, stayRef: StayRef, memo: String?): WishItem`** — 숙소 저장(계정 귀속, 기기 전환 유지).
  - 가드: 메모는 SECURITY-05 검증. 스토리: US-E3-04 · 결정: D22
- **`updateWishlistMemo(wishItemId: WishItemId, memo: String): WishItem` / `removeFromWishlist(wishItemId: WishItemId): Unit`** — 저장 항목 메모 수정·삭제.
  - 스토리: US-E3-04
- **`listWishlist(accountId: AccountId, page: PageReq): Page<WishItem>`** — 저장 목록(숙소명·대표 사진·가격대·위치·메모, "변동 가능" 표기 플래그).
  - 가드: 원본 소실 항목은 저장 당시 캐시로 표시+"최신 정보 확인 불가" 플래그.
  - 스토리: US-E3-04, US-E4-04, US-E09-06 · 결정: G103

## M4 Saved Accommodation — 등록 숙소·거점 연결

계정 레벨 숙소 풀(D15), 등록 3경로(G31), 내부 숙소 ID 매핑(D17), 여행 거점 연결·날짜 비중첩 검증, 날짜별 기준 거점 타임라인(G41)을 소유한다.

- **`registerStay(accountId: AccountId, input: StayRegistration): SavedStay`** — 예약한/직접 찾은 숙소를 계정 풀에 등록한다.
  - 입력: `StayRegistration{place: PlaceRef(검색 선택|링크 프리필|지도 핀), checkIn: Date, checkOut: Date, party: PartySize, otaBookingNo?: String, amount?: Money, memo?: String}` — 숙소·날짜·인원 필수, 예약번호·금액 선택. 출력: `SavedStay` (등록 직후 클라이언트는 [AI 일정 생성하기] 온램프 노출).
  - 가드: 체크아웃>체크인, 날짜 필수 — 위반 시 `ValidationFailed` 인라인 필드. 좌표 미확정이면 `CoordinateUnresolved` 반환으로 "지도에서 위치 확인" 단계 요구. 국내 좌표 범위 검증(G120).
  - 실패·폴백: 지도/장소 검색 API 실패 → 지도 핀 직접 지정 흐름 폴백(침묵 실패 금지).
  - 스토리: US-E3-06, US-E3-08, US-E4-03, US-E4-10 · 결정: D09, D15, G31, G120
- **`parseOtaLink(url: String): StayPrefill`** — OTA 예약 링크 붙여넣기에서 URL 패턴 파싱만으로(페이지 fetch 없음) 숙소명·위치를 추출해 등록 폼 프리필을 만든다.
  - 가드: 지원 도메인 화이트리스트 검증.
  - 실패·폴백: `LinkParseFailed` — 수동 핀 지정 흐름 안내.
  - 스토리: US-E3-08 · 결정: G31
- **`resolveStayIdentity(externalRefs: List<ExternalStayRef>): CanonicalStayId`** — 소스별 외부 ID N:1 매핑 — 좌표+이름 유사도 자동 매칭(+운영자 보정 훅).
  - 스토리: US-E3-05(동일 숙소 OTA 묶기 기반) · 결정: D17
- **`updateStay(savedStayId: SavedStayId, patch: StayPatch): SavedStay` / `deleteStay(savedStayId: SavedStayId): StayDeletionResult`** — 날짜·인원·예약번호·금액 수정, 등록 취소.
  - 가드: 여행에 연결된 숙소 삭제/기간 변경은 영향 블록 나열 후 차단형 확인(G39). 날짜 변경 시 거점 비중첩 재검증(D15).
  - 스토리: US-E3-06, US-E09-06 · 결정: G39, D15
- **`listMyStays(accountId: AccountId): List<SavedStayView>`** — 저장(위시)/등록 통합 목록 — 출처 라벨 구분, 저장 항목엔 '등록하기' 액션, 연결된 여행·거점 사용 여부 표시.
  - 가드: 선택 항목 미입력은 "미입력" 명시값(빈칸 금지).
  - 스토리: US-E3-09, US-E09-06 · 결정: G103, D15
- **`linkToTrip(savedStayId: SavedStayId, tripId: TripId, stayPeriod: DateRange): BaseAssignment`** — 숙소를 여행 거점으로 연결한다(일자별 다중 거점 지원).
  - 가드: **같은 여행 내 거점끼리 날짜 비중첩 검증**(D15) — 겹치면 `ConflictDetected(overlapDates)`로 조정 요구. 체크인/아웃이 여행 기간 이탈 시 `PeriodMismatch(proposal: 기간 확장)` 경고 반환(자동 변경 없음).
  - 스토리: US-E3-07, US-E4-03, US-E4-04, US-E4-06, US-E4-07 · 결정: D15
- **`unlinkFromTrip(assignmentId: AssignmentId): UnlinkResult`** — 거점 해제. 출력에 후속 선택지(일정 재생성 여부) 포함.
  - 가드: 재생성 거부 시 기존 일정 유지+`M14.cancelTripReminders` 트리거+재생성 유도 배지 신호(G97).
  - 스토리: US-E09-06, US-E09-02 · 결정: G97
- **`getBaseTimeline(tripId: TripId): BaseTimeline`** — 여행 기간 각 날짜의 기준 거점을 산출한다(다중 숙소=체크인 순 전환, 공백일=직전 숙소, 첫날 공백=여행지 중심 좌표).
  - 출력: `BaseTimeline{days: List<{date, base: Stay | RegionCenter, isSmartDefault}}` — 스마트 기본 거점은 비차단 안내 플래그.
  - 스토리: US-E4-06, US-E4-07, US-E5-01, US-E8-05 · 결정: G41, G50, D15
- **`proposeTripDatesFromStay(savedStayId: SavedStayId): DateProposal`** — "체크인/아웃을 여행 기간으로 가져올까요?" 제안 값 산출(수락·수정은 M6에서).
  - 가드: 기존 여행 기간과 충돌 시 두 값을 나란히 반환해 사용자 선택 유도.
  - 스토리: US-E4-05 · 결정: D21, G42

## M5 Affiliate Link — OTA 딥링크·고지·클릭 집계

OTA 숙소명 검색 딥링크(D09), 제휴 수수료 고지, 아웃바운드 클릭 기록(내부 집계), 복귀 핸드오프 카드(G32)를 소유한다.

- **`getOtaLinkOptions(stayId: CanonicalStayId): List<OtaLinkOption>`** — 동일 숙소(내부 ID 기준)에 연결된 OTA 목록을 반환해 사용자가 이탈 대상을 선택하게 한다.
  - 스토리: US-E3-05 · 결정: D17, D09
- **`buildDeeplink(stayId: CanonicalStayId, ota: OtaPartner): DeeplinkWithDisclosure`** — 숙소명 기반 OTA 검색 딥링크와 고지 문구를 생성한다.
  - 출력: `DeeplinkWithDisclosure{url, disclosure{external: "예약·결제는 외부 OTA", affiliate?: "제휴 수수료 수령 가능(추가 비용 없음)"}, showDisclosure: Boolean}` — 고지 숨김 설정 반영.
  - 실패·폴백: 링크 검증 실패/무응답 → `LinkUnavailable` — 같은 숙소의 다른 OTA 링크 또는 장소 검색 우회 경로 동봉.
  - 스토리: US-E3-05, US-E4-10, US-E09-12 · 결정: D09
- **`recordOutboundClick(accountId: AccountId, stayId: CanonicalStayId, ota: OtaPartner): ClickId`** — 아웃바운드 클릭을 내부 집계 전용으로 기록한다(사용자 비노출, LLM 입력 구조적 배제 — SECURITY-11).
  - 스토리: US-E3-05 · 결정: D09, D31
- **`getReturnHandoffCard(accountId: AccountId): HandoffCard?`** — 딥링크 이탈 후 24시간 내 첫 복귀 시 1회 노출할 "예약하셨나요? → 등록하기" 카드(최근 이탈 숙소 1건)를 반환한다.
  - 가드: 무시(디스미스) 이력 있으면 `null`.
  - 스토리: US-E3-05 · 결정: G32
- **`dismissHandoffCard(accountId: AccountId, cardId: CardId): Unit`** — 카드 무시 기록(재노출 방지).
  - 스토리: US-E3-05 · 결정: G32
- **`setDisclosureHidden(accountId: AccountId, hidden: Boolean): Unit`** — 제휴 고지 "다시 보지 않기" 설정·해제.
  - 스토리: US-E09-12

## M6 Trip Creation — 여행·필수 방문지

여행 CRUD(제목 N6, 겹침 차단 D21, 기간 규칙 G42), 예산(전체 총액 D26), 날짜별 시간창(G119), 필수 방문지(한도 G40·사본 G129·변경 미리보기 G43)를 소유한다.

- **`createTrip(accountId: AccountId, input: TripInput): Trip`** — 여행지·날짜(필수)와 인원·예산·제목(선택)으로 새 여행을 생성한다.
  - 입력: `TripInput{destination: RegionRef, dates: DateRange, party: PartySize = 1, budgetTotal?: Money, title?: String, companionType?, transportModes?, perDayWindows?: List<DayWindow>}` — 예산은 전체 총액(항공 제외), 여행 속성(동행·이동·예산대)은 계정 취향을 기본값 제안(G134). 출력: `Trip` (미입력 제목은 '{여행지} N박M일' 자동 생성).
  - 가드: **기존 여행과 날짜 구간 겹침 차단**(D21 — 활성 여행 최대 1개), 시작일 오늘 이후·최대 30일(G42), 국내 한정 좌표/권역 검증(G120), 제목 금칙어 `C3.checkText`(N6).
  - 실패·폴백: `DateOverlap(conflictingTripId, dates)` — 차단·사유 안내. `ValidationFailed`.
  - 스토리: US-E4-01, US-E4-02, US-E4-11 · 결정: D21/Δ3, D26/Δ2, G42, G119, G120, G134, N6
- **`updateTrip(tripId: TripId, patch: TripPatch): Trip`** — 제목·인원·예산·여행 속성 수정.
  - 가드: 제목 금칙어 재검증. 예산 수정 시 파생값(1인·1일) 재계산은 조회 시점 파생.
  - 스토리: US-E4-11, US-E4-01 · 결정: N6, D26
- **`updateTripDates(tripId: TripId, dates: DateRange, confirmation: ImpactConfirmation?): TripDateChangeResult`** — 여행 기간 변경(숙소 날짜 가져오기 수락 포함).
  - 가드: D21 겹침·G42 규칙 재검증. 기간 축소로 영향받는 블록(거점·필수 방문지·일정)이 있으면 영향 목록을 반환하고 `confirmation` 없이는 적용하지 않음(차단형 확인, G39).
  - 스토리: US-E4-05, US-E4-01 · 결정: D21, G42, G39
- **`updateTripWindow(tripId: TripId, perDayWindows: List<DayWindow>): Trip`** — 날짜별 이용 가능 시작/종료 시각 설정(기본 09:00~21:00, 첫날 도착·마지막날 출발 반영).
  - 스토리: US-E4-01 · 결정: G119, D29
- **`getTrip(tripId: TripId): TripDetail`** — 여행 상세(기간·거점 요약·필수 방문지·일정 유무·상태).
  - 스토리: US-E4-01~11 횡단, US-E2-02
- **`listTrips(accountId: AccountId, filter: TripStatusFilter): List<TripCard>`** — 예정/진행 중/종료 구분 목록(D19 전이 규칙 단일 정본). 카드에 목적지·기간·D-day·등록 숙소 수·일정 유무·진행률(여행 중=방문 체크 비율, 여행 전=D-day만).
  - 가드: '진행 중'은 항상 최대 1건(D21).
  - 스토리: US-E09-07, US-E2-02 · 결정: D19, D21, G3
- **`addMustVisit(tripId: TripId, source: PoiRef | List<SavedPoiId>, timing: Timing): List<MustVisit>`** — 필수 방문지 추가 — 검색 추가 또는 저장 POI 체크박스 일괄 투입.
  - 입력: `Timing = Anytime`(포함만 보장) `| Fixed(date, startTime, duration)`. 저장 POI 투입은 **사본 복제**(원본 독립, G129), 권역 밖 항목은 경고 배지+기본 해제 신호(G158).
  - 가드: 하루 3곳×여행 일수 상한 — 초과 시 `LimitExceeded` 차단(G40). 좌표 미확정 POI는 핀 지정 요구. 소실 POI('확인 불가')는 투입 제외 안내(G8).
  - 스토리: US-E4-08, US-E2-05 · 결정: G40, G129, G158, G8
- **`updateMustVisit(mustVisitId: MustVisitId, patch: MustVisitPatch): MustVisit` / `removeMustVisit(mustVisitId: MustVisitId): MustVisitChangeResult`** — 필수 방문지 수정·삭제(시각 고정 토글 포함).
  - 가드: 생성된 일정이 있으면 해당 범위 재계산 필요 신호를 출력에 동봉.
  - 스토리: US-E4-08 · 결정: G43
- **`previewMustVisitChange(tripId: TripId, ops: List<MustVisitOp>): ChangePreview`** — 확정 일정에서의 필수 방문지 변경에 대한 전/후 비교 미리보기를 생성한다(승인 시에만 적용).
  - 실패·폴백: 솔버 비가용 → `UpstreamUnavailable` — 변경 보류 안내.
  - 스토리: US-E4-08 · 결정: G43, D28
- **`deleteTrip(tripId: TripId): CascadePreview` / `confirmDeleteTrip(tripId: TripId, token: PreviewToken): Unit`** — 연쇄 영향(일정·기록·회고) 고지 후 2단계 소프트 삭제(30일 유예).
  - 스토리: US-E09-07(목록 관리 일반) · 결정: D18

## M7 Place Data — POI 정본·후보 풀

POI 수집·정규화·canonical ID(G133), 하이브리드 캐싱(D13), closed-set 후보 풀(G115), 저장 POI(G8), 인기 장소 집계(G2), 영업시간 공급(G192)을 소유한다.

- **`searchPoi(query: String, region: RegionRef?, filters: PoiFilters?): Page<Poi>`** — 장소(명소·맛집·카페 등) 검색 — 카카오 정본+네이버 2차 폴백(D08).
  - 가드: 한국어명 정본+영문 alias(G123 — 한국어 필드 우선, 없으면 원문).
  - 실패·폴백: 1차 소스 실패 → 2차 소스 폴백, 전체 실패 → `UpstreamUnavailable`+재시도.
  - 스토리: US-E2-05, US-E4-08, US-E8-01 · 결정: D08, G123, G133
- **`getPoi(canonicalId: CanonicalPoiId): PoiDetail`** — POI 상세(영업시간·카테고리·좌표·출처 표기). 누락 필드는 '미확인' 명시값, 혼잡도는 1차 '미확인' 통일(G199).
  - 스토리: US-E6-02, US-E5-03 · 결정: D13, G199
- **`savePoi(accountId: AccountId, poiRef: PoiRef): SavedPoi` / `unsavePoi(accountId: AccountId, savedPoiId: SavedPoiId): Unit`** — 장소 우선 저장(Case A 온램프)·해제. 상한 없음.
  - 스토리: US-E2-05 · 결정: G8
- **`listSavedPois(accountId: AccountId): List<SavedPoiView>`** — 저장 POI 목록(이름·카테고리·지역). 원본 소실 항목은 '확인 불가' 배지 유지.
  - 스토리: US-E2-05, US-E4-08 · 결정: G8, G129(투입은 사본 — M6 소유)
- **`getCandidatePool(context: TripContext): CandidatePool`** — 일정 생성·재계획용 closed-set 후보 풀을 구성한다(영업시간·카테고리·체류 기본값·비용 밴드 포함).
  - 출력: `CandidatePool{candidates: List<CandidatePoi>, poolVersion}` — LLM은 이 ID 목록에서만 선택 가능(그라운딩 실패 구조적 불가, G115). 사용자 저장 장소 우선 소싱 신호 포함(G53).
  - 가드: 규모 가정 지역당 후보 5천(G142).
  - 실패·폴백: 외부 소스 부분 실패 → 캐시·TTL 내 데이터로 축소 구성+`degraded` 표기.
  - 스토리: US-E5-02, US-E7-04 · 결정: G115, D13, G53
- **`snapshotUserConfirmed(poiRef: PoiRef, purpose: SnapshotPurpose): PoiSnapshot`** — 사용자 확정 데이터(등록 숙소·방문 체크·필수 방문지)의 확정 시점 스냅샷을 '사용자 입력 데이터'로 영구 저장한다.
  - 스토리: US-E4-08, US-E8-01(기록 귀속 기반) · 결정: D13
- **`resolvePoiIdentity(externalRef: ExternalPoiRef): CanonicalPoiId`** — 소스별 place_id → 내부 canonical ID 매핑(좌표 50m 근접+명칭 유사도).
  - 스토리: (횡단 — POI 참조 정합) · 결정: G133/G148
- **`getTrendingPlaces(region: RegionRef): List<TrendingPlace>`** — 최근 7일 저장+방문 가중합의 일 1회 배치 집계 결과('지금 인기 장소').
  - 실패·폴백: 집계 미존재 → 빈 결과+클라이언트는 섹션 비표시(허위 데이터 금지).
  - 스토리: US-E2-02 · 결정: G2
- **`getNearbyRecommendations(coord: GeoPoint, context: NearbyContext): List<Poi>`** — 현장 주변 추천(열람·저장 자유, 일정 편입은 Plan-B 재계획 흐름 경유 신호 포함).
  - 스토리: US-E6-02 · 결정: G64
- **`getStayDurationDefaults(category: PoiCategory): StayDurationRange`** — 카테고리별 체류 시간 기본값(운영 정적 테이블 20~30종, 최소·권장·최대)을 C2/M8에 공급한다.
  - 스토리: US-E5-03 · 결정: G51
- **`refreshOpeningHours(poiIds: List<CanonicalPoiId>): OpeningHoursRefreshResult`** — 정기 영업시간 변경 감지용 재조회 배치(TourAPI·지도) — M9 휴무 트리거 입력.
  - 가드: 당일 임시휴무는 자동 감지 대상 아님(수동 트리거 대응).
  - 스토리: US-E7-02 · 결정: G192

## M8 Itinerary Generation — 일정 생성·편집·확정

생성 오케스트레이션(LLM 점수→솔버 배치), 3방식 분기(G48), 편집 검증(D28), plan/current 이중 모델(D14), 확정/해제 상태 머신(D20), LOCK·warm-start(G46/G136)를 소유한다.

- **`generateItinerary(tripId: TripId, mode: GenerationMode): GenerationSession`** — 여행 전체의 날짜별 일정 생성을 시작한다. `mode = FullAuto | Together | Manual`.
  - 입력: tripId(거점 타임라인·필수 방문지·취향·시간창은 서버가 M4/M6/M2에서 재조회 — D31 패턴). 출력: `GenerationSession{sessionId, firstDayEta}`.
  - 가드: 등록 숙소 0개면 `PreconditionFailed(NO_BASE)` — 단 '숙소 나중 등록' 온램프(mode 무관, `recommendStayZone` 경로)는 허용. 체크인/아웃 누락·역전 시 날짜 수정 유도. 지오코딩 미확정 숙소는 생성 보류.
  - 실패·폴백: LLM 실패 → 결정론적 솔버(규칙 점수+TOPTW)로 지속+"기본 모드" 고지. 첫 1일 5초/전체 20초 한계 초과 → 솔버 폴백 고지(D38). 전 경로 실패 → 숙소+시각 고정 필수 방문지만 배치한 최소 일정+재시도 노출.
  - 스토리: US-E5-01, US-E5-02, US-E5-03, US-E5-04, US-E5-09, US-E5-10, US-E1-14 · 결정: D28, D38, D31, G50, G115
- **`streamGenerationProgress(sessionId: SessionId): Flow<GenerationProgress>`** — 단계 텍스트·진행률·경과 시간·일자별 완료 이벤트 스트림(첫 1일 우선 노출).
  - 스토리: US-E5-09 · 결정: D38
- **`cancelGeneration(sessionId: SessionId, keepDraft: Boolean = true): DraftItinerary?`** — 생성 취소 — 부분 일정을 초안으로 저장하고 '이어서 생성' 진입점을 남긴다.
  - 스토리: US-E5-09 · 결정: G161
- **`resumeGeneration(itineraryId: ItineraryId): GenerationSession`** — 초안에서 이어서 생성(기존 부분 결과 warm-start).
  - 스토리: US-E5-09 · 결정: G161, G136
- **`switchGenerationMode(sessionId: SessionId, newMode: GenerationMode): GenerationSession`** — 분기 도중 방식 전환(예: 같이 고르기→나머지 완전 AI) — 이미 확정한 슬롯 보존.
  - 스토리: US-E5-10
- **`getSlotCandidates(sessionId: SessionId, slotCursor: SlotCursor, concept: Concept, radius: Meters): SlotCandidateSet`** — 같이 고르기: 슬롯별 컨셉·반경 내 후보 제시. 기준점=직전 확정 슬롯(첫 슬롯=등록 숙소).
  - 출력: 반경 내 후보(+반경 확대 시 예상 이동 거리 표기), 반경 밖 후보는 `disabled` 플래그. 0건이면 반경 확대/컨셉 변경 제안 동봉.
  - 스토리: US-E5-10 · 결정: G48, D25
- **`chooseSlotCandidate(sessionId: SessionId, slotId: SlotId, candidateId: CandidateId): NextSlotContext`** — 같이 고르기 슬롯 확정 후 다음 슬롯 컨텍스트 반환.
  - 스토리: US-E5-10 · 결정: G48
- **`getItinerary(itineraryId: ItineraryId): ItineraryView`** — 일정 조회(시간표/지도 공용 데이터 — 슬롯·고정/추천 구분·이동 구간 `DistanceRange`).
  - 가드: 이동 구간에 수단·거리 요약만 포함, 소요시간 필드 없음(D25/Δ1). 여행 중 조회는 온라인 전제 — 실패 시 오류·재시도(D24/Δ6, 오프라인 스냅샷 없음).
  - 실패·폴백: 지도 타일 실패는 클라이언트 시간표형 폴백(데이터는 본 메서드가 정상 제공).
  - 스토리: US-E5-06, US-E5-08, US-E5-12, US-E2-03 · 결정: D25/Δ1, D24/Δ6, D14
- **`getExplanations(itineraryId: ItineraryId, slotIds: List<SlotId>?): Map<SlotId, Explanation>`** — 슬롯별 추천 이유(취향 근거)+배치 이유(솔버 제약 근거) 설명 문구.
  - 가드: 표시 전용 — 설명·배치 불일치 시 검증 시각 우선. LLM 실패 시 설명만 생략("불러오지 못했어요"), 일정 자체는 영향 없음.
  - 스토리: US-E5-05, US-E5-09 · 결정: D11, ADR-0011
- **`validateEdit(itineraryId: ItineraryId, ops: List<EditOp>): ValidationResult`** — 편집(추가/삭제/재정렬/시간 변경)의 서버 확정 검증 — 클라이언트 경량 검증기와 규칙 공유(C2 명세).
  - 출력: `ValidationResult{violations: List<Violation{slotId, kind: TRAVEL_TIME|OPENING_HOURS|FIXED_CONFLICT, detail}>}`.
  - 스토리: US-E5-07 · 결정: D28
- **`getInsertableSlots(itineraryId: ItineraryId, poiId: CanonicalPoiId): List<SlotWindow>`** — POI 추가 시 현재 동선 기준 삽입 가능한 시간대 후보만 산출한다.
  - 스토리: US-E5-07 · 결정: D28
- **`applyEdit(itineraryId: ItineraryId, ops: List<EditOp>, onViolation: ViolationPolicy): EditResult`** — 편집 저장. `onViolation = AutoFix | SaveAsIs`.
  - 가드: `AutoFix`는 위반 구간의 **시각·순서만** 조정(POI 추가/삭제 없음, G49). `SaveAsIs`는 위반 보존+지속 가시화 플래그. 확정 상태 일정이면 `unlockForEdit` 선행 필요(D20). 저장 성공 시 changelog 기록.
  - 실패·폴백: 저장 네트워크 오류 — 클라이언트 로컬 임시 보관+"저장 대기 중"+재연결 동기화.
  - 스토리: US-E5-07, US-E5-08 · 결정: G49, D28, D20, G132
- **`lockSlot(itineraryId: ItineraryId, slotId: SlotId): Slot` / `unlockSlotLock(itineraryId: ItineraryId, slotId: SlotId): Slot`** — 슬롯 고정(LOCK)·해제 — 완전 AI 초안의 사후 필수화.
  - 스토리: US-E5-10, US-E4-08 · 결정: G46
- **`replaceSlot(itineraryId: ItineraryId, slotId: SlotId, preference: ReplacePreference?): List<SlotCandidate>`** — 개별 슬롯 교체 후보 제시(솔버 검증 통과분만).
  - 스토리: US-E5-10 · 결정: G46, D28
- **`regenerate(itineraryId: ItineraryId, scope: RegenerationScope): GenerationSession`** — 재생성 — LOCK 슬롯·체류 고정값·수동 추가 POI를 고정 블록으로 보존(warm-start)하고 나머지만 재생성. 여행당 활성 일정 1개 유지, 이전 상태는 changelog 추적.
  - 스토리: US-E5-10, US-E4-09, US-E09-06 · 결정: G46/G136
- **`confirmItinerary(itineraryId: ItineraryId): ConfirmedSnapshot`** — 일정 확정 — plan 스냅샷 동결(불변), 이후 변경은 current에만.
  - 가드: 확정 시 `M14.scheduleTripNotifications` 트리거.
  - 스토리: US-E5-12 · 결정: D14
- **`unlockForEdit(itineraryId: ItineraryId): DraftState`** — 확정 해제 → '편집 중' 전환. 저장 후 재확정 필요, 재확정 전 신규 공개 불가(후속 커뮤니티 연계 가드).
  - 스토리: US-E5-12 · 결정: D20
- **`recommendStayZone(tripId: TripId): StayZoneRecommendation`** — 숙소 나중 등록 온램프 — 동선 무게중심·평균 이동 거리 기반 숙소 권역 추천(before/after '추정 이동 거리' 비교, 전역 최적 비보장 표기).
  - 스토리: US-E5-11 · 결정: D25

## M9 Plan-B Detection — 자동 트리거 감지

트리거 4종 하이브리드 감지(D27), 빈도 상한·억제(G58/G195), 민감도, 휴식 모드 연동 억제(G54)를 소유한다. 위치 의존 트리거는 앱 내 법정 동의 상태를 게이트로 한다(D34).

- **`evaluateServerTriggers(): List<TriggerEvent>`** — (스케줄러 전용) 활성 여행 전체에 대한 날씨(강수 60%·기상특보, M11)·휴무(M7 재조회) 배치 폴링 평가. 폴링 주기: 날씨 1시간·휴무 당일 아침 1회(remote config).
  - 가드: 외부 API 무응답 시 **트리거 침묵**(허위 알림 금지) — 다음 정상 응답 시 재평가. 발화 이벤트는 M14 발송 경로로 전달(푸시는 심각 사유 한정).
  - 스토리: US-E7-02, US-E09-03 · 결정: D27, D10, G192, ADR-0011
- **`reportClientSignal(tripId: TripId, signal: ClientSignal): TriggerDecision`** — 클라이언트 포그라운드 이벤트(이동 지연·체류 초과·위치 갱신) 수신·판정.
  - 입력: `ClientSignal = Delay(legRef, observedGap) | Overstay(slotId, elapsed) | Position(coord, at)`. 출력: `TriggerDecision{fire: Boolean, banner?: TriggerBanner, suppressedBy?: SuppressReason}`.
  - 가드: 위치정보 동의 철회 상태면 이동 지연 트리거 미평가(위치 비의존만 유지, D34). 판정 로직은 clock·외부 데이터 주입 가능한 순수 함수 분리(G116). 지연 임계 30분 등 파라미터는 remote config(G106).
  - 스토리: US-E7-02, US-E8-01(체류 초과 연결), US-E6-01 · 결정: D27, D34, G106, G116
- **`getActiveTriggerBanners(tripId: TripId): List<TriggerBanner>`** — 현재 유효한 비차단 배너/칩 목록(트리거 유형·영향 일정·데이터 출처·감지 시각·'대안 보기' 딥링크).
  - 스토리: US-E7-02, US-E09-03 · 결정: D27
- **`dismissTrigger(triggerId: TriggerId): Unit`** — 배너 닫기 — 동일 방문지·동일 사유 재노출 방지, 2회 연속 무시 시 당일 동일 사유 억제 카운트 반영.
  - 스토리: US-E7-02 · 결정: G58
- **`updateSensitivity(accountId: AccountId, level: SensitivityLevel): SensitivityConfig`** — 민감도 3단계(적게/보통/많이 — 상한 ±50% 조정) 설정.
  - 스토리: US-E7-02, US-E09-03 · 결정: G58/G195
- **`getSuppressionState(tripId: TripId): SuppressionState`** — 현재 억제 상태(전역 상한 시간당 2/일 8, 휴식 모드 경미 트리거 억제, 10분 중복 제한) 조회 — 클라이언트 배너 렌더 판단용.
  - 스토리: US-E7-02, US-E7-06 · 결정: G58, G54, US-E09-03(10분 규칙)

## M10 Itinerary Recalculation — Plan-B 재계획

재계획 세션(사유→후보→검증→확정), 당일 잔여 재정렬(C10), 수동 수정 폴백, 휴식 모드 전이(G54/G159)를 소유한다.

- **`startReplan(tripId: TripId, reason: ReplanReason?, method: ReplanMethod, position: GeoPoint?): ReplanSession`** — 재계획 세션 시작. `reason = Weather|Closure|Delay|Cancelled|Fatigue|None`, `method = DelegateToAi | ManualEdit`.
  - 가드: '여행 중' 상태에서만 허용(날짜 구간 진입 판정, D19) — 위반 시 `PreconditionFailed(NOT_ACTIVE)`. 등록 숙소 0개여도 날짜 구간 안이면 허용. 진행 중 여행은 D21로 단수 — 대상 모호성 없음. `position` 미제공+GPS 불가 시 세션은 `positionRequired` 상태로 생성.
  - 스토리: US-E7-01, US-E7-03, US-E7-12 · 결정: D19, D21, D22
- **`setManualPosition(sessionId: ReplanSessionId, position: ManualPosition): ReplanSession`** — GPS 폴백 — 지도 핀/장소 검색으로 현재 위치 수동 입력('추정 출발지' 표기 플래그).
  - 가드: 생략 시 마지막 완료 방문지 또는 등록 숙소를 기준점으로 사용(가정 명시 플래그).
  - 스토리: US-E7-10 · 결정: D27
- **`getAlternatives(sessionId: ReplanSessionId): AlternativeSet`** — 대안 후보 2~3개 산출(10초 목표).
  - 출력: `AlternativeSet = Candidates(List<Alternative>) | Fallback(reasonLine, options: [SkipOne, RestMode, ManualEdit])` — 각 `Alternative{slots, rationale(사유 연결 LLM 설명), distanceFromHere: DistanceRange, stayEstimate, slackToNextFixed}` — 소요시간 필드 없음(D25/Δ1), 추정치·산출 기준 표기.
  - 가드: 전 후보 솔버 하드 제약 검증 통과분만(숙소 시간창·영업시간·이동 부등식·고정 시각). 후보 소싱은 저장 장소 우선→주변 확장(G53). 고정 제약(숙소·시각 고정 방문지·완료 방문지)은 재배치 대상 제외.
  - 실패·폴백: 후보 0건/전부 검증 실패/10초 초과 → `Fallback` 3종+사유 한 줄(D38, ADR-0011). LLM 실패 → 결정론적 후보 산출(설명 축약).
  - 스토리: US-E7-04, US-E7-05, US-E7-12, US-E7-03 · 결정: G53, D38, D25, C2 검증
- **`previewAlternative(sessionId: ReplanSessionId, choiceId: ChoiceId): ChangePreview`** — 확정 직전 전/후 비교 — 추가/삭제/시간 이동 diff, 영향 지표(총 이동 거리 증감·방문지 수 증감·숙소 복귀 시각 변화 — 소요시간 없음).
  - 스토리: US-E7-08 · 결정: D25, D14
- **`confirmAlternative(sessionId: ReplanSessionId, choiceId: ChoiceId): CurrentItinerary`** — 대안 확정 — 확정 버튼 시점 솔버 재검증 1회 후 current에만 반영, changelog 기록.
  - 가드: 재정렬 범위=당일 잔여만, 시각 고정·완료 방문지 보존(warm-start). 이월 방문지는 미배치 목록 보관+확정 전 명시 동의(C10). plan 스냅샷 불변(D14). 확정 시 `M14.rescheduleOnItineraryChange` 트리거.
  - 실패·폴백: 재검증 무효화 → `RevalidationFailed` — 후보 재산출 안내(G56).
  - 스토리: US-E7-07, US-E7-08, US-E7-09 · 결정: G56, C10, D14, G57/G132
- **`keepCurrentPlan(sessionId: ReplanSessionId): Unit`** — '기존 유지' — 어떤 변경도 저장하지 않고 세션 종료.
  - 스토리: US-E7-06
- **`getUnplacedPois(tripId: TripId): List<UnplacedPoi>` / `rescheduleUnplaced(tripId: TripId, unplacedId: UnplacedId, targetDate: Date): ReplanSession`** — 미배치(이월) 목록 조회와 날짜 선택 시 해당 일 재계산 세션 시작.
  - 스토리: US-E7-07 · 결정: C10
- **`applyManualEdit(tripId: TripId, ops: List<ManualEditOp>): ManualEditResult`** — 수동 일정 수정 폴백 — 순서 재배열·방문지 삭제·예정 시각 직접 입력.
  - 가드: **숙소 체크인/체크아웃 고정 제약은 위반 불가** — 침범 입력 시 경고+저장 차단. 외부 데이터 누락 항목(이동시간 자동 계산 불가 등) 표기 플래그 동봉. API 복구 시 자동 검증 재활성(수동 결과 유지).
  - 스토리: US-E7-11, US-E7-12 · 결정: ADR-0011, D28
- **`enterRestMode(tripId: TripId, resumeAt: Instant?): RestState`** — 휴식 모드 전환('여행 중' 하위 상태) — 경미 트리거·일정 알림 억제, 심각 사유만 유지.
  - 스토리: US-E7-06 · 결정: G54
- **`resumeFromRest(tripId: TripId): ReplanProposal`** — 재개(수동 또는 재개 시각 도달 알림 후) — 남은 일정 재계산 제안 반환.
  - 스토리: US-E7-06 · 결정: G159

## M11 Weather & Context — 기상 데이터

기상청 공공데이터포털 단기예보·특보(D10) 수집·캐시, 좌표→격자 변환, M8/M9 공급을 소유한다.

- **`getForecast(grid: GridCoord, timeRange: TimeRange): Forecast`** — 단기예보(강수확률 포함) 조회 — TTL 캐시 경유.
  - 실패·폴백: `UpstreamUnavailable` — 호출측(M9)은 트리거 침묵, (M8)은 날씨 비반영 생성 지속.
  - 스토리: US-E7-02 · 결정: D10, ADR-0011
- **`getActiveAlerts(region: RegionRef): List<WeatherAlert>`** — 기상특보 조회(심각 트리거·휴식 모드 예외 알림의 입력).
  - 스토리: US-E7-02, US-E7-06 · 결정: D10, G54
- **`getPrecipitationRisk(coord: GeoPoint, arrivalWindow: TimeRange): PrecipRisk`** — 방문지 도착 시간대의 강수확률 60% 판정용 파생 값(트리거 (a) 전용).
  - 스토리: US-E7-02 · 결정: D10, D27
- **`convertToGrid(coord: GeoPoint): GridCoord`** — 위경도→기상청 격자 좌표 변환(결정적 순수 함수).
  - 스토리: (M9/M8 내부 공급) · 결정: D10
- **`getProviderHealth(): ProviderHealth`** — 데이터 최신성·실패율 상태 — 관측성·침묵 정책 판단 입력(RESILIENCY-05·10).
  - 스토리: US-E7-02(허위 알림 금지 기반) · 결정: ADR-0011

## M12 Travel Archive — 기록(actual)·changelog·GPS

방문 기록(actual)·사진·메모·즉석 방문, GPS 발자취(G55/G73), 오프라인 동기화·충돌(G74), changelog 통일 스키마(G132), plan/current/actual 대조를 소유한다.

- **`checkVisit(tripId: TripId, slotId: SlotId, status: VisitStatus, at: Instant?): VisitRecord`** — 방문 완료/방문 안 함 체크 — actual 시각 자동 기록(기기 시각), 사용자 수정 가능.
  - 가드: 실제 체류 시간=해당 체크 시각~다음 장소 체크 시각으로 산출, plan/actual 구분 보관(D23·D14). 체류 초과 판정 신호를 M9로 공급.
  - 스토리: US-E8-01, US-E6-01 · 결정: D23, D14
- **`updateVisitRecord(visitId: VisitId, patch: VisitPatch): VisitRecord`** — 기록 수정(시각·상태) — 여행 종료 후에도 허용, 회고·분석은 자동 갱신하지 않음(수동 '다시 생성'만).
  - 스토리: US-E8-01, US-E8-11 · 결정: C11
- **`addImpromptuVisit(tripId: TripId, place: PoiRef | FreeText): VisitRecord`** — 계획에 없던 즉석 방문 추가 — 자유 텍스트는 좌표·카테고리 없이 저장(분석 '기타' 처리).
  - 스토리: US-E8-01 · 결정: G77
- **`attachMedia(visitId: VisitId, photos: List<PhotoUpload>, memo: String?): MediaResult`** — 장소별 사진(최대 20장, 클라이언트 압축 장당 5MB·긴 변 2048px)·메모 첨부.
  - 출력: `MediaResult{accepted, pending: List<UploadTicket>, rejected}` — 부분 실패 격리(메모·체크는 사진 실패와 독립 저장).
  - 실패·폴백: 업로드 실패 → 로컬 큐 '업로드 대기'+자동 재시도 3회, 이후 수동 재시도 버튼.
  - 스토리: US-E8-02 · 결정: G75/G145
- **`appendGpsTrack(tripId: TripId, polyline: SimplifiedPolyline, period: TimeRange): TrackSegment`** — 포그라운드 저빈도(1~5분) 수집분의 단순화 폴리라인 보존(원시 좌표는 가공 후 파기).
  - 가드: **GPS 기록 옵트인 동의 전제** — 미동의 호출은 `PermissionDenied`(D34). 수집·이용 사실은 append-only 법정 로그 기록(N2).
  - 스토리: US-E7-13, US-E8-03 · 결정: G55/G73, D34/N2
- **`purgeLocationData(accountId: AccountId): PurgeReceipt`** — 옵트인 철회·탈퇴 시 GPS 위치 데이터 즉시 파기(법정 로그는 보존).
  - 스토리: US-E1-17, US-E8-03, US-E09-11 · 결정: D34/N2
- **`getRouteComparison(tripId: TripId, date: Date): RouteComparison`** — 계획 동선 vs 실제 GPS 경로 지도 비교 데이터(+누적 실제 이동 거리·걸음 수 환산 추정, 추정 표기).
  - 가드: 옵트인/권한 없으면 실제 경로 레이어 `disabled(reason)` — 계획 동선만 반환. 걸음 수=GPS 거리 환산 추정만(추가 권한 없음, G59).
  - 스토리: US-E7-13 · 결정: G55, G59, G72, D34
- **`syncOfflineRecords(batch: OfflineBatch): SyncResult`** — 오프라인 입력(방문 체크·사진 메타·메모·수동 체크인) 일괄 동기화.
  - 출력: `SyncResult{applied, conflicts: List<RecordConflict{recordId, local, server}>}` — 레코드 단위 버전 비교, 사진·메모 추가는 합집합 자동 병합.
  - 가드: 오프라인 보장은 '입력' 한정(조회는 온라인 전제, Δ6).
  - 스토리: US-E8-12 · 결정: G74, D24/Δ6
- **`resolveSyncConflict(conflictId: ConflictId, resolution: KeepLocal | KeepServer): VisitRecord`** — 충돌 항목별 사용자 선택 적용.
  - 스토리: US-E8-12 · 결정: G74
- **`appendChangeLog(entry: ChangeLogEntry): ChangeLogId`** — 통일 diff 스키마(행위자·출처 유형·사유·전/후 값·유발 참조) 기록 — Plan-B·편집·(후속)공동편집·어시스턴트 공용.
  - 스토리: US-E7-09, US-E8-04 · 결정: G57/G132, G13(후속 참조 필드)
- **`getTripTimeline(tripId: TripId): TripTimeline`** — plan(불변)/current/actual/changelog 4종 대조 뷰 데이터(라벨 구분·차이 하이라이트 입력).
  - 스토리: US-E8-04, US-E8-11, US-E7-09 · 결정: D14
- **`getTripRecords(tripId: TripId): TripRecordBundle`** — 날짜별 기록 묶음(방문·사진·메모·귀속 기준 거점) — 귀속은 M4.getBaseTimeline 기준(숙소 없는 날은 날짜만).
  - 스토리: US-E8-05, US-E8-11 · 결정: D15, G41
- **`listArchivedTrips(accountId: AccountId): List<ArchivedTripCard>` / `getRecordCalendar(accountId: AccountId, month: YearMonth): RecordCalendar`** — 여행 단위 기록 목록·월 캘린더 마킹(겹침 없음 보장 — D21).
  - 가드: 0건이면 빈 상태 안내용 메타 반환.
  - 스토리: US-E8-11, US-E8-14 · 결정: D21, C11
- **`getVisitStats(scope: TripId | AccountId): VisitStats`** — 방문 체크 완료 비율(진행률 G3)·누적 방문 장소 수(스타일 분석 10곳 게이트 카운트)·이동 거리 합산(G72 혼합) 집계.
  - 스토리: US-E2-02, US-E8-09, US-E09-08 · 결정: G3, G72

## M13 AI Reflection — 회고·요약·스타일 분석

당일 회고·전체 요약·스타일 분석(10곳 게이트)·공유 카드를 소유한다. LLM 호출은 전부 C1 경유(상위 티어).

- **`generateDailyReflection(tripId: TripId, date: Date): Reflection | FallbackCard`** — 방문·사진·메모·변경 이력 기반 당일 회고 초안 생성.
  - 가드: 부분 데이터는 가용분만으로 생성+누락 명시('사진 없음' 등). 기록 0건이면 `NoActivity` 변형(수동 기록 유도). 오프라인 구간은 생성 보류 후 복구 시 생성. 완료 시 M14 알림 트리거.
  - 실패·폴백: LLM 실패/타임아웃 → `FallbackCard{방문 N곳·이동 Nkm·사진 N장}` (침묵 실패 금지).
  - 스토리: US-E8-06, US-E8-12, US-E09-04 · 결정: D11, ADR-0011
- **`editReflection(reflectionId: ReflectionId, content: ReflectionEdit): Reflection`** — 회고 문구 수정·메모 덧붙임 — 원본 초안과 별도 저장, 수정본이 최종 표시본.
  - 가드: 폴백 카드 위 직접 작성도 허용.
  - 스토리: US-E8-07 · 결정: G78
- **`regenerateReflection(reflectionId: ReflectionId, overwriteConfirmed: Boolean): Reflection`** — 회고 재생성 — 수정본 존재 시 `overwriteConfirmed` 없이는 `OverwriteWarning` 반환(확인 후에만 교체).
  - 스토리: US-E8-07 · 결정: G78
- **`generateTripSummary(tripId: TripId): TripSummary`** — 여행 종료 전이(D19) 시 전체 요약 생성 — 지도 히어로(방문 핀·날짜별 동선·거점 마커)+통계(방문 수·총 이동 거리 G72 혼합 합산·사진 수·하이라이트).
  - 가드: 위치 데이터 전무 시 지도 대신 방문 목록 구성 신호. 실패 시 날짜별 기본 카드 모음 폴백.
  - 스토리: US-E8-08, US-E09-04 · 결정: D19/Δ4, G72, D34
- **`regenerateTripSummary(tripId: TripId): TripSummary`** — 종료 후 기록 편집분 반영은 본 수동 재생성으로만.
  - 스토리: US-E8-11 · 결정: C11
- **`analyzeTravelStyle(accountId: AccountId): StyleAnalysis | PendingGate`** — 방문 패턴(카테고리 분포·평균 체류·이동 반경·활동 밀도) 기반 스타일 분석 — 취향 7종 축 매핑 자체 택소노미.
  - 출력: 누적 방문 10곳 미만이면 `PendingGate{current: N, required: 10, previewFromPreferences}` (임시 미리보기=정식 분석 아님 명시).
  - 스토리: US-E8-09, US-E09-08 · 결정: G76
- **`getStyleSummaryCard(accountId: AccountId): StyleCard`** — 마이페이지 요약 카드(대표 디스크립터+선호 칩+dot 게이지+분석 여행 수·갱신 시점).
  - 스토리: US-E09-08 · 결정: G76
- **`buildShareCard(source: TripSummaryRef | DailyReflectionRef, format: CardFormat, caption: CaptionEdit?): ShareCardImage`** — SNS 공유 카드 이미지 구성(9:16/1:1/4:5) — 대표 사진·제목·기간·통계·워터마크.
  - 가드: 사진 0장이면 지도·통계 기반 레이아웃 대체. 여행 미종료/요약 미생성이면 `PreconditionFailed`. 캡션 금칙어 검증.
  - 스토리: US-E8-13 · 결정: D25(거리 통계만), C3

## M14 Notification — 알림 스케줄링·발송·알림함

서버 스케줄링 발송(D32), FCM 단일 채널(D12), 종류별 토글, 방해금지(G100), 인앱 알림함(90일)을 소유한다.

- **`scheduleTripNotifications(tripId: TripId): SchedulePlan`** — 여행 확정 시 리마인드 일괄 스케줄링 — D-1, 당일 요약(기본 08:00, 설정 가능), 개별 일정 N분 전(0/15/30/60, 기본 30).
  - 가드: 알림 본문에 장소명·시작 시각·직전 지점으로부터의 **이동 거리만** 포함(소요시간 없음, D25). 일정 없는 여행은 "숙소 등록" 안내 1회만.
  - 스토리: US-E09-02 · 결정: D32, D25, D12
- **`rescheduleOnItineraryChange(tripId: TripId): SchedulePlan`** — 일정 변경(편집·Plan-B 확정) 시 발송 시각 전면 재계산.
  - 스토리: US-E09-02, US-E7-09 · 결정: D32
- **`cancelTripReminders(tripId: TripId, reason: CancelReason): Unit`** — 리마인드 중단(출발점 해제 후 재생성 거부 등) — 재생성 유도 배지 신호와 짝.
  - 스토리: US-E09-02, US-E09-06 · 결정: G97
- **`sendEvent(event: NotificationEvent): DeliveryResult`** — 이벤트성 알림 발송 단일 진입점(숙소 등록/저장 완료, Plan-B 트리거, 회고 완료, 시스템·보안).
  - 가드: 종류·채널 토글 평가 → 방해금지(기본 22~08시) 억제 — **여행 '진행 중' Plan-B 알림만 예외**(G100). 억제·푸시 OFF여도 인앱 알림함엔 적재(채널 독립). Plan-B 중복은 동일 일정 10분 내 1회. 보안·계정 시스템 알림은 전 채널 OFF여도 인앱 표시.
  - 실패·폴백: FCM 실패 → 인앱 적재는 보장+재시도, 침묵 실패 금지.
  - 스토리: US-E09-01, US-E09-03, US-E09-04, US-E7-02 · 결정: D12, D32, G100
- **`getSettings(accountId: AccountId): NotificationSettings` / `updateToggles(accountId: AccountId, perType: Map<NoticeType, ChannelToggles>): NotificationSettings`** — 종류별(숙소 완료·D-1·당일·일정 전·Plan-B·회고 완료 — Δ10 목록 정본) 푸시/인앱 독립 토글. 즉시 저장·다음 알림부터 반영.
  - 가드: OS 푸시 권한 거부 상태 플래그 반환(토글 비활성 표시용).
  - 스토리: US-E09-05 · 결정: D32, Δ10
- **`updateQuietHours(accountId: AccountId, window: TimeWindow): NotificationSettings`** — 방해금지 창 조정(기본 22~08시).
  - 스토리: US-E09-05, US-E09-02 · 결정: G100
- **`getInbox(accountId: AccountId, page: PageReq): Page<Notice>` / `markRead(accountId: AccountId, noticeIds: List<NoticeId>): Unit`** — 인앱 알림함(90일 보존)·읽음 관리. 홈 상단 미읽음 배지 카운트 포함.
  - 스토리: US-E09-05, US-E2-02 · 결정: D32
- **`registerDeviceToken(accountId: AccountId, deviceId: String, fcmToken: String): Unit`** — 기기별 FCM 토큰 등록·갱신(다기기 허용, 로그아웃 시 해제).
  - 스토리: US-E09-05(발송 기반) · 결정: D12, D36

## M18 Trip Execution — 여행 중 실행 허브

활성 일정 허브 상태 머신, 도착 확인 프롬프트(D23), 방문 시작/완료 전이(actual은 M12 위임), 실제 체류 측정(M9 신호), 여행 종료 전이(D19), 진행률(G3)을 소유한다. (Δ7 신설)

- **`getActiveHub(accountId: AccountId): ActiveTripHub?`** — 진행 중 여행(항상 최대 1개 — D21)의 활성 일정 허브 상태를 반환한다.
  - 출력: `ActiveTripHub{trip, todaySlots(진행 상태: 완료/진행 중/다음), nextSlot, nextLegDistance: DistanceRange, progress, restState?}` — 소요시간 필드 없음(D25). 홈 활성 카드·일정 탭이 수렴하는 단일 허브 데이터.
  - 가드: 온라인 전제 — 실패 시 오류·재시도(오프라인 조회 미보장, D24/Δ6).
  - 스토리: US-E2-02, US-E2-03, US-E5-08, US-E6-03 · 결정: D21, D25, D24/Δ6
- **`onGeofenceEnter(tripId: TripId, poiId: CanonicalPoiId, at: Instant): ArrivalPrompt?`** — 포그라운드 지오펜스 진입 신고 → '도착하셨나요?' 확인 프롬프트 생성(프롬프트만 — 자동 기록 없음).
  - 가드: 위치 권한 없음/저정확도면 프롬프트 미생성(수동 체크인 대체). 백그라운드 위치 미사용(G62).
  - 스토리: US-E6-01 · 결정: D23, D27, G62
- **`confirmArrival(tripId: TripId, slotId: SlotId, via: Prompt | Manual): VisitState`** — 도착 확정(항상 사용자 탭) — 방문 '진행 중' 전이, 기록 진입점 활성.
  - 스토리: US-E6-01 · 결정: D23
- **`completeVisit(tripId: TripId, slotId: SlotId): VisitTransition`** — 방문 완료 — `M12.checkVisit` 위임(actual 기록), 실제 체류 시간 측정 확정, M9에 체류 신호 공급.
  - 스토리: US-E6-01, US-E8-01 · 결정: D23, Δ7
- **`skipVisit(tripId: TripId, slotId: SlotId): VisitTransition` / `undoVisitAction(tripId: TripId, slotId: SlotId): VisitTransition`** — 장소 스킵·방문 완료 취소.
  - 스토리: US-E6-01 · 결정: D23
- **`getPlaceContext(tripId: TripId, slotId: SlotId): PlaceContextCard`** — 현장 장소 카드 — 영업시간·예상 체류(추정 표기·출처)·주변 추천(M7)·다음 일정까지 여유 시간(계획 시작 시각까지 단순 차이 — 이동시간 미반영), 혼잡도 '미확인' 고정.
  - 스토리: US-E6-02 · 결정: G67, G199, D25
- **`getNextLegDistance(tripId: TripId, from: GeoPoint?): LegDistance`** — 현재 위치→다음 예정지 거리(거리 기반 추정·추정 표기) — 화면 진입/포커스 시 1회+수동 새로고침.
  - 가드: 위치 불가 시 등록 숙소/직전 장소 기준 대체+가정 표기. 내부적으로 `C2.estimateTravel` 사용하되 **거리만 반환**.
  - 스토리: US-E6-03 · 결정: D25, G65
- **`buildExternalMapHandoff(tripId: TripId, slotId: SlotId): MapHandoff`** — 외부 지도 앱 위임 페이로드(출발·도착 좌표) 생성 — 설치 앱 목록 시트는 클라이언트.
  - 실패·폴백: 외부 앱 없음 → 웹 지도 폴백 URL, 그것도 불가 → 거리 요약 최종 안내(소요시간 없음).
  - 스토리: US-E6-03, US-E5-06 · 결정: G66, D25
- **`handleExternalMapReturn(tripId: TripId, coord: GeoPoint?): ReturnAction`** — 외부 지도 복귀 처리 — 다음 예정지 근접 시 도착 확인 프롬프트 반환, 아니면 허브 복귀.
  - 스토리: US-E6-03 · 결정: G66, D23
- **`endTripManually(tripId: TripId): TripEnded`** — '여행 종료' 수동 버튼 — 종료 전이+회고 생성 트리거(M13).
  - 스토리: US-E7-01, US-E8-08, US-E09-07 · 결정: D19/Δ4
- **`autoEndTrips(): List<TripEnded>`** — (스케줄러 전용) 종료일 다음날 00:00 자동 종료 배치(숙소 유무 무관 단일 규칙).
  - 스토리: US-E8-08, US-E09-04 · 결정: D19/Δ4
- **`getTripProgress(tripId: TripId): TripProgress`** — 방문 체크 완료 비율(여행 중 진행률) — 홈 카드·여행 목록 공급.
  - 스토리: US-E2-02, US-E09-07 · 결정: G3

## M15 Community `[후속]` — 공개 스냅샷·반응·모더레이션

> 후속 출시 게이트. 공개 스냅샷 모델(D16)·기본 비공개·모더레이션 인프라(D35)는 1차 데이터 모델에 선반영. 개요 수준 정의 — 상세는 해당 유닛 Functional Design.

- `publishItinerary(tripId, options: PublishOptions): PublishedPost` — 확정/종료 일정의 게시 시점 스냅샷 공개 — 권역 마스킹·상대 날짜(G84)·EXIF GPS 기본 제거(G185)·캡션/해시태그 검증(G85, C3) · US-E11-05 · D16
- `updatePublishedSnapshot(postId): PublishedPost` / `unpublish(postId): Unit` — 명시적 공개본 업데이트·즉시 철회(재공개 시 식별자 재사용 금지) · US-E11-05, US-E11-07 · D16
- `browseFeed(filters, page): Page<PostCard>` — 공개 일정 피드(필터·정렬만, 키워드 검색 없음 — C13; 차단·검토 중 비노출) · US-E11-01 · G84, C13
- `getPostDetail(postId): PostDetail` / `getPublicProfile(accountId): PublicProfile` — 읽기전용 상세(plan만 — actual·GPS·개인 메모 비노출)·공개 프로필(소셜 그래프 없음) · US-E11-02, US-E11-03 · D16, ADR-0013
- `cloneToMyTrip(postId, myDates: DateRange): ClonedTrip` — 스냅샷 독립 복제(날짜 입력 필수 G156, 출처 표기 보존, D21 겹침 검증 경유) · US-E11-06 · G156, D21
- `likePost(postId): LikeState` / `addComment(postId, text): Comment` / `deleteComment(commentId): Unit` — 반응(토글 정규화·300자·오래된순·금칙어·묶음 알림은 M14 위임) · US-E11-08, US-E11-09 · G85, D18
- `reportContent(targetRef, reason): ReportReceipt` / `hideAuthor(accountId): Unit` — 신고(N회 누적 자동 보류 G178)·양방향 숨김 · US-E11-04 · G86, G178
- `getReportQueue(filter): Page<ReportCase>` / `resolveReport(caseId, action: Hold|Restore|Delete): ModerationResult` — 운영자 전용(별도 운영자 인증·처리 이력 기록·단계적 제재 G179) · US-E11-10 · D35, G179

## M16 Assistant `[후속]` — 대화 오케스트레이션

> 후속 출시 게이트. LLM 아키텍처(D11)·서버 재조회 권한 경계(D31)만 1차 선반영. 어시스턴트는 통역·중개자 — 확정은 항상 솔버·기존 모듈 소유.

- `openSession(screenContext: ScreenContext): AssistantSession` — 화면 컨텍스트(현재 여행·숙소·일정 상태) 자동 전달 세션 개시 — 컨텍스트는 ID만, 서버 재조회 주입(D31) · US-E10-01 · D22/Δ5
- `sendMessage(sessionId, utterance: String): AssistantResponse` — 자연어 처리 — 재질의는 04 필터 축으로 변환(Δ9), 가드레일(주입·유출 거절), rate-limit, 모든 응답에 다음 행동 1개 동봉 · US-E10-02, US-E10-06, US-E10-07 · D11, D31, Δ9
- `undoLastInterpretation(sessionId): AssistantResponse` — 직전 조건 변환 롤백 · US-E10-02 · ADR-0011
- `reviewProgress(sessionId): ReviewSummary` — 진행 검토 — 솔버 검증 위반 정보만 인용(미검증 수치 생성 금지), 거리만 표시(D25) · US-E10-03 · ADR-0008·0015
- `proposeAction(sessionId, intent): ActionProposal` / `applyProposal(sessionId, proposalId): DelegatedResult` — 제안 생성과 명시적 수락 시 기존 모듈(M8 편집/M10 재계획/D20 해제 흐름) 위임 실행 — 단독 확정 금지, 변경 전후 비교 동봉 · US-E10-04 · D20, G18
- `getThread(tripId): ConversationThread` — 여행 단위 스레드 1개, 생명주기 연동 보관, changelog 단방향 참조(G13) · US-E10-05 · G135/G197
- (폴백 규약) LLM/솔버/외부 실패 시 기존 비대화형 흐름 지속 — 첫 응답 3초·전체 15초 목표(D38), 어시스턴트 문구·솔버 검증 불일치 시 검증값 우선 · US-E10-08 · ADR-0011

## M17 Collaborative Editing `[후속]` — 초대·잠금·동기화

> 후속 출시 게이트(ADR-0016). 서버 권위+낙관적 잠금·WebSocket·최대 10명·수렴 3초(D30)는 1차 설계 확장 여지로만 반영.

- `createInvite(tripId, role: Editor|Viewer): InviteLink` / `revokeInvite(inviteId): Unit` — 다회용·만료 7일 초대 링크(권한 지정) 발급·회수 · US-E12-01 · G91
- `acceptInvite(token): Participation` — 수락 즉시 참여(비로그인 진입은 로그인 게이트 후 재개 — D22/Δ5), 최대 10명 · US-E12-01 · D30
- `updateParticipantRole(tripId, accountId, role): Participation` / `removeParticipant(tripId, accountId): Unit` — 소유자 전용 권한 변경·제거(서버 측 강제 — SECURITY-08, 공유 범위 C1: plan+숙소 위치·날짜만) · US-E12-02 · C1
- `acquireLock(tripId, itemRef): LockLease` / `heartbeat(leaseId)` / `releaseLock(leaseId)` — 항목 단위 soft lock(TTL 60~120초, 광역 편집은 day 승격 G89, 소유자 강제 해제) · US-E12-03 · D30, G89
- `submitEdit(tripId, itemRef, edit, baseVersion): EditOutcome` — 항목별 버전 낙관적 잠금 저장+솔버 재검증(편집자는 위반 시 AI 자동 보정만 — G90) · US-E12-03, US-E12-05 · D30, G90
- `resolveConflict(conflictId, chosen: VersionRef): Resolution` — 오프라인 재연결 충돌 — 감지 당사자 선택+서버 직렬화 선착 적용 · US-E12-04, US-E12-06 · G93
- `getEditHistory(tripId): List<ChangeEntry>` / `revertChange(entryId): EditOutcome` — 공동 편집 이력(ADR-0013 통합)·되돌리기(솔버 재검증 경유) · US-E12-07 · G132
- `endSharing(tripId): Unit` / `leaveTrip(tripId): Unit` — 공유 종료(소유자 단독 귀속 복귀)·자진 이탈 — 알림은 초대·권한 제거·종료만(G95/C8), 현장 실행 액션은 소유자 전용(G94) · US-E12-08 · G94, G95/C8, D18

## C1 LLM Gateway — 공통 LLM 호출 계층

단일 벤더 관리형 API 서버 경유(D11), 기능별 모델 티어 라우팅, 서버 재조회 컨텍스트 주입(D31)을 소유한다. 모듈은 LLM을 직접 호출하지 않는다.

- **`call<T>(feature: LlmFeature, contextRefs: List<ResourceRef>, prompt: PromptSpec, schema: OutputSchema<T>): TypedResult<T>`** — 기능별 LLM 호출 단일 진입점. `feature = PreferenceScoring | Reflection | Explanation | Requery(후속) | Conversation(후속)`.
  - 입력: 컨텍스트는 **리소스 ID 참조만** — 게이트웨이가 요청자 권한으로 서버 재조회해 주입(D31). 티어: 취향 해석·재질의=경량, 회고·설명=상위(D11). 출력: 스키마 검증 통과한 타입 결과.
  - 가드: 타 사용자 비공개 데이터·내부 지표(제휴 수수료 등)는 구조적으로 입력 미포함(SECURITY-11). 국외 이전 최소화 — 전송 필드 목적 최소화(G181). 사용자별 rate-limit.
  - 실패·폴백: 스키마 검증 실패 → 제한 재시도 후 `LlmOutputInvalid` — 호출측 결정론 폴백 유도. 타임아웃/장애 → `UpstreamUnavailable` — **모든 호출측은 LLM 없이 기능 지속 가능해야 함**(ADR-0011).
  - 스토리: US-E5-02, US-E5-05, US-E8-06, US-E8-08, US-E8-09, US-E7-04(설명), 후속 E10 전체 · 결정: D11, D31, SECURITY-11
- **`resolveContext(requester: Principal, refs: List<ResourceRef>): InjectedContext`** — 권한 검증 재조회 주입기(모듈 공용) — 권한 밖 참조는 조용히 제외가 아니라 `PermissionDenied`.
  - 스토리: (D31 구현 기반) · 결정: D31, SECURITY-08
- **`checkRateLimit(accountId: AccountId, feature: LlmFeature): RateLimitDecision`** — 사용자·기능별 상한 판정(사전 차단용).
  - 스토리: US-E10-06(후속), 1차 생성·회고 남용 방지 · 결정: D11, SECURITY-11
- **`getUsageHealth(): LlmHealth`** — 벤더 상태·쿼터 소진율(80% 알람 입력 — RESILIENCY-09) — 폴백 선제 전환 판단.
  - 스토리: US-E5-09(폴백 기반) · 결정: RESILIENCY-09·10

## C2 Solver Engine — 최적화·하드 제약 검증

OPTW/TOPTW 최적화, 하드 제약 검증, 거리 기반 이동시간 추정(내부), 결정론적 폴백을 소유한다. M8·M10 공용.

- **`solve(problem: ItineraryProblem): ItinerarySolution`** — 후보 선택·순서·시각 배치 최적화.
  - 입력: `ItineraryProblem{days: List<DayContext{window, base(open-ended 전환일 G50)}, candidates: CandidatePool(+LLM 선호 점수 보상값), fixedBlocks(숙소·시각 고정 방문지·LOCK·완료 방문지), mustInclude(포함 고정형), stayDurations(min/rec/max), prefs(페이스·이동 수단), budgetWeight(소프트 — G47)}`. 출력: 하드 제약 전부 만족하는 해(위반 배치는 결과에서 구조적 배제).
  - 가드: 하드 제약 — 영업시간 내 배치·이동시간 부등식(직전 종료+이동≤다음 시작, 안전계수 G106)·고정 블록 불변·일수 귀속(자정 초과=시작일, D29). 예산은 하드 제약 아님(G47). 사용자 노출 시각은 본 엔진 검증값만(LLM 임의 시각 금지). 하루 1개 이하만 가능하면 억지 채움 없이 사유 신호.
  - 실패·폴백: LLM 점수 부재 시 규칙 점수 기반 **결정론적 폴백 모드**로 항상 해 산출 가능(무작위성 제거·시드 고정 — PBT 대상).
  - 스토리: US-E5-01~04, US-E5-09, US-E5-10, US-E7-04, US-E7-07, US-E4-09 · 결정: D28, D29, G47, G50, G106
- **`validate(itinerary: ItineraryLike, constraints: ConstraintSet): List<Violation>`** — 임의 일정(편집 결과·대안·확정 직전 재검증)의 하드 제약 검증 — 서버 확정 검증 단일 진입점.
  - 가드: 확정 버튼 시점 재검증(G56)도 본 메서드 재사용. 결정적 — clock 주입 가능(G116).
  - 스토리: US-E5-07, US-E7-04, US-E7-08, US-E4-08 · 결정: D28, G56
- **`repair(itinerary: ItineraryLike, violations: List<Violation>, policy: MinimalChange): RepairResult`** — 'AI 자동 보정' — 위반 구간의 **시각·순서만** 최소 조정(POI 추가/삭제 없음).
  - 실패·폴백: 보정 불가 → `RepairInfeasible(reasons)` — 호출측은 '그대로 저장' 또는 수동 수정 유도.
  - 스토리: US-E5-07 · 결정: G49
- **`estimateTravel(from: GeoPoint, to: GeoPoint, mode: TransportMode): TravelEstimate`** — 거리 기반 이동 추정 — TMap 도로 거리(폴백: 직선×우회계수 1.3)+수단별 속도·안전계수(대중교통 ×1.5, 도보 ×1.4, 버퍼 15분 — remote config).
  - 출력: `TravelEstimate{distance: DistanceRange, internalDuration: Duration}` — **`internalDuration`은 솔버 부등식·트리거 판정 내부 계산 전용, 어떤 표시용 DTO에도 매핑 금지**(D25/Δ1). 표시 계층엔 `distance`만 전파.
  - 실패·폴백: 라우팅 API 실패 → 직선거리 추정 폴백+추정 플래그(호출측 표기).
  - 스토리: US-E5-03, US-E5-06, US-E6-03, US-E7-03(전부 내부 계산 경유) · 결정: D25/Δ1, G106, D08
- **`exportClientValidationSpec(): ValidationRuleSpec`** — 클라이언트 경량 검증기(RN `shared/validation`)와 공유할 제약 규칙 명세(버전 포함) 내보내기 — 편집 시 즉시 검증과 서버 확정 검증의 규칙 단일 정본화.
  - 스토리: US-E5-07 · 결정: D28

## C3 Content Moderation — 금칙어·텍스트 검증

금칙어 사전 검증(1차: 닉네임·여행 제목, 확장: 메모·캡션·댓글)을 소유한다.

- **`checkText(text: String, context: ModerationContext): ModerationResult`** — 금칙어 사전 매칭 검증. `context = Nickname | TripTitle | Memo | Caption(후속) | Comment(후속)`.
  - 출력: `ModerationResult = Pass | Blocked(matchedCategories)` — 매칭 원문은 응답에 미포함(우회 학습 방지).
  - 가드: 가입 시점 닉네임 검증부터 적용(G23). 후속 커뮤니티는 텍스트 자동 필터만 — 이미지·스팸은 신고 기반(G86).
  - 실패·폴백: 사전 로드 실패 시 fail-closed(저장 차단) — 닉네임 자동 생성 경로는 사전 검증된 어휘풀이므로 영향 없음.
  - 스토리: US-E1-03, US-E1-12, US-E4-11, US-E8-13(캡션), 후속 US-E11-05·09 · 결정: G23, G86, N6
- **`checkTextBatch(items: List<TextItem>): List<ModerationResult>`** — 다필드 폼(제목+메모 등) 일괄 검증.
  - 스토리: (위와 동일 폼 최적화)
- **`getDictionaryVersion(): DictVersion`** — 배포된 금칙어 사전 버전 조회(운영 갱신·P8 선결 과제 추적).
  - 스토리: (운영) · 결정: G23, P8

---

## 스토리 커버리지 매트릭스

> 1차 102개 스토리 전수 — 스토리별 담당 메서드. "클라이언트 전용"은 서버 퍼사드가 불필요한 순수 UI 스토리(앱 셸 컴포넌트 소유). 후속 26개는 모듈 수준 매핑.

### Epic 1 — 가입·온보딩 (18)

| 스토리 | 담당 메서드 |
|---|---|
| US-E1-01 | M1.signUpWithSocial · M1.signUpWithEmail · M1.verifyEmail · M1.resendVerificationEmail · M1.signIn · M1.requestPasswordReset/resetPassword |
| US-E1-02 | M1.recordConsent |
| US-E1-03 | M2.generateNickname · M2.updateNickname |
| US-E1-04 | M1.updateLocationConsent/getLocationConsentState (OS 다이얼로그·프리프롬프트는 클라이언트 location 계층) |
| US-E1-05 | M2.updatePreferences |
| US-E1-06 | M2.updatePreferences (예산 축 — 총액 원값+구간 저장) |
| US-E1-07 | M2.updatePreferences (동행 단일+반려동물 불리언) |
| US-E1-08 | M2.updatePreferences |
| US-E1-09 | M2.updatePreferences (이동 방식 — 내부 계산 한정 반영) |
| US-E1-10 | M2.updatePreferences |
| US-E1-11 | M2.getOnboardingState/completeOnboarding · M2.getPendingPreferencePrompts |
| US-E1-12 | M2.updateNickname · M2.updatePreferences |
| US-E1-13 | M2.getPersonalizationInput (M3 필터 기본값·M8 생성 입력·M7 추천 반영) |
| US-E1-14 | M2.getPreferences (중립 기본값) · M8.generateItinerary (무실패 보장) |
| US-E1-15 | M2.updatePreferences (페이스 축) |
| US-E1-16 | M1.signUpWithSocial · M1.signUpWithEmail (ageConfirmation 가드) |
| US-E1-17 | M1.recordConsent (위치 약관 분리 필수) · M1.updateLocationConsent · M12.purgeLocationData |
| US-E1-18 | M1.getRequiredConsents · M1.reconsent |

### Epic 2 — 앱 셸·홈·내비게이션 (6)

| 스토리 | 담당 메서드 |
|---|---|
| US-E2-01 | M1.getBootstrapStatus · M1.getRequiredConsents · M2.getOnboardingState |
| US-E2-02 | M6.listTrips · M18.getActiveHub/getTripProgress · M7.getTrendingPlaces · M12.getVisitStats · M14.getInbox(미읽음 배지) · M2.getPendingPreferencePrompts |
| US-E2-03 | M18.getActiveHub · M8.getItinerary · M6.listTrips (탭 상태 보존·딥링크 라우팅 G6/G7은 클라이언트) |
| US-E2-04 | 클라이언트 전용 (앱 셸 탭바 노출 규칙 — 서버 메서드 불요) |
| US-E2-05 | M7.savePoi/unsavePoi · M7.searchPoi · M7.listSavedPois · M6.addMustVisit(시드 투입) |
| US-E2-06 | M1.getBootstrapStatus (minSupportedVersion·forceUpdate) |

### Epic 3 — 숙소 탐색·저장·등록 (11)

| 스토리 | 담당 메서드 |
|---|---|
| US-E3-01 | M3.searchStays · M3.getLivePrice |
| US-E3-02 | M3.searchStays (filters·zeroResultDiagnosis) |
| US-E3-03 | M3.getStayDetail |
| US-E3-04 | M3.addToWishlist · M3.updateWishlistMemo · M3.removeFromWishlist · M3.listWishlist |
| US-E3-05 | M5.getOtaLinkOptions · M5.buildDeeplink · M5.recordOutboundClick · M5.getReturnHandoffCard/dismissHandoffCard |
| US-E3-06 | M4.registerStay · M4.updateStay/deleteStay |
| US-E3-07 | M4.linkToTrip (거점 간 비중첩 검증) |
| US-E3-08 | M4.registerStay (3경로) · M4.parseOtaLink · M7.searchPoi(장소 검색 경로) |
| US-E3-09 | M4.listMyStays |
| US-E3-10 | M3.searchStays (zeroResultDiagnosis·직접 등록 우회 안내) |
| US-E3-11 | M3.searchStays (degraded 부분 실패 플래그) |

### Epic 4 — 여행 생성·거점·필수 방문지 (11)

| 스토리 | 담당 메서드 |
|---|---|
| US-E4-01 | M6.createTrip · M6.updateTripWindow · M2.getPendingPreferencePrompts |
| US-E4-02 | M6.createTrip (숙소 미등록 상태 허용) · M8.generateItinerary(NO_BASE 가드) |
| US-E4-03 | M4.linkToTrip · M4.registerStay · M4.updateStay/deleteStay |
| US-E4-04 | M3.listWishlist · M4.registerStay(프리필)+linkToTrip |
| US-E4-05 | M4.proposeTripDatesFromStay · M6.updateTripDates |
| US-E4-06 | M4.linkToTrip · M4.getBaseTimeline (스마트 기본 거점) |
| US-E4-07 | M4.linkToTrip (연속 숙박 단일 등록) · M4.getBaseTimeline (N박 체류 표시 데이터) |
| US-E4-08 | M6.addMustVisit/updateMustVisit/removeMustVisit · M6.previewMustVisitChange · M7.searchPoi/listSavedPois |
| US-E4-09 | C2.solve (fixedBlocks·mustInclude 불변) · M8.regenerate (warm-start 보존) |
| US-E4-10 | M5.buildDeeplink (정본 재사용) · M4.registerStay |
| US-E4-11 | M6.createTrip (자동 제목) · M6.updateTrip · C3.checkText |

### Epic 5 — AI 일정 생성·확정 (12)

| 스토리 | 담당 메서드 |
|---|---|
| US-E5-01 | M8.generateItinerary · M4.getBaseTimeline · C2.solve |
| US-E5-02 | M7.getCandidatePool · C1.call(PreferenceScoring) · C2.solve (소프트 예산 가중치) |
| US-E5-03 | C2.solve/validate · M7.getStayDurationDefaults · C2.estimateTravel(내부) |
| US-E5-04 | C2.solve (고정 블록) · M6.addMustVisit · M8.generateItinerary (충돌 시 사용자 선택 유도) |
| US-E5-05 | M8.getExplanations · C1.call(Explanation) |
| US-E5-06 | M8.getItinerary (2보기 공용 데이터·거리 요약만) · M18.buildExternalMapHandoff(길찾기 위임) |
| US-E5-07 | M8.validateEdit · M8.applyEdit · M8.getInsertableSlots · C2.repair · C2.exportClientValidationSpec |
| US-E5-08 | M8.applyEdit(저장) · M18.getActiveHub (실행 연계) |
| US-E5-09 | M8.generateItinerary(폴백 체인) · M8.streamGenerationProgress · M8.cancelGeneration/resumeGeneration |
| US-E5-10 | M8.generateItinerary(mode) · M8.getSlotCandidates/chooseSlotCandidate · M8.switchGenerationMode · M8.lockSlot/replaceSlot · M8.regenerate |
| US-E5-11 | M8.recommendStayZone · M4.registerStay(추천 숙소 등록) |
| US-E5-12 | M8.confirmItinerary · M8.unlockForEdit · M8.getItinerary(확정본 열람) |

### Epic 6 — 여행 중 현장 실행 (3)

| 스토리 | 담당 메서드 |
|---|---|
| US-E6-01 | M18.onGeofenceEnter · M18.confirmArrival · M18.completeVisit · M18.skipVisit/undoVisitAction · M12.checkVisit(위임) · M9.reportClientSignal(체류 신호) |
| US-E6-02 | M18.getPlaceContext · M7.getPoi · M7.getNearbyRecommendations |
| US-E6-03 | M18.getNextLegDistance · M18.buildExternalMapHandoff · M18.handleExternalMapReturn |

### Epic 7 — Plan-B 재계획 (13)

| 스토리 | 담당 메서드 |
|---|---|
| US-E7-01 | M10.startReplan (사유 5종+없음, '여행 중' 가드) |
| US-E7-02 | M9.evaluateServerTriggers · M9.reportClientSignal · M9.getActiveTriggerBanners · M9.dismissTrigger · M9.updateSensitivity · M11.getForecast/getActiveAlerts/getPrecipitationRisk · M7.refreshOpeningHours · M14.sendEvent |
| US-E7-03 | M10.startReplan (영향 분석 입력 구성) · C2.estimateTravel(내부) |
| US-E7-04 | M10.getAlternatives · M7.getCandidatePool(저장 우선 소싱) · C2.validate |
| US-E7-05 | M10.getAlternatives (Alternative 표시 필드 — 거리·체류·여유, 소요시간 없음) |
| US-E7-06 | M10.keepCurrentPlan · M10.enterRestMode/resumeFromRest · M9.getSuppressionState |
| US-E7-07 | M10.confirmAlternative (당일 잔여 재정렬) · M10.getUnplacedPois/rescheduleUnplaced |
| US-E7-08 | M10.previewAlternative · M10.confirmAlternative (G56 재검증) |
| US-E7-09 | M12.appendChangeLog · M10.confirmAlternative(기록 트리거) · M12.getTripTimeline |
| US-E7-10 | M10.setManualPosition |
| US-E7-11 | M10.applyManualEdit (숙소 고정 제약 불가침) |
| US-E7-12 | M10.startReplan(method) · M10.getAlternatives(폴백 3종) · M10.applyManualEdit |
| US-E7-13 | M12.appendGpsTrack · M12.getRouteComparison |

### Epic 8 — 여행 기록·회고 (14)

| 스토리 | 담당 메서드 |
|---|---|
| US-E8-01 | M12.checkVisit · M12.updateVisitRecord · M12.addImpromptuVisit · M9.reportClientSignal(체류 초과 연결) |
| US-E8-02 | M12.attachMedia |
| US-E8-03 | M1.updateLocationConsent · M12.appendGpsTrack · M12.purgeLocationData |
| US-E8-04 | M12.getTripTimeline · M12.appendChangeLog |
| US-E8-05 | M12.getTripRecords · M4.getBaseTimeline (귀속 기준) |
| US-E8-06 | M13.generateDailyReflection · C1.call(Reflection) |
| US-E8-07 | M13.editReflection · M13.regenerateReflection |
| US-E8-08 | M18.endTripManually/autoEndTrips(트리거) · M13.generateTripSummary · M12.getVisitStats(G72 합산) |
| US-E8-09 | M13.analyzeTravelStyle · M12.getVisitStats(10곳 게이트) |
| US-E8-10 | M2.getPersonalizationInput · M2.updatePersonalizationConsent |
| US-E8-11 | M12.listArchivedTrips · M12.getTripRecords/getTripTimeline · M13.regenerateTripSummary · M12.updateVisitRecord(종료 후 편집) |
| US-E8-12 | M12.syncOfflineRecords · M12.resolveSyncConflict |
| US-E8-13 | M13.buildShareCard |
| US-E8-14 | M12.getRecordCalendar · M12.listArchivedTrips |

### Epic 9 — 알림·마이페이지·설정 (14)

| 스토리 | 담당 메서드 |
|---|---|
| US-E09-01 | M14.sendEvent (M4.registerStay/M3.addToWishlist 이벤트 발행) |
| US-E09-02 | M14.scheduleTripNotifications · M14.rescheduleOnItineraryChange · M14.cancelTripReminders · M14.updateQuietHours |
| US-E09-03 | M14.sendEvent (Plan-B, 10분 중복 제한) · M9.updateSensitivity · M9.getActiveTriggerBanners |
| US-E09-04 | M14.sendEvent (M13 회고 완료 이벤트) |
| US-E09-05 | M14.getSettings/updateToggles · M14.getInbox/markRead · M14.registerDeviceToken |
| US-E09-06 | M4.listMyStays · M4.linkToTrip/unlinkFromTrip · M8.regenerate(재생성 확인) · M14.cancelTripReminders |
| US-E09-07 | M6.listTrips · M18.getTripProgress |
| US-E09-08 | M13.getStyleSummaryCard · M13.analyzeTravelStyle |
| US-E09-09 | M2.getProfileSummary/updateBasicInfo · M2.exportUserData · M1.requestAccountDeletion/cancelAccountDeletion |
| US-E09-10 | M2.updatePreferences |
| US-E09-11 | M1.updateLocationConsent/getLocationConsentState · M12.purgeLocationData |
| US-E09-12 | M5.buildDeeplink(고지 동봉) · M5.setDisclosureHidden |
| US-E09-13 | M1.getPolicyDocuments (오픈소스 라이선스는 클라이언트 번들) |
| US-E09-14 | M1.updateMarketingConsent · M1.recordConsent |

### 후속 에픽 — 모듈 수준 매핑 (26)

| 에픽 | 스토리 | 담당 모듈 (보조) |
|---|---|---|
| E10 AI 어시스턴트 | US-E10-01 ~ US-E10-08 (8) | **M16 Assistant** (C1 호출·D31 경계, M8/M10 위임 실행, C2 검증값 인용) |
| E11 여행자 커뮤니티 | US-E11-01 ~ US-E11-10 (10) | **M15 Community** (C3 텍스트 필터, M14 묶음 알림, M6 복제 시 D21 검증, D35 운영자 도구 포함) |
| E12 동행 공동 편집 | US-E12-01 ~ US-E12-08 (8) | **M17 Collaborative Editing** (C2 재검증, M12 changelog 통합, M14 초대·종료 알림) |

### 커버리지 검증 요약

- [x] 1차 102개 스토리 전수 → 메서드 매핑 완료 (누락 0. US-E2-04만 클라이언트 전용으로 서버 메서드 불요 판정 — 앱 셸 UI 규칙)
- [x] 후속 26개 스토리 → 모듈 수준 매핑 완료 (M15: 10, M16: 8, M17: 8)
- [x] 전역 규칙 반영 확인 — D25(estimateTravel 내부 전용·전 표시 DTO 거리만) · D22(공통 규약 1: 전 퍼사드 인증 필수) · D21(createTrip·updateTripDates·cloneToMyTrip 하드 가드) · SECURITY-08(공통 규약 1: 소유권 검증) · ADR-0011(전 메서드 실패·폴백 명시, 침묵 실패 금지)
