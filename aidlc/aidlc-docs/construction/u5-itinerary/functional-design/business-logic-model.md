# U5 AI 일정 생성·확정 — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 5종(+같이 고르기 서브플로우) + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U5-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) M8·C1·C2, 서비스 정본 [services.md](../../../inception/application-design/services.md) S1
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 최적화 알고리즘·전송·저장·티어·remote config 구현체는 NFR/Infra 단계 소유.

---

## FLOW-1 일정 생성 (컨텍스트 로드 → 후보 풀 → LLM 점수 → 솔버 배치 → 첫날 5초 → 백그라운드)

**진입**: 생성 방식 선택 → `M8.generateItinerary(tripId, mode)` (정본 서비스 S1)
**관련**: US-E5-01~05, US-E5-09, US-E5-10 · BR-U5-01~24 · D11, D13, D14, D25, D28, D29, D38, G40, G41, G46, G48, G50, G51, G106, G115, G119, G129, G161

```text
S1 세션 개설 [BR-U5-21·22]
   M8.generateItinerary → GenerationSession(status=COLLECTING), 진행 스트림 채널(단계·진행률·경과)
   가드: 등록 숙소 존재 ∨ 나중 등록 온램프(NO_BASE 예외) [BR-U5-02]
   ├─ E1 날짜 누락·역전 → 생성 안 함, 날짜 수정 화면 [BR-U5-03, INV-ITIN3]
   └─ E2 거점 좌표 미확정(지오코딩 실패) → 지도 핀 지정 요청, 생성 보류 [BR-U5-04]
S2 컨텍스트 병렬 로드 (CP2 소비) [BR-U5-05·10]
   M6.getTrip(날짜·속성 G134·시간창 D29/G119·필수 방문지 G40)、M2.getPreferences(중립 기본값 채움)、M4.listBaseAssignments(거점·체크인/아웃)
   방어 재검증: D21(겹침)·D15(비중첩)·G40(한도)·G120(국내 범위) [CP2 시나리오 3]
   ├─ E3 M6·M4 로드 실패 → 세션 FAILED, 재시도 노출
   ├─ E4 M2 실패 → 중립 기본값 전량 대체(FAILED 아님 — 무실패 보장) [BR-U5-10, INV-SCORE3]
   └─ B1 첫날 거점 공백 → 여행지 중심 좌표 기본 거점(G41) / 위치 거부 → 등록 숙소 기준+가정 표기 [BR-U5-05]
S3 후보 풀 구성 — M7.getCandidatePool(closed-set G115) [BR-U5-06·07·08]
   canonical POI + 영업시간 + 체류 기본값 min/rec/max(G51) + 국내 좌표(G120). 저장 POI 투입분 사본(G129)
   ├─ E5 외부 소스 실패 → 정본 캐시로 축소 풀 + 기본 모드 플래그 / 정본도 불가 → 저장 POI+필수 방문지 최소 풀
   ├─ E6 소실(LOST)·미확인(UNVERIFIED) → 시드 제외 / '영업시간 미확인' 후보 분리 [BR-U5-07]
   └─ E7 조건 과협소 후보 0건 → 0 만든 조건 표시+완화 제안 / 예산 미충족 → 저비용 대안 우선 [BR-U5-08]
S4 LLM 선호 점수 — C1.call(PreferenceScoring, candidateIds, prefs) [BR-U5-09·11]
   경량 티어(D11), 전 일자 공용 1회 호출(INV-SCORE2), 서버 재조회 컨텍스트 주입(D31 — 내부 지표 미주입 SECURITY-11)
   closed-set 검증: 출력 스키마 + ID 화이트리스트 교차(환각 0, INV-SCORE1). 예산=소프트 가중치(하드 아님 G47)
   └─ E8 타임아웃(2.5초)·스키마 위반 → 규칙 점수 폴백(isFallback=true)+"기본 모드" 고지 [BR-U5-19]
S5 솔버 day1 배치 — C2.solve(day1) [BR-U5-13~18]
   하드 제약 4종: HC1 영업시간·HC2 이동 부등식(G106)·HC3 고정 블록 불가침·HC4 시간창(D29). 위반 배치 구조적 배제(INV-SOLVE1)
   포함 고정형=필수 노드+공간 앵커, 체류 압축 시 표시 사유(stayCompressedFlag), 전환일 편도(G50)
   ├─ E9 고정 블록 상호 충돌 → 자동 변경 없이 강조+사용자 질의 [BR-U5-15, INV-FIX2]
   ├─ E10 필수 방문지 배치 불가 → 사유+3안(다른 날짜/고정 해제/인접 조정) [INV-FIX3]
   └─ E11 하루 1개 이하만 가능 → "이동 거리가 멀어 하루 한 곳만" [BR-U5-16, INV-SOLVE4]
S6 TX1: day1 초안 저장(독립 TX) → 세션 DAY1_READY → 첫 1일 응답 반환 (5초 게이트) [BR-U5-21, INV-SESS2]
S7 백그라운드 잔여 일자 루프 — C2.solve(dayN), S4 점수 재사용(LLM 재호출 없음), 일자별 독립 TX [BR-U5-21]
   ├─ E12 20초 한계 → 잔여 결정론 단독 모드 완성 + DETERMINISTIC_ONLY 고지 [BR-U5-23]
   ├─ B2 사용자 '취소' → 부분 일정 초안 저장(CANCELLED_KEPT)+'이어서 생성'(resumeGeneration) [BR-U5-24, INV-SESS3]
   └─ B3 '백그라운드로 계속' → 완료 시 화면 복귀 시 반영
S8 TX-final: 세션 GENERATED + outbox ItineraryGenerated(전체 완료 TX에서만) [INV-SESS1]
   └─ E13 전 경로 실패 → 숙소+시각 고정 필수만 최소 일정 + "다시 시도"(MINIMAL_ONLY) [BR-U5-24, INV-SESS4]
```

