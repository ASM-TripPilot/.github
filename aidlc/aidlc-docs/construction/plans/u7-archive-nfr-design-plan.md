# U7 기록·회고 — NFR·Infra 통합 실행 계획 (Plan)

> 2026-07-04 · CONSTRUCTION · U7 (M12 Travel Archive · **M13 AI Reflection** + `features/archive`·`shared/storage` 오프라인 로컬 큐)
> 입력: [unit-of-work.md](../../inception/application-design/unit-of-work.md) U7(범위·산출물·DoD·리스크 — **선결 신규 없음**: P6·P1·P7 기완료) · [stories.md](../../inception/user-stories/stories.md) Epic 8(US-E8-01~14) · [requirements.md](../../inception/requirements/requirements.md) §6(§6.1·§6.2·§6.3 M12·M13=High·§6.4·§6.7 G144 관측성)·§7.3(**G74 오프라인·G75/G145 사진·G168 이미지 파이프라인·G132 changelog·G72·G59**)·N2(§5.3 GPS 법정 로그·파기)·D14·D18·D34 · [component-dependency.md](../../inception/application-design/component-dependency.md)(§3.1 archive/actual/changelog 생명주기·M12·M13·§2.13 M13→C1) · [shared-infrastructure.md](../shared-infrastructure.md)(**SI — §5.2 사진 버킷 예약분·§5.4 CloudFront 예약분·§0 U7 표·§6 시크릿·§3 RDS·§8 관측성**)
> 산출: `../u7-archive/nfr-requirements/nfr-requirements.md`·`tech-stack-decisions.md`, `../u7-archive/nfr-design/nfr-design-patterns.md`, `../u7-archive/infrastructure-design/infrastructure-design.md`

---

## 1. 전역 재사용 선언 (재결정 금지)

**본 계획은 U7의 NFR Requirements·NFR Design·Infrastructure Design 3스테이지를 하나의 실행으로 통합한다.** U7은 **사진 저장(S3+CloudFront)이 프로젝트 최초로 등장하는 유닛**이다 — SI가 "U7 예약"으로 명시해 둔 사진 버킷(§5.2)과 CloudFront 배포(§5.4)를 **실제 활성화**하므로, U1~U6과 달리 **스토리지·CDN 인프라 증분이 실질적으로 존재**한다. 그 외 플랫폼 골격(컴퓨트·네트워크·DB·시크릿·관측성)과 **회고 연산(C1 LLM Gateway)** 은 전역 정본을 상속한다. 신규 사용자 질문 없음 — 아래 정본을 재사용한다(재질문·재결정 금지).

| 정본 | 출처 | U7에서의 처리 |
|---|---|---|
| 전역 확정 결정 GD-1~7 | SI 헤더 · U1 nfr-design §2 | 재사용(AWS 서울·GitHub Actions·버전 고정 롤백·롤링·경량 프로세스·복원력 테스트 Operations 이연) |
| 공유 인프라 정본 | [shared-infrastructure.md](../shared-infrastructure.md)(SI) | **참조 + 예약분 활성화** — SI §0 유닛 표: "U7 = **사진 S3 버킷 + CloudFront(§5.2 — 예약분 활성화)**". 델타는 infrastructure-design.md 델타 표에서 **신규 리소스**로 명시 |
| 스택 대분류 | requirements.md §2.3(D02) · U1 tech-stack §2 | Spring Boot Kotlin + PostgreSQL + RN Expo 재사용. U7은 **이미지 스토리지·CDN·EXIF 제거·썸네일·오프라인 큐** 지점만 구체화 |
| **회고 연산(C1) 정본** | **U5 [tech-stack-decisions.md](../u5-itinerary/nfr-requirements/tech-stack-decisions.md)·[nfr-design-patterns.md](../u5-itinerary/nfr-design/nfr-design-patterns.md)** | **재사용(신규 없음)** — C1 LLM Gateway(단일 벤더 서버 경유 D11·티어 라우팅·서버 재조회 D31·rate-limit·비용 계측·출력 스키마 검증)는 U5가 프로덕션 품질로 확정한 공통 자산. M13 회고·요약·스타일 분석(S4)은 **상위 티어 소비자**로 호출(재정의 금지). 신규 LLM 벤더·인프라 표면 0 |
| 사진 정책 정본 | requirements.md **§7.3 G75/G145**·G168 | 장소당 20장·클라 압축(5MB·긴 변 2048px)·서버 썸네일·원본 기기 보관·**S3 호환 오브젝트 스토리지+CDN** 상속 |
| 오프라인 정본 | requirements.md **Δ6/D24·G74** · stories US-E8-12 · component-dependency §3.1 | **입력만 오프라인 보장**(조회는 온라인 전제)·로컬 큐·레코드 단위 버전 비교·사진/메모 합집합 병합 상속 |
| changelog 정본 | requirements.md **G57/G132** · component-dependency §3.1 | 통합 diff 스키마(행위자·출처·사유·전/후 값)·diff 누적 재구성=스냅샷 동등성 — Plan-B·공동편집·어시스턴트 공용(U9~U11 참조 정본) |
| 위치정보법·파기 정본 | requirements.md **D34·N2·§6.5** · SI §5.3(append-only 법정 로그) | GPS 옵트인·철회/탈퇴 시 즉시 파기·법정 로그(**U1 append-only 테이블 재사용**) 상속 |
| 규모 정본 | requirements.md §6.8(G142) — DAU 1천 | 사진 스토리지·CDN 비용 상한을 DAU 1천 기준으로 산정(과설계 금지) |
| 관측성·시크릿·스케줄러 정본 | SI §8(CloudWatch·A15 비용)·§6(Secrets Manager·KMS)·§1.2(ShedLock) | 재사용 — CloudFront 서명 키(Secrets Manager)·사진 스토리지 비용 알람(A15)·GPS 파기 잡(ShedLock) 편입 |

