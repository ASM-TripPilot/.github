# U3 숙소·장소 데이터 — 도메인 엔티티 (Domain Entities)

> 2026-07-04 · Functional Design · 소유 모듈: M3 Accommodation Search, M4 Saved Accommodation, M5 Affiliate Link, M7 Place Data
> 정본 관계: [components.md](../../../inception/application-design/components.md) M3·M4·M5·M7의 엔티티 개요를 인터페이스 계약 수준으로 상세화한다. 본 문서가 U3 엔티티·불변식의 정본이다. 기술 중립 — 저장 기술·컬럼 타입·인덱스·캐시 구현체는 NFR/Infrastructure Design 소유.
> 표기: `필수`=NULL 불가, `선택`=NULL 허용(NULL의 의미를 반드시 명기), `유니크`=제약 범위 내 유일. 근거 ID는 requirements.md(D·G·ADR)·stories.md(US-E3)·본 유닛 규칙([business-rules.md](./business-rules.md) BR-U3-xx)을 참조한다.
> **P2(지도 API 약관·D13 스냅샷 적법성)·P4(TourAPI 캐싱 조건)가 캐싱 허용/금지 필드의 법적 경계를 확정한다** — 본 설계는 소스별 캐싱 매트릭스(§10)를 데이터 모델에 명문화해 정책 변경을 컬럼 수준으로 흡수한다(ADR-0017).

## 0. 엔티티 지도

```text
[M7 Place Data — POI 정본]
Poi 1 ──── * PoiSourceRef            (소스별 place_id N:1 매핑, G133/G148)
Poi 1 ──── * PoiSnapshot             (사용자 확정 시점 사본 — 영구, D13)
SavedPlace * ──── 1 PoiSnapshot      (계정 저장 POI = 스냅샷 참조, Case A 온램프)
(독립) TrendingAggregate             (인기 장소 일 1회 배치, G2)
(독립) StayTimeTable                 (카테고리별 체류 기본값, G51)

[M3 Accommodation Search — 탐색·위시리스트]
StaySearchCache (stayRef) ── TTL 캐시(정적 콘텐츠 — 영구 금지, D13)
Account 1 ──── * WishlistItem        (계정 귀속 위시, staleFlag)
WishlistItem * ──── 1 StayIdentity   (숙소 참조 — 내부 ID 또는 외부 ID 매핑)

[M4 Saved Accommodation — 등록 숙소·거점]
Account 1 ──── * SavedStay           (계정 레벨 풀, D15)
SavedStay * ──── 1 StayIdentity      (내부 숙소 ID 귀속)
StayIdentity 1 ──── * StayIdentityMapping  (소스별 외부 ID N:1, D17)
SavedStay 1 ──── * BaseAssignment    (여행 거점 연결 — 데이터·API는 U3, 여행 내 비중첩 검증은 U4 소비 CP1)

[M5 Affiliate Link — 딥링크·클릭]
Account 1 ──── * AffiliateClick      (아웃바운드 클릭 집계 — 사용자·LLM 비노출)
(독립) OtaPartner                    (파트너별 URL 템플릿·약관 최소치)
(예약) PostbackRecord                (스키마만 — 1차 미사용, G29)
```

---

## 1. Poi — 장소 정본 (M7)

관광지·식당·카페·숙소 장소의 canonical 정본. **canonical POI ID가 전 유닛 장소 참조의 단일 키**이며, 소스별 place_id는 이 아래에 N:1로 매달린다 [G133/G148, D17 동일 패턴]. U5 POI 그라운딩 하드 제약(출력 POI ⊆ 후보 풀)의 전제 무결성을 소유한다.

