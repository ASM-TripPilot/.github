# U8 알림·마이페이지·설정 — Functional Design 실행 계획 (Plan)

> 2026-07-04 · Construction/Functional Design · 대상 유닛: **U8 알림·마이페이지·설정**(M14 Notification + 각 모듈 설정 화면 + 홈 카드 최종 통합)
> 소유 스토리: Epic 9(US-E09-01~14) = **14 스토리**(알림 스케줄링·발송 파이프라인·인앱 알림함·마이페이지·계정 삭제 연쇄·데이터 내보내기·위치 3층·마케팅 동의·정책 재열람·홈 카드 최종 통합).
> 형식·깊이 기준: 직전 유닛 [u7-archive/functional-design](../u7-archive/functional-design/) 4종 산출물. 기술 중립(전송·저장·FCM 구현체·스케줄러 큐·remote config 수치=방해금지 기본값·중복 억제 창·90일·30일·리마인드 오프셋은 NFR/Infrastructure Design 소유).
> 정본 참조: unit-of-work.md §U8 · unit-of-work-dependency.md **CP5(U3·U5·U6·U7 → U8 소비)** · components.md §M14 Notification·§6.1 `notification`/`settings` feature · component-methods.md §M14 · services.md **S5(알림 스케줄링·발송)·S6(계정 삭제 연쇄)·S9(데이터 내보내기)** · requirements.md(D12·D18·D32·D34/N2·§5.3 N5·N8·§6.7 G144·§7.4 G97·G99·G100·G101·G103·G182·C4·C8/G95·C12/Δ10) · PRD 12(정본) · u7-archive/functional-design/domain-entities.md §12(CP5 공급 계약 ReflectionReady) · u2-appshell/functional-design/domain-entities.md §4.1(HomeDashboardModel 슬롯 스키마).
> **U8은 1차 출시 마감 유닛** — 신규 도메인 로직(M14 알림)보다 **CP5 소비·전 모듈 설정 통합·계정 삭제 연쇄 완결·홈 카드 최종 통합**이 중심이다.

---

## 0. 산출물·경로

| # | 산출물 | 경로 | 상태 |
|---|---|---|---|
| 1 | 실행 계획(본 문서) | `plans/u8-notification-functional-design-plan.md` | [x] |
| 2 | 도메인 엔티티 | `u8-notification/functional-design/domain-entities.md` | [x] |
| 3 | 비즈니스 규칙 카탈로그 | `u8-notification/functional-design/business-rules.md` | [x] |
| 4 | 비즈니스 로직 모델 + Testable Properties | `u8-notification/functional-design/business-logic-model.md` | [x] |
| 5 | 프런트엔드 컴포넌트 | `u8-notification/functional-design/frontend-components.md` | [x] |

---

## 1. 실행 체크리스트

### 단계 A — 입력 정본 흡수
- [x] U7 산출물 4종 정독(깊이·추적 ID·구조 기준 확립) — 특히 CP5 공급 계약(ReflectionReady·3계열 대조 데이터)
- [x] unit-of-work.md §U8(포함·제외·DoD·리스크) + CP5(소비 — 4계열 이벤트) 흡수
- [x] components.md §M14 Notification(상태 머신·발송 파이프라인·소유 엔티티) + §6.1 `notification`·`settings` feature 흡수
- [x] component-methods.md §M14(scheduleTripNotifications·rescheduleOnItineraryChange·cancelTripReminders·sendEvent·getSettings/updateToggles·updateQuietHours·getInbox/markRead·registerDeviceToken) 흡수
- [x] services.md S5(알림 스케줄링·발송 파이프라인)·S6(계정 삭제 연쇄·모듈별 파기 핸들러·법정 보존 분리)·S9(데이터 내보내기·읽기 전용 수집·부분 실패 마커) 흡수
- [x] requirements.md D12·D18·D32·D34/N2·N5·N8·§6.7 G144·§7.4(G97·G99·G100·G101·G103·G182·C4·C8/G95·C12/Δ10)·PRD 12 정본 흡수
- [x] u2 HomeDashboardModel §4.1 슬롯 스키마(홈 카드 최종 통합 검증 대상) 흡수

