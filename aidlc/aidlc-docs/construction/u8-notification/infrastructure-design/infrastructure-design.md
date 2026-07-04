# U8 알림·마이페이지·설정 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · **U8 (1차 마감 유닛)** · M14 Notification (+각 모듈 설정 화면) · `features/notification`·`features/mypage`·`features/settings`
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U8은 SI를 참조만 하고 재정의하지 않는다.** 본 문서는 U8가 SI에 추가하는 **인프라 증분**(FCM 아웃바운드·서버 알림 스케줄 잡·알림함/삭제 감사 스키마·데이터 내보내기)만 기술한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md). 삭제 연쇄 상태 모델·법정 로그: **U1·U6 재사용(신규 없음)**.
> **마감 유닛 첨부**: U8은 1차 마지막 유닛이므로 **§8에 전 유닛(U1~U8) 인프라 총괄 부록**을, §5·§7에 **전 유닛 통합 관측성·컴플라이언스 종합**을 가볍게 첨부한다.

## 0. 요약 — U8 인프라 증분: 신규 AWS 리소스 0 (FCM 아웃바운드·시크릿·알림 스케줄 잡·RDS 스키마)

U8은 푸시·스케줄링·개인정보 마감 유닛이나, **신규 AWS 리소스는 0**이다 — SI가 이미 U8을 명시적 소비자로 예약해 두었기 때문이다: **NAT 아웃바운드(§2.1) + 스케줄 잡 프레임(§1.2 ShedLock) + FCM 시크릿(§6 예약분) + 아웃박스(§4) + IAM 태스크 역할(§7.1) + 알람 프레임(§8.2)**(SI §0 유닛 표: "U8 = FCM 아웃바운드, SNS/알림 스케줄링(서버 소유 D32 — DB 기반 잡)"). U8 인프라 결정은 넷 — **① FCM 벤더 아웃바운드·시크릿(`external/fcm`), ② 서버 알림 스케줄 잡(기존 스케줄러 — J4·J6·J9), ③ 알림함·삭제 감사 저장(RDS 스키마), ④ 데이터 내보내기(동기 직다운로드 — S3 미도입)** 이며, 전부 기존 리소스 내 확장/설정이다. **SES(내보내기 안내·시스템 메일)는 U1 재사용, 삭제 연쇄 상태 모델·법정 로그는 U1·U6 재사용.**

**SI 예약분 활성화 정합 확인**:
- SI §6 "FCM 서비스 계정 키(U8) Secrets Manager(예약)" → **활성화: `external/fcm` 항목 등록**(§1)
- SI §1.2 "U1~U8 스케줄러 잡 … U8 알림 스케줄링 포함" → **활성화: J4·J6·J9 ShedLock 잡**(§2)
- SI §4 "푸시 = FCM(U8): 서버 아웃바운드(NAT 경유) — 인프라 신규 요소 없음" → **활성화: FCM 아웃바운드**(§1)
- SI §8.2 A10 "U8 알림 스케줄링 포함"·A11·A12·A14 → **활성화: FCM 소스 편입**(§5)

**선결 경계**: FCM **Firebase 프로젝트·서비스 계정 키·APNs 인증 키(iOS)·발송 쿼터**는 **P9(스토어 개발자 계정·심사 요건)** 병렬 발급 대상이다(nfr-requirements.md §8.1). 본 문서는 아웃바운드 경로·시크릿 항목·스케줄 잡·저장 스키마까지 **벤더 중립으로 확정**한다(개발 비차단, 출시는 P9 게이트).

---

## 1. FCM 벤더 아웃바운드 · 시크릿 보관 (SI §2·§6 상속)

- **아웃바운드 경로**: ECS(프라이빗 앱 서브넷) → **NAT Gateway**(SI §2.1, 기존 1개) → FCM(`fcm.googleapis.com`, HTTP v1). `sg-app` 아웃바운드 443은 SI §2.2가 이미 허용("소셜 IdP·기상청·카카오·TMap·LLM·**FCM** 등 목적지 IP 가변 SaaS 예외 문서화") — **SG 변경 0**. U8 아웃바운드 목적지 문서화(egress 허용 목록):

