# U7 기록·회고 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M12 Travel Archive, M13 AI Reflection (+ C1 LLM Gateway 재사용 계약 수준)
> 정본 관계: [components.md](../../../inception/application-design/components.md) M12·M13 섹션 + [component-methods.md](../../../inception/application-design/component-methods.md) M12·M13 메서드를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U7 엔티티·불변식·상태 필드의 정본이다. 기술 중립 — 저장 기술·오브젝트 스토리지·CDN·LLM 티어·remote config 수치(사진 상한·재시도 횟수·게이트 임계·보존 기간)는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL 의미 명기), `유니크`=제약 범위 내 유일, `불변`=생성 후 변경 불가, `추가전용`=append-only, `파생`=저장 안 함(조회 시점 계산). 근거 ID는 requirements.md(D·Δ·N·G·C·ADR)·stories.md(US-E8)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U7-xx)을 참조한다.
> **U7은 CP4(U6→U7)의 소비자이자 CP5(U7→U8)의 공급자다** — 방문 체크·여행 종료·changelog diff·GPS 폴리라인(U6 [u6-execution/functional-design/domain-entities.md](../../u6-execution/functional-design/domain-entities.md) §11 CP4 공급 계약)을 소비해 actual·회고·기록을 생산하고(§10 소비 계약), 회고 완료(`ReflectionReady`)를 U8 알림에 공급한다(§11 공급 계약).
> **plan/current는 U5/M8 소유 입력**(CP3 파생·회고 대조용) — 본 유닛은 참조·소비만 하고 **actual·changelog는 M12(U7)가 소유하는 정본**이다(D14·ADR-0013, FD-U7-02).

## 0. 엔티티 지도

```text
[M12 Travel Archive — actual·사진·메모·changelog·GPS·오프라인 동기화 정본]
Trip 1 ──── * VisitRecord            (실제 방문 기록 actual — CP4 VisitChecked 소비·D23·D14)
VisitRecord ◁── ImpromptuVisit       (즉석 방문 특수화 — slotRef=NULL·좌표/카테고리 없음 '기타' G77)
VisitRecord 1 ──── 0..* MediaAttachment  (사진·메모 — 장소당 20장·클라 압축·오프라인 큐·부분 실패 격리 G75/G145)
Trip 1 ──── * ChangeLogEntry         (통합 diff 스키마 G132 — Plan-B·(후속)공동편집·어시스턴트 공용·append-only)
Trip 1 ──── 0..* GpsTrack            (날짜별 단순화 폴리라인 — 수집 U6·영속 M12·옵트인 D34/N2)
OfflineSyncRecord                    (오프라인 입력 동기화 봉투 — 레코드 단위 버전·충돌 G74·Δ6)

[M13 AI Reflection — 회고·요약·분석·공유 카드]
Trip 1 ──── 0..* Reflection          (당일 회고 — 초안/수정본 G78·부분 데이터·폴백)
Trip 1 ──── 0..1 TripSummary         (전체 여행 요약 — 지도 히어로·G72 혼합 이동거리)
Account 1 ──── 0..1 TravelStyleAnalysis  (스타일 분석 — 10곳 게이트 G76·정식/임시 미리보기)
(TripSummaryRef | DailyReflectionRef) ──── ShareCard  (SNS 공유 카드 — 3포맷·사진 유/무)

[CP4 입력 — U6 소유(본 유닛은 구독·소비, §10)]
VisitChecked(start/complete/skip) · DayClosed · TripEnded · changelog diff(G132) · GPS 폴리라인

[CP3 참조 입력 — U5/M8 소유(회고 대조용·재정의 금지)]
Itinerary(plan 불변 스냅샷 / current 가변) + Slot(canonicalPoiId·계획 체류·시각)

[CP5 출력 — U8 M14 소비(§11)]
ReflectionReady(당일/전체 요약/스타일 분석)
```

**plan/actual/changelog/current 4계열 소유 경계(D14·ADR-0013, FD-U7-02)**

| 계열 | 소유 모듈 | 성질 | U7 관여 |
|---|---|---|---|
| plan(불변 스냅샷) | M8(U5) | 확정 시 동결·바이트 불변 | **참조·소비만**(회고 대조 기준 — plan⊗actual) |
| current(가변 현재본) | M8·M10(U5·U6) | 재계획 시 갱신 | 참조·소비만(changelog 재구성 대조) |
| **actual(실제 방문 기록)** | **M12(U7)** | 방문 체크·사진·메모·체류 | **소유 정본** — CP4 VisitChecked 생산 |
| **changelog(변경 이력)** | **M12(U7)** | 항목 단위 diff·append-only | **소유 정본** — 보관·열람·재생(G132) |

---

## 1. VisitRecord — 실제 방문 기록 (M12, actual 영속 정본)

