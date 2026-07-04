# U8 알림·마이페이지·설정 — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 5종 + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U8-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) §M14, 서비스 정본 [services.md](../../../inception/application-design/services.md) **S5(알림 스케줄링·발송)·S6(계정 삭제 연쇄)·S9(데이터 내보내기)**
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 전송·저장·FCM·스케줄러 큐·remote config 구현체는 NFR/Infra 단계 소유.
> **마감 유닛 원칙(FD-U8)**: U8은 신규 도메인 로직(M14 발송 파이프라인·알림함)만 신규이고, 마이페이지·설정·삭제 연쇄·내보내기·홈 카드는 **각 모듈 퍼사드 소비·통합**이다 — 소유는 각 모듈, U8은 오케스트레이션·UI·연쇄 완결.

---

## FLOW-1 알림 스케줄링 (CP5 이벤트 구독 → 스케줄 → 재계산)

**진입**: `ItineraryConfirmed` ∨ `ItineraryChanged` ∨ 거점 해제 → `M14.scheduleTripNotifications` / `rescheduleOnItineraryChange` / `cancelTripReminders` (정본 서비스 S5)
**관련**: US-E09-02 · BR-U8-01·02·04 · D32, D25, G97 · services S5

```text
S1 확정 → 리마인드 세트 스케줄 [BR-U8-01] — CP5 ItineraryConfirmed 소비
   NotificationSchedule 생성: D-1 · 당일 요약(기본 08:00·설정) · 개별 일정 N분 전(0/15/30/60·기본 30)
   payload{장소명·시작 시각·이동 거리만 — 소요시간 없음 D25 INV-SCHED4} · dedupeKey=tripId+slotRef+type
   서버 스케줄 통일(클라 로컬 스케줄 0 INV-SCHED1)
   └─ E1 등록 숙소 없음(일정 없음) → "숙소 등록하면 일정 알림" 유도 1회만(반복 0 INV-SCHED5) [BR-U8-01]
S2 일정 변경 → 전면 재계산 [BR-U8-02] — CP5 ItineraryChanged 소비
   해당 tripId SCHEDULED 전량 CANCELLED → 재계산(dedupeKey 멱등 — 중복 수신에도 중복 생성 0 INV-SCHED2)
S3 거점 해제·재생성 거부 → cancelTripReminders [BR-U8-04]
   기존 일정 유지 + 리마인드 중단(CANCELLED) + 재생성 유도 배지 신호(G97 INV-SCHED3)
S4 TripEnded → 잔여 리마인드 취소
   관련 SCHEDULED 잔여분 CANCELLED
S5 fireAt 도달(J4) → FLOW-2 발송 파이프라인
```

**멱등성**: `ItineraryChanged` 재계산은 (tripId, 일정 상태)에 결정적, dedupeKey로 중복 스케줄 0. **사후 조건**: 서버 스케줄 통일(INV-SCHED1), 재계산 멱등(INV-SCHED2), 소요시간 부재(INV-SCHED4).

---

## FLOW-2 발송 파이프라인 (토글 → 방해금지 → 억제 → FCM+알림함 선적재)

**진입**: due 리마인드(J4) ∨ 이벤트성 알림 → `M14.sendEvent` / 발송 파이프라인(S5)
**관련**: US-E09-01·03·04·05 · BR-U8-05~09 · D12, D32, G100 · services S5