**사후 조건**: 출력 POI ⊆ 후보 풀(환각 0, INV-SLOT1). 사용자 노출 시각·순서는 전부 C2 검증값(INV-SLOT2). 소요시간 표기 0(D25). 폴백 계단은 침묵 실패 없이 최소 일정까지 도달(INV-SESS4).

---

## FLOW-1b 같이 고르기 서브플로우 (mode=TOGETHER)

**진입**: mode=TOGETHER 선택 → S1~S4는 FLOW-1 공통, S5부터 슬롯 단위 루프
**관련**: US-E5-10 · BR-U5-35·37 · G48, D25

```text
T1 슬롯 컨텍스트 — 기준점 산출 [BR-U5-35]
   기준점 = 직전 확정 슬롯 위치(첫 슬롯 = 등록 숙소) [G48]
T2 후보 제시 — M8.getSlotCandidates(slotCursor, concept, radius)
   반경 내 후보(+반경 확대 시 예상 이동 거리 표기), 반경 밖 후보 disabled(회색) [BR-U5-35]
   └─ E1 반경 내 0건 → 반경 확대(다음 이동 수단)/컨셉 변경 제안
T3 사용자 선택 → M8.chooseSlotCandidate → C2 검증(하드 제약)+다음 슬롯 컨텍스트 반환 [BR-U5-13~16]
T4 루프 T1~T3 반복(하루 동선 완성)
T5 (선택) 방식 전환 — switchGenerationMode(→FullAuto) → 확정 슬롯 고정 블록 보존, 남은 일정만 완전 AI [BR-U5-37, INV-ITIN5]
```

**사후 조건**: 최종 시각·순서·이동 거리는 솔버 검증값(D25 소요시간 미표시). 도중 전환 시 진행분 손실 0(INV-ITIN5).

---

## FLOW-2 편집 재검증 (클라 경량 + 서버 확정)

**진입**: 일정 편집(추가·삭제·재정렬·시간 변경) → `M8.validateEdit` → `M8.applyEdit`
**관련**: US-E5-07, US-E5-08 · BR-U5-25~28, 36, 41 · D28, G49, D20

