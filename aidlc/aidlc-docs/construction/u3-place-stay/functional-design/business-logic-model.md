# U3 숙소·장소 데이터 — 비즈니스 로직 모델 (Business Logic Model)

> 2026-07-04 · Functional Design · 핵심 프로세스 플로우 5종 + Testable Properties(PBT-01)
> 참조: 엔티티·불변식 [domain-entities.md](./domain-entities.md), 규칙 [business-rules.md](./business-rules.md)(BR-U3-xx), 메서드 계약 [component-methods.md](../../../inception/application-design/component-methods.md) M3·M4·M5·M7
> 표기: 플로우 ID `FLOW-x`, 단계 `Sx`, 분기 `Bx`, 예외 `Ex`. 기술 중립 — 전송·저장·캐시 구현체는 NFR/Infra 단계 소유.

---

## FLOW-1 숙소 탐색 → 필터 → 상세 → 위시

**진입**: 탐색 랜딩 '숙소' 진입 → 여행지(또는 '내 주변') 입력 → `M3.searchStays(query, page)`
**관련**: US-E3-01~04, US-E3-10, US-E3-11 · BR-U3-01~09, 23, 29~31 · D09, D13, G33, G34, D25

```text
S1 여행지 입력 검증 [BR-U3-01]
   ├─ E1 여행지 공백 → 탐색 버튼 비활성 유지(서버 호출 없음)
   └─ B1 '내 주변' 선택 → 위치 권한 판정(U1 effectiveCapabilities) [BR-U3-05]
        └─ 권한 거부 → 등록 숙소/여행지 중심 좌표로 대체 조회(중단 없음)
S2 탐색 실행 — TourAPI+지도(M7 경유) 조회, "숙소를 모으는 중" 로딩 [BR-U3-02]
   ├─ E2 일부 소스 실패 → 가용 결과+degraded "일부 정보가 빠졌을 수 있어요" [BR-U3-36]
   ├─ E3 1차(카카오) 실패 → 2차(네이버) 폴백 [D08]
   └─ E4 전체 실패 → "지금 숙소 정보를 불러올 수 없어요"+재시도 / 직접 등록(FLOW-2) 우회
S3 결과 카드 구성 — 숙소명·위치·대표 가격대(정적) [BR-U3-03]
   (정확 가격 일괄 조회·캐싱 없음 — INV-CACHE2)
S4 필터·정렬 적용 [BR-U3-06~09]
   ├─ 유형 필터(TourAPI cat3 정본, 전면) · 편의시설(채움률 검증 항목만)
   ├─ 대표 가격대순(가격순 정렬 없음) · 거리(여행지 중심 직선거리 — 소요시간 미표시 D25)
   ├─ E5 필터 0건 → 원인 필터 표기+완화 제안(탭 시 조정 재검색) [BR-U3-36]
   └─ E6 결과 0건/데이터 부족 지역 → "직접 등록하면 일정을 만들어 드려요"+등록 경로 [US-E3-10]
S5 페이지네이션/무한스크롤 — N건 이상 추가 노출
S6 상세 진입 — M3.getStayDetail(정적 콘텐츠: 위치·사진·가격대·편의시설·체크인/아웃 시각) [BR-U3-23]
   ├─ 리뷰·평점 미표시 → "외부 OTA에서 확인" 딥링크 [INV-MTX4]
   └─ 누락 필드 "미확인" 명시값(빈칸 금지)
S7 '가격 보기' 탭(선택) → 날짜·인원 바텀시트 → M3.getLivePrice(라이브·비캐싱) [BR-U3-04]
   └─ E7 UpstreamUnavailable → "외부 OTA에서 확인" 딥링크 대체
S8 위시 저장 → M3.addToWishlist(계정 귀속·기기 전환 유지) [BR-U3-29]
   └─ 위시 카드 "외부 OTA 기준, 변동 가능" 병기, 메모 추가/삭제 [BR-U3-30]
```

**사후 조건**: 목록·상세 어디에도 정확 1박 가격이 저장(캐싱)되지 않는다(INV-CACHE2). 소요시간 표기 0(D25). 위시는 계정 귀속으로 기기 전환에 무손실.

---

## FLOW-2 숙소 등록 3경로 (지도 검색 · 링크 파싱 · 핀 지정)

**진입**: 탐색 결과·저장 목록 '[예약했어요/등록]' 또는 직접 등록 진입 → `M4.registerStay(input)`
**관련**: US-E3-06, US-E3-08 · BR-U3-10~14, 25~26 · D09, D13, D15, G31, G120