| 목적지 | 어댑터(포트 · 소유) | 용도 |
|---|---|---|
| **FCM**(`fcm.googleapis.com`, HTTP v1 — iOS 포함 D12) | `FcmPort` · M14 | 서버 스케줄링 푸시 발송(리마인드·Plan-B·회고·등록 완료) — **P9 콘솔/인증서** |
| (재사용) Amazon SES | `MailDeliveryPort` · M1(U1) | 데이터 내보내기 안내·계정/보안 시스템 메일 — **U1 재사용**(SDK+IAM, 아웃바운드 신규 아님) |

- **시크릿(SI §6 예약분 활성화)**: FCM 서비스 계정 키(Firebase Admin JSON)를 **Secrets Manager** 항목으로 등록: **`external/fcm`**(SI §6이 "FCM 서비스 계정 키(U8) Secrets Manager(예약)"로 예약한 항목 활성화). ECS 태스크 정의 `secrets` ARN 참조 주입 — 코드·이미지·Terraform state 평문 0(NFR-U8-SEC-07·NFR-U1-SEC-11). APNs 인증 키는 FCM 콘솔 등록(P9 — Secrets Manager 아님). 로테이션은 Firebase 콘솔 키 갱신 주기 준수(P9 후 확정).
- **어댑터 타임아웃·재시도·dead letter**(NFR-U8-RES-02)는 앱 소관(Firebase Admin SDK + Resilience4j — 재사용) — 인프라는 SI §8.2 **A12(FCM 어댑터 실패율)·A11(FCM 쿼터 80%)·A10(발송 큐·dead letter 적체)** 알람을 제공(§5). SES는 SI §8.2 **A13(바운스·발송 실패)** 재사용.

---

## 2. 서버 알림 스케줄 잡 — 기존 ShedLock 스케줄러 재사용 (신규 리소스 0)

U8 잡은 SI §1.2 확정 모델(앱 프로세스 내 Spring `@Scheduled` + **PostgreSQL ShedLock 분산 락**)을 재사용한다 — **별도 발송 워커·큐·EventBridge 없음**(발송량 소량·저빈도, G142·NFR-U8-SC-01).

| 잡 | 소유 | 주기 (원격 구성 G195 유사) | 내용 | 근거 |
|---|---|---|---|---|
| J4 리마인드 발송 큐 | M14 | **분 단위** | due 도달 알림 스케줄 행 → 발송 파이프라인(토글→방해금지→중복 억제→인앱 적재→FCM) → SENT/FAILED | services.md §4 J4, D12·D32 |
| J6 삭제 유예 만료 처리 | M1 | **일 1회** | purgeDueAt 경과 `DEACTIVATED` 계정 → `PURGING` 전이 + `AccountDeletionExpired` → 모듈별 파기 핸들러 | services.md §4 J6, D18 |
| J9 알림함 보존 정리 | M14 | **일 1회** | 90일 초과 알림함 항목(D32) → 삭제 | services.md §4 J9, G99 |

- **중복 실행 방지**: J4는 **`SKIP LOCKED` 클레임 + dedupeKey + 발송 상태 전이**(`PENDING→CLAIMED→SENT/FAILED`)로 태스크 2개 이중 발송 배제(SI §1.2·services.md §4 J4). J6·J9는 상태/시각 기반 조회 + 조건부 전이 = 자연 멱등.
- **발송 파이프라인 실패 처리(J4)**: FCM 발송 재시도 3회 지수 백오프 → 최종 실패 시 **dead letter + 계측**. **인앱 알림함은 발송 전(파이프라인 4단계) 적재**되어 관측 보장(침묵 실패 금지, services.md §4 J4·§S5, PAT-U8-02).
- **삭제 연쇄(J6)**: 모듈별 파기 핸들러(M2·M4·M6·M8·M12·M13·M14) 실패는 **이벤트 재전달로 재시도**(at-least-once), 전 핸들러 완료 확인 후 계정 최종 삭제. 핸들러는 **기삭제 skip으로 멱등**(services.md §S6·§4 J6).
- **원격 구성**: 리마인드 오프셋(D-1·당일 8시·개별 N분)·방해금지 창(22~08시)·발송 상한은 **remote config**(NFR-U8-PERF-04·REL-04) — 코드 재배포 없이 조정(OPS-01).
- **전환 트리거**: 분당 발송량·FCM 호출량이 태스크·쿼터를 상시 압박하면 EventBridge Scheduler + 발송 워커 분리 재평가(SI §1.2 상속). **그 전까지 별도 워커 미도입**(과설계 금지).