```text
S1 편집 조작 — 클라이언트 경량 검증기 즉시 재검증 [BR-U5-25]
   ConstraintSpec(C2 발행·버전 있음) 소비 — 영업시간·이동시간·고정 충돌 즉시 검사(UX용 사본)
   └─ E1 명세 버전 불일치 → 로컬 검사 보수적 비활성화(오판 방지 D28)
S2 위반 표시 — 비차단 경고 배지+사유(violations) [BR-U5-26]
   POI 추가 시 getInsertableSlots(현재 동선 기준 삽입 가능 시간대만)
S3 서버 확정 검증 — M8.validateEdit → C2.validate(정본) [BR-U5-25]
   가드: 확정 상태 일정이면 unlockForEdit 선행 필요(D20) [BR-U5-33]
S4 저장 — M8.applyEdit(onViolation) [BR-U5-27·28]
   ├─ B1 위반 있음 → "○곳에서 시간이 안 맞아요" 요약 + 'AI 자동 보정 / 그대로 저장'
   │   ├─ AutoFix → C2.repair(시각·순서만·POI 불변 G49) [BR-U5-28]
   │   │     └─ E2 RepairInfeasible → '그대로 저장'/수동 수정 유도
   │   └─ SaveAsIs → 위반 보존+지속 가시화(INV-SLOT3) [BR-U5-27]
   └─ B2 위반 없음 → 저장(시각·순서·고정 블록 보존) [BR-U5-41]
S5 저장 결과 — changelog 기록, 실행 기준선 갱신(CP3) [BR-U5-41]
   └─ E3 네트워크 오류 → 로컬 임시 보관("저장 대기 중" pendingSyncFlag)+재연결 자동 동기화
```

**사후 조건**: 클라 검증과 서버 확정 검증이 동일 규칙 명세(ConstraintSpec)로 동치(PBT U5-P12). 위반 저장 시에도 차단 없이 지속 가시화(INV-SLOT3).

---

## FLOW-3 확정 / 해제 (plan 동결 ↔ 재확정)

**진입**: '일정 확정' → `M8.confirmItinerary` / '일정 수정' → `M8.unlockForEdit`
**관련**: US-E5-12 · BR-U5-31~33 · D14, D20, D13

```text
S1 확정 요청 — 서버 확정 검증 C2.validate 통과 게이트 [BR-U5-31]
   └─ E1 위반 잔존 → 'AI 자동 보정/그대로 저장' 선택 완료 후 진행(PRD 06-7 예외)
S2 TX 확정 — plan 스냅샷 동결(불변, INV-PLAN1) [BR-U5-31, D14]
   + M7.snapshotUserConfirmed(확정 POI 영구 스냅샷 D13) + status DRAFT/EDITING→CONFIRMED [INV-ITIN2]
   + outbox ItineraryConfirmed → M6 여행 CONFIRMED 전이·M14 리마인드 스케줄(CP5)
S3 확정 화면 — 시간표/지도 토글·읽기 전용·D-day/출발일·확정 의미 한 줄 설명 [BR-U5-32]
S4 (사후) 확정 해제 — M8.unlockForEdit [BR-U5-33]
   가드: 여행 시작 전(status≠INTRIP) — 여행 중은 해제 전이 불가(current만 편집) [INV-ITIN4]
   CONFIRMED→EDITING, '편집 중' 표시, 기존 공개본 유지·재확정 전 신규 공개 불가(D20)
   └─ 저장 후 재확정(FLOW-3 S1 재진입) → 새 PlanSnapshot 생성(기존 plan 불변)
S5 여행 ACTIVE 전이(M18) → CONFIRMED→INTRIP, plan/current 분리 작동(INV-CUR2) [D14]
```

**사후 조건**: 확정 후 임의 편집·재계획·재확정이 기존 plan을 바이트 동일하게 유지(INV-PLAN1, PBT U5-P7). 상태 전이는 §1.3 전이표 밖 불가(INV-ITIN4, PBT U5-P9).

---

## FLOW-4 재생성 보존 (warm-start)

**진입**: '다시 생성' → `M8.regenerate(itineraryId, scope)`
**관련**: US-E5-10, US-E4-09 · BR-U5-29·30 · G46/G136

```text
S1 고정 블록 수집 — warm-start 보존 대상 식별 [BR-U5-30]
   LOCK 슬롯(locked=true) ∪ 체류 고정(stayFixedFlag) ∪ 수동 추가(sourceType=USER_ADDED)
   ∪ 시각 고정 필수 방문지(MUST_VISIT_FIXED) ∪ 거점 블록(BASE) [INV-SLOT4]
S2 재배치 — C2.solve(fixedBlocks 보존, 나머지만) [BR-U5-30]
   고정 블록 전후 동일(INV-FIX1·INV-SOLVE5), 나머지 슬롯만 재최적화
S3 활성 일정 유지 — 여행당 1개(INV-ITIN1), version++, 이전 상태 changelog 추적(G136)
S4 멱등 — 동일 입력 반복 재생성에도 고정분 불변(멱등) [BR-U5-30, PBT U5-P2]
```

