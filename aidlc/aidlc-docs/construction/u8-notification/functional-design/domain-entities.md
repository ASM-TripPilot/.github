# U8 알림·마이페이지·설정 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: **M14 Notification**(신규 도메인) + 각 모듈 설정·삭제 연쇄 보완(M1·M2·M4·M6·M12·M13 퍼사드 소비 — 참조·통합 수준)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §M14 + [component-methods.md](../../../inception/application-design/component-methods.md) §M14 메서드를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U8 엔티티·불변식·상태 필드의 정본이다. 기술 중립 — FCM 구현체·스케줄러 큐·저장 기술·remote config 수치(방해금지 기본값·중복 억제 창·90일·30일·리마인드 오프셋)는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL 의미 명기), `유니크`=제약 범위 내 유일, `불변`=생성 후 변경 불가, `추가전용`=append-only, `파생`=저장 안 함(조회 시점 계산). 근거 ID는 requirements.md(D·N·G·C·Δ·ADR)·stories.md(US-E09)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U8-xx)을 참조한다.
> **U8은 CP5(U3·U5·U6·U7 → U8)의 소비자다** — StayRegistered/LinkedToTrip(U3/U4)·ItineraryConfirmed/Changed(U5)·TriggerFired(U6)·ReflectionReady(U7)를 단일 구독·스케줄링(D32)하며, `AccountDeletionExpired`(U1)를 구독해 알림 데이터를 연쇄 파기한다(§11 소비 계약).
> **소유 경계**: M14(알림 스케줄·알림함·토글·방해금지·토큰)는 **U8 소유 정본**. 계정 삭제 연쇄(S6)·데이터 내보내기(S9)는 **M1.application 소유**이며 U8이 완결·UI를 제공. 마케팅 동의·위치 3층·취향은 **M1/M2 소유 데이터**를 U8이 설정 UI로 관리(참조). 마이페이지 집계는 **읽기 전용 BFF**(각 모듈 퍼사드 소비).

## 0. 엔티티 지도

```text
[M14 Notification — U8 소유 정본]
Account 1 ──── * NotificationSchedule   (서버 스케줄 리마인드 — 일정 변경 전면 재계산 D32)
Account 1 ──── * Notice                  (인앱 알림함 — 90일 보존·읽음 G99·채널 독립 적재)
Account 1 ──── 1 NotificationToggle      (종류×채널 독립 토글 — 즉시 반영 D32·NoticeType 정본 C12/Δ10)
Account 1 ──── 1 QuietHours              (방해금지 창 — 22~08·진행 중 Plan-B 예외 G100)
Account 1 ──── * PushToken               (기기별 FCM 토큰 — 다기기·무효 정리 D12)

[설정 데이터 — M1/M2 소유·U8 관리 UI(참조)]
Account 1 ──── 1 MarketingConsent        (마케팅 수신 동의 — 수집 U1·철회 U8·1차 미발송 N8)
Account 1 ──── 1 LocationConsentState     (위치 3층 — OS×법정×GPS 옵트인·모델 U1·관리 U8 D34/G182)
Account 1 ──── 1 AffiliateNoticePref      (OTA 제휴 고지 다시 보기 토글 US-E09-12)

[계정 라이프사이클 — M1.application 소유·U8 완결]
Account 1 ──── 0..1 AccountDeletionCascade (소프트 삭제·30일 유예·연쇄 목록·법정 보존 분리 D18/S6)
Account 1 ──── * DataExportRequest         (데이터 내보내기 — JSON 즉시·사진 제외·부분 마커 G101/S9)

[읽기 전용 집계 — 각 모듈 퍼사드 소비(BFF)]
MyPageView                                (숙소 통합 목록 G103·여행 구분 조회·스타일 요약 카드 — 저장 안 함)

[CP5 입력 — U3·U5·U6·U7 소유(본 유닛은 구독·소비, §11)]
StayRegistered/StayLinkedToTrip · ItineraryConfirmed/ItineraryChanged · TriggerFired · ReflectionReady
[U1 입력] AccountDeletionExpired(연쇄 파기 트리거)

[홈 카드 최종 통합 — U2 슬롯 스키마(재정의 금지·§10 검증)]
HomeDashboardModel(U2 소유) × {U3 인기 장소·U4 예정 여행·U6 활성 카드·U7 추억·U8 미읽음 배지}
```

**알림 종류 정본(NoticeType — C12/Δ10, PRD 12-5)**

| NoticeType | 채널 기본 | 발행 원천(CP5) | 방해금지 예외 |
|---|---|---|---|
| STAY_REGISTERED | 푸시+인앱 | U3 `StayRegistered`/U4 `StayLinkedToTrip` | 아니오 |
| TRIP_REMINDER_D1 | 푸시+인앱 | U5 `ItineraryConfirmed`(D-1 스케줄) | 아니오 |
| TRIP_REMINDER_DAY | 푸시+인앱 | 당일 요약(기본 08:00·설정 가능) | 아니오 |
| TRIP_REMINDER_SLOT | 푸시+인앱 | 개별 일정 N분 전(0/15/30/60·기본 30) | 아니오 |
| PLAN_B | 푸시+인앱 | U6 `TriggerFired`(severe만 푸시) | **예(진행 중 여행·severe만)** |
| REFLECTION_READY | 푸시+인앱 | U7 `ReflectionReady` | 아니오 |
| SYSTEM_SECURITY | 인앱 강제 | M1 보안·계정(시스템) | — (전 채널 OFF에도 인앱 표시) |
| (예약) COMMUNITY_SOCIAL | — | (후속) U10 좋아요·댓글 | 미노출 |
| (예약) COEDIT_INVITE | — | (후속) U11 초대·권한·종료 G95/C8 | 미노출 |