---

## 3. 알림함·삭제 감사 저장 — RDS 스키마 확장 (Flyway)

- **저장소**: SI §3의 **기존 RDS PostgreSQL**(단일 `trippilot` DB·`public` 스키마) 재사용 — **신규 DB 인스턴스·클러스터 없음**. U1 Flyway 파이프라인으로 U8 마이그레이션 적용.
- **U8 신규 테이블(~7~8개)**: 알림 스케줄(멱등 키 tripId+slotId+type·due·발송 상태), 인앱 알림함(종류·읽음 상태·시각 인덱스·90일 보존), 종류별 토글(푸시/인앱 독립), 방해금지·알림 설정, FCM 기기 토큰(등록·무효 정리), dedupe 이력, dead letter, **계정 삭제 감사**(요청·유예 만료·모듈별 파기 완료·최종 삭제 이력). 마이페이지·설정·마케팅 동의 상태는 M2/M4 등 기존 테이블 재사용(집계 조회).
- **소유권·권한**: U8 일반 테이블(알림 스케줄·알림함·토글·토큰)은 `app_user` 표준 DML(accountId 스코핑, NFR-U8-SEC-06). **계정 삭제 감사 로그는 append-only 성격** — 삭제·변조 방지를 위해 U1 법정 로그 롤 모델(§5.3 A안·`app_user` UPDATE/DELETE REVOKE + IAM `s3:DeleteObject`/`logs:Delete*` Deny)을 감사 로그에도 적용(SECURITY-14·NFR-U8-OBS-02).
- **법정 보존 데이터 — U1·U6 재사용, U8 파기 제외**(D18·N2·NFR-U8-SEC-02):
  - **위치정보 법정 로그**(append-only, U1 생성·U6 기록 개시)·**동의 증적**(약관·위치·마케팅 버전/시각, U1)·**감사 로그**는 계정 삭제 연쇄에서 **분리 보관 유지** — U8 파기 핸들러의 접근 대상이 아니다(권한 분리, SI §3·§5.3·§7.1). GPS 폴리라인·위치 파생은 삭제 요청 즉시 파기(D34, services.md §S6 2단계).
- **볼륨**: DAU 1천·활성 여행 ≤1/사용자(D21)·90일 보존 → 알림함·스케줄은 일 수만 행 수준(NFR-U8-SC-02·SI §3) — gp3 20GB(오토스케일 100GB 상한) 내 충분. J9가 90일 초과 정리, 삭제 감사는 append 성격이나 저빈도.

---

## 4. 데이터 내보내기 · 마이페이지 집계 — 신규 리소스 0

- **데이터 내보내기(S9 — S3 미도입 권고)**: **동기 JSON 스트리밍 직다운로드**(services.md §S9 — "비동기 잡 없이 동기, 1차 규모 §6.8 충분"). 서버가 읽기 전용으로 각 모듈 퍼사드(M2·M6·M8·M12 메타·M13·M14)를 수집해 JSON 스트리밍 응답한다. **S3 임시 저장·비동기 생성·서명 URL은 미도입** — 근거: 텍스트만·사진 제외(G101)로 응답 크기 소량이고, 온디맨드·저빈도(rate-limit 시간당 1회)라 스트리밍 응답이 충분(NFR-U8-SC-03·과설계 금지). **전환 트리거**: 사진 포함 전체 아카이브(후속 G186)·대용량 시 S3 임시(U7 photos 버킷 프리픽스 재사용) + 비동기 생성 + 서명 URL 재평가.
- **마이페이지 집계(BFF성)**: 등록 숙소(M4)·여행 목록(M6)·스타일 분석(M13)·알림 설정(M14)을 **병렬 조회 집약**(신규 저장 없음 — 기존 유닛 데이터 재사용). 일부 실패해도 가용 카드만 부분 응답(NFR-U8-PERF-03·침묵 실패 금지). 컴퓨트는 기존 태스크 2개 흡수.
- **홈 카드 최종 통합**: 홈 대시보드 카드(여행·인기 장소·추억·활성 일정)는 U2가 레이아웃·빈 상태를, 후행 유닛이 데이터를 공급했고 **U8이 최종 통합 검증**(unit-of-work.md U2·U8). 신규 인프라 없음 — BFF성 조회 재사용.

