# U7 기록·회고 — Infrastructure Design

> 2026-07-04 · CONSTRUCTION · U7 (M12 Travel Archive · M13 AI Reflection + `features/archive`·`shared/storage` 오프라인 로컬 큐)
> 공유 인프라 정본은 [shared-infrastructure.md](../../shared-infrastructure.md)(이하 SI)다. **U7은 SI를 참조하되, SI가 "U7 예약"으로 명시한 스토리지·CDN 자원(§5.2 사진 버킷·§5.4 CloudFront)을 실제 활성화한다** — U1~U6과 달리 **신규 AWS 리소스가 실질적으로 발생하는 첫 유닛**이다. 그 외 컴퓨트·네트워크·DB·시크릿·관측성·회고 연산(C1)은 SI·U1·U5 정본을 재사용한다. NFR 근거: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), 스택: [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md). 배포 파이프라인 정본: [deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md).

## 0. 요약 — U7 인프라 증분: 신규 리소스 = S3 사진 버킷 + CloudFront (SI 예약분 활성화)

U7은 **사진 저장·서빙이 프로젝트 최초로 등장**하는 유닛이다. SI는 이를 예견해 **§5.2에 `trippilot-{env}-photos` 버킷을 "버킷·정책만 선생성, 파이프라인은 U7"으로**, **§5.4에 CloudFront 배포를 "U7 예약"으로**, **§0 유닛 표에 "U7 = 사진 S3 버킷 + CloudFront(§5.2 — 예약분 활성화)"** 로 예약해 두었다. **본 문서는 이 예약분을 활성화(파이프라인·정책·배포 확정)한다.** U7 인프라 결정은 다섯 — **① S3 사진 버킷 활성화, ② CloudFront 배포(OAC·서명 URL), ③ 이미지 처리(클라 전처리 + 서버 썸네일), ④ 사진 메타·changelog·GPS 파기(RDS·U1 append-only 재사용), ⑤ 회고 LLM(U5 C1 재사용)** 이다.

**SI 예약분 활성화 정합 확인**(§6 델타 표 재확인):
- SI §5.2 `trippilot-{env}-photos`(버저닝 + 이전 버전 30일 파기) → **활성화**(§1)
- SI §5.4 CloudFront + S3 OAC(서명 URL·TLS1.2+·표준 로깅·기본 루트 객체 없음) → **활성화**(§2)
- SI §6 "사진 버킷은 U7에서 CMK 필요성 재평가" → **재평가 완료: aws/s3 관리형 유지**(§5)
- SI §11.2 storage 모듈 "photos/CloudFront는 자리만 생성(U7 활성화)" → **Terraform storage 모듈 활성화**(§7)

---

## 1. S3 사진 버킷 활성화 (SI §5.2 예약분 · SECURITY-01)

- **버킷**: `trippilot-{env}-photos`(SI §5.2 예약 — 이름·기준선 상속). U1 storage 모듈이 자리만 생성한 버킷의 **정책·수명주기·이벤트를 U7에서 확정**.
- **기준선(SI §5.1 상속·전 버킷 공통)**:

| 항목 | 값 | 근거 |
|---|---|---|
| 퍼블릭 액세스 | **전면 차단**(계정 레벨 Block Public Access ON) | SECURITY-09, NFR-U7-SEC-03 |
| 버저닝 | **ON** | SI §5.1·5.2, RESILIENCY-12(사진 버저닝 백업성) |
| 기본 암호화 | **SSE-KMS**(`aws/s3` 관리형 — §5 CMK 재평가 결과) | SECURITY-01, NFR-U7-SEC-01 |
| 전송 암호화 | **TLS-only 버킷 정책**(`aws:SecureTransport=false` Deny) | SECURITY-01 |
| 오리진 접근 | **CloudFront OAC만 허용**(버킷 정책 — 직접 접근 차단) | SECURITY-09, §2 |
| 수명주기 | **버저닝 + 이전 버전 30일 파기**(SI §5.2). 삭제 마커·미완료 멀티파트 업로드 7일 정리 | SI §5.2, 비용(NFR-U7-SC-01) |
| 접근 경로 | **S3 Gateway 엔드포인트**(SI §2.1 — NAT 요금 우회·프라이빗) | SECURITY-07 |