일정 슬롯 1건(또는 즉석 방문 1건)의 **실제 방문 기록(actual)**. M18(U6) VisitState가 발행하는 `VisitChecked` 이벤트를 소비해 생산·보관하며, 방문 시각·실제 체류·좌표를 계획(plan) 체류와 함께 저장한다. **상태 머신·전이 강제는 U6(M18) 소유**이고 본 엔티티는 **영속·수정·대조**만 소유한다(경계 FD-U7-01·U6 INV-VISIT5). 도착 확정은 항상 사용자 탭 결과(자동 기록 없음 D23).

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| visitId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| slotRef | 식별자(ItinerarySlot 참조) | 선택 | 대상 계획 슬롯. **NULL=즉석 방문**(§2 ImpromptuVisit) [G77] |
| poiSnapshotRef | 식별자(POI 스냅샷 참조) | 선택 | 방문 장소 정본 스냅샷(계획 슬롯·저장 POI 파생). NULL=자유 텍스트 즉석 방문 [D13, G77] |
| freeText | 문자열 | 선택 | 즉석 자유 입력 장소명(좌표·카테고리 없음 → 분석 '기타'). NULL=POI 참조 기록 [G77] |
| visitStatus | 열거 {DONE, SKIPPED} | 필수 | 방문 완료/방문 안 함(스킵·취소) — CP4 전이 유형 파생 [D23] |
| arrivedAt | 시각(기기 시각) | 선택 | 도착(방문 시작) 시각 — 사용자 탭·**직접 수정 가능**. NULL=미도착 [D23] |
| departedAt | 시각 | 선택 | 방문 종료 시각. NULL=미완료. 다음 장소 체크로 추정 시 estimatedFlag=true [D23] |
| departedEstimatedFlag | 불리언 | 필수 · 기본 false | departedAt이 다음 장소 체크로 추정된 값인지(허위 정확성 방지) [D23] |
| actualStay | Duration | 파생 | `departedAt − arrivedAt`(둘 다 존재 시) — 계획 체류와 함께 보관·대조 [D23, D14] |
| coord | 좌표 | 선택 | 방문 좌표. **NULL=위치 미동의·GPS 불가**(좌표 없는 기록 정상 생성) [D34, PRD 09-3] |
| baseAttribution | 구조체 {stayRef?, date} | 필수 | 귀속 기준(날짜별 기준 숙소 참조·숙소 없는 날은 날짜만) [PRD 09-5] |
| syncState | 열거 {LOCAL, PENDING, SYNCED, CONFLICT} | 필수 · 기본 `SYNCED` | 오프라인 동기화 상태(§9) — LOCAL/PENDING은 클라 큐 [G74, Δ6] |
| recordVersion | 정수 | 필수 · 기본 1 | 레코드 단위 버전(충돌 비교 키·멱등) [G74] |
| recordedAt | 시각 | 필수 · 불변 | 원 발생 시각(오프라인 입력 시 클라 생성·보존 — 늦은 동기화 정합) [S8] |

### 1.2 파생값 (저장 안 함)

| 파생값 | 계산 | 근거 |
|---|---|---|
| plannedStay | slotRef→계획 슬롯 체류(plan 파생) | actualStay 대조 기준(plan⊗actual) [D14] |
| diffKind | plan 대비 {MATCHED, SKIPPED(미방문), ADDED(추가·즉석), REORDERED(순서 변경)} | 계획-실제 비교 뷰 라벨(3계열 대조 US-E8-04) [D14, PRD 09-4] |
| categoryForStyle | poiSnapshot 카테고리(즉석 자유 입력은 '기타') | 스타일 분석 카테고리 분포 입력 [G76, G77] |

### 1.3 불변식 (Invariants)

- **INV-REC1 (actual 영속 경계 — CP4)**: VisitRecord는 M18(U6) `VisitChecked` 이벤트로만 생산되며(사용자 탭 결과·자동 기록 없음 D23), 방문 상태 전이 강제는 U6 소유다 — U7은 영속·수정·대조만. 종료 후 편집(`updateVisitRecord`)은 허용하되 **상태 재전이가 아니라 기록 편집**이다(C11) [FD-U7-01, CP4, U6 INV-VISIT5, C11].
- **INV-REC2 (체류 결정성·추정 전파)**: actualStay = departedAt − arrivedAt은 동일 (arrivedAt, departedAt) 입력에 결정적 산출 — departedAt이 다음 장소 체크 추정이면 departedEstimatedFlag=true로 전파(허위 정확성 금지). arrivedAt 부재 시 actualStay 미산출(NULL) [D23, U6 INV-VISIT3, PBT U7-P2].
- **INV-REC3 (plan/actual 분리 보관 — D14)**: actual(실제)과 plan(계획 스냅샷·U5 소유)은 **별개 계열로 저장**되고 회고·비교는 plan⊗actual 대조로만 산출한다 — actual 기록이 plan 스냅샷을 변형하지 않는다(plan 불변, U5 INV-PLAN1 소비) [D14, ADR-0013, FD-U7-02].
- **INV-REC4 (수정 가능·감사)**: arrivedAt·visitStatus 등 사용자 수정은 항상 허용(자동 기록값 포함)되며, 일정을 변경하는 편집은 없으므로 changelog 대상 아님 — 방문 기록 수정은 actual 계열 내 갱신이다(일정 변경 이력과 구분) [D23, PRD 09-1].
- **INV-REC5 (좌표 선택성)**: coord는 GPS 옵트인·권한 충족 시에만 채워지며, 부재 시에도 방문 기록은 정상 생성된다 — 위치 미동의가 기록 생성을 막지 않는다(수동 체크인 기본 수단) [D34, PRD 09-3].

---

## 2. ImpromptuVisit — 즉석 방문 (M12, VisitRecord 특수화 G77)

계획에 없던 장소의 **즉석 방문 기록**. VisitRecord의 특수화로 `slotRef=NULL`이며 POI 검색(좌표·카테고리 있는 스냅샷) 또는 자유 텍스트(좌표·카테고리 없음) 두 경로를 허용한다. 별도 테이블이 아니라 VisitRecord의 변형(slotRef·poiSnapshotRef·freeText 조합)으로 표현한다.

| 변형 | slotRef | poiSnapshotRef | freeText | 스타일 분석 카테고리 |
|---|---|---|---|---|
| 계획 방문 | 필수 | 필수(슬롯 POI) | NULL | POI 카테고리 |
| 즉석 방문(POI 검색) | NULL | 필수(검색 POI) | NULL | POI 카테고리 |
| 즉석 방문(자유 텍스트) | NULL | NULL | 필수 | **'기타'**(좌표·카테고리 없음) |