```text
S1 종류별 토글 확인 [BR-U8-05] — 단일 게이트 순서 고정
   perType[type].{push, inApp} 평가(발송 시점 최신 토글 INV-TOGGLE1)
   └─ B1 종류 완전 off → 종결(적재도 없음·사용자 의사 INV-NOTICE2)
S2 방해금지 창 [BR-U8-06]
   isWithinWindow(fireAt)(22~08·자정 넘김 경계 정확 INV-DND1)
   ├─ B2 창 내 · (진행 중 여행 ∧ PLAN_B ∧ severe) → 예외 통과(즉시 푸시 INV-DND2)
   └─ B3 창 내 · 비예외 → suppressReason=DND(푸시 보류·인앱 적재는 유지 INV-DND3)
S3 중복 억제 [BR-U8-07]
   dedupeKey 10분 창 내 재발송 → suppressReason=DEDUPE(Plan-B 동일 일정 10분 1회)
   └─ E1 위치 동의 철회 상태 → 이동 지연 기반 알림 미평가·위치 비의존(날씨·휴무)만 [BR-U8-07·D34]
   └─ E2 외부 API 실패 → 허위 알림 금지(침묵·M9 연동·다음 정상 응답 재평가) [BR-U8-07]
S4 TX: 인앱 알림함 선(先) 적재 [BR-U8-09]
   Notice 적재(90일 보존 G99·채널 독립 — 억제·푸시 OFF여도 종류 on이면 적재 INV-NOTICE1)
   deliverySource ∈ {FIRED, SUPPRESSED_DND, SUPPRESSED_DEDUPE}
S5 FCM 발송(TX 밖) [BR-U8-08] — D12 단일 채널
   대상 ACTIVE 토큰만(타 계정 0 INV-TOKEN1)
   ├─ E3 FCM 실패 → 재시도 3회 지수 백오프 → dead letter+계측(인앱 이미 적재·침묵 실패 금지 G144)
   └─ E4 FCM 무효 토큰 → STALE 정리(INV-TOKEN2)
S6 시스템 알림 예외 [BR-U8-11]
   SYSTEM_SECURITY → 전 채널 OFF에도 인앱 적재·표시(INV-NOTICE4)
```

**사후 조건**: 꺼진 종류 발송 0(B1), 창 내 비예외 발송 0(B3), 10분 창 중복 0(S3), 억제분 인앱 적재(S4), 시스템 알림 강제 표시(S6). **결정 함수**: 발송 판정(토글×방해금지×중복)은 순수 함수로 분리(U8-P2·P3).

---

## FLOW-3 계정 삭제 연쇄 (S6 · 유예 · 만료 배치 · 법정 보존)

**진입**: 설정 계정 삭제 → `M1.requestAccountDeletion` → (30일 후) 만료 배치 J6 → `AccountDeletionExpired` 구독(모듈별 파기)
**관련**: US-E09-09 · BR-U8-13~15 · D18, D34/N2, C4 · services S6

```text
S1 삭제 요청 [BR-U8-13] — 재확인 필수
   삭제 대상 사전 고지(등록 숙소·여행 기록·회고) + 재확인(INV-DEL5)
   TX: state ACTIVE→DEACTIVATED · purgeDueAt=+30일 · 전 세션/토큰 무효화 · 즉시 비노출
   유예 중 동일 식별자 재가입 제한(C4)
S2 위치 데이터 즉시 파기 [BR-U8-15] — 유예 없음(D34)
   GPS 폴리라인·위치 파생 즉시 파기(INV-DEL4) · 위치 법정 로그(append-only)는 분리 보관 유지(N2)
S3 30일 내 재로그인 → 복구
   state ACTIVE 복원 · purge 취소(INV-DEL1)
S4 만료 배치(J6) → PURGING [BR-U8-14]
   purgeDueAt 경과 → state PURGING + outbox AccountDeletionExpired 발행
S5 모듈별 파기 핸들러(구독) [BR-U8-14] — 각 모듈 소유·멱등·독립 TX
   M4 등록 숙소 · M6 여행·필수 방문지 · M8 일정 · M12 기록·changelog·사진(스토리지 객체) · M13 회고 · M14 알림함·스케줄·토큰 · M2 프로필·취향
   이미 삭제 리소스 skip(멱등 INV-DEL3) · 부분 진행 허용(PURGING 진행 정본) · at-least-once 재시도
   (후속) M15 커뮤니티 → '삭제된 사용자' 익명화
S6 완료 → PURGED
   전 핸들러 완료 확인 → 계정 레코드 최종 삭제
   잔존 = 법정 보존 집합(동의 증적·위치 로그·감사 로그)과 정확히 일치(INV-DEL2)
```

