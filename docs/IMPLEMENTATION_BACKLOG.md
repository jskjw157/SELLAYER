# SELLAYER — 구현 백로그 / 개발 이슈 분할 v0.1

> **문서 ID:** SL-BACKLOG-001 · **작성일:** 2026-10-16 (KST)  
> **상태:** Issue 후보 목록 (아직 실제 GitHub Issues로 생성하지 않음)  
> **관련:** [PRD](./PRD.md) · [TDD](./TECHNICAL_DESIGN.md) · [UX](./UX_SPEC.md) · [Quality](./QUALITY_PLAN.md) · [Deployment](./DEPLOYMENT.md)

## 1. 사용법

- 다음의 각 BL 항목은 독립 개발 이슈 후보. PoC/외부 승인에 의해 조정 가능.
- 우선순위: **P0** 첫 출시/보안 게이트, **P1** 초기 베타 가치 강화, **P2** 이후 확장.
- 상태: Proposed → Ready → In Progress → Review → Done. "Done" 기준 아래 정의.
- 난이도/일정은 엔지니어 인원·VPS·AI 품질 실측 전이라 **산정하지 않음**.
- 실제 이슈 생성 전 epic·담당자·리포·PR 범위 확인하고 과도한 동시 개발을 제한.
- 각 이슈 템플릿은 **목적/범위/비범위/의존성/요구사항/인수기준/테스트/운영 영향/보안 고려사항** 포함.

## 2. 선행 검증 스파이크 (우선)

| ID | P | 작업 | 결과물 / 완료 기준 |
|---|---|---|---|
| BL-001 | P0 | 주얼리 Golden set 구축 | 합법적 원본 30~50 SKU(목표) 특징·권리/옵션 메타와 테스트 매트릭스 작성 |
| BL-002 | P0 | 이미지 API 모델/가격/품질 PoC | 4종 컷 × 3종 참조 수 비교, Fidelity/원가/지연/재생성 지표 |
| BL-003 | P0 | 제품 레이어 합성 vs AI 편집 비교 | 착용샷 2가지 파이프라인 정량/사람 검수 보고 |
| BL-004 | P0 | JSON PDP + DOM 편집 + Chromium Export PoC | 8개 섹션, 텍스트/이미지 수정, 동일한 출력 확인 |
| BL-005 | P0 | Cafe24 앱스토어/OAuth 경로 조사 | 공식 앱 등록/권한/심사/상품 수정 권한 검증 |
| BL-006 | P1 | Naver 솔루션마켓 입점 경로 조사 | 사업자/승인/SELLER 토큰/상품 수정 필드 제약 조사 |
| BL-007 | P0 | Hostinger 서버 자원 조사 | Ubuntu 24.04 CPU/RAM/storage, Docker, 도메인, SSH 최소 권한, 배포 방식 기록 |

**PoC 산출물**은 사용자 승인 전 유료 대량 생성/실상품 게시를 하지 않는다.

## 3. EP-01 Foundation / Auth / Workspace

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-010 | P0 | pnpm monorepo + web/worker/packages scaffold | 없음 | lint/typecheck/build pass |
| BL-011 | P0 | Postgres migration & DB client | BL-010 | dev DB에서 migration repeatable |
| BL-012 | P0 | 세션 인증 / 로그인 / 로그아웃 | BL-011 | invalid session 401, cookie secure |
| BL-013 | P0 | Workspace/membership + 접근 guard | BL-012 | 서로 다른 tenant IDOR test PASS |
| BL-014 | P0 | 앱 UI shell / sidebar / dashboard | BL-012 | UX-01/02 flow |
| BL-015 | P0 | Audit event / requestId / error contract | BL-010 | 5xx/401/403 로그에서 비밀정보 미노출 |
| BL-016 | P1 | 브랜드 프리셋 및 기본 템플릿 설정 | BL-013 | user theme 분리, 재사용 가능 |

## 4. EP-02 Product / Upload / Fidelity Locks

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-020 | P0 | 상품/옵션 API + 스키마 | BL-013 | 실버/골드 Variant 유지 |
| BL-021 | P0 | Object storage signed upload | BL-013 | upload init/complete, MIME/size 검사 |
| BL-022 | P0 | 원본 불변 Asset/sha/EXIF 처리 | BL-021 | 원본 바이트 덮어쓰기 불가 |
| BL-023 | P0 | 이미지 앵글/상품/무드 참조 분리 UI | BL-022 | 이미지 role 혼동 방지 |
| BL-024 | P0 | 제품 특징 추론 API + 사용자 검증 | BL-020/022 | inferred/unverified 분리 |
| BL-025 | P0 | Critical/Major lock 관리 UI/API | BL-024 | Lock 변경 후 기존 생성 결과 경고 |
| BL-026 | P1 | 누끼/크롭·회전·정제 모듈 | BL-022 | 원본 보존, 파생 자산 생성 |