### 1.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| canonicalPoiId | 식별자 | 필수 · 유니크 · 불변 | **정본 키** — 동일 실세계 장소는 정확히 1개(§1.3 INV-POI1). 앱 내부 생성 ID(외부 place_id와 독립) — 영구 저장 허용(식별자는 캐싱 금지 대상 아님, PRD 04 註 ③) [G133] |
| nameKo | 문자열 | 필수 | **한국어명 정본**. API 응답의 한국어 필드 우선, 없으면 원문 노출 [G123/G148] |
| aliasEn | 문자열 | 선택 | 영문 alias(대조·검색 보조). NULL=영문 미확보 [G148] |
| coord | 구조체 {lat, lng} | 선택 | 좌표(위도·경도). NULL=지오코딩 미확보(=`UNVERIFIED`/`LOST` 사유). 소스별 캐싱 정책은 §10 — 지도 API 유래 좌표는 영구 저장 아닌 TTL 캐시, TourAPI 유래는 정본 저장 [D08, D13, G124] |
| category | 열거(표준 카테고리) | 필수 · 기본 `UNKNOWN` | 소스별 카테고리 코드를 단일 표준 스키마로 정규화(ADR-0009). 숙소 유형(호텔·게스트하우스·펜션·모텔·호스텔·아파트·B&B·한옥 등)의 한국형 분류는 TourAPI cat3 정본 [PRD 04-2] |
| openingHours | 구조체(요일별 구간) | 선택 | 영업시간. NULL=미확인 → dataStatus `UNVERIFIED`(빈칸 금지·"미확인" 표기의 원천) [PRD 04-3 예외, G192] |
| stayRange | 구조체 {min, rec, max} | 선택 | 카테고리 체류 기본값의 조인 참조(정본은 StayTimeTable §12). NULL=테이블 미매핑 [G51] |
| dataStatus | 열거 {ACTIVE, UNVERIFIED, LOST, CLOSED} | 필수 · 기본 `UNVERIFIED` | POI 생애 상태 — `ACTIVE`(정상)·`UNVERIFIED`(영업시간·좌표 미확인)·`LOST`(정본 동기화에서 확인 불가 → '확인 불가' 배지·시드 투입 제외, G8)·`CLOSED`(폐업 확인). §1.2 상태 규칙 |
| sourceAttribution | 문자열 | 필수 | 출처 표기(약관 요건 — 지도 API 응답 표기 의무) [ADR-0017] |

> **PoiSourceRef**(Poi 서브엔티티): `{source: 열거(KAKAO, NAVER, TOURAPI, OTA), placeId: 문자열}` — 소스별 외부 식별자. `(source, placeId)`는 canonicalPoiId에 N:1로 귀속. 식별자 자체는 영구 저장 허용.

### 1.2 dataStatus 상태 규칙 (정본)

```text
   [수집·정규화]
       │
       ▼
   UNVERIFIED ──(좌표·영업시간 확보)──▶ ACTIVE
       │                                  │
       │                                  │ (정본 동기화 배치 — 소스에서 확인 불가)
       │◀─(재조회 성공)────────────────────┤
       ▼                                  ▼
     LOST ◀──(동기화 확인 불가)──────────── LOST
       │
       │ (폐업 확인 신호)
       ▼
    CLOSED
```

- 확정 배치는 `UNVERIFIED`를 자동 확정하지 않는다 — 사용자 확인 후보로 분리 판정 데이터만 공급(M7이 소비자 M8·M9에 상태 전파) [PRD 06-3 예외].
- `LOST`·`CLOSED` POI는 후보 풀·필수 방문지 시드 투입에서 제외(G8) — 단 이미 저장된 PoiSnapshot(사용자 확정 사본)은 마지막 스냅샷으로 잔존 표시(허위 최신성 금지).

### 1.3 불변식 (Invariants)

- **INV-POI1 (canonical 유일성 — 하드 제약)**: 동일 실세계 장소(좌표 50m 근접 ∧ 명칭 유사도 임계 이상)는 정확히 1개 canonicalPoiId에 매핑된다 — 소스별 place_id는 N:1. 이 무결성은 D37 하드 제약 '숙소 기준점·POI 그라운딩' 계열의 전제이며 머지 차단 테스트 대상 [G133/G148, U3 DoD, PBT U3-P1].
- **INV-POI2 (매핑 단조·비파괴)**: PoiSourceRef 추가로 canonicalPoiId는 변경되지 않는다(같은 입력 재수집 시 canonical 불변 — 멱등·대칭) [PBT U3-P1].
- **INV-POI3 (미확인 명시)**: coord·openingHours가 NULL이면 dataStatus는 `ACTIVE`일 수 없다(`UNVERIFIED` 이상) — "미확인"은 빈칸이 아니라 명시 상태값으로 전파 [PRD 04-3 예외, G199].
- **INV-POI4 (한국어명 존재)**: nameKo는 항상 값을 가진다(한국어 필드 없으면 원문으로 채움) — 표시 공백 금지 [G123].
- **INV-POI5 (캐싱 게이트)**: 지도 API(KAKAO·NAVER) 유래 좌표·영업시간·명칭 필드는 §10 매트릭스의 '영구 금지' 셀에 해당하면 TTL 캐시로만 보관하며 만료 후 재조회를 강제한다. 정본 영구 저장은 캐싱 허용 소스(TourAPI) 또는 사용자 확정 스냅샷(§2)에 한한다 [D13, ADR-0017, G193, PBT U3-P3].