**설계 지배 원칙**: U7 NFR 본질은 **① 오프라인 우선 입력(핵심 — 로컬 큐·재시도·G74 무손실 병합), ② 스토리지·미디어(핵심 — 사진 파이프라인·S3·CDN·EXIF 프라이버시), ③ 데이터 정합(changelog diff 재구성·plan/actual 대조), ④ 복원력(업로드 실패 큐·회고 LLM 폴백)** 이다. 회고 연산(C1)은 **U5 자산 재사용(신규 벤더·인프라 표면 0)** 이고, 신규 인프라는 **S3 사진 버킷 + CloudFront(SI 예약분 활성화)** 뿐이며 SI가 예약한 스토리지 프레임 내에서 종결된다.

---

## 2. 통합 실행 체크리스트

### 2.1 준비 — 정본 로드

- [x] U7 범위 로드 (unit-of-work.md U7 — 목적·포함·제외·산출물·DoD·리스크·**선결 신규 없음**: P6 LLM/U5·P1·P7/U6 기완료, 오브젝트 스토리지·CDN 선정은 Infrastructure Design 결정)
- [x] Epic 8 스토리 로드 (US-E8-01 방문 체크·02 사진/메모 첨부·03 GPS 옵트인·04 plan/current/actual/changelog 구분·05 귀속·06 당일 회고·07 수정/재생성·08 전체 요약·09 스타일 분석·10 개인화·11 마이페이지 열람·12 오프라인 동기화·13 공유 카드·14 캘린더)
- [x] 사진 정책 정본 로드 (§7.3 G75/G145 — 20장·5MB/2048px 클라 압축·서버 썸네일·원본 기기 보관·S3+CDN G168)
- [x] 오프라인 정본 로드 (Δ6/D24 — 입력만 보장·조회 온라인 전제 / G74 — 레코드 버전 비교·사진/메모 합집합)
- [x] changelog 정본 로드 (G57/G132 — 통합 diff·누적 재구성=스냅샷 동등성 PBT)
- [x] 위치정보법·파기 정본 로드 (D34·N2·§6.5 — GPS 옵트인·철회/탈퇴 즉시 파기·append-only 법정 로그 U1 재사용)
- [x] 회고 연산 정본 로드 (U5 C1 LLM Gateway — 재사용, 신규 없음. 상위 티어·10곳 게이트·서버 재조회 D31·실패 시 기본 카드 폴백)
- [x] 공유 인프라 정본 로드 (SI §0 U7 표·**§5.1 S3 기준선·§5.2 photos 버킷 예약분·§5.4 CloudFront 예약분**·§6 시크릿/KMS·§3 RDS·§8 관측성/A15·§1.2 ShedLock)
- [x] 전역 결정 GD-1~7 재확인 (재질문 없음 — §1 정본 상속)

### 2.2 NFR Requirements (nfr-requirements.md)

