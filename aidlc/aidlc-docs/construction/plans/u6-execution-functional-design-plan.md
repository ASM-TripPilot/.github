# U6 여행 중 실행·Plan-B — Functional Design 실행 계획 (Plan)

> 2026-07-04 · Construction/Functional Design · 대상 유닛: **U6 여행 중 실행·Plan-B**(M18 Trip Execution · M9 Plan-B Detection · M10 Itinerary Recalculation · M11 Weather & Context)
> 소유 스토리: Epic 6(US-E6-01~03) + Epic 7(US-E7-01~13) = **16 스토리**(1차 유닛 최다·상태 머신 최다).
> 형식·깊이 기준: 직전 최중량 유닛 [u5-itinerary/functional-design](../u5-itinerary/functional-design/) 4종 산출물. 기술 중립(전송·저장·티어·지오펜스 SDK·remote config 수치는 NFR/Infrastructure Design 소유).
> 정본 참조: unit-of-work.md §U6 · unit-of-work-dependency.md CP3·CP4·CP5 · components.md M18·M9·M10·M11·§3.1·3.3 · component-methods.md M18·M9·M10·M11 · services.md S2·S4 · requirements.md(D10·D13·D14·D19·D23·D25·D27·D28·D34·D38·§7.3) · PRD 07·08(정본).

---

## 0. 산출물·경로

| # | 산출물 | 경로 | 상태 |
|---|---|---|---|
| 1 | 실행 계획(본 문서) | `plans/u6-execution-functional-design-plan.md` | [x] |
| 2 | 도메인 엔티티 | `u6-execution/functional-design/domain-entities.md` | [x] |
| 3 | 비즈니스 규칙 카탈로그 | `u6-execution/functional-design/business-rules.md` | [x] |
| 4 | 비즈니스 로직 모델 + Testable Properties | `u6-execution/functional-design/business-logic-model.md` | [x] |
| 5 | 프런트엔드 컴포넌트 | `u6-execution/functional-design/frontend-components.md` | [x] |

---

## 1. 실행 체크리스트

### 단계 A — 입력 정본 흡수
- [x] U5 산출물 4종 정독(깊이·추적 ID·구조 기준 확립) — 특히 CurrentItinerary·PlanSnapshot(CP3 입력)·C1·C2 재사용 계약
- [x] unit-of-work.md §U6(포함·제외·DoD·리스크) + CP3(소비)·CP4(공급)·CP5(TriggerFired) 흡수
- [x] components.md M18·M9·M10·M11 + §3.1 여행 상태 머신 + §3.3 방문 상태 머신(정본) 흡수
- [x] component-methods.md M18·M9·M10·M11 메서드 계약 흡수
- [x] services.md S2(Plan-B 재계획·10초 예산 단계표)·S4(여행 종료→회고) 흡수
- [x] requirements.md D10·D13·D14·D19·D23·D25·D27·D28·D34·D38·N2·§7.3(G53~C11)·G182·G106·G116 흡수
- [x] PRD 07(Plan-B 13스토리)·08(현장 이용 3스토리) 정본 흡수

### 단계 B — 도메인 엔티티(파일 2)
- [x] 엔티티 지도(M18·M9·M10·M11 + CP3 입력 + CP4 출력)
- [x] ActiveTripHub(활성 여행 허브 상태·진행률·현재/다음 슬롯·여유 시간) — M18
- [x] VisitState(방문 상태 머신 정본 §3.3 — 다음→도착확인대기→진행중→완료/스킵·체류 측정 D23) — M18
- [x] TripExecutionState(여행 실행 상태·종료 전이 D19·ACTIVE 하위 NORMAL/REST) — M18
- [x] RestMode(휴식 모드 G54) — M18
- [x] TriggerEvent + SuppressionState + SensitivitySetting + PollingSchedule(4종·상한/억제 G58/G195) — M9
- [x] ReplanSession(사유·방식·기준 위치·후보·확정·당일 잔여 C10) — M10
- [x] Alternative + UnplacedList(후보 2~3개·이월) — M10
- [x] WeatherData(ForecastCache·WeatherAlert — 기상청 D10) — M11
- [x] GpsTrail(발자취 폴리라인·옵트인 D34/G55/G73 — 수집 U6·영속 M12) — M12 계약
- [x] 속성표·불변식(INV-xx)·상태 전이표(방문·여행 실행·트리거 수명·재계획 세션)
- [x] CP3 소비 계약(§) + CP4 공급 계약(VisitChecked·DayClosed·TripEnded·changelog diff·GPS 폴리라인) + C1/C2 재사용 계약
- [x] 엔티티-스토리 추적 요약