### 단계 B — 도메인 엔티티(파일 2)
- [x] 엔티티 지도(M14 소유 + CP5 소비 이벤트 + 각 모듈 설정 참조 + 홈 카드 통합)
- [x] NotificationSchedule(서버 스케줄·일정 변경 전면 재계산 D32·멱등 키·리마인드 오프셋) — M14
- [x] Notice/InboxNotice(인앱 알림함·90일 보존·읽음 G99·채널 독립 적재) — M14
- [x] NotificationToggle(종류별 푸시/인앱 독립 토글·즉시 반영 D32·NoticeType 정본 C12/Δ10) — M14
- [x] QuietHours/DndSetting(방해금지 22~08·진행 중 Plan-B 예외 G100) — M14
- [x] PushToken(FCM 단일·기기별·다기기·무효 정리 D12) — M14
- [x] MarketingConsent(N8·수집·철회·1차 미발송) — M1/M2 참조·U8 관리
- [x] DataExportRequest(G101·JSON 즉시·사진 제외·부분 실패 마커 S9) — M1 참조·U8 관리
- [x] AccountDeletionCascade(S6·30일 유예·연쇄 대상 목록·법정 보존 분리·익명화 D18) — M1 소유·U8 완결
- [x] LocationConsentState(위치 3층 D34/G182 — U1 모델 참조·U8 관리 UI)
- [x] AffiliateNoticePref(제휴 고지 다시 보기 토글 US-E09-12)
- [x] MyPageView(숙소·여행·스타일 집계 G103 — 읽기 전용 BFF)
- [x] 속성표·불변식(INV-xx)·상태 필드(schedule status·notice 만료·삭제 유예)
- [x] CP5 소비 계약(StayRegistered/LinkedToTrip·ItineraryConfirmed/Changed·TriggerFired·ReflectionReady + AccountDeletionExpired)

### 단계 C — 비즈니스 규칙(파일 3)
- [x] 규칙 색인(그룹 A~G)·규칙 ID `BR-U8-xx`(전역 유일)
- [x] A. 알림 스케줄링(서버 통일 D32·D-1/당일/N분전·일정 변경 재계산·소요시간 미표시 D25·숙소 미등록 유도 1회·거점 해제 중단 G97)
- [x] B. 발송 파이프라인(종류별 토글 → 방해금지 G100 → 중복 억제 10분 → FCM+인앱 선적재·FCM 발송 D12·시스템 알림 예외)
- [x] C. 인앱 알림함(90일 보존 G99·읽음·미읽음 배지·채널 독립 적재)·종류별 토글(즉시 반영)·알림 종류 정본(C12/Δ10)
- [x] D. 계정 삭제 연쇄(재확인·30일 유예·복구·모듈별 연쇄 목록·익명화·법정 보존 분리 D18/S6)·데이터 내보내기(JSON 즉시·사진 제외·부분 마커 G101/S9)
- [x] E. 위치 3층 설정(철회 시 위치 비의존만 D34/G182·GPS 옵트인 철회 즉시 파기)·마케팅 동의(수집·철회·1차 미발송 N8)
- [x] F. 마이페이지(숙소 통합 목록 G103·출발점 전환·G97 배지·여행 구분 조회·스타일 요약 카드)·취향 직접 설정(직접 설정 우선·예산 총액 D26)
- [x] G. 고객지원·정책 재열람(N5)·OTA 제휴 고지(다시 보기 토글)·홈 카드 최종 통합(U2 슬롯 채움 검증)
- [x] 각 규칙 조건/동작/위반 시 처리/근거

