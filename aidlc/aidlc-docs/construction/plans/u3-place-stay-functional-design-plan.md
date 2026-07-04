# U3 숙소·장소 데이터 — Functional Design 실행 플랜

> 2026-07-04 · CONSTRUCTION 페이즈 · 유닛 U3 (E3 스토리 11개 — US-E3-01~11)
> 정본 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U3 섹션 + [unit-of-work-dependency.md](../../inception/application-design/unit-of-work-dependency.md) CP1 계약, [stories.md](../../inception/user-stories/stories.md) Epic 3, [components.md](../../inception/application-design/components.md) M3·M4·M5·M7, [component-methods.md](../../inception/application-design/component-methods.md) M3·M4·M5·M7, [requirements.md](../../inception/requirements/requirements.md) §7.1~7.2·D08·D09·D13·D15·D17, docs/PRD/04-숙소탐색.md

## 질문 처리 방침

**사용자 연속 진행 지시에 따라, 본 스테이지의 확인 질문은 requirements.md의 기확정 결정(D01~D38·Δ1~Δ10·G 제안 기본값 — 문서 승인으로 확정)으로 대체한다.** U3 소관 핵심 결정: D09(계약 전 MVP 숙소), D13(하이브리드 POI 캐싱), D15(계정 레벨 숙소 풀), D17(내부 숙소 ID·소스 매핑), D08(카카오+TMap+네이버 폴백), 그리고 §7.1~7.2의 G 기본값(G8·G31·G32·G33·G34·G129·G133·G148·G158). Functional Design에서 추가 확정할 설계 결정은 아래 "설계 결정 목록"(FD-U3-xx)에 기록한다.

> **선결 과제 전제**: P2(지도 API 약관·D13 스냅샷 적법성)·P4(TourAPI 캐싱 조건)는 **U3 Functional Design 착수 전 완료(설계 차단)** 항목이다. 본 설계는 소스별 캐싱 매트릭스를 데이터 모델에 컬럼 수준으로 반영(domain-entities §10)해 법무 확정 결과를 셀 교체로 흡수하도록 구성했다.

## 실행 체크리스트

- [x] 정본 문서 로드·대조 (unit-of-work U3 / dependency CP1 / stories E3 / components M3·M4·M5·M7 / component-methods M3·M4·M5·M7 / requirements §7.1~7.2·D08·09·13·15·17 / PRD 04)
- [x] 도메인 엔티티 정의 — [domain-entities.md](../u3-place-stay/functional-design/domain-entities.md)
  - [x] Poi — canonical POI ID 정본 키(G133)·소스별 place_id N:1·한국어정본+영문alias(G148)·영업시간·카테고리·좌표·dataStatus 소실 플래그(G8)
  - [x] PoiSnapshot — 사용자 확정 데이터 영구 저장(D13)·소실 독립 불변식
  - [x] SavedPlace — 계정 저장 POI(Case A 온램프, G8·G129)
  - [x] StaySearchCache — 정적 콘텐츠 TTL 캐시·영구 캐싱 금지·정확 가격 비보관(D13, PRD 04 註)
  - [x] SavedStay — 등록 숙소·계정 레벨(D15)·날짜/좌표 검증 상태·등록 3경로(G31·G120)
  - [x] StayIdentity·StayIdentityMapping — 내부 숙소 ID↔소스 외부 ID N:1(D17)·매칭 결정성
  - [x] BaseAssignment — 거점 연결(CP1 공급, 여행 내 비중첩 검증은 U4 소비)
  - [x] WishlistItem — 숙소 위시(계정 귀속·가격 변동 고지·별도 목록 G129)
  - [x] AffiliateClick·OtaPartner·PostbackRecord — 아웃바운드 클릭 집계(사용자·LLM 비노출)·복귀 핸드오프(G32)·포스트백 스키마 여지(G29)
  - [x] TrendingAggregate·StayTimeTable — 인기 장소(G2)·체류 기본값(G51) 보조 엔티티
  - [x] 소스별 캐싱 허용/금지 매트릭스(D13) — 필드×소스 총함수·불변식 명문화(§10)
  - [x] 엔티티별 속성 표(타입·제약·근거 ID)·관계·불변식