> **제외(Δ10/C12)**: '체크인 임박'·'예약 링크 리마인드'는 종류 목록에서 제외한다(후속 검토).

---

## 1. NotificationSchedule — 서버 스케줄 리마인드 (M14, D32 정본)

여행 확정 시 일괄 생성되는 **서버 스케줄링 리마인드**(D-1·당일 요약·개별 일정 N분 전). 모든 리마인드는 서버 스케줄링으로 발송하며(D32 — 클라이언트 로컬 스케줄 없음), **일정 변경 시 발송 시각을 전면 재계산**한다. 발송 시점(fireAt)에 J4 배치가 발송 파이프라인(§2 발송)을 태운다.

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| scheduleId | 식별자 | 필수 · 유니크 | — |
| targetAccountId | 식별자(Account 참조) | 필수 | 발송 대상 계정 |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행(리마인드 스케줄 단위) |
| slotRef | 식별자(Slot 참조) | 선택 | 개별 일정 리마인드의 대상 슬롯. NULL=여행 단위(D-1·당일 요약) [D32] |
| type | 열거<NoticeType 리마인드계열> | 필수 | TRIP_REMINDER_D1 / _DAY / _SLOT [Δ10] |
| fireAt | 시각 | 필수 | 발송 예정 시각(서버 산출·재계산 대상) [D32] |
| dedupeKey | 문자열 | 필수 | 멱등 키 `tripId+slotRef+type`(중복 스케줄·중복 발송 방지) [S5, INV-SCHED2] |
| payload | 구조체 {title, body, deeplink, distanceText?} | 필수 | 알림 본문(장소명·시작 시각·**이동 거리만**·소요시간 없음 D25)+딥링크(탭 스택 푸시 G7) [D25, G7] |
| status | 열거 {SCHEDULED, FIRED, SUPPRESSED, CANCELLED} | 필수 · 기본 `SCHEDULED` | 수명(§1.3) [components M14] |
| suppressReason | 열거 {TOGGLE_OFF, DND, DEDUPE} | 선택 | SUPPRESSED 사유. NULL=억제 아님 [G100] |

### 1.2 파생값

| 파생값 | 계산 | 근거 |
|---|---|---|
| distanceText | 직전 지점(등록 숙소 또는 직전 일정)→대상 장소 이동 거리(추정 표기·소요시간 없음) | 리마인드 본문 규칙 [D25, US-E09-02] |
| isDue | fireAt ≤ now ∧ status=SCHEDULED | J4 발송 큐 진입 판정 [S5] |

### 1.3 스케줄 상태 전이 (status)

```text
   [ItineraryConfirmed → scheduleTripNotifications] → SCHEDULED(fireAt·dedupeKey)
     │
     ├─ [ItineraryChanged → rescheduleOnItineraryChange] → 기존 SCHEDULED 전량 CANCELLED → 새 SCHEDULED(재계산·멱등 키 유지)
     ├─ [거점 해제·재생성 거부 → cancelTripReminders] → CANCELLED(리마인드 중단·재생성 배지 G97)
     ├─ [TripEnded] → 잔여 SCHEDULED CANCELLED
     │
     fireAt 도달(J4) → 발송 파이프라인(§2 발송):
       ├─ 종류 토글 off ──▶ SUPPRESSED(TOGGLE_OFF·적재도 없음)
       ├─ 방해금지 창(비예외) ──▶ SUPPRESSED(DND·인앱 적재는 유지)
       ├─ 10분 창 중복 ──▶ SUPPRESSED(DEDUPE)
       └─ 통과 ──▶ FIRED(인앱 선적재 + FCM 발송)
```

### 1.4 불변식 (Invariants)

- **INV-SCHED1 (서버 스케줄 통일 — D32)**: 모든 리마인드는 서버 NotificationSchedule로만 존재하며 클라이언트 로컬 스케줄은 없다 — 발송 시각 산출·재계산은 서버 소유(단일 정본) [D32, FD-U8-01].
- **INV-SCHED2 (변경 시 전면 재계산·멱등)**: `ItineraryChanged` 수신 시 해당 tripId의 SCHEDULED 리마인드를 전량 무효화(CANCELLED)하고 재계산한다 — 동일 (tripId, 일정 상태)에 대해 재계산 결과는 결정적이고, 동일 이벤트 중복 수신에도 dedupeKey(`tripId+slotRef+type`)로 스케줄이 중복 생성되지 않는다 [D32, S5, PBT U8-P1].
- **INV-SCHED3 (거점 해제·재생성 거부 시 중단 — G97)**: 출발점 숙소 해제 후 사용자가 일정 재생성을 거부하면 기존 일정은 유지하되 리마인드 발송을 중단(CANCELLED)하고 재생성 유도 배지 신호를 낸다 — 리마인드 없는 유효 일정 상태가 침묵되지 않는다 [G97, US-E09-02·06].
- **INV-SCHED4 (본문 소요시간 부재 — D25)**: payload 본문은 장소명·시작 시각·이동 거리만 담고 소요시간 필드가 없다 — 어떤 리마인드에도 소요시간 표기 없음 [D25, US-E09-02].
- **INV-SCHED5 (숙소 미등록 유도 1회)**: 등록 숙소가 없어 일정이 없으면 개별 일정 리마인드 대신 "숙소를 등록하면 일정 알림을 받을 수 있습니다" 유도 안내를 1회만 스케줄한다(반복 없음) [US-E09-02 예외, PRD 12-2 예외].