---

## 5. 외부 알람 · 관측 (SI §8.2 구체화 + 마감 유닛 통합 종합)

- **U8 외부 API·잡 알람(SI §8.2 — U8이 FCM의 첫 소비자)**:

| 알람(SI 프레임) | U8 소스 | 임계 | 단계 |
|---|---|---|---|
| A11 외부 쿼터 80% | **FCM** 발송량 vs 문서화 쿼터(P9) | 80% | P2 |
| A12 어댑터 실패율·서킷 open | **FCM** 발송 실패율 | 실패율 >20%(5분) | P2 |
| A10 잡·큐 적체 | **리마인드 발송 큐(due age)·dead letter**·삭제/정리 잡 지연 | > 15분 / dead letter 누적 | P2 |
| A13 SES 바운스·발송 실패 | **SES**(내보내기 안내·시스템 메일 — U1 재사용) | U1 §4 임계 | P2 |
| A14 보안 이벤트 | **삭제·내보내기·재인증 실패·동의 변경** | 로그 메트릭 필터 | P1/P2 |

- 알람 라우팅은 SNS `trippilot-{env}-alerts` → OPS-02 프로세스(SI §8.2). FCM 쿼터 한도는 **P9 후 문서화**(NFR-U8-SC-01). 커스텀 메트릭(발송 성공/실패·dead letter 건수)은 SI §7.1 태스크 역할 `cloudwatch:PutMetricData` 허용 내.
- **삭제 배치 감사 관측(NFR-U8-OBS-02)**: 계정 삭제 요청·유예 만료·모듈별 파기 완료·최종 삭제를 감사 로그·커스텀 메트릭으로 계측(법정 보존 분리 증빙). 발송·삭제·내보내기 로그는 **PII 마스킹**(§0 PAT-OBS-01 상속) — 감사 로그와 운영 로그 분리.

### 5.1 마감 유닛 전 유닛 통합 관측성 종합 (G144 — 가볍게)

U8 마감 시점에 전 유닛 관측이 SI §8.2 알람 프레임에 수렴함을 확인한다(신규 관측 인프라 0):

| 관측 축 | 알람 | 소스 유닛 |
|---|---|---|
| 외부 어댑터 실패율·서킷(A12)·쿼터(A11) | A11·A12 | U3 지도 · U5 LLM · U6 기상청 · **U8 FCM** · (U1 SES=A13) |
| 잡·큐 적체(A10) | A10 | U6 폴링 · **U8 리마인드/삭제/정리 잡·dead letter** · J10 아웃박스(전 유닛) |
| 보안 이벤트(A14) | A14 | U1 인증 · **U8 삭제/내보내기/동의 변경** |
| 비용 예산(A15) | A15 | 전 유닛(U7 사진 스토리지·U5 LLM 비용·U8 FCM 발송) |

**종합 소견**: 전 유닛 외부 의존(지도·LLM·기상청·SES·FCM)과 잡(폴링·발송·삭제·아웃박스)의 실패·적체는 **A10~A14 단일 프레임 + 대시보드 1장(`trippilot-{env}-ops`)** 으로 종결된다. U8은 **FCM·발송·삭제·내보내기 소스만 편입**하며 신규 관측 인프라를 도입하지 않는다(G144·§6.7).

---