---

## 2. PoiSnapshot — 사용자 확정 스냅샷 (M7, 영구)

사용자가 **확정한** 데이터(등록 숙소·방문 체크·필수 방문지·저장 POI)의 확정 시점 사본. '사용자 입력 데이터'로 취급해 **영구 저장**하며(D13, P2 법무 확인 전제), 원본 Poi가 `LOST`/`CLOSED`가 되어도 잔존한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| snapshotId | 식별자 | 필수 · 유니크 · 불변 | — |
| canonicalPoiId | 식별자(Poi 참조) | 필수 | 원본 참조(라이브 대조용) — 원본 소실 시에도 스냅샷은 독립 잔존 |
| purpose | 열거 {SAVED_PLACE, MUST_VISIT, VISIT_CHECK, REGISTERED_STAY} | 필수 | 확정 맥락. MUST_VISIT·VISIT_CHECK는 U4·U6 소비(U3은 SAVED_PLACE·REGISTERED_STAY 생산이 주) [D13] |
| nameKo | 문자열 | 필수 | 확정 시점 명칭 사본(영구) |
| coord | 구조체 {lat, lng} | 선택 | 확정 시점 좌표 사본 — **사용자 입력 데이터로 영구 저장 허용**(지도 API 실시간 캐싱 금지의 예외 축, D13) [G124, G70] |
| category | 열거 | 필수 | 확정 시점 카테고리 사본 |
| openingHours | 구조체 | 선택 | 확정 시점 영업시간 사본. NULL=확정 시점 미확인 |
| sourceAttribution | 문자열 | 필수 | 확정 시점 출처 표기(약관 요건) |
| snapshotAt | 시각(UTC) | 필수 · 불변 | 확정 시각 |

**불변식**
- **INV-SNAP1 (불변)**: 스냅샷은 생성 후 수정되지 않는다 — 원본 갱신이 스냅샷에 역류하지 않는다(대조를 위해 시점 고정). 재확정은 새 스냅샷 생성 [D13, G129 사본 원칙과 동형].
- **INV-SNAP2 (소실 독립)**: 원본 Poi.dataStatus=`LOST`여도 스냅샷은 파기되지 않고 마지막 사본으로 표시된다(허위 최신성 금지·확인 불가 배지 병기) [G8, G70].
- **INV-SNAP3 (영구 게이트)**: 영구 저장은 purpose가 사용자 확정 4종에 한하며, 탐색·추천 풀 데이터(StaySearchCache·후보 풀)는 스냅샷화 대상이 아니다(TTL 캐시로만 존재) [D13, §10].

---

## 3. SavedPlace — 계정 저장 POI (M7, Case A 온램프)

'장소 우선' 진입(US-E2-05)에서 사용자가 저장한 POI. 상한 없음 [G8]. 저장 실체는 PoiSnapshot(purpose=SAVED_PLACE) 참조이며, CP1의 '저장 POI' 계약 공급원이다(필수 방문지 시드 — 여행 투입 시 **사본 복제**는 U4 M6 소유 G129).

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| savedPlaceId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account FK) | 필수 | 계정 귀속 — 기기 전환 유지 [D22] |
| snapshotRef | 식별자(PoiSnapshot 참조) | 필수 | 저장 시점 스냅샷(영구) [D13] |
| savedAt | 시각(UTC) | 필수 | 저장 일시 |
| staleFlag | 불리언(파생) | 필수 · 기본 false | 원본 Poi가 `LOST`면 true → '확인 불가' 배지, 필수 방문지 시드 투입 제외 신호 [G8] |

**불변식**
- **INV-SP1 (계정 유니크)**: `(accountId, canonicalPoiId)`는 유니크 — 동일 계정의 동일 장소 중복 저장 금지(재저장은 멱등).
- **INV-SP2 (소실 표기)**: staleFlag=true 항목은 목록에 '확인 불가' 배지로 표시되되 삭제되지 않는다(사용자 삭제만 파기) [G8].

---

## 4. StaySearchCache — 숙소 탐색 정적 콘텐츠 TTL 캐시 (M3)

