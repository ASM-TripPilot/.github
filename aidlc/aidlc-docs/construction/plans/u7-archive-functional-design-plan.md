# U7 기록·회고 — Functional Design 실행 계획 (Plan)

> 2026-07-04 · Construction/Functional Design · 대상 유닛: **U7 기록·회고**(M12 Travel Archive · M13 AI Reflection)
> 소유 스토리: Epic 8(US-E8-01~14) = **14 스토리**(actual·changelog 3계열 대조·사진 파이프라인·오프라인 동기화·LLM 회고·스타일 분석·공유 카드).
> 형식·깊이 기준: 직전 유닛 [u6-execution/functional-design](../u6-execution/functional-design/) 4종 산출물. 기술 중립(전송·저장·오브젝트 스토리지·CDN·LLM 티어·remote config 수치는 NFR/Infrastructure Design 소유).
> 정본 참조: unit-of-work.md §U7 · unit-of-work-dependency.md CP4(U6→U7 소비)·CP5(U7→U8 공급) · components.md M12·M13·§3.3(방문 상태 머신 — 영속 M12) · component-methods.md M12·M13 · services.md S4(여행 종료→회고)·S8(오프라인 기록 동기화) · requirements.md(D14·D18·D23·D34·§7.3 G72·G74·G75·G76·G77·G78·G132·G145·C11·G59·G13) · PRD 09(정본) · u6-execution/functional-design/domain-entities.md(VisitState·CP4 공급 계약).

---

## 0. 산출물·경로

| # | 산출물 | 경로 | 상태 |
|---|---|---|---|
| 1 | 실행 계획(본 문서) | `plans/u7-archive-functional-design-plan.md` | [x] |
| 2 | 도메인 엔티티 | `u7-archive/functional-design/domain-entities.md` | [x] |
| 3 | 비즈니스 규칙 카탈로그 | `u7-archive/functional-design/business-rules.md` | [x] |
| 4 | 비즈니스 로직 모델 + Testable Properties | `u7-archive/functional-design/business-logic-model.md` | [x] |
| 5 | 프런트엔드 컴포넌트 | `u7-archive/functional-design/frontend-components.md` | [x] |

---

## 1. 실행 체크리스트

### 단계 A — 입력 정본 흡수
- [x] U6 산출물 4종 정독(깊이·추적 ID·구조 기준 확립) — 특히 VisitState·CP4 공급 계약(VisitChecked·DayClosed·TripEnded·changelog diff·GPS 폴리라인)
- [x] unit-of-work.md §U7(포함·제외·DoD·리스크) + CP4(소비)·CP5(ReflectionReady 공급) 흡수
- [x] components.md M12 Travel Archive·M13 AI Reflection + §3.3(방문 상태 머신 — 기록 영속 경계) 흡수
- [x] component-methods.md M12·M13 메서드 계약 흡수
- [x] services.md S4(여행 종료→회고·당일 회고·멱등)·S8(오프라인 기록 동기화·레코드 단위 독립 커밋) 흡수
- [x] requirements.md D14·D18·D23·D34·N2·§7.3(G57/G132·G72·G74·G75/G145·G76·G77·G78·G59·C11·G13)·PRD 09 정본 흡수

### 단계 B — 도메인 엔티티(파일 2)
- [x] 엔티티 지도(M12·M13 + CP4 입력 소비 + CP5 출력 공급)
- [x] VisitRecord(actual 영속 — CP4 소비, M18 VisitState와 경계 FD-U7-01) — M12
- [x] ImpromptuVisit(즉석 방문 G77 — VisitRecord 특수화·좌표/카테고리 없음 '기타') — M12
- [x] MediaAttachment(Photo·Memo — 사진 메타·오프라인 큐·EXIF 제거 G75/G145·부분 실패 격리) — M12
- [x] ChangeLogEntry(통합 diff 스키마 G132 — Plan-B·어시스턴트·공동편집 공용 보관 정본·append-only) — M12
- [x] Reflection(초안/수정본 G78·부분 데이터·폴백 카드) — M13
- [x] TripSummary(전체 요약·지도 히어로·G72 혼합 이동거리) — M13
- [x] TravelStyleAnalysis(10곳 게이트 G76·임시 미리보기·취향 7종 축) — M13
- [x] ShareCard(3포맷·사진 유/무 레이아웃) — M13
- [x] OfflineSyncRecord(충돌 G74·레코드 단위 버전·합집합 병합) — M12
- [x] GpsTrack(영속 정본 — 수집 U6·CP4 소비·옵트인 철회 파기 D34/N2) — M12
- [x] 속성표·불변식(INV-xx)·상태 필드(syncState·uploadState·reflection status)
- [x] plan/actual/changelog 3계열 대조(D14) 구조 + CP4 소비 계약 + CP5 공급 계약(ReflectionReady)
- [x] 엔티티-스토리 추적 요약