**사후 조건**: 재생성 전후 고정 블록·LOCK·수동 추가·시각 고정분 POI·시각·체류 동일(INV-SLOT4). 나머지만 재배치되며 하드 제약 4종 통과(C2 재사용, CP3 시나리오 2).

---

## FLOW-5 숙소 권역 추천 (숙소 나중 등록 온램프)

**진입**: 숙소 미정 상태에서 일정 구성 완료 → `M8.recommendStayZone(tripId)`
**관련**: US-E5-11 · BR-U5-43·44 · D25

```text
S1 동선 부족 게이트 [INV-ZONE1]
   └─ E1 방문지 2개 미만 등 동선 부족 → 권역 추천 미제공, 일반 탐색 유도(S3.146)
S2 무게중심 산출 — 방문지 분포 centroid + 방문지 간 평균 이동 거리 [BR-U5-44]
S3 권역 후보 산출 — '이 동선에 묵으면 이동 최단' 권역, 평균 이동 거리 순 [BR-U5-44]
   before/after '추정 이동 거리'(소요시간 없음 D25), 전역 최적 비보장 표기(INV-ZONE2)
S4 추천 숙소 등록 → M4.registerStay(추천 숙소) → 기준점 삼아 FLOW-1 재정렬(warm-start) [INV-ZONE3]
   └─ B1 미등록일(당일치기·이동일) → 숙소 없이 동선만 유지(appliedBaseRef=NULL) [BR-U5-43]
```

**사후 조건**: before/after는 '추정 이동 거리'만(소요시간 0, D25). 추천 숙소 등록 시 US-E5-01 생성·재정렬 정상 수행(INV-ZONE3).

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장 구성, D37 계층 분리). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check. 시드 로깅·수축(shrinking) 필수(PBT-08). **C2·C1 하드 제약 검증은 PR CI에서 실코드 테스트(모킹 금지, D37)** — 외부 LLM·거리 API만 어댑터 fake. 아래 속성은 U5 Code Generation의 DoD 항목(속성 테스트 존재·통과)이며, **U5 DoD가 명시한 6속성**((a)영업시간 내 배치·(b)이동 부등식·(c)고정 블록 불변·(d)폴백 결정성·(e)상태 머신 전이·(f)plan/current 직렬화 왕복)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**C2 Solver Engine (서버 — Kotest · oracle 테스트 대상)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U5-P1 | 하드 제약 불변식 (하드 제약 본체 · **oracle**) | **솔버 하드 제약**: 임의 후보 풀·시간창·고정 블록·이동 파라미터 입력에 대해 solve 출력의 모든 배치가 (a) 영업시간 내(HC1), (b) 인접 슬롯 이동 부등식 `직전 endAt+이동 ≤ 다음 startAt`(HC2), (c) 고정 블록 시각 불변(HC3), (d) 시간창 귀속·자정 초과=시작일(HC4)을 만족하고, **위반 배치는 solution에 부재**(INV-SOLVE1). **작은 인스턴스는 무차별 대입(oracle)으로 대조** — 솔버가 유효 배치를 부당 배제하지 않고 위반 배치를 통과시키지 않음을 이중 확인 | 후보 풀 생성기(좌표·영업시간·휴무 분포) + 시간창·고정 블록·이동 파라미터 조합 + 소규모 인스턴스(oracle 전수 대조용) |
| U5-P2 | warm-start 보존·멱등 | **고정 블록 보존**: LOCK·체류 고정·수동 추가·시각 고정 필수 방문지·거점 블록을 포함한 임의 일정에 regenerate/warm-start solve를 적용하면 (a) 고정분 POI·시각·체류가 전후 동일(INV-SLOT4·INV-SOLVE5), (b) 나머지만 재배치, (c) 반복 재생성에 고정분 불변(멱등) | 일정 생성기(고정/비고정 슬롯 혼합·LOCK 주입) + 재생성 반복 횟수 |
| U5-P3 | 결정론 폴백 결정성 | **폴백 모드 결정성**: deterministicFlag=true(또는 LLM 점수 부재)일 때 동일 입력이 동일 출력(무작위성 제거·시드 고정, INV-SOLVE2). 유효 해 항상 산출(빈 결과 0, 제약 만족) | 문제 인스턴스 생성기 + 실행 순서·병렬성 셔플(결정성 확인) |
| U5-P4 | 이동 추정 결정성·표시 게이트 | **이동 추정**: estimateTravel이 (a) 동일 (from,to,mode,파라미터)에 동일 출력(결정적·clock 주입, INV-TRAVEL2), (b) 라우팅 실패 시 직선거리 폴백+estimatedFlag=true(INV-TRAVEL3), (c) **어떤 표시용 DTO에도 internalDuration 미포함**(거리만 전파, INV-TRAVEL1) — 표시 DTO 스키마에 소요시간 필드 부재 정적 보장 | 좌표 쌍·이동 수단 생성기 + 라우팅 성공/실패 주입 + DTO 매핑 정적 검사 |
| U5-P6 | 예산 소프트 단조성 | **예산 가중치 단조성**: 예산대가 낮아질수록 저비용 카테고리 POI의 목적함수 보상이 단조 증가(비감소)하고, **예산 초과 배치가 하드 차단되지 않음**(INV-SOLVE3 — 예산 비하드화) | 예산대 시퀀스(단조 감소) + POI 비용 카테고리 분포 |

