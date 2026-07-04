# U7 기록·회고 — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 5종 + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U7-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) M12·M13, 서비스 정본 [services.md](../../../inception/application-design/services.md) S4(여행 종료→회고)·S8(오프라인 기록 동기화)
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 전송·저장·오브젝트 스토리지·CDN·LLM 티어·remote config 구현체는 NFR/Infra 단계 소유.
> **재사용 원칙(FD-U7-06)**: 회고·요약·스타일 서술 LLM 호출은 U5 C1(상위 티어)을 그대로 재사용하고, 실패 폴백 소유자는 M13(기본 카드)이다 — U7은 actual·changelog 영속·오프라인 동기화·회고 오케스트레이션만 신규.

---

## FLOW-1 방문 기록 → 사진·메모 (CP4 소비 · 오프라인 큐)

**진입**: U6 `VisitChecked`(start/complete/skip) 이벤트 수신 ∨ 사용자 사진·메모 첨부 → `M12.checkVisit`(구독) / `M12.attachMedia` / `M12.addImpromptuVisit`
**관련**: US-E8-01·02·05 · BR-U7-01~04·08~11 · D23, D14, G77, G75/G145, CP4

```text
S1 actual 방문 기록 생산 [BR-U7-01] — CP4 VisitChecked 소비
   VisitChecked(start) → VisitRecord{arrivedAt(사용자 탭·수정 가능)} · VisitChecked(complete) → departedAt·actualStay 산출
   actualStay = departedAt − arrivedAt(결정적·INV-REC2) → 계획 체류와 함께 보관(plan/actual 구분 D14)
   departedAt이 다음 장소 체크 추정이면 departedEstimatedFlag=true(허위 정확성 0)
   → 체류 초과분 M9 신호 공급(체류 초과 트리거 입력) [BR-U7-02]
   ├─ B1 즉석 방문(계획 밖) → M12.addImpromptuVisit(POI 검색 ∨ 자유 텍스트 — 자유는 좌표/카테고리 없음 '기타' INV-IMP2) [BR-U7-03]
   └─ B2 스킵/취소 → visitStatus=SKIPPED(diffKind=SKIPPED 대조 노출)
S2 귀속 [BR-U7-04]
   숙소·날짜 기준 귀속(다중 숙소=날짜별 기준 숙소·숙소 없는 날=날짜만 M4.getBaseTimeline)
S3 사진·메모 첨부 — M12.attachMedia [BR-U7-08~11]
   사진: 클라 압축(5MB/2048px)·EXIF 제거(exifStripped)·서버 썸네일·원본 기기 보관·장소당 ≤20장
   uploadState LOCAL→QUEUED→(성공)DONE / (실패)RETRYING(≤3회 지수 백오프)→FAILED('다시 시도' 버튼)
   MediaResult{accepted, pending, rejected} — 부분 결과 명시
   ├─ E1 사진 업로드 실패 → 로컬 보관·'업로드 대기' · 메모·방문 체크는 독립 저장(부분 실패 격리 INV-MEDIA1) [BR-U7-09]
   └─ B3 오프라인 → shared/storage 로컬 큐 적재('동기화 대기') → FLOW-3 [BR-U7-12]
S4 3계열 대조 반영 [BR-U7-28]
   plan(불변·U5)/actual(M12)/changelog(M12) 구분 저장 → getTripTimeline 대조 뷰(라벨·차이 하이라이트)
```

**사후 조건**: actual은 CP4 이벤트로만 생산(자동 기록 0·INV-REC1), 체류 결정적(INV-REC2), 사진 실패가 메모·체크를 막지 않음(INV-MEDIA1), plan/actual 분리(INV-REC3).

---

## FLOW-2 당일 회고 · 전체 요약 생성 (여행 종료 S4 · C1 · 폴백)

**진입**: `DayClosed`(일자 경계) ∨ `TripEnded`(자동 익일 00:00 ∨ 수동) → `M13.generateDailyReflection` / `M13.generateTripSummary` (정본 서비스 S4)
**관련**: US-E8-06·07·08 · BR-U7-15~21 · D19/Δ4, D11, G72, G78, C11, ADR-0011 · services S4