```text
S1 등록 경로 선택(단일 폼 수렴) [BR-U3-13]
   ├─ 경로 a: 지도/장소 검색(M7) → 숙소 선택
   │   └─ E1 동명·유사 다건 → 후보+핀 목록 제시, 사용자 정확 1곳 선택(좌표 확정)
   ├─ 경로 b: OTA URL 붙여넣기 → M4.parseOtaLink [BR-U3-25]
   │   ├─ 화이트리스트 내 패턴 파싱만(fetch 없음) → 숙소명·위치 프리필
   │   └─ E2 화이트리스트 밖/추출 실패 → LinkParseFailed → 경로 c 폴백 [BR-U3-26]
   └─ 경로 c: 지도 핀 직접 지정 → 좌표 확정(coordConfirmed=true)
S2 폼 자동 채움 — 숙소명·위치·가격대(경로 a/b 성공 시), 사용자는 날짜·인원 입력
S3 저장 검증 [BR-U3-10]
   ├─ 필수: 숙소·체크인/아웃·인원 / 선택: 예약번호·금액
   ├─ E3 checkOut ≤ checkIn 또는 날짜 누락 → ValidationFailed 인라인(저장 차단) [INV-STAY1 하드 제약]
   ├─ E4 coordConfirmed=false → CoordinateUnresolved → "지도에서 위치 확인" 강제 [BR-U3-11, INV-STAY2]
   └─ E5 국내 좌표 범위 이탈 → 차단+사유 [G120]
S4 canonical 숙소 ID 해석 — M4.resolveStayIdentity (FLOW-5) [BR-U3-18]
S5 스냅샷 영구 저장 — M7.snapshotUserConfirmed(REGISTERED_STAY) [BR-U3-34, INV-SNAP1]
S6 SavedStay 생성(계정 레벨 풀 — 여행 없이 등록 가능) [BR-U3-12, INV-STAY3]
S7 등록 완료 → [AI 일정 생성하기] 온램프 즉시 노출 [BR-U3-14]
S8 (사후) 수정·취소 — 연결 숙소는 영향 블록 차단형 확인(G39), 날짜 변경 시 비중첩 재검증(FLOW-3)
```

**가드**: coordConfirmed=false인 SavedStay는 거점 자격·일정 생성 기준점 투입 불가(INV-STAY2) — CP1 소비(U4)에서도 동일 차단. 지도 검색 API 무응답 시에도 항상 핀 지정 폴백 존재(침묵 실패 금지).

---

## FLOW-3 OTA 딥링크 이동 → 복귀 핸드오프

**진입**: 상세·저장 화면 [외부 OTA에서 예약하기] → `M5.getOtaLinkOptions` → `M5.buildDeeplink`
**관련**: US-E3-05 · BR-U3-20~24, 27~28 · D09, D17, G32

```text
S1 OTA 선택 목록 — 내부 숙소 ID 기준 동일 숙소 OTA 묶기 [BR-U3-20]
S2 이동 직전 제휴 수수료 고지 표시 [BR-U3-22]
   └─ '다시 보지 않기' 설정 시 showDisclosure=false (setDisclosureHidden)
S3 딥링크 생성·이동 — 숙소명 검색 딥링크(외부 브라우저/인앱 웹뷰) [BR-U3-21]
   └─ E1 링크 404·만료·무응답 → LinkUnavailable → 다른 OTA 링크/장소 검색 우회 [BR-U3-24]
S4 아웃바운드 클릭 기록 — M5.recordOutboundClick(내부 집계·사용자/LLM 비노출) [INV-CLICK1]
   └─ 기록 실패는 흐름 비차단(best-effort, INV-CLICK3)
S5 【앱 포그라운드 복귀】 M5.getReturnHandoffCard [BR-U3-27]
   ├─ B1 이탈 후 24h 내 첫 복귀 → 최근 이탈 숙소 1건 카드 1회 노출(handoffShownAt 기록)
   │     └─ 카드 탭 → 빠른 등록(OTA_RETURN 경로 — FLOW-2 S2로 자동 채움 진입)
   ├─ B2 이미 노출/무시 이력(handoffDismissedAt) → null(재노출 억제) [BR-U3-28]
   └─ B3 24h 경과 → null
S6 카드 무시 → M5.dismissHandoffCard(재노출 억제) — 계속 탐색(비강제) [BR-U3-28]
```

**사후 조건**: 핸드오프 카드는 이탈 건당 정확히 1회 노출(INV-CLICK2). 아웃바운드 클릭은 사용자·LLM 컨텍스트에 구조적 비노출(SECURITY-11). 앱은 예약 확정 상태를 추적하지 않는다.

