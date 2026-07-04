# U3 숙소·장소 데이터 — 프런트엔드 컴포넌트 (Frontend Components)

> 2026-07-04 · Functional Design · 클라이언트 feature: `features/stay`(탐색·상세·위시리스트·등록·핸드오프) + `shared/map`(카카오 SDK 브리지)
> 정본 관계: [components.md](../../../inception/application-design/components.md) §6.1 `stay` 행의 화면 목록을 컴포넌트 계층으로 상세화. 서버 능력은 [component-methods.md](../../../inception/application-design/component-methods.md) M3·M4·M5·M7 메서드를 참조하고, 폼 검증 규칙은 [business-rules.md](./business-rules.md) BR-U3-xx의 클라이언트측 대응(UX용 사본 — **판정 정본은 항상 서버**, D28 원칙 준용)이다.
> 기술 중립: 상태 관리 라이브러리·내비게이션 구현체·스타일 시스템 선정은 NFR/Code Generation 소유. 지도 렌더링은 **카카오 지도 SDK 네이티브 모듈(Expo config plugin prebuild)** — 브리지 인터페이스만 본 문서가 규정하고 구현체는 NFR/Infra 소유.
> data-testid 규약: `placestay-{screen}-{role}` (E2E·컴포넌트 테스트 앵커).

## 1. 숙소·장소 화면 플로우 (정본)

```text
[탐색 랜딩 — U2 소유('숙소/장소/여행자 일정' 3진입), U3은 '숙소'·'장소' 백엔드 활성화]
        │
        ▼
┌─ StaySearchScreen (여행지/내 주변 입력 → 결과 목록) ───────────────┐
│   여행지 입력(공백 시 탐색 비활성) · '내 주변'(위치 폴백) · 결과 카드      │
│   · 필터/정렬 시트 진입 · 0건/부분 실패/데이터 부족 안내                 │
└──────────────────────────────────────────────────────────────┘
   │ 카드 탭                    │ 저장                     │ [예약했어요/등록]
   ▼                           ▼                          ▼
StayDetailScreen           WishlistScreen            StayRegisterScreen (3경로)
(정적 콘텐츠·미확인 표기)    (계정 귀속·메모·변동 고지)     (지도검색/링크파싱/핀지정)
   │ '가격 보기'                                            │
   ▼                                                       ▼
LivePriceSheet             OtaSelectSheet → 제휴 고지 → 외부 이동    [AI 일정 생성하기] 온램프(U4 진입)
(날짜·인원 → 라이브)          (내부 ID로 OTA 묶기)
                                   │ 포그라운드 복귀(24h·1회)
                                   ▼
                           ReturnHandoffCard ──탭──▶ StayRegisterScreen(OTA_RETURN 자동 채움)
```

**전역 규칙**
- 숙소 탐색·상세·위시·등록 화면군은 5탭 셸(U2) 내 '탐색' 탭 스택에 쌓인다(탭 규칙 G6·G7은 U2 shared/ui 소유).
- '내 주변' 탐색·등록 지도 조작 직전에만 위치 권한 프리프롬프트(U1 `LocationPrePromptFrame`, purposeContext=NEARBY_SEARCH)를 발화한다 — **위치 just-in-time 발화 1번째 지점** [US-E1-04].
- 모든 서버 오류는 표준 오류 타입으로 정규화되어 침묵 실패 없이 표면화(shared/api) [ADR-0011].
- 리뷰·평점 UI는 존재하지 않는다 — 외부 OTA 딥링크로만 위임(INV-MTX4).

## 2. 컴포넌트 계층