## 5. EP-03 AI Generation / Queue / Cost

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-030 | P0 | Provider contract / fake provider | BL-010/011 | 성공/429/5xx/timeout tests |
| BL-031 | P0 | Quote API + priceVersion + cost ceiling | BL-020/024/030 | 설정별 예상 비용 표시 |
| BL-032 | P0 | Credit ledger/reservation atomic transaction | BL-031 | 이중차감/초과사용 불가 |
| BL-033 | P0 | Job/Batch DB 모델 + outbox | BL-032 | 일관성 유지 후 큐 전달 |
| BL-034 | P0 | Redis/BullMQ Worker + ack/reconcile | BL-033 | worker restart 복구 |
| BL-035 | P0 | 실제 ImageProvider 어댑터 | BL-030/034 + PoC | 참조 이미지·마스크 지원범위 검증 |
| BL-036 | P0 | Prompt builder / Feature Locks 적용 | BL-025/035 | 고정/가변 지침 분리 |
| BL-037 | P0 | 이미지 저장·후처리·메타데이터 | BL-035 | 중복 Asset/정산 방지 |
| BL-038 | P0 | retries, provider unknown outcome, cancellation | BL-034/035 | 중복 유료 호출 방지/복구 |
| BL-039 | P0 | 배치 진행 API/UI + 부분 성공 | BL-034/037 | 10개 중 1 실패 → 9개 보존 |
| BL-03A | P1 | 모델/브랜드 프리셋 / prompt templates | BL-036 | 일관된 설정 재사용 |
| BL-03B | P1 | 100장 배치 운영 설정 / concurrency limits | BL-038/039 | mock load 100개, 예산 차단 |

## 6. EP-04 Gallery / Review / QA

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-040 | P0 | 원본/생성 갤러리 UI/API | BL-037 | 프로젝트/옵션별 filter |
| BL-041 | P0 | 원본 vs 결과 비교 모드 | BL-040 | 확대/슬라이더/좌우 비교 |
| BL-042 | P0 | Review 승인/거절/사유/히스토리 | BL-040 | 미승인 자산 PDP 자동 사용 금지 |
| BL-043 | P0 | Regenerate new Job + history | BL-038/042 | 기존 자산/문서 불변 |
| BL-044 | P1 | 자동 Fidelity QA 경고 | BL-001/002/040 | 테스트셋 FN/FP 측정 |
| BL-045 | P1 | Color/Variant contamination warning | BL-044 | 실버/골드 혼합 경고 |

## 7. EP-05 PDP Documents / Templates

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-050 | P0 | PDP JSON v1 + Zod validation/migrations | BL-010 | schemaVersion roundtrip |
| BL-051 | P0 | 11종 PDP 블록 라이브러리 | BL-050 | storybook/snapshot |
| BL-052 | P0 | 기본 주얼리 템플릿 | BL-051 | Hero/Feature/Detail/Spec/Notice |
| BL-053 | P0 | AI PDP draft builder + fact provenance | BL-042/052 | 미확인 사실 게시 차단 |
| BL-054 | P0 | Revision storage / ETag/If-Match APIs | BL-050 | 두 탭 충돌시 409 |
| BL-055 | P1 | 템플릿 저장/복제 + 브랜드 토큰 | BL-052/016 | 재사용/서체 일치 |

## 8. EP-06 Figma-like Editor

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-060 | P0 | 3패널 editor shell + canvas/zoom | BL-051/054 | UX-10 기본 화면 |
| BL-061 | P0 | Layers tree selection/reorder/lock | BL-060 | 트리와 캔버스 동기화 |
| BL-062 | P0 | Text block direct editing/typography | BL-060 | 한글 입력/줄바꿈 정확 |
| BL-063 | P0 | Image crop/fit/replace/approved assets | BL-060/042 | 미승인 이미지 기본 선택 불가 |
| BL-064 | P0 | Section add/delete/clone/rearrange | BL-061 | stable unique node ID |
| BL-065 | P0 | Commands undo/redo + keyboard | BL-061~064 | 텍스트 편집 충돌 회피 |
| BL-066 | P0 | Autosave/debounce/revision conflicts | BL-054/065 | 저장 실패/409 복구 |
| BL-067 | P0 | Revision history/restore | BL-054/066 | 게시본 불변 |
| BL-068 | P1 | Alignment guides/limited overlay | BL-060 | export와 위치 동일 |
| BL-069 | P1 | Mobile simple editing/read-only canvas | BL-060 | 핵심 사용 가능 |