```text
S1 트리거 수신 [BR-U7-15·20] — CP4 소비
   DayClosed(여행·일자·방문 요약) → 당일 회고 · TripEnded(종료 방식·시각) → 전체 요약·스타일 분석
   멱등: M13 (tripId, date|kind) 핸들러 — 기존 존재 시 skip(중복 수신 안전 INV-SUM1)
   └─ E1 오프라인 구간 → 당일 회고 생성 보류 → 복구 후 생성 ∨ 기본 카드(INV-REFL4) [BR-U7-16]
S2 입력 수집
   방문·사진·메모·changelog 대조(D14) · 이동 거리(GPS 동의 구간 실측 + 미동의 추정 혼합 G72·추정 표기) [BR-U7-22]
   └─ E2 기록 0건 → NoActivity('오늘 기록된 활동이 없습니다'·수동 기록 유도) [BR-U7-17]
S3 C1 생성 (상위 티어 D11) — U5 C1 재사용
   C1.call(reflection, serverContext) — 회고 문구·요약 서술
   ├─ E3 부분 데이터 → 가용분만 생성 + 누락 명시('사진 없음'·'위치 기록 없음' basis.missingItems INV-REFL3) [BR-U7-17]
   └─ E4 C1 실패/타임아웃 → FallbackCard{방문 N곳·이동 Nkm·사진 N장}(통계만·직접 작성 가능 INV-REFL2) [BR-U7-16]
S4 전체 요약 지도 히어로 [BR-U7-20·21]
   mapHeroSpec(방문 순서 핀·날짜별 동선·거점 마커·사진 미리보기) · routeMode 구간별(동의=GPS_ACTUAL·미동의=MANUAL_LINKED)
   ├─ E5 위치 데이터 전무 → 지도 대신 방문 목록(순서·날짜) 대체(MAP_UNAVAILABLE) [BR-U7-21]
   └─ E6 요약 생성 실패 → 날짜별 기본 카드 모음 폴백(FALLBACK) [BR-U7-21]
S5 저장·알림 [BR-U7-15]
   TX: 회고/요약 저장 + outbox ReflectionReady(유형: DAILY|TRIP_SUMMARY|STYLE_ANALYSIS) → M14(발송은 U8 CP5)
S6 수정·재생성 [BR-U7-18·19]
   editReflection → editedContent 별도 저장(최종 표시본 INV-REFL1)
   regenerateReflection → 수정본 존재 시 OverwriteWarning → overwriteConfirmed 후에만 교체(G78)
   종료 후 편집분 반영은 수동 regenerateTripSummary/regenerateReflection만(자동 갱신 없음 C11)
```

**멱등성**: `DayClosed`·`TripEnded` 핸들러는 (tripId, date|kind) 기준 중복 수신 안전. **사후 조건**: 회고 트리거는 CP4로만 개시, 실패는 기본 카드 폴백(빈 화면 0), 수정본 보존(INV-REFL1).

---

## FLOW-3 오프라인 동기화 · 충돌 해소 (레코드 단위 독립 커밋 S8)

**진입**: 재연결 감지 → `M12.syncOfflineRecords(batch)` → (충돌) `M12.resolveSyncConflict`
**관련**: US-E8-12 · BR-U7-12~14 · G74, D24/Δ6 · services S8

```text
S1 오프라인 입력 (클라 storage) [BR-U7-12]
   방문 체크·사진 메타·메모·수동 체크인 → 로컬 큐(recordId UUID + recordedAt + baseVersion)·'동기화 대기'
   오프라인 보장은 '입력' 한정 — 조회는 온라인 전제(D24·오류·재시도 안내만 INV-SYNC4)
S2 재연결 → 배치 업로드 — M12.syncOfflineRecords [BR-U7-13]
   레코드 단위 독립 TX(배치 원자성 의도적 부재·부분 성공 허용 INV-SYNC1)
   recordId 이미 적용 → skip(멱등)
S3 병합 판정 [BR-U7-13]
   ├─ 충돌 없음 → 적용 + VisitChecked(syncedFromOffline=true·recordedAt 원 시각 보존)
   │     └─ E1 recordedAt 과거 임계 초과 → M9 트리거 평가 제외(오발화 방지 INV-SYNC5) [BR-U7-14]
   ├─ 사진·메모 추가 → 합집합 자동 병합(충돌 아님 INV-SYNC2)
   └─ 상태·시각 충돌(서버 측 변경 존재) → conflict 목록 반환(레코드별 local/server 2버전)
S4 충돌 해소 — M12.resolveSyncConflict [BR-U7-13]
   사용자 항목별 선택(KeepLocal/KeepServer) → 새 버전으로 적용(무손실·임의 폐기 금지)
   응답 per-record 결과(applied/conflict/failed)
S5 하류 전파
   VisitChecked 구독 → actual 타임라인 반영 · 오프라인 구간 당일 회고는 복구 후 생성(FLOW-2 E1)
```