```text
features/stay
├─ StayNavigator                         — 숙소 스택 컨테이너('탐색' 탭 하위)
│  ├─ StaySearchScreen
│  │  ├─ DestinationSearchBar            — 여행지/내 주변 입력(공백 시 탐색 비활성)
│  │  ├─ FilterSortBar                   — 필터·정렬 진입(대표 가격대순·거리)
│  │  ├─ StayResultList                  — 결과 카드(숙소명·위치·대표 가격대) · 페이지네이션/무한스크롤
│  │  │  └─ StayResultCard               — 카드 단위(저장·'가격 보기'·[예약했어요/등록])
│  │  ├─ SearchLoadingState              — "숙소를 모으는 중"
│  │  ├─ DegradedBanner                  — "일부 정보가 빠졌을 수 있어요"(부분 실패)
│  │  └─ ZeroResultGuide                 — 0건·데이터 부족 지역 안내(직접 등록 온램프)
│  ├─ FilterSortSheet
│  │  ├─ StayTypeFilter                  — 유형 다중 선택(전면 제공)
│  │  ├─ AmenityFilter                   — 편의시설(채움률 검증 항목만)
│  │  ├─ PriceBandFilter                 — 대표 가격대 구간
│  │  ├─ DistanceFilter                  — 여행지 중심 직선거리(소요시간 미표시)
│  │  └─ FilterZeroDiagnosis             — 0건 원인 필터·완화 제안
│  ├─ StayDetailScreen
│  │  ├─ StayMapPin                      — 위치 핀(shared/map 브리지)
│  │  ├─ StaticContentSection            — 사진·가격대·편의시설·체크인/아웃 시각('미확인' 표기)
│  │  ├─ ExternalReviewEntry             — "리뷰·평점 보기 → 외부 OTA" 딥링크 위임
│  │  ├─ LivePriceButton                 — '가격 보기' → LivePriceSheet
│  │  └─ OtaBookButton                   — [외부 OTA에서 예약하기] → OtaSelectSheet
│  ├─ LivePriceSheet                     — 날짜·인원 입력 → 라이브 가격(세션 재사용·비캐싱)
│  ├─ WishlistScreen
│  │  ├─ WishStayCard                    — 숙소명·사진·가격대·위치·"변동 가능" 병기
│  │  ├─ WishMemoEditor                  — 메모 추가/수정/삭제
│  │  └─ StaleBadge                      — "최신 정보 확인 불가"(소실)
│  ├─ OtaSelectSheet
│  │  ├─ OtaOptionRow                    — OTA명 목록(내부 ID 묶기)
│  │  └─ AffiliateDisclosureBanner       — 제휴 수수료 고지('다시 보지 않기' 연동)
│  ├─ ReturnHandoffCard                  — 복귀 핸드오프(24h·1회·무시 억제)
│  ├─ StayRegisterScreen                 — 등록 단일 폼(3경로 수렴)
│  │  ├─ RegisterPathTabs                — 지도검색 / 링크붙여넣기 / 핀지정
│  │  ├─ MapSearchPicker                 — 장소 검색·동명 다건 후보 선택(핀)
│  │  ├─ OtaLinkPasteField               — URL 붙여넣기(화이트리스트 파싱·프리필)
│  │  ├─ MapPinDropField                 — 지도 핀 직접 지정(coordConfirmed)
│  │  ├─ StayDateRangePicker             — 체크인/체크아웃(순서 검증)
│  │  ├─ PartyField / OptionalFields     — 인원(필수)·예약번호·금액(선택)
│  │  └─ CoordUnresolvedGuard            — "지도에서 위치 확인" 강제 배너
│  ├─ MyStayListScreen                   — 저장/등록 통합 목록(출처 라벨·거점 사용 여부·"미입력")
│  └─ SavedPlaceListScreen               — 장소(POI) 저장 목록(Case A 온램프 — '확인 불가' 배지)
└─ shared/map
   ├─ KakaoMapBridge                     — 카카오 지도 SDK 네이티브 브리지(config plugin — 구현체 NFR/Infra)
   ├─ MapView                            — 지도 표시·핀·현위치(effectiveCapabilities 질의)
   └─ PinDropController                  — 핀 드래그·좌표 확정
```

---

## 3. 탐색 화면군

### 3.1 StaySearchScreen
- **props**: 없음(라우트 루트) · **state**: 여행지 입력값, 검색 중 플래그, 결과 페이지, degraded 소스 목록, 필터 상태.
- **폼 검증(서버 규칙 대응)**: 여행지 공백 ⇔ 탐색 버튼 비활성(BR-U3-01). '내 주변' 선택 시 위치 권한 프리프롬프트(U1 프레임) 후 `effectiveCapabilities` 질의 — 거부 시 등록 숙소/여행지 중심 좌표 폴백(BR-U3-05).
- **상호작용**: 탐색 → `M3.searchStays` [FLOW-1]. 부분 실패 → `DegradedBanner`, 전체 실패 → 재시도+직접 등록 우회, 0건/데이터 부족 → `ZeroResultGuide`(직접 등록 온램프). 카드 탭 → StayDetailScreen, 저장 → `M3.addToWishlist`, [예약했어요/등록] → StayRegisterScreen(자동 채움).
- **data-testid**: `placestay-search-bar`, `placestay-search-resultcard`, `placestay-search-zeroguide`, `placestay-search-degraded`.
- **사용 서버 능력**: `M3.searchStays`, `M3.addToWishlist`.