**불변식**
- **INV-IMP1 (즉석 방문 슬롯 부재)**: ImpromptuVisit는 slotRef=NULL로만 존재하며 plan 대비 `diffKind=ADDED`로 대조에 노출된다 — 계획에 없던 방문이 침묵 누락되지 않는다 [G77, PRD 09-1, PRD 09-4].
- **INV-IMP2 (자유 입력 좌표·카테고리 부재)**: freeText 즉석 방문은 좌표·카테고리 없이 저장되고 스타일 분석에서 **'기타'로 처리**된다 — 자유 입력에 허위 좌표·카테고리를 조작하지 않는다 [G77, INV-STYLE 연계].
- **INV-IMP3 (동일 파이프라인)**: 즉석 방문도 사진·메모·오프라인 큐·귀속(§1·§3·§9)에서 계획 방문과 동일 규칙을 따른다 — 특수화는 슬롯 부재만, 나머지 생애는 VisitRecord와 동일 [G77].

---

## 3. MediaAttachment — 사진·메모 첨부 (M12, 부분 실패 격리)

방문 기록에 첨부되는 **사진(Photo)·메모(Memo)**. 사진은 클라이언트 압축 후 업로드하고 서버 썸네일을 생성하며 원본은 기기에 보관한다. **사진 업로드는 방문 체크·메모와 부분 실패 격리**(사진 실패가 메모·체크 저장을 막지 않음).

### 3.1 Photo — 사진

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| photoId | 식별자 | 필수 · 유니크 | — |
| visitId | 식별자(VisitRecord 참조) | 필수 | 귀속 방문(장소당 **최대 20장** — 상한 remote config) [G75/G145] |
| storageKey | 문자열 | 선택 | 오브젝트 스토리지 키. NULL=업로드 미완료(로컬 보관) [G168] |
| thumbnailKey | 문자열 | 선택 | 서버 생성 썸네일 키. NULL=업로드 미완료 [G75] |
| takenAt | 시각 | 필수 | 촬영/저장 시각(장소와 함께 저장) [PRD 09-2] |
| sizeMeta | 구조체 {width, height, bytes} | 필수 | 압축 결과 메타(**클라 압축 장당 5MB·긴 변 2048px**) [G75/G145] |
| exifStripped | 불리언 | 필수 · 기본 true | **클라 압축 시 EXIF(위치 포함) 제거 완료** 여부 [G75/G145, D34] |
| uploadState | 열거 {LOCAL, QUEUED, RETRYING, FAILED, DONE} | 필수 · 기본 `LOCAL` | 업로드 생애(§3.3) — 재시도 ≤3회 [PRD 09-2 예외] |
| retryCount | 정수 | 필수 · 기본 0 | 재시도 누적(상한 3 — remote config·초과 시 FAILED) [PRD 09-2] |

### 3.2 Memo — 메모

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| memoId | 식별자 | 필수 · 유니크 | — |
| visitId | 식별자(VisitRecord 참조) | 필수 | 귀속 방문 |
| text | 문자열 | 필수 | 자유 텍스트 메모 [PRD 09-2] |
| editedAt | 시각 | 필수 | 최종 편집 시각 |
| syncState | 열거 {LOCAL, PENDING, SYNCED, CONFLICT} | 필수 · 기본 `SYNCED` | 오프라인 동기화 상태(추가는 합집합 병합·§9) [G74] |

### 3.3 사진 업로드 상태 머신 (uploadState)

```text
   LOCAL(기기 보관·압축 완료·EXIF 제거)
     │ 네트워크 가용 → 업로드 시도
     ▼
   QUEUED ──(업로드 성공)──▶ DONE(storageKey·thumbnailKey 확정)
     │  └──(실패)──▶ RETRYING(retryCount++·지수 백오프)
     ▼                          │
   RETRYING ──(성공)──▶ DONE     │ retryCount > 3
     └──────────────────────────▶ FAILED('업로드 실패, 다시 시도' 버튼·수동 재시도)
   FAILED ──(사용자 수동 재시도)──▶ QUEUED
```

### 3.4 불변식

- **INV-MEDIA1 (부분 실패 격리)**: 사진 업로드 실패는 **메모·방문 체크 저장을 막지 않는다** — 사진은 로컬 보관+큐 잔류('업로드 대기'), 메모·체크는 독립 저장(uploadState와 무관) [PRD 09-2 예외, PBT U7-P4].
- **INV-MEDIA2 (재시도 멱등·상한)**: 동일 사진의 업로드 재시도는 최대 3회(remote config)까지 자동, 각 재시도는 멱등(중복 업로드·중복 레코드 0)이며 3회 초과 시 FAILED(수동 재시도 버튼) — 성공 시 DONE으로 종결(재시도 중단) [PRD 09-2, PBT U7-P4].
- **INV-MEDIA3 (EXIF 제거 — 위치 경계)**: 사진은 클라 압축 시 EXIF(위치 포함) 제거 후 업로드된다(exifStripped=true) — 위치 데이터는 GPS 옵트인 경로(§8 GpsTrack)로만 별도 관리한다(사진 메타에 위치 은닉 금지 D34). 커뮤니티 게시 시 추가 EXIF GPS 제거는 U10(G185) 소관 [G75/G145, D34, FD-U7-05].
- **INV-MEDIA4 (장소당 상한)**: 한 방문(visitId)당 사진은 최대 20장(remote config) — 상한 초과 첨부는 차단(사유 안내) [G75/G145].
- **INV-MEDIA5 (원본 기기 보관)**: 서버는 압축본·썸네일만 보존하고 원본은 기기에 보관한다 — 서버 부담 축소·원본 재업로드는 사용자 기기에서만 [G75/G145].