숙소 탐색 결과 카드·상세의 **정적 콘텐츠**(명칭·위치·유형·편의시설·대표 가격대·사진 URL·체크인/아웃 시각) TTL 캐시. **정확 가격·재고는 캐싱하지 않으며**(OTA 약관), 라이브 호출은 사용자가 '가격 보기'/딥링크 시에만 발생한다 [D09, D13, G196].

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| stayRef | 식별자(StayIdentity 참조) | 필수 | 내부 숙소 ID 기준 캐시 키 |
| nameKo | 문자열 | 필수 | 숙소명(한국어 정본) |
| coord | 구조체 {lat, lng} | 선택 | 위치(지도 핀) — §10 매트릭스: 지도 API 유래는 TTL, TourAPI 유래는 정본 |
| stayType | 열거 | 선택 | 숙소 유형(TourAPI cat3 정본) |
| amenities | 목록<열거> | 선택 | 편의시설 — **채움률 검증된 항목만**(핵심 3종 주차·조식·와이파이부터 확대) [PRD 04-2] |
| priceBand | 열거 {저가, 중간, 고급, 럭셔리} | 선택 | **대표 가격대(정적 분류)** — 콘텐츠 메타로 정적 표시·캐싱 가능(정확 1박 요금이 아니므로 가격 캐싱 금지 대상 아님) [D09, PRD 04 註 ②] |
| photos | 목록<URL> | 선택 | 사진 URL(정적 콘텐츠 — 약관 갱신주기 준수 캐싱) |
| checkInOutTime | 구조체 {checkIn, checkOut} | 선택 | 체크인/아웃 **시각**(정적 콘텐츠, 날짜 아님) |
| fetchedAt | 시각(UTC) | 필수 | 수집 시각 |
| ttlExpiresAt | 시각(UTC) | 필수 | `= fetchedAt + 약관 허용 TTL` — 만료 후 재조회 강제(영구 저장 금지) [D13, ADR-0017] |

**불변식**
- **INV-CACHE1 (영구 금지)**: StaySearchCache에는 정본 개념이 없다 — 모든 행은 ttlExpiresAt를 가지며 만료분은 무효로 취급(재조회 없이 최신으로 표시 금지) [D13, ADR-0017, PBT U3-P3].
- **INV-CACHE2 (가격 비보관)**: 정확 1박 가격·재고는 어떤 필드에도 저장되지 않는다 — priceBand(정적 구간)만 허용, 정확 가격은 라이브 결과로만 존재하고 캐싱 0 [D09, PRD 04 註 ②, PBT U3-P3].
- **INV-CACHE3 (갱신 실패 표기)**: TTL 만료 후 재조회 실패 시 stale 데이터에 '갱신 실패' 표기를 병기한다(허위 최신성 금지) [ADR-0011, M7 실패 책임].

---

## 5. StayIdentity + StayIdentityMapping — 내부 숙소 ID·소스 매핑 (M4)

숙소 식별 통합. **내부 canonical 숙소 ID**가 정본이며, 소스별 외부 ID(OTA·지도)는 N:1로 매달린다(POI와 동일 패턴, D17). 동일 숙소에 연결된 복수 OTA를 묶는 기반(US-E3-05 OTA 선택 목록).

### 5.1 StayIdentity 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| canonicalStayId | 식별자 | 필수 · 유니크 · 불변 | **내부 숙소 ID 정본 키** — SavedStay·WishlistItem·StaySearchCache가 참조 [D17] |
| nameKo | 문자열 | 필수 | 숙소명(한국어 정본) |
| coord | 구조체 {lat, lng} | 선택 | 대표 좌표(매칭 기준). NULL=핀 미확정(등록 시 SavedStay.coordConfirmed로 별도 관리) |
| matchConfidence | 수치 [0,1] | 필수 | 좌표+이름 유사도 매칭 신뢰도 |
| operatorOverride | 불리언 | 필수 · 기본 false | 운영자 보정 여부(자동 매칭 오류 수동 교정 여지) [D17] |

### 5.2 StayIdentityMapping 속성 (StayIdentity 서브엔티티)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| canonicalStayId | 식별자(StayIdentity FK) | 필수 | 귀속 내부 ID |
| source | 열거 {KAKAO, NAVER, TOURAPI, OTA:{partnerCode}} | 필수 | 외부 소스 |
| externalId | 문자열 | 필수 | 소스별 외부 숙소 ID(place_id 등) |

**불변식**
- **INV-SID1 (N:1 라운드트립 — 하드 제약)**: `(source, externalId)`는 정확히 1개 canonicalStayId로 해석된다 — 외부 ID → 내부 ID는 전역·결정적(N:1). 라운드트립(외부 ID 해석 후 재해석 동일) 보장 [D17, U3 DoD, PBT U3-P2].
- **INV-SID2 (복합 유니크)**: `(source, externalId)` 조합은 전역 유일 — 하나의 외부 ID가 두 내부 ID에 매핑될 수 없다.
- **INV-SID3 (매칭 결정성·대칭성)**: 좌표(50m 근접)+명칭 유사도 매칭은 동일 입력에 동일 결과(결정적), 대칭(A가 B에 매칭 ⇔ B가 A에 매칭) — 운영자 보정은 자동 결과를 덮어쓰되 보정 자체도 결정적 [D17, PBT U3-P4].