---

## 2. Notice — 인앱 알림함 (M14, 90일 보존 G99)

발송 파이프라인을 통과·억제한 알림의 **인앱 알림함 레코드**. 90일 보존·읽음 관리하며, **채널 토글에 독립적으로 적재**(푸시 OFF여도 적재 지속, 역도 동일)한다. 방해금지·중복으로 푸시가 억제된 알림도 인앱 알림함에는 적재된다(침묵 실패 금지).

### 2.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| noticeId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account 참조) | 필수 | 소유 계정(소유권 격리 SECURITY-08) |
| type | 열거<NoticeType> | 필수 | 알림 종류(정본 C12/Δ10) [Δ10] |
| body | 구조체 {title, text, distanceText?} | 필수 | 알림 본문(소요시간 없음 D25) [D25] |
| deeplink | 구조체 {targetTab, stackPush} | 필수 | 딥링크 대상(탭 활성화+스택 푸시 G7) [G7] |
| createdAt | 시각 | 필수 · 불변 | 적재 시각(정렬·만료 기준) |
| readAt | 시각 | 선택 | 읽음 시각. NULL=미읽음(미읽음 배지 카운트 대상) [G99] |
| expiresAt | 시각 | 필수 · 파생 | createdAt + 90일(J5 정리 배치 대상) [G99, D32] |
| deliverySource | 열거 {FIRED, SUPPRESSED_DND, SUPPRESSED_DEDUPE} | 필수 | 적재 경로(억제분 구분·배지 처리) [S5] |

### 2.2 파생값

| 파생값 | 계산 | 근거 |
|---|---|---|
| unreadBadgeCount | readAt=NULL ∧ expiresAt>now 인 Notice 수 | 홈 상단 미읽음 배지(U2 topBar 슬롯 §10) [US-E2-02] |
| isExpired | expiresAt ≤ now | 90일 만료 정리 판정(J5) [G99] |

### 2.3 불변식

- **INV-NOTICE1 (채널 독립 적재)**: 인앱 알림함 적재는 푸시 채널 토글·방해금지·중복 억제와 독립이다 — 푸시 OFF·방해금지 억제·중복 억제 상황에도 종류 토글이 완전 off가 아니면 인앱 적재는 유지된다(억제=푸시 보류이지 적재 보류 아님) [D32, G100, PRD 12-5, PBT U8-P2].
- **INV-NOTICE2 (종류 off 시 미적재)**: 종류 자체가 off(사용자 의사)면 인앱 적재도 없다 — 이는 채널 토글(푸시/인앱)과 구분되는 종류 수준 억제(발송 파이프라인 1단계에서 종결) [S5, PRD 12-5].
- **INV-NOTICE3 (90일 보존 경계 — G99)**: Notice는 createdAt+90일에 만료되며 J5 정리 배치가 만료분을 제거한다 — 보존 경계는 정확히 90일(remote config)·조기 삭제 없음 [G99, D32, PBT U8-P6].
- **INV-NOTICE4 (시스템 알림 강제 표시)**: SYSTEM_SECURITY 종류(보안·계정)는 전 채널 OFF에도 인앱 Notice로 적재·표시된다 — 사용자 토글이 시스템 알림을 억제하지 못한다 [PRD 12-5 예외].
- **INV-NOTICE5 (소유권 격리)**: Notice는 accountId 소유자만 조회·읽음 처리 가능하다 — 타 계정 알림 미수신·미조회(토큰·대상 격리) [SECURITY-08, unit-of-work §U8 DoD].

---

## 3. NotificationToggle — 종류×채널 토글 (M14, 즉시 반영 D32)

계정별 **알림 종류×채널(푸시/인앱) 독립 토글** 설정. 별도 저장 버튼 없이 즉시 저장되어 **다음 알림부터 반영**된다. 리마인드 오프셋(개별 일정 N분 전·당일 요약 시각)도 함께 보관한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 소유 계정 |
| perType | 맵<NoticeType, {push: bool, inApp: bool}> | 필수 | 종류별 푸시·인앱 독립 토글(기본 전 종류 on) [PRD 12-5] |
| reminderSlotOffsetMin | 열거 {0, 15, 30, 60} | 필수 · 기본 30 | 개별 일정 리마인드 오프셋 [US-E09-02] |
| dayReminderTime | 시각(로컬) | 필수 · 기본 08:00 | 당일 일정 요약 발송 시각(설정 가능) [US-E09-02] |
| pbSensitivity | 열거 {LESS, NORMAL, MORE} | 필수 · 기본 NORMAL | Plan-B 자동 트리거 민감도(M9 위임 — U8은 설정 저장) [US-E09-03] |
| updatedAt | 시각 | 필수 | 최종 변경 시각(즉시 반영 확인) |