**사후 조건**: recordId 멱등(재적용 0·INV-SYNC1), 합집합 무손실(INV-SYNC2), 해소 결정성·독립 레코드 순서 무관 수렴(INV-SYNC3), 조회 오프라인 미보장(INV-SYNC4).

---

## FLOW-4 스타일 분석 (10곳 게이트 · 임시 미리보기 · 개인화)

**진입**: 스타일 분석 조회 ∨ TripEnded 후 배치 → `M13.analyzeTravelStyle`
**관련**: US-E8-09·10 · BR-U7-23·24 · G76, PRD 09-9·12-10

```text
S1 게이트 판정 [BR-U7-23]
   basedOnVisitCount = M12.getVisitStats 누적 방문 장소 수(즉석 자유 입력 포함·카테고리 '기타')
   ├─ < 10곳 → PendingGate{current:N, required:10, previewFromPreferences}
   │     → PREVIEW(온보딩 취향 기반 임시 미리보기·정식 아님 명시)+진행 게이지('N/10곳') INV-STYLE1
   └─ ≥ 10곳 → OFFICIAL 생성 진행
S2 분석 생성 [BR-U7-23]
   metrics{카테고리 분포·평균 체류·이동 반경·활동 밀도} · 취향 7종 축 매핑 자체 택소노미(임시와 스키마 공유)
   구체 수치·카테고리 제시 + 근거 방문 데이터 동반
S3 개인화 신호 [BR-U7-24]
   분석 결과 → M2 경유 다음 여행 추천 입력(직접 설정 우선 > 자동 분석)
   └─ B1 개인화 미동의 → 과거 기록 추천 입력 제외·기본 추천만
```

**사후 조건**: 게이트는 방문 수에 단조(비축소·INV-STYLE1), 방문 수 기준 우선(INV-STYLE2), 미달도 빈 칸 없음(임시 미리보기).

---

## FLOW-5 공유 카드 (진입 · 자동 구성 · 3포맷 · 사진 0장 폴백)

**진입**: 전체 요약(US-E8-08) ∨ 당일 회고(US-E8-06) 화면 '공유' → `M13.buildShareCard`
**관련**: US-E8-13 · BR-U7-25 · D25, C3

```text
S1 진입 게이트 [BR-U7-25]
   여행 종료·요약 생성 후에만 진입 — 미종료/요약 미생성 → PreconditionFailed(진입점 비활성 INV-SHARE1)
S2 자동 구성 [BR-U7-25]
   대표 사진·제목·기간·핵심 통계(방문 수·총 이동 거리·사진 수 — 소요시간 없음 D25)·동선·워터마크
   ├─ E1 사진 0장 → 지도 경로·통계 기반 '사진 없는 카드'(NO_PHOTO 레이아웃 INV-SHARE2) [BR-U7-25]
S3 편집·내보내기 [BR-U7-25]
   포맷 선택(9:16/1:1/4:5) · 캡션·해시태그 편집(금칙어 검증 C3)
   → 기기 이미지 저장 ∨ OS 공유 시트(외부 SNS·메신저 — 정적 이미지 채널·커뮤니티와 별개 INV-SHARE3)
```

