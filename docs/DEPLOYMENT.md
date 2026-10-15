# SELLAYER — Hostinger VPS / GitHub Actions 배포 상세 설계

> **문서 ID:** SL-OPS-001 · **버전:** v0.1 · **작성일:** 2026-10-16 (KST)  
> **상태:** **배포 설계만 완료된 초안** — 워크플로/컨테이너/실서버 환경은 아직 구축하지 않음.  
> **대상:** Hostinger VPS · Ubuntu 24.04 · GitHub Repository jskjw157/SELLAYER (현재 공개 저장소)  
> **선행:** [기술 설계](./TECHNICAL_DESIGN.md) · [PRD](./PRD.md) · [품질 계획](./QUALITY_PLAN.md)

## 0. 목표와 안전 경계

- GitHub PR의 필수 검사가 성공해야 main으로 병합할 수 있도록 **branch protection** 계획.
- main 반영 시 코드/이미지 **동일 커밋에서 빌드된 OCI 이미지**를 생성하고 버전으로 저장.
- Production은 **GitHub Actions Environment 승인 단계**를 통과한 릴리스만 배포하도록 구성.
- 서버에서 수동 git pull + 임의 npm install + 즉시 재시작 방식 대신, **검증된 이미지와 태그/다이제스트로 배포**.
- 정상 배포 전후 서비스 상태/DB 호환성/복구를 확인하며 실패 시 롤백 절차 제공.
- 사용자 원본/생성 이미지와 시크릿/DB 백업은 앱 컨테이너 이미지 및 Git 저장소에 포함하지 않음.
- **제품 개발·PoC 전** 실 VPS에 배포하거나 유료 AI를 호출하지 않는다.

## 1. 사전 점검 (사용자 Hostinger 계정에서 확인)

| 항목 | 확인 질문 / 결과 | 상태 |
|---|---|---|
| Ubuntu | **24.04** | 사용자 확인 |
| Hostinger VPS IP | 아직 제공되지 않음 | TBD |
| VPS CPU/RAM/SSD | 플랜 불명 | TBD (최초 온보딩에서 확인) |
| SSH 접속 | 가능 여부, 포트, 공개키 설정 | TBD |
| 배포 도메인 | sellayer 서비스용 전용 도메인 / subdomain | TBD |
| DNS | A/AAAA 레코드 및 TTL | TBD |
| SSH 배포 계정 | non-root 전용 계정 | TBD |
| AI 공급자 키 | 모델 요금제/Usage spending limit | TBD |
| 객체 스토리지 | R2/S3 등 공급자 및 버킷 | TBD |
| 결제 서비스 | 테스트/실서비스 PG 정책 | M1 이전 TBD |
| 채널 앱 등록 | Cafe24/Naver 심사/권한 | M1/M2 외부 의존성 |

필요한 정보는 **주소/사양/접속 방식 정도**이며, SSH 비밀키, API 키, DB 암호, Cafe24/네이버 client secret을 **채팅으로 공유하지 않는다.**

### 1.1 서버 현황 확인 예시 (배포 전, 읽기 전용)

~~~bash
lsb_release -a
uname -r
nproc
free -h
df -h
ip -brief address
docker --version
docker compose version
sudo ufw status verbose
~~~

없거나 권한이 없는 명령은 환경에 따라 건너뛰며, 모든 패키지 설치/방화벽 변경/데몬 설정은 서버 현황과 접속 복구 경로를 확인한 뒤 수행한다.

### 1.2 VPS 크기와 워커 분리

- Web/API, Worker, Redis, PostgreSQL, Chromium Export를 1대에 둘 수는 있지만 **메모리 피크가 큰 이미지/브라우저 렌더 작업** 때문에 용량 측정이 필수.
- 최소 VPS 스펙 수치를 현재 단계에서 단정하지 않음. Health/부하 테스트 시 실제 메모리와 CPU 소모 기반으로 튜닝.
- object storage를 VPS 로컬 파일시스템과 분리.
- 리소스 부족 시 1차로 Export Worker의 동시성을 제한하고, 2차로 Worker 또는 Postgres를 별도 인스턴스로 이동.
- Redis/DB는 public Docker port publish 금지. 앱 컨테이너 네트워크로만 노출.