**불변식**
- **INV-TOGGLE1 (즉시 반영)**: perType·오프셋·방해금지 변경은 별도 저장 없이 즉시 저장되고 다음 알림 발송부터 반영된다 — 변경 시점 이전에 스케줄된 발송에는 발송 시점의 최신 토글이 적용된다(발송 시점 평가) [PRD 12-5, PBT U8-P4].
- **INV-TOGGLE2 (종류×채널 독립)**: 종류의 푸시 off가 인앱 off를 강제하지 않고(역도 동일), 각 종류의 두 채널은 독립 토글이다 — 한 종류 푸시 off + 인앱 on 조합이 유효 [PRD 12-5, US-E09-05].
- **INV-TOGGLE3 (종류 목록 정본 — C12/Δ10)**: perType 키는 PRD 12 종류 정본만 포함하고 '체크인 임박'·'예약 링크 리마인드'는 없다. (후속) 커뮤니티·공동편집 종류는 U10·U11 출시 시 하위 호환 추가 [Δ10, C12, PRD 12-5].
- **INV-TOGGLE4 (OS 권한 종속 표시)**: OS 푸시 권한 거부 상태면 푸시 토글은 비활성 표시(설정 이동 안내)되나 인앱 토글은 독립 동작한다 — OS 권한은 발송 게이트가 아니라 표시 상태 [PRD 12-5, US-E09-05].

---

## 4. QuietHours (DndSetting) — 방해금지 창 (M14, G100)

계정별 **방해금지 시간 창**. 기본 22~08시이며 이 구간의 푸시를 억제한다. **예외: 여행 '진행 중'의 Plan-B(severe) 알림만 방해금지 창을 통과**한다. 억제된 알림은 인앱 알림함에 적재된다(§2 INV-NOTICE1).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 소유 계정 |
| window | 구조체 {startLocal, endLocal} | 필수 · 기본 {22:00, 08:00} | 방해금지 창(로컬 시각·자정 넘김 허용) [G100] |
| enabled | 불리언 | 필수 · 기본 true | 방해금지 사용 여부 |

**파생값**: isWithinWindow(at) — 발송 판정 시각이 창 내인지(자정 넘김 정확 판정), isDndException(noticeType, tripState, severity) — PLAN_B ∧ 진행 중 여행 ∧ severe면 true(창 내여도 통과).

**불변식**
- **INV-DND1 (창 판정 결정성·경계 — G100)**: isWithinWindow(at)은 동일 (window, at)에 결정적이고 자정 넘김(22~08) 경계를 정확히 판정한다 — 창 시작·종료 경계값·자정 경계에서 오분류 없음 [G100, PBT U8-P3].
- **INV-DND2 (진행 중 Plan-B severe 예외만)**: 방해금지 창을 통과하는 예외는 **여행 '진행 중' ∧ PLAN_B ∧ severe** 단 하나다 — 경미(minor) Plan-B·비진행 여행·기타 종류는 창 내에서 전부 억제된다 [G100, US-E09-03, PBT U8-P3].
- **INV-DND3 (억제 ≠ 소실)**: 방해금지로 억제된 알림은 인앱 알림함에 적재되어 사용자 관측 가능하다(푸시만 보류) — 침묵 실패 금지 [G100, ADR-0011, INV-NOTICE1].

---

## 5. PushToken — FCM 디바이스 토큰 (M14, D12)

계정별·기기별 **FCM 디바이스 토큰**. FCM 단일 채널(iOS 포함, D12)의 발송 대상이며 다기기를 허용하고 무효 토큰은 정리한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tokenId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account 참조) | 필수 | 소유 계정(대상 격리) [SECURITY-08] |
| deviceId | 문자열 | 필수 · 유니크(계정 내) | 기기 식별(다기기 허용·기기별 1토큰) [D36] |
| fcmToken | 문자열 | 필수 | FCM 토큰(갱신 대상) [D12] |
| state | 열거 {ACTIVE, STALE, REVOKED} | 필수 · 기본 `ACTIVE` | 토큰 생애(무효 응답 시 STALE·로그아웃 시 REVOKED) [D12] |
| updatedAt | 시각 | 필수 | 최종 갱신 시각 |

**불변식**
- **INV-TOKEN1 (대상 격리)**: FCM 발송은 대상 계정의 ACTIVE 토큰에만 향한다 — 타 계정 토큰으로 발송 0(대상 격리) [SECURITY-08, unit-of-work §U8 DoD].
- **INV-TOKEN2 (무효 토큰 정리)**: FCM 무효 응답 시 해당 토큰을 STALE로 표시·정리하고 재발송하지 않는다 — 로그아웃 시 REVOKED [D12, S5].

---

## 6. MarketingConsent — 마케팅 수신 동의 (M1/M2 소유·U8 관리, N8)

**마케팅 수신 동의** 상태. 온보딩에서 수집(선택 항목·U1)하고 설정에서 부여·철회·즉시 저장한다. **1차 출시에서 마케팅 알림 발송은 없다**(발송 기능 후속) — 본 엔티티는 동의 상태의 수집·관리만 다룬다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 소유 계정 |
| granted | 불리언 | 필수 · 기본 false(온보딩 선택) | 마케팅 수신 동의 여부 [N8] |
| consentEvidenceRef | 식별자(동의 증적 참조) | 필수 | 동의·철회 증적(버전·시각 — U1 동의 증적 테이블) [N2 정합] |
| updatedAt | 시각 | 필수 | 최종 변경 시각(즉시 저장) |

