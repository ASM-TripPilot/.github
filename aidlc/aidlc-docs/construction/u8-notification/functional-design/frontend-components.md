# U8 알림·마이페이지·설정 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/notification`(인앱 알림함·딥링크 라우팅·푸시 프리프롬프트) + `features/settings`(마이페이지·알림 설정·계정 관리·위치 3층·마케팅·취향·정책 재열람)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `notification`·`settings` 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) §M14 + 각 모듈 설정 퍼사드(M1·M2·M4·M6·M12·M13), 폼·검증 규칙은 [business-rules.md](./business-rules.md) BR-U8-xx의 클라이언트측 대응(판정 정본은 항상 서버 — 발송 파이프라인·삭제 연쇄는 서버 M14·M1, 마이페이지 집계는 서버 BFF)이다.
> 기술 중립: 상태 관리·내비게이션·스타일·FCM 토큰 브리지 구현체는 NFR/Code Generation 소유. **홈 대시보드·5탭 셸은 U2 `features/home`·내비게이션 컨테이너 재사용**(U8은 신규 셸 미생성 — 미읽음 배지 공급+최종 통합 검증만). **기록·회고 상세 화면은 U7 `features/archive` 재사용**(마이페이지에서 진입).
> data-testid 규약: `settings-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 알림·설정 화면 플로우 (정본)

```text
[CP5 이벤트(U3·U5·U6·U7) → M14 서버 스케줄·발송] · [5탭 '마이' 탭(U2 셸) → 마이페이지·설정]
        │
        ▼
┌─ NotificationInbox ('알림함' — 딥링크 진입·상단 배지 U2 topBar) ───────────────────────────┐
│   Notice 목록(90일·읽음 G99) · 미읽음 배지 · 딥링크 라우팅(탭 활성화+스택 푸시 G7) · 0건 빈 상태 │
└──────────────────────────────────────────────────────────────────────────────────────────┘
   │ 알림 탭 → 딥링크 대상(execution/archive/planb)
   ▼
┌─ MyPageHome ('마이' 탭 루트 — 숙소·여행·스타일 요약 진입) ──────────────────────────────────┐
│   StaySection(통합 목록 G103) · TripSection(예정/진행/종료 D19·D21) · StyleSummaryCard(10곳 게이트) │
└──────────────────────────────────────────────────────────────────────────────────────────┘
   │ 숙소 관리         │ 여행 목록         │ 스타일 상세        │ 설정                │ 계정 관리
   ▼                  ▼                  ▼                   ▼                    ▼
StayManageScreen   TripListScreen    TravelStyleScreen   SettingsHome         AccountManageScreen
(출발점 전환 G97)   (구분 조회)        (U7 archive 재사용)  (알림·위치·취향 진입)  (삭제 D18·내보내기 G101)
   │ 출발점 전환                                            │
   ▼                                                       ├─ NotificationSettingsScreen (토글·방해금지·오프셋)
RegenConfirmDialog                                         ├─ LocationConsentScreen (3층 G182·철회 영향 고지)
(재생성 확인·배지 G97)                                      ├─ PreferenceSettingsScreen (취향·예산 총액 D26)
                                                           ├─ MarketingConsentToggle (N8)
                                                           ├─ AffiliateNoticeToggle (제휴 고지 다시 보기)
                                                           └─ PolicyDocsScreen (약관 3종·오픈소스·문의 N5)