- [x] 비즈니스 규칙 카탈로그 — [business-rules.md](../u3-place-stay/functional-design/business-rules.md) (BR-U3-01~39, 조건/동작/위반 시 처리/근거)
  - [x] 숙소 탐색(여행지만·날짜 미입력 D09·대표 가격대 정적·라이브 가격 G33) / 필터·정렬(가격순 없음·거리 기준점 G34·소요시간 미표시 D25)
  - [x] 등록(계정 레벨 D15·좌표 미확정 핀 지정 강제·날짜 검증) / 거점 연결 준비(CP1 공급·비중첩 U4 소비)
  - [x] 숙소 ID 통합(좌표+이름 유사도 D17) / OTA 딥링크·수수료 고지(D09·크롤링 금지·리뷰 미표시) / URL 파싱(화이트리스트 G31)
  - [x] 복귀 핸드오프(24h·1회 G32) / 위시리스트(계정 귀속) / POI 캐싱 정책(D13 소스별) / 소실 POI(확인 불가 배지 G8) / 포스트백 이연(G29 수동 경로만)
- [x] 비즈니스 로직 모델 — [business-logic-model.md](../u3-place-stay/functional-design/business-logic-model.md)
  - [x] 프로세스 플로우 5종(탐색→필터→상세→위시 / 등록 3경로 / OTA 딥링크→복귀 핸드오프 / POI 정본 파이프라인 / 숙소 ID 통합 매칭)
  - [x] Testable Properties(PBT-01) — 컴포넌트별 속성 식별 표(U3-P1~P10), U3 DoD 5속성 커버리지 대조, 하드 제약 대조
- [x] 프런트엔드 컴포넌트 설계 — [frontend-components.md](../u3-place-stay/functional-design/frontend-components.md) (탐색·필터·상세·위시·등록 3경로 폼·복귀 핸드오프·shared/map 카카오 브리지·M3/M4/M5/M7 매핑·data-testid)
- [x] 추적 ID(US/D/G/BR/FD/PBT) 일관 표기 검수
- [x] 확장 규칙(SECURITY·RESILIENCY·PBT) 관련 조항 반영 검수 — 기술 중립 유지(알고리즘·인프라·SDK 구현체는 NFR/Infra 이관)

## 설계 결정 목록 (본 스테이지 확정분)

| # | 결정 | 내용 | 근거 |
|---|---|---|---|
| FD-U3-01 | canonical 키 이원화 | POI는 `canonicalPoiId`, 숙소는 `canonicalStayId`를 각각 정본 키로 분리 — 소스별 외부 ID는 양쪽 모두 N:1 매핑(동일 패턴). POI/숙소는 별개 정체성(숙소도 장소지만 SavedStay는 예약 컨텍스트 보유) | D17, G133/G148 |
| FD-U3-02 | 소스별 캐싱 매트릭스 총함수화 | `cachingPolicy(source, field) → {PERMANENT, TTL_CACHE, LIVE_ONLY, NOT_HELD}`를 전 (소스×필드) 셀 정의 총함수로 확정(§10). 지도 API 비식별자=TTL, TourAPI·사용자 확정 스냅샷=영구, 정확 가격=LIVE_ONLY. 법무(P2·P4) 결과는 셀 교체로 흡수 | D13, ADR-0017, G70/G105/G124/G169/G193 |
| FD-U3-03 | 사용자 확정 스냅샷 영구화 축 | 지도 API 영구 캐싱 금지의 정책적 예외를 'PoiSnapshot(사용자 입력 데이터)'로 한정 확정 — 탐색·추천 풀은 스냅샷화 금지(TTL만). 원본 갱신은 스냅샷에 역류하지 않음(시점 고정) | D13, P2, G124 |
| FD-U3-04 | dataStatus 4상태 | Poi 생애를 `ACTIVE / UNVERIFIED / LOST / CLOSED` 4상태로 확정. UNVERIFIED는 자동 확정 안 함(사용자 확인 후보 분리), LOST/CLOSED는 후보 풀·필수 방문지 시드 투입 제외(스냅샷은 잔존) | G8, G192, G199 |
| FD-U3-05 | 매칭 보수 정책 | 좌표(50m)+명칭 유사도 매칭은 임계 미달 시 신규 canonical 생성(과병합보다 과분리 우선) + 운영자 보정 여지 — 오매칭이 U5 그라운딩을 오염시키는 리스크 최소화 | D17, G133/G148, U3 리스크 |
| FD-U3-06 | 좌표 확정 게이트 | coordConfirmed=false SavedStay는 거점 자격·일정 생성 기준점 투입 불가 — "지도에서 위치 확인" 강제. CP1 소비(U4)에서도 동일 차단(이중 방어) | D15, PRD 04-6·8 예외, CP1 |
| FD-U3-07 | 거점 비중첩 분업 | BaseAssignment 스키마·비중첩 판정 순수 함수는 U3 공급, 여행 컨텍스트 검증 실행·거점 UI는 U4 소비 — 계정 풀 미연결 숙소는 검증 대상 아님 | D15, unit-of-work U3 §제외, CP1 |
| FD-U3-08 | 복귀 핸드오프 1회성 모델 | AffiliateClick의 handoffShownAt/handoffDismissedAt로 24h·1회·무시 억제를 상태 기반 확정 — 최근 이탈 숙소 1건 한정, 비강제 | G32 |
| FD-U3-09 | 포스트백 이연 | 1탭 자동 등록은 후속 이연 — 1차는 복귀 핸드오프+수동 빠른 등록만. PostbackRecord는 집계용 스키마 여지만(수신 로직 미구현) | D09, G29/G108 |
| FD-U3-10 | 정확 가격 라이브 한정 | 정확 1박 가격은 '가격 보기'/딥링크 명시 액션에서만 라이브 조회·비캐싱, 목록은 대표 가격대(정적 구간)만 — 가격순 정렬 미제공('대표 가격대순' 대체) | D09, G33, G196, PRD 04 註 |