- **오브젝트 구조(프리픽스)**: `{accountId}/{tripId}/{placeId}/{photoId}`(원본 압축본) + `{...}/thumb/{photoId}`(썸네일). accountId 프리픽스로 IAM·소유권 경계(NFR-U7-SEC-02). 태스크 역할 `s3:PutObject/GetObject`는 photos 프리픽스 한정(SI §7.1 상속) — **로그 버킷 아님(DeleteObject Deny는 log-archive·alb-logs 대상, photos는 이전 버전 30일 수명주기가 정리)**.
- **presigned PUT**(tech-stack §6): 서버가 발급하는 presigned URL은 **content-length ≤5MB·content-type 이미지 조건**(SECURITY-05) + 짧은 만료 + 단일 객체 스코프. 원본은 기기 보관(G75)이므로 **원본 업로드 경로 없음**(압축본만).

---

## 2. CloudFront 배포 활성화 (SI §5.4 예약분 · SECURITY-02·08·09)

- **배포**: 사진 서빙 CDN — `trippilot-{env}-photos` 오리진, **OAC**(Origin Access Control)로 버킷 직접 접근 차단(SI §5.4). SI가 예약한 원칙(TLS 1.2+ viewer·표준 로깅·기본 루트 객체 없음)을 확정.

| 항목 | 값 | 근거 |
|---|---|---|
| 오리진 접근 | **OAC** — CloudFront만 S3 오리진 접근(버킷 퍼블릭 차단 §1) | SECURITY-09, NFR-U7-SEC-03 |
| **접근 제어** | **서명 URL(Signed URL)** — 소유권 검증 후 짧은 만료 URL 발급(사진=사적 자산) | **SECURITY-08**, NFR-U7-SEC-02, PAT-U7-03 |
| viewer 프로토콜 | **HTTPS only, TLS 1.2+**(redirect-to-https) | SECURITY-01, SI §5.4 |
| 로깅 | **표준 로깅 활성**(S3 로그 프리픽스) | SECURITY-02, SI §5.4 |
| 기본 루트 객체 | **없음**(디렉터리 리스팅 불가) | SECURITY-09, SI §5.4 |
| 캐시 정책 | 썸네일·압축본 캐시(서명 URL 쿼리 파라미터 캐시 키 제외 표준 정책) — 이미지 로딩·오리진 부하 최소화 | NFR-U7-PERF-04·05 |
| 서명 키 | **CloudFront 키 그룹**(퍼블릭 키 등록) + **개인 키 = Secrets Manager**(§5) | PAT-SEC-05, SI §6 |

- **서명 URL 흐름**: 클라 사진 조회 요청 → 서버가 **accountId 소유권 검증**(NFR-U7-SEC-02) → CloudFront 서명 URL(짧은 만료) 발급 → 클라가 CloudFront에서 이미지 로딩. 비소유자·만료 URL은 접근 불가(SECURITY-08). 목록은 썸네일 URL, 상세는 압축 원본 URL.

```text
[클라] --조회 요청(인증)--> [ECS: 소유권 검증(accountId)] --발급--> [CloudFront 서명 URL(짧은 만료)]
                                                                          |
[클라] <-------------- 이미지(TLS) ---------------- [CloudFront(엣지 캐시)] --OAC--> [S3 photos(퍼블릭 차단)]
```

**텍스트 대안**: 클라이언트가 인증된 조회 요청을 보내면 ECS가 accountId 소유권을 검증한 뒤 짧은 만료의 CloudFront 서명 URL을 발급한다. 클라이언트는 그 URL로 CloudFront 엣지 캐시에서 이미지를 TLS로 받으며, CloudFront는 OAC로만 퍼블릭 차단된 S3 photos 버킷 오리진에 접근한다. 비소유자·만료 URL은 거부된다.