## 9. EP-07 Export / Print

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-070 | P0 | Shared document renderer and fonts | BL-051/062/063 | 브라우저/서버 출력 계약 |
| BL-071 | P0 | Export queue/Playwright screenshots | BL-070/034 | PNG/긴 이미지 청크 |
| BL-072 | P0 | ZIP gallery + export manifest | BL-037/071 | 파일명/Asset 출처/Revision 추적 |
| BL-073 | P0 | Export parity snapshot suite | BL-071 | 글자/레이아웃 회귀 PASS |
| BL-074 | P1 | Sanitized HTML/PDF output | BL-071 | 태그 허용/외부 이미지 수명 테스트 |

## 10. EP-08 Cafe24 / Publishing

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-080 | P0(M1) | Official Cafe24 app/OAuth installation | BL-005/013 | 승인된 mall_id, state 및 refresh |
| BL-081 | P0(M1) | Existing Product read/import and mapping | BL-080 | 매장/상품 ID 정확 |
| BL-082 | P0(M1) | Allowed content renderer + sanitization | BL-074/081 | 제품 상세 설명 규격 확인 |
| BL-083 | P0(M1) | before snapshot + previewDiff + approval | BL-081/082 | target/hash bind |
| BL-084 | P0(M1) | Safe update + verify/re-read | BL-083 | 가격/재고/옵션 불변 |
| BL-085 | P0(M1) | Retry/rollback guidance + audit log | BL-084 | 미확인 게시 성공으로 표시 X |
| BL-086 | P0(M1) | Appstore public launch reviews | BL-080~085 | 카페24 정책/심사 승인 |

## 11. EP-09 Billing / Ops / Compliance

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-090 | P0 | Dashboard usage/quota and budget guards | BL-031/032 | 일/월 한도 정책 |
| BL-091 | P0 | Admin queue / error / cost dashboard | BL-039 | provider 비용과 잔액 구분 |
| BL-092 | P0(M1) | Paid plan / credit packages / PG | BL-032 | 이중 청구/환불/정산 |
| BL-093 | P0(M1) | Customer TOS / privacy / rights & deletion | BL-013/022 | 실제 약관/개인정보 처리·삭제 |
| BL-094 | P0 | Security tests (auth/IDOR/SSRF/XSS/CSRF) | BL-011~022 | 치명적 결함 0 |
| BL-095 | P1 | Analytics funnel / success KPIs | BL-014 | 개인정보 최소 계측 |

## 12. EP-10 Naver Channel / Multi Commerce

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-100 | P0(M2) | Naver solution app/SELLER authorization | BL-006 | 심사·승인·테넌트/SELLER 격리 |
| BL-101 | P0(M2) | 상품/옵션 mapping + existing read | BL-100 | 원상품/채널상품 구별 |
| BL-102 | P0(M2) | Naver detailed content renderer | BL-074/101 | HTML·이미지 포맷 유효 |
| BL-103 | P0(M2) | Preview/commit/re-read verification | BL-083/102 | 가격·재고 불변, 타 스토어 차단 |
| BL-104 | P0(M2) | Subscription/revocation/error states | BL-100/103 | 권한 해제 즉시 반영 |

## 13. EP-11 CI/CD · 인프라 · 안정성