---

## 6. SavedStay — 등록 숙소 (M4, 계정 레벨 풀)

사용자가 외부에서 예약했거나 직접 찾은 숙소를 '등록'한 **계정 레벨 풀** 항목. 여행 없이 등록 가능하며(D15), 여행 연결(BaseAssignment §7) 시 AI 일정 생성의 거점·기간 기준점이 된다(ADR-0002). 예약 상태는 추적하지 않고 사용자 입력 기록만 보관한다.

### 6.1 속성

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| savedStayId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account FK) | 필수 | **계정 레벨 풀 귀속** — 여행 없이 등록·유지 [D15] |
| stayIdentityRef | 식별자(StayIdentity 참조) | 필수 | 내부 숙소 ID [D17] |
| placeSnapshot | 구조체 {nameKo, address, coord} | 필수 | 등록 시점 장소 사본(이름·주소·좌표 — 확정 시점 동결, PoiSnapshot 원칙) [D13] |
| checkIn | 날짜 | 필수 | 체크인 날짜(§6.2 검증) |
| checkOut | 날짜 | 필수 | 체크아웃 날짜 — `checkOut > checkIn`(§6.3 INV-STAY1) [PRD 04-6 예외] |
| party | 정수(양수) | 필수 | 인원 |
| coordConfirmed | 불리언 | 필수 · 기본 false | 좌표 확정 여부 — false면 "지도에서 위치 확인" 강제, 확정 전 거점 자격 없음(§6.3 INV-STAY2) [PRD 04-6·8 예외] |
| registrationPath | 열거 {OTA_RETURN, MAP_SEARCH, LINK_PARSE, PIN_DROP} | 필수 | 등록 경로 — OTA 복귀 핸드오프·지도 검색·링크 파싱·핀 지정 [PRD 04-6·8, G31] |
| otaName | 문자열 | 선택 | OTA명(선택 입력). NULL=미입력 → "미입력" 표기(빈칸 금지) [PRD 04-9 예외] |
| reservationNo | 문자열 | 선택 | 예약번호(선택). NULL=미입력 |
| amount | 구조체(금액) | 선택 | 결제 금액(선택). NULL=미입력 |
| memo | 문자열 | 선택 | 메모 — SECURITY-05 검증 |
| priceBand | 열거 | 선택 | 대표 가격대(자동 채움 시) |
| registeredAt | 시각(UTC) | 필수 | 등록 일시 |

### 6.2 등록 3경로별 좌표 확보 (registrationPath)

| 경로 | 좌표 확보 | coordConfirmed 초기값 | 폴백 |
|---|---|---|---|
| MAP_SEARCH | 지도/장소 검색 선택 → 자동 확보(M7) | true(단일 확정 선택 시) | 동명 다건 → 후보 선택 강제 [PRD 04-8 예외] |
| LINK_PARSE | URL 패턴 파싱(화이트리스트, fetch 없음) → 부분 프리필 | 대개 false(위치 특정 불가 시) | 추출 실패 → PIN_DROP 안내 [G31] |
| PIN_DROP | 지도 핀 직접 지정 | true(핀 확정 시) | — |
| OTA_RETURN | 복귀 핸드오프 카드 → 빠른 등록(숙소명·위치 자동 채움) | 매칭 성공 시 true | 미확정 시 지도 확인 |

### 6.3 불변식

- **INV-STAY1 (날짜 순서 — 하드 제약)**: `checkOut > checkIn` ∧ 두 날짜 모두 존재. 위반 시 저장 차단·인라인 표시 — 등록 숙소 필수 필드·날짜 순서 검증은 D37 하드 제약 계열, 머지 차단 테스트 [PRD 04-6 예외, U3 DoD, PBT U3-P6].
- **INV-STAY2 (좌표 확정 게이트)**: `coordConfirmed=false`인 SavedStay는 거점 자격이 없다 — 여행 연결(BaseAssignment) 및 일정 생성 기준점 투입은 "지도에서 위치 확인"으로 coordConfirmed=true가 된 후에만 [PRD 04-6·8 예외, CP1 시나리오 2].
- **INV-STAY3 (계정 풀 독립)**: SavedStay는 여행 연결 없이 존재 가능(계정 레벨 풀) — 여행 삭제가 SavedStay를 파기하지 않는다(BaseAssignment만 해제) [D15].
- **INV-STAY4 (예약 상태 부재)**: SavedStay는 예약 진행/확정 상태 필드를 갖지 않는다 — 사용자 입력 기록만(예약·결제는 외부 OTA) [PRD 04-9].
- **INV-STAY5 (선택 항목 명시값)**: otaName·reservationNo·amount가 NULL이면 목록 표시는 "미입력" 명시값(빈칸·입력 강요 금지) [PRD 04-9 예외].

