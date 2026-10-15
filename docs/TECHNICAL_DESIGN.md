# SELLAYER — 기술 설계서 (Technical Design Document)

> **문서 ID:** SL-TDD-001 · **버전:** v0.1 · **작성일:** 2026-10-16 (KST)  
> **상태:** Proposed / 개발 전 설계안, 구현 및 검증 전  
> **원본 요구사항:** [PRD](./PRD.md) · [로드맵](./ROADMAP.md) · [채널 요구사항](./CHANNEL_INTEGRATIONS.md)  
> **동반 명세:** [편집기·사용자 경험](./UX_SPEC.md) · [품질 검증](./QUALITY_PLAN.md) · [배포 설계](./DEPLOYMENT.md)

## 0. 요약 및 결정 수준

SELLAYER는 **하나의 제품 프로젝트에서 이미지 자산을 생성·검수하고, JSON 문서로 상세페이지를 제작·편집하며, 채널별 렌더링으로 게시하는 멀티테넌트 SaaS**다.

- **M0**: 독립 SaaS 업로드 → 생성 → 리뷰 → 상세페이지 편집 → 다운로드.
- **M1**: Cafe24 승인 연결 → 기존 상품의 콘텐츠만 수정.
- **M2**: Naver 공식 솔루션 승인 후 스마트스토어 연결.
- **M3**: 고급 자유형 편집·높은 처리량·팀·다른 채널·신규 상품 등록.
- **절대 원칙**: 원본 불변, 제품 특징 검증, 수동 승인, 작업·비용 추적, 편집 가능한 문서 원본 보관, 게시 전 명시적 확인.
- **AI 사용 비용**: 판매용 이미지 생성은 상업용 공급자 API의 별도 과금과 사용량 제한을 따른다. ChatGPT 개인 구독의 내장 생성 기능을 판매용 백엔드로 설계하지 않는다.

설계 용량은 초기 **워크스페이스당 하루 100개 이미지 요청**을 가정한 큐 기반 처리다. 이것은 보장 SLA가 아니다. 시간·원가·동시성은 PoC 실측 후 확정한다.

### 0.1 중요한 아키텍처 결정 (초안)

| ID | 결정 | 선택 이유 | 반례 / 검증 |
|---|---|---|---|
| ADR-01 | 모듈형 모놀리식 + 별도 워커 | 한 대의 VPS에서 운영 단순화, 향후 워커 분리 가능 | 멀티서비스 분리는 사용량 실측 뒤 |
| ADR-02 | PostgreSQL = 데이터 SoT, Redis/BullMQ = 전달/스케줄링 | Redis 장애가 데이터 소실로 이어지지 않도록 | 작업 기록 기반 재동기화 테스트 |
| ADR-03 | S3 호환 비공개 객체 저장소 | 원본/생성 이미지는 VPS 로컬 디스크와 독립된 영속성 | R2/S3 실제 비용·리전·CORS 선택 |
| ADR-04 | 웹 문서 원본은 버전이 있는 JSON | 진짜 요소 단위 편집·수정·채널 재출력 | schema migration / export parity |
| ADR-05 | 에디터는 DOM+CSS 중심, 복잡한 조작은 캔버스 계층 보조 | 긴 세로 PDP와 글자/표 출력 일치 | Fabric/Konva PoC와 비교 |
| ADR-06 | 브라우저 렌더러 공유, 서버 Playwright로 출력 | WYSIWYG와 산출물 간 차이 축소 | 폰트·페이지 분할·이미지 접근 재현성 |
| ADR-07 | 사용자 승인 전 채널 게시 불가 | 고가 상품 형태 왜곡/잘못된 상품정보 피해 방지 | 승인을 필수 상태 머신으로 검증 |
| ADR-08 | 이미지/문구 공급자 추상화 | 최신 모델/가격/정책 변경에 대응 | 참조 이미지·마스크·결과 메타 필드 매핑 |
| ADR-09 | 기존 상품 콘텐츠 갱신만 우선 | 상품등록 시 가격·재고·고시 등 확장 범위 과대화 방지 | 채널 API patch 제약 확인 |

## 1. 요구사항 ↔ 시스템 책임

| 도메인 | 책임 | PRD | 주요 산출물 |
|---|---|---|---|
| Identity | 인증·워크스페이스·권한 | FR-001~005, NFR-01/02 | Member/Session |
| Catalog | 상품/변형/검증 사실/고정 특징 | FR-010~015 | Product, FeatureLock |
| Media | 이미지 업로드, 비공개 저장, 버전/출처 | FR-010/011, FR-040 | Asset |
| AI Orchestration | 작업·공급자·비용·재시도·품질검증 | FR-020~046, FR-080~082 | Batch, Job, Attempt |
| Creative Review | 승인·거절·비교·재생성 | FR-026/029/030 | AssetReview |
| PDP | 구성 블록·초안 생성·스키마·리비전 | FR-050~056 | PDPDocument, Revision |
| Editor | 조작 명령·레이어·실시간/자동 저장 | FR-060~068 | Editor commands |
| Export | HTML/PNG/ZIP/PDF(추후) | FR-070/071 | ExportManifest |
| Channel | 설치/인증·diff·승인·게시·검증 | FR-072~079 | Connection, PublishAttempt |
| Billing/Ops | 견적·예약·소모·환원·감사·관측 | FR-080~086, NFR-10 | Ledger, AuditLog |

## 2. 전체 구조

~~~mermaid
flowchart LR
  U[판매자 브라우저] --> RP[HTTPS Reverse Proxy]
  RP --> WEB[Next.js Web + API/BFF]
  WEB --> DB[(PostgreSQL)]
  WEB --> OBJ[(Private S3-compatible Objects)]
  WEB --> RED[(Redis / BullMQ)]
  RED --> WORK[Generation / Export Worker]
  WORK --> DB
  WORK --> OBJ
  WORK --> AI[Image & Text Provider API]
  WORK --> QA[Quality & Fidelity Checks]
  WORK --> EXP[Headless Browser Export]
  WEB --> CH[Channel Adapter Interface]
  CH --> CF[Cafe24 Admin API]
  CH --> NV[Naver Commerce API]
  WEB --> MET[Metrics / Logs / Audit]
  WORK --> MET