---

## 4. ChangeLogEntry — 변경 이력 통합 diff (M12, G132 보관 정본)

여행의 **일정 변경 이력**을 항목 단위 diff로 보관하는 **통합 스키마(G132)의 정본**. Plan-B(U6)·(후속) 공동편집(M17)·어시스턴트(M16)가 발행하는 모든 일정 변경을 하나의 스키마로 수용한다. **append-only**이며 스냅샷은 diff 누적 재구성으로 산출한다(current 대조 원장). U7이 보관·열람·재생을 소유하고, U6은 CP4로 diff를 **생산**(보관·열람 아님).

### 4.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| changeLogId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| targetRef | 식별자(ItinerarySlot 참조) | 필수 | 변경 대상 슬롯(항목 단위 diff) [G132] |
| actor | 구조체 {kind, ref} | 필수 · kind∈{USER, SYSTEM} | 행위자(변경자 닉네임·시스템) [G132, PRD 07-9] |
| sourceType | 열거 {PLAN_B, MANUAL, CO_EDIT, ASSISTANT} | 필수 | 출처 유형 — **PLAN_B·MANUAL은 1차, CO_EDIT·ASSISTANT는 후속 예약 enum**(스키마 변경 없이 편입) [G132, INV-CLOG3] |
| reason | 문자열 | 선택 | 변경 사유(휴무·날씨·이동 지연 등). NULL=사유 없음 [PRD 07-9, PRD 09-4] |
| beforeValue | 구조체 {poiInternalId?, time?, order?} | 선택 | 변경 전 값(POI는 **내부 ID 참조** — 스냅샷 아님). NULL=신규 추가 [G132] |
| afterValue | 구조체 {poiInternalId?, time?, order?} | 선택 | 변경 후 값. NULL=삭제 [G132] |
| at | 시각 | 필수 · 불변 | 발생 시각(순서 재구성 키) [G132] |
| schemaVersion | 정수 | 필수 | 스키마 버전(전방 호환 — U7 리스크 완화분) [CP4, G132] |
| causeMessageRef | 식별자(대화 메시지 참조) | 선택 | (후속 G13) 유발 어시스턴트 메시지 단방향 참조. 1차 NULL [G13] |

### 4.2 파생값

| 파생값 | 계산 | 근거 |
|---|---|---|
| reconstructedSnapshot | 확정 시점 기준 + at 오름차순 diff 누적 적용 | current 대조·회고 변경 요약 — **재구성 = current 스냅샷 동등**(INV-CLOG1) [G132] |

### 4.3 불변식

- **INV-CLOG1 (diff 누적 재구성 = current 동등성 — 1급 속성)**: 확정 시점 기준(plan)에 changelog diff 시퀀스를 at 오름차순으로 누적 적용한 재구성 결과는 **current 스냅샷과 일치**한다 — 임의 변경 시퀀스에 대해 재생 결과 = current(라운드트립). CP4로 U6이 생산한 실데이터로 통합 검증한다 [G132, CP4 시나리오 3, PBT U7-P1].
- **INV-CLOG2 (append-only·불변)**: ChangeLogEntry는 추가 전용 — 기존 항목의 수정·삭제 없음. at·beforeValue·afterValue는 생성 후 불변(감사 무결성) [G132, SECURITY-14 정합].
- **INV-CLOG3 (출처 유형 전방 호환)**: sourceType enum은 CO_EDIT·ASSISTANT를 **예약**하고 schemaVersion으로 전방 호환한다 — U9(어시스턴트)·U11(공동편집) 편입 시 스키마 변경 0(U10 공개 이력 공용) [G132, CP4 계약 변경 통제, INV-CLOG3].
- **INV-CLOG4 (POI 내부 ID 참조)**: before/afterValue의 POI는 내부 ID 참조로 저장한다(스냅샷 복제 아님) — 재구성 시 POI 정본을 조회해 해석(공개 스냅샷·마스킹은 U10 소관) [G132, PRD 09-4].
- **INV-CLOG5 (생산·보관 경계 — CP4)**: U6(M18·M10)이 CP4로 diff를 **생산**하고, U7(M12)이 보관·열람·재생을 소유한다 — 이 분업이 CP4의 핵심(U6 재계획 확정 TX에서 발행, U7이 구독·영속) [CP4, U6 INV-REPLAN2].

---

## 5. Reflection — 회고 (M13, 초안/수정본 G78)

**당일 회고**(DayClosed 트리거)의 초안·수정본. 방문·사진·메모·changelog를 근거로 C1(상위 티어)이 초안을 생성하고, 사용자가 수정하면 수정본을 원본과 **별도 저장**해 최종 표시본으로 사용한다. LLM 실패 시 기본 카드로 폴백한다.

### 5.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| reflectionId | 식별자 | 필수 · 유니크 | — |
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| date | 날짜 | 필수 · 유니크(여행 내) | 대상 일자(당일 회고 1일 1건) [PRD 09-6] |
| draftContent | 텍스트 | 선택 | AI 생성 **원본 초안**. NULL=생성 실패·폴백 상태 [PRD 09-6] |
| editedContent | 텍스트 | 선택 | 사용자 **수정본**(원본과 별도). NULL=미수정 — 존재 시 최종 표시본(INV-REFL1) [G78] |
| basis | 구조체 {visitCount, distanceKm, photoCount, majorChanges, missingItems} | 필수 | 입력 데이터 요약 + **누락 명시**('사진 없음'·'위치 기록 없음') [PRD 09-6 예외] |
| status | 열거 {GENERATED, FALLBACK_CARD, EDITED, NO_ACTIVITY} | 필수 | 회고 상태(§5.3) [ADR-0011] |

