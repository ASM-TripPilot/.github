# U7 기록·회고 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/archive`(기록 타임라인·방문 편집·즉석 방문·사진·회고·전체 요약·스타일 분석·공유 카드·캘린더) + `shared/storage`(오프라인 입력 로컬 큐 Δ6)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `archive` 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M12·M13 메서드를 참조하고, 폼·검증 규칙은 [business-rules.md](./business-rules.md) BR-U7-xx의 클라이언트측 대응(판정 정본은 항상 서버 — 회고·요약·분석은 서버 M13, changelog 재구성·오프라인 병합은 서버 M12)이다.
> 기술 중립: 상태 관리·내비게이션·스타일·오프라인 큐 저장 구현체는 NFR/Code Generation 소유. **지도 렌더링은 U3 `shared/map`(카카오 지도 SDK 브리지) 재사용 + U5 지도 뷰·U6 PlanVsActualMapScreen 레이어 재사용** — U7은 새 지도 브리지를 만들지 않는다(FD-U7-10 연계).
> data-testid 규약: `archive-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 기록·회고 화면 플로우 (정본)

```text
[여행 중 방문 체크(U6 execution) → CP4 VisitChecked → M12 actual] · [여행 종료 TripEnded → M13 회고]
        │
        ▼
┌─ RecordTabScreen ('기록' 탭 — 여행별 기록 목록 + 캘린더 진입) ──────────────────────────┐
│   여행 단위 목록(예정/진행/종료) · RecordCalendar(월 마킹·겹침 없음 D21) · 0건 빈 상태     │
└──────────────────────────────────────────────────────────────────────────────────────┘
   │ 여행 선택
   ▼
┌─ TripRecordDetailScreen (날짜별 기록 상세 — plan/actual/changelog 3계열 대조 D14) ───────┐
│   ThreeSeriesTimeline(계획·실제·변경 라벨·색상·아이콘 구분) · 미방문/추가/순서 변경 하이라이트 │
│   SyncStatusBanner(동기화 대기·충돌 배지)                                                  │
└──────────────────────────────────────────────────────────────────────────────────────┘
   │ 방문 편집          │ 사진·메모        │ 즉석 방문           │ 회고               │ 전체 요약/공유
   ▼                   ▼                 ▼                    ▼                   ▼
VisitRecordEditor   MediaAttachSheet  ImpromptuVisitSheet  DailyReflectionCard  TripSummaryScreen
(시각·상태 수정 D23) (사진 큐·메모 G75)  (POI 검색/자유 G77)   (초안·수정·재생성 G78) (지도 히어로·통계)
                          │ 업로드 실패/대기                                      │ '공유'
                          ▼                                                      ▼
                    UploadQueueStatus                                      ShareCardEditor
                    (재시도 3회·수동 재시도)                                 (3포맷·사진 0장 폴백)
   │ 계획 vs 실제 경로 / 스타일 분석
   ▼                              ▼
RouteComparisonView          TravelStyleScreen
(U3/U5/U6 map 재사용·발자취)   (10곳 게이트·임시 미리보기·게이지)
        │ 오프라인 충돌
        ▼
