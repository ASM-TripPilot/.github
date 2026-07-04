# U1 기반·계정·온보딩 — 논리 컴포넌트 (Logical Components)

> 2026-07-04 · CONSTRUCTION · NFR Design · U1
> 목적: NFR 설계 패턴([nfr-design-patterns.md](./nfr-design-patterns.md) PAT-x·OPS-x)을 실어 나르는 **기술 중립 논리 컴포넌트**의 목록·책임·통합 지점·장애 모드를 확정한다. **AWS 매핑은 참조(GD-1=AWS 전제)이며, 제품·수치 확정은 Infrastructure Design 소관**(nfr-requirements.md §8 PD-1~6·9). 애플리케이션 내부 모듈 경계(M1·M2·C3·common)는 components.md·NFR-U1-MT-01이 정본이고, 본 문서는 그 모듈들이 의존하는 **NFR 실현 계층**을 다룬다.
> 표기: 추적 ID(SECURITY-xx·RESILIENCY-xx·NFR-U1-xx·D·G·FLOW·PD), 패턴 참조(PAT-x).

## 컴포넌트 지도 (ASCII)

```text
[모바일 클라이언트 (RN/Expo)]
   │ HTTPS (TLS 1.2+, SECURITY-01)
   ▼
[LC-1 API 게이트웨이/로드밸런서 계층]  ←─ shallow /health (PAT-RES-02)
   │ Multi-AZ 분산 (RESILIENCY-08)
   ▼
[애플리케이션 인스턴스 (app 모듈, 2+ AZ · 롤링 배포 GD-5)]
   ├─ [LC-2 인증 필터] ──── JWK 셋 ──── [LC-6 시크릿 저장소]
   ├─ [LC-3 레이트리미터] ─┐
   ├─ M1 Auth · M2 Profile · C3 Moderation
   │      │ 단일 DB 트랜잭션 (상태 변경 + outbox insert + 감사 이벤트)
   │      ▼
   ├─ [LC-4 세션/토큰 저장]──┴─ PostgreSQL (Multi-AZ) ── [LC-7 법정 로그 저장소(append-only)]
   ├─ [LC-10 아웃박스 릴레이] ──→ 도메인 이벤트 구독자 / [LC-8 감사 이벤트 파이프라인]
   └─ [LC-5 이메일 발송 큐] ──→ 외부 메일 서비스 (PD-1)
                                        │
[LC-9 메트릭·알람] ← 전 컴포넌트 계측 ──┘ → OPS-02 알람 라우팅
```

---

## LC-1 API 게이트웨이/로드밸런서 계층 `[전역]`

- **책임**: 단일 진입점 — TLS 종단(SECURITY-01), 다중 AZ 트래픽 분산(RESILIENCY-08), shallow 헬스체크(`/health`) 기반 비정상 인스턴스 축출·롤링 배포 중 점진 교체 게이트(GD-5, PAT-RES-02), 접속 로그(중개 계층 로깅 — SECURITY-02의 실현 지점), 요청 크기·프로토콜 1차 방어선.
- **통합 지점**: 클라이언트 HTTPS ↔ 애플리케이션 인스턴스(2+ AZ). GitHub Actions 배포 워크플로(GD-3)가 인스턴스 등록/해제를 통해 롤링을 구동. deep 헬스체크(`/health/deep`)는 **연결하지 않는다**(관측 전용 — PAT-RES-02 원칙).
- **장애 모드·대응**: (a) 단일 AZ 장애 → 헬스체크 축출 + 잔여 AZ로 재분배(정적 안정성 — 제어 플레인 조작 없이 지속, RESILIENCY-08) — OPS-03 SC-1 검증 대상. (b) 계층 자체 장애 → 관리형 서비스의 자체 다중 AZ에 위임(자체 구현 금지). (c) 배포 중 신 인스턴스 미준비 → readiness 미충족 시 트래픽 미유입(NFR-U1-AV-05).
- **AWS 매핑(참조)**: ALB(다중 AZ) + ACM 인증서. WAF 부가 여부는 Infrastructure Design.

## LC-2 인증 필터 `[전역 — U1 스캐폴드 소유]`