### 3.2 FilterSortSheet
- **props**: `currentFilters` · **state**: 유형·편의시설·가격대·거리 선택값, 정렬 기준.
- **폼 검증(서버 규칙 대응)**: 정렬 옵션에 **가격순 없음** — '대표 가격대순'만(BR-U3-08). 거리 필터는 여행지 중심 직선거리, **소요시간 표기 없음**(BR-U3-09, D25). 편의시설은 채움률 검증 항목만 렌더(BR-U3-07). 0건 시 `FilterZeroDiagnosis`가 원인 필터·완화 제안(BR-U3-36).
- **data-testid**: `placestay-filter-type`, `placestay-filter-amenity`, `placestay-filter-priceband`, `placestay-filter-distance`, `placestay-filter-zerodiagnosis`.
- **사용 서버 능력**: `M3.searchStays`(필터 인자).

### 3.3 StayDetailScreen
- **props**: `stayId: CanonicalStayId` · **state**: 정적 상세 로드 상태.
- **폼 검증(서버 규칙 대응)**: 리뷰·평점 UI 없음 — `ExternalReviewEntry`로 외부 OTA 위임(BR-U3-23, INV-MTX4). 누락 필드는 "미확인" 명시값(빈칸 금지). `StayMapPin`은 shared/map 브리지로 위치 표시.
- **상호작용**: `M3.getStayDetail`. '가격 보기' → `LivePriceSheet`, [외부 OTA에서 예약하기] → `OtaSelectSheet`(FLOW-3).
- **data-testid**: `placestay-detail-mappin`, `placestay-detail-unknown`(미확인 필드), `placestay-detail-externalreview`, `placestay-detail-livepricebtn`.
- **사용 서버 능력**: `M3.getStayDetail`, `M7.getPoi`(장소 상세 공용).

### 3.4 LivePriceSheet
- **props**: `stayId` · **state**: 날짜·인원 입력(세션 재사용), 라이브 조회 상태.
- **상호작용**: 날짜·인원 입력 → `M3.getLivePrice`(라이브·비캐싱, BR-U3-04). `UpstreamUnavailable` → "외부 OTA에서 확인" 딥링크 대체.
- **data-testid**: `placestay-liveprice-daterange`, `placestay-liveprice-party`, `placestay-liveprice-result`.
- **사용 서버 능력**: `M3.getLivePrice`.

---

## 4. 위시리스트 화면

### 4.1 WishlistScreen
- **props**: 없음 · **state**: 위시 목록 페이지, 메모 편집 상태.
- **폼 검증(서버 규칙 대응)**: 카드는 숙소명·사진·가격대·위치+**"외부 OTA 기준, 변동 가능"** 병기(BR-U3-30, INV-WISH2). 소실 항목은 `StaleBadge`. 메모는 SECURITY-05 검증(클라이언트 사본).
- **상호작용**: `M3.listWishlist`. 메모 → `M3.updateWishlistMemo`, 삭제 → `M3.removeFromWishlist`. 카드 [등록하기] → StayRegisterScreen 자동 채움.
- **data-testid**: `placestay-wishlist-card`, `placestay-wishlist-memo`, `placestay-wishlist-stale`, `placestay-wishlist-variablenotice`.
- **사용 서버 능력**: `M3.listWishlist`, `M3.updateWishlistMemo`, `M3.removeFromWishlist`.

---

## 5. OTA 딥링크·복귀 핸드오프 화면군

### 5.1 OtaSelectSheet
- **props**: `stayId` · **state**: OTA 옵션 목록, 고지 숨김 설정.
- **상호작용**: `M5.getOtaLinkOptions`(내부 ID 기준 묶기 BR-U3-20) → `OtaOptionRow` 목록. 이동 직전 `AffiliateDisclosureBanner` 표시(BR-U3-22, showDisclosure 반영) → `M5.buildDeeplink` → 외부 이동 + `M5.recordOutboundClick`(best-effort). `LinkUnavailable` → 다른 OTA/장소 검색 우회(BR-U3-24).
- **data-testid**: `placestay-otaselect-option`, `placestay-otaselect-disclosure`, `placestay-otaselect-unavailable`.
- **사용 서버 능력**: `M5.getOtaLinkOptions`, `M5.buildDeeplink`, `M5.recordOutboundClick`, `M5.setDisclosureHidden`.