SyncConflictResolver (레코드 단위 두 버전·항목별 선택 G74)
```

**전역 규칙**
- 기록·회고 화면군은 5탭 셸(U2) 내 '기록' 탭 스택 + 마이페이지(U8) 진입에서 접근한다(마이페이지 통합 최종은 U8). 여행 단위 목록·캘린더 양쪽 진입(BR-U7-28).
- **계획(plan)·실제(actual)·변경(changelog) 3계열은 항상 라벨·색상·아이콘으로 시각 구분**해 렌더한다(D14·US-E8-04). plan은 U5 불변 스냅샷 참조, actual·changelog는 M12 소유.
- **사진 업로드 실패는 메모·방문 체크와 격리** — 사진은 '업로드 대기'·재시도 3회·수동 재시도, 메모·체크는 독립 저장(BR-U7-09).
- **오프라인 입력은 shared/storage 로컬 큐**('동기화 대기')로 저장하고 복구 시 자동 동기화 — **조회는 온라인 전제**(오프라인 캐시 없음·오류·재시도 안내 D24).
- **회고 수정본은 최종 표시본** — 재생성 시 수정본 존재하면 덮어쓰기 경고(BR-U7-19). 실패 시 기본 카드·직접 작성(BR-U7-16).
- **모든 통계·거리는 추정 표기**('약 Nkm 추정'·걸음 수 추정)이며 **소요시간 필드는 어떤 화면에도 없다**(D25 승계). 공유 카드도 거리 통계만.
- **GPS·실제 경로 레이어는 옵트인/권한에 게이트** — 미동의 시 disabled+사유 표기(계획 동선/방문 목록 폴백 BR-U7-26).

## 2. 컴포넌트 계층

```text
features/archive
├─ ArchiveNavigator                        — 기록 스택('기록' 탭·마이페이지 진입 — 통합 U8)
│  ├─ RecordTabScreen                       — 여행별 기록 목록(예정/진행/종료)·0건 빈 상태
│  │  ├─ ArchivedTripCard                   — 여행 카드(제목·기간·방문 수·대표 사진)
│  │  └─ RecordCalendar                     — 월 캘린더(여행 기간 마킹·겹침 없음 D21·월 이동)
│  ├─ TripRecordDetailScreen                — 날짜별 기록 상세(3계열 대조)
│  │  ├─ ThreeSeriesTimeline                — plan/actual/changelog 라벨·색상·아이콘 구분(D14)
│  │  │  ├─ PlanActualDiffRow               — 미방문/추가/순서 변경 하이라이트(diffKind)
│  │  │  └─ ChangeLogItem                   — 변경 이력(시각·전/후 장소·사유·행위자 G132)
│  │  ├─ SyncStatusBanner                   — 동기화 대기·충돌 배지(오프라인 상태)
│  │  └─ BaseAttributionHeader              — 날짜별 기준 숙소·귀속(숙소 없는 날=날짜만 D15)
│  ├─ VisitRecordEditor                     — 방문 기록 편집(도착/종료 시각·상태 수정 D23)
│  │  ├─ VisitTimeEditRow                   — 자동 기록 시각 직접 수정(수정 가능)
│  │  └─ VisitStatusToggle                  — 방문 완료/방문 안 함·취소(undo)
│  ├─ ImpromptuVisitSheet                   — 즉석 방문 추가(POI 검색 ∨ 자유 텍스트 G77)
│  │  ├─ PoiSearchInput                     — POI 검색(좌표·카테고리 있는 스냅샷)
│  │  └─ FreeTextPlaceInput                 — 자유 입력(좌표·카테고리 없음 → 분석 '기타')
│  ├─ MediaAttachSheet                      — 사진·메모 첨부(장소당 20장·클라 압축 G75/G145)
│  │  ├─ PhotoPicker                        — 사진 선택·압축(5MB/2048px·EXIF 제거)
│  │  ├─ UploadQueueStatus                  — 업로드 대기/재시도(≤3)/실패·수동 재시도 버튼
│  │  └─ MemoEditor                         — 자유 텍스트 메모(사진 실패와 독립 저장)
│  ├─ DailyReflectionCard                   — 당일 회고(초안·수정·재생성 G78)
│  │  ├─ ReflectionContentView              — 회고 문구(수정본 우선 표시 INV-REFL1)
│  │  ├─ ReflectionEditControls             — 수정·메모 덧붙임(수정본 별도 저장)
│  │  ├─ RegenerateButton                   — 재생성(수정본 존재 시 OverwriteWarning 다이얼로그)
│  │  └─ FallbackReflectionCard             — 기본 카드(방문 N곳·이동 Nkm·사진 N장·직접 작성)
│  ├─ TripSummaryScreen                     — 전체 여행 요약(지도 히어로·통계·하이라이트)
│  │  ├─ MapHeroView                        — 방문 순서 핀·날짜별 동선·거점 마커·사진 미리보기(U3/U5 map)
│  │  ├─ SummaryStatsBar                    — 총 방문·총 이동 거리(G72 추정)·총 사진 수
│  │  ├─ DailyHighlightList                 — 날짜별 하이라이트
│  │  └─ SummaryFallbackView                — 지도 대체(방문 목록)·기본 카드 모음 폴백
│  ├─ RouteComparisonView                   — 계획 vs 실제 경로(U6 PlanVsActualMap 재사용·발자취)
│  │  ├─ PlannedRouteLayer / ActualTrailLayer — 계획 동선·GPS 발자취(옵트인 게이트)
│  │  └─ TrailStatsBar                      — 누적 실제 거리·걸음 수(추정 표기 G59)
│  ├─ TravelStyleScreen                     — 스타일 분석(10곳 게이트 G76)
│  │  ├─ StyleMetricsChart                  — 카테고리 분포 막대·이동 반경 원·분포점(지도)
│  │  ├─ StylePreviewCard                   — 임시 미리보기(정식 아님 명시·온보딩 취향 기반)
│  │  └─ GateProgressGauge                  — 진행 게이지('N/10곳')
│  ├─ ShareCardEditor                       — SNS 공유 카드(3포맷·캡션·해시태그)
│  │  ├─ CardFormatSelector                 — 9:16 / 1:1 / 4:5
│  │  ├─ CardCaptionInput                   — 캡션·해시태그 편집(금칙어 검증 C3)
│  │  └─ NoPhotoCardLayout                  — 사진 0장 폴백(지도·통계 기반)
│  └─ SyncConflictResolver                  — 오프라인 충돌 해소(레코드 단위 두 버전·항목별 선택 G74)
└─ shared/storage                           — 오프라인 입력 로컬 큐(U7 신규 클라이언트 자산, Δ6)
   ├─ OfflineRecordQueue                    — 방문 체크·사진 메타·메모·수동 체크인 로컬 큐(recordId·recordedAt)
   ├─ SyncOrchestrator                      — 재연결 감지·배치 동기화·per-record 결과 반영
   └─ PhotoUploadWorker                     — 사진 업로드 큐·압축·재시도 3회·부분 실패 격리