---

## 7. BaseAssignment — 여행 거점 연결 (M4, CP1 공급 · 검증은 U4)

SavedStay(계정 풀)를 특정 여행의 거점으로 연결하는 조인. **데이터·API는 U3(M4) 소유**이나, 여행 컨텍스트가 필요한 **여행 내 거점 날짜 비중첩 검증·거점 UI는 U4(M6)가 완성**한다(명시적 제외 — unit-of-work U3 §제외). U3은 조인 스키마와 비중첩 판정 함수(순수 함수)를 공급한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| assignmentId | 식별자 | 필수 · 유니크 | — |
| savedStayId | 식별자(SavedStay FK) | 필수 | 연결 대상 숙소(coordConfirmed=true 전제 INV-STAY2) |
| tripId | 식별자(Trip 참조) | 필수 | 연결 여행(엔티티 정본은 U4 M6) |
| dateRange | 구조체 {start, end} | 필수 | 거점 담당 날짜 구간(다박 연속 표현) |
| isSmartDefault | 불리언 | 필수 · 기본 false | 스마트 기본 거점(공백일=직전 숙소, 첫날 공백=여행지 중심 좌표 G41) — 비차단 안내 [PRD 05-6, G41] |
| active | 불리언 | 필수 · 기본 true | 거점 활성 여부(해제 시 false — 기존 일정 유지 G97) |

**불변식**
- **INV-BASE1 (여행 내 비중첩 — U4 소비)**: 같은 tripId에 연결된 active BaseAssignment들의 dateRange는 서로 겹치지 않는다. **검증 실행은 U4**(계정 풀의 미연결 숙소는 검증 대상 아님) — U3은 비중첩 판정 순수 함수와 `ConflictDetected(overlapDates)` 계약을 공급 [D15, PBT U3-P5, CP1].
- **INV-BASE2 (좌표 전제)**: BaseAssignment는 coordConfirmed=true인 SavedStay만 참조(INV-STAY2) — 미확정 좌표 거점 금지.
- **INV-BASE3 (해제 비파괴)**: unlink(active=false)는 기존 일정을 파기하지 않는다 — 리마인드 중단·재생성 유도 배지 이벤트만 발행(U8 소비) [G97].

---

## 8. WishlistItem — 숙소 위시리스트 (M3)

탐색 중 마음에 든 숙소의 계정 귀속 저장 목록. 등록 숙소와 **별도 목록**(G129 원칙과 동형) — 위시는 후보 비교용, 등록은 거점용.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| wishItemId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account FK) | 필수 | 계정 귀속 — 기기 전환 유지(로그인 필수 전제) [D22] |
| stayRef | 식별자(StayIdentity 참조) | 필수 | 숙소 참조(내부 ID) |
| memo | 문자열 | 선택 | 메모(예: "역에서 가까움") — SECURITY-05 검증. NULL=미작성 |
| savedAt | 시각(UTC) | 필수 | 저장 일시 |
| snapshotAtSave | 구조체 {nameKo, priceBand, photo, coord} | 필수 | 저장 시점 카드 표시용 사본(가격 변동 가능 고지 대상) |
| staleFlag | 불리언(파생) | 필수 · 기본 false | 외부 소스 소실 시 true → "최신 정보 확인 불가" 표기 |

**불변식**
- **INV-WISH1 (계정 유니크)**: `(accountId, stayRef)`는 유니크 — 동일 숙소 중복 위시 금지(재저장 멱등).
- **INV-WISH2 (가격 변동 고지)**: 위시 카드의 가격 표기는 항상 "외부 OTA 기준, 변동 가능"을 병기 — 저장 시점 가격을 확정 가격으로 표시하지 않는다 [PRD 04-4 예외].
- **INV-WISH3 (소실 표기)**: staleFlag=true면 저장 당시 캐시로 표시+확인 불가 플래그(빈 카드 금지).