- [x] **오프라인 우선(핵심)** — 기록 입력 로컬 큐(방문 체크·사진·메모·수동 체크인)·동기화 대기 표시·재접속 자동 동기화·충돌 해소(레코드 버전+항목별 선택·사진/메모 합집합 G74)·조회는 온라인 전제(D24/Δ6)
- [x] **스토리지·미디어(핵심)** — 사진 장당 5MB·긴 변 2048px 클라 압축·장소당 20장(G75/G145)·서버 썸네일·EXIF GPS 제거 위치(클라 vs 서버 G168)·CDN 서빙
- [x] **성능** — 사진 업로드(비차단 백그라운드 큐·presigned 발급)·썸네일 생성 시간·회고 생성 시간(비동기 이벤트 구동)·요약/기록 조회
- [x] **복원력** — 사진 업로드 실패 로컬 큐·지수 백오프 재시도 3회·메모/체크는 업로드와 독립 저장(US-E8-02)·회고 LLM 실패/타임아웃 기본 카드 폴백(US-E8-06)·워크로드 M12·M13=High(입력 로컬 큐 보존, 조회/생성만 지연 §6.3)
- [x] **보안·프라이버시** — EXIF GPS 제거(기본 자동)·GPS 데이터 철회/탈퇴 즉시 파기(D34/N2)·사진 접근 제어(소유권·서명 URL)·업로드 파일 검증(크기/타입 SECURITY-05)·스토리지 암호화(SECURITY-01)
- [x] **규모** — 사진 스토리지·CDN 비용 상한(DAU 1천 G142)·원본 기기 보관으로 서버 부담 축소·과설계 금지
- [x] **회고** — C1 재사용(상위 티어·10곳 게이트·서버 재조회 D31)·회고 LLM 타임아웃·비동기(DayClosed/TripEnded 구동)·ReflectionReady 발행까지가 U7
- [x] 미결·선결 — **신규 선결 없음**(P6/P1/P7 기완료), 오브젝트 스토리지·CDN 확정은 Infrastructure Design 소관
- [x] 확장 규칙 컴플라이언스 요약(준수/상속/N/A + 근거)

### 2.3 Tech Stack Decisions (tech-stack-decisions.md)

- [x] **이미지 스토리지 — Amazon S3**(SI §5.2 예약분) 대안(S3 호환 자체 호스팅·타 클라우드)+사유
- [x] **CDN — CloudFront + 서명 URL**(SI §5.4 예약분·OAC) 대안(퍼블릭 버킷·앱 프록시 서빙)+사유
- [x] **EXIF GPS 제거 — 클라 전처리 vs 서버** — **원본 GPS 기기 이탈 여부 G168 권고**(클라 제거 시 원본 GPS 서버 미도달=프라이버시 최적) + 서버 방어적 재제거(심층 방어) 트레이드오프 명시
- [x] **썸네일 생성 — 클라 vs 서버 Lambda vs 앱(ECS)** — G75 서버 썸네일, 앱(ECS) 동기 생성 권고(Lambda/EventBridge 미도입 G142) 대안+사유
- [x] **사진 업로드 파이프라인 — 서명 URL 직업로드(presigned PUT) vs 서버 경유** 대안+사유(ECS 대역폭 오프로드 vs 서버 방어 트레이드오프)
- [x] **오프라인 큐 — `shared/storage` 로컬 영속 큐·재시도** 대안(메모리 큐)+사유
- [x] **회고 = U5 C1 재사용(신규 없음 명시)** — LLM Gateway·티어 라우팅·서버 재조회·비용 계측은 U5 확정분
- [x] 각 선택 "재사용/증분" 명시 + 신규는 대안 1줄 + 사유(프라이버시 트레이드오프 병기)

### 2.4 NFR Design Patterns (nfr-design-patterns.md)

- [x] PAT-U7-01 오프라인 우선 입력 큐·재시도·충돌 해소(레코드 단위·합집합 G74 — 무손실 PBT)
- [x] PAT-U7-02 사진 업로드 파이프라인(클라 압축·EXIF 제거·서명 URL 직업로드·백그라운드 큐)
- [x] PAT-U7-03 CDN 서명 URL 접근 제어(OAC·소유권·짧은 만료)
- [x] PAT-U7-04 changelog 통합 diff 보관·재구성(G132 — 누적 재생=스냅샷 동등성)
- [x] PAT-U7-05 회고 폴백(C1 실패/타임아웃→기본 카드·부분 데이터 명시 US-E8-06)
- [x] PAT-U7-06 GPS 파기(D34/N2 — 옵트인 철회·탈퇴 시 즉시 파기, 법정 로그는 append-only 잔존)
- [x] 각 패턴: 문제/적용(설계)/U7 적용 지점/검증 기준(규칙 ID) 표기
- [x] 전역 상속 패턴(C1 재사용·PAT-SEC·PAT-RES·PAT-OBS·PAT-PERF) "U5·U1·shared 정본 상속" 표기

### 2.5 Infrastructure Design (infrastructure-design.md)