**불변식**
- **INV-MKT1 (수집·관리만·1차 미발송)**: MarketingConsent는 동의 상태 수집·표시·철회만 지원하고 1차 출시에서 이 동의로 마케팅 알림을 발송하지 않는다 — 발송 파이프라인(§2)은 마케팅 종류를 미포함 [N8, US-E09-14].
- **INV-MKT2 (철회 즉시 반영·증적)**: 철회는 즉시 저장되고 동의 증적에 버전·시각을 남긴다 — 철회 이력이 침묵되지 않는다 [N8, N2].

---

## 7. LocationConsentState — 위치 3층 동의 (U1 모델·U8 관리, D34/G182)

**위치 동의 3층 상태**의 U8 관리 뷰. 세 층(OS 권한 × 앱 내 위치기반서비스 법정 동의 × GPS 여행 기록 옵트인)을 설정에서 구분 표시·제어하고, 철회 직전 영향 기능을 고지한다. **데이터 모델은 U1 소유**(법정 로그 append-only 포함), U8은 관리 UI·철회 영향 고지·재확인·파기 연동을 소유한다.

| 속성(관리 뷰) | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 소유 계정 |
| osPermission | 열거 {GRANTED, DENIED, UNDETERMINED} | 필수 | OS 위치 권한(1층 — 앱 토글의 상위 게이트) [G182] |
| legalConsent | 불리언 | 필수 | 앱 내 위치기반서비스 법정 동의(2층·별도 필수 동의 N2) [D34/N2] |
| gpsRecordOptIn | 불리언 | 필수 · 기본 false | GPS 여행 기록 옵트인(3층·별도 옵트인 N2) [D34/N2] |
| consentLogRef | 식별자(위치 법정 로그 참조) | 필수 | 위치정보 수집·이용 법정 로그(append-only·앱은 삭제 권한 없음 N2) [N2, SECURITY-14] |

### 7.1 위치 3층 조합 동작 매트릭스 (G182 명문화)

| OS 권한 | 법정 동의 | GPS 옵트인 | 이동 지연 트리거 | 실시간 Plan-B(위치) | 현위치 추천 | 예정 일정 알림·날씨·휴무 | GPS 발자취 기록 |
|---|---|---|---|---|---|---|---|
| GRANTED | true | true | ✅ | ✅ | ✅ | ✅(위치 비의존) | ✅ |
| GRANTED | true | false | ✅ | ✅ | ✅ | ✅ | ❌(옵트인 없음) |
| GRANTED | false | (n/a) | ❌ | ❌ | ❌ | ✅ | ❌ |
| DENIED | (any) | (any) | ❌ | ❌ | ❌ | ✅ | ❌ |
| UNDETERMINED | (any) | (any) | ❌(프리프롬프트) | ❌ | ❌ | ✅ | ❌ |

> 판독: **위치 의존 기능**(이동 지연·실시간 Plan-B·현위치 추천·GPS 발자취)은 세 층 충족 시에만 동작하고, **위치 비의존 기능**(예정 일정 알림·날씨·휴무 트리거)은 철회·거부 어느 조합에서도 계속 동작한다(전 조합 폴백 존재).

**불변식**
- **INV-LOC1 (3층 게이트·위치 비의존 유지 — D34)**: 위치 의존 기능은 세 층(OS×법정×옵트인) 전부 충족 시에만 활성되고, 어느 한 층 철회·거부 시에도 위치 비의존 기능(예정 일정 알림·날씨·휴무 트리거)은 계속 동작한다 — 철회가 앱 전체를 무력화하지 않는다 [D34, G182, PBT U8-P7].
- **INV-LOC2 (철회 영향 고지·재확인)**: GPS 옵트인·법정 동의 철회 직전에 영향 기능(이동 지연 알림 중단·위치 기반 재계획 제한·현위치 추천 비활성)을 구체 고지하고 재확인을 받는다 — 침묵 철회 없음 [D34, US-E09-11].
- **INV-LOC3 (GPS 철회·탈퇴 즉시 파기)**: GPS 옵트인 철회·계정 탈퇴 시 위치 데이터를 즉시 파기(U7 `purgeLocationData` 연동·30일 유예 없음)하되 위치 법정 로그(append-only)는 분리 보관 유지한다 [D34/N2, U7 INV-GPS2, S6.2].
- **INV-LOC4 (OS 권한 상위 게이트)**: OS 권한 거부면 앱 내 위치 토글은 비활성 표시(설정 이동 안내)되고 하위 층 동작 불가 — OS 권한이 2·3층의 상위 게이트 [G182, US-E09-11 예외].

---

## 8. AffiliateNoticePref — OTA 제휴 고지 설정 (U8, US-E09-12)

외부 OTA 링크 이동 전 **제휴(수익) 링크 고지**의 표시 여부 설정. 한 번 확인하면 다시 보지 않도록 설정할 수 있고, 설정에서 언제든 다시 켤 수 있다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 소유 계정 |
| showAffiliateNotice | 불리언 | 필수 · 기본 true | 외부 링크 이동 전 제휴 고지 표시 여부 [US-E09-12] |
| updatedAt | 시각 | 필수 | 최종 변경 시각 |