---

## 9. AffiliateClick + OtaPartner + PostbackRecord — OTA 딥링크·클릭 (M5)

### 9.1 AffiliateClick — 아웃바운드 클릭 집계

숙소 예약·결제로 나가는 OTA 딥링크 클릭의 **내부 집계 전용** 기록. **사용자·LLM 컨텍스트에 비노출**(내부 운영 지표 — SECURITY-11 정합). 복귀 핸드오프 카드(§9.2 로직)의 판정 데이터를 겸한다.

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| clickId | 식별자 | 필수 · 유니크 | — |
| accountId | 식별자(Account FK) | 필수 | 클릭 주체 |
| stayRef | 식별자(StayIdentity 참조) | 필수 | 이탈 대상 숙소(내부 ID) |
| otaPartner | 문자열(OtaPartner.partnerCode) | 필수 | 이탈 OTA |
| clickedAt | 시각(UTC) | 필수 · 불변 | 클릭 시각(복귀 24h 판정 기준) [G32] |
| handoffShownAt | 시각(UTC) | 선택 | 복귀 핸드오프 카드 노출 시각. NULL=미노출(1회 노출 추적) [G32] |
| handoffDismissedAt | 시각(UTC) | 선택 | 카드 무시 시각. NULL=미무시 → 재노출 억제 판정 [G32] |

### 9.2 OtaPartner — 파트너별 딥링크 템플릿

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| partnerCode | 문자열 | 필수 · 유니크 | 파트너 코드 |
| deeplinkTemplate | 문자열(URL 템플릿) | 필수 | 숙소명 검색 딥링크 조립 템플릿(실호출 없는 URL 조립 — remote config화) [D09, G31] |
| policyNote | 문자열 | 필수 | 약관 최소치·수수료 고지 문안 참조 |
| active | 불리언 | 필수 | 활성 여부 |

### 9.3 PostbackRecord — 포스트백(1차 미사용, 스키마만)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| (스키마만) | — | — | 예약 확인→1탭 자동 등록의 **설계 여지만** 확보 — 1차 미구현, 집계용 스키마만. 1차 등록은 복귀 핸드오프+수동 경로만 [G29/G108, D09] |

**불변식**
- **INV-CLICK1 (사용자·LLM 비노출)**: AffiliateClick은 내부 집계 전용 — 사용자 응답·LLM 입력 컨텍스트에 구조적으로 포함되지 않는다 [SECURITY-11, D31].
- **INV-CLICK2 (복귀 카드 1회성)**: 딥링크 이탈 후 24시간 내 첫 복귀에 대해 handoff 카드는 최근 이탈 숙소 1건에 정확히 1회 노출된다 — handoffDismissedAt 또는 handoffShownAt이 있으면 재노출 0 [G32, PBT U3-P8].
- **INV-CLICK3 (best-effort)**: 클릭 기록 실패는 사용자 흐름을 차단하지 않는다(비동기 best-effort, 지표 손실만 계측) [ADR-0011].

---

## 10. 소스별 캐싱 허용/금지 매트릭스 (D13 정본)

> 지도 API 영구 캐싱 금지 약관(ADR-0017)과 후보 풀 적재 요구의 양립을 필드×소스로 명문화한다. **정본 판정**: 셀 값 = 해당 소스 유래 필드의 저장 정책. 법무 확인(P2·P4) 결과로 셀만 교체하면 되도록 데이터 모델에 컬럼 수준 반영. 판정 함수 `cachingPolicy(source, field) → {PERMANENT, TTL_CACHE, LIVE_ONLY, NOT_HELD}`는 총함수(전 셀 정의) — 서버 M7·M3가 공유.

| 필드 | TourAPI(공공) | 카카오/네이버(지도 API) | OTA(딥링크) | 사용자 확정 스냅샷 |
|---|---|---|---|---|
| 식별자(place_id·canonicalId) | 영구(PERMANENT) | 영구(식별자는 캐싱 금지 대상 아님) | 영구 | 영구 |
| 명칭(한국어정본+영문alias) | 영구/정본 저장 | TTL 캐시 | (딥링크만·미보유) | 영구(스냅샷) |
| 좌표 | 영구/정본 저장 | **TTL 캐시**(영구 금지) | 미보유 | 영구(사용자 입력 데이터, D13) |
| 영업시간 | TTL 캐시/보강 정본 | TTL 캐시 | 미보유 | 영구(스냅샷) |
| 카테고리·유형 | 영구/정본(cat3 정본) | TTL 캐시 | 미보유 | 영구(스냅샷) |
| 편의시설(amenities) | TTL 캐시(채움률 검증) | TTL 캐시 | 미보유 | 영구(스냅샷) |
| 대표 가격대(정적 구간) | TTL 캐시 | TTL 캐시 | 정적 메타 표시 가능 | 영구(등록 시) |
| **정확 1박 가격·재고** | 미보유 | 미보유 | **LIVE_ONLY**(캐싱 금지) | 미보유(캐싱 0) |
| 사진 URL | TTL 캐시 | TTL 캐시 | (콘텐츠 API 계약 시) | 영구(스냅샷) |
| 리뷰·평점 | 미표시 | 미표시 | **NOT_HELD**(외부 OTA 위임) | 미표시 |