**사후 조건**: 미종료 여행 공유 진입 0(INV-SHARE1), 사진 0장도 빈 카드 없음(INV-SHARE2), 통계에 소요시간 없음(D25).

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장, D37 계층 분리). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check. 시드 로깅·수축(shrinking) 필수(PBT-08). **데이터 정합성 로직(diff 재생·오프라인 병합·이동 거리 합산)이 U7 PBT 효용 최대 영역**이며, 외부(LLM·오브젝트 스토리지)만 어댑터 fake(D37). 아래 속성은 U7 Code Generation의 DoD 항목이며, **U7 DoD 명시 속성**((a)changelog diff 누적 재구성=스냅샷 동등성·(b)오프라인 병합 수렴·합집합·무손실·(c)plan/actual 대조 함수·(d)직렬화 왕복·(e)이동 거리 혼합 합산)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**M12 Travel Archive (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U7-P1 | changelog 재구성 동등성 (1급·하드 계열) | **diff 누적 재구성 = current 동등성**: 확정 시점 기준(plan)에 임의 changelog diff 시퀀스를 발생 시각 오름차순으로 누적 적용한 **재구성 결과가 current 스냅샷과 일치**(INV-CLOG1). (a) 라운드트립(plan→diff 누적→current 재구성=current), (b) append-only 순서 불변(동일 시퀀스 동일 재구성), (c) **CP4 오라클**: U6 재계획 실생산 diff 시퀀스로 통합 재실행(재구성=U6 current) | changelog diff 시퀀스 생성기(추가/삭제/시간 이동/순서 변경 혼합) + plan 기준 스냅샷 + U6 CP4 생산분 |
| U7-P2 | actual 기록 불변식 | **actual 결정성·분리**: (a) actualStay = departedAt − arrivedAt이 동일 (arrivedAt, departedAt)에 동일 산출(결정적·clock 주입, INV-REC2), (b) departedAt 다음 장소 체크 추정이면 departedEstimatedFlag=true 전파(허위 정확성 0), (c) arrivedAt 부재 시 미산출(NULL), (d) **plan/actual 분리** — actual 기록 시퀀스가 plan 스냅샷을 바이트 불변 유지(INV-REC3·U5 INV-PLAN1 소비) | 방문 시각 쌍 생성기(실측/추정 혼합) + plan 스냅샷 + 다음 장소 체크 시각 주입 |
| U7-P3 | 오프라인 병합 수렴 (하드 계열) | **오프라인 충돌 해소 결정성·교환법칙·무손실**: 임의 순서 동기화에 대해 (a) recordId 멱등(재적용 0·INV-SYNC1), (b) 사진·메모 추가 **합집합 병합**(어떤 순서에도 합집합 동일·무손실 INV-SYNC2), (c) 독립 레코드 동기화 순서를 셔플해도 최종 수렴 상태 동일(**병합 교환법칙** INV-SYNC3), (d) 상태·시각 충돌은 사용자 선택으로만 해소·양쪽 보존(임의 폐기 0) | 오프라인 배치 생성기(방문 체크·사진·메모·수동 체크인 혼합) + 순서 셔플 + 서버 측 변경 주입 |
| U7-P4 | 사진 큐 재시도 멱등 | **사진 오프라인 큐 재시도 멱등**: 임의 (네트워크 실패/성공) 시퀀스에 대해 (a) uploadState 머신 LOCAL→QUEUED→RETRYING(≤3)→FAILED/DONE 전이표만 성공, (b) 재시도 멱등(중복 업로드·중복 Photo 레코드 0), (c) 3회 초과 시 FAILED(수동 재시도만)·성공 시 DONE 종결, (d) **부분 실패 격리** — 사진 실패가 메모·방문 체크 저장을 막지 않음(INV-MEDIA1) | 네트워크 결과 시퀀스 생성기(실패/성공 분포) + 사진·메모·체크 동반 입력 |
| U7-P5 | 이동 거리 혼합 합산 결정성 | **이동거리 혼합 산출 결정성**: 임의 (GPS 동의 구간·미동의 구간) 분할에 대해 (a) 총 이동 거리=실측 폴리라인 합산+장소 간 거리 추정의 혼합이 **구간 분할·순서 불변에 결정적**(INV-SUM2), (b) 구간 재분할(동일 경로 다른 split)에도 합산 동일(구간 분할 불변), (c) 걸음 수=거리 환산 추정 결정적(G59) | 경로 구간 생성기(GPS/미동의 혼합·split 변형) + 좌표열 주입 |
| U7-P(직렬화) | 오프라인 큐 페이로드 직렬화 왕복 | **직렬화 왕복**: 임의 오프라인 큐 레코드(방문 체크·사진 메타·메모·수동 체크인 payload)가 직렬화→역직렬화 라운드트립에서 동등(무손실) — recordId·recordedAt·baseVersion 보존 | 오프라인 레코드 생성기(유형·필드 분포) |

**M13 AI Reflection (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U7-P6 | 스타일 게이트 단조 | **스타일 분석 10곳 게이트 단조성**: 임의 누적 방문 수 시퀀스에 대해 (a) OFFICIAL은 basedOnVisitCount ≥ 10에서만 생성(경계 정확), (b) 방문 수 증가가 게이트 통과를 뒤집지 않음(**단조·비축소** INV-STYLE1), (c) 미달은 PREVIEW+진행 게이지(빈 칸 0), (d) 방문 수 기준 우선(완료 여행 수와 충돌 시 방문 수 INV-STYLE2) | 누적 방문 수 시퀀스 생성기(10 경계 분포) + 완료 여행 수 주입 |
| U7-P7 | 회고 수정본 보존 | **회고 수정본 보존**: 임의 (생성·수정·재생성) 시퀀스에 대해 (a) editedContent 존재 시 displayContent=editedContent(수정본 우선 INV-REFL1), (b) 재생성이 수정본을 덮어쓰려면 overwriteConfirmed 필수(미확인 시 수정본 보존·G78), (c) 폴백 카드 위 직접 작성 보존(FALLBACK_CARD→EDITED), (d) 종료 후 자동 갱신 0(수동 재생성만 C11) | 회고 상태 전이 시퀀스 생성기(생성/편집/재생성/확인 혼합) |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| M13 회고·요약 LLM 생성(generateDailyReflection·generateTripSummary 본문) | No PBT properties identified | LLM 서술 생성은 비결정·품질 편차 — 폴백 계단(기본 카드·지도 대체·NoActivity)은 예시 기반 통합 테스트, C1은 어댑터 fake(D37). 검증 속성은 폴백 도달(침묵 실패 금지)·멱등(U7-P7 인접) |
| M13 공유 카드 구성(buildShareCard) | No PBT properties identified | 이미지 레이아웃 조립·진입 게이트 — 사진 0장 폴백·미종료 차단은 예시 기반. 캡션 금칙어는 C3 계약 테스트 |
| M12 대조 뷰 조립(getTripTimeline·getTripRecords) | No PBT properties identified | 읽기 모델 집약(plan/actual/changelog 라벨 구분)·귀속(M4 기준) — diff 재구성 핵심은 U7-P1로 분해, 나머지는 예시 기반 |
| M12 캘린더·목록(getRecordCalendar·listArchivedTrips) | No PBT properties identified | 월 마킹·목록 집약(겹침 없음은 D21 상류 보장) — 0건 빈 상태는 예시 기반 |
| M12 GPS 파기(purgeLocationData) | No PBT properties identified | 파기 실행·법정 로그 분리 보관 — 삭제 연쇄(S6) 계약 테스트+테이블 주도(옵트인 상태별). 잔존=법정 집합 검증은 U8 삭제 연쇄 PBT와 연동 |

### 커버리지 대조 (U7 DoD → 속성)

| U7 DoD 명시 속성 | 대응 |
|---|---|
| (a) changelog diff 누적 재구성 = 스냅샷 동등성 | **U7-P1(CP4 오라클·라운드트립)** |
| (b) 오프라인 병합(임의 순서 수렴·사진/메모 합집합·충돌 무손실) | U7-P3 |
| (c) plan/actual 대조 함수 | U7-P2(d) |
| (d) 직렬화 왕복(오프라인 큐 페이로드) | U7-P(직렬화) |
| (e) 이동 거리 혼합 합산(구간 분할 불변) | U7-P5 |
| (설계 확장) actual 기록 불변식(체류 결정성·추정 전파) | U7-P2 |
| (설계 확장) 사진 오프라인 큐 재시도 멱등·부분 실패 격리 | U7-P4 |
| (설계 확장) 스타일 분석 10곳 게이트 단조성 | U7-P6 |
| (설계 확장) 회고 수정본 보존 | U7-P7 |

**속성 합계: 8개** (M12: 6[P1·P2·P3·P4·P5·직렬화] · M13: 2[P6·P7]) — 프롬프트 필수 7속성(changelog 재구성·actual 불변식·사진 큐 멱등·충돌 해소 결정성/교환법칙·이동거리 혼합·스타일 게이트 단조·회고 수정본 보존) 전부 포함 + 직렬화 왕복 확장.

### 하드 제약(D37) 대조 — U7은 계정 무결성·changelog 무결성 소관

| 하드 제약 | U7 방어 속성·불변식 |
|---|---|
| **계정 무결성**(기록·사진·회고 소유권 격리·회고 LLM 입력 서버 재조회 D31) | SECURITY-08(소유권)·U5 INV-SCORE4 재사용(§13 C1 재사용)·삭제 연쇄(S6) |
| **changelog 무결성**(전/후 값·행위자 필수·재구성 정합) | **U7-P1** · INV-CLOG1·2·4 · BR-U7-05·06 |
| **회고 대조 무결성**(plan 불변·actual 분리) | U7-P2(d) · INV-REC3 · U5 INV-PLAN1 소비 |
| **위치 데이터 파기**(옵트인 철회·탈퇴 즉시 파기 N2) | INV-GPS2 · BR-U7-27 · S6.2(법정 로그 분리) |

> **C1 재사용 명시(FD-U7-06)**: U7은 LLM 게이트웨이·closed-set·계정 무결성(D31 서버 재조회)을 재구현하지 않는다 — 회고·요약·스타일 서술은 **U5 C1을 호출**하고, 실패 폴백 소유자는 M13(기본 카드)이다. 하드 제약 "계정 무결성"의 LLM 입력 경계는 U5 INV-SCORE4(SECURITY-11)를 재사용한다.