**불변식**
- **INV-AFF1 (고지 기본 표시·재활성 가능)**: 제휴 고지는 기본 표시이며 '다시 보지 않기' 확인 후 숨김 가능하되 설정에서 언제든 재활성된다 — 고지 정본 문안(외부 이동·제휴 수수료·추가 비용 없음)은 표시 조건과 무관하게 고정 [US-E09-12].

---

## 9. AccountDeletionCascade · DataExportRequest — 계정 라이프사이클 (M1.application 소유·U8 완결)

### 9.1 AccountDeletionCascade — 계정 삭제 연쇄 (S6, D18)

**소프트 삭제 + 30일 유예** 계정 삭제의 연쇄 상태. 요청 즉시 전 표면 비노출·세션 무효화, 위치 데이터 즉시 파기, 30일 후 만료 배치가 `AccountDeletionExpired`를 발행해 **각 모듈이 자기 데이터를 파기**한다. **법정 보존 데이터만 분리 보관**한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 삭제 대상 계정 |
| state | 열거 {ACTIVE, DEACTIVATED, PURGING, PURGED} | 필수 · 기본 `ACTIVE` | 삭제 생애(§9.3) [D18] |
| purgeDueAt | 시각 | 선택 | 완전 삭제 예정(요청 +30일). NULL=삭제 미요청 [D18] |
| cascadeTargets | 목록<모듈 파기 대상> | 필수 · 파생 | 연쇄 파기 대상 목록(§9.2) [S6] |
| legalRetentionSet | 목록<보존 대상> | 필수 · 불변 | 법정 보존 집합(동의 증적·위치 로그·감사 로그 — 분리 보관·앱 삭제 권한 없음) [D18, N2, SECURITY-14] |
| requestedAt | 시각 | 선택 · 불변 | 삭제 요청 시각(재확인 완료 시점) |

### 9.2 연쇄 파기 대상 목록 (cascadeTargets — S6.5)

| 모듈 | 파기 대상 | 파기 시점 |
|---|---|---|
| M4 Saved Accommodation | 등록 숙소·위시리스트 | 만료 배치(AccountDeletionExpired) |
| M6 Trip Creation | 여행·필수 방문지 | 만료 배치 |
| M8 Itinerary | 일정(plan/current)·슬롯 | 만료 배치 |
| M12 Travel Archive | 방문 기록·changelog·사진(오브젝트 스토리지 객체 포함) | 만료 배치 |
| M13 AI Reflection | 회고·요약·스타일 분석 | 만료 배치 |
| M14 Notification | 알림함·스케줄·FCM 토큰 | 만료 배치 |
| M2 User Profile | 프로필·취향 | 만료 배치 |
| **위치 데이터(GPS)** | GPS 폴리라인·위치 파생 | **요청 즉시(유예 없음 D34)** |
| (후속) M15 Community | 공개 콘텐츠 → '삭제된 사용자' 익명화 | 만료 배치 |

**법정 보존(분리 보관·파기 제외)**: 동의 증적·위치정보 수집·이용 법정 로그·감사 로그.

### 9.3 삭제 상태 전이 (state)

```text
   ACTIVE ──requestAccountDeletion(재확인 완료)──▶ DEACTIVATED(즉시 비노출·세션 무효화·purgeDueAt=+30일)
     │                                                │ 위치 데이터 즉시 파기(유예 없음 D34)
     │ 30일 내 재로그인 → 복구(ACTIVE·purge 취소)      │
     ▼                                                ▼
   ACTIVE(복구)                              purgeDueAt 경과(J6 만료 배치) → PURGING(AccountDeletionExpired 발행)
                                              모듈별 독립 TX 파기(멱등·부분 진행 허용) → 전 핸들러 완료 → PURGED(최종 삭제)
                                              (법정 보존 집합만 잔존)
```

### 9.4 불변식

- **INV-DEL1 (소프트 삭제·30일 유예·복구 — D18)**: 삭제 요청은 즉시 전 표면 비노출·세션 무효화·purgeDueAt(+30일) 기록이며, 30일 내 재로그인으로 복구(ACTIVE 복원·purge 취소) 가능하다. 유예 중 동일 식별자 재가입 제한(C4) [D18, C4, S6.1·3].
- **INV-DEL2 (연쇄 완전성·법정 보존 분리 — 하드 계열)**: 만료 배치 후 잔존 데이터는 **법정 보존 집합(legalRetentionSet)과 정확히 일치**한다 — cascadeTargets의 전 모듈 소유 데이터가 파기되고 법정 보존(동의 증적·위치 로그·감사 로그)만 남는다(잔존 = 법정 집합) [D18, S6.5, PBT U8-P5].
- **INV-DEL3 (모듈별 멱등·독립 TX)**: 각 모듈 파기 핸들러는 이미 삭제된 리소스를 skip(멱등)하고 독립 TX로 부분 진행을 허용하며(`PURGING` 상태가 진행 정본), at-least-once 재전달로 재시도한다 — 전 핸들러 완료 확인 후에만 계정 레코드 최종 삭제(PURGED) [S6.5, FD-U8-06, PBT U8-P5].
- **INV-DEL4 (위치 데이터 즉시 파기)**: 위치 데이터(GPS 폴리라인·위치 파생)는 삭제 요청 즉시 파기(30일 유예 없음)되고 위치 법정 로그만 분리 보관된다 — 여타 데이터의 30일 유예와 구분 [D34/N2, S6.2, INV-LOC3].
- **INV-DEL5 (삭제 재확인·사전 고지)**: 계정 삭제는 등록 숙소·여행 기록·회고 등 삭제 대상을 사전 고지하고 재확인을 받은 후에만 진입한다 — 확인 없는 삭제 없음 [D18, US-E09-09].