~~~

- **브라우저**: 제품 입력·원본 업로드·생성 설정·갤러리 검토·상세페이지 편집.
- **Web/BFF**: 모든 리소스에 세션·워크스페이스 권한 검증, 공개 API v1, 워크플로 제어.
- **DB**: 비즈니스 사실/권한/문서/자산 메타/작업 상태/크레딧 원장.
- **Object Storage**: 원본·생성물·미리보기·export. DB에는 키·해시·메타만 저장.
- **Redis**: 큐, 제한, 임시 상태/이벤트. 결제원장/완료상태의 유일한 저장소로 사용하지 않음.
- **Worker**: 프로바이더 호출, 리사이즈/검증, 결과 업로드, 작업·비용 확정, 결과 알림.
- **Export**: 서버측 고정 버전의 Chromium·폰트로 렌더.
- **Adapter**: API 인증과 게시 규격만 담당하며 핵심 문서 모델은 변경하지 않음.

### 2.1 Repo / 패키지 레이아웃 (계획)

~~~text
SELLAYER/
├── apps/
│   ├── web/                      # Next.js app router, REST v1, dashboard, editor
│   └── worker/                   # 이미지 생성, QA, export, publish jobs
├── packages/
│   ├── db/                       # migrations, query helpers, tenancy guards
│   ├── contracts/                # Zod DTO, schemas, API error codes
│   ├── domain/                   # use-cases, idempotency, ledger invariants
│   ├── editor-schema/            # PDP document v1 + migrations
│   ├── editor-renderer/          # web + export shared document rendering
│   ├── image-providers/          # provider adapters and prompt compiler
│   ├── channel-adapters/         # Cafe24 / Naver interfaces
│   └── ui/                       # reusable design system
├── infra/
│   ├── compose/                  # versioned compose manifests
│   ├── proxy/                    # HTTPS / reverse proxy
│   ├── scripts/                  # deploy, migrate, backup, rollback
│   └── monitoring/               # alerts / health
├── tests/
│   ├── fixtures/                 # synthetic images and privacy-safe samples
│   ├── contract/
│   ├── integration/
│   ├── e2e/
│   └── visual/
├── docs/
│   ├── PRD.md
│   ├── TECHNICAL_DESIGN.md
│   ├── UX_SPEC.md
│   ├── QUALITY_PLAN.md
│   ├── CHANNEL_INTEGRATIONS.md
│   └── DEPLOYMENT.md
├── .github/workflows/           # CI, image publish, controlled release
└── pnpm-workspace.yaml
~~~

코드 생성 이전에 pnpm/Node LTS/Next.js/Prisma/Playwright 등의 **실제 지원 버전과 보안 공지**를 확인하고 lockfile로 고정한다.

## 3. 테넌시와 접근 통제

### 3.1 데이터 소유권

모든 판매자 리소스는 workspace_id 필드를 가진다. Project/Asset/Job/PDP/ChannelConnection/CreditLedger/PublishAttempt 조회는 **필수로 workspace scope를 포함**해야 한다.

- 브라우저가 workspace_id를 직접 보냈더라도 서버는 **인증된 membership과 교차 검증**한다.
- 앱 컨텍스트에 활성 워크스페이스를 두되 query 별 소유권 검사를 생략하지 않는다.
- 역할: OWNER, ADMIN, EDITOR, VIEWER. M0은 OWNER 중심으로 구현하더라도 API는 역할 확장을 고려.
- 리소스 ID는 추측이 어려운 UUID/ULID를 사용하지만 보안 경계는 **ID 비예측성이 아닌 권한 확인**.
- PostgreSQL RLS는 방어층으로 검토(워크스페이스 context 누락 시 기본 deny); App-level guard 필수.
- 예약/관리/결제 API는 사용 권한뿐 아니라 입력 정책·예산·CSRF 보호·rate limit 필요.
- 쿠키 기반 세션은 HttpOnly/Secure/SameSite. OAuth 연결에는 state·redirect URI 정확 검증, PKCE는 공급자 지원 시.
- 공급자 API 키/판매자 토큰은 서버측 암호화 저장, 로그 redaction, 열람·회전 권한 최소화.

### 3.2 인가 매트릭스

| 작업 | OWNER | ADMIN | EDITOR | VIEWER |
|---|---|---|---|---|
| 원본/상품/생성 작업 관리 | O | O | O | 읽기 |
| 결과 승인/거절 | O | O | O | 읽기 |
| PDP 편집/내보내기 | O | O | O | 읽기/허용시 다운로드 |
| 채널 연결·해제 | O | O | - | - |
| 외부 채널 게시 | O | O | 승인 정책에 따라 | - |
| 결제·예산 한도 수정 | O | 관리자 위임 시 | - | - |
| 멤버 초대/삭제 | O | 승인 정책에 따라 | - | - |

M0에서는 임의의 역할을 노출하지 않으며, 미구현 역할 기능은 기본 거부.

## 4. 도메인 모델과 테이블

### 4.1 핵심 ERD