**C1 LLM Gateway (서버 — Kotest · closed-set 실코드 검증)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U5-P5 | closed-set 그라운딩 (하드 제약) | **POI 그라운딩**: 임의의 (후보 풀, LLM 응답) 쌍에 대해 (a) 출력 슬롯의 canonicalPoiId ⊆ 후보 풀(환각 0, INV-SLOT1·INV-SCORE1), (b) 후보 밖 ID 반환은 C1 출구에서 드롭·계측(통과 0), (c) 점수는 후보 ID에만 부여 — LLM 응답을 적대적으로 오염(후보 밖 ID·중복·형식 위반)시켜도 최종 출력 POI ⊄ 후보 풀 = 0 | 후보 풀 + 적대적 LLM 응답 생성기(후보 밖 ID·인젝션 페이로드·스키마 위반) |
| U5-P12 | 검증 규칙 동치 (D28) | **클라↔서버 규칙 동치**: 임의 편집 결과에 대해 클라이언트 경량 검증기(ConstraintSpec 소비)와 서버 C2.validate가 **동일 위반 집합**을 산출한다(동일 버전 명세). 버전 불일치 시 클라 검사는 보수적 비활성화(통과→거부 UX 0) | 편집 결과 생성기(경계 위반·정상 혼합) + 명세 버전 일치/불일치 주입 |