## 6. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 발송·삭제·정리·마이페이지 집계는 기존 태스크 2개 흡수(발송량 소량·저빈도) | §2·§4, G142·NFR-U8-SC-01 |
| 네트워크 VPC·NAT·ALB (SI §2) | **0** | 0 — **FCM NAT 아웃바운드**·`sg-app` 443·`/api/*` 재사용. 아웃바운드 허용 목록 문서화만(FCM 추가) | §1, SI §2.1·2.2 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 ~7~8개**(알림 스케줄·인앱 알림함·종류별 토글·방해금지·FCM 토큰·dedupe·dead letter·삭제 감사) — 기존 RDS·Flyway 확장. 법정 로그·동의 증적 U1·U6 재사용 | §3, unit-of-work.md U8 |
| **시크릿·KMS (SI §6)** | **0**(서비스) | **Secrets Manager 항목 1개**(`external/fcm` — SI §6 예약분 활성화). SES 키 없음(SDK+IAM) | §1, SI §6 |
| 메시징·비동기 (SI §4) | **0** | 0 — 서버 스케줄링은 기존 ShedLock, 발송/삭제 구독은 기존 아웃박스(이벤트 소비자 추가). **브로커·큐 미도입** | §2·§4, SI §1.2·§4 |
| 메일 (SI §4) | **0** | 0 — 내보내기 안내·시스템 메일 = U1 SES `MailDeliveryPort` 재사용 | §1·§4, SI §4 |
| 스토리지 S3 (SI §5) | **0** | 0 — 데이터 내보내기는 동기 직다운로드(S3 임시 미도입). 삭제 연쇄의 사진 S3 객체 파기는 U7 photos 버킷 대상(기존) | §4, SI §5 |
| 관측성 CloudWatch (SI §8) | **0** | FCM A11·A12·A10·A14·삭제 감사 소스 편입·대시보드 위젯 — 기존 프레임. 마감 유닛 통합 종합(§5.1) | §5, SI §8.2 |

**총계: 신규 AWS 리소스 0 / 변경 = RDS 스키마 확장(테이블 ~7~8개) + Secrets Manager 항목 1개(`external/fcm`) + FCM 아웃바운드 활성화 + 알림 스케줄 잡 3종(기존 스케줄러) + 삭제 연쇄 완성 + 데이터 내보내기 직다운로드.** U8은 푸시·개인정보 마감 유닛이지만 인프라는 SI가 예약한 프레임(NAT·스케줄러·시크릿·아웃박스·SES·알람) 내에서 종결된다 — **신규 외부는 FCM 아웃바운드 1종**이라 신규 인프라 표면이 없다(과설계 금지·발송량 소량 규모 정합).

---

## 7. 배포 — shared 파이프라인 재사용 (deployment-architecture.md 갈음)

U8은 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 재사용한다.