### 단계 C — 비즈니스 규칙(파일 3)
- [x] 규칙 색인(그룹 A~G)·규칙 ID `BR-U6-xx`(전역 유일)
- [x] A. 도착 감지·방문 전이(지오펜스→확인 프롬프트만 D23·자동 기록 없음·체류 측정·스킵/취소·외부 지도 복귀 프롬프트)
- [x] B. 여행 실행 상태·종료(익일 00시+수동 D19·DayClosed·진행률)
- [x] C. 자동 트리거 하이브리드(4종·서버/클라 분리 D27·기상청 강수60%·특보 D10·상한/민감도/억제 G58/G195·판정 순수 함수 G116·위치 게이트 D34·허위 알림 금지)
- [x] D. 재계획 세션(진입·영향 분석 입력·후보 소싱 G53·2방식·대안 0개 폴백 3종·확정 재검증 G56·당일 잔여 C10·current 갱신 D14·changelog·전후 비교·숙소 고정 ADR-0006)
- [x] E. 휴식 모드(억제·심각 유지 G54·재개 재계산 G159)
- [x] F. 위치 폴백·GPS(**위치 3층 동의 동작 매트릭스 G182**·GPS 발자취 D34/G55/G73·걸음 수 G59·위치 수동 입력 US-E7-10)
- [x] G. 표시(소요시간 미표시 여유시간 G67/D25·거리 갱신 G65·외부 지도앱 G66)
- [x] 각 규칙 조건/동작/위반 시 처리/근거

### 단계 D — 비즈니스 로직 모델 + Testable Properties(파일 4)
- [x] FLOW-1 도착→방문 체크→체류 측정(CP4 생산)
- [x] FLOW-2 자동 트리거 감지→억제→알림
- [x] FLOW-3 재계획 세션(사유→후보 C1/C2 재사용→검증→확정→current 갱신 D14→changelog·10초 단계 예산)
- [x] FLOW-4 여행 종료 전이(자동/수동 D19·DayClosed)
- [x] FLOW-5 휴식 모드(진입·억제·재개 재계산)
- [x] Testable Properties(PBT-01): 방문 전이 가드·체류 결정성·트리거 상한/억제 멱등·민감도 단조·current만 변경·plan 불변·당일 잔여 범위·종료 전이 단일성·GPS 라운드트립·격자 변환 결정성·**C2 재계획 하드 제약 = U5 재사용 명시**
- [x] No-PBT 컴포넌트·커버리지 대조·하드 제약 대조

### 단계 E — 프런트엔드 컴포넌트(파일 5)
- [x] 화면 플로우(정본) + 컴포넌트 계층(`features/execution`·`features/planb`·`shared/location`)
- [x] 활성 일정 허브·도착 프롬프트·방문 체크·현장 장소 상세·다음 예정지/외부 길찾기
- [x] 재계획 플로우(사유·기준 입력·후보 비교·전후·확정)·휴식 모드
- [x] 계획vs실제 지도(GPS 발자취) — **U3 shared/map·U5 재사용**
- [x] props/state·data-testid(`execution-{screen}-{role}`)·백엔드 능력(M18/M9/M10/M11)
- [x] shared/location(권한 3층 상태 G182·포그라운드 수집)·화면-스토리-능력 추적 매트릭스

---

## 2. 설계 결정 (FD-U6-xx) — 질문 대체

> 본 유닛은 정본 문서(components.md·component-methods.md·services.md S2·PRD 07·08·requirements.md)가 결정을 이미 확정한 상태다. 아래는 Functional Design 단계에서 **정본을 인터페이스 계약·불변식 수준으로 상세화하며 내린 판단**을 명시(질문 없이 결정)한다.