```

**전역 규칙**
- 알림·설정 화면군은 5탭 셸(U2) 내 '마이' 탭 스택 + 알림함(딥링크·상단 배지 진입)에서 접근한다. 홈 대시보드·5탭 셸은 U2 재사용(신규 셸 미생성).
- **모든 리마인드·알림 본문은 이동 거리만·소요시간 없음**(D25 승계). 스케줄·발송 판정 정본은 서버 M14.
- **알림 설정 변경은 별도 저장 버튼 없이 즉시 저장**(다음 알림부터 반영 BR-U8-10). OS 푸시 권한 거부 시 푸시 토글 비활성+설정 이동 안내.
- **인앱 알림함은 90일 보존·읽음** — 푸시 OFF·방해금지·중복 억제분도 적재(채널 독립 BR-U8-09). 미읽음 배지는 U2 topBar 슬롯에 공급.
- **위치 철회는 영향 고지+재확인 후에만**(BR-U8-18) — 철회 후 위치 비의존 기능(예정 알림·날씨·휴무)은 유지. OS 권한 거부 시 앱 토글 비활성.
- **계정 삭제는 삭제 대상 사전 고지+재확인**(BR-U8-13) — 소프트 삭제·30일 유예. 데이터 내보내기는 JSON 즉시·사진 제외.
- **마이페이지 집계는 읽기 전용**(각 모듈 퍼사드) — 스타일 10곳 게이트·여행 분류·숙소 통합 목록 정본은 상류 모듈(재정의 없음).

## 2. 컴포넌트 계층

```text
features/notification
├─ NotificationNavigator                   — 알림 스택(딥링크 진입·상단 배지 진입)
│  ├─ NotificationInbox                     — 인앱 알림함(90일·읽음 G99)
│  │  ├─ NoticeListItem                     — 알림 항목(종류 아이콘·본문·읽음 상태·시각)
│  │  ├─ UnreadBadge                        — 미읽음 카운트(U2 topBar 슬롯 공급)
│  │  └─ InboxEmptyState                    — 0건 빈 상태
│  ├─ DeeplinkRouter                        — 알림 탭 → 대상 탭 활성화+스택 푸시(G7)
│  └─ PushPrepromptSheet                    — 푸시 권한 프리프롬프트(OS 다이얼로그 전 안내)