### 9.5 DataExportRequest — 데이터 내보내기 (S9, G101)

**텍스트 데이터 JSON 즉시 다운로드**. 읽기 전용 수집(각 모듈 퍼사드)·**사진 파일 제외**(메타만)·동기 스트리밍. 특정 모듈 수집 실패 시 해당 섹션 오류 마커 포함(침묵 누락 금지).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| exportId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account 참조) | 필수 | 요청 계정(본인 재인증 필수) [S9] |
| sections | 목록<{module, status: OK\|ERROR}> | 필수 | 수집 섹션별 결과(부분 실패 마커) [S9, G101] |
| requestedAt | 시각 | 필수 · 불변 | 요청 시각(rate-limit 시간당 1회·감사 로그) [S9, SECURITY-03] |

**포함 섹션(읽기 전용)**: M2 프로필·취향, M6 여행·필수 방문지, M8 일정(plan/current), M12 방문 기록·메모·changelog(사진 파일 제외·메타만), M13 회고, M14 알림 설정.

**불변식**
- **INV-EXPORT1 (완전성·부분 마커 — G101)**: 내보내기는 포함 섹션 전부를 수집하고, 특정 모듈 수집 실패 시 해당 섹션에 오류 마커를 포함한다 — 부분 성공을 완전본으로 위장하지 않는다(침묵 누락 금지) [S9, G101, PBT U8-P8].
- **INV-EXPORT2 (사진 제외·즉시 다운로드)**: 사진 파일은 제외하고 메타만 포함하며 JSON 즉시 다운로드로 응답한다(비동기 잡 없이 동기) — 사진 포함 전체 아카이브는 후속(G186) [G101, G186, S9].
- **INV-EXPORT3 (민감 작업 방어)**: 내보내기는 rate-limit(계정당 시간당 1회)+본인 재인증을 요구하고 감사 로그를 남긴다 [S9, SECURITY-03].

---

## 10. MyPageView — 마이페이지 읽기 전용 집계 (BFF, G103)

마이페이지의 **읽기 전용 집계**(저장 안 함). 숙소 통합 목록(G103)·여행 구분 조회(예정/진행/종료)·스타일 요약 카드를 각 모듈 퍼사드로 조립한다. **소유는 각 모듈**(M4·M6·M13), U8은 조립·표시·출발점 전환 위임만 소유한다.

### 10.1 집계 슬롯

| 슬롯 | 데이터 출처(퍼사드) | 페이로드 개요 | 빈 상태 규칙 | 근거 |
|---|---|---|---|---|
| stayList | M4 `listMyStays` | 저장/등록 통합 목록+출처 라벨(외부 OTA 예약 등록/앱 내 저장)·숙소명·위치·체크인/아웃·연결 여행·저장 항목 '등록하기' 액션 | 0건이면 "숙소를 탐색하고 등록하면…"+탐색 진입 | G103, D15 |
| tripList | M6 `listTrips` | 여행 예정/진행 중/종료 구분(D19 전이·D21로 진행 중 최대 1개)·목적지·기간·등록 숙소 수·일정 수·회고 진입 | 지난 여행 0건이면 "아직 종료된 여행이 없습니다"·회고 진입 숨김 | D19, D21, US-E09-07 |
| styleSummaryCard | M13 `getStyleSummaryCard` | 대표 디스크립터 한 줄+선호 카테고리 칩+특성 dot 게이지·분석 여행 수·마지막 갱신 | 누적 방문 <10곳이면 "방문 장소가 10곳 이상 쌓이면…(현재 N곳)" 안내 | US-E09-08, G76 |

### 10.2 불변식

- **INV-MYPAGE1 (읽기 전용 집계 — G103)**: MyPageView는 각 모듈 퍼사드를 읽어 조립하는 읽기 전용 뷰이며 도메인 상태를 생성·변경하지 않는다 — 소유는 M4·M6·M13, U8은 조립·표시만 [G103, FD-U8-09].
- **INV-MYPAGE2 (출발점 전환 위임·G97)**: 숙소 '일정 출발점으로 등록/해제' 전환은 M4 퍼사드로 위임하고, 출발점 변경 시 해당 여행 재생성 여부를 확인하며 거부 시 리마인드 중단+재생성 배지(G97)로 이어진다 — U8은 전환 UI·확인만 [G103, G97, INV-SCHED3].
- **INV-MYPAGE3 (스타일 게이트 승계)**: 스타일 요약 카드의 10곳 게이트는 U7(M13) 정본을 승계한다(재정의 없음) — 미달 시 안내(현재 N곳)만 표시 [G76, U7 INV-STYLE1].
- **INV-MYPAGE4 (진행 중 여행 단수)**: tripList의 '진행 중'은 D21로 항상 최대 1개다 [D21, US-E09-07].

---

## 11. CP5 소비 참조 계약 (U3·U5·U6·U7 → U8 입력) — 알림 이벤트

U8은 4개 유닛이 CP5로 발행하는 이벤트를 **단일 구독·스케줄링(D32)**한다(스키마 정본은 발행 유닛, 본 유닛 재정의 금지). 공통 envelope는 `common/core`(U1 계약) 위에서 동작하며, 미지 유형은 무시가 아닌 계측 대상으로 처리한다(침묵 실패 금지).