- **책임**: 전 요청 deny-by-default 인증(PAT-SEC-01) — Bearer 토큰 서명·exp·iss·aud 무상태 검증(PAT-PERF-01, NFR-U1-SEC-17), 공개 화이트리스트 관리(NFR-U1-SEC-16 전수 목록), 인증 주체(accountId) 확립 → 객체 수준 인가(IDOR 차단, NFR-U1-SEC-18)·계정 상태별 접근 축소(DELETION_PENDING — FLOW-6 S4)의 입력 제공, 보안 헤더·일반화 오류 응답(SECURITY-04·15).
- **통합 지점**: 필터 체인 3·5단(tech-stack §2.3), JWT 서명 검증 키는 JWK 셋(kid 롤오버 — LC-6·PD-3), 401/403은 감사 이벤트로 LC-8에 발행, 상관 ID(1단)와 MDC 공유(PAT-OBS-01).
- **장애 모드·대응**: (a) 서명 키 조회 불가(기동 시) → fail-fast 기동 실패(무검증 서빙 금지). (b) 검증 실패·변조 토큰 → fail-closed 401 + 계측(폴백 없음 — 보안 게이트). (c) DB 열화 → 영향 없음(무상태 — 검증 경로에 DB 의존 부재가 이 컴포넌트의 존재 이유, NFR-U1-AV-01).
- **AWS 매핑(참조)**: 애플리케이션 내장(Spring Security 6.4) — 별도 관리형 인증 서비스 미사용(tech-stack §2.3 대안 비교 확정).

## LC-3 레이트리미터 (로그인·재발송·공개 엔드포인트) `[전역 — 정본 U1]`

- **책임**: 2축 제한(PAT-SEC-03) — 계정 단위 점진 지연(2·4·8·16·32초→15분 잠금, NFR-U1-SEC-07)과 출처 단위 상한(로그인 분당 10·시간당 100, 재발송 분당 1·일 5회 G22, NFR-U1-SEC-08). 429+retryAfter 응답, 잠금·지연 발동의 감사 이벤트 발행, 수치의 외부화 설정 수용(NFR-U1-MT-04).
- **통합 지점**: 필터 체인 4단(공개 인증 엔드포인트 한정), 카운터 정본은 LC-4와 동일 PostgreSQL(인스턴스 로컬 금지 — NFR-U1-SEC-09·AV-02), 후속 유닛의 고비용 엔드포인트(U5 LLM 사용자별 상한 등)가 동일 메커니즘 재사용, 발동 메트릭은 LC-9.
- **장애 모드·대응**: (a) 카운터 저장소 열화 → 로그인 자체가 이미 불가한 상황(동일 DB)이므로 독립 장애 모드 없음 — 단 카운터 갱신 실패가 로그인 성공 경로를 차단하지 않게 기록 실패는 계측 후 통과(가용성 우선, 방어는 해시 비용이 하한 유지). (b) 설정 오적용(과도 잠금) → 외부화 설정 즉시 조정(재빌드 불요) — OPS-02 완화 절차. (c) 공유 캐시 승격 필요(규모 성장) → PD-9 재평가 시 저장 어댑터만 교체(정본 인터페이스 유지).
- **AWS 매핑(참조)**: 1차는 애플리케이션+PostgreSQL. 승격 시 ElastiCache(Redis) — PD-9 대기.

## LC-4 세션/토큰 저장 (무상태 액세스 + 리프레시 저장소) `[U1]`

- **책임**: 이원 구조 — (a) **액세스 토큰(1h)은 저장하지 않는다**(무상태 검증, PAT-PERF-01), (b) **RefreshSession(90일, 기기 체인)은 PostgreSQL 정본** — tokenHash만 저장(원문 금지, NFR-U1-SEC-06), 회전 체인(chainId·rotatedAt) 관리, 재사용 감지 판정 데이터 제공(FLOW-5, PAT-SEC-02), 무효화 매트릭스 집행(NFR-U1-SEC-05), 90일 만료분 정리 배치(NFR-U1-SC-03).
- **통합 지점**: M1 토큰 발급·갱신·revoke 유스케이스 전용(타 모듈 직접 접근 금지 — 모듈 경계 NFR-U1-MT-01), 회전은 조건부 UPDATE 원자 연산, 재사용 감지 시 LC-8에 즉시 이벤트, 클라이언트 대응부는 expo-secure-store(C-3 — OS 보안 저장소, NFR-U1-SEC-28).
- **장애 모드·대응**: (a) DB 열화 → 로그인·갱신 불가(Critical 본질), 단 기발급 액세스 토큰의 조회성 기능은 최대 1시간 지속(우아한 저하 — PAT-RES-03 표) → Multi-AZ 페일오버 복구(OPS-03 SC-2). (b) 동시 갱신 경합 → 단일 비행 큐(클라이언트)+조건부 UPDATE(서버)의 2중 방어 — 오탐 체인 무효화 방지. (c) 정리 배치 실패 → 기능 영향 없음(만료 검증은 조회 시점) — 볼륨 증가만, 배치 실패 알람(LC-9 배치 행).
- **AWS 매핑(참조)**: RDS for PostgreSQL Multi-AZ(PD-5 — PITR·백업 보존은 Infrastructure Design).