---

## 3. 이미지 처리 — 클라 전처리 기본 + 서버 썸네일 (신규 컴퓨트 0)

- **클라 전처리(기본)**: 압축(5MB/2048px NFR-U7-MEDIA-01) + **EXIF GPS 제거**(§4 tech-stack — G168 원본 GPS 기기 이탈 방지). 원본 기기 보관(G75).
- **서버 썸네일(앱 ECS 동기 — tech-stack §5)**: 클라가 압축본을 S3에 직업로드(§1 presigned) 후 **attach 확정 API** 호출 → 서버가 S3에서 압축본 읽어(S3 게이트웨이 엔드포인트) **EXIF 재검증·재제거(심층 방어)·매직 바이트·크기 검증** → 썸네일 생성·S3 업로드 → 메타 저장.
- **Lambda·EventBridge 미도입**(SI §1.2·§4 정합·G142): 썸네일 트리거는 **attach API**(S3 이벤트 아님)이므로 서버리스 파이프라인 불요. 저빈도(DAU 1천·첨부 시점)로 기존 ECS 태스크 2개가 흡수(NFR-U7-SC-03) — 신규 컴퓨트 0.
- **전환 트리거**: 사진 처리량이 ECS API p95에 간섭하거나 월 처리 수만 장 초과 시 Lambda 썸네일 분리 재평가(NFR-U7-SC-04 — SI §1.2 잡 분리 트리거 정합).

```text
[클라] 압축·EXIF제거 ─presigned PUT─> [S3 photos: 압축본]
   │                                        ▲(썸네일 write)
   └─ attach 확정 API ─> [ECS: EXIF 재검증·매직바이트·크기 → 썸네일 생성] ─┘
                              │
                              └─> [RDS: 사진 메타(스토리지 키·썸네일 키)]
```

**텍스트 대안**: 클라이언트가 압축·EXIF 제거한 이미지를 presigned URL로 S3에 직접 올린 뒤 attach 확정 API를 호출한다. ECS는 S3에서 압축본을 읽어 EXIF를 재검증·재제거하고 매직 바이트·크기를 검증한 후 썸네일을 생성해 S3에 다시 쓰고, 사진 메타(스토리지 키·썸네일 키)를 RDS에 저장한다. S3 이벤트가 아닌 API 트리거이므로 Lambda가 필요 없다.

---

## 4. 사진 메타·changelog·GPS 파기 — RDS 스키마 확장 (U1 재사용)

- **저장소**: SI §3 **기존 RDS PostgreSQL**(단일 `trippilot` DB·`public` 스키마) 재사용 — **신규 DB 인스턴스·클러스터 없음**. U1 Flyway 파이프라인(배포 전 일회성 `app_migrate` 태스크)으로 U7 마이그레이션 적용.
- **U7 신규 테이블(~7개, unit-of-work U7 산출물)**: actual 방문 기록, 사진 메타(스토리지 키·썸네일 키·촬영 시각·귀속), changelog(통합 diff G132), 회고(초안/수정본·유형), 스타일 분석 결과, 동기화 버전(레코드 단위 G74), 이동 거리 집계(G72). 전부 `app_user` 표준 DML(SI §3 롤 모델) — append-only REVOKE 대상 아님(일반 CRUD).
- **위치정보 법정 로그 — U1 append-only 재사용**(SI §5.3·NFR-U7-SEC-07): GPS 처리 사실 로그는 **U1이 생성한 append-only 테이블**(`location_legal_log`)에 기록. `app_user` UPDATE/DELETE REVOKE·IAM 명시 Deny 2중 강제 유지(신규 테이블 아님).
- **GPS 파기(D34/N2·PAT-U7-06)**: GPS 옵트인 철회·탈퇴 시 위치 데이터(발자취 폴리라인·GPS 귀속) **즉시 파기**. 파기는 철회 이벤트 즉시 처리 또는 ShedLock 잡(SI §1.2). **법정 로그는 파기 대상 아님**(6개월+ 보존·append-only 잔존 — 파기 잡 권한은 위치 데이터 테이블 한정, 법정 로그 미접근). 계정 삭제 연쇄(D18)에서도 법정 보존 집합만 잔존(U8 연쇄 정합).
- **볼륨**: DAU 1천·활성 여행 ≤1/사용자(D21)로 기록·메타는 소량 — SI §3 gp3 20GB(오토스케일 100GB) 내 충분(NFR-U7-SC-01). 사진 바이너리는 S3(§1), RDS는 메타·키만.

