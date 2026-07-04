# U8 알림·마이페이지·설정 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · **U8 (마감 유닛)** · M14 Notification (+각 모듈 설정 화면) · `features/settings`·`features/mypage`·`features/notification`
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U8(FCM·서버 스케줄링 D32·알림함·계정 삭제 연쇄 D18·내보내기·위치 3층·고객지원 N5·마케팅 N8·홈 카드 최종 통합) · [stories.md](../../inception/user-stories/stories.md) **Epic 9(US-E09-01~14)** · [services.md](../../inception/application-design/services.md) **§S5(알림 스케줄링·발송)·§S6(계정 삭제 연쇄)·§S9(데이터 내보내기)**·§4(잡 **J4 리마인드·J6 삭제 유예·J9 알림함 보존**) · [requirements.md](../../inception/requirements/requirements.md) §6·**D12(FCM 단일)·D18(삭제 연쇄)·D32(서버 스케줄링)·D34(위치정보법)**·§7.4(**G99 알림함·G100 방해금지·G144 관측성·G198 푸시 인프라**)·G97·N5·N8·Δ10 · [shared-infrastructure.md](../shared-infrastructure.md)(SI — FCM 예약 §6·§0 U8 표·스케줄러 §1.2·아웃박스 §4·NAT §2.1·SES §4)
> 산출: `../u8-notification/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u8-notification/nfr-design/nfr-design-patterns.md`, `../u8-notification/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U8의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 실행으로 통합한다.** U8은 **1차 마감 유닛**(unit-of-work.md U8 — 알림·마이페이지·설정·홈 카드 최종 통합)이며 NFR 본질은 **정시 발송 신뢰성(서버 스케줄링 D32)·이중 알림 파이프라인 복원력(FCM 실패→인앱 선적재 D12)·계정 삭제 연쇄 완전성과 법정 보존 분리(D18)·개인정보 통제(내보내기·마케팅·위치 3층)** 다. **인프라·플랫폼 골격은 전역 정본을 상속**하고, U8 증분(FCM 아웃바운드·알림 스케줄 잡·알림함/삭제 감사 스키마·데이터 내보내기)만 소유한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U8에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | SI 헤더 · U1 nfr-design §2 | 재사용(AWS 서울·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조만** — SI §0 유닛 표: "U8 = FCM 아웃바운드, SNS/알림 스케줄링(서버 소유 D32 — DB 기반 잡)". 델타는 infrastructure-design.md 델타 표 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2 | Spring Boot Kotlin + PostgreSQL + RN Expo 재사용. U8은 **FCM 서버 SDK·서버 폴링 잡·알림함/삭제 스키마·내보내기** 지점만 구체화 |
| 푸시 인프라 정본 | requirements.md **D12·G198·G166** | **FCM 단일(iOS 포함 FCM 경유)·모든 알림 서버 스케줄링·인앱 알림함 자체 DB 단일 파이프라인** — 재선택 없음(G198=A) |
| 알림 스케줄링 정본 | requirements.md **D32** · services.md §S5 | 서버 스케줄링 통일·일정 변경 시 재계산·알림함 90일 보존·읽음 관리 상속 |
| 스케줄러·아웃박스 정본 | SI §1.2(ShedLock)·§4(DB 아웃박스, 브로커 없음) · services.md §4(J4·J6·J9) | 재사용 — **기존 스케줄러·아웃박스 위에 U8 잡(리마인드 발송·삭제 유예 만료·알림함 보존 정리) 추가** |
| SES(내보내기 링크·트랜잭션 메일) | SI §4 · U1 §4 | **재사용(신규 없음)** — 데이터 내보내기 안내·시스템 메일은 U1 SES 어댑터 재사용 |
| 계정 삭제 연쇄 정본 | requirements.md **D18·C4** · services.md §S6 | U1이 상태 모델·유예 배치 골격 생성 → **U8이 전 모듈 연쇄 파기 완성**(M14 알림·스케줄·FCM 토큰 파기 포함), 법정 보존 분리 유지 |
| 위치정보법 정본 | requirements.md **D34**·N2·G182 | 위치 3층 동의(OS 권한 × 법정 동의 × GPS 옵트인) **설정 제어** — U1 동의 모델·U6 법정 로그 재사용 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 · 활성 여행 ≤1/사용자(D21) | 리마인드 발송량·알림함·삭제 배치 전부 소량(과설계 금지 — 별도 워커·큐·푸시 SaaS 미도입) |

**설계 지배 원칙**: U8 NFR 본질은 **① 신뢰성(정시 발송 D32·중복 억제·일정 변경 재계산 멱등), ② 복원력(FCM 실패→인앱 알림함 선적재 보존 D12·삭제 배치 실패 재시도·내보내기 부분 실패 노출), ③ 보안·개인정보(삭제 연쇄 완전성·법정 보존 분리 D18·내보내기 접근 제어·마케팅 정보통신망법·위치 3층 D34)** 다. 신규 외부는 **FCM 아웃바운드 1종**뿐이며 SI가 예약한 NAT·시크릿(`external/fcm`)·스케줄러·아웃박스 프레임 내에서 종결된다(신규 AWS 리소스 사실상 0 — 과설계 금지). **마감 유닛이므로 전 유닛 인프라 총괄·통합 관측성(G144)·컴플라이언스 종합을 가볍게 첨부**한다.

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U8 범위 로드 (unit-of-work.md U8 — FCM·서버 스케줄링·종류별 토글·방해금지·알림함 90일·마이페이지·계정 삭제 연쇄·내보내기·위치 3층·고객지원 N5·마케팅 N8·홈 카드 최종 통합·**선결 P9 스토어 심사·지원 연락처**)
- [x] Epic 9 스토리 로드 (US-E09-01 등록 완료 알림·02 리마인드·03 Plan-B 알림·04 회고 완료·05 종류별 토글·인앱 알림함·06 마이페이지 숙소·07 여행 목록·08 스타일 분석·09 계정/내보내기·10 취향·11 위치 동의 관리·12 제휴 고지·13 고객지원·14 마케팅)
- [x] 알림 파이프라인 정본 로드 (services.md §S5 — 토글→방해금지→중복 억제→인앱 적재→FCM 발송→결과 기록, J4 발송 큐)
- [x] 삭제 연쇄 정본 로드 (services.md §S6 — 비활성화 TX·위치 즉시 파기·J6 유예 만료·모듈별 파기 핸들러·법정 보존 분리, D18·C4)
- [x] 내보내기 정본 로드 (services.md §S9 — 재인증·rate-limit·읽기 전용 수집·JSON 스트리밍 동기 다운로드·부분 실패 마커, G101/G186)
- [x] 확정 결정 로드 (D12 FCM 단일·D18 삭제 연쇄·D19 여행 종료·D25 소요시간 미표시·D32 서버 스케줄링·D34 위치정보법·G97 출발점 해제·G99 알림함·G100 방해금지·G198 푸시 인프라·Δ10 알림 종류)
- [x] 공유 인프라 정본 로드 (SI §0 U8 표·§2.1 NAT·§6 FCM 시크릿 예약·§1.2 ShedLock·§4 아웃박스·§7.1 IAM·§8.2 A10·A12·A14)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)
- [x] **마감 유닛 종합 입력 로드** (U1~U7 infrastructure-design.md 델타 표 — 전 유닛 인프라 총괄·관측성 A1~A15·컴플라이언스 종합용)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] **성능** — 알림 스케줄링(J4 분 단위 정밀도)·발송 지연(인앱 적재 발송 전 완료)·알림함 조회(페이지네이션·인덱스 300ms)·마이페이지 집계(BFF성 병렬·부분 응답)·설정 토글 즉시 반영
- [x] **정확성·신뢰성** — 정시 발송 스케줄링 정확도(D32)·중복 억제(dedupeKey 10분)·일정 변경 시 재계산(ItineraryChanged 전체 무효화 후 재계산·멱등 키)
- [x] **복원력** — FCM 실패→**인앱 알림함 선적재 보존**(D12·침묵 실패 금지)·재시도 3회 지수 백오프→dead letter·삭제 배치 실패 재시도(at-least-once·핸들러 멱등)·내보내기 부분 실패 노출
- [x] **보안·개인정보** — 계정 삭제 연쇄 완전성(전 모듈 파기)·법정 보존 분리(D18·위치 로그·동의 증적·감사)·데이터 내보내기 접근 제어(재인증·rate-limit·소유권)·마케팅 동의(정보통신망법·수집/철회·발송 없음 N8)·위치 3층(D34·G182 설정 제어)
- [x] **규모** — 리마인드 발송량(DAU 1천·활성 여행 ≤1)·알림함 90일 보존(J9)·삭제 배치 소량·과설계 금지(별도 워커·큐·푸시 SaaS 미도입)
- [x] **관측성** — 알림 발송 성공/실패(A12·dead letter)·삭제 배치 감사·잡 적체(A10) + **마감 유닛 전 유닛 통합 관측성 종합(G144)**
- [x] 미결·선결 — **P9(스토어 개발자 계정·심사 요건·지원 연락처 N5)** 연결, FCM 콘솔·푸시 인증서는 P9 병렬, 개발은 어댑터 fake로 비차단
- [x] 확장 규칙 컴플라이언스 요약(준수/상속/N/A + 근거) + **마감 유닛 컴플라이언스 종합**

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] **FCM SDK** — 서버 Firebase Admin SDK(HTTP v1)·토큰 관리(D12·무효 토큰 정리)·iOS 포함 FCM 경유·`external/fcm` 시크릿 (대안+사유)
- [x] **SES(내보내기 안내·시스템 메일)** — U1 SES 어댑터 재사용(신규 없음 명시)
- [x] **스케줄링** — 기존 ShedLock 재사용(J4 분 단위·J6 일 1회·J9 일 1회) vs 별도 워커/큐 (과설계 판단·대안+사유)
- [x] **인앱 알림함(RDS)** — 자체 DB 단일 파이프라인(D12) vs 푸시 SaaS 인박스 (대안+사유)
- [x] **삭제 배치** — 기존 스케줄러 + 아웃박스 이벤트 구독 파기 핸들러(신규 인프라 없음)
- [x] 각 선택 "재사용/증분" 명시 + 신규는 대안 1줄 + 사유 (신규 최소)

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U8-01 서버 스케줄링 정시 발송 + 일정 변경 재계산(D32 — 무효화 후 재계산·멱등)
- [x] PAT-U8-02 FCM 발송 + 인앱 선적재 이중 파이프라인(D12 — 침묵 실패 금지)
- [x] PAT-U8-03 방해금지·중복 억제(G100 — 인앱 적재·푸시 보류·Plan-B severe 예외)
- [x] PAT-U8-04 계정 삭제 유예·연쇄 배치·법정 보존 분리(S6/D18 — 상태 머신·at-least-once·멱등)
- [x] PAT-U8-05 데이터 내보내기 — 동기 스트리밍 직다운로드 + 비동기 전환 예약(S9)
- [x] PAT-U8-06 알림함 보존(90일 G99 — 읽음 상태·J9 정리)
- [x] 각 패턴: 문제/적용/U8 적용 지점/검증 기준(규칙 ID) 표기
- [x] 전역 상속 패턴(PAT-SEC·PAT-RES·PAT-OBS·PAT-PERF·PAT-U5-04 아웃박스) "U1·shared 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] FCM 아웃바운드 — 기존 NAT(SI §2.1)·`sg-app` 443 재사용·시크릿 **`external/fcm`**(SI §6 예약분 활성화)·토큰 관리는 앱 소관
- [x] 서버 스케줄러 잡 — **기존 ShedLock 재사용**(J4 리마인드 분 단위·J6 삭제 유예 만료 일 1회·J9 알림함 보존 일 1회)·별도 워커 미도입 근거
- [x] 알림함·삭제 감사(RDS) — 알림 스케줄·알림함·발송 상태·토글·방해금지·삭제 감사 스키마(기존 RDS·Flyway 확장)
- [x] 데이터 내보내기(S3 임시 or 직다운로드) — **동기 직다운로드 권고(규모 §6.8 정합)**·S3 미도입·전환 트리거
- [x] SI 대비 델타 표(신규 — FCM 시크릿·알림 스케줄 잡)
- [x] 배포 — shared deployment-architecture.md 재사용 갈음(서버 모듈 흡수·FCM 시크릿 등록·모바일 expo-notifications 네이티브 델타 검토)
- [x] **마감 유닛 종합 — 전 유닛 인프라 총괄 표 부록**(U1 기반·U3 캐시내부·U5 LLM아웃바운드·U6 기상청·U7 S3/CloudFront·U8 FCM)
- [x] 컴플라이언스 요약 + 마감 유닛 통합 관측성·컴플라이언스 종합

### 2.6 검증·마감

- [x] 추적 ID(NFR-U8-xx·PAT-U8-xx·SECURITY·RESILIENCY·PBT·D·G·GD·US-E09·CP·S5·S6·S9·J) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 규모 정합 점검 — 리마인드 발송량·활성 여행 ≤1 기준(과설계 금지: 별도 발송 워커·큐·푸시 SaaS·내보내기 S3 미도입) 확인
- [x] **마감 유닛 종합 첨부 확인**(전 유닛 인프라 총괄·통합 관측성 G144·컴플라이언스 종합)
- [x] 산출물 4종 생성 완료

---

## 3. 선결 과제 연결 (P9 — §8)

| 과제 | 시점 | 본 계획의 비차단 대응 |
|---|---|---|
| **P9 앱스토어·플레이스토어 개발자 계정 및 심사 요건(지원 연락처 등)** | **U8 출시 전 필수**(심사 리드타임 고려 U1 기간 중 착수 권장 — unit-of-work.md U1 참고) | (1) **FCM 콘솔·푸시 인증서(APNs 인증 키·Firebase 프로젝트)** 는 P9와 병렬 발급 — `FcmPort` 어댑터를 **fake로 개발·테스트 선행**(D37 계층 분리), 실키는 `external/fcm` 시크릿 주입. (2) **US-E09-13 고객 지원 연락처(이메일 문의 링크)·정책 문서 재열람·오픈소스 라이선스**는 앱스토어 심사·위치정보법 상시 열람 요건(N5) — 설정 화면에 선결. (3) 스토어 심사 대응(개인정보 처리방침 URL·데이터 삭제 경로 노출)은 U8 산출물(삭제·내보내기·정책 재열람)과 직결 |

**연결 원칙**: P9는 **출시·심사 게이트**이며, 본 NFR·Infra 산출물(FCM 어댑터 중립·`external/fcm` 시크릿 프레임·설정 정책 재열람·삭제/내보내기 경로)은 **비차단**이다(개발은 fake·플레이스홀더로 진행, 실키·실문안은 P9 후 주입).

---

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u8-notification/nfr-requirements/nfr-requirements.md` | U8 NFR 요구 — 성능·정확성/신뢰성·복원력·보안/개인정보·규모·관측성(마감 유닛 통합 종합) + P9 선결 |
| `../u8-notification/nfr-requirements/tech-stack-decisions.md` | FCM SDK·SES 재사용·ShedLock 스케줄링·인앱 알림함(RDS)·삭제 배치 스택 (대안+사유·신규 최소) |
| `../u8-notification/nfr-design/nfr-design-patterns.md` | U8 패턴 6종(정시 발송·이중 파이프라인·방해금지/중복 억제·삭제 연쇄·내보내기·알림함 보존) + 전역 상속 표기 |
| `../u8-notification/infrastructure-design/infrastructure-design.md` | U8 인프라 증분(FCM 아웃바운드·스케줄 잡·알림함/삭제 스키마·내보내기) + SI 델타 표 + **전 유닛 인프라 총괄 부록** + 배포 갈음 |

## 5. 다음 단계

- U8 Functional Design(선행) 또는 Code Generation: 본 산출물의 알림 파이프라인(토글→방해금지→중복 억제→인앱 적재→FCM)·스케줄 재계산 멱등·삭제 연쇄 상태 머신·데이터 내보내기·마이페이지 집계·위치 3층 설정 제어·홈 카드 최종 통합·CP5(TriggerFired·ItineraryConfirmed/Changed·ReflectionReady·TripEnded·StayRegistered 소비자) 계약을 구현. **전 유닛 완료 후 Build and Test(§CONSTRUCTION 마감)로 진행** — E2E 종단 흐름(숙소 저장→등록→일정 생성→재계획→알림·기록) 누적 실행.
- 본 계획 §1의 전역 결정 상속은 후속 게이트 유닛(U9 어시스턴트·U10 커뮤니티·U11 공동편집)에서도 동일 재사용(재질문 금지). FCM 파이프라인·삭제 연쇄·법정 보존 분리 패턴은 후속 알림/모더레이션의 참조 정본이다.