## 2. 전체 배포 토폴로지

~~~mermaid
flowchart TD
  DEV[개발자 GitHub PR] --> CI[GitHub Actions CI]
  CI -->|PASS + review| MAIN[Protected main]
  MAIN --> BUILD[GitHub Actions Build & Scan]
  BUILD --> GHCR[(GitHub Container Registry)]
  BUILD --> APPROVE[Production Environment Approval]
  APPROVE --> SSH[Restricted SSH Deploy]
  SSH --> VPS[Hostinger Ubuntu 24.04]
  GHCR --> VPS
  VPS --> PROXY[Caddy or Nginx TLS Proxy]
  PROXY --> WEB[SELLAYER Web/API]
  WEB --> PG[(Postgres Private)]
  WEB --> RED[(Redis Private)]
  RED --> WRK[SELLAYER Worker]
  WRK --> OBJ[(Private Object Storage)]
  WRK --> AI[Image Provider API]
  VPS --> BACKUP[(Encrypted Off-VPS Backups)]
  VPS --> LOG[Metrics + Alerts]
~~~

**초기 기본안**: GHCR 이미지를 빌드하고 서버가 digest/SHA 버전을 pull해서 Compose로 실행. 서버 SSH로 소스코드를 다운받아 실시간 빌드하지 않는다.

### 2.1 환경 분리

| 환경 | 용도 | 데이터 | 도메인 |
|---|---|---|---|
| Local | 개발/단위 테스트 | Mock/synthetic | localhost |
| CI | 통합/보안/브라우저 검증 | ephemeral DB/Redis, fake providers | GitHub-hosted runner |
| Staging | 실제 export·API smoke 및 릴리스 검증 | 별도 테스트 데이터/계정 | TBD (Hostinger 자원 필요) |
| Production | 실제 사용자 | 별도 DB, 별도 버킷 prefix/credentials | TBD |

Staging을 같은 VPS에 둘 경우 DB/Redis/객체 저장소를 완전히 분리하고 자원 여유를 측정해야 함. 불가능하면 GitHub Actions ephemeral 테스트 + 별도 staging 환경 선택을 제품 출시 전 결정.

## 3. GitHub 브랜치 / 배포 정책

### 3.1 브랜치 전략