~~~mermaid
erDiagram
  USER ||--o{ MEMBERSHIP : belongs
  WORKSPACE ||--o{ MEMBERSHIP : has
  WORKSPACE ||--o{ PRODUCT_PROJECT : owns
  PRODUCT_PROJECT ||--o{ PRODUCT_VARIANT : contains
  PRODUCT_PROJECT ||--o{ ASSET : stores
  PRODUCT_PROJECT ||--o{ VERIFIED_FACT : describes
  PRODUCT_PROJECT ||--o{ FEATURE_LOCK : locks
  PRODUCT_PROJECT ||--o{ GENERATION_BATCH : requests
  GENERATION_BATCH ||--o{ GENERATION_JOB : contains
  GENERATION_JOB ||--o{ GENERATION_ATTEMPT : tries
  GENERATION_JOB ||--o{ ASSET : produces
  ASSET ||--o{ ASSET_REVIEW : reviewed
  PRODUCT_PROJECT ||--o{ PDP_DOCUMENT : designs
  PDP_DOCUMENT ||--o{ PDP_REVISION : versions
  WORKSPACE ||--o{ CHANNEL_CONNECTION : connects
  PRODUCT_PROJECT ||--o{ CHANNEL_PRODUCT_LINK : targets
  CHANNEL_CONNECTION ||--o{ CHANNEL_PRODUCT_LINK : maps
  PDP_REVISION ||--o{ PUBLISH_ATTEMPT : publishes
  WORKSPACE ||--o{ CREDIT_LEDGER : accounts
  GENERATION_BATCH ||--o{ CREDIT_LEDGER : settles
~~~

### 4.2 테이블의 최소 필드와 불변 조건

| 테이블 | 핵심 필드 | 제약 / 인덱스 |
|---|---|---|
| users | id, email, email_verified_at, created_at | email normalized unique; password hash가 있으면 Argon2id |
| workspaces | id, name, status, created_at | billing/usage scope |
| memberships | workspace_id, user_id, role | unique(workspace_id,user_id) |
| product_projects | id, workspace_id, title, category, status | index(workspace_id,updated_at desc) |
| product_variants | id, project_id, sku, option_label, color_code | unique(project_id,sku) when SKU present |
| verified_facts | id, project_id, variant_id?, key, value_json, source, verified_by, verified_at | inferred_unverified는 게시 불가 |
| feature_locks | id, project_id, variant_id?, feature_type, value_json, severity, note | prompt+QA 공통 입력 |
| assets | id, workspace_id, project_id, job_id?, type, object_key, mime, width, height, byte_size, sha256, parent_asset_id?, created_at | 원본 교체 금지, object_key unique |
| asset_reviews | id, workspace_id, asset_id, reviewer_id, decision, reasons_json, created_at | 최신 리뷰 상태 materialize |
| generation_batches | id, workspace_id, project_id, client_key, quote_id, status, requested, succeeded, failed, created_at | unique(workspace_id,client_key) |
| generation_jobs | id, batch_id, scene_type, variant_id?, prompt_version, status, leased_at?, attempt_count, result_asset_id? | index(status,created_at) |
| generation_attempts | id, job_id, provider, provider_request_id?, provider_state, cost_json, error_code, started_at, ended_at | provider unknown outcome 추적 |
| provider_usages | id, attempt_id, provider, model, pricing_version, input_units, output_units, estimated_cost, final_cost | 결제 가능 단위와 분리 |
| pdp_documents | id, workspace_id, project_id, schema_version, current_revision_id, draft_version | unique(project_id,document_type) optional |
| pdp_revisions | id, document_id, revision_number, document_json, sha256, actor_id, created_at, reason | unique(document_id,revision_number), immutable |
| channel_connections | id, workspace_id, platform, external_account_id, encrypted_credentials, scopes, status, expires_at | unique(platform,workspace_id,external_account_id) |
| channel_product_links | id, project_id, connection_id, external_product_id, external_variant_id?, metadata_json | unique(connection_id,external_product_id) |
| publish_attempts | id, workspace_id, document_revision_id, connection_id, external_product_id, approval_id, idempotency_key, status, before_snapshot_ref, after_snapshot_ref | unique(workspace_id,idempotency_key) |
| subscriptions | id, workspace_id, plan_code, status, billing_provider_ref | 개인정보·결제정보는 PG 저장 |
| credit_ledger | id, workspace_id, reservation_id?, job_id?, type, delta, currency, created_at | 추가형 append-only, 동일 정산 중복 차단 |
| audit_events | id, workspace_id, actor, action, target_type, target_id, correlation_id, redacted_json, created_at | 주기적 보관 정책 / 위변조 탐지 고려 |

**표준 필드**: PK UUID/ULID, UTC timestamp, created_at/updated_at, 리소스에 따라 deleted_at. 원본 Asset와 Revision은 수정 대신 새 버전/소프트 삭제로 보존하며 보관·법적 삭제 정책을 따름.

### 4.3 물리 설계 방향

- PostgreSQL은 정규화된 관계형 핵심 엔티티 + 제한된 JSONB(특징/문서/메타).
- 테넌트 대상 쿼리용 복합 인덱스(workspace_id, project_id/updated_at/status).
- 워커 상태 잠금은 transaction/unique constraint 사용. **Redis Queue에 job 존재 = DB에서 유효한 작업 존재**라는 가정을 하지 않음.
- 객체 키 예시: w/{workspaceId}/p/{projectId}/original/{assetId}.webp, w/.../generated/{assetId}.png; 브라우저에는 키 대신 단시간 유효 서명 URL을 반환.
- 실제 객체 키는 외부 제공 파일명/경로를 그대로 쓰지 않음.
- 원본은 SHA-256, 이미지 치수/방향, EXIF 처리 정책, 지워진 메타데이터 여부를 저장.
- 가격 기록은 정수 크레딧 또는 소수점 고정 Decimal로 표현하여 JS float 누적 오차 금지.

## 5. API 설계

### 5.1 공통 규칙

- Prefix: **/api/v1**; 서버 세션 + 활성 workspace 권한 필수.
- Mutation에서 X-Idempotency-Key(클라이언트 생성 UUID)를 이용하는 엔드포인트는 **body hash까지 저장**하여 같은 키/다른 본문 요청이면 409.
- Cursor 기반 페이지네이션; createdAt/ID 복합 커서. 클라이언트의 임의 offset에 의존하지 않음.
- JSON 응답, ISO 8601 UTC, 문자열 id. 민감 값/token은 응답·로그에 넣지 않음.
- 긴 작업은 202 Accepted + operation/job ID. 브라우저는 폴링 기본, SSE는 최적화.
- DTO 입력/출력은 Zod runtime 검증, unknown 필드 거부 또는 명시적 허용.
- HTTP: 400 잘못된 입력, 401 미인증, 403 권한 부족, 404 다른 테넌트 포함 감춤, 409 충돌/버전 불일치, 422 검증 사실 누락, 429 예산·속도 제한, 5xx 의존성 장애. 예산 부족은 별도 앱 에러코드.

### 5.2 주요 엔드포인트 제안

| Method | Route | 목적 | 권한 / 특기 |
|---|---|---|---|
| POST | /workspaces | 생성 | 로그인 |
| GET | /workspaces/:id/projects | 목록 | member |
| POST | /workspaces/:id/projects | 생성 | owner/editor |
| GET/PATCH | /projects/:id | 상품 정보 조회/수정 | member + owner/editor |
| POST | /projects/:id/uploads/init | 업로드 서명 URL 발급 | 최대 허용 MIME/byte 지정 |
| POST | /projects/:id/uploads/complete | 실제 저장 객체 검증·Asset 등록 | checksum 및 MIME 재확인 |
| PUT | /projects/:id/facts | 검증 사실 수정/확인 | 버전 충돌 검사 |
| PUT | /projects/:id/feature-locks | 변형 금지 특징 | 기존 생성 결과 경고/무효화 규칙 |
| POST | /projects/:id/generation-quotes | 품질/수량/참조 대상 예상 비용 | 저장된 quote/TTL |
| POST | /projects/:id/generation-batches | 예약+작업 요청 | Idempotency-Key 필수 |
| GET | /generation-batches/:id | 진행률/부분 결과/실패 이유 | member |
| POST | /generation-jobs/:id/cancel | best-effort 취소 | 실행 공급자 취소 보장 아님 |
| POST | /assets/:id/reviews | 승인/거절 및 이유 | reviewer permission |
| POST | /assets/:id/regenerations | 원본 참조+실패 이유 재생성 | quote/비용 승인 |
| POST | /projects/:id/pdp/documents | 승인 자산으로 PDP 생성 | 명시적 생성 요청 |
| GET | /pdp/documents/:id | 현재 revision/ETag | member |
| PATCH | /pdp/documents/:id | patch 기반 편집/자동저장 | If-Match 필요 |
| GET/POST | /pdp/documents/:id/revisions | 버전 조회/복구 | member/editor |
| POST | /pdp/documents/:id/exports | 렌더 작업 요청 | export job |
| GET | /exports/:id | 결과 다운로드 URL/유효기간 | member |
| POST | /channels/:platform/connect/start | OAuth/연동 시작 | owner/admin, M1/M2 |
| GET | /channels/:id/products | 외부 기존 상품 조회 | connection owner |
| POST | /publish-previews | diff + 검증 결과 | M1/M2 |
| POST | /publish-attempts | 명시적 승인 토큰으로 게시 | elevated auth, M1/M2 |
| GET | /publish-attempts/:id | 게시 상태·재조회 결과 | member |
| GET | /billing/usage | 예약·확정 사용량/잔액 | owner/admin |
| GET | /health/live | Liveness | 최소 비공개 또는 방화벽 제한 |
| GET | /health/ready | Dependency readiness | 내부 접근 |

### 5.3 생성 요청 예제

~~~json
{
  "projectId": "project-uuid",
  "quoteId": "quote-uuid",
  "referenceAssetIds": ["front-uuid", "side-uuid", "detail-uuid"],
  "variantId": "variant-uuid",
  "shots": [
    {
      "type": "MODEL_WEAR",
      "count": 4,
      "aspectRatio": "4:5",
      "quality": "high",
      "presetId": "studio-natural-uuid"
    },
    {
      "type": "PRODUCT_SOLO",
      "count": 2,
      "aspectRatio": "1:1",
      "quality": "medium"
    }
  ],
  "approvedCostCeilingCredits": 120
}
~~~

- 서버는 quote의 plan hash, 액세스 가능한 원본, 현재 feature lock 및 잔액을 검사 후 reservation 기록.
- 응답 예시: HTTP 202 + batchId, jobCount=6, status=QUEUED, reservedCredits, statusUrl.
- count와 총 예상 크레딧은 서버가 계산하며 클라이언트가 보낸 총액을 신뢰하지 않음.

### 5.4 표준 오류

~~~json
{
  "error": {
    "code": "PRODUCT_FACTS_UNVERIFIED",
    "message": "상세페이지에 필요한 제품 소재 정보가 아직 확인되지 않았습니다.",
    "fieldErrors": [{ "path": "facts.material", "code": "REVIEW_REQUIRED" }],
    "retryable": false,
    "requestId": "req-uuid"
  }
}
~~~

문서 편집 실패는 EDIT_CONFLICT, 결제는 BUDGET_EXCEEDED, AI 공급자 오류는 PROVIDER_UNAVAILABLE, 채널은 CHANNEL_AUTH_EXPIRED/PUBLISH_VERIFICATION_PENDING 같은 안정된 도메인 코드 사용. 내부 stack trace는 반환하지 않는다.

## 6. 이미지 생성/검수 파이프라인

### 6.1 입력 / 제품 특징 분석

1. 브라우저가 이미지 MIME/크기 사전 검사 → 서버에서 짧은 수명의 업로드 URL 생성.
2. 객체 저장 완료 후 서버가 실제 헤더·매직 바이트·용량·가로세로·권리 고지 확인.
3. 원본 SHA 저장, EXIF 위치정보 제거를 기본값으로 하고 회전 방향 정규화. 원본 보관 정책과 공개·파생물 처리 구분.
4. AI 특징 추출(추론): 카테고리, 제품/스톤/체인 형태, 장식·클라스프 위치, 색상, 주의 부위.
5. 사용자 화면에서 추론값 확인/수정 → verified_fact + feature_lock 스냅샷 고정.
6. 모델/샷/배경/프롬프트 사양과 원본의 역할(상품 참조/무드 참조/모델 참조)을 명시.
7. 입력 이미지는 최종 고객의 권한 범위에서만 공급자에 전달; 만료 URL·파일 ID 등 구현 방식은 공급자별 결정.

### 6.2 단계별 상태 흐름

~~~mermaid
stateDiagram-v2
  [*] --> QUOTED
  QUOTED --> RESERVED: confirm + idempotency
  RESERVED --> QUEUED
  QUEUED --> RUNNING
  RUNNING --> GENERATED: provider success + stored
  RUNNING --> RETRY_WAIT: transient error
  RETRY_WAIT --> RUNNING
  RUNNING --> OUTCOME_UNKNOWN: timeout / lost response
  OUTCOME_UNKNOWN --> GENERATED: reconcile
  OUTCOME_UNKNOWN --> FAILED: provider confirmed failed
  GENERATED --> NEEDS_REVIEW
  NEEDS_REVIEW --> APPROVED
  NEEDS_REVIEW --> REJECTED
  REJECTED --> QUOTED: regenerate new job
  QUEUED --> CANCELLED: cancel
  RETRY_WAIT --> CANCELLED: cancel
  APPROVED --> [*]
  CANCELLED --> [*]
  FAILED --> [*]
~~~

PRD의 외부 표시는 QUEUED/RUNNING/SUCCEEDED/FAILED/CANCELLED의 단순 상태를 유지할 수 있으나, 내부 시도 상태와 자산 승인 상태는 분리한다.

### 6.3 작업 생성과 멱등성

- API transaction 1: quote 유효·가격/한도 검증, reservation + generation_batch + generation_jobs 행 작성. unique(client_key).
- transaction commit 후 **Outbox 이벤트** 발행 → dispatcher가 Redis 큐 등록. API→Redis 동시 성공에 의존한 dual-write 회피.
- Worker가 job claim 후 attempt 생성, 공급자 호출 전/후 시점 기록.
- Job 완료로 받은 파일은 임시 키에 올려 검증 후 최종 Asset 생성. 동일 job/attempt 결과가 두 번 도착해도 정산/Asset 중복 방지.
- Worker dead/redeploy 시 DB 상태와 BullMQ 작업을 대조하는 reconciliation 스케줄러 필요.
- 재시도: 429, 일부 5xx, 일시 네트워크 장애에만 제한된 지수 백오프 + jitter. 잘못된 상품정보/권리/정책 오류는 자동 재시도 금지.
- **요청 타임아웃 후 공급자 측 성공/실패가 미확인**이면 OUTCOME_UNKNOWN에 보류. 멱등 지원 여부 확인 전 원본 작업 재전송 금지 (이중 비용 위험). 공급자 조회 기능 부재 시 운영자 검토/보수적 환원 정책 마련.

### 6.4 이미지 공급자 추상화

~~~ts
type ShotType = "MODEL_WEAR" | "PRODUCT_SOLO" | "DETAIL" | "LIFESTYLE";
type GenerationMode = "EDIT_REFERENCE" | "GENERATE" | "COMPOSITE";

interface ImageGenerationRequest {
  jobId: string;
  mode: GenerationMode;
  shotType: ShotType;
  referenceInputs: Array<{ assetId: string; role: "PRODUCT" | "STYLE" | "MODEL" }>;
  lockedFacts: Array<{ key: string; value: unknown; severity: "CRITICAL" | "MAJOR" }>;
  prompt: string;
  promptVersion: string;
  size: { width: number; height: number };
  quality: "low" | "medium" | "high";
  costCeilingCredits: number;
}

interface ImageGenerationProvider {
  getCapabilities(): Promise<{ multipleReferences: boolean; mask: boolean; transparency: boolean }>;
  estimateCost(request: ImageGenerationRequest): Promise<{ estimatedCredits: number }>;
  generate(request: ImageGenerationRequest): Promise<{
    outputs: Array<{ bytes: Uint8Array; mime: "image/png" | "image/jpeg" | "image/webp" }>;
    model: string;
    requestId?: string;
    providerUsage?: Record<string, unknown>;
  }>;
}
~~~

**현재 구현 시점 검토할 OpenAI Image API:** 생성/편집 엔드포인트 및 GPT Image 모델을 지원한다. 멀티 참조를 사용하되 공급자별 이미지 개수·해상도·마스크 형식·가격 상이. 공식 [이미지 편집 API](https://developers.openai.com/api/reference/resources/images/methods/edit)와 [이미지 생성 가이드](https://developers.openai.com/api/docs/guides/image-generation)에서 현재 한도를 확인한다. **모든 공급자가 마스크·투명 배경·동일한 output 수량을 지원한다고 가정하지 않는다.**

### 6.5 상품 동일성 (Fidelity) 전략

완벽한 텍스트 프롬프트만으로 장식 개수, 체인 잠금장치, 스톤 배열 등을 보장하지 못한다.

- **Level A / Source:** 다각도 참조 원본, 색상별 원본, 실제 크기(선택), 잠금 특징을 명시.
- **Level B / Prompt:** 변형 불가 특징 목록을 생성 프롬프트의 별도 블록으로 전달; 모델·무드 참조는 상품 참조와 분리.
- **Level C / Pipeline:** 필요하면 제품 누끼를 정확한 실물 오버레이로 합성하고 인물/배경 영역만 생성하는 COMPOSITE 모드도 PoC.
- **Level D / Automated QA:** 원본·생성 컷에서 형태/색상·부품/로고·중요 장식 위치·인체 오류를 비교해 PASS/WARN/FAIL 후보 표기.
- **Level E / Human Review:** 최종 승인/거절·사유 기록이 없는 자산은 상세페이지 자동 선택·게시 기본 목록에서 제외.

QA 모델은 **결정 보조 도구**이고 제품이 동일하다는 객관적 보증이 아니다. FAIL 후보를 자동으로 삭제하지 않고 사용자에게 이유/비교를 제공. 색상 옵션은 Variant별 생성·잠금 특징으로 분리.

### 6.6 프롬프트 구성

~~~text
[상품 참조 역할] 정면: asset-a, 측면: asset-b, 디테일: asset-c
[판매자가 확인한 제품 사실] 소재=925 silver, 구성=4mm round 4-prong ... (실제 검증된 경우만)
[절대 변경 금지] 스톤 개수·배열, 귀걸이 힌지, 체인/클라스프, 금속 색상, 장식 위치
[변경 가능] 모델, 포즈, 의상, 배경, 화각, 자연광, 표현 무드
[샷 타입] MODEL_WEAR / 귀걸이 귓불에 자연스럽게 위치
[금지 요소] 허위 로고, 상품 추가, 글자/워터마크, 모델 얼굴 무단 실존 인물 모사
[출력] aspectRatio=4:5, 품질 설정, promptVersion=v1
~~~

동일 상품/동일 프리셋이라도 재생성 시 결과가 달라질 수 있다. 고정 모델/시드 지원은 공급자별로 조사하며 결과를 약속하지 않는다.

## 7. 비용 / 크레딧 / 정산 트랜잭션

**실제 API 공급자 원가와 사용자 크레딧은 다른 단위다.** 비용표/환율/마진 정책에 따라 변환.

- quote: 설정·모델·해상도·참조 이미지 개수/크기·재생성 포함 여부를 보고 보수적으로 예측. TTL과 priceVersion 부여.
- reserve: 사용자가 최종 확인한 상한 이내만 크레딧 홀드.
- settle: 작업 완료 시 측정 가능한 사용량과 과금 규칙에 따라 사용 크레딧 확정. 차액 해제.
- fail/refund: 공급자 실패 후 실제 비용과 사용자 과금 정책 차이 명시. 공급자 호출 후 결과 미확인 상태는 보류·운영 처리.
- **이중 차감 방지**: workspace_id + job_id + charge_kind unique; ledger는 append-only이며 같은 이벤트 중복 소비를 무시.
- 백엔드 예산 한도: 사용자별 하루/월 최대 이미지/최대 크레딧/전체 Provider spending guard.
- 승인 전 미리보기에는 견적 범위·한도·단위와 무료/유료 정책을 표시.
- 운영 로그에는 provider pricingVersion과 실제 소비량을 보관하고, 향후 요금 변동 시 과거 내역을 재산정하지 않는다.
- 악의적 반복 요청/비정상 이용을 고려한 서버 측 동시 생성량 상한, 무료 체험 제한, 차단 로직.

## 8. 편집 가능한 PDP 문서 — Data Contract

### 8.1 Source of Truth

PDP의 원본은 **PDPDocument JSON**이다. Export PNG/HTML/ZIP와 채널 게시본은 **고정된 revision에서 파생**되는 산출물이다.

- 고정된 공통 스키마에 대한 SchemaVersion 관리.
- asset ID는 사내 private assets의 signed URL에 직접 종속되지 않고, 렌더러가 적법한 공개 URL 또는 서버 로딩 방식으로 resolve.
- 에디터는 구조화된 section 순서를 유지. 블록 내부 제한된 배치·오버레이(타이틀 배지 등) 허용.
- AI 초안은 같은 JSON AST를 생성해야 하며, 임의 raw HTML 문자열을 SoT로 저장하지 않는다.
- Layout style은 허용된 토큰·속성만 지원 (CSS 값 전체 무제한 허용 금지).

### 8.2 문서 예제

~~~json
{
  "schemaVersion": 1,
  "documentId": "pdp-doc-uuid",
  "revision": 7,
  "page": {
    "name": "Silver Tennis Bracelet",
    "baseWidthPx": 860,
    "background": "#FFFFFF",
    "locale": "ko-KR"
  },
  "theme": {
    "fontFamily": "Pretendard",
    "headingColor": "#1F2937",
    "textColor": "#4B5563",
    "accentColor": "#96785F"
  },
  "sections": [
    {
      "id": "sec-hero",
      "type": "SECTION",
      "name": "대표 이미지",
      "locked": false,
      "style": { "paddingY": 48, "background": "#FAF7F2" },
      "children": [
        {
          "id": "node-photo",
          "type": "IMAGE",
          "assetId": "approved-asset-uuid",
          "props": { "fit": "cover", "alt": "상품 대표 이미지" },
          "style": { "width": "100%", "aspectRatio": "4:5" }
        },
        {
          "id": "node-title",
          "type": "TEXT",
          "props": { "text": "매일의 빛을 더하는 실버" },
          "style": { "fontSize": 34, "textAlign": "center" }
        }
      ]
    }
  ],
  "metadata": {
    "originProjectId": "project-uuid",
    "sourceAssetIds": ["approved-asset-uuid"]
  }
}
~~~

숫자/폰트/색상은 설계 예시이며 운영 템플릿의 확정 디자인이 아니다. 스키마 검증기는 **asset 참조 유효성, 허용된 스타일, 중복 node ID, 순환 참조, 섹션 크기, 검증되지 않은 사실**을 검사한다.

### 8.3 편집 명령·저장 프로토콜

- UI state(선택, hover, 확대, 임시 drag)은 DB에 저장하지 않고 로컬 editor store에 유지.
- 사용자 변경은 CreateNode, UpdateProps, ReorderNode, MoveNode, DeleteNode, SetSectionStyle, ReplaceAsset 명령으로 표현.
- 명령은 undo stack에 저장. Undo/Redo는 **편집 중 로컬 히스토리**, 문서 Revision은 **장기 버전**으로 분리.
- 자동 저장: 입력 중 debounce(예: 700–1200ms, PoC로 조정), 텍스트/composition 이벤트에 영향 없게 설계.
- 서버 PATCH 요청에 baseRevision / If-Match 사용; 서버는 예상 Revision 비교 후 하나의 atomic update+snapshot 작성.
- 저장 충돌(409): 조용히 덮어쓰기 금지. 변경 비교·내 변경 복사·최신본 불러오기/수동 병합 선택.
- 자산 삭제가 이미 게시된 revision을 깨뜨리지 않게 soft delete와 referential checks 사용.
- 게시 버전은 불변 revision hash를 참조하며 편집하는 동안 이미 게시된 버전이 자동 덮어쓰이지 않음.

### 8.4 WYSIWYG 렌더링 / Export

- 브라우저 편집기와 출력 워커는 가능한 범위에서 **같은 문서 컴포넌트/토큰과 폰트 파일** 사용.
- Export worker는 특정 document revision을 로드, 검증, 서버 내부 렌더 URL의 접근 통제 후 **고정 Chromium/viewport/DPR**로 렌더.
- 긴 상세페이지는 브라우저/이미지 도구의 최대 높이 제한을 고려해 **섹션 경계 우선 청크 이미지**로 내보내고, 요청에 따라 최종 결합 이미지 제공(한도 내).
- 외부 공개 HTML은 스크립트 없는 allowlist fragment 또는 자체 저장된 영속 이미지로 전환; 스토어 HTML 정책 검증 전 판매자에게 무조건 호환이라고 약속하지 않는다.
- 반응형 출력을 지원하더라도 채널별 **고정 너비 PDP 프리셋**을 우선 제공하고 미리보기에서 동일 크기를 사용.
- 이미지 preload/load 완료 대기와 폰트 로딩 완료 후 screenshot; 누락 시 fail-fast.
- Export artifact마다 revisionSha, rendererVersion, fontVersion, generatedAt, dimensions 기록.
- 브라우저와 Export 간 위치/줄바꿈/색상 편차는 [품질계획](./QUALITY_PLAN.md)의 visual snapshot 테스트로 관리.

## 9. 채널 어댑터 / 게시 거래

### 9.1 채널 독립 인터페이스

~~~ts
interface CommerceAdapter {
  connect(context: AuthorizationStart): Promise<{ redirectUrl: string }>;
  listProducts(context: ConnectedMerchant, cursor?: string): Promise<ProductPage>;
  getProduct(context: ConnectedMerchant, productId: string): Promise<ExternalProduct>;
  validatePatch(target: ExternalProduct, output: ChannelRenderedContent): ValidationResult;
  previewDiff(target: ExternalProduct, output: ChannelRenderedContent): ContentDiff;
  applyPatch(context: ConnectedMerchant, productId: string, patch: ContentPatch): Promise<RemoteMutationResult>;
  verify(context: ConnectedMerchant, productId: string, expected: ContentPatch): Promise<VerificationResult>;
}
~~~

- Cafe24와 Naver는 인증/상품 데이터 모델이 다르므로 토큰 처리, 상세 콘텐츠 업데이트 필드, 이미지 업로드 로직을 **별도 구현**.
- 상품/옵션/재고/가격 전체를 PUT해야 하는 채널은 서버 최신 재조회 후 **변경 허용 필드만 변경**하고 무관 필드 원본 보존 검증을 수행.
- 멀티 쇼핑몰 / 스토어 계정마다 외부 ID를 독립 매핑.
- **권리/상품정보 검증 부족, 토큰 만료, API 승인 미완료**는 publish preview/commit 진입 차단.
- **외부 판매자에게 스토어 Client Secret을 요구하는 비공식 우회 금지**.

### 9.2 게시 2단계 절차

~~~mermaid
sequenceDiagram
  participant U as User
  participant API as SELLAYER API
  participant DB as DB
  participant AD as Channel Adapter
  participant EXT as Cafe24 / Naver
  U->>API: Preview documentRevision + targetProduct
  API->>AD: Get current product
  AD->>EXT: GET product
  EXT-->>AD: Current content/version
  API->>DB: Store before snapshot + prepared patch
  API-->>U: Render preview + changed fields + confirmation token
  U->>API: Confirm exact preview token
  API->>DB: Validate hash / auth / expiry / mark approved
  API->>AD: Apply allowlisted patch
  AD->>EXT: Update allowed content
  EXT-->>AD: API response
  API->>AD: Re-read changed product
  AD->>EXT: GET product
  EXT-->>AD: Current content
  API->>DB: Verify, audit, after snapshot
  API-->>U: VERIFIED or remediation details
~~~

- Preview token은 short-lived, workspace/actor/channel/externalProductId/revisionSha/contentPatchHash/expiry에 결합한다.
- 외부 호출 직전 상품을 재조회하여 preview의 baseHash와 달라졌으면 **409 재검토** 처리.
- 외부 API 호출 성공만으로 VERIFIED 표시 금지. 비동기 반영 플랫폼은 VERIFICATION_PENDING을 표시하고 재조회 제한.
- publish idempotencyKey 동일 요청 중복 승인 시 단일 publish_attempt 반환.
- rollback은 보관한 이전 스냅샷으로 **복구 가능 여부 확인·별도 승인**을 거치며 무조건 100% 자동 복원 보장 금지.

## 10. 성능 · 확장 · 장애 설계

### 10.1 운영 가정

- Pilot 기준 하루 100장/워크스페이스를 대상으로 큐 설계를 검증; 전체 서비스 동시 사용자/월 PV는 TBD.
- 워커 동시성은 Provider별 QPS·토큰·일일 비용 제한에 맞춰 동적으로 설정.
- 웹 트래픽 급증과 AI 호출 비용이 함께 증가하지 않도록 **요청 접수 API와 처리 워커 분리**.
- 렌더/이미지 툴은 메모리 큰 작업으로 별도 concurrency, CPU/memory limit 지정.
- VPS 최소 CPU/RAM/디스크는 **현재 Hostinger 플랜 미확인**: 실사양 측정 후 필요시 worker 또는 exporter 별도 머신 이전.

### 10.2 장애 시나리오

| 고장 | 시스템 반응 | 복구 |
|---|---|---|
| AI 공급자 429 | rate-limit 기반 backoff, job progress 업데이트 | 제한 회복 후 지속 |
| 공급자 결과 미확인 | OUTCOME_UNKNOWN, 중복 과금 위험 경고 | 공급자 조회/운영 재조정 |
| Redis 재시작 | 예약/작업 DB에서 re-enqueue | idempotency + outbox replay |
| 워커 재시작 | RUNNING lease expiry 감지 | 같은 시도 재전송 안전성 확인 |
| DB unavailable | 조회/수정/게시 fail closed | DB 복구, 큐 소비 정지 |
| Object Storage unavailable | 새 생성/내보내기 큐 대기 | 공급자 호출 전에 확인 |
| 편집 저장 충돌 | 에디터 409 및 복구 UI | 최신 revision 기준 rebase/manual |
| 게시 중 연결 만료 | READ-only 상태 또는 FAILED | 고객 재인증 및 새 preview |
| Export 이미지 로드 실패 | EXPORTED 성공 처리 금지 | 소스 복구 / 작업 재실행 |
| 배포 중 schema mismatch | rollout 중단, 이전 버전 유지 | 확장-마이그레이션 우선 |

### 10.3 데이터 보존과 복구

- DB PITR/일일 백업 전략(선택한 관리형/자체호스팅 구성에 따라 확정), **VPS 외부로 암호화 보관**.
- 비공개 원본/승인본의 삭제 정책과 사용자 탈퇴/환불/법적 보존 기준을 구분.
- 삭제 요청은 assets + derived exports + 공개 URL/CDN 캐시까지 처리 가능한 범위 명시.
- 암호화 키/DB 자격증명 복구 테스트 및 백업 복호화 복원 테스트 주기 설정.
- 영속 자산을 VPS 컨테이너 레이어 안에 보관하지 않는다.

## 11. 테스트 전략 (요약)

- Contract: DTO(schemaVersion 포함), provider capabilities, 채널 어댑터 fake + 테스트몰 sandbox.
- Unit: 특징 Lock 적용, 필수 사실 검증, 레이아웃 AST, undo/redo, ledger sum/idem.
- Integration: Postgres+Redis+object storage + outbox/recovery, 실패/중복/크레딧 환원.
- Browser E2E: 로그인 → 원본 → 생성 mock → 승인 → PDP 편집 → export → 복구.
- Visual Regression: 브라우저 편집/서버 출력 이미지 diff (글자 줄바꿈·폰트·수평 가이드).
- Security: IDOR/CSRF/SSRF/upload content spoofing, 시크릿/토큰 redaction, 관리자 권한, HTML/XSS.
- Domain Quality: 실제 주얼리/패션 앵글별 색상·잠금장치·보석/단추/형태 검수, 결과 승인 비율 및 비용.
- Channel smoke: Cafe24 M1, Naver M2에 **권한 있는 테스트 상점과 더미 상품**만 사용. 가격/재고 불변 검사.

상세 테스트 데이터셋과 출시 게이트는 [QUALITY_PLAN.md](./QUALITY_PLAN.md)에 기록한다.

## 12. 초기 구현 순서

1. Monorepo / 품질 도구 / DB migration + workspace guard / 인증
2. Asset 업로드 및 불변 저장, Project/Variant/Fact/Lock CRUD
3. 생성 quote/reservation/ledger + outbox + 큐 + provider fake
4. 실제 이미지 API PoC + 생성 공급자 어댑터, 추론/QA + gallery review
5. JSON PDP schema v1 + generator + editor renderer + history/patch
6. Export worker와 visual parity, ZIP/PNG 출력
7. 운영/사용량/보안/백업 구축 → Private Alpha Gate
8. Cafe24 공식 연결·검증 가능한 상품 콘텐츠 업데이트 → M1
9. Naver 솔루션 승인 뒤 커넥터 → M2
10. 배치 모델 최적화와 추가 채널 → M3

## 13. 결정 보류 / 기술 검증 항목

| 항목 | 후보 | 결정 위한 증거 |
|---|---|---|
| 에디터 절대좌표 vs 블록 | DOM 블록 + 제한 overlay (제안) | HTML/PNG 재현성 PoC |
| 이미지 모델 | OpenAI GPT Image 계열/다른 상용 모델 | 실제 상품 Fidelity/단가/지연 |
| 객체 저장소 | R2, S3, Hostinger 제공 S3 호환 | 비용, 가용 리전, URL/CORS, 인증 |
| DB 배포 | VPS Postgres vs 관리형 | Hostinger 자원/운영 역량/백업 |
| 인증 | 검증된 세션 라이브러리 | OAuth·보안·결제 플로우 |
| 배포 전략 | GHCR + SSH 배포 (제안) | 접근 제한, staging, rollback 가능성 |
| QA 모델 | 규칙 기반 + 비전 모델 보조 | 30~50 SKU 실제 평가 정확도 |
| 데이터 보존 | 계정/상품/원본별 정책 | 법률/환불/상업 라이선스 검토 |

## 14. 참고 문서

- [OpenAI Image Edit API](https://developers.openai.com/api/reference/resources/images/methods/edit)
- [OpenAI Image Generation Guide](https://developers.openai.com/api/docs/guides/image-generation)
- [Next.js 인증·인가 가이드](https://nextjs.org/docs/app/guides/authentication)
- [GitHub Deployments/Environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)
- [Cafe24 OAuth code](https://developers.cafe24.com/en/app/front/app/develop/oauth/oauthcode)
- [Naver Commerce API](https://apicenter.commerce.naver.com/docs/introduction)

**이 문서는 구현을 지시하는 설계 기준안이며, 실제 공급자 품질/요금/외부 플랫폼 권한/서버 스펙은 아직 검증되지 않았다.**