**매트릭스 불변식**
- **INV-MTX1 (지도 API 영구 금지)**: `cachingPolicy(KAKAO|NAVER, {명칭·좌표·영업시간·카테고리})` ≠ PERMANENT — 지도 API 유래 비식별자 필드의 영구 저장 0(TTL 캐시만) [ADR-0017, G193, PBT U3-P3].
- **INV-MTX2 (정확 가격 비보관)**: `cachingPolicy(*, 정확 1박 가격)` = LIVE_ONLY ∨ 미보유 — 어떤 소스의 정확 가격도 저장되지 않는다 [D09, PBT U3-P3].
- **INV-MTX3 (사용자 확정 영구화)**: 사용자 확정 스냅샷 열은 좌표·영업시간을 영구 저장(D13) — 지도 API 실시간 캐싱 금지의 정책적 예외 축(P2 법무 확인 전제) [G124, G70].
- **INV-MTX4 (리뷰 미보유)**: 리뷰·평점은 어떤 소스에서도 앱 내부에 보관·표시하지 않는다(원문·점수 모두 외부 OTA 딥링크 위임) [PRD 04-3].

---

## 11. TrendingAggregate — 인기 장소 집계 (M7)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| region | 식별자(지역) | 필수 | 집계 지역 범위 |
| canonicalPoiId | 식별자(Poi 참조) | 필수 | 대상 POI |
| weightedScore | 수치 | 필수 | 최근 7일 저장+방문 가중합 [G2] |
| computedAt | 시각(UTC) | 필수 | 일 1회 배치 갱신 시각 |

**불변식**: **INV-TREND1 (허위 데이터 금지)**: 집계 미존재 시 빈 결과 반환 — 클라이언트는 '지금 인기 장소' 섹션 비표시(가짜 데이터 금지) [G2]. **INV-TREND2 (표시 게이트)**: `region`은 홈 대시보드 슬롯(U2)에 공급 — U3은 데이터만 생산, 노출은 U2 소비.

---

## 12. StayTimeTable — 카테고리 체류 기본값 (M7)

| 속성 | 타입 | 제약 | 의미·근거 |
|---|---|---|---|
| category | 열거(20~30종) | 필수 · 유니크 | 표준 카테고리 |
| min / rec / max | 정수(분) | 필수 | 체류 시간 범위(최소·권장·최대) — remote config 보정 [G51] |

**불변식**: **INV-STT1 (범위 정합)**: `min ≤ rec ≤ max`. **INV-STT2 (공급 계약)**: U5 C2/M8이 소비하는 정적 테이블 — U3은 정본 제공, 실측 보정은 출시 후.

---

## 13. 엔티티-스토리 추적 요약

| 엔티티 | 주 근거 스토리 | 주 결정 |
|---|---|---|
| Poi·PoiSourceRef | US-E3-01·02·03 | D08, D13, G133/G148, G123, G199 |
| PoiSnapshot | (US-E4-08·US-E8-01 소비) | D13, G70, G124 |
| SavedPlace | US-E2-05 | D13, G8, G129 |
| StaySearchCache | US-E3-01·02·03·11 | D09, D13, G196 |
| StayIdentity·StayIdentityMapping | US-E3-05 | D17, G133/G148 |
| SavedStay | US-E3-06·07·08·09 | D09, D13, D15, G31, G120 |
| BaseAssignment | US-E3-07 (검증 소비 U4) | D15, G41, G97 |
| WishlistItem | US-E3-04 | D22, G103, G129 |
| AffiliateClick·OtaPartner·PostbackRecord | US-E3-05 | D09, D17, G28, G29/G108, G32 |
| TrendingAggregate | (US-E2-02 소비) | G2 |
| StayTimeTable | (US-E5-03 소비) | G51 |