### 단계 D — 비즈니스 로직 모델 + Testable Properties(파일 4)
- [x] FLOW-1 알림 스케줄링(CP5 이벤트 구독 → 스케줄/재계산 → 발송 큐)
- [x] FLOW-2 발송 파이프라인(종류별 토글 → 방해금지 → 중복 억제 → FCM+알림함 선적재)
- [x] FLOW-3 계정 삭제 연쇄(S6·재확인·30일 유예·복구·만료 배치·모듈별 파기·법정 보존)
- [x] FLOW-4 데이터 내보내기(S9·읽기 전용 수집·부분 실패 마커·JSON 즉시)
- [x] FLOW-5 설정 변경 반영(토글·방해금지·위치 3층·마케팅·취향 — 즉시 저장) + 홈 카드 최종 통합(U2 슬롯 채움 검증)
- [x] Testable Properties(PBT-01): 알림 스케줄 재계산 멱등성·방해금지 창 판정 결정성/경계·중복 억제 창 불변식·토글 즉시 반영·삭제 연쇄 완전성(전 소유 데이터 커버·법정 보존 분리 불변식)·데이터 내보내기 완전성·위치 3층 조합 동작 매트릭스
- [x] No-PBT 컴포넌트·커버리지 대조·CP5 통합 검증 시나리오

### 단계 E — 프런트엔드 컴포넌트(파일 5)
- [x] 화면 플로우(정본) + 컴포넌트 계층(`features/notification`·`features/settings`)
- [x] 알림함·알림 설정(토글·방해금지)·마이페이지(숙소·여행·스타일·예약 기록)·계정 관리(삭제·내보내기)·위치 동의 3층 설정·마케팅 토글·고객지원/정책
- [x] props/state·data-testid(`settings-{screen}-{role}`)·백엔드 능력(M14 + 각 모듈 설정 API U1~U7)
- [x] 홈 카드 최종 통합(U2 HomeDashboard 슬롯 × U3~U7 데이터 전체) 검증 뷰
- [x] shared/ 재사용·화면-스토리-능력 추적 매트릭스

---

## 2. 설계 결정 (FD-U8-xx) — 질문 대체

> 본 유닛은 정본 문서(components.md M14·component-methods.md M14·services.md S5·S6·S9·PRD 12·requirements.md)가 결정을 이미 확정한 상태다. 아래는 Functional Design 단계에서 **정본을 인터페이스 계약·불변식 수준으로 상세화하며 내린 판단**을 명시(질문 없이 결정)한다.