### 단계 C — 비즈니스 규칙(파일 3)
- [x] 규칙 색인(그룹 A~G)·규칙 ID `BR-U7-xx`(전역 유일)
- [x] A. 방문 기록(actual·수정 가능 D23·즉석 방문 G77·숙소/날짜 귀속)
- [x] B. changelog 통합 보관(G132·재구성 정합·append-only·출처 유형 enum·후속 참조)
- [x] C. 사진·미디어(장소당 20장·클라 압축 5MB/2048px·EXIF 제거 G75/G145·오프라인 큐·재시도 3회·부분 실패 격리)
- [x] D. 오프라인 동기화 충돌(레코드 단위 버전·사진/메모 합집합 G74·조회 미보장 D24 경계)
- [x] E. 회고 생성(당일·전체 요약·C1 경유·실패 시 기본 카드 폴백·수정본 별도 G78·부분 데이터 명시·C11)
- [x] F. 스타일 분석(10곳 게이트 G76·임시 미리보기·이동거리 혼합 산출 G72·개인화 신호)
- [x] G. 공유 카드·GPS 귀속·파기(3포맷·사진 0장 폴백·GPS 파기 연동 D34/N2·캘린더)
- [x] 각 규칙 조건/동작/위반 시 처리/근거

### 단계 D — 비즈니스 로직 모델 + Testable Properties(파일 4)
- [x] FLOW-1 방문 기록·사진(CP4 VisitChecked 소비·오프라인 큐·부분 실패 격리)
- [x] FLOW-2 당일 회고·전체 요약 생성(여행 종료 S4·DayClosed/TripEnded·C1·폴백·멱등)
- [x] FLOW-3 오프라인 동기화·충돌 해소(레코드 단위 독립 커밋·합집합·사용자 선택)
- [x] FLOW-4 스타일 분석(10곳 게이트·임시 미리보기·개인화)
- [x] FLOW-5 공유 카드(진입·자동 구성·3포맷·사진 0장 폴백)
- [x] Testable Properties(PBT-01): changelog diff 누적 재구성=current 동등성(CP4 오라클·라운드트립)·actual 불변식·사진 큐 재시도 멱등·충돌 해소 결정성·병합 교환법칙·이동거리 혼합 결정성·스타일 10곳 게이트 단조·회고 수정본 보존
- [x] No-PBT 컴포넌트·커버리지 대조·CP4 통합 검증 시나리오

### 단계 E — 프런트엔드 컴포넌트(파일 5)
- [x] 화면 플로우(정본) + 컴포넌트 계층(`features/archive`·`shared/storage`)
- [x] 방문 기록·사진 첨부·즉석 방문·회고 열람/편집·전체 요약(지도 히어로)·스타일 분석·공유 카드·오프라인 동기화 상태
- [x] 계획/실제 대조(plan/actual/changelog 3계열)·캘린더 탐색
- [x] props/state·data-testid(`archive-{screen}-{role}`)·백엔드 능력(M12/M13)
- [x] shared/storage(오프라인 입력 로컬 큐 Δ6)·지도 재사용(U3/U5)·화면-스토리-능력 추적 매트릭스

---

## 2. 설계 결정 (FD-U7-xx) — 질문 대체

> 본 유닛은 정본 문서(components.md·component-methods.md·services.md S4·S8·PRD 09·requirements.md)가 결정을 이미 확정한 상태다. 아래는 Functional Design 단계에서 **정본을 인터페이스 계약·불변식 수준으로 상세화하며 내린 판단**을 명시(질문 없이 결정)한다.