- **서버(M14 모듈 + 알림/삭제 감사 마이그레이션 + 삭제 연쇄 파기 핸들러)**: `server/**` 경로이므로 `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). 알림 스케줄·알림함·토큰·삭제 감사 테이블 추가는 Flyway forward-only(신규 테이블 = N-1 호환 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속. **FCM 시크릿(`external/fcm`, §1)은 배포 전 Secrets Manager 등록**(P9 후). 알림 스케줄 잡은 기존 ShedLock 스케줄러 흡수(신규 워크플로 없음).
- **모바일(`features/notification`·`features/mypage`·`features/settings`)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **설정·마이페이지·알림함 화면은 대부분 순수 JS/TS로 OTA 반영 가능**. 단 **`expo-notifications`(FCM 토큰 등록·푸시 수신 표시)는 네이티브 모듈**이다 — U1/U2 시점에 도입됐으면 순수 JS OTA, **신규 도입·FCM 설정 파일(google-services.json·GoogleService-Info.plist) 추가 시 EAS Build 재빌드 델타**(OTA 불가 — U3 지도 SDK와 동일 성격). config plugin·푸시 권한을 U8 초기 스파이크로 확인(tech-stack-decisions.md U8-C1). 딥링크 탭 스택 푸시(G7)는 U2 내비게이션 상속.
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. **리마인드 오프셋·방해금지 창·발송 상한·FCM 키는 remote config·시크릿으로 환경별 독립**(SI §9). dev는 FCM 테스트 프로젝트·검증 기기 토큰으로 발송 검증(prod 콘솔과 분리).

**U8 배포 델타**: 서버는 기존 워크플로 흡수(0 델타). 모바일은 **expo-notifications·FCM 설정 파일로 인한 네이티브 재빌드 가능성**(U3 지도 SDK와 동일 — 기존 도입이면 OTA)이 유일한 델타. 유일한 배포 전 조치는 **FCM 시크릿 등록(P9 후)** 이며, 인프라 파이프라인 변경은 없다.

---

## 8. 부록 — 전 유닛(U1~U8) 인프라 총괄 (마감 유닛 종합)

**U8은 1차 마지막 유닛이므로, 어느 유닛이 어떤 AWS 리소스를 활성화했는지 총괄한다.** SI가 정의한 공유 인프라 골격 위에서 각 유닛이 활성화·증분한 요소는 아래와 같다(전 유닛 인프라가 SI 프레임 내에서 종결됨을 확인):

| 유닛 | 인프라 역할 | 활성화·증분한 요소 | 신규 AWS 리소스 |
|---|---|---|---|
| **U1** | **기반 정의 유닛** | ECS Fargate·ALB·VPC·NAT·RDS Multi-AZ·Secrets Manager·KMS·CloudWatch·SES·ShedLock 스케줄러·아웃박스·S3 로그 버킷·IAM·OIDC·Terraform — **SI 전체 골격 활성화** | 전 기반(SI §1~§11) |
| **U2** | 앱 셸·부트스트랩 | remote config성 앱 구성(부트스트랩 API) | 0(DB 설정 1) |
| **U3** | 숙소·장소 데이터 | **NAT 아웃바운드**(TourAPI·카카오·네이버) · **Caffeine 인메모리 캐시(내부)** · 외부 쿼터 알람(A11) | 0(NAT·캐시 라이브러리 재사용) |
| **U4** | 여행 생성 | 추가 없음(기존 컴퓨트·DB) | 0 |
| **U5** | AI 일정 생성 | **NAT 아웃바운드**(LLM·TMap) · LLM 비용 계측 메트릭 · LLM API 키 시크릿 | 0(시크릿 항목·메트릭) |
| **U6** | 여행 중 실행·Plan-B | **NAT 아웃바운드**(기상청) · 서버 폴링 잡(ShedLock) · **위치 법정 로그 기록 개시**(append-only·월 스냅샷) · `external/kma` 시크릿 | 0(시크릿 항목·잡·스키마) |
| **U7** | 기록·회고 | **S3 photos 버킷 + CloudFront**(SI §5.2·§5.4 예약분 활성화) · `cloudfront/photo-signing-key` 시크릿 · 회고 LLM(U5 재사용) | **S3 photos·CloudFront**(SI 예약 활성화) |
| **U8** | **알림·마이페이지·설정(마감)** | **FCM 아웃바운드** · `external/fcm` 시크릿 · 알림 스케줄 잡(J4·J6·J9 ShedLock) · **계정 삭제 연쇄 완성**(법정 보존 분리) · 데이터 내보내기(직다운로드) · 홈 카드 최종 통합 | 0(시크릿 항목·잡·스키마) |

**총괄 소견**: 1차 8개 유닛 중 **실질 신규 AWS 리소스가 발생한 유닛은 U7(사진 S3+CloudFront) 하나뿐**이며, 나머지는 SI가 U1에서 정의·예약한 프레임(NAT 아웃바운드·ShedLock 스케줄러·Secrets Manager 항목·아웃박스·알람) 내에서 증분(시크릿 항목·스케줄 잡·RDS 스키마·아웃박스 이벤트 타입)으로 종결됐다. 이는 **모듈러 모놀리스(D04) + 공유 인프라 정본(SI) + 규모 정합(G142)** 설계가 의도대로 작동해, 유닛마다 인프라를 재설계하지 않고 예약분을 활성화하는 구조가 마감까지 유지됐음을 뜻한다. 후속 게이트 유닛(U9 어시스턴트·U10 커뮤니티 어드민+CloudFront·U11 WebSocket)에서 SI가 예약한 재평가 트리거(경량 pub/sub·WebSocket 스티키니스·어드민 오리진)가 발동될 예정이다(SI §0·§4).

---

## 9. 컴플라이언스 요약 (U8 Infrastructure Design 시점) + 마감 종합

| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-10 | **준수** | §1·§5 — FCM 어댑터 타임아웃·재시도·dead letter는 앱(Firebase Admin+Resilience4j), 인프라는 A12·A11·A10 알람. 폴백은 nfr-design PAT-U8-02(인앱 선적재) |
| RESILIENCY-09 | 준수 | §5 — FCM 쿼터 80%(A11)·발송 상한 원격 구성·오토스케일 SI §1 상속 |
| RESILIENCY-05·07 | 준수(증분) | §5 — FCM 실패율·서킷·발송 큐/dead letter 적체(A10·A12)·삭제 감사 메트릭·대시보드 편입 |
| RESILIENCY-01·02·08·11·12 | 상속 | Multi-AZ·백업·워크로드 분류(알림함 High·FCM Medium·삭제 High)는 SI·U1 — 발송/삭제는 기존 스케줄러 흡수 |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트는 전역 결정(GD-2~7)·deployment-architecture.md 상속. 리마인드/방해금지/상한 변경은 OPS-01(§7) |
| SECURITY-14 로깅·법정 보존 | **준수(핵심)** | §3 — 계정 삭제 감사 append-only(U1 롤 모델 재사용)·법정 보존 3종(위치 로그·동의 증적·감사) 분리 보관·앱 삭제 권한 없음(REVOKE+IAM Deny)·§5 삭제 감사 계측 |
| SECURITY-06·07 | 준수 | §1 — 최소 권한(FCM 시크릿 ARN)·NAT 경유·SG deny-by-default(SI §2.2·§7.1) |
| SECURITY-08 | 준수 | §3·§4 — 알림함·설정·마이페이지·내보내기·삭제 accountId 스코핑(nfr §5, IDOR) |
| SECURITY-05 | 준수 | §4 — 데이터 내보내기 재인증·rate-limit·설정 입력 검증(nfr-design PAT-U8-05, 앱) |
| SECURITY-10·12 | 준수(상속) | §1·§7 — FCM 키 Secrets Manager·평문 0·시크릿 스캔(U1 CI 상속) |
| SECURITY-11 | **N/A** | U8은 LLM 직접 호출 없음(회고 알림은 ReflectionReady 구독·U7 소유) |
| SECURITY-01·02·03·04·09·13·15 | 상속 | 암호화·중개 로깅·앱 로깅(발송/삭제 PII 마스킹)·헤더·하드닝·무결성·예외 처리는 SI·U1 정본 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — 발송 판정 순수 함수·삭제 상태 머신·재계산 멱등·알림함 보존은 Code Generation 실행(U8 DoD) |

**차단 소견 없음.** U8 신규 AWS 리소스 0(§6) — SI가 예약한 NAT·스케줄러·시크릿(`external/fcm`)·아웃박스·SES·알람 프레임 내에서 인프라가 종결된다. 신규 외부는 FCM 아웃바운드 1종이다. **비개발 선결 P9(스토어 개발자 계정·심사 요건·지원 연락처 N5)** 는 FCM 콘솔·인증서·심사를 확정하는 대상이며(§0·nfr-requirements.md §8.1), 본 인프라 산출물(아웃바운드 경로·시크릿 항목·스케줄 잡·저장 스키마·삭제 연쇄·내보내기)은 벤더 중립·fake로 준비되어 **개발 비차단**(출시는 P9 게이트)이다. **마감 유닛 종합(§8 전 유닛 인프라 총괄·§5.1 통합 관측성)** 으로 1차 인프라가 SI 프레임 내에서 종결됨을 확인한다.