| ID | 결정 | 근거·정본 |
|---|---|---|
| FD-U8-01 | **알림 스케줄링은 서버 통일(D32)** — 리마인드(D-1·당일 요약·개별 일정 N분 전)는 M14가 `ItineraryConfirmed` 수신 시 일괄 스케줄, `ItineraryChanged` 수신 시 **해당 여행 리마인드 전면 무효화 후 재계산**(멱등 키 tripId+slotId+type). 클라이언트 로컬 스케줄링 없음 | D32, S5, component-methods M14 |
| FD-U8-02 | **발송 파이프라인은 단일 게이트 순서 고정** — ①종류별 토글 → ②방해금지 창(22~08·진행 중 Plan-B severe만 예외 G100) → ③중복 억제(동일 dedupeKey 10분 창) → ④인앱 알림함 **선(先) 적재**(TX) → ⑤FCM 발송(TX 밖·재시도 3회). 인앱 적재는 채널 토글에 독립, 억제분도 적재(푸시만 보류) | S5, G100, D12, component-methods M14 |
| FD-U8-03 | **인앱 알림함(Notice)은 90일 보존·읽음 관리(G99/D32)** — 푸시 OFF여도 인앱 적재 지속(역도 동일·채널 독립), 종류별 토글이 완전 off(종류 자체 off)면 적재도 없음(사용자 의사). 단 **보안·계정 시스템 알림은 전 채널 OFF에도 인앱 표시**(예외) | D32, G99, PRD 12-5 |
| FD-U8-04 | **알림 종류 정본은 PRD 12 목록(Δ10/C12)** — 숙소 등록/저장 완료·여행 시작 D-1·당일 일정 요약·개별 일정 시작 전·Plan-B 재계획·회고 완료. **'체크인 임박'·'예약 링크 리마인드'는 제외**. (후속) 커뮤니티 좋아요·댓글·공동편집 알림 종류는 예약(미노출) | Δ10, C12, PRD 12-5 |
| FD-U8-05 | **계정 삭제 연쇄는 소프트 삭제 + 30일 유예(D18/S6)** — 요청 즉시 전 표면 비노출·세션 무효화·재확인 필수, **위치 데이터는 즉시 파기**(유예 없음 D34), 30일 내 재로그인 복구 가능, 만료 배치가 `AccountDeletionExpired` 발행 → **각 모듈이 자기 파기 핸들러 소유**(M6·M8·M12·M13·M14·M2·M4). **법정 보존 데이터(동의 증적·위치 로그·감사 로그)만 분리 보관**, (후속) 커뮤니티 콘텐츠는 '삭제된 사용자' 익명화 | D18, S6, C4, D34/N2 |
| FD-U8-06 | **모듈별 파기 핸들러는 멱등**(이미 삭제된 리소스 skip) — 배치 전체 원자성 없음(각자 독립 TX·부분 진행 허용, `PURGING` 상태가 진행 정본), at-least-once 재전달로 재시도, 전 핸들러 완료 확인 후 계정 레코드 최종 삭제. **"잔존 데이터 = 법정 보존 집합"은 릴리스 게이트 PBT** | S6, D18, unit-of-work §U8 DoD |
| FD-U8-07 | **데이터 내보내기는 텍스트 JSON 즉시 다운로드(G101/S9)** — 읽기 전용 수집(M2·M6·M8·M12 메타·M13·M14 설정), **사진 파일 제외(메타만)**, 동기 스트리밍(1차 규모), rate-limit(계정당 시간당 1회)+본인 재인증. **특정 모듈 수집 실패 시 해당 섹션 오류 마커 포함**(침묵 누락 금지 — 부분 성공을 완전본으로 위장하지 않음). 사진 포함 전체 아카이브는 후속(G186) | G101, S9, G186 |
| FD-U8-08 | **위치 동의는 3층 관리 UI(D34/G182)** — OS 권한 × 앱 내 위치기반서비스 법정 동의 × GPS 여행 기록 옵트인. **U1이 데이터 모델 소유**, U8은 각 층 구분 표시·제어·철회 영향 고지·재확인 UI. **철회 후 위치 비의존 기능(예정 일정 알림·날씨·휴무 트리거) 유지**, GPS 옵트인 철회·탈퇴 시 즉시 파기(U7 `purgeLocationData` 연동). OS 권한 거부 시 토글 비활성+설정 이동 안내 | D34/N2, G182, U7 INV-GPS2 |
| FD-U8-09 | **마이페이지는 읽기 전용 집계(MyPageView, G103)** — 숙소 저장/등록 통합 목록+출처 라벨(M4)·여행 예정/진행/종료 구분(M6·D19·D21)·스타일 요약 카드(M13). 출발점 전환은 M4 퍼사드 위임(전환 시 재생성 확인, 거부 시 리마인드 중단+재생성 배지 G97). U8은 조립·표시만, 소유는 각 모듈 | G103, G97, D15, D19, D21 |
| FD-U8-10 | **마케팅 수신 동의는 수집·관리만(N8)** — 온보딩 수집분(U1)의 현재 상태 표시·토글 부여/철회·즉시 저장. **1차 출시에서 마케팅 알림 발송 없음**(발송은 후속). 위치 3층·마케팅·취향은 M1/M2 소유 데이터, U8은 설정 UI·즉시 반영 | N8, US-E09-14 |
| FD-U8-11 | **취향 직접 설정은 온보딩 동일 선택지(US-E09-10)** — 수정 값이 다음 AI 생성·재계획 입력에 반영, **직접 설정 값 > 자동 스타일 분석**(충돌 시 우선), 예산은 여행 전체 총액(항공 제외) 기준·1인/1일은 파생 표기(D26). M2 소유, U8은 설정 진입·즉시 반영 | US-E09-10, D26, U7 BR-U7-24 |
| FD-U8-12 | **홈 카드 최종 통합 검증은 U2 HomeDashboardModel 슬롯 × U3~U7 데이터 전체(FD-U8-신규 책임)** — U2가 고정한 슬롯 스키마(activeTripCard·upcomingTripCard·trendingPlaces·memoryCard·preferencePromptCard·topBar)에 U6·U4·U3·U7·U1·U8(getInbox 미읽음 배지) 데이터가 부분 응답·침묵 실패 금지 원칙대로 전량 채워짐을 검증. **슬롯 스키마는 U2 소유(재정의 금지)**, U8은 통합 검증만 | u2 §4.1, US-E2-02, CP5 |
| FD-U8-13 | **CP5는 미지 유형을 무시가 아닌 계측 대상으로 처리(침묵 실패 금지)** — envelope는 `common/core`(U1) 소유, 유형 추가(U10·U11 알림)는 하위 호환 확장. G144 관측성(외부 API·FCM 실패율·발송 계측) 준수 | CP5, G144, unit-of-work-dependency §CP5 |
| FD-U8-14 | **추적 ID 체계**: 엔티티 불변식 `INV-{도메인}`(SCHED/NOTICE/TOGGLE/DND/TOKEN/MKT/EXPORT/DEL/LOC/AFF/MYPAGE), 규칙 `BR-U8-xx`, 속성 `U8-Pxx`, 설계 결정 `FD-U8-xx`, data-testid `settings-{screen}-{role}` | U7 관례 승계 |