### 5.2 파생값

| 파생값 | 계산 | 근거 |
|---|---|---|
| displayContent | editedContent 존재 시 editedContent, 아니면 draftContent, 둘 다 NULL이면 기본 카드(basis) | 최종 표시본 선택(수정본 우선 INV-REFL1) [G78] |

### 5.3 회고 상태 전이 (status)

```text
   [DayClosed 트리거 → M13.generateDailyReflection]
     │
     ├─ 기록 0건 ──▶ NO_ACTIVITY('오늘 기록된 활동이 없습니다'·수동 기록 유도)
     ├─ C1 성공 ──▶ GENERATED(draftContent·basis)
     └─ C1 실패/타임아웃 ──▶ FALLBACK_CARD(통계만·basis·LLM 문구 없음)
   GENERATED / FALLBACK_CARD ──(사용자 편집 editReflection)──▶ EDITED(editedContent 별도 저장)
   GENERATED / EDITED ──(regenerateReflection)──▶ [수정본 존재 시 OverwriteWarning → 확인 후에만 교체 G78]
```

### 5.4 불변식

- **INV-REFL1 (수정본 별도·우선 보존)**: editedContent는 draftContent와 **별도 저장**되고 존재 시 최종 표시본이다 — 재생성이 수정본을 덮어쓰려면 `overwriteConfirmed`(덮어쓰기 경고 확인)가 필수(G78). 확인 없는 재생성은 수정본을 보존한다 [G78, PBT U7-P8].
- **INV-REFL2 (폴백 침묵 금지)**: C1 실패·타임아웃 시 빈 화면 대신 **기본 카드('방문 N곳·이동 Nkm·사진 N장')**를 제공하고, 사용자는 기본 카드 위에 직접 작성할 수 있다(status=FALLBACK_CARD→EDITED) [ADR-0011, PRD 09-6·7, INV-REFL1].
- **INV-REFL3 (부분 데이터 누락 명시)**: 가용 데이터만으로 생성하고 생성 불가 항목(사진 0장→사진 하이라이트·위치 없음→지도 경로)은 제외하되 basis.missingItems에 **누락을 명시**한다('사진 없음'·'위치 기록 없음') — 누락을 침묵하지 않는다 [PRD 09-6 예외].
- **INV-REFL4 (오프라인 생성 보류)**: 오프라인 구간에서는 당일 회고 자동 생성을 보류하고 복구 후 생성하거나 기본 카드로 대체한다 — 생성은 온라인 전제 [Δ6, PRD 09-12 예외].
- **INV-REFL5 (종료 후 수동 재생성만 — C11)**: 여행 종료 후 기록 편집분의 회고 반영은 사용자 수동 '다시 생성'으로만 갱신되고 자동 갱신은 없다 [C11, INV-REFL1].

---

## 6. TripSummary — 전체 여행 요약 (M13, TripEnded 트리거)

여행 종료(`TripEnded`) 시 생성되는 **전체 요약**. 지도 히어로(방문 순서 핀·날짜별 동선·기준 숙소 마커)를 중심으로 총계와 날짜별 하이라이트를 구성한다. 이동 거리는 GPS 동의 구간 실측 + 미동의 구간 추정을 혼합 합산한다(G72).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 · 유니크 | 귀속 여행(여행당 1건) |
| stats | 구조체 {visitCount, totalDistanceKm, photoCount} | 필수 | 총 방문 수·**총 이동 거리(G72 혼합·추정 표기)**·총 사진 수 [PRD 09-8, G72] |
| dailyHighlights | 목록<{date, highlight}> | 필수 | 날짜별 하이라이트 [PRD 09-8] |
| mapHeroSpec | 구조체 {orderedPins, dailyRoutes, stayMarkers, routeMode} | 필수 | 지도 히어로 스펙 — routeMode∈{GPS_ACTUAL, MANUAL_LINKED} 구간별(동의=실측·미동의=체크인 연결) [PRD 09-8, D34] |
| status | 열거 {GENERATED, FALLBACK, MAP_UNAVAILABLE} | 필수 | 생성/폴백(날짜별 기본 카드 모음)/지도 대체(방문 목록) [ADR-0011] |

**불변식**
- **INV-SUM1 (종료 트리거 유일·멱등)**: TripSummary는 `TripEnded`(자동 익일 00:00·수동 버튼 D19/Δ4)로만 생성되며, M13의 (tripId, kind) 핸들러는 기존 요약 존재 시 skip(이벤트 중복 수신 안전) [D19/Δ4, services S4, PBT 연계].
- **INV-SUM2 (이동 거리 혼합·추정 표기)**: totalDistanceKm은 GPS 동의 구간 실측 폴리라인 합산 + 미동의 구간 장소 간 거리 추정의 혼합 합산이며 **추정치임을 표기**한다(구간 분할·순서 불변에 결정적) [G72, G59, PBT U7-P5].
- **INV-SUM3 (지도·폴백 계단)**: 위치 데이터 전무 시 지도 대신 방문 목록(순서·날짜)으로 대체(MAP_UNAVAILABLE)하고, 요약 생성 실패 시 날짜별 기본 카드 모음 폴백(FALLBACK) — 어떤 경로도 빈 화면 없음 [PRD 09-8 예외, ADR-0011].
- **INV-SUM4 (종료 후 수동 재생성 — C11)**: 종료 후 기록 편집분 반영은 `regenerateTripSummary` 수동 재생성으로만(자동 갱신 없음) [C11].

---

## 7. TravelStyleAnalysis — 여행 스타일 분석 (M13, 10곳 게이트 G76)