| ID | 결정 | 근거·정본 |
|---|---|---|
| FD-U6-01 | **방문 상태 머신은 components.md §3.3을 정본으로 인터페이스 계약 수준 상세화**하고, M18이 상태 소유·전이 강제, actual 영속은 M12(U7)로 CP4 경계 분리. 엔티티명은 정본 `VisitState`(M18) 사용, 기록 영속 `VisitRecord`는 M12 소유임을 명기(혼동 방지) | components.md §3.3, M18·M12, CP4 |
| FD-U6-02 | **U6는 CP3(U5→U6)의 소비자이자 CP4(U6→U7)의 공급자**. plan/current 이원 구조·slotId·고정 블록은 U5 소유 입력으로 재정의하지 않고 참조·소비만. 재계획 결과는 **current에만** 반영, plan은 U5 불변 스냅샷 유지(INV-PLAN1 소비) | D14, CP3, CP4 |
| FD-U6-03 | **C1·C2는 U5 완성 재사용 자산**(재계획 사유 해석=C1 경량 티어, 하드 제약 4계열 재검증=C2). 본 유닛은 재사용 계약(입출력·하드 제약 재사용 선언)까지만 정의하고 구현체는 U5·NFR 소유. **재계획 검증은 U5 하드 제약(U5-P1)을 그대로 재사용**(중복 재구현 금지) | components.md C1·C2, DoD, U5-P1 |
| FD-U6-04 | **트리거 판정은 clock·외부 데이터(날씨·체류·이동) 주입 순수 함수로 분리**(G116) — 상한·억제·민감도 불변식을 결정적 시뮬레이션 테스트 대상으로. remote config 수치(시간당 2/일 8·±50%·임계 15/20분)는 NFR 소유, 본 문서는 불변식·단조성만 규정 | G116, G58/G195, G106 |
| FD-U6-05 | **위치 동의 3층(OS 권한 × 앱 내 법정 동의 × GPS 기록 옵트인) 동작 매트릭스를 business-rules.md에 명문화**(N2/G182 Functional Design 이관 결정) — 전 조합에 폴백 존재(권한 거부→수동 도착·위치 수동 입력, 옵트인 거부→발자취 미수집). 매트릭스 자체가 테이블 주도 테스트의 스펙 | N2, G182, unit-of-work §U6 리스크 |
| FD-U6-06 | **소요시간(internalDuration) 전면 미표시(D25)를 U6 표시 계층 전역 규칙으로 승계** — 다음 예정지 거리(G65)·여유 시간(G67 단순 차이)·대안 후보(distanceFromHere)·전후 비교 지표 전부 거리·시각만. internalDuration은 C2 이동 부등식·M9 지연 트리거 판정 내부 전용(U5 INV-TRAVEL1 소비) | D25/Δ1, G65, G67, U5 INV-TRAVEL1 |
| FD-U6-07 | **재계획 재정렬 범위=당일 잔여만(C10)**, 이월 방문지는 미배치 목록(UnplacedList)에 보관·사용자 날짜 선택 시 해당 일 재계산. current 갱신은 확정 TX(S2 11단계)에서만 — 그 전 어떤 후보도 일정 미반영 | C10, services S2.3, D14 |
| FD-U6-08 | **여행 종료 전이는 단일 규칙(종료일 익일 00:00 자동 + 수동 버튼) + 멱등**(자동/수동 경합 시 조건부 전이·이미 종료면 no-op, D19/Δ4). 종료가 `TripEnded`·`DayClosed`(일자 경계) 회고 트리거의 유일 소스 | D19/Δ4, services S4, CP4 |
| FD-U6-09 | **GPS 발자취 수집은 U6, 기록 귀속·영속은 M12(U7)** — U6은 옵트인 전제 포그라운드 저빈도 수집 + 단순화 폴리라인 생산(원시 좌표 파기)까지, CP4로 공급. 계획vs실제 지도는 U3 `shared/map`·U5 지도 뷰 재사용(신규 브리지 미생성) | G55/G73, D34, CP4, U3 shared/map |
| FD-U6-10 | **트리거 (a)날씨 계열은 M11 실패 시 침묵**(허위 알림 금지 — `DataUnavailable`≠'맑음'), 나머지 계열 정상. 폴링 실패는 침묵 실패 금지 예외가 아니라 "잘못된 정보로 행동 유도 금지" 원칙이며 실패율은 관측 계측 | ADR-0011, RESILIENCY-10, PRD 07-2 예외 |
| FD-U6-11 | **추적 ID 체계**: 엔티티 불변식 `INV-{도메인}` (HUB/VISIT/EXEC/REST/TRIG/REPLAN/ALT/WX/GPS), 규칙 `BR-U6-xx`, 속성 `U6-Pxx`, data-testid `execution-{screen}-{role}` | U5 관례 승계 |

---

## 3. 위험·주의 (Functional Design 한정)

- **상태 머신 3중첩**(여행 실행 ACTIVE 하위 · 방문 · 트리거 수명 · 재계획 세션)이 동시 작동 — 각 전이표를 독립 명문화하고 교차 가드(예: 여행 중이 아니면 재계획 진입 불가·휴식 모드 중 경미 트리거 억제)를 불변식으로 고정.
- **위치 동의 3층 × OS 권한 × GPS 정확도** 조합 폭증 — 매트릭스로 축약하고 전 조합 폴백 존재를 강제(빈 동작 금지).
- **CP4 공급 스키마 정밀도**가 U7 착수 게이트 — VisitChecked·changelog diff·GPS 폴리라인 필드를 U7 리스크 완화(스키마 버전 필드) 포함해 확정.
- 소요시간 미표시(D25)를 표시 계층 전역에서 **구조적으로** 보장(DTO에 소요시간 필드 부재 정적 검사) — U5 INV-TRAVEL1 재사용.

---

## 4. 완료 판정

- [x] 5개 산출물 생성 완료, 추적 ID(INV·BR-U6·U6-P·FD-U6·data-testid) 일관.
- [x] 16 스토리(E6 3 + E7 13) 전 커버 — 엔티티·규칙·플로우·화면 추적 매트릭스에 매핑.
- [x] CP3 소비·CP4 공급·CP5(TriggerFired) 계약 명문화, C1/C2 재사용 선언(U5-P1 하드 제약 재사용).
- [x] Testable Properties: 프롬프트 필수 8속성 전부 포함 + 확장(총 11속성).