---

## 3. 위험·주의 (Functional Design 한정)

- **계정 삭제 연쇄 누락(모듈 추가 시마다 대상 증가)** — 법적 리스크. 삭제 연쇄를 이벤트 구독 계약(`AccountDeletionExpired`)으로 강제, 각 모듈이 자기 파기 핸들러 소유. **"잔존 데이터 = 법정 보존 집합" PBT(U8-P5)를 릴리스 게이트**로 고정. 법정 보존(동의 증적·위치 로그·감사 로그)과 파기 대상의 경계를 불변식으로 명문화.
- **발송 파이프라인 게이트 순서·조합 폭발** — 토글×방해금지×중복 억제×채널 독립 적재의 조합. 발송 판정을 **순수 결정 함수로 분리**해 PBT(U8-P2·P3)로 조합 전수(꺼진 종류 발송 0·창 내 비예외 발송 0·10분 창 중복 0·억제분 알림함 적재).
- **위치 3층 조합 동작 매트릭스(G182)** — OS 권한 × 법정 동의 × GPS 옵트인 8조합 × 철회 영향. 조합별 동작 매트릭스를 명문화(테이블 주도 테스트 U8-P7). 전 조합 폴백 존재(철회 후 위치 비의존 기능 유지).
- **CP5 소비 스키마 정밀도** — 4계열 이벤트(U3·U5·U6·U7)의 발행 스키마 확정이 U8 착수 게이트. envelope 변경은 발행 4유닛 회귀 유발 — 하위 호환 확장만.
- **스토어 심사 반려(권한 고지·지원 연락처·정책 문서)** — N5 요건(정책 재열람·이메일 문의·오픈소스 라이선스)을 수용 기준으로 완결. P9 심사 요건 체크리스트를 Build and Test 입력으로.

---

## 4. 완료 판정

- [x] 5개 산출물 생성 완료, 추적 ID(INV·BR-U8·U8-P·FD-U8·data-testid) 일관.
- [x] 14 스토리(E9) 전 커버 — 엔티티·규칙·플로우·화면 추적 매트릭스에 매핑.
- [x] CP5 소비(StayRegistered/LinkedToTrip·ItineraryConfirmed/Changed·TriggerFired·ReflectionReady + AccountDeletionExpired) 계약 명문화.
- [x] Testable Properties: 프롬프트 필수 7속성 전부 포함(스케줄 재계산 멱등·방해금지 결정성/경계·중복 억제 창·토글 즉시 반영·삭제 연쇄 완전성/법정 분리·내보내기 완전성·위치 3층 매트릭스).