계정 누적 방문 패턴 기반 **스타일 분석**. **누적 방문 장소 수 ≥ 10곳 단일 게이트**로 판정하며, 미달 시 온보딩 취향 기반 임시 미리보기(정식 아님 명시)+진행 게이지를 노출한다. 분류는 온보딩 취향 7종 축 매핑 자체 택소노미.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| accountId | 식별자(Account 참조) | 필수 · 유니크 | 귀속 계정(계정 단위 누적 분석) |
| metrics | 구조체 {categoryDistribution, avgStay, travelRadius, activityDensity} | 선택 | 카테고리 분포·평균 체류·이동 반경·활동 밀도. NULL=게이트 미달(임시 미리보기) [PRD 09-9] |
| basedOnVisitCount | 정수 | 필수 | 근거 누적 방문 장소 수(게이트 판정 카운트) [PRD 09-9] |
| mode | 열거 {OFFICIAL, PREVIEW} | 필수 | OFFICIAL(≥10곳 정식)·PREVIEW(<10곳 온보딩 취향 기반 임시) [G76] |
| taxonomyAxis | 열거<취향 7종 축> | 필수 | 온보딩 취향 7종 축 매핑(임시 미리보기와 스키마 공유) [G76] |
| updatedAt | 시각 | 필수 | 최종 분석 시점 |

**불변식**
- **INV-STYLE1 (10곳 단일 게이트·단조)**: 정식 분석(mode=OFFICIAL)은 basedOnVisitCount ≥ 10에서만 생성되고, 미달 시 PREVIEW(임시 미리보기·정식 아님 명시)+진행 게이지('N/10곳')를 반환한다. 게이트는 방문 수에 대해 **단조**(방문 수 증가가 게이트 통과를 뒤집지 않음) [PRD 09-9, G76, PBT U7-P7].
- **INV-STYLE2 (방문 수 기준 우선)**: '완료 여행 N건' 표현은 안내용 환산일 뿐이며, 실제 트리거 조건은 항상 누적 방문 장소 수 기준이다(충돌 시 방문 수 우선) [PRD 09-9, FD-U7-08].
- **INV-STYLE3 (근거 동반·자체 택소노미)**: 분석은 구체 수치·카테고리로 제시하고 근거 방문 데이터를 동반하며, 분류는 취향 7종 축 매핑 자체 택소노미(임시 미리보기와 스키마 공유)를 사용한다 — 즉석 자유 입력은 '기타' [PRD 09-9, G76, G77].

---

## 8. GpsTrack — GPS 발자취 (M12, 영속 정본·수집 U6)

계획 동선 vs 실제 경로 비교의 원천. **수집·단순화는 U6**(포그라운드 저빈도·원시 좌표 파기), **기록 귀속·영속·경로 비교 데이터는 M12(U7)**(CP4 소비). GPS 기록 옵트인 전제이며 철회·탈퇴 시 즉시 파기한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| tripId | 식별자(Trip 참조) | 필수 | 귀속 여행 |
| date | 날짜 | 필수 · 유니크(여행 내) | 수집 일자(날짜별 귀속) |
| simplifiedPolyline | 폴리라인(단순화) | 필수 · 불변 | **단순화 폴리라인만 보존**(원시 좌표 파기 후 — 수집 U6) [G55/G73] |
| consentRef | 식별자(ConsentRecord 참조) | 필수 | GPS 기록 옵트인 동의 참조 — 미동의 귀속 불가 [D34/N2] |

**파생 표시값**: cumulativeDistance(누적 실제 이동 거리·추정), estimatedSteps(걸음 수=GPS 거리 환산 추정 G59) — 둘 다 **추정 표기 강제**.

**불변식**
- **INV-GPS1 (영속·귀속 경계 — CP4)**: U6은 수집·폴리라인 생산까지, **기록 귀속·경로 비교·영속 정본은 M12(U7)** — CP4로 공급된 폴리라인을 날짜·여행에 귀속한다 [CP4, U6 INV-GPS5, FD-U7-10].
- **INV-GPS2 (옵트인 철회 즉시 파기 — N2)**: 옵트인 철회·계정 탈퇴 시 GpsTrack·위치 파생 데이터는 **즉시 파기**(30일 유예 없음 D34, `purgeLocationData`) — 단 법정 로그(수집·이용 사실)는 분리 보관 유지(앱은 자기 로그 삭제 권한 없음) [D34/N2, S6.2].
- **INV-GPS3 (경로 비교 폴백)**: 옵트인/권한 없는 구간은 실제 경로 레이어 `disabled(reason)`로 계획 동선만 제공하고, 위치 데이터 전무 시 방문 목록으로 대체(§6 INV-SUM3 정합) — 미동의 구간은 수동 체크인 장소 순서 연결 [D34, PRD 09-8].

---

## 9. OfflineSyncRecord — 오프라인 동기화 봉투 (M12, 충돌 G74)

여행 중 오프라인 상태에서 기기 로컬에 저장된 **기록 입력**(방문 체크·사진 메타·메모·수동 체크인)의 동기화 봉투. 레코드 단위 버전 비교로 충돌을 판정하며, **배치 전체 원자성은 의도적으로 없다**(레코드별 독립 커밋·부분 성공 허용). 사진·메모 추가는 합집합 병합(충돌 아님), 상태 필드만 충돌 대상.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| recordId | 식별자(클라 생성 UUID) | 필수 · 유니크 | 레코드 멱등 키(이미 적용된 레코드 skip) [S8] |
| kind | 열거 {VISIT_CHECK, PHOTO_META, MEMO, MANUAL_CHECKIN} | 필수 | 입력 유형 [PRD 09-12] |
| recordedAt | 시각 | 필수 · 불변 | 원 발생 시각(늦은 동기화 시 M9 트리거 평가 제외 임계) [S8] |
| baseVersion | 정수 | 필수 | 기준 서버 버전(충돌 비교) [G74] |
| syncState | 열거 {LOCAL, PENDING, SYNCED, CONFLICT} | 필수 · 기본 `LOCAL` | 동기화 생애(§9.2) [Δ6] |
| payload | 구조체 | 필수 | 유형별 입력 페이로드(직렬화 왕복 대상 — PBT U7-P… 큐 페이로드) [S8] |