**사후 조건**: 소프트 삭제·복구 가능(INV-DEL1), 잔존=법정 집합(INV-DEL2·하드), 모듈별 멱등·독립 TX(INV-DEL3), 위치 즉시 파기(INV-DEL4).

---

## FLOW-4 데이터 내보내기 (S9 · 읽기 전용 수집 · 부분 마커)

**진입**: 설정 데이터 내보내기 → `M1.requestDataExport`
**관련**: US-E09-09 · BR-U8-16 · G101, G186 · services S9

```text
S1 요청 [BR-U8-16] — 민감 작업 방어
   rate-limit(계정당 시간당 1회) + 본인 재인증(INV-EXPORT3)
S2 읽기 전용 수집(각 모듈 퍼사드·TX 불요) [BR-U8-16]
   M2 프로필·취향 · M6 여행·필수 방문지 · M8 일정(plan/current) · M12 방문 기록·메모·changelog(사진 파일 제외·메타만 G101) · M13 회고 · M14 알림 설정
   └─ E1 특정 모듈 수집 실패 → 해당 섹션 오류 마커 포함(침묵 누락 금지·완전본 위장 0 INV-EXPORT1)
S3 JSON 스트리밍 직렬화 → 즉시 다운로드 [BR-U8-16]
   동기(비동기 잡 없이 — 1차 규모) · 사진 제외(INV-EXPORT2)
   └─ E2 전체 실패 → 오류+재시도 안내
S4 감사 로그 기록 [BR-U8-16]
   내보내기 이력 감사 로그(SECURITY-03) · 사진 포함 전체 아카이브는 후속(G186)
```

**사후 조건**: 완전성+부분 마커(INV-EXPORT1), 사진 제외·즉시(INV-EXPORT2), 민감 작업 방어(INV-EXPORT3).

---

## FLOW-5 설정 변경 반영 + 홈 카드 최종 통합

**진입**: 설정 화면 토글·값 변경 → 각 모듈 설정 API / 홈 대시보드 진입 → 홈 집약 조회
**관련**: US-E09-05·10·11·12·14, US-E2-02 · BR-U8-10·17~26 · D32, D34, N8, D26

```text
S1 알림 설정 변경 [BR-U8-10·12] — 즉시 저장
   updateToggles(perType) · updateQuietHours(window) · 오프셋·당일 시각·민감도
   별도 저장 버튼 없이 즉시 저장 → 다음 알림부터 반영(발송 시점 평가 INV-TOGGLE1)
S2 위치 3층 설정 [BR-U8-17·18] — 철회 영향 고지
   3층 구분 표시·제어(OS×법정×옵트인 G182)
   ├─ B1 철회 → 영향 고지+재확인(INV-LOC2) → 위치 의존 비활성·위치 비의존 유지(INV-LOC1)
   ├─ B2 GPS 옵트인 철회·탈퇴 → 위치 데이터 즉시 파기(U7 purgeLocationData INV-LOC3)
   └─ B3 OS 권한 거부 → 앱 내 토글 비활성+설정 이동 안내(INV-LOC4)
S3 마케팅 동의 토글 [BR-U8-19] — 수집·관리만
   부여/철회 즉시 저장+증적(1차 발송 없음 INV-MKT1)
S4 취향 직접 설정 [BR-U8-23] — 직접 설정 우선
   온보딩 동일 선택지 → 다음 AI 생성·재계획 입력 반영(직접 설정 > 자동 스타일)
   예산 총액 기준(항공 제외 D26)·1인/1일 파생 표기
S5 제휴 고지 토글 [BR-U8-25]
   AffiliateNoticePref(다시 보기·재활성 INV-AFF1)
S6 홈 카드 최종 통합 검증 [BR-U8-26] — U2 슬롯 채움
   HomeDashboardModel(U2 소유 슬롯) × {U6 활성·U4 예정·U3 인기·U7 추억·U1 취향·U8 미읽음 배지 getInbox}
   부분 응답·침묵 실패 금지(u2 INV-HD1 승계)·communityRecordCard 미노출 게이트 유지(INV-HD4)
```