## LC-5 이메일 발송 큐 (비동기·재시도) `[U1 — MailDeliveryPort 뒤]`

- **책임**: 인증 링크·비밀번호 재설정·재발송(FLOW-2 S6~S8)의 **비동기 발송** — API는 접수 확인만 동기 응답(§1.2, p95 300ms), 실발송은 큐 소비 잡이 수행. 지수 백오프+지터 재시도, 최대 시도 후 실패 확정(사장 큐 상당 — 실패 레코드 보존·계측), 발송 서비스 서킷(PAT-RES-01 표), 도달 SLO 5분/95% 계측, 재발송 상한(G22)은 LC-3이 선행 차단.
- **통합 지점**: M1 가입·재설정 유스케이스 → 발송 요청 레코드는 **업무 트랜잭션과 원자적으로 적재**(아웃박스와 동일 원리 — 커밋 후 소비, services.md §0.2 "외부 호출은 TX 밖"), MailDeliveryPort 어댑터(PD-1 제품 대기 — 로컬/CI는 fake 캡처, D37), 실패율·큐 적체는 LC-9.
- **장애 모드·대응**: (a) 메일 서비스 장애 → 서킷 open·잡 대기, 가입 트랜잭션은 이미 성공(미인증 유지+재발송 UI — FLOW-2 E4, 침묵 실패 금지). (b) 큐 적체 → 알람(P2) + 도달 SLO 위반 계측. (c) 중복 발송 위험(재시도) → 발송 요청 ID 멱등 키로 소비자 멱등 처리, 구 토큰 무효화(INV-E2·U1-P12)가 보안측 안전망. (d) 인스턴스 재시작 → 요청 레코드가 DB 정본이므로 유실 없음(at-least-once).
- **AWS 매핑(참조)**: Amazon SES(발송, PD-1) + 큐 실체는 1차 DB 폴링 잡(규모 §2에서 충분), 승격 시 SQS — Infrastructure Design.

## LC-6 시크릿 저장소 `[전역]`

- **책임**: 전 시크릿(소셜 클라이언트 시크릿·Apple p8·JWT 서명 키(JWK 셋)·메일 API 키·DB 자격 증명)의 저장·기동 시 주입·로테이션(PAT-SEC-05, NFR-U1-SEC-11). JWT 키는 kid 롤오버(신 키 발급·구 키 검증 병행 창) 지원 구조(PD-3).
- **통합 지점**: `app` 모듈 설정 로더(외부화 설정 인터페이스 — 코드가 아는 것은 설정 키 이름뿐), LC-2(검증 키)·SocialOAuthPort 4종·MailDeliveryPort·DataSource, GitHub Actions는 OIDC 임시 자격 증명으로 접근(장수 키 금지), Apple p8 서명 JWT의 6개월 갱신은 OPS-01 변경 기록 대상.
- **장애 모드·대응**: (a) 기동 시 시크릿 조회 불가 → **fail-fast 기동 실패**(묵시 기본값·빈 시크릿 서빙 금지) — 롤링 배포가 신 인스턴스 미준비로 안전 정지(구 버전 유지). (b) 런타임 로테이션 중 불일치 → JWK 병행 창으로 무중단(kid 매칭), 기타 시크릿은 재기동 주입(롤링으로 무중단). (c) 유출 의심 → 즉시 로테이션 + 해당 자격 증명 의존 세션/키 무효화 — OPS-02 보안 인시던트 절차.
- **AWS 매핑(참조)**: AWS Secrets Manager(로테이션) 또는 SSM Parameter Store + AWS KMS(키 계층·저장 암호화 SECURITY-01) — PD-2·3 확정은 Infrastructure Design.

## LC-7 법정 로그 저장소 (append-only) `[U1 — 테이블·권한 모델 소유]`