| ID | 결정 | 근거·정본 |
|---|---|---|
| FD-U7-01 | **actual 영속 정본은 M12(U7) `VisitRecord`**이며 M18(U6) `VisitState`(실행 중 상태 머신)와 경계 분리. U7은 CP4 `VisitChecked` 이벤트를 구독해 actual을 생산·보관하고, 방문 상태 전이 강제는 U6 소유(재정의 금지). 종료 후 기록 편집은 M12가 허용하되 상태 재전이 아님(C11) | CP4, components §3.3, M18·M12, U6 INV-VISIT5 |
| FD-U7-02 | **plan/actual/changelog 3계열 대조(D14)** — plan(불변 스냅샷, U5/M8 소유·CP3 파생), current(가변, M8·M10 소유), **actual·changelog는 M12(U7) 소유 정본**. 회고·비교는 plan⊗actual 대조, changelog는 current 재구성 원장. U7은 plan/current를 참조·소비만(재정의 금지) | D14, ADR-0013, CP3, CP4 |
| FD-U7-03 | **changelog 통합 diff 스키마(G132)는 M12(U7)가 보관·열람·재생하는 정본** — U6(Plan-B)·(후속) M17(공동편집)·M16(어시스턴트)가 **공용**. 출처 유형 enum에 CoEdit·Assistant를 **예약**하고 스키마 버전 필드로 전방 호환. 핵심 속성: **diff 누적 재구성 = current 스냅샷 동등성** | G57/G132, CP4, U9~U11 공용 |
| FD-U7-04 | **오프라인 동기화는 레코드 단위 독립 커밋**(배치 전체 원자성 의도적 부재, S8) — recordId(클라 UUID) 멱등, 사진·메모 **추가는 합집합 자동 병합**(충돌 아님), **상태 필드(완료/스킵·시각)만 충돌 대상**·사용자 항목별 선택. 어떤 경로도 무손실(양쪽 보존) | G74, D24/Δ6, services S8 |
| FD-U7-05 | **사진 파이프라인은 방문 체크·메모와 부분 실패 격리** — 클라 압축(5MB/2048px)·서버 썸네일·원본 기기 보관, uploadState 머신(LOCAL→QUEUED→RETRYING(≤3)→FAILED/DONE), **EXIF(위치 포함) 클라 압축 시 제거**(위치는 GPS 옵트인 경로로만 별도 관리 D34 정합). 커뮤니티 게시 시 추가 EXIF GPS 제거는 U10(G185) 소관 | G75/G145, PRD 09-2, D34 |
| FD-U7-06 | **회고·요약·분석 LLM 호출은 전부 C1(U5 자산·상위 티어) 재사용** — 실패·타임아웃 시 **대체 산출 소유자는 M13**(기본 카드 폴백·통계만). 침묵 실패 금지: '데이터 없음'/누락 항목을 회고에 명시. 재구현 0(C1 계약만 소비) | D11, ADR-0011, components M13·C1 |
| FD-U7-07 | **회고 수정본은 원본 초안과 별도 저장**(editedContent), 수정본이 최종 표시본. 재생성은 수정본 존재 시 **덮어쓰기 경고 후 확인 시에만 교체**(G78). 종료 후 편집분 반영은 수동 '다시 생성'만(C11 — 자동 갱신 없음) | G78, C11, PRD 09-7 |
| FD-U7-08 | **스타일 분석 게이트는 누적 방문 장소 수 ≥ 10곳 단일 기준**(완료 여행 수 아님 — 충돌 시 방문 수 우선, PRD 09-9). 미달 시 온보딩 취향 기반 **임시 미리보기**(정식 아님 명시)+진행 게이지. 게이트는 방문 수에 대해 단조(비축소) | G76, PRD 09-9 |
| FD-U7-09 | **이동 거리는 혼합 산출(G72)** — GPS 동의 구간=실측 폴리라인 합산 + 미동의 구간=장소 간 거리 추정, 걸음 수=거리 환산 추정(추가 권한 없음 G59). 전부 **추정 표기 강제**. 구간 분할·순서 불변에 결정적 | G72, G59, ADR-0009 |
| FD-U7-10 | **GPS 발자취 영속 정본은 M12(U7)** — 수집·단순화는 U6, U7은 CP4 폴리라인을 귀속·보관·경로 비교 데이터화. **옵트인 철회·탈퇴 시 즉시 파기**(`purgeLocationData`, 30일 유예 없음), 법정 로그(수집·이용 사실)는 분리 보관 유지 | D34/N2, CP4, G55/G73, S6.2 |
| FD-U7-11 | **CP5 공급은 `ReflectionReady` 이벤트까지**(회고 완료 알림 — 발송은 U8). 마이페이지 통합·알림 발송·전체 아카이브 내보내기는 U8 소관(본 유닛 제외) | CP5, unit-of-work §U7 제외 |
| FD-U7-12 | **추적 ID 체계**: 엔티티 불변식 `INV-{도메인}` (REC/IMP/MEDIA/CLOG/REFL/SUM/STYLE/SHARE/SYNC/GPS), 규칙 `BR-U7-xx`, 속성 `U7-Pxx`, data-testid `archive-{screen}-{role}` | U6 관례 승계 |

---

## 3. 위험·주의 (Functional Design 한정)

- **changelog 통합 스키마(G132)가 후속 3유닛(U9~U11) 공용** — 초기 설계 실수 시 파급 최대. U10 공개 이력·U11 공동편집 이력 사용 시나리오까지 검토 범위에 포함, 출처 유형 enum·스키마 버전 필드 예약, **diff 재생 PBT(U7-P1)를 회귀 안전망**으로 고정.
- **오프라인 충돌 해소 UX 복잡도(G74)** — 자동 병합(합집합) 범위를 최대화하고 사용자 선택은 레코드 단위 최소화, 어떤 경로도 무손실(양쪽 보존)을 PBT(U7-P3)로 강제.
- **CP4 소비 스키마 정밀도** — VisitChecked·changelog diff·GPS 폴리라인을 U6 생산분과 정합 검증(재생 동등성 통합 테스트). U7 착수 게이트는 U6 CP4 스키마 확정.
- **actual/plan/current/changelog 4계열 혼동 금지** — actual·changelog는 U7 소유, plan/current는 U5/M8 소유 참조. 소유 경계를 불변식(FD-U7-02)으로 고정.

---

## 4. 완료 판정

- [x] 5개 산출물 생성 완료, 추적 ID(INV·BR-U7·U7-P·FD-U7·data-testid) 일관.
- [x] 14 스토리(E8) 전 커버 — 엔티티·규칙·플로우·화면 추적 매트릭스에 매핑.
- [x] CP4 소비(VisitChecked·DayClosed·TripEnded·changelog diff·GPS 폴리라인) + CP5 공급(ReflectionReady) 계약 명문화, C1 재사용 선언.
- [x] Testable Properties: 프롬프트 필수 7속성 전부 포함 + 확장(총 8속성).