---

## FLOW-4 POI 정본 파이프라인 (수집 → 정규화 → canonical ID → 캐싱)

**진입**: (배치) POI 정본 동기화 일 1회 / (온디맨드) 탐색·검색·후보 풀 요청 시 실시간 조회
**관련**: US-E3-01~03(간접), 횡단 · BR-U3-02, 32~38 · D08, D13, G8, G115, G123, G133/G148, G192, ADR-0009·0017

```text
S1 수집 — 카카오 장소 검색·지오코딩(1차)+TourAPI, 실패 시 네이버(2차) [D08]
S2 정규화 — 소스별 카테고리 코드 → 단일 표준 스키마, 한국어명 정본+영문 alias [ADR-0009, G123/G148]
S3 canonical POI ID 매핑 — M7.resolvePoiIdentity [BR-U3-32]
   ├─ 좌표 50m 근접 + 명칭 유사도 → 기존 canonical 귀속(N:1 place_id) [INV-POI1]
   ├─ B1 임계 미달 → 신규 canonicalPoiId 생성(보수적 — 과병합보다 과분리)
   └─ 재수집 동일 입력 → canonical 불변(멱등·대칭) [INV-POI2, PBT U3-P1]
S4 캐싱 판정 — cachingPolicy(source, field) 총함수(§10 매트릭스) [BR-U3-33]
   ├─ TourAPI(공공)·식별자 → 영구/정본 저장
   ├─ 지도 API 비식별자(명칭·좌표·영업시간·카테고리) → TTL 캐시만(영구 금지) [INV-MTX1]
   ├─ 정확 1박 가격 → LIVE_ONLY(캐싱 0) [INV-MTX2]
   └─ TTL 만료 → 재조회 강제, 실패 시 stale+'갱신 실패' 표기 [BR-U3-35, INV-CACHE1]
S5 dataStatus 판정
   ├─ 좌표·영업시간 확보 → ACTIVE / 미확인 → UNVERIFIED(사용자 확인 후보 분리) [INV-POI3]
   └─ 동기화 확인 불가 → LOST → '확인 불가' 배지, 후보 풀·시드 투입 제외 [BR-U3-37, G8]
S6 후보 풀 구성(소비 — U5) — closed-set(LLM은 이 ID에서만 선택, 그라운딩 구조적 보장) [G115]
S7 사용자 확정 시 → M7.snapshotUserConfirmed(영구 스냅샷) [BR-U3-34, INV-SNAP1]
S8 (배치) 인기 장소 집계(G2)·영업시간 재조회(G192) — 정기 영업시간 변경만 자동, 임시휴무 제외
```

**소실 처리 멱등(S5·S8)**: 정본 동기화 재실행이 LOST 상태·스냅샷·시드 제외 신호를 중복 변경하지 않는다(BR-U3-38, PBT U3-P9).

---

## FLOW-5 숙소 ID 통합 매칭

**진입**: 등록(FLOW-2 S4)·OTA 묶기(FLOW-3 S1) 시 → `M4.resolveStayIdentity(externalRefs)`
**관련**: US-E3-05, US-E3-06 · BR-U3-18~20 · D17

```text
S1 외부 참조 수신 — {source, externalId} 목록
S2 기존 매핑 조회 — (source, externalId) → canonicalStayId [INV-SID2]
   ├─ B1 존재 → 기존 내부 ID 반환(라운드트립 — 재해석 동일) [INV-SID1, PBT U3-P2]
   └─ 부재 → S3
S3 좌표+이름 유사도 자동 매칭 [BR-U3-19]
   ├─ 좌표 50m 근접 + 명칭 유사도 임계 이상 → 기존 canonicalStayId 귀속(N:1 매핑 추가)
   │     (결정적·대칭적 — 동일 입력 동일 결과, A↔B) [INV-SID3, PBT U3-P4]
   └─ B2 임계 미달 → 신규 canonicalStayId 생성(보수 — 오매칭 방지)
S4 운영자 보정 여지 — operatorOverride로 자동 결과 교정(보정도 결정적) [D17]
S5 반환 — canonicalStayId(SavedStay·WishlistItem·OTA 묶기의 정본 키)
```

**사후 조건**: 임의의 외부 참조 집합에 대해 각 (source, externalId)는 정확히 1개 canonicalStayId로 해석(N:1)되고, 재해석은 동일 결과(INV-SID1 라운드트립).

---

## Testable Properties (PBT-01 — 속성 식별 정본)