- **책임**: 두 계열의 불변 증적 — (a) **위치정보 법정 로그**(N2·D34): 수집·이용·제공 사실 확인자료, 6개월+ 보존, **애플리케이션 DB 역할에 DELETE/UPDATE 권한 없음(DB 권한 분리로 강제)**, 실기록 발생은 U6부터이나 테이블·권한·기록 인터페이스는 U1 소유(NFR-U1-SEC-24). (b) **동의 증적**(ConsentRecord — N3·N8): append-only 행 추가만(GRANT/REVOKE 이력, INV-C1·C2), 계정 파기 시에도 법정 보존 분리(D18, FLOW-6 S5).
- **통합 지점**: Flyway 마이그레이션에 권한 분리 DDL 포함(S-9 — 마이그레이션 리뷰가 U1 DoD·OPS-01 체크 항목), 기록 경로는 INSERT 전용 리포지토리(JPA UPDATE 경로 원천 배제 — tech-stack §2.7 단서), M1 동의 유스케이스(FLOW-3)·U6 위치 수집(후속), GPS 옵트인 철회 연쇄의 PURGE 기록(U1-P16), 백업 최우선 대상(§3.1 — OPS-03 SC-3 복원 검증 항목).
- **장애 모드·대응**: (a) 기록 실패 → **동반 업무 트랜잭션 전체 롤백**(증적 없는 상태 변경 금지 — 동의 없이 진행되는 온보딩 차단, N2·N3 fail-closed). (b) 권한 DDL 회귀(누군가 앱 역할에 DELETE 부여) → Testcontainers 권한 검증 테스트가 PR CI에서 차단(NFR-U1-MT-02). (c) 보존 기간 내 용량 증가 → 파티셔닝 여지 확보(NFR-U1-SC-03), 아카이브 저장소는 PD-4와 함께 Infrastructure Design.
- **AWS 매핑(참조)**: 동일 RDS 내 분리 스키마+역할 분리(1차), 장기 아카이브는 S3(Object Lock 검토) — Infrastructure Design.

## LC-8 감사 이벤트 파이프라인 `[전역 — 이벤트 목록 U1]`

- **책임**: SECURITY-14 감사 이벤트 전수(NFR-U1-SEC-26 5분류 — 계정 생명주기·인증·동의·권한·프로필)의 수집→저장(90일+)→**알림 라우팅**(NFR-U1-SEC-27 임계 (a)~(e), 재사용 감지는 건당 즉시)을 담당. 행위자·시각·결과·사유 코드·상관 ID 구조화(PAT-OBS-02), PII 마스킹 규칙 동일 적용(accountId만).
- **통합 지점**: 발행측은 M1·M2·C3 유스케이스(FLOW-1~7 전 분기)와 LC-2(401/403)·LC-3(잠금) — **아웃박스(LC-10) 경유로 업무 TX와 원자적**(SECURITY-13), 소비측은 감사 저장소+임계 평가→OPS-02 심각도 라우팅(P1/P2/P3), Spring Security 이벤트 훅(AuthenticationEventPublisher — tech-stack §2.3)이 인증 계열 발행 지점.
- **장애 모드·대응**: (a) 파이프라인 소비 지연 → 이벤트는 아웃박스에 보존(유실 없음 — at-least-once), 지연 알람(LC-9 릴레이 행). (b) 알림 채널 장애 → 이벤트 저장은 독립적으로 지속(저장이 정본, 알림은 파생) — 채널 복구 후 미처리 임계 재평가. (c) 이벤트 폭주(공격 상황) → 임계 평가는 집계 기반(건당 처리 아님 — 재사용 감지만 건당), 파이프라인 벌크헤드(PAT-PERF-02 잡 풀 분리).
- **AWS 매핑(참조)**: CloudWatch Logs(저장·메트릭 필터)+SNS(알림) 상당 — PD-4 확정은 Infrastructure Design.

## LC-9 메트릭·알람 `[전역]`

- **책임**: PAT-OBS-03 정의표의 수집·평가·발화 — 골든 시그널(§1.2 p95 목표 대조), 인증 보안, 외부 의존(서킷 상태·실패율), 자원·용량(풀·argon2 병렬도·쿼터 80%, RESILIENCY-09), 복원력 자세(단일 AZ 운전·백업 실패, RESILIENCY-07), 배치·잡(정리 배치·아웃박스 릴레이 지연), 클라이언트(부트스트랩 3초 발동률 G5). 운영 대시보드 1면 제공.
- **통합 지점**: 애플리케이션 메트릭 노출 엔드포인트(스캐폴드 — Micrometer 상당), deep 헬스체크(`/health/deep`)를 관측 입력으로 소비(PAT-RES-02 — LB 아님), 전 알람은 OPS-02 라우팅 표(P1/P2/P3)로 접속, APM·수집 제품은 PD-4 대기.
- **장애 모드·대응**: (a) 관측 스택 장애 → 서비스 기능 무영향(관측은 데이터 평면 밖) — 단 "눈이 먼 상태"이므로 관측 스택 자체 헬스가 P2. (b) 알람 폭풍 → 심각도 계층(P3는 비긴급 집계)과 임계 조정(외부화)으로 완화 — 반복 소음은 OPS-02 회고의 시정 조치. (c) 임계 미조정 오탐/미탐 → 초안 임계임을 명시(PAT-OBS-03) — 운영 초기 조정 주기를 OPS-01 변경 기록으로 추적.
- **AWS 매핑(참조)**: CloudWatch Metrics/Alarms/Dashboards + SNS, APM은 X-Ray 상당 — PD-4 확정은 Infrastructure Design.