---

## 5. 회고 LLM·시크릿·KMS (U5 재사용 + 증분 항목)

- **회고 LLM = U5 C1 재사용**(신규 인프라·벤더 0): M13 회고·요약·스타일 분석은 U5 C1 LLM Gateway를 상위 티어(D11)로 호출(component-dependency §2.13·NFR-U7-REF-01). **LLM API 키는 U5에서 등록한 Secrets Manager 항목**(SI §6 — U5 예약분) 재사용 — U7 신규 시크릿 없음. NAT 아웃바운드(SI §2.1)·A11·A12 알람(SI §8.2)도 U5 상속.
- **CloudFront 서명 키(신규 시크릿 항목)**: 서명 URL 개인 키를 **Secrets Manager 신규 항목** `cloudfront/photo-signing-key`(SI §6 프레임 내 항목 추가 — 신규 서비스 아님)로 보관. 퍼블릭 키는 CloudFront 키 그룹 등록. ECS 태스크 역할 `secretsmanager:GetSecretValue`(명시 ARN — SI §7.1 상속) 주입 — 코드·이미지·Terraform state 평문 0(PAT-SEC-05).
- **KMS — CMK 재평가 결과(SI §6 단서 종결)**: SI §6이 "사진 버킷은 U7에서 CMK 필요성 재평가"를 남겼다. **재평가 결론: `aws/s3` 관리형 키 유지(CMK 미도입).** 근거: 서명 URL·OAC 접근 제어는 CloudFront/S3 정책 계층에서 종결되어 KMS 키 정책 분리가 추가 통제를 주지 않고, CMK는 월 $1/키 + 운영 부담(G142). **전환 트리거**: 사진 데이터에 크로스 계정 공유·키 정책 분리·규정 요구 발생 시 CMK 도입(SI §6 원칙 정합).

---

## 6. SI 대비 델타 표

| 계층 (SI 절) | 신규 AWS 리소스 | 변경 | 근거 |
|---|---|---|---|
| 컴퓨트 ECS (SI §1) | **0** | 0 — 썸네일 생성·기록 조회는 기존 태스크 2개 흡수(저빈도·attach 트리거) | §3, NFR-U7-SC-03, G142 |
| 네트워크 VPC·NAT·ALB (SI §2) | **0** | 0 — S3 게이트웨이 엔드포인트(기존 SI §2.1)·`sg-app` 재사용. LLM NAT 아웃바운드 U5 상속 | §1·§5 |
| 데이터베이스 RDS (SI §3) | **0** | **테이블 ~7개**(actual·사진 메타·changelog·회고·스타일·동기화 버전·이동 거리) — 기존 RDS·Flyway 확장. 법정 로그 U1 재사용 | §4, unit-of-work U7 |
| **스토리지 S3 (SI §5.2)** | **활성화 — `trippilot-{env}-photos` 버킷**(SI 예약분: 버킷 자리→정책·수명주기·이벤트 확정) | 버저닝·SSE-KMS·수명주기(30일)·OAC 정책·presigned | **§1, SI §5.2 예약 활성화** |
| **CDN CloudFront (SI §5.4)** | **활성화 — CloudFront 배포**(SI 예약분: 배포 자리→OAC·서명 URL·로깅 확정) | OAC·서명 URL·TLS1.2+·표준 로깅·키 그룹 | **§2, SI §5.4 예약 활성화** |
| 시크릿·KMS (SI §6) | **0**(서비스) | **Secrets Manager 항목 1개**(`cloudfront/photo-signing-key`) + **CMK 재평가=aws/s3 유지**. LLM 키 U5 재사용 | §5, SI §6 |
| 관측성 CloudWatch (SI §8) | **0** | 사진 스토리지·CDN 비용 A15 편입·업로드 실패율·회고 폴백율 커스텀 메트릭·CloudFront 로깅 | §0 PAT-OBS-03, SI §8.2 A15 |
| 회고 LLM (U5 C1) | **0** | 0 — U5 C1·LLM 키·NAT·A11/A12 전부 재사용 | §5, U5 |