### 9.1 충돌 판정·해소

| 상황 | 처리 | 근거 |
|---|---|---|
| recordId 이미 적용 | skip(멱등) | INV-SYNC1 [S8] |
| 서버 측 변경 없음 | 적용 + `VisitChecked`(syncedFromOffline=true, recordedAt 원 시각 보존) | S8 3단계 |
| 서버 측 변경 존재(상태·시각 충돌) | conflict 목록 반환 → 사용자 항목별 선택(KeepLocal/KeepServer) | G74 [PRD 09-12] |
| 사진·메모 **추가** | 합집합 자동 병합(충돌 아님) | INV-SYNC2 [G74] |

### 9.2 동기화 상태 (syncState)

```text
   LOCAL(오프라인 로컬 큐·'동기화 대기') ──재연결──▶ PENDING(배치 업로드)
     PENDING ──(충돌 없음·멱등 적용)──▶ SYNCED
     PENDING ──(상태·시각 충돌)──▶ CONFLICT ──(사용자 항목별 선택·재제출)──▶ SYNCED
```

### 9.3 불변식

- **INV-SYNC1 (레코드 멱등·독립 커밋)**: recordId 기준 이미 적용된 레코드는 skip(멱등)되고, 배치는 레코드별 독립 TX로 부분 성공을 허용한다(전체 원자성 의도적 부재) — 응답에 per-record 결과(applied/conflict/failed) [S8, PBT U7-P3].
- **INV-SYNC2 (합집합 병합·무손실)**: 사진·메모 **추가**는 충돌 없이 합집합 병합되고, 상태 필드(완료/스킵·시각)만 충돌 대상이다 — 어떤 해소 경로도 데이터 무손실(양쪽 보존·임의 폐기 금지) [G74, PBT U7-P3].
- **INV-SYNC3 (해소 결정성·교환법칙)**: 동일 충돌 집합에 대해 사용자 선택(KeepLocal/KeepServer)이 같으면 결과가 결정적이고, 독립 레코드의 동기화 순서를 바꿔도 최종 수렴 상태가 동일하다(병합 교환법칙) [G74, PBT U7-P3].
- **INV-SYNC4 (입력 한정·조회 미보장)**: 오프라인 보장은 기록 '입력'에 한정한다 — 일정 '조회'는 온라인 전제(D24)로 오류·재시도 안내만. 조회 오프라인 캐시 없음 [D24/Δ6, PRD 09-12].
- **INV-SYNC5 (늦은 동기화 오발화 방지)**: recordedAt이 현재 시각 임계를 초과한(뒤늦은) 동기화는 `VisitChecked` 발행 시 M9 트리거 평가에서 제외한다(과거 기록의 재계획 오발화 방지) [S8, U6 INV-TRIG 정합].

---

## 10. ShareCard — SNS 공유 카드 (M13, 3포맷)

전체 요약·당일 회고를 **SNS 공유용 정적 이미지 카드**로 구성한다. 대표 사진·제목·기간·핵심 통계·동선·워터마크를 자동 구성하며 3포맷을 지원하고 사진 0장 시 지도·통계 레이아웃으로 대체한다. **외부 SNS로 내보내는 정적 이미지 채널**(커뮤니티 앱 내부 동선 공개와 병렬·별개).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| source | 참조 {TripSummaryRef \| DailyReflectionRef} | 필수 | 공유 원천(전체 요약 또는 당일 회고) [PRD 09-13] |
| format | 열거 {STORY_9_16, SQUARE_1_1, FEED_4_5} | 필수 | 카드 포맷 [PRD 09-13] |
| caption | 문자열 | 선택 | 캡션(편집 가능·금칙어 검증 C3) [PRD 09-13, C3] |
| hashtags | 목록<문자열> | 선택 | 해시태그(편집 가능) [PRD 09-13] |
| layoutVariant | 열거 {WITH_PHOTO, NO_PHOTO} | 필수 | 사진 유/무 레이아웃 — 사진 0장은 지도·통계 기반 [PRD 09-13 예외] |

**불변식**
- **INV-SHARE1 (생성 전제)**: 공유 카드는 여행 종료·요약 생성 후에만 진입 가능 — 미종료/요약 미생성이면 진입점 비활성(`PreconditionFailed`) [PRD 09-13 예외].
- **INV-SHARE2 (사진 0장 폴백)**: 대표 사진이 없으면(사진 0장) 지도 경로·통계 기반 '사진 없는 카드' 레이아웃으로 자동 대체(NO_PHOTO) — 빈 카드 없음 [PRD 09-13 예외].
- **INV-SHARE3 (외부 채널 경계·통계만)**: 공유 카드는 외부 SNS 내보내기용 정적 이미지이며 거리 통계만 표기(소요시간 없음 D25) — 앱 내부 동선 공개(커뮤니티 U10)와 별개 채널. 캡션은 금칙어 검증 [PRD 09-13, D25, C3].

---

## 11. CP4 소비 참조 계약 (U6 → U7 입력) — actual·changelog·GPS

U7은 U6(M18·M10)이 CP4로 발행하는 이벤트·객체를 구독해 actual·회고·기록을 생산한다(스키마 정본은 U6, 본 유닛 재정의 금지). `common/core` 이벤트 버스(U1 계약) 위에서 동작한다.