> PBT 전체 강제(D05~07 확장 구성). 프레임워크: 서버=Kotest Property Testing, 클라이언트=fast-check. 시드 로깅·수축(shrinking) 필수(PBT-08). 아래 속성은 U3 Code Generation의 DoD 항목(속성 테스트 존재·통과)이다. **U3 DoD가 명시한 5속성**(POI 매칭 멱등성·대칭성 / 숙소 ID N:1 매핑 불변식 / TTL 캐시 만료 / OTA 링크 파서 / 등록 폼 검증)을 전부 포함하고 확장한다.

### 컴포넌트별 속성 식별 표

**M7 Place Data (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U3-P1 | 매칭 불변식·멱등·대칭 (하드 제약) | **canonical POI ID 매핑**: 임의의 소스별 place_id 집합(좌표 근접·명칭 유사 다양도)에 대해 (a) 동일 실세계 장소(50m 근접∧명칭 유사 임계↑)는 정확히 1개 canonicalPoiId(INV-POI1), (b) 같은 입력 재수집 시 canonical 불변(멱등), (c) 매칭 대칭(A→B ⇔ B→A), (d) 임계 미달은 신규 생성(과병합 0) | place_id 생성기(좌표 지터·명칭 편집거리 변형·동명 원거리/유사명 근접 케이스) + 재수집 순서 셔플 |
| U3-P3 | 캐싱 정책 게이트 (총함수) | **소스별 캐싱**: `cachingPolicy(source, field)`가 전 (소스×필드) 셀에 정확히 1값 반환(총함수)이며 (a) 지도 API 비식별자 필드 영구 저장 0(INV-MTX1), (b) 정확 1박 가격 캐싱 0(INV-MTX2), (c) TTL 만료분 조회는 재조회 강제(만료 후 stale을 최신으로 반환 0, INV-CACHE1), (d) 사용자 확정 스냅샷만 좌표·영업시간 영구 | (소스×필드) 전수 조합 + 가상 clock(TTL 경계 점프) + 저장/조회 시퀀스 생성기 |
| U3-P9 | 멱등성·소실 처리 | **소실 POI 시드 제외**: 임의의 동기화 이벤트 시퀀스(확인 불가·재확인·폐업 혼합)에 대해 (a) 최종 dataStatus=LOST면 후보 풀·필수 방문지 시드 투입 제외 신호 true, (b) 배치 재실행 멱등(스냅샷 중복 생성 0·상태 진동 0, INV-SNAP2·BR-U3-38), (c) 스냅샷은 소실에도 잔존(파기 0) | dataStatus 전이 시퀀스 생성기(중복 이벤트 포함) + 스냅샷 존재 케이스 |

**M4 Saved Accommodation (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U3-P2 | N:1 매핑 라운드트립 (하드 제약) | **숙소 ID N:1 매핑**: 임의의 (source, externalId) 집합에 대해 (a) 각 외부 ID는 정확히 1개 canonicalStayId로 해석(INV-SID1), (b) 해석→재해석 라운드트립 동일, (c) 하나의 외부 ID가 두 내부 ID에 매핑 0(INV-SID2), (d) 매핑 추가로 기존 canonical 불변 | 외부 참조 집합 생성기(동일 숙소 다소스·상이 숙소 유사명) + 해석 순서 셔플 |
| U3-P4 | 매칭 결정성·대칭성 | **좌표+이름 유사도 매칭**: 임의 숙소 쌍에 대해 (a) 동일 입력 동일 결과(결정적), (b) 대칭(A가 B에 매칭 ⇔ B가 A에), (c) 운영자 보정 적용 후에도 결정성 유지, (d) 임계 미달은 신규 canonical(보수) | 숙소 쌍 생성기(좌표 지터·명칭 편집거리) + operatorOverride 주입 |
| U3-P5 | 논리 게이트·비중첩 | **거점 날짜 비중첩(판정 공급)**: 같은 여행에 연결된 임의 거점 dateRange 집합에 대해 (a) 비중첩 판정이 겹침 존재 ⇔ ConflictDetected(overlapDates), (b) 계정 풀 미연결 숙소는 판정 대상 제외, (c) 판정 순수 함수(동일 입력 동일 출력) — U4가 소비하는 계약 | dateRange 집합 생성기(인접·포함·부분겹침·비접촉) + 연결/미연결 혼합 |
| U3-P6 | 검증 함수 동치 (하드 제약) | **등록 폼 검증**: 임의 등록 입력에 대해 통과 ⇔ (숙소∧체크인∧체크아웃∧인원 존재 ∧ checkOut>checkIn ∧ 국내 좌표 범위 ∧ coordConfirmed), 실패 사유는 첫 위반 항목과 일치, 좌표 미확정은 CoordinateUnresolved | 등록 입력 생성기(날짜 경계 역전·동일일·누락·국외 좌표·미확정 좌표 변형) |

**M3 Accommodation Search (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U3-P10 | 진단 함수 정확성 | **필터 0건 진단**: 임의 필터 조합에서 결과 0건이면 진단은 (a) 0으로 만든 필터를 정확히 식별(제거 시 결과>0인 필터만 원인), (b) 완화 제안이 존재, (c) 필터 완전 해제 시 진단=∅ | 필터 조합 생성기 + 가상 결과셋(필터별 매칭 분포) |

**M5 Affiliate Link (서버 — Kotest)**

| ID | 카테고리 | 속성 서술 | 제너레이터 요구 |
|---|---|---|---|
| U3-P7 | 화이트리스트 방어 + 폴백 | **OTA 링크 파서**: 임의 URL 문자열에 대해 (a) 화이트리스트 도메인·패턴 밖 URL 전부 거부(LinkParseFailed), (b) 페이지 fetch 부작용 0(순수 파싱), (c) 파싱 실패는 항상 수동 핀 지정 폴백 경로 반환(막다른 흐름 0), (d) 성공 시 추출 필드는 URL에서만 유래 | URL 생성기(화이트리스트 내/밖·부분 일치·인젝션 페이로드·비URL 문자열) |
| U3-P8 | 1회성·시간 게이트 | **복귀 핸드오프 카드**: 임의의 (이탈·복귀·무시·시각 경과) 시퀀스에 대해 (a) 이탈 후 24h 내 첫 복귀에 최근 1건 정확히 1회 노출, (b) 무시(dismiss) 또는 노출 후 재노출 0(INV-CLICK2), (c) 24h 경과 후 노출 0, (d) 복수 이탈 시 최근 1건만 | 이탈/복귀 이벤트 시퀀스 생성기 + 가상 clock(24h 경계) |

### 속성 없는 컴포넌트 (No PBT properties identified)

| 컴포넌트 | 판정 | 사유 |
|---|---|---|
| M3 탐색 조회(searchStays 외부 조립) | No PBT properties identified | 외부 소스 조립·페이지네이션 — 보편 양화 도메인 로직은 필터 0건 진단(U3-P10)으로 한정. 부분 실패 degraded 합성은 예시 기반 통합 테스트+어댑터 fake로 검증(RESILIENCY-10) |
| M5 딥링크 조립(buildDeeplink) | No PBT properties identified | URL 템플릿 조립(remote config) — 파서(U3-P7)와 달리 순수 문자열 치환. 고지 판정은 예시 기반 테스트로 충분 |
| M7 인기 장소 집계·체류 테이블 공급 | No PBT properties identified (U3 시점) | 가중합 집계·정적 테이블 조회 — 도메인 불변식보다 배치 정확성. 집계 미존재 시 빈 결과(허위 데이터 금지)는 예시 기반 테스트 |

### 커버리지 대조 (U3 DoD → 속성)

| U3 DoD 명시 속성 | 대응 |
|---|---|
| POI 매칭 멱등성·대칭성(같은 입력 재수집 시 canonical 불변) | U3-P1 |
| 숙소 ID N:1 매핑 불변식(외부 ID는 정확히 1개 내부 ID) | U3-P2 |
| TTL 캐시 만료 속성(만료 후 재조회 강제) | U3-P3 |
| OTA 링크 파서(화이트리스트 밖 URL 전부 거부·파싱 실패 폴백) | U3-P7 |
| 등록 폼 검증 함수 | U3-P6 |
| (설계 확장) 좌표+이름 유사도 매칭 결정성·대칭성 | U3-P4 |
| (설계 확장) 거점 날짜 비중첩 판정(CP1 공급) | U3-P5 |
| (설계 확장) 소실 POI 시드 제외 멱등성 | U3-P9 |
| (설계 확장) 복귀 핸드오프 1회성 / 필터 0건 진단 | U3-P8 / U3-P10 |

**속성 합계: 9개** (M7: 3 · M4: 3 · M3: 1 · M5: 2)

### 하드 제약(D37) 대조

| 하드 제약(U3 소관) | 방어 속성·불변식 |
|---|---|
| canonical POI ID 무결성(동일 실세계 장소 1 canonical) | U3-P1 · INV-POI1 |
| 등록 숙소 필수 필드·날짜 순서(체크아웃>체크인) | U3-P6 · INV-STAY1 |