```

---

## 3. 기록 화면군 (features/archive)

### 3.1 RecordTabScreen + RecordCalendar (기록 목록·캘린더)
- **props**: `accountId` · **state**: 여행별 기록 목록(예정/진행/종료), 캘린더 월·마킹.
- **폼 검증(서버 규칙 대응)**: 저장 기록을 여행 단위로 목록화(BR-U7-28). 월 캘린더에 여행 기간(체크인~체크아웃) 마킹 — **여행 겹침 차단(D21)으로 비중첩**. '기록' 탭·마이페이지 양쪽 진입·월 이동. **0건이면 빈 목록/빈 캘린더 대신 '아직 기록된 여행이 없습니다'+새 여행 생성 진입점**.
- **상호작용**: `M12.listArchivedTrips` · `M12.getRecordCalendar(month)`.
- **data-testid**: `archive-recordtab-list`, `archive-recordtab-empty`, `archive-calendar-month`, `archive-calendar-mark`.
- **사용 서버 능력**: `M12.listArchivedTrips`, `M12.getRecordCalendar`.

### 3.2 TripRecordDetailScreen + ThreeSeriesTimeline (3계열 대조)
- **props**: `tripId` · **state**: 날짜별 기록 묶음, plan/actual/changelog 대조 데이터, 동기화 상태.
- **폼 검증(서버 규칙 대응)**: **계획·실제·변경 3종을 라벨·색상·아이콘으로 시각 구분**(D14·US-E8-04). 계획과 실제가 다른 장소(미방문·추가·순서 변경)를 한눈에 비교(PlanActualDiffRow diffKind). 각 변경 이력 항목은 **변경 시각·전/후 장소·사유·행위자**(G132) 포함. 날짜별 기준 숙소 귀속(숙소 없는 날=날짜만 BR-U7-04). 동기화 대기·충돌 배지 노출.
- **상호작용**: `M12.getTripTimeline`(plan/current/actual/changelog 4종) · `M12.getTripRecords`(날짜별·귀속 기준).
- **data-testid**: `archive-detail-timeline`, `archive-detail-planactual`, `archive-detail-changelog`, `archive-detail-syncbanner`.
- **사용 서버 능력**: `M12.getTripTimeline`, `M12.getTripRecords`, `M4.getBaseTimeline`(귀속 기준).

### 3.3 VisitRecordEditor + ImpromptuVisitSheet (방문 편집·즉석 방문)
- **props**: `tripId`, `visitId?` · **state**: 방문 기록(시각·상태), 즉석 입력(POI/자유).
- **폼 검증(서버 규칙 대응)**: 자동 기록된 방문 시각을 **직접 수정 가능**(D23). 방문 완료/방문 안 함·취소(undo). **여행 종료 후에도 편집 허용**하되 회고·분석 자동 갱신 없음(수동 재생성 C11). 즉석 방문은 **POI 검색(좌표·카테고리) 또는 자유 텍스트(좌표·카테고리 없음 → 분석 '기타')**(BR-U7-03, G77). 상태 전이 강제는 서버(U6) — 편집은 actual 기록 갱신.
- **상호작용**: `M12.updateVisitRecord(patch)` · `M12.addImpromptuVisit(PoiRef | FreeText)`.
- **data-testid**: `archive-visit-timeedit`, `archive-visit-statustoggle`, `archive-impromptu-poisearch`, `archive-impromptu-freetext`.
- **사용 서버 능력**: `M12.updateVisitRecord`, `M12.addImpromptuVisit`.

### 3.4 MediaAttachSheet + UploadQueueStatus (사진·메모)
- **props**: `visitId` · **state**: 사진 목록·업로드 상태, 메모, 큐.
- **폼 검증(서버 규칙 대응)**: 한 장소에 **사진 여러 장(≤20장)+메모**(BR-U7-08·10). 사진은 **클라 압축(5MB/2048px)·EXIF 제거** 후 업로드·서버 썸네일·원본 기기 보관. 업로드 실패 시 **'업로드 대기'·자동 재시도 3회·'업로드 실패, 다시 시도' 버튼**(BR-U7-09). **메모·방문 체크는 사진 실패와 무관하게 저장**(부분 실패 격리). 촬영/저장 시각·장소 함께 저장.
- **상호작용**: `M12.attachMedia(photos, memo)` → `MediaResult{accepted, pending, rejected}`. 업로드 워커는 `shared/storage` PhotoUploadWorker.
- **data-testid**: `archive-media-photopicker`, `archive-media-uploadqueue`, `archive-media-retry`, `archive-media-memo`.
- **사용 서버 능력**: `M12.attachMedia`.

### 3.5 DailyReflectionCard (당일 회고)
- **props**: `tripId`, `date` · **state**: 회고(초안/수정본·status), 재생성 경고.
- **폼 검증(서버 규칙 대응)**: 방문·사진·메모·변경 이력 기반 당일 회고 초안 표시(방문 수·이동 거리·사진 수·주요 변경 포함 BR-U7-15). **수정본이 최종 표시본**(수정본 별도 저장 INV-REFL1). **재생성 시 수정본 존재하면 `OverwriteWarning` 다이얼로그**(확인 후에만 교체 G78). **C1 실패 시 기본 카드(통계만)+직접 작성**(BR-U7-16). **부분 데이터 누락 명시**('사진 없음'·'위치 기록 없음' BR-U7-17). 기록 0건이면 'NoActivity' 안내+수동 기록 버튼. 오프라인 구간 생성 보류.
- **상호작용**: `M13.generateDailyReflection` · `M13.editReflection` · `M13.regenerateReflection(overwriteConfirmed)`.
- **data-testid**: `archive-reflection-content`, `archive-reflection-edit`, `archive-reflection-regenerate`, `archive-reflection-overwritewarn`, `archive-reflection-fallback`.
- **사용 서버 능력**: `M13.generateDailyReflection`, `M13.editReflection`, `M13.regenerateReflection`.

### 3.6 TripSummaryScreen + MapHeroView (전체 요약)
- **props**: `tripId` · **state**: 요약(통계·하이라이트·mapHeroSpec·status).
- **폼 검증(서버 규칙 대응)**: 총 방문 수·**총 이동 거리(G72 혼합·추정 표기)**·총 사진 수·날짜별 하이라이트(BR-U7-20). **지도 히어로(방문 순서 핀·날짜별 동선·기준 숙소 마커)** 중심·방문지 사진 미리보기. 동의 구간=GPS 경로·미동의 구간=수동 체크인 연결 경로. **위치 전무 시 방문 목록 대체·생성 실패 시 기본 카드 모음 폴백**(BR-U7-21). 소요시간 없음(D25).
- **상호작용**: `M13.generateTripSummary` · `M13.regenerateTripSummary`(종료 후 편집분 C11) · 지도는 U3 `shared/map`·U5 map 재사용.
- **data-testid**: `archive-summary-maphero`, `archive-summary-stats`, `archive-summary-highlights`, `archive-summary-fallback`.
- **사용 서버 능력**: `M13.generateTripSummary`, `M13.regenerateTripSummary`, `M12.getVisitStats`(G72 합산), (U3)`shared/map`.

### 3.7 RouteComparisonView (계획 vs 실제 경로 — U6/U3/U5 map 재사용)
- **props**: `tripId`, `date` · **state**: 계획 동선·실제 발자취 레이어, 누적 거리·걸음 수, 옵트인/권한 상태.
- **폼 검증(서버 규칙 대응)**: 계획 동선+실제 GPS 발자취 함께/토글, **누적 실제 거리·걸음 수 추정 표기**(걸음 수=거리 환산 G59). **옵트인/권한 없으면 실제 레이어 disabled+사유 표기**(계획 동선만 BR-U7-26). 위치 데이터 전무 시 방문 목록 폴백.
- **상호작용**: `M12.getRouteComparison` · U6 `PlanVsActualMapScreen` 레이어·U3 `shared/map` 재사용.
- **data-testid**: `archive-route-planned`, `archive-route-actual`, `archive-route-stats`, `archive-route-disabled`.
- **사용 서버 능력**: `M12.getRouteComparison`, (U3)`shared/map`.

### 3.8 TravelStyleScreen (스타일 분석)
- **props**: `accountId` · **state**: 분석(metrics·mode·basedOnVisitCount), 게이트 진행.
- **폼 검증(서버 규칙 대응)**: **누적 방문 10곳 게이트**(BR-U7-23). 정식(OFFICIAL)은 카테고리 분포 막대·이동 반경 원·분포점(지도)·구체 수치+근거 방문 데이터. **미달 시 임시 미리보기(정식 아님 명시)+진행 게이지('N/10곳')**. 분류는 취향 7종 축 자체 택소노미. 즉석 자유 입력은 '기타'.
- **상호작용**: `M13.analyzeTravelStyle` → `StyleAnalysis | PendingGate` · `M12.getVisitStats`(10곳 카운트).
- **data-testid**: `archive-style-chart`, `archive-style-preview`, `archive-style-gauge`.
- **사용 서버 능력**: `M13.analyzeTravelStyle`, `M12.getVisitStats`, (U8 연동)`M13.getStyleSummaryCard`.

### 3.9 ShareCardEditor (SNS 공유 카드)
- **props**: `source`(TripSummaryRef | DailyReflectionRef) · **state**: 포맷·캡션·해시태그·레이아웃.
- **폼 검증(서버 규칙 대응)**: 전체 요약·당일 회고 화면 '공유'에서 진입(BR-U7-25). **여행 미종료/요약 미생성이면 진입점 비활성**(INV-SHARE1). 대표 사진·제목·기간·통계(거리만·소요시간 없음 D25)·동선·워터마크 자동 구성. **3포맷(9:16/1:1/4:5)**·캡션·해시태그 편집(금칙어 C3). **사진 0장이면 '사진 없는 카드'(지도·통계)** 대체(INV-SHARE2). 기기 저장·OS 공유 시트(외부 SNS — 커뮤니티와 별개).
- **상호작용**: `M13.buildShareCard(source, format, caption)`.
- **data-testid**: `archive-share-format`, `archive-share-caption`, `archive-share-nophoto`, `archive-share-export`.
- **사용 서버 능력**: `M13.buildShareCard`.

### 3.10 SyncConflictResolver (오프라인 충돌 해소)
- **props**: `tripId` · **state**: 충돌 목록(레코드별 local/server 2버전), 선택.
- **폼 검증(서버 규칙 대응)**: 오프라인 입력과 서버 충돌 시 **레코드 단위 두 버전을 제시해 항목별 선택**(KeepLocal/KeepServer BR-U7-13). **사진·메모 추가는 합집합 자동 병합**(충돌 아님)·상태 필드만 충돌. **어떤 경로도 무손실**(임의 폐기 금지).
- **상호작용**: `M12.syncOfflineRecords` → `SyncResult{applied, conflicts}` · `M12.resolveSyncConflict(resolution)`.
- **data-testid**: `archive-conflict-list`, `archive-conflict-keeplocal`, `archive-conflict-keepserver`.
- **사용 서버 능력**: `M12.syncOfflineRecords`, `M12.resolveSyncConflict`.

---

## 4. shared/storage — 오프라인 입력 로컬 큐 (U7 신규, Δ6)

- **책임**: 여행 중 오프라인 상태의 **기록 입력**(방문 체크·사진 메타·메모·수동 체크인)을 기기 로컬 큐에 보존하고, 재연결 시 배치 동기화한다. 레코드마다 클라 생성 UUID(recordId)+원 발생 시각(recordedAt)+베이스 버전을 부여한다. **오프라인 보장은 '입력' 한정** — 일정 '조회'는 온라인 전제(오프라인 캐시 미제공 D24). **판정 정본은 서버**(충돌 판정·병합은 M12). **저장 구현체·암호화는 NFR/Code Generation 소유**, 본 문서는 계층 계약만 규정한다.
- **props**: 없음(전역 클라이언트 서비스) · **state**: 로컬 큐(레코드·동기화 상태), 사진 업로드 큐, 네트워크 상태.
- **상호작용·폴백**:
  - `OfflineRecordQueue`: 오프라인 입력 → 로컬 큐 적재('동기화 대기' 표시). recordId·recordedAt 부여.
  - `SyncOrchestrator`: 재연결 감지 → `M12.syncOfflineRecords(batch)` 배치 업로드 → per-record 결과(applied/conflict/failed) 반영. 충돌은 `SyncConflictResolver`로 위임(사용자 선택).
  - `PhotoUploadWorker`: 사진 압축(5MB/2048px·EXIF 제거) → 업로드 큐 → 재시도 3회 지수 백오프 → 실패 시 수동 재시도 배지. **부분 실패 격리**(사진 실패가 메모·체크 저장 무영향).
  - 늦은 동기화(recordedAt 과거 임계 초과)는 서버가 M9 트리거 평가 제외(오발화 방지 — 클라는 원 시각 표시).
- **data-testid**: `archive-storage-queue`, `archive-storage-syncstatus`, `archive-storage-photoworker`.
- **사용 서버 능력**: `M12.syncOfflineRecords`, `M12.resolveSyncConflict`, `M12.attachMedia`.

## 5. shared/ 재사용·U7 완성분 (요약)

| 계층 | U7 범위 | 근거 |
|---|---|---|
| shared/map (U3 소유) | **재사용** — 전체 요약 지도 히어로·계획 vs 실제 경로·순서 핀·발자취 레이어(카카오 브리지). U7은 신규 지도 브리지 미생성 | components §6.1, U3 shared/map, FD-U7-10 |
| U5 일정 지도 뷰·U6 PlanVsActualMap | **재사용** — 순서 핀·계획 동선·GPS 발자취 레이어(경로 비교) | U5 MapView·U6 PlanVsActualMapScreen |
| shared/storage | **신규** — 오프라인 입력 로컬 큐·동기화 오케스트레이터·사진 업로드 워커 | Δ6, D24, G74, G75 |
| shared/api | 회고 생성 진행·폴백 고지(침묵 실패 금지)·업로드 큐·동기화 충돌·오프라인 조회 미보장 오류 안내 | ADR-0011, D24 |
| shared/ui | 3계열 라벨·색상·아이콘 구분·'추정' 표기·업로드 상태 배지·전후 diff·공유 카드 레이아웃(3포맷) | D14, D25, PRD 09-4·13 |

## 6. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M12·M13) |
|---|---|---|
| RecordTabScreen + RecordCalendar | US-E8-11, US-E8-14 | M12.listArchivedTrips · getRecordCalendar |
| TripRecordDetailScreen + ThreeSeriesTimeline | US-E8-04·05·11 | M12.getTripTimeline · getTripRecords · (U3)M4.getBaseTimeline |
| VisitRecordEditor + ImpromptuVisitSheet | US-E8-01 | M12.updateVisitRecord · addImpromptuVisit · (U6)M12.checkVisit |
| MediaAttachSheet + UploadQueueStatus | US-E8-02 | M12.attachMedia |
| DailyReflectionCard | US-E8-06·07 | M13.generateDailyReflection · editReflection · regenerateReflection |
| TripSummaryScreen + MapHeroView | US-E8-08 | M13.generateTripSummary · regenerateTripSummary · M12.getVisitStats · (U3)shared/map |
| RouteComparisonView | US-E7-13, US-E8-03·08 | M12.getRouteComparison · (U3)shared/map |
| TravelStyleScreen | US-E8-09·10 | M13.analyzeTravelStyle · M12.getVisitStats · M13.getStyleSummaryCard |
| ShareCardEditor | US-E8-13 | M13.buildShareCard |
| SyncConflictResolver | US-E8-12 | M12.syncOfflineRecords · resolveSyncConflict |
| shared/storage | US-E8-02·12 | M12.syncOfflineRecords · resolveSyncConflict · attachMedia · (U6)M12.appendGpsTrack |