| 입력 이벤트(발행 유닛) | U8 소비 용도 | 소비 시 처리 |
|---|---|---|
| `StayRegistered`(U3) / `StayLinkedToTrip`(U4) | 숙소 등록/저장 완료 알림 | `sendEvent` — 숙소명·위치·체크인/아웃+출발점 문구(날짜 없으면 "날짜 입력" 안내) [US-E09-01] |
| `ItineraryConfirmed`(U5) | 리마인드 세트 일괄 스케줄 | `scheduleTripNotifications` — D-1·당일 08시·개별 N분 전(INV-SCHED1) [D32] |
| `ItineraryChanged`(U5) | 리마인드 전면 재계산 | `rescheduleOnItineraryChange` — 전량 무효화 후 재계산(멱등 dedupeKey INV-SCHED2) [D32] |
| `TriggerFired`(U6) | Plan-B 재계획 알림 | `sendEvent` — severe만 푸시·10분 중복 억제·진행 중 방해금지 예외(INV-DND2)·위치 철회 시 위치 비의존만 [G100, D34] |
| `ReflectionReady`(U7) | 회고 완료 알림 | `sendEvent` — "여행 기록이 정리되었습니다"·회고 진입(데이터 없으면 "회고 생성 못 함" 안내) [US-E09-04] |
| `AccountDeletionExpired`(U1) | 알림 데이터 연쇄 파기 | M14 파기 핸들러 — 알림함·스케줄·FCM 토큰 삭제(멱등 INV-DEL3) [S6.5] |

**검증 방법 — 통합 테스트 시나리오(CP5)**
1. **유형별 파이프라인 전수**: 5계열 이벤트 각각에 대해 종류별 토글→방해금지(22~08)→중복 억제(10분 창)→FCM+인앱 적재의 전 단계를 통과·차단 조합으로 검증(꺼진 종류 발송 0·억제분 알림함 적재).
2. **리마인드 재계산 멱등성**: `ItineraryChanged` 수신 시 기존 스케줄 재계산·동일 이벤트 중복 수신에도 스케줄 중복 생성 0(dedupeKey 멱등).
3. **방해금지 예외 분기**: 방해금지 창 내 `TriggerFired`(severe·진행 중 여행)만 즉시 발송, 경미·비진행 여행은 억제 후 인앱 적재.

> **계약 변경 통제**: envelope는 `common/core`(U1) 소유 — 변경은 발행 4유닛(U3·U5·U6·U7) 회귀 유발. 유형 추가(U10·U11)는 하위 호환 확장(미지 유형 계측 처리, FD-U8-13) [CP5, unit-of-work-dependency §CP5].

---

## 12. 홈 카드 최종 통합 검증 (U2 슬롯 × U3~U7 데이터) — §10 연계

U8은 마감 유닛으로서 U2가 고정한 HomeDashboardModel 슬롯 스키마(재정의 금지)에 전 유닛 데이터가 부분 응답·침묵 실패 금지 원칙대로 전량 채워짐을 **통합 검증**한다(신규 슬롯 없음).

| U2 슬롯 | 데이터 공급 유닛 | U8 최종 통합 검증 요점 |
|---|---|---|
| activeTripCard | U6 `M18.getActiveHub`·`getTripProgress` | 진행 중 여행 카드·진행률(여행 중만)·execution 허브 진입 |
| upcomingTripCard | U4 `M6.listTrips` | 예정 여행·D-day·0건 emptyState |
| trendingPlaces | U3 `M7.getTrendingPlaces` | 인기 장소·조회 실패 시 슬롯 생략(부분 응답) |
| memoryCard | U7 `M12.getVisitStats` | 추억 진입·기록 0건 미노출 |
| preferencePromptCard | U1 `M2.getPendingPreferencePrompts` | 점진 취향 카드 |
| **topBar** | U1 세션 + **U8 `M14.getInbox`** | **워드마크 + 미읽음 배지 카운트(U8 공급분)** |
| communityRecordCard | (U10 후속) | 미노출 게이트 유지(INV-HD4 승계) |

> **경계(FD-U8-12)**: 슬롯 스키마·부분 응답·미노출 규칙은 **U2 소유**(u2 §4.1 INV-HD1~5 재정의 금지). U8은 미읽음 배지 데이터 공급 + 전 슬롯 채움 통합 검증만 담당 [u2 §4.1, US-E2-02].

---

## 13. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| NotificationSchedule | US-E09-01·02·04 | D32, D25, G97 |
| Notice(알림함) | US-E09-05, US-E2-02 | D32, G99, G100 |
| NotificationToggle | US-E09-05 | D32, Δ10/C12 |
| QuietHours(DndSetting) | US-E09-02·03 | G100 |
| PushToken | US-E09-05 | D12, D36 |
| MarketingConsent | US-E09-14 | N8 |
| LocationConsentState | US-E09-11 | D34/N2, G182 |
| AffiliateNoticePref | US-E09-12 | US-E09-12 |
| AccountDeletionCascade | US-E09-09 | D18, S6, C4, D34 |
| DataExportRequest | US-E09-09 | G101, S9, G186 |
| MyPageView | US-E09-06·07·08 | G103, G97, D15, D19, D21, G76 |