**사후 조건**: 설정 즉시 반영(INV-TOGGLE1), 위치 철회 시 위치 비의존 유지(INV-LOC1), 마케팅 1차 미발송(INV-MKT1), 홈 슬롯 전량 채움(부분 응답 완전성 u2 INV-HD1).

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장, D37 계층 분리). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check. 시드 로깅·수축(shrinking) 필수(PBT-08). **U8 PBT 효용 최대 영역은 발송 판정 함수(토글×방해금지×억제)와 삭제 연쇄 완전성(법정 보존 분리)**이며, 외부(FCM·스케줄러)만 어댑터 fake(D37). 아래 속성은 U8 Code Generation의 DoD 항목이며, **unit-of-work §U8 DoD 명시 속성**((a)발송 파이프라인 결정 함수·(b)리마인드 재계산 멱등성·(c)삭제 연쇄 후 잔존=법정 보존 집합·(d)알림함 90일 보존 경계)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**M14 Notification (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U8-P1 | 리마인드 재계산 멱등성 | **알림 스케줄 재계산 멱등**: 임의 (일정 확정→변경 시퀀스)에 대해 (a) `ItineraryChanged` 재계산은 (tripId, 일정 상태)에 결정적(동일 상태→동일 스케줄), (b) 동일 이벤트 중복 수신에도 dedupeKey(tripId+slotRef+type)로 스케줄 중복 생성 0(멱등 INV-SCHED2), (c) 변경 시 기존 SCHEDULED 전량 무효화 후 재계산(잔류 스케줄 0), (d) 재계산 결과 payload에 소요시간 필드 부재(INV-SCHED4) | 일정 확정·변경 이벤트 시퀀스 생성기(슬롯 추가/삭제/시각 이동 혼합) + 중복 이벤트 주입 |
| U8-P2 | 발송 파이프라인 결정 함수 | **발송 판정 결정성·토글**: 임의 (종류 토글 × 채널 × 알림 유형) 입력에 대해 (a) **꺼진 종류 발송 0**(종류 완전 off면 적재도 없음 INV-NOTICE2), (b) 종류×채널 독립(푸시 off + 인앱 on 유효 — 인앱 적재 유지 INV-NOTICE1·TOGGLE2), (c) 발송 시점 최신 토글 평가(변경 전 스케줄에도 최신 적용 INV-TOGGLE1), (d) 판정은 동일 입력에 결정적(순수 함수) | 토글 조합 생성기(종류×{push,inApp}×on/off) + 알림 유형 분포 + 토글 변경 타이밍 주입 |
| U8-P3 | 방해금지 창 판정·중복 억제 | **방해금지 결정성·경계 + 억제 창 불변**: 임의 (방해금지 창 × 발송 시각 × 종류 × 여행 상태 × 심각도)에 대해 (a) isWithinWindow 결정적·**자정 넘김(22~08) 경계 정확**(경계값 오분류 0 INV-DND1), (b) **창 내 비예외 발송 0**·예외는 (진행 중 ∧ PLAN_B ∧ severe) 단 하나(INV-DND2), (c) **10분 창 중복 발송 0**(동일 dedupeKey), (d) **억제분(DND/DEDUPE)은 인앱 알림함 적재**(푸시만 보류 INV-DND3·NOTICE1) | 방해금지 창 생성기(자정 넘김 포함) + 발송 시각(경계 분포) + 여행 상태·심각도 + 10분 창 내 재발송 주입 |
| U8-P4 | 토글 즉시 반영 | **설정 즉시 반영 왕복**: 임의 (설정 변경 시퀀스)에 대해 (a) 토글·방해금지·오프셋 변경이 즉시 저장되어 이후 발송 판정에 반영(발송 시점 평가 INV-TOGGLE1), (b) 설정 직렬화 왕복 무손실(perType·오프셋·창 보존), (c) 변경 순서 무관하게 최종 상태 = 마지막 변경(last-write-wins 결정성) | 설정 변경 시퀀스 생성기(토글·창·오프셋 혼합) + 순서 셔플 |
| U8-P5 | 삭제 연쇄 완전성 (1급·하드 계열) | **삭제 연쇄 후 잔존 = 법정 보존 집합**: 임의 (계정 소유 데이터 집합 × 파기 이벤트 순서)에 대해 (a) 만료 배치 후 **전 모듈 소유 데이터 파기**(cascadeTargets 완전 커버 — 잔존 사용자 데이터 0), (b) **잔존 = 법정 보존 집합(동의 증적·위치 로그·감사 로그)과 정확히 일치**(법정 분리 불변식 INV-DEL2), (c) 모듈별 핸들러 멱등(재적용·중복 파기 0 INV-DEL3), (d) 파기 순서 셔플·부분 재시도에도 최종 수렴 동일(독립 TX·at-least-once), (e) 위치 데이터는 요청 즉시 파기(30일 유예 없음 INV-DEL4) | 계정 소유 데이터 집합 생성기(전 모듈 분포+법정 보존 항목) + 파기 이벤트 순서 셔플 + 부분 실패/재시도 주입 |
| U8-P6 | 알림함 보존 경계 | **인앱 알림함 90일 보존 경계**: 임의 (적재 시각 × 현재 시각)에 대해 (a) createdAt+90일 초과분만 J5 정리(경계 정확·조기 삭제 0 INV-NOTICE3), (b) 미읽음 배지 카운트 = readAt NULL ∧ 미만료 Notice 수(결정적), (c) 읽음 처리 멱등(중복 markRead 무해) | 적재 시각 시퀀스 생성기(90일 경계 분포) + 읽음 처리 주입 |

**M1.application / 설정 (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U8-P7 | 위치 3층 조합 매트릭스 | **위치 3층 조합 동작 결정성**: 임의 (OS 권한 × 법정 동의 × GPS 옵트인) 8조합에 대해 (a) 위치 의존 기능(이동 지연·실시간 Plan-B·현위치 추천·GPS 기록)은 3층 전부 충족 시에만 활성(매트릭스 domain-entities §7.1 일치 INV-LOC1), (b) **위치 비의존 기능(예정 일정 알림·날씨·휴무 트리거)은 전 조합에서 동작**(철회가 앱 무력화 0), (c) OS 권한 거부면 하위 층 무효(상위 게이트 INV-LOC4), (d) GPS 옵트인 철회 시 위치 데이터 파기 트리거·법정 로그 잔존(INV-LOC3) | 3층 조합 생성기(8조합 전수) + 기능 종류 주입 |
| U8-P8 | 데이터 내보내기 완전성 | **내보내기 완전성·부분 마커**: 임의 (모듈 수집 성공/실패 조합)에 대해 (a) 포함 섹션 전부 시도(완전성 — 침묵 누락 0 INV-EXPORT1), (b) 실패 섹션에 오류 마커 포함(부분 성공을 완전본 위장 0), (c) 사진 파일 제외·메타만(INV-EXPORT2), (d) 성공 조합에서 직렬화 왕복 무손실 | 모듈 수집 결과 조합 생성기(성공/실패 분포) + 데이터 유형 주입 |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| M14 FCM 발송(sendEvent FCM 경로) | No PBT properties identified | 외부 FCM 전송·재시도는 어댑터 fake(D37) — 인앱 이중화·재시도 3회·무효 토큰 정리는 예시 기반 통합 테스트, 대상 격리는 계약 테스트 |
| MyPageView 집계(마이페이지 조립) | No PBT properties identified | 읽기 모델 집약(숙소 통합·여행 구분·스타일 카드)·각 모듈 퍼사드 소비 — 게이트·분류 정본은 상류(M4·M6·M13), 빈 상태는 예시 기반 |
| 정책 재열람·제휴 고지·문의(고객 지원) | No PBT properties identified | 정적 문서 표시·고지 문안·토글 — 예시 기반. 다시 보기 재활성은 상태 토글 계약 테스트 |
| 홈 카드 최종 통합(HomeDashboard) | No PBT properties identified | 슬롯 조립·부분 응답 완전성 PBT는 **U2-P4 소유**(U8은 미읽음 배지 공급+통합 검증만) — U8은 CP5 통합 시나리오로 검증 |

### 커버리지 대조 (U8 DoD → 속성)

| U8 DoD 명시 속성(unit-of-work §U8) | 대응 |
|---|---|
| 발송 파이프라인 결정 함수(꺼진 종류 0·창 내 비예외 0·10분 창 중복 0·억제분 알림함 적재) | **U8-P2 + U8-P3** |
| 리마인드 재계산 멱등성(동일 일정 상태→동일 스케줄) | U8-P1 |
| 삭제 연쇄 후 잔존 = 법정 보존 집합과 정확히 일치 | **U8-P5(1급·하드 계열)** |
| 알림함 보존 경계(90일) | U8-P6 |
| (프롬프트 필수) 토글 즉시 반영 | U8-P4 |
| (프롬프트 필수) 위치 3층 조합 동작 매트릭스 | U8-P7 |
| (프롬프트 필수) 데이터 내보내기 완전성 | U8-P8 |

**속성 합계: 8개** (M14: 6[P1·P2·P3·P4·P5·P6] · M1/설정: 2[P7·P8]) — 프롬프트 필수 7속성(알림 스케줄 재계산 멱등·방해금지 창 판정 결정성/경계·중복 억제 창 불변·토글 즉시 반영·삭제 연쇄 완전성/법정 분리·데이터 내보내기 완전성·위치 3층 조합 매트릭스) 전부 포함 + 알림함 90일 보존 경계 확장.

### 하드 제약(D37) 대조 — U8은 계정 무결성 소관(마감 유닛)

| 하드 제약 | U8 방어 속성·불변식 |
|---|---|
| **계정 무결성**(삭제 연쇄 완전성·법정 보존 분리·타 사용자 알림 미수신) | **U8-P5**(잔존=법정 집합) · INV-DEL2·3·4 · INV-TOKEN1·INV-NOTICE5(대상 격리) · BR-U8-14·15 |
| **발송 무결성**(꺼진 종류 발송 0·방해금지 예외 정확·중복 억제) | U8-P2·U8-P3 · INV-DND1·2 · INV-NOTICE1·2 · BR-U8-05·06 |
| **위치 데이터 파기**(옵트인 철회·탈퇴 즉시 파기 N2) | U8-P7(d) · INV-LOC3·DEL4 · BR-U8-15 · U7 INV-GPS2 재사용 |
| **개인정보 주권**(내보내기 완전성·침묵 누락 금지) | U8-P8 · INV-EXPORT1 · BR-U8-16 |

> **마감 유닛 통합(FD-U8-12·13)**: U8은 CP5(U3·U5·U6·U7) 4계열 이벤트를 단일 구독·스케줄링(D32)하고, 홈 카드 슬롯 스키마는 U2 소유(재정의 금지)를 통합 검증만 한다. 미지 알림 유형(U10·U11)은 무시가 아닌 계측 대상(침묵 실패 금지 G144). 삭제 연쇄는 각 모듈이 `AccountDeletionExpired` 파기 핸들러를 소유하는 이벤트 계약 위에서 완결한다.