**총계: 신규 AWS 리소스 = S3 photos 버킷 활성화 + CloudFront 배포 활성화(SI §5.2·§5.4 예약분) / 그 외 = RDS 스키마 확장(~7 테이블) + Secrets Manager 항목 1개 + 관측성 편입.** U7은 **사진 스토리지·CDN이 실질 증분**인 첫 유닛이나, SI가 예약해 둔 자리를 활성화하는 것이므로 아키텍처 이탈 없이 종결된다(회고·컴퓨트·DB·LLM은 재사용). **SI 예약분 활성화 정합 확인 완료**(§0).

---

## 7. 배포 — shared 파이프라인 재사용 + S3/CloudFront Terraform 모듈 활성화

U7은 **별도 배포 아키텍처 문서를 만들지 않는다** — 전역 배포 파이프라인 정본([deployment-architecture.md](../../u1-foundation/infrastructure-design/deployment-architecture.md), U1 스캐폴드 생성)을 재사용한다. 인프라 증분은 **Terraform storage 모듈 활성화**로 반영한다.

- **인프라(Terraform)**: SI §11.2 `modules/storage`가 "photos/CloudFront는 자리만 생성(U7 활성화)"로 예약한 부분을 **활성화** — S3 photos 버킷 정책·수명주기·CloudFront 배포·OAC·서명 키 그룹·Secrets Manager 항목을 storage/security 모듈에 추가. `terraform plan` diff가 PR 리뷰 산출물(SI §11.1 변경 관리)·apply는 수동 승인. 적용 순서: security(서명 키)→storage(버킷·CloudFront). 프로바이더·모듈 버전 고정(SECURITY-10 IaC).
- **서버(M12·M13 모듈 + 사진 메타·changelog 마이그레이션)**: `server/**` 경로 — `server-ci.yml`·`server-deploy.yml` **수정 없이** 커버(U1이 "유닛별 모듈 추가는 워크플로 수정 불요"로 완성). Flyway forward-only(신규 테이블 추가 = N-1 호환 "자유", OPS-01). 롤백=버전 고정 재배포(GD-4)·롤링(GD-5) 상속. 서명 키는 배포 전 Secrets Manager 등록.
- **모바일(U7 화면 + `shared/storage` 오프라인 큐 + 클라 이미지 압축/EXIF)**: `apps/mobile/**` 경로 — `mobile-ci.yml`(tsc·ESLint·Jest+fast-check) 커버. **클라 이미지 압축·EXIF strip 라이브러리가 네이티브 모듈이면 EAS Build 재빌드 필요**(순수 JS면 OTA 가능 — Code Generation에서 라이브러리 선정 시 확정, U3 카카오 SDK와 동일 OTA vs 네이티브 구분 상속). `shared/storage` 로컬 큐 저장 엔진(SQLite/MMKV류)도 동일 판정.
- **환경**: dev(자동)·prod(승인 게이트) 승격 상속. dev photos 버킷·CloudFront는 축소 사양이나 **보안 기준선(퍼블릭 차단·SSE·서명 URL·OAC)은 양 환경 동일**(SI §9 — dev 완화 금지).