- main: 프로덕션 배포 기준선. 직접 push 제한, PR 승인 + required checks.
- feat/*, fix/*: 기능 단위 브랜치.
- docs/*: 설계/문서.
- hotfix/*: 장애 수정 PR + 최소 동일 검증.
- release image: git SHA 태그와 digest를 기록. latest 단독 배포는 금지.
- production deploy: GitHub Environment approval; 주요 시스템 변경은 maintainer reviewer와 체인지 로그 필수.

### 3.2 PR 필수 검사 (CI)

| Job | 실행 | 실패 조건 |
|---|---|---|
| lint | ESLint, format | 코드 스타일/위반 |
| typecheck | TypeScript, schema generation | 타입 오류 |
| unit | 도메인/ledger/editor test | 불변식 깨짐 |
| integration | Postgres/Redis/스토리지 fake, outbox 테스트 | 컨테이너 간 계약 오류 |
| security | lockfile audit, secrets scan, upload/IDOR tests | 위험 패키지/권한 결함 |
| editor-visual | Playwright 고정 폰트 snapshot | 출력 일치 회귀 |
| build | web/worker production build, Dockerfile build | 빌드/마이그레이션 호환 실패 |
| contract | image provider/채널 mock contract | API 계약 호환 실패 |

AI 유료 Provider 테스트는 main CI에서 **기본적으로 실행하지 않음**. PoC 전용 승인된 수동 workflow에서만 실행하고 예상 비용 한도를 둔다.

### 3.3 GitHub Actions 파일 계획

~~~text
.github/workflows/
  ci.yml                  # pull_request, main
  publish-images.yml      # main, git SHA -> GHCR
  deploy-staging.yml      # 필요 시
  deploy-production.yml   # environment: production + 승인
  backup-check.yml        # 백업 무결성/복구 리허설 (리소스 정책 따라)
~~~

각 워크플로가 구현되기 전까지는 위 파일이 **실제로 존재한다고 가정하지 않는다**.

## 4. CI / 배포 워크플로 구체화

### 4.1 CI 의사 YAML (실제 파일은 구현 단계)

~~~yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@PINNED_COMMIT_SHA
      - uses: pnpm/action-setup@PINNED_COMMIT_SHA
      - uses: actions/setup-node@PINNED_COMMIT_SHA
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
~~~

- PINNED_COMMIT_SHA는 예시 자리표시자. workflow를 만들 때 공식 Action의 검증된 전체 SHA를 고정.
- 실제 저장소의 package.json/스크립트가 만들어진 뒤 CI 단계/경로 확정.
- 브라우저 E2E는 별도 job과 테스트용 컨테이너 의존성 정의. 외부 AI 유료 생성/스토어 실상품 변경 금지.
- PR from fork는 production secrets 접근 차단.

### 4.2 이미지 빌드 / 레지스트리

1. main의 정확한 commit SHA를 checkout.
2. web, worker 각각 Dockerfile build; build-time secret 최소화, 취약점 스캔.
3. GHCR 인증은 Actions 기본 GITHUB_TOKEN + packages:write 최소 권한.
4. 이미지 태그: ghcr.io/OWNER/sellayer-web:COMMIT_SHA, sellayer-worker:COMMIT_SHA.
5. SHA와 이미지 **digest**를 release manifest로 저장. 배포는 digest를 우선 사용.
6. 필요하면 SBOM/provenance attestation과 의존성 스캔 결과 첨부.
7. 같은 커밋을 재배포해도 같은 artifact를 사용하는 불변 릴리스 전략.

### 4.3 Production 단계

~~~mermaid
sequenceDiagram
  participant GH as GitHub Actions
  participant CR as GHCR
  participant VPS as Hostinger Deploy Service
  participant DB as PostgreSQL
  participant APP as SELLAYER Containers
  GH->>CR: Build/push immutable images
  GH->>GH: Required checks + environment approval
  GH->>VPS: SSH deploy approved digest
  VPS->>DB: Backup/check + additive migration
  VPS->>CR: Pull specific image digest
  VPS->>APP: Start new web/worker release
  VPS->>APP: /health/ready smoke
  alt healthy
    VPS-->>GH: Record successful release SHA
  else unhealthy
    VPS->>APP: Restore previous app images
    VPS-->>GH: FAIL + alert
  end
~~~

- 배포 스크립트는 서버에서 **flock 잠금**을 잡아 동시 배포 방지.
- 변경된 config/manifest는 사전에 검증하고 prod 현재 릴리스 SHA/digest 기록.
- Docker Compose는 웹/워커/DB/Redis 등 서비스별 healthcheck, restart policy, volume, memory cap, internal network 사용.
- 배포 중 생성/예약 작업은 DB 상태 머신과 outbox에 의해 복원. 배포로 작업이 중복 전송/청구되지 않도록.
- Worker graceful shutdown: 새 작업 수락 중단, 진행 중 작업 lease 및 공급자 결과 확인, 강제 종료 시 OUTCOME_UNKNOWN.
- DB migrate는 **expand-contract 패턴** (신규 컬럼 추가 → 앱 호환 → 나중에 컬럼 제거). 롤백 시 DB destructive down migration을 즉시 실행하지 않음.
- 새 릴리스 /health/ready + 내부 종단 smoke test + 큐 프로세스 확인 후 성공 선언.
- 실패 시 마지막 정상 이미지로 롤백하되 DB가 이전 앱과 호환되어야 함. 복원이 필요한 migration이면 사고 복구 플랜으로 분리.

## 5. Docker Compose 서비스

### 5.1 제안 topology

~~~text
docker-compose production project: sellayer
├── reverse-proxy
│   └── HTTPS :443 → web :3000
├── web
│   ├── SERVER-SIDE API
│   └── Editor + Seller UI
├── worker
│   ├── AI generation queue
│   ├── QA / image normalization
│   └── Export jobs (memory constrained)
├── postgres (internal only)
├── redis (internal only)
└── external services
    ├── S3-compatible storage
    ├── GPT Image / text provider
    ├── Email/Payments (later)
    └── Cafe24/Naver APIs (M1/M2)
~~~

PostgreSQL을 VPS에서 운영할 경우 DB 데이터는 별도 named volume/호스트 데이터 디렉터리에 저장하고 **정기 백업 + 오프사이트 복구**가 필수. Redis는 AOF/정책을 선택하되 비즈니스 정합성은 DB outbox로 복구.

### 5.2 컨테이너 베스트 프랙티스

- 멀티 스테이지 빌드, 고정 버전 Node/Playwright Chromium, 비루트 사용자.
- 필요한 Linux capabilities 최소화, read-only rootfs (가능한 컨테이너), seccomp, no-new-privileges, image scan.
- 공급자 호출과 이미지 처리 워커는 메모리/CPU quota 및 concurrency 제한.
- 컨테이너 내부 업로드/영속 결과물 보관 금지.
- DB·Redis·worker 관리 API 포트 외부 공개 금지.
- Docker socket을 Web/Worker 컨테이너에 mount하지 않음.
- 로그 수집시 개인정보/원본 base64/API 키/스토어 OAuth 토큰 차단.
- 독립적인 liveliness/ready/migration status 확인.

## 6. 도메인·TLS·방화벽

- 사용자 facing 서비스는 별도 HTTPS 도메인 사용(아직 미결정). Cafe24 OAuth 등록은 **정확한 HTTPS redirect URI** 필요.
- Reverse Proxy는 Caddy 자동 TLS 또는 Nginx + Certbot을 평가(둘 중 하나 선택).
- 서버 방화벽/UFW 및 Hostinger 방화벽에 **22(SSH, 관리자 IP/승인 경로), 80(인증/리다이렉트), 443(HTTPS)** 외 최소 노출.
- SSH 키 기반 인증, 가능하면 root 직접 로그인 금지, password login 제한(정상 접근이 확실히 확보된 뒤만 변경).
- IP allowlist / VPN 검토, SSH known_hosts는 서버 콘솔에서 확인한 호스트 키 지문을 Github Secrets/Variables에 등록.
- **StrictHostKeyChecking=no를 배포 편의 때문에 사용하지 않는다.**
- CORS는 승인된 도메인만 허용, SameSite/CSRF/세션·프록시 헤더 구성 확인.
- 객체 스토리지 브라우저 업로드 CORS는 필요한 origin/method/header만 허용.

## 7. Secrets / 권한 분리

### 7.1 필요한 시크릿 (예시 키 이름)

| Secret / Variable | 어디에 저장 | 역할 | 주의 |
|---|---|---|---|
| VPS_HOST | GitHub Environment variable | 서버 접속 주소 | 주소 자체는 비밀 아님 |
| VPS_PORT / VPS_USER | Environment variables | 제한된 배포 계정 | root 피함 |
| VPS_SSH_KEY | GitHub Environment secret | SSH 개인키 | repo, CI 로그에 넣지 않음 |
| VPS_HOST_KEY | Repo/Env variable or secret | 검증된 호스트 지문 | DNS 값만으로 신뢰 금지 |
| GHCR_PULL_TOKEN | VPS 보안 파일 / Docker credential | private package 읽기 | read:packages 최소 범위 |
| DATABASE_URL | VPS secret store | DB 접속 | 웹/워커/마이그레이션 권한 분리 |
| REDIS_URL | VPS secret store | 큐 연결 | public bind 금지 |
| OBJECT_STORAGE_* | VPS secret store | 비공개 자산 읽기/쓰기 | prefix/bucket 최소 권한 |
| OPENAI_API_KEY | VPS secret store | 이미지 생성 API | 판매자 브라우저에 노출 금지 |
| APP_SESSION_SECRET | VPS secret store | 세션 서명/암호화 | 회전 절차 |
| ENCRYPTION_KEY | 별도 안전한 secret store | OAuth 토큰 암호화 | 백업·복구 시 키 보존 |
| CAFE24_CLIENT_SECRET | 서버 only, M1 | OAuth | 채널 승인 전 미사용 |
| NAVER_CLIENT_SECRET | 서버 only, M2 | 솔루션 인증 | 사용자 개인 비밀키 요구 금지 |
| SENTRY_DSN/ALERT_WEBHOOK | 정책에 따라 | 운영 알림 | 고객 데이터 마스킹 |

**불변 원칙:** 이미지 API 비밀키, 판매자 액세스 토큰, DB 패스워드는 GitHub 공개 저장소와 클라이언트 JS bundle에 절대 저장하지 않는다. 채팅에도 붙여 넣지 않는다.

### 7.2 권한 경계

- Github Actions: 소스 읽기/패키지 쓰기 + production deploy environment 전용 키로만 서버 접근.
- SSH Deploy User: 특정 배포 스크립트 호출 범위만. 일반 docker 그룹 권한은 사실상 root에 준하므로 가능하면 root-owned 검증 스크립트/제한된 sudo command allowlist.
- Web service identity: 사용자 요청/읽기 쓰기 최소 DB scope와 스토리지 key.
- Worker identity: 대상 jobs/objects, 공급자 호출/정산 최소 권한.
- DBA/백업 권한: 관리자 전용, 앱 비밀키와 분리 가능 시 별도 계정.

## 8. 서버 디렉터리 / 릴리스 관리 (제안)

~~~text
/srv/sellayer/
├── releases/
│   ├── commit-aaa111/
│   │   └── manifest.json (digests, migration version, deployedAt)
│   └── commit-bbb222/
├── current -> releases/commit-bbb222
├── shared/
│   ├── compose.prod.yml
│   └── proxy/
├── secrets/
│   └── production.env       # owner-only, never git tracked
├── backups/
│   └── local-staging/       # offsite replication REQUIRED
└── logs/
~~~

- secrets 디렉터리 권한 700, env 파일 권한 600 또는 필요 컨테이너 권한에 맞게 최소화.
- current symlink / release manifest는 참고용. 실제 Compose에 image digest가 반영되어 있는지 체크.
- 복구를 위해 최근 두 개 이상의 정상 image digest/환경 메타/backup pointer 보존.
- 실제 운영체제 경로는 디스크 공간/보안 정책 확인 후 결정.

## 9. Observability / 알림

| 측정 | 시스템 | 경고 예시 |
|---|---|---|
| Uptime, API p95 latency | Web/API metrics | 오류율/응답시간 급증 |
| Job backlog / oldest age | Queue monitor | 생성 대기 급증 |
| Worker running/unknown | DB + Worker | 외부 API 결과 미확인 누적 |
| AI estimated vs billed usage | Provider usage | 단위 원가 상승 / 예산 임계 |
| Export failures | Renderer | 폰트 로드 실패, 높이 제한 |
| DB health, disk, backup age | VPS + PG | DB/디스크/백업 이상 |
| OAuth failures | Channel connection | 토큰 갱신·연동 실패 |
| Publish verified failure | Publish log | 상품 반영 오류 |
| Security unauthorized access | App audit | IDOR/비정상 요청 탐지 |

- 로그 필드: requestId, correlationId, jobId, workspaceId(hashed/opaque), elapsedMs, modelVersion, status. 이메일/실제 원본 이미지/토큰/프롬프트 본문은 로그 정책에 따라 최소화.
- 릴리스 후 synthetic health 스모크와 수동 사용자 journey 점검.
- 이메일/Slack 등 알림은 운영 채널 선택 후 구현.

## 10. 백업·복구

### 10.1 백업 항목

- PostgreSQL: 데이터 + schema/migration 버전 + 주기적 consistent dump/백업. RPO/RTO는 실제 서비스 공개 전에 수치 결정.
- object storage: bucket lifecycle/versioning 필요성, 파생물/원본의 보존 정책.
- encrypted OAuth/데이터 키: 키를 백업하지 않으면 DB 복구 시 암호화 토큰 사용 불가.
- Redis: 임시 큐. **복구 SoT는 DB/outbox**, Redis만 백업한다고 job 복구 완료가 아님.
- Nginx/Caddy 설정, service manifest, secret names/rotation docs, TLS 자동 갱신 절차.

### 10.2 복구 리허설

1. 원격 암호화 백업 파일을 **테스트 환경**으로 복원.
2. schema migration 버전/DB 무결성 확인.
3. 승인된/미승인 이미지 object 관계와 서명 URL 접근 확인.
4. 중단됐던 생성 작업을 DB에서 재구성. 공급자 결과 미확인 작업은 자동 중복 호출하지 않음.
5. 이전 앱 digest로 재배포하여 200/ready, 테스트 사용자 세션, Export, queue recovery 검증.
6. 실제 고객 스토어에 임의 publish 요청 금지.

## 11. 운영 체크리스트 / 릴리스 Go-No-Go

### 인프라 준비

- [ ] Hostinger plan의 CPU/RAM/disk와 root console 복구 경로 확인
- [ ] VPS 관리용 백업/스냅샷 및 오프사이트 DB 백업 구성
- [ ] Linux 보안 업데이트 / SSH 접근 최소 권한
- [ ] 도메인과 DNS / HTTPS 인증서 / 갱신 검증
- [ ] Docker Engine / Compose 설치·버전 고정
- [ ] GHCR push/pull 및 deploy account 최소 권한
- [ ] 객체 스토리지·API 공급자 요금 상한
- [ ] Application env/secrets 생성, 로그 노출 테스트

### 코드·운영 검증

- [ ] CI 필수 체크와 protected main 적용
- [ ] DB backward-compatible migration, smoke endpoint
- [ ] image/worker memory limits & queue rate limits
- [ ] staging에서 E2E/Export/ledger 검증
- [ ] 실제 외부 이미지 생성은 명시적 비용 승인 받고 제한된 테스트만
- [ ] 배포 실패 rollback/헬스 상태 점검
- [ ] 1개월 운영계획(장애 연락, 고객지원, 계정 삭제)
- [ ] Cafe24/Naver 배포 전 별도 심사/플랫폼 테스트

## 12. 실제 배포 시 필요한 권한/준비물

- **이 채팅에서 Hostinger VPS의 SSH 접근 권한이 연결되어 있다고 가정하지 않는다.**
- 배포 구현 단계에는 사용자가 자신의 서버에서 안전하게 키/도메인/환경변수를 설정하거나, 허용된 배포 수단으로 연결하는 작업이 필요.
- GitHub에 커밋하는 것은 **서버 배포를 의미하지 않는다**. Workflow 생성과 서버 연결, Healthcheck 확인까지 해야 완료.
- 초기에는 앱 코드/컨테이너가 없으므로 이 문서만으로 실행되는 제품은 없다.
- GitHub 액션 required reviewers/환경 보호 기능은 계정/저장소 공개여부/플랜 조건을 실설정 화면에서 확인.

## 13. 참고 문서

- [GitHub Actions Deployment Environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [GitHub Deploy Control](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments)
- [GitHub Packages / GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Compose Production](https://docs.docker.com/compose/how-tos/production/)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Cafe24 OAuth](https://developers.cafe24.com/en/app/front/app/develop/oauth/oauthcode)

**현 상태**: Hostinger OS만 확인됨(Ubuntu 24.04). 서버 권한/도메인/VPS 사양/앱 코드가 마련되면 이 문서의 체크리스트와 테스트를 충족시켜 배포 파일을 실제 구현한다.