## 산출물

| 파일 | 내용 |
|---|---|
| `u3-place-stay/functional-design/domain-entities.md` | 엔티티 12종 정의·속성 표·관계·불변식 + 소스별 캐싱 매트릭스(§10) |
| `u3-place-stay/functional-design/business-rules.md` | BR-U3-01~39 규칙 카탈로그 |
| `u3-place-stay/functional-design/business-logic-model.md` | 프로세스 플로우 5종 + Testable Properties(PBT-01 — 속성 9개) |
| `u3-place-stay/functional-design/frontend-components.md` | 숙소·장소 화면 플로우·컴포넌트 계층·shared/map 카카오 브리지·M3/M4/M5/M7 매핑·data-testid |

## 확장 규칙 컴플라이언스 요약 (Functional Design 스테이지 적용분)

| 규칙 | 상태 | 근거 |
|---|---|---|
| SECURITY-05 (입력 스키마 검증·링크 파싱 방어) | 준수 | 등록 폼 검증(BR-U3-10)·URL 화이트리스트 파싱(BR-U3-25)·메모 검증(BR-U3-29) |
| SECURITY-08 (소유권 검증 — IDOR) | 준수 | 위시·등록 숙소·저장 POI 계정 귀속(BR-U3-29·12) |
| SECURITY-11 (내부 지표 LLM 비주입) | 준수 | AffiliateClick 사용자·LLM 비노출(INV-CLICK1) |
| RESILIENCY-10 (외부 어댑터 타임아웃·서킷·폴백) | 준수(설계 명세) | 카카오→네이버 폴백(D08)·부분 실패 degraded·전체 실패 수동 등록 우회(BR-U3-36)·지도 SDK 폴백(§7) — 구현 수치는 NFR |
| PBT-01 (속성 식별) | 준수 | 속성 9개 식별(U3-P1~P10)·U3 DoD 5속성 커버리지 대조 |
| 그 외 SECURITY/RESILIENCY 세부(알고리즘·수치·인프라) | N/A(본 스테이지) | 기술 중립 — NFR Requirements/Design·Infrastructure Design 소유 |
| 하드 제약(D37) | 준수(식별) | canonical POI ID 무결성(INV-POI1·U3-P1)·등록 날짜 순서(INV-STAY1·U3-P6) 식별 — 테스트 구현은 Code Generation |