**M8 Itinerary Generation (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U5-P7 | plan 불변성 (하드 제약 계열) | **plan 스냅샷 불변**: 확정된 일정에 임의의 편집·재계획·재확정 연산 시퀀스를 적용해도 기존 PlanSnapshot.frozenDays가 **바이트 동일**하게 유지된다(확정 후 plan 변경 0, INV-PLAN1). 재확정은 새 스냅샷 생성이며 기존 것을 변형하지 않음 | 확정 일정 + 편집/재계획/재확정 연산 시퀀스 생성기 |
| U5-P8 | current↔plan 분리 | **plan/current 분리**: status=INTRIP에서 current의 임의 변경(편집·Plan-B)이 plan에 역류하지 않는다(INV-PLAN2·INV-CUR2) — plan↔current 참조 비공유(물리적 독립), current 변경 후 plan 동일 | INTRIP 일정 + current 변경 연산 생성기 |
| U5-P9 | 상태 전이 가드 | **일정 상태 머신**: 임의 (상태·트리거) 시퀀스에 대해 (a) §1.3 전이표의 조합만 성공, (b) 표 밖 전이(INTRIP→CONFIRMED 역행·DRAFT→INTRIP 직행·여행 중 unlockForEdit) 전부 거부(INV-ITIN4), (c) CONFIRMED→EDITING은 여행 시작 전만, (d) 활성 일정 ≤1 불변(INV-ITIN1) | 상태·트리거 시퀀스 생성기(유효/무효 전이 혼합) + 여행 상태(시작 전/중) 주입 |
| U5-P10 | 직렬화 왕복 | **plan/current 직렬화 왕복**: 임의 일정(plan·current·day·slot·고정 블록·violations)을 직렬화→역직렬화한 결과가 원본과 동등(무손실 왕복, INV-CUR3) | 일정 구조 생성기(고정 블록·위반·다일자·다거점 포함) |
| U5-P11 | 반경 게이트 (같이 고르기) | **같이 고르기 반경**: 임의 슬롯 진행에서 (a) 기준점=직전 확정 슬롯(첫 슬롯=등록 숙소, G48), (b) 반경 내 후보만 활성·반경 밖 disabled, (c) 반경 확대 시 추가 후보에 예상 이동 거리 표기, (d) 반경 내 0건 → 확대/컨셉 변경 제안(빈 결과 0) | 슬롯 시퀀스 생성기(좌표 분포·반경 변화) + 확정 슬롯 위치 |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| M8 생성 오케스트레이션(generateItinerary 시퀀스) | No PBT properties identified | 외부 모듈 조립·TX 경계·진행 스트림 — 보편 양화 도메인 로직은 하위 C1(U5-P5)·C2(U5-P1~3)·상태 머신(U5-P9)으로 분해. 폴백 계단·점진 노출·세션 상태 전이는 예시 기반 통합 테스트+어댑터 fake(RESILIENCY-10·S1 실패 분기) |
| M8 추천·배치 이유(getExplanations) | No PBT properties identified | 표시 전용 문구 조립(검증값 우선은 U5-P1이 시각 정본을 보장) — 설명-시각 불일치 시 검증값 표시는 예시 기반 테스트 |
| M8 숙소 권역 추천(recommendStayZone) | No PBT properties identified (U5 시점) | 무게중심 기하 계산·추정 표기 — 도메인 불변식보다 산출 정확성. 동선 부족 게이트(INV-ZONE1)·소요시간 미표시(D25)는 예시 기반 테스트 |
| C1 티어 라우팅·rate-limit | No PBT properties identified | 라우팅·상한 정책(수치는 NFR) — closed-set(U5-P5)·컨텍스트 권한(SECURITY-08 계약 테스트)과 달리 정책 테이블. 예시 기반+계약 테스트 |

### 커버리지 대조 (U5 DoD → 속성)

| U5 DoD 명시 속성 | 대응 |
|---|---|
| (a) 영업시간 내 배치 | U5-P1(HC1) |
| (b) 인접 슬롯 이동시간 부등식 | U5-P1(HC2) |
| (c) 고정 블록 불변(warm-start 전후 동일) | U5-P1(HC3) · U5-P2 |
| (d) 폴백 모드 결정성(동일 입력→동일 출력) | U5-P3 |
| (e) 일정 상태 머신(초안→편집중→확정→해제) 전이 불변식 | U5-P9 |
| (f) plan/current 직렬화 왕복 | U5-P10 |
| (설계 확장) 이동 추정 결정성·소요시간 표시 게이트 | U5-P4 |
| (설계 확장) POI 그라운딩 closed-set(환각 0) | U5-P5 |
| (설계 확장) 예산 소프트 가중치 단조성 | U5-P6 |
| (설계 확장) plan 스냅샷 불변성 / current↔plan 분리 | U5-P7 / U5-P8 |
| (설계 확장) 같이 고르기 반경 게이트 / 클라↔서버 규칙 동치 | U5-P11 / U5-P12 |

**속성 합계: 12개** (C2: 5 · C1: 2 · M8: 5)

### 하드 제약(D37 — 본 유닛이 4계열 본체) 대조

| 하드 제약(U5 소관) | 방어 속성·불변식 |
|---|---|
| **숙소 기준점**(모든 일자 동선이 해당 구간 거점 기준) | U5-P1(고정 블록·거점 base) · INV-DAY1·3 · BR-U5-01·04 |
| **충돌 무배치**(시간 중복·영업시간 밖·이동 부등식 위반 배치 0) | U5-P1(HC1·HC2·HC4 · oracle 대조) · INV-SOLVE1 |
| **POI 그라운딩**(출력 POI ⊆ 후보 풀 — closed-set) | U5-P5 · INV-SLOT1·INV-SCORE1 |
| **계정 무결성**(타 계정 데이터 미혼입 D31) | U5(C1 resolveContext 권한 검증 · SECURITY-08 계약 테스트) · INV-SCORE4 |

> **C2 oracle 테스트 명시**: C2 Solver Engine은 소규모 문제 인스턴스에 대해 **무차별 대입(brute-force 전수 열거) 대조**를 oracle로 사용한다 — 솔버 해가 하드 제약을 만족하는지(부당 통과 0)와 유효 배치를 부당 배제하지 않는지(부당 배제 0)를 이중 확인한다(U5-P1). 대규모 인스턴스는 불변식 속성(HC1~HC4 만족)만 검증한다(oracle 비용 한계).