| ID | P | Task | 의존 | 통과 조건 |
|---|---|---|---|---|
| BL-110 | P0 | .github/workflows/ci.yml | BL-010 | PR build/tests pass |
| BL-111 | P0 | Dockerfile Web/Worker, Compose env | BL-010/034 | local production-mode boot |
| BL-112 | P0 | GHCR artifact publish by SHA | BL-111 | immutable image release |
| BL-113 | P0 | Hostinger secrets/TLS/proxy/firewall | BL-007/111 | HTTPS, public DB/Redis closed |
| BL-114 | P0 | Restricted SSH Deploy + prod env approval | BL-112/113 | PR만으로 prod 배포 안 됨 |
| BL-115 | P0 | DB backup/offsite and restore rehearsal | BL-113 | restore demonstrable |
| BL-116 | P0 | Healthcheck, release smoke, rollback | BL-114 | 실패 시 이전 digest 실행 |
| BL-117 | P0 | Job recovery after restart/redeploy | BL-034/116 | no duplicate generation/charge |
| BL-118 | P1 | Monitoring, queue alerts, cost alerts | BL-091/116 | alerts tested |
| BL-119 | P1 | Staging isolation and promotion policy | BL-114 | separate keys/data/resources |

## 14. GitHub Issue 본문 템플릿

~~~markdown
### 목적
왜 이 변경이 필요한지 사용자 문제와 함께 설명

### 요구사항
PRD: FR-0xx, NFR-0x
UX: UX-AC-0x
Architecture: TDD Section x
Epic: EP-0x
Priority: P0 / P1 / P2

### 완료 조건 (Acceptance)
- [ ] 사용자가 실제로 수행할 수 있는 행동 기준
- [ ] 일반 성공 시나리오
- [ ] 실패/권한/중복/충돌 시나리오
- [ ] 비용/보안 영향 검증

### 설계/비범위
작업 경계, API/DB 변경, 디자인 변경, 이후 작업에 맡길 사항

### Test Plan
Unit / Integration / E2E / Visual / Security
테스트 코드 또는 로그의 증거 링크

### Release/Operations
Migration? Environment variables? Rollback? Monitoring?
~~~

## 15. Definition of Done

개발 이슈 Done은 코드가 push되었다는 의미가 아니다.

- [ ] 계약/타입과 실패 동작 정의.
- [ ] 실제 구현 + 재현 가능한 테스트 및 관련 요구사항 링크.
- [ ] 다중 테넌트 권한, API 키 비노출, 원본 불변 조건 점검.
- [ ] 생성/게시/결제 부작용의 멱등/복구·장애 시 행동 확인.
- [ ] UX 실패/빈 상태/로딩 및 접근성 기본 동작 확인.
- [ ] 성능/원가(해당 시) 실측, 회귀 테스트.
- [ ] DB 스키마/API/운영 문서 최신화, breaking changes/롤백 기록.
- [ ] 리뷰 승인, CI PASS, 보호 브랜치 정책 충족.
- [ ] 외부 플랫폼 등록/승인 필요 작업은 **실제 승인 후에만** 출시 완료 처리.

## 16. 의존성 핵심 경로

~~~mermaid
flowchart LR
  P1[BL-001 상품 Golden Set] --> P2[BL-002 이미지 품질 PoC]
  P3[BL-004 Editor/Export PoC] --> A[BL-010 Monorepo]
  A --> I[EP-01 Auth]
  I --> F[EP-02 Product Upload & Locks]
  P2 --> G[EP-03 Generation]
  F --> G
  G --> R[EP-04 Review]
  R --> D[EP-05 PDP]
  D --> E[EP-06 Editor]
  E --> O[EP-07 Export]
  O --> C[EP-08 Cafe24]
  C --> N[EP-10 Naver]
  A --> OPS[EP-11 CI/CD & Infra]
  G --> B[EP-09 Billing/Ops]
~~~

**권장 첫 개발 슬라이스**: 원본 3장 업로드 → 특징 Lock → fake provider로 이미지 4장 → 승인 → 템플릿 PDP → 텍스트/이미지 변경 → PNG Export. 이 플로우가 실제로 작동한 뒤 유료 Provider와 외부 채널 연결.

## 17. 수용·의사결정이 남은 사항

1. 첫 버전이 **주얼리 전용**인지 또는 초기부터 전 카테고리 제공인지.
2. 최소 원본 이미지/추가 참조 안내 정책.
3. 실제 사용 가능한 상업용 이미지 공급자와 단가·원본 보존률.
4. 편집기 화면 디자인 시안 및 컨트롤 PoC 확정.
5. 초기 과금과 무료 체험 정책, 크레딧 환원 정책.
6. Hostinger VPS 리소스, 도메인, 인증, 배포 운영자.
7. Cafe24/네이버 앱 심사·사업자 정보·테스트 판매자 계정.

> 이 문서는 이슈 분할 제안이며 실제 코딩, GitHub Issue 생성, VPS 배포 완료를 의미하지 않는다.
