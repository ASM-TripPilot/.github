# 요구사항 명확화 질문 (Clarification Questions)

38개 답변을 분석한 결과, 아래 항목이 명확하지 않아 확인이 필요합니다.
각 질문의 `[Answer]:` 태그 뒤에 선택지 문자를 기입하시고 완료되면 알려주세요.

---

## Ambiguity 1: Q5(Security Baseline) 답변이 'C(기타)'인데 서술 내용이 비어 있음

Q5에서 C(기타 — 직접 서술)를 선택하셨지만 서술 내용이 없어, 보안 확장 규칙의 활성화 여부·적용 범위를 판단할 수 없습니다.

### Clarification Question 1
보안(Security Baseline) 확장 규칙을 어떻게 적용할까요?

A) 전체 강제 — 모든 SECURITY 규칙을 차단(blocking) 제약으로 강제 (Q1에서 '실서비스 출시'를 선택하셨으므로 권장)

B) 전체 생략 — SECURITY 규칙을 적용하지 않음

C) 부분 적용 — 적용할 영역을 아래에 직접 서술 (예: 인증·개인정보 관련 규칙만)

D) 기타 (아래 [Answer]: 태그 뒤에 직접 서술)

[Answer]: A

---

## Ambiguity 2: Q2 커스텀 답변("React Native + Spring + PostgreSQL")의 세부 미정

선택하신 조합은 명확하지만, 설계·코드 생성 단계에 영향을 주는 두 가지 세부 결정이 남아 있습니다.

### Clarification Question 2
React Native 개발 방식은 무엇으로 할까요? (국내 지도 SDK는 네이티브 모듈이 필요해 어느 쪽이든 네이티브 코드 연동이 발생합니다)

A) Expo (development build + prebuild) — 빌드·업데이트 도구 체계 활용, 네이티브 모듈은 config plugin으로 관리 (권장)

B) React Native CLI (bare) — 네이티브 프로젝트 직접 관리, 도구 제약 없음

C) 기타 (아래 [Answer]: 태그 뒤에 직접 서술)

[Answer]: A

### Clarification Question 3
Spring Boot 백엔드의 주 언어는 무엇으로 할까요?

A) Kotlin — 널 안정성·간결성, PBT 프레임워크는 Kotest Property Testing

B) Java — 레퍼런스·인력 풀 최대, PBT 프레임워크는 jqwik

C) 기타 (아래 [Answer]: 태그 뒤에 직접 서술)

[Answer]: A

---

## 추가 필수 질문: 복원력(Resiliency) 확장 — RTO/RPO 목표

Q6에서 복원력 기준선 적용에 동의하셨습니다. 해당 규칙(RESILIENCY-02)은 요구사항 확정 전에 복구 목표를 반드시 확인하도록 요구합니다. 이 답변이 DR(재해 복구) 전략과 인프라 이중화 수준을 결정합니다.

### Clarification Question 4
RTO(목표 복구 시간)/RPO(목표 복구 시점) 목표는 무엇인가요?

A) 시간 단위 — Backup & Restore 전략. 최저 비용($). 백업만 유지, 장애 시 IaC 재배포 + 백업 복원. 비핵심 워크로드에 적합

B) 수십 분 단위 — Pilot Light 전략. 비용 $$. 데이터는 실시간 복제, 서비스는 유휴 상태로 대기

C) 분 단위 — Warm Standby 전략. 비용 $$$. 데이터 실시간 + 서비스 축소 용량 상시 가동. 비즈니스 크리티컬용

D) 실시간 근접 — Multi-site Active/Active 전략. 최고 비용($$$$). 다중 리전 동시 서비스. 무중단 요구용

E) N/A — 단일 리전 배포로 충분, 리전 간 DR 불필요. 리전 내 다중 가용영역(Multi-AZ)으로 대응

F) 기타 (아래 [Answer]: 태그 뒤에 직접 서술)

[Answer]: A