features/settings
├─ SettingsNavigator                        — 마이페이지·설정 스택('마이' 탭 U2 셸 진입)
│  ├─ MyPageHome                            — 마이페이지 루트(숙소·여행·스타일 요약)
│  │  ├─ StaySectionCard                    — 숙소 통합 요약(→ StayManageScreen)
│  │  ├─ TripSectionCard                    — 여행 요약(예정/진행/종료 → TripListScreen)
│  │  └─ StyleSummaryCard                   — 스타일 요약(디스크립터·칩·dot 게이지·10곳 게이트)
│  ├─ StayManageScreen                      — 숙소·예약 관리(통합 목록·출처 라벨 G103)
│  │  ├─ StayListItem                       — 숙소 항목(명·위치·체크인/아웃·출처·연결 여행)
│  │  ├─ StartPointToggle                   — 일정 출발점 등록/해제(M4 위임)
│  │  ├─ RegisterAction                     — 저장 항목 '등록하기' 액션
│  │  ├─ RegenConfirmDialog                 — 출발점 변경 시 재생성 확인(거부 시 배지 G97)
│  │  └─ StayEmptyState                     — 0건('숙소 탐색·등록' 유도+탐색 진입)
│  ├─ TripListScreen                        — 여행 목록 구분 조회(예정/진행/종료 D19·D21)
│  │  ├─ TripListItem                       — 여행 항목(목적지·기간·숙소 수·일정 수)
│  │  ├─ ReflectionEntryLink                — 종료 여행 회고 진입(U7 archive 재사용)
│  │  └─ TripEmptyState                     — 지난 여행 0건('종료된 여행 없음'·회고 진입 숨김)
│  ├─ SettingsHome                          — 설정 루트(알림·위치·취향·계정·정책 진입)
│  ├─ NotificationSettingsScreen            — 알림 설정(종류×채널 토글·방해금지·오프셋)
│  │  ├─ PerTypeToggleRow                   — 종류별 푸시/인앱 독립 토글(즉시 저장)
│  │  ├─ QuietHoursPicker                   — 방해금지 창(22~08 기본·G100)
│  │  ├─ ReminderOffsetPicker               — 개별 일정 오프셋(0/15/30/60)·당일 요약 시각
│  │  ├─ SensitivityPicker                  — Plan-B 민감도(적게/보통/많이)
│  │  └─ OsPermissionBanner                 — OS 권한 거부 시 푸시 토글 비활성+설정 이동
│  ├─ LocationConsentScreen                 — 위치 3층 설정(OS×법정×옵트인 G182)
│  │  ├─ ThreeLayerToggleGroup              — 3층 구분 토글(용도 고지)
│  │  ├─ RevokeImpactDialog                 — 철회 영향 고지+재확인(BR-U8-18)
│  │  └─ LocationOsPermissionBanner         — OS 권한 거부 시 앱 토글 비활성+설정 이동
│  ├─ PreferenceSettingsScreen              — 취향 직접 설정(온보딩 동일 선택지·직접 설정 우선)
│  │  └─ BudgetTotalInput                   — 예산 총액(항공 제외 D26)·1인/1일 파생 표기
│  ├─ MarketingConsentToggle                — 마케팅 수신 동의(수집·철회·1차 미발송 N8)
│  ├─ AffiliateNoticeToggle                 — OTA 제휴 고지 다시 보기(재활성 US-E09-12)
│  ├─ AccountManageScreen                   — 개인정보·계정(기본 정보·내보내기·삭제)
│  │  ├─ BasicInfoEditor                    — 닉네임·이메일 조회·수정
│  │  ├─ DataExportButton                   — 데이터 내보내기(JSON 즉시·사진 제외·재인증)
│  │  ├─ AccountDeleteFlow                  — 계정 삭제(대상 사전 고지·재확인·30일 유예 D18)
│  │  └─ DeletionRecoveryNotice             — 유예 중 복구 안내(30일 내 재로그인)
│  └─ PolicyDocsScreen                      — 정책 재열람(약관 3종·오픈소스 라이선스·이메일 문의 N5)
└─ (재사용) HomeIntegrationVerifyView        — 홈 카드 최종 통합 검증(U2 HomeDashboard 슬롯 × U3~U7)
```

---

## 3. 알림 화면군 (features/notification)

### 3.1 NotificationInbox + DeeplinkRouter (인앱 알림함)
- **props**: `accountId` · **state**: 알림 목록(종류·본문·읽음·시각), 미읽음 카운트, 페이지.
- **폼 검증(서버 규칙 대응)**: 90일 보존·읽음 상태 관리(BR-U8-09). **푸시 OFF·방해금지·중복 억제분도 적재**(채널 독립). 알림 탭 시 **딥링크 대상 탭 활성화+스택 푸시**(G7). **미읽음 배지 카운트를 U2 topBar 슬롯에 공급**. 0건이면 빈 상태.
- **상호작용**: `M14.getInbox(page)` · `M14.markRead(noticeIds)`.
- **data-testid**: `settings-inbox-list`, `settings-inbox-unreadbadge`, `settings-inbox-empty`, `settings-inbox-deeplink`.
- **사용 서버 능력**: `M14.getInbox`, `M14.markRead`.

### 3.2 PushPrepromptSheet (푸시 권한 프리프롬프트)
- **props**: 없음(전역) · **state**: OS 푸시 권한 상태.
- **폼 검증(서버 규칙 대응)**: OS 다이얼로그 전 용도 안내(프리프롬프트). 권한 부여 시 `registerDeviceToken`로 기기 토큰 등록(다기기·로그아웃 시 해제). **OS 권한 거부는 오류가 아니라 안내 상태**(알림 설정 화면 토글 비활성 연동).
- **상호작용**: `M14.registerDeviceToken(deviceId, fcmToken)`.
- **data-testid**: `settings-push-preprompt`, `settings-push-permissiondenied`.
- **사용 서버 능력**: `M14.registerDeviceToken`.

---

## 4. 마이페이지 화면군 (features/settings — 마이페이지)

### 4.1 MyPageHome (마이페이지 루트)
- **props**: `accountId` · **state**: 숙소 요약·여행 요약·스타일 요약.
- **폼 검증(서버 규칙 대응)**: 읽기 전용 집계(MyPageView §10)·각 모듈 퍼사드 조립. StyleSummaryCard는 **10곳 게이트 승계**(미달 시 '현재 N곳' 안내 BR-U8-22). 여행 예정/진행/종료 요약(진행 중 최대 1개 D21).
- **상호작용**: `M4.listMyStays` · `M6.listTrips` · `M13.getStyleSummaryCard`.
- **data-testid**: `settings-mypage-staysection`, `settings-mypage-tripsection`, `settings-mypage-stylecard`.
- **사용 서버 능력**: `M4.listMyStays`, `M6.listTrips`, `M13.getStyleSummaryCard`.

### 4.2 StayManageScreen (숙소·예약 관리)
- **props**: `accountId` · **state**: 숙소 통합 목록(출처 라벨), 출발점 상태.
- **폼 검증(서버 규칙 대응)**: **저장/등록 통합 목록+출처 라벨 구분**(G103), 각 항목 숙소명·위치·체크인/아웃·연결 여행. **저장 항목 '등록하기' 액션**. **"일정 출발점으로 등록/해제" 전환**(M4 위임) — 출발점 변경 시 **재생성 확인**, 거부 시 **리마인드 중단+재생성 유도 배지**(G97 BR-U8-20). 0건이면 '숙소 탐색·등록' 유도+탐색 진입.
- **상호작용**: `M4.listMyStays` · `M4.linkToTrip/unlinkFromTrip` · `M8.regenerate`(재생성 확인) · `M14.cancelTripReminders`(거부 시 G97).
- **data-testid**: `settings-stay-list`, `settings-stay-startpointtoggle`, `settings-stay-register`, `settings-stay-regenconfirm`, `settings-stay-empty`.
- **사용 서버 능력**: `M4.listMyStays`, `M4.linkToTrip`, `M4.unlinkFromTrip`, `M8.regenerate`, `M14.cancelTripReminders`.

### 4.3 TripListScreen (여행 목록 구분 조회)
- **props**: `accountId` · **state**: 여행 목록(예정/진행/종료 분류).
- **폼 검증(서버 규칙 대응)**: **예정/진행 중/종료 분류**(종료 전이 D19·진행 중 최대 1개 D21 BR-U8-21), 각 여행 목적지·기간·등록 숙소 수·일정 수. **종료 여행 회고 진입**(U7 archive 재사용). 지난 여행 0건이면 '종료된 여행 없음' 안내+회고 진입 숨김.
- **상호작용**: `M6.listTrips` · (진입)U7 `features/archive`.
- **data-testid**: `settings-triplist-upcoming`, `settings-triplist-active`, `settings-triplist-ended`, `settings-triplist-reflectionlink`, `settings-triplist-empty`.
- **사용 서버 능력**: `M6.listTrips`.

### 4.4 TravelStyleScreen (스타일 분석 상세 — U7 재사용)
- **props**: `accountId` · **state**: 스타일 분석(metrics·mode·basedOnVisitCount).
- **폼 검증(서버 규칙 대응)**: 마이페이지 요약 카드에서 진입, **상세 분석은 U7 `TravelStyleScreen` 재사용**(10곳 게이트·임시 미리보기·게이지 정본 U7). 분석 여행 수·마지막 갱신 시점 표기.
- **상호작용**: `M13.analyzeTravelStyle` · `M13.getStyleSummaryCard` · (재사용)U7 archive.
- **data-testid**: `settings-style-summary`, `settings-style-detaillink`.
- **사용 서버 능력**: `M13.getStyleSummaryCard`, (U7)`M13.analyzeTravelStyle`.

---

## 5. 설정 화면군 (features/settings — 설정)

### 5.1 NotificationSettingsScreen (알림 설정)
- **props**: `accountId` · **state**: 종류×채널 토글, 방해금지 창, 오프셋·당일 시각·민감도, OS 권한 상태.
- **폼 검증(서버 규칙 대응)**: **종류×채널(푸시/인앱) 독립 토글**(BR-U8-10)·즉시 저장·다음 알림부터 반영. **방해금지 창(22~08 기본)**·개별 일정 오프셋(0/15/30/60)·당일 요약 시각·Plan-B 민감도(적게/보통/많이). **OS 푸시 권한 거부 시 푸시 토글 비활성**+"기기 설정에서 알림 권한 허용" 안내·설정 이동. 종류 목록 정본은 PRD 12(체크인 임박·예약 링크 리마인드 제외 Δ10).
- **상호작용**: `M14.getSettings` · `M14.updateToggles(perType)` · `M14.updateQuietHours(window)`.
- **data-testid**: `settings-noti-typetoggle`, `settings-noti-quiethours`, `settings-noti-offset`, `settings-noti-sensitivity`, `settings-noti-ospermission`.
- **사용 서버 능력**: `M14.getSettings`, `M14.updateToggles`, `M14.updateQuietHours`.

### 5.2 LocationConsentScreen (위치 3층 설정)
- **props**: `accountId` · **state**: 3층 상태(OS 권한·법정 동의·GPS 옵트인).
- **폼 검증(서버 규칙 대응)**: **3층 구분 표시·제어**(OS 권한 × 법정 동의 × GPS 옵트인 G182)·용도 고지(이동 지연·실시간 Plan-B·주변 추천). **철회 직전 영향 고지+재확인**(이동 지연 알림 중단·위치 재계획 제한·현위치 추천 비활성 BR-U8-18). 철회 후 위치 비의존 기능(예정 알림·날씨·휴무) 유지. **GPS 옵트인 철회·탈퇴 시 위치 데이터 즉시 파기**(U7 연동). OS 권한 거부 시 앱 토글 비활성+설정 이동.
- **상호작용**: (U1)위치 동의 3층 API · (U7)`M12.purgeLocationData`(옵트인 철회 시).
- **data-testid**: `settings-location-oslayer`, `settings-location-legallayer`, `settings-location-gpslayer`, `settings-location-revokeimpact`, `settings-location-ospermission`.
- **사용 서버 능력**: (U1)위치 동의 상태 조회·갱신, (U7)`M12.purgeLocationData`.

### 5.3 PreferenceSettingsScreen (취향 직접 설정)
- **props**: `accountId` · **state**: 취향 7종(선호 카테고리·일정 밀도·이동 선호·예산대).
- **폼 검증(서버 규칙 대응)**: **온보딩 동일 선택지**로 직접 입력·수정(BR-U8-23), 다음 AI 생성·재계획 입력 반영. **직접 설정 값 > 자동 스타일 분석**(충돌 시 우선). **예산 총액(항공 제외 D26) 기준 입력**·1인/1일 파생 표기.
- **상호작용**: `M2.getPreferences` · `M2.updatePreferences`.
- **data-testid**: `settings-pref-categories`, `settings-pref-budgettotal`, `settings-pref-density`.
- **사용 서버 능력**: `M2.getPreferences`, `M2.updatePreferences`.

### 5.4 MarketingConsentToggle + AffiliateNoticeToggle (동의·고지 토글)
- **props**: `accountId` · **state**: 마케팅 동의 상태, 제휴 고지 표시 여부.
- **폼 검증(서버 규칙 대응)**: **마케팅 수신 동의 현재 상태 확인·토글**(부여/철회·즉시 저장·증적·**1차 미발송** N8 BR-U8-19). **제휴 고지 다시 보기 토글**(한 번 확인 후 숨김·설정에서 재활성 US-E09-12 BR-U8-25).
- **상호작용**: (M1/M2)마케팅 동의 갱신 · `M1.updateAffiliateNoticePref`.
- **data-testid**: `settings-marketing-toggle`, `settings-affiliate-toggle`.
- **사용 서버 능력**: (M1/M2)마케팅 동의 조회·갱신, `M1.updateAffiliateNoticePref`.

### 5.5 AccountManageScreen (개인정보·계정 관리)
- **props**: `accountId` · **state**: 기본 정보(닉네임·이메일), 내보내기 상태, 삭제 유예 상태.
- **폼 검증(서버 규칙 대응)**: 닉네임·이메일 조회·수정. **데이터 내보내기(JSON 즉시·사진 제외·본인 재인증·rate-limit** BR-U8-16) — 부분 실패 시 섹션 오류 마커 고지. **계정 삭제(삭제 대상 사전 고지+재확인·소프트 삭제·30일 유예** D18 BR-U8-13) — 등록 숙소·여행 기록·회고 삭제 고지, 유예 중 복구 안내(30일 내 재로그인).
- **상호작용**: `M2.updateBasicInfo` · `M1.requestDataExport` · `M1.requestAccountDeletion`.
- **data-testid**: `settings-account-basicinfo`, `settings-account-export`, `settings-account-delete`, `settings-account-deleteconfirm`, `settings-account-recovery`.
- **사용 서버 능력**: `M2.updateBasicInfo`, `M1.requestDataExport`, `M1.requestAccountDeletion`.

### 5.6 PolicyDocsScreen (고객 지원·정책 재열람)
- **props**: 없음 · **state**: 정책 문서 목록.
- **폼 검증(서버 규칙 대응)**: **이용약관·개인정보처리방침·위치기반서비스 이용약관 3종 상시 재열람**(N5)·**오픈소스 라이선스 목록**(심사 요건)·**이메일 문의 링크**(고객 지원 연락처 BR-U8-24).
- **상호작용**: 정책 문서 조회(정적/원격 config) · 이메일 문의(OS 메일 앱).
- **data-testid**: `settings-policy-terms`, `settings-policy-privacy`, `settings-policy-location`, `settings-policy-oss`, `settings-policy-support`.
- **사용 서버 능력**: 정책 문서 조회 API(원격 config).

---

## 6. 홈 카드 최종 통합 검증 (U2 HomeDashboard 슬롯 × U3~U7)

U8은 마감 유닛으로서 **U2 `HomeDashboard`(features/home 재사용)의 슬롯 스키마에 전 유닛 데이터가 전량 채워짐을 통합 검증**한다(신규 화면·신규 슬롯 없음 — U2 슬롯 스키마 재정의 금지).

| U2 슬롯(u2 §4.1) | 공급 유닛·능력 | U8 통합 검증 |
|---|---|---|
| activeTripCard | U6 `M18.getActiveHub`·`getTripProgress` | 진행 중 여행 카드·진행률(여행 중만)·execution 진입 |
| upcomingTripCard | U4 `M6.listTrips` | 예정 여행·D-day·0건 emptyState |
| trendingPlaces | U3 `M7.getTrendingPlaces` | 인기 장소·조회 실패 시 슬롯 생략(부분 응답) |
| memoryCard | U7 `M12.getVisitStats` | 추억 진입·기록 0건 미노출 |
| preferencePromptCard | U1 `M2.getPendingPreferencePrompts` | 점진 취향 카드 |
| **topBar** | U1 세션 + **U8 `M14.getInbox`** | **워드마크 + 미읽음 배지 카운트(U8 공급분)** |
| communityRecordCard | (U10 후속) | 미노출 게이트 유지(u2 INV-HD4) |

- **폼 검증(서버 규칙 대응)**: **슬롯 스키마·부분 응답·미노출 규칙은 U2 소유**(u2 §4.1 INV-HD1~5 재정의 금지 BR-U8-26). U8은 **미읽음 배지 데이터 공급 + 전 슬롯 채움 통합 검증**만. 일부 슬롯 실패 시에도 가용 카드 우선(침묵 실패 금지 u2 INV-HD1 승계).
- **data-testid**: `settings-home-integration-badge`(미읽음 배지 공급 확인) — 그 외 홈 슬롯 testid는 U2 소유.
- **사용 서버 능력**: `M14.getInbox`(미읽음 배지) + (검증 대상)U3~U7 슬롯 능력.

## 7. shared/ 재사용·U8 완성분 (요약)

| 계층 | U8 범위 | 근거 |
|---|---|---|
| features/home (U2 소유) | **재사용** — 홈 대시보드·5탭 셸. U8은 신규 셸 미생성·미읽음 배지 공급+최종 통합 검증 | u2 §4.1, FD-U8-12 |
| features/archive (U7 소유) | **재사용** — 마이페이지 여행 회고·스타일 상세 진입 | U7 archive, US-E09-07·08 |
| shared/location (U6 소유) | **재사용** — 위치 3층 상태(G182)·OS 권한 상태. U8은 설정 관리 UI·철회 영향 고지 | U6 shared/location, D34/G182 |
| features/notification | **신규** — 인앱 알림함·딥링크 라우팅·푸시 프리프롬프트 | D32, G99, G7 |
| features/settings | **신규** — 마이페이지·알림 설정·계정 관리·위치 3층·마케팅·취향·정책 재열람 | G103, N5, N8, D18 |
| shared/ui | 알림 종류 아이콘·읽음 배지·토글 로우·방해금지 창 피커·삭제 재확인 다이얼로그·정책 문서 뷰 | D32, D18, N5 |

## 8. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력 |
|---|---|---|
| NotificationInbox + DeeplinkRouter | US-E09-05, US-E2-02 | M14.getInbox · markRead |
| PushPrepromptSheet | US-E09-05 | M14.registerDeviceToken |
| MyPageHome | US-E09-06·07·08 | M4.listMyStays · M6.listTrips · M13.getStyleSummaryCard |
| StayManageScreen + RegenConfirmDialog | US-E09-06 | M4.listMyStays · linkToTrip/unlinkFromTrip · M8.regenerate · M14.cancelTripReminders |
| TripListScreen | US-E09-07 | M6.listTrips · (U7)archive |
| TravelStyleScreen | US-E09-08 | M13.getStyleSummaryCard · (U7)M13.analyzeTravelStyle |
| NotificationSettingsScreen | US-E09-02·03·05 | M14.getSettings · updateToggles · updateQuietHours |
| LocationConsentScreen | US-E09-11 | (U1)위치 동의 3층 · (U7)M12.purgeLocationData |
| PreferenceSettingsScreen | US-E09-10 | M2.getPreferences · updatePreferences |
| MarketingConsentToggle | US-E09-14 | (M1/M2)마케팅 동의 |
| AffiliateNoticeToggle | US-E09-12 | M1.updateAffiliateNoticePref |
| AccountManageScreen | US-E09-09 | M2.updateBasicInfo · M1.requestDataExport · requestAccountDeletion |
| PolicyDocsScreen | US-E09-13 | 정책 문서 조회 API |
| HomeIntegrationVerifyView | US-E2-02 | M14.getInbox + (검증)U3~U7 슬롯 능력 |