### 5.2 ReturnHandoffCard
- **props**: 없음(포그라운드 복귀 트리거) · **state**: 카드 표시 여부.
- **상호작용**: 포그라운드 복귀 시 `M5.getReturnHandoffCard` — 이탈 후 24h 내 첫 복귀에 최근 1건 1회 노출(BR-U3-27). 탭 → StayRegisterScreen(OTA_RETURN 자동 채움). 무시 → `M5.dismissHandoffCard`(재노출 억제 BR-U3-28) — 계속 탐색(비강제).
- **data-testid**: `placestay-handoff-card`, `placestay-handoff-register`, `placestay-handoff-dismiss`.
- **사용 서버 능력**: `M5.getReturnHandoffCard`, `M5.dismissHandoffCard`.

---

## 6. 등록 화면군

### 6.1 StayRegisterScreen (등록 3경로 단일 폼 — US-E3-08 정본)
- **props**: `prefill?: StayPrefill`(검색·핸드오프·링크 파싱 결과) · **state**: 선택 경로(RegisterPathTabs), 폼 값(숙소·날짜·인원·선택 항목), coordConfirmed, 필드별 인라인 오류.
- **폼 검증(서버 규칙 대응)**:
  - 필수: 숙소·체크인/체크아웃·인원. `StayDateRangePicker`는 체크아웃>체크인 즉시 인라인(BR-U3-10 사본 — 정본은 서버 `ValidationFailed`).
  - `CoordUnresolvedGuard`: coordConfirmed=false면 "지도에서 위치 확인" 강제 배너, 확정 전 저장 차단(BR-U3-11, INV-STAY2).
  - 선택: 예약번호·금액(미입력 허용).
- **경로별 상호작용**:
  - MAP_SEARCH: `MapSearchPicker` → `M7.searchPoi` → 동명 다건 시 후보+핀 선택(좌표 확정).
  - LINK_PARSE: `OtaLinkPasteField` → `M4.parseOtaLink`(화이트리스트 파싱·fetch 없음 BR-U3-25) → 프리필. 실패 → 핀 지정 폴백(BR-U3-26).
  - PIN_DROP: `MapPinDropField`(shared/map `PinDropController`) → 좌표 확정.
- **제출**: `M4.registerStay` [FLOW-2] → 성공 시 [AI 일정 생성하기] 온램프(U4 진입) 노출(BR-U3-14).
- **data-testid**: `placestay-register-pathtabs`, `placestay-register-mapsearch`, `placestay-register-linkpaste`, `placestay-register-pindrop`, `placestay-register-daterange`, `placestay-register-coordguard`, `placestay-register-submit`.
- **사용 서버 능력**: `M4.registerStay`, `M4.parseOtaLink`, `M7.searchPoi`.

### 6.2 MyStayListScreen (등록·저장 통합 목록)
- **props**: 없음 · **state**: 통합 목록(출처 라벨), 거점 사용 여부.
- **폼 검증(서버 규칙 대응)**: 저장(위시)/등록 통합 — 출처 라벨('외부 OTA 예약 등록'/'앱 내 저장'), 연결 여행·거점 사용 여부 표시. 선택 항목(예약번호·금액) 미입력은 **"미입력"** 명시값(빈칸 금지 BR-U3-30). 저장 항목엔 '등록하기' 액션.
- **상호작용**: `M4.listMyStays`. 수정 → `M4.updateStay`, 취소 → `M4.deleteStay`(연결 시 차단형 확인 G39).
- **data-testid**: `placestay-mystaylist-item`, `placestay-mystaylist-sourcelabel`, `placestay-mystaylist-basebadge`, `placestay-mystaylist-notentered`.
- **사용 서버 능력**: `M4.listMyStays`, `M4.updateStay`, `M4.deleteStay`.

### 6.3 SavedPlaceListScreen (장소 POI 저장 — Case A 온램프)
- **props**: 없음 · **state**: 저장 POI 목록.
- **상호작용**: `M7.listSavedPois`(이름·카테고리·지역, 원본 소실 '확인 불가' 배지 G8). 저장/해제 → `M7.savePoi`/`M7.unsavePoi`. '이 장소들로 여행 만들기' 진입(U2 온램프 셸 백엔드 활성화 — 필수 방문지 투입은 U4).
- **data-testid**: `placestay-savedplace-item`, `placestay-savedplace-lostbadge`.
- **사용 서버 능력**: `M7.listSavedPois`, `M7.savePoi`, `M7.unsavePoi`.