## LC-10 트랜잭셔널 아웃박스 릴레이 `[전역 — common/core 소유, services.md §0.2 정합]`

- **책임**: 도메인 이벤트·감사 이벤트의 신뢰 가능한 발행 — 소유 모듈의 애그리거트 상태 변경 + 아웃박스 레코드 INSERT를 **단일 DB 트랜잭션**으로 묶고(transactional outbox), 커밋 후 릴레이가 **at-least-once** 발행. 동일 애그리거트 키 내 순서 보장(교차 애그리거트 순서 무보장), 이벤트 봉투에 이벤트 ID·상관 ID 포함 — **전 구독자 이벤트 ID 기준 멱등 처리 필수**(services.md §0.2 정본 그대로).
- **통합 지점**: `common/core`가 계약(이벤트 타입·발행 인터페이스·봉투) 소유 — U1은 계약+릴레이 골격을 산출하고 최초 실소비는 U3(StayRegistered)부터(business-logic-model.md "속성 없는 컴포넌트" 표 근거), U1 자체 소비자는 LC-8(감사)와 삭제 연쇄 파기 훅(FLOW-6 S5 — 훅 계약 골격), 릴레이 지연·미발행 잔량은 LC-9 계측.
- **장애 모드·대응**: (a) 릴레이 프로세스 중단 → 이벤트는 아웃박스 테이블에 보존(유실 없음), 재기동 시 미발행분 재개 — 릴레이 지연 알람(P2). (b) 중복 발행(at-least-once 본질) → 구독자 멱등 처리로 흡수(계약 위반은 설계 리뷰 차단). (c) 발행 대상(구독자) 실패 → 재시도+실패 격리(특정 구독자 실패가 타 구독자 발행을 차단하지 않음 — 벌크헤드). (d) 아웃박스 테이블 증가 → 발행 완료분 정리 배치(LC-9 배치 행 계측).
- **AWS 매핑(참조)**: 1차는 인프로세스 릴레이+PostgreSQL(모놀리스 D04 — 외부 브로커 불요), 워커 분리 시 SQS/SNS 재평가(services.md §0.2 전환 계획과 정합) — Infrastructure Design.

---

## 컴포넌트 × 규칙 커버리지 요약

| 컴포넌트 | 핵심 규칙 | 검증 스테이지 |
|---|---|---|
| LC-1 게이트웨이/LB | RESILIENCY-06·08, SECURITY-01·02, GD-5 | Infrastructure Design |
| LC-2 인증 필터 | SECURITY-04·08·15, NFR-U1-AV-01·SEC-16~19 | Code Generation (아키텍처 테스트 CI) |
| LC-3 레이트리미터 | SECURITY-11·12, NFR-U1-SEC-07~09, RESILIENCY-09 | Code Generation (U1-P13·P14) |
| LC-4 세션/토큰 저장 | SECURITY-12, D36, NFR-U1-SEC-05·06, NFR-U1-AV-02·03 | Code Generation (U1-P2) |
| LC-5 이메일 발송 큐 | RESILIENCY-10, G22, ADR-0011, PD-1 | Code Generation + Infrastructure Design |
| LC-6 시크릿 저장소 | SECURITY-12(NFR-U1-SEC-11), SECURITY-01, PD-2·3 | Infrastructure Design |
| LC-7 법정 로그 저장소 | N2·N3·D18, SECURITY-14, NFR-U1-SEC-23~25 | Code Generation (Testcontainers 권한 검증) |
| LC-8 감사 파이프라인 | SECURITY-13·14, NFR-U1-SEC-26·27 | Code Generation + Infrastructure Design |
| LC-9 메트릭·알람 | RESILIENCY-05·07·09, G5, PD-4 | Infrastructure Design |
| LC-10 아웃박스 릴레이 | services.md §0.2, SECURITY-13 | Code Generation |

**미결 유지 항목**: PD-1~6·9(제품·수치)는 본 문서의 "AWS 매핑(참조)"대로 Infrastructure Design에서 확정한다. 본 문서의 책임·장애 모드 계약은 제품 선택과 무관하게 유효하다(Port/어댑터 격리 — components.md §2-2).