| 입력 이벤트/객체(U6 소유) | U7 소비 용도 | 소비 시 처리 |
|---|---|---|
| `VisitChecked`(start/complete/skip·slotRef·canonicalPoiId·도착 시각·전이 유형·실제 체류·departedEstimatedFlag) | actual 방문 기록(VisitRecord) 생산·보관·plan 대조 | 멱등 생산(syncedFromOffline 정합)·체류 결정성 승계(INV-REC1·2) [CP4, D23] |
| `DayClosed`(여행·일자·당일 방문 요약) | 당일 회고 초안 자동 생성 트리거(M13) | (tripId, date) 멱등 생성·오프라인 보류(INV-REFL4) [CP4] |
| `TripEnded`(여행·종료 방식·종료 시각) | 전체 요약·스타일 분석 트리거 | (tripId, kind) 멱등 skip·자동/수동 각각 도달(INV-SUM1) [CP4, D19/Δ4] |
| changelog diff(G132 통합 스키마) | changelog 보관·열람·재생(재구성 대조) | append-only 영속·재구성=current 동등(INV-CLOG1·5) [CP4, G132] |
| GPS 폴리라인(단순화·원시 파기·consentRef) | GpsTrack 귀속·경로 비교 데이터 | 날짜·여행 귀속·철회 시 파기 연동(INV-GPS1·2) [CP4, D34] |

**검증 방법 — 통합 테스트 시나리오(CP4)**
1. **actual 생산 파이프라인**: 방문 완료 체크(U6) → `VisitChecked` 발행 → U7 actual 방문 기록 생성 → plan/actual/changelog 3계열 대조(US-E8-04) 정확 표시.
2. **회고 트리거 연쇄**: `TripEnded`(자동·수동 각각) 수신 시 전체 요약 생성 기동, LLM 실패 시 기본 카드 폴백까지 도달(침묵 실패 금지).
3. **changelog 재생 동등성**: U6 재계획 생산 changelog diff 시퀀스를 U7이 순서대로 재생한 결과가 current 스냅샷과 일치(U7 PBT 1급 속성 U7-P1을 U6 실생산 데이터로 통합 검증).

> **계약 변경 통제**: changelog 통합 스키마(G132)는 후속 3유닛(U9·U10·U11) 공용 — schemaVersion 필드로 전방 호환, diff 재생 PBT(U7-P1)를 회귀 안전망으로(CP4 §167).

---

## 12. CP5 공급 계약 (U7 → U8 출력) — 회고 완료 이벤트

U7 산출(회고·요약·분석)의 완료를 U8 알림이 소비한다. U7(M13)이 발행하고 U8(M14·S5)이 단일 구독·스케줄링한다.

| 이벤트/객체 | 필드(개요) | 경계 무결성 |
|---|---|---|
| `ReflectionReady` | 회고 유형(DAILY/TRIP_SUMMARY/STYLE_ANALYSIS), 여행·일자 참조, 대상 계정 — 회고 완료 알림 트리거 | INV-REFL2, INV-SUM1 |
| plan/actual/changelog 3계열 대조 데이터 | getTripTimeline(라벨 구분·차이 하이라이트) — U8 마이페이지·(후속)U10 공개 스냅샷의 소스 | INV-REC3, INV-CLOG1 |

> **제외(U8 소관)**: 회고 완료 알림 **발송**·마이페이지 통합·전체 아카이브 내보내기는 U8 소유. U7은 `ReflectionReady` 발행까지(FD-U7-11) [CP5, unit-of-work §U7 제외].

---

## 13. C1 재사용 계약 (U5 자산 — 계약 수준)

U7(M13)의 회고·요약·스타일 분석 LLM 호출은 전부 U5가 완성한 **C1 LLM Gateway(상위 티어)를 재사용**한다(중복 재구현 금지, FD-U7-06).

| 자산 | U7 재사용 용도 | 재사용 계약(입출력) | 폴백 소유 |
|---|---|---|---|
| C1 LLM Gateway | 당일 회고·전체 요약·(스타일 분석 서술) 생성 | `C1.call(reflection, serverContext)` 상위 티어(D11) · 서버 재조회 컨텍스트 주입(D31) · 출력 스키마 검증 | 실패·타임아웃 시 **M13 기본 카드 폴백**(통계만·침묵 실패 금지, ADR-0011) |

> **재사용 원칙(FD-U7-06)**: U7은 C1의 계약 인터페이스만 소비하고 LLM 게이트웨이·closed-set·계정 무결성(D31 서버 재조회)을 재구현하지 않는다 — 회고 LLM 입력은 서버 재조회로 요청자 권한 밖 데이터 구조적 미포함(U5 INV-SCORE4·SECURITY-11 재사용) [D11, D31, SECURITY-11].

---

## 14. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| VisitRecord | US-E8-01, US-E8-04·05, US-E6-01 | D23, D14, CP4, C11 |
| ImpromptuVisit | US-E8-01 | G77 |
| MediaAttachment(Photo·Memo) | US-E8-02 | G75/G145, G168, D34 |
| ChangeLogEntry | US-E8-04, US-E7-09 | G57/G132, G13, CP4 |
| Reflection | US-E8-06·07·12 | G78, C11, ADR-0011, D11 |
| TripSummary | US-E8-08 | D19/Δ4, G72, D34, C11 |
| TravelStyleAnalysis | US-E8-09·10 | G76, G77 |
| GpsTrack | US-E8-03, US-E7-13 | G55/G73, D34/N2, G59, G72 |
| OfflineSyncRecord | US-E8-12 | G74, D24/Δ6, S8 |
| ShareCard | US-E8-13 | D25, C3 |