---

## 7. shared/map — 카카오 지도 SDK 브리지 (U3 신규)

- **책임**: 카카오 지도 SDK 네이티브 모듈의 **재사용 브리지** — 지도 표시·핀·현위치·핀 드래그 좌표 확정. **구현체(Expo config plugin·네이티브 모듈)는 NFR/Infrastructure Design 소유**, 본 문서는 브리지 인터페이스와 화면 계약만 규정한다.
- **props**: `KakaoMapBridge{ center: Coord, pins: List<Pin>, onPinDrop?, showMyLocation?: Boolean }` · **state**: 지도 준비 상태, 현위치 가용성(U1 `effectiveCapabilities` 질의 — 서버 위치 서비스=L1∧L2).
- **상호작용·폴백**:
  - `MapView`: 위치 핀 표시(StayDetailScreen), 현위치는 권한 허용 시에만(거부 시 여행지 중심 좌표 폴백).
  - `PinDropController`: 등록 핀 지정(coordConfirmed 확정) — MapSearchPicker 동명 다건 후보의 핀 선택 공용.
  - **SDK 네이티브 모듈 불안정 폴백**: 지도 렌더 실패 시 좌표·주소 텍스트 폴백(핀 지정은 좌표 입력 폴백) — 침묵 실패 금지(ADR-0011, U3 리스크 완화 "지도 표시 범위 축소 폴백").
- **data-testid**: `placestay-map-view`, `placestay-map-pindrop`, `placestay-map-mylocation`.
- **사용 서버 능력**: `M7.searchPoi`, `M7.getPoi`(지오코딩·POI 좌표), U1 `getLocationConsentState`(권한 상태).

---

## 8. shared/ U3 완성분 (요약)

| 계층 | U3 완성 범위 | 근거 |
|---|---|---|
| shared/map | 카카오 지도 SDK 브리지 신규(지도·핀·현위치·핀지정) — 구현체는 NFR/Infra | components §6.2, U3 리스크(config plugin) |
| shared/api | 탐색 부분 실패 degraded 합성 소비·라이브 가격 비캐싱 경로·아웃바운드 링크 이동 | ADR-0011, D09 |
| shared/ui | 숙소 카드·필터 시트·통합 목록 라벨·'미확인'/'미입력' 명시값 컴포넌트 | PRD 04-3·9 |
| shared/location | '내 주변' just-in-time 발화 1번째 지점(U1 프레임 purposeContext=NEARBY_SEARCH) | US-E1-04, G182 |

## 9. 화면-스토리-서버 능력 추적 매트릭스

| 컴포넌트 | 스토리 | 서버 능력(M3·M4·M5·M7) |
|---|---|---|
| StaySearchScreen | US-E3-01, US-E3-10, US-E3-11 | M3.searchStays · M3.addToWishlist |
| FilterSortSheet | US-E3-02 | M3.searchStays(필터) |
| StayDetailScreen | US-E3-03 | M3.getStayDetail · M7.getPoi |
| LivePriceSheet | US-E3-01 | M3.getLivePrice |
| WishlistScreen | US-E3-04 | M3.listWishlist · updateWishlistMemo · removeFromWishlist |
| OtaSelectSheet | US-E3-05 | M5.getOtaLinkOptions · buildDeeplink · recordOutboundClick · setDisclosureHidden |
| ReturnHandoffCard | US-E3-05 | M5.getReturnHandoffCard · dismissHandoffCard |
| StayRegisterScreen | US-E3-06, US-E3-08 | M4.registerStay · parseOtaLink · M7.searchPoi |
| MyStayListScreen | US-E3-09 | M4.listMyStays · updateStay · deleteStay |
| SavedPlaceListScreen | US-E2-05(백엔드 활성화) | M7.listSavedPois · savePoi · unsavePoi |
| (거점 연결 — 여행 UI는 U4) | US-E3-07 | M4.linkToTrip · getBaseTimeline · resolveStayIdentity |
| shared/map (지도 뷰) | US-E3-03·08 | M7.searchPoi · getPoi · (U1)getLocationConsentState |