**U7 배포 델타**: **Terraform storage 모듈 활성화(S3 photos·CloudFront — SI 예약분)** 가 인프라 델타. 서버는 기존 워크플로 흡수(0 델타). 모바일은 이미지 압축/오프라인 큐 라이브러리의 네이티브 여부에 따른 EAS 재빌드 검토(Code Generation 확정).

---

## 8. 컴플라이언스 요약 (U7 Infrastructure Design 시점)

| 규칙 | 판정 | 근거 |
|---|---|---|
| SECURITY-01 암호화 | **준수(핵심)** | §1 S3 SSE-KMS·TLS-only·§2 CloudFront TLS1.2+ — 사진 스토리지 첫 도입분 |
| SECURITY-02 액세스 로깅 | 준수 | §2 CloudFront 표준 로깅 활성(SI §5.4 예약 원칙 확정) |
| SECURITY-05 입력 검증 | 준수 | §1 presigned 조건(크기·타입)·§3 attach 재검증(매직 바이트·EXIF) |
| SECURITY-08 접근 제어 | **준수(핵심)** | §2 서명 URL 소유권 스코핑(IDOR 차단)·§1 accountId 프리픽스 IAM 경계 |
| SECURITY-09 하드닝 | **준수(핵심)** | §1 퍼블릭 전면 차단·§2 OAC·기본 루트 객체 없음(디렉터리 리스팅 불가) |
| SECURITY-06·07 | 준수 | §1 S3 게이트웨이 엔드포인트(NAT 우회·프라이빗)·태스크 역할 photos 프리픽스 한정(SI §7.1) |
| SECURITY-14 로깅·법정 로그 | **준수(핵심)** | §4 위치 법정 로그 U1 append-only 재사용·2중 강제 유지·§5 GPS 파기(D34)/보존 분리 |
| SECURITY-10·12 | 준수(상속) | §5 서명 키 Secrets Manager·평문 0·§7 IaC 버전 고정·시크릿 스캔 게이트 U1 상속 |
| SECURITY-11 | 준수(상속) | §5 회고 C1 서버 재조회·rate-limit U5 상속 — U7 무관 증분 |
| SECURITY-03·04·13·15 | 상속 | 로깅·헤더·무결성·예외 처리 U1 스캐폴드 정본 — 사진 어댑터도 마스킹·상관 ID |
| RESILIENCY-02·11·12 | **준수(증분)** | §1 사진 버킷 버저닝(백업성 — RESILIENCY-12)·SI §3 백업 상속 |
| RESILIENCY-01·08·09 | 상속 | 워크로드 분류(M12·M13 High nfr §0.1)·Multi-AZ·오토스케일 SI 상속 — U7 신규 워크로드 없음 |
| RESILIENCY-05·07 | 준수(증분) | §6 사진 비용 A15·업로드 실패율·회고 폴백율 계측 편입 |
| RESILIENCY-10 | 준수(상속) | 회고 C1 서킷·타임아웃 U5 상속·사진 업로드 재시도(nfr §4.1) |
| RESILIENCY-03·04·14·15 | 상속 | 변경 관리·롤백·복원력 테스트 전역 결정(GD-2~7)·deployment-architecture.md 상속 |
| PBT-01~10 | 상속 | CI 실행 요건 deployment-architecture.md §2 — 오프라인 무손실·changelog 재구성(G132)은 Code Generation 실행 |

**차단 소견 없음.** U7은 **사진 스토리지·CDN이 실질 인프라 증분인 첫 유닛**이며, SI가 §5.2·§5.4에 예약한 자리(S3 photos 버킷·CloudFront)를 활성화해 아키텍처 이탈 없이 종결한다. CMK 재평가(=aws/s3 유지)로 SI §6 단서를 종결하고, 회고·컴퓨트·DB·LLM은 U5·U1·SI 정본을 재사용한다(과설계 금지 — Lambda·EventBridge·전용 트랜스코딩·ElastiCache 미도입).