- [x] **S3 사진 버킷** — SI §5.2 `trippilot-{env}-photos` 예약분 활성화(버저닝·퍼블릭 차단·SSE-KMS·수명주기·TLS-only SECURITY-01)
- [x] **CloudFront 배포** — SI §5.4 예약분 활성화(OAC·서명 URL·HTTPS TLS1.2+·표준 로깅 SECURITY-02·기본 루트 객체 없음)
- [x] **이미지 처리** — 클라 전처리 기본(압축·EXIF 제거) + 서버 썸네일(앱 ECS 동기, Lambda 미도입) — 서명 URL 직업로드 흐름
- [x] **사진 메타·changelog·법정 로그** — 기존 RDS 스키마 확장(Flyway ~7 테이블)·**위치정보 법정 로그 U1 append-only 재사용**·GPS 파기 잡(ShedLock)
- [x] **회고 LLM** — U5 C1 재사용(신규 인프라·시크릿 0 — LLM API 키는 U5 등록분)
- [x] **CloudFront 서명 키** — Secrets Manager 신규 항목(SI §6 프레임 내)·KMS는 aws/s3 유지(CMK 미도입 근거·트리거)
- [x] SI 대비 델타 표(**신규 AWS 리소스 — S3 photos 버킷·CloudFront 배포**, SI 예약분 활성화 정합 확인)
- [x] 배포 — shared deployment-architecture.md 재사용 + **S3/CloudFront Terraform 모듈(SI §11.2 storage 모듈) 활성화**·클라 압축 라이브러리 EAS 델타 검토
- [x] 컴플라이언스 요약

### 2.6 검증·마감

- [x] 추적 ID(NFR-U7-xx·PAT-U7-xx·U7-S/C·SECURITY·RESILIENCY·PBT·D·G·GD·US-E8·CP·SI·§) 전 항목 표기
- [x] 콘텐츠 검증(content-validation) — ASCII 다이어그램·표 구문 확인
- [x] 규모 정합 점검 — DAU 1천 기준 사진 스토리지·CDN 비용(과설계 금지: Lambda/EventBridge·ElastiCache·별도 워커 미도입) 확인
- [x] **SI 예약분 활성화 정합** — SI §5.2 photos 버킷·§5.4 CloudFront가 U7에서 활성화됨을 델타 표에서 명시 확인
- [x] 산출물 4종 생성 완료

---

## 3. 선결 과제 연결 (§8 — U7 신규 없음)

| 과제 | 시점 | 본 계획의 비차단 대응 |
|---|---|---|
| P6 LLM 벤더 계약·비용·국외 이전 고지 | **U5에서 기완료** | M13 회고·요약·스타일 분석은 U5 C1 재사용 — 신규 벤더 계약·비용 산정 불요. 국외 이전 고지는 P7 처리방침 반영(기완료) |
| P1 위치기반서비스사업 신고 · P7 위치기반서비스 약관 | **U6에서 기완료** | GPS 발자취의 기록 귀속·옵트인 철회 파기(D34)는 U1·U6 법정 전제 위에서 동작 — U7은 파기 연동만 |

**U7 신규 비개발 선결 없음.** 인프라 결정(오브젝트 스토리지=S3·CDN=CloudFront)은 SI 예약분 활성화로 Infrastructure Design 내에서 종결된다(외부 선결 불요).

---

## 4. 산출물 목록

| 파일 | 내용 |
|---|---|
| `../u7-archive/nfr-requirements/nfr-requirements.md` | U7 NFR 요구 — 오프라인 우선(핵심)·스토리지/미디어(핵심)·성능·복원력·보안/프라이버시·규모·회고 + 선결(신규 없음) |
| `../u7-archive/nfr-requirements/tech-stack-decisions.md` | S3·CloudFront 서명 URL·EXIF 제거(클라 vs 서버 G168 권고)·썸네일·업로드 파이프라인·오프라인 큐·회고 U5 재사용(대안+사유) |
| `../u7-archive/nfr-design/nfr-design-patterns.md` | U7 패턴 6종(오프라인 큐·사진 파이프라인·CDN 서명 URL·changelog 재구성·회고 폴백·GPS 파기) + 전역 상속(C1·보안·복원력·관측성) |
| `../u7-archive/infrastructure-design/infrastructure-design.md` | **U7 인프라 증분(실질 — 신규 S3 버킷·CloudFront)** + SI 예약분 활성화 정합 + 델타 표 + 배포 갈음(storage Terraform 모듈 활성화) |

## 5. 다음 단계

- U7 Functional Design(선행) 또는 Code Generation: 본 산출물의 오프라인 병합 무손실(G74 PBT)·changelog diff 누적 재구성=스냅샷 동등성(G132 PBT)·사진 파이프라인(압축·EXIF·서명 URL)·plan/actual 대조·회고 폴백·CP4(소비자 VisitChecked·TripEnded·DayClosed·changelog)·CP5(공급자 ReflectionReady) 계약을 구현.
- 본 계획 §1의 전역 결정·C1 재사용은 U8 이후에도 동일 재사용(재질문 금지). **changelog diff 스키마(G132)·S3+CloudFront 사진 파이프라인은 후속 U10(커뮤니티 게시·EXIF G185)·U11(공동편집 이력)의 참조 정본**이다(unit-of-work.md U7 리스크: changelog 스키마 후속 3유닛 공용).
