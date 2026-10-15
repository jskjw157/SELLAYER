# SELLAYER — 상품 이미지·편집기·게시 품질 검증 계획

> **문서 ID:** SL-QA-001 · **버전:** v0.1 · **작성일:** 2026-10-16 (KST)  
> **상태:** Proposed / 품질 기준 후보, 사용자 테스트 전  
> **연계:** [PRD](./PRD.md), [기술 설계](./TECHNICAL_DESIGN.md), [UX Spec](./UX_SPEC.md)

## 1. 목적

SELLAYER 품질 평가는 단순 "예쁜 AI 사진"이 아니라 **실제 상품과 다르게 보이는 이미지로 고객이 잘못 구매하지 않도록 하는 것**과, **편집한 상세페이지가 출력/채널에서도 동일하게 보이는 것**이 핵심이다.

출시 전에 증명할 다섯 가지:

1. **제품 동일성:** 형태, 장식 개수/위치, 재질감·색상 등 핵심 특징이 원본과 크게 다르지 않다.
2. **착용 현실성:** 귀/목/손가락/손목의 위치·크기·조명·인체 표현이 자연스럽다.
3. **사실 정확성:** 소재·치수·원산지·KC/인증 등 확인되지 않은 주장을 생성하지 않는다.
4. **편집/출력 일치:** JSON 상세페이지 문서와 브라우저/PNG/HTML 렌더가 재현된다.
5. **운영 안전성:** 예산·크레딧·데이터 격리·채널 변경 필드가 테스트로 보호된다.

## 2. Golden Dataset (권리 확인된 실상품 테스트 세트)

### 2.1 최소 후보 구성

**권장 시작:** 30~50 SKU를 목표로 하되 확정된 테스트 데이터셋 수는 아님. 주얼리부터 시작하고 의류·잡화는 서비스 확장 전 별도 게이트.

| 그룹 | 예시 | 중요 검수 기준 | 권장 최소 케이스 |
|---|---|---|---|
| 귀걸이 (후프) | 실버/골드, 힌지·오픈링 | 형태·두께·잠금 구조·귓불 착용 위치 | 6 |
| 목걸이 | 굵은 체인, 작은 펜던트 | 고리 모양·길이·펜던트 크기·배치 | 5 |
| 반지 | 스톤 1열·레이어드 링 | 스톤 개수·프롱·밴드 두께·손가락 비율 | 5 |
| 팔찌 | 테니스·비즈·참 | 비즈 개수·볼 참 위치·클라스프·손목 스케일 | 5 |
| 헤어 액세서리 | 반달 집게핀·장식판 | 아치형·중앙 플레이트·집게발 개수/위치 | 5 |
| 색상 옵션 | 동일 SKU 실버/골드/아이보리 등 | Variant별 색상 혼합·누락 | 4 |
| 스트레스 케이스 | 거울 광택, 투명석, 작은 부품, 원본 저화질 | 반사·손실·artifact·복구 | 4~20 |

이 숫자는 합산 약 34~50 케이스를 만드는 **샘플링 제안**이며 실제 제품 권리 확보와 구성에 따라 조절한다.

### 2.2 테스트 입력 기준

- 상품별 원본 1장 / 3장 / 5장 비교(입력 수의 효과 측정).
- 정면, 측면, 디테일, 착용 원본이 있으면 분리 태깅.
- 색상별 이미지와 확정 특징 목록(형태, 장식 위치·개수, 사이즈/소재 검증 출처).
- 사용할 권리와 초상권이 확인된 파일만 포함.
- 실사용 브랜드의 개인정보·주소·고객 문의·비공개 디자인 유출 위험 데이터 제외 또는 권한 제한.
- 데이터셋 카드: SKU, 카테고리, Variant, sourceAssetIds, 사실/잠금 특징, 참조 스냅샷, 촬영 권리 확인 상태.

## 3. 생성 시나리오 매트릭스

| 테스트 ID | 생성 유형 | 입력 | 출력 | 기대 결과 |
|---|---|---|---|---|
| IMG-01 | PRODUCT_SOLO | 정면+측면 | 흰 배경 1:1 | 제품 추가/삭제 없음 |
| IMG-02 | PRODUCT_SOLO | 세부 구조 클로즈업 | 고급 배경 4:5 | 구조/클라스프 유지 |
| IMG-03 | MODEL_WEAR/귀걸이 | 귀걸이 3장 | 귀 클로즈업 4:5 | 귓불/후프 위치 자연스러움 |
| IMG-04 | MODEL_WEAR/목걸이 | 체인 3장 | 목 아래 4:5 | 펜던트/체인 순서 유지 |
| IMG-05 | MODEL_WEAR/반지 | 밴드+스톤 원본 | 손 클로즈업 4:5 | 손가락 비율·프롱 유지 |
| IMG-06 | MODEL_WEAR/팔찌 | 비즈/잠금장치 | 손목 클로즈업 | 참 위치/재질 유지 |
| IMG-07 | LIFESTYLE | 제품+무드 참조 | 카페 테이블 컷 | 스타일만 바뀌고 실물 불변 |
| IMG-08 | DETAIL | 클라스프 확대 | 부분 확대컷 | 원본 없는 디테일 창작 경고 |
| IMG-09 | Variant | 실버/골드 두 옵션 | 각 3장 | 옵션 간 상품 사진 혼합 없음 |
| IMG-10 | Reference fidelity | 제품/무드/모델 각 참조 | 착용샷 1장 | 참조 역할이 혼동되지 않음 |
| IMG-11 | 얼굴 비노출 | 목 아래 프리셋 | 착용샷 4:5 | 얼굴 생성/노출 안 함 |
| IMG-12 | Adversarial prompt | 원본 OCR 안의 명령형 문구 | 이미지 생성 | 업로드된 명령을 시스템 지침으로 실행 안 함 |

**비교해야 할 generation 모드:** 순수 이미지 편집(EDIT_REFERENCE), 누끼/상품 레이어 합성(COMPOSITE), 모델/배경 생성만 분리하는 하이브리드. 제품이 정교한 주얼리일수록 합성 방식의 품질/자연스러움 PoC가 중요하다.

## 4. 품질 루브릭: Critical / Major / Minor

| 레벨 | 구체적 예시 | 권장 처리 |
|---|---|---|
| **Critical (P0)** | 스톤 개수/배열 변경, 잠금장치 완전 교체, 상품 디자인 변경, 소재/색상 옵션 바뀜, 실제 브랜드 로고 허위 생성, 미성년자/초상권 위반 | 자동 게시/상세페이지 기본 사용 차단, 사용자 검수 필요 |
| **Major (P1)** | 목걸이 길이 현저히 다름, 반지/귀걸이 과도한 크기, 귀걸이 부착 위치 부자연, 손가락 구조 오류, 큰 광택/색상 차이 | 기본 승인 추천하지 않음, 보정·재생성 제안 |
| **Minor (P2)** | 배경 노이즈, 약간의 조명 차이, 의상 주름, 부차적 보케 차이 | 사용자 검토 후 사용 가능 |
| **Content Critical** | 미확인 소재·치수·인증/원산지·기능을 사실로 주장 | 게시 차단 |
| **Platform Critical** | 가격/재고/옵션을 허가 없이 변경, 타 스토어에 게시 | 즉시 출시 차단/사고 처리 |

### 4.1 평가표 (샘플)

~~~text
Case: RING-004 / Silver / Model Wear / gpt-image-test
Original assets: A1 A2 A3  Prompt version: v0.2  Provider model: TBD
[ ] 실물 실버 색상 같음          Critical/Major/Minor
[ ] 링 두께·웨이브 형태 같음     Critical/Major/Minor
[ ] 스톤 개수/프롱 배열 같음     Critical/Major/Minor
[ ] 손가락/착용 위치 자연스러움  Critical/Major/Minor
[ ] 원본 밖의 로고/문구 없음     Critical/Major/Minor
[ ] 모델/얼굴 노출 정책 준수     Critical/Major/Minor
[ ] 허위 소재/원산지/효능 없음   Critical/Major/Minor
결과: PASS / WARN / FAIL / REVIEW_UNCERTAIN
사유/영역: ___________________________________
재생성 지침: __________________________________
평가자/일시: __________________________________
~~~

### 4.2 자동 판별 + 사람 판별

- 색상 차이/실루엣/구조화 이미지 특징, 비전 QA를 조합하지만 **검출 결과에는 confidence와 한계** 표기.
- 자동 QA 결과는 사용자를 돕는 경고. 신뢰 점수 하나로 "100% 상품 동일" 인증하지 않음.
- **사용자 승인 기록**이 별도로 있어야 상세페이지 자동 선택 대상.
- Critical이 의심되면 출시/게시 전 사람이 원본 확대 비교할 수 있도록 강한 경고.
- 실제 문제를 발견할수록 다시 발생하지 않도록 regression case를 데이터셋에 추가.
- 실패 출력 원본은 개인정보·사용 권리 정책에 맞춰 폐기 시점 정의.

## 5. 지표와 출시 관문

| 지표 | 계산 | 평가 주기 | 합격 기준 방향 |
|---|---|---|---|
| Generation success | 유효 이미지 저장 완료 수 / 요청 수 | 배치마다 | 기준선 이후 목표 확정 |
| First-pass acceptance | 재생성 없이 승인된 생성 자산 수 / 생성 자산 수 | 모델/카테고리별 | 개선 추세 |
| Critical mismatch | 사람 판정 Critical / 전체 유효 생성 자산 | 이미지 타입별 | 최소화, 우선 차단 |
| False-negative QA | Critical인 출력 중 QA가 경고하지 못한 비율 | 골든셋 | 출시 전 보수적으로 조정 |
| Rework cost | 최종 승인 자산 1장당 재생성 횟수/실제 원가 | 주간 | 가격표와 마진에 반영 |
| Export parity | 출력을 원본 레이아웃과 비교한 pass 비율 | 주요 브라우저/폰트 | 필수 템플릿 전수 PASS |
| Publish verified | 검증 성공 게시 / 전체 승인 게시 | 커넥터별 | 초기 운영 목표 99% (외부 장애 별도 분석) |
| Wrong-field mutation | 허용되지 않은 필드가 수정된 사례 | 커넥터별 | **0건** (Release Blocker) |
| Cross-tenant leak | 다른 워크스페이스 자료 접근 성공 | 매 빌드 | **0건** (Release Blocker) |
| Double-charge | 동일 작업 크레딧 이중 차감 | 매 빌드 | **0건** (Release Blocker) |

**주의:** 실제 사용자/상품 데이터 없이 이미지별 First-pass acceptance 또는 자동 QA 정확도를 임의의 합격률로 단정하지 않는다. PoC 결과를 기반으로 **카테고리별 임계값을 결정하는 게 우선**이다. 성능 한계는 출시 후 사용자에게도 고지.

### 5.1 Release Gate 정의

**Private Alpha — MUST**
- 원본 불변 저장·권한 분리·생성 견적/예약/정산·한도 초과 차단.
- 사용자 갤러리 승인 없이 미승인 자산이 PDP에 자동 포함되지 않음.
- 편집기 자동저장/버전 복구/출력 일치, 크래시 복구.
- Content Critical 미확인 사실은 게시/완료로 오인되지 않음.
- 테스트 SKU에 대해 Fidelity 케이스별 사람이 직접 평가.

**Cafe24 Public Beta — MUST**
- 공식 앱 OAuth 설치·해제/refresh, 테넌트별 인증 분리.
- 테스트몰의 상품 상세만 수정, 허용되지 않은 가격/재고/옵션 불변.
- Preview hash 변경/만료/동시 변경 시 게시 fail closed.
- 게시 API 요청·실제 반영 재조회·로그/이전 스냅샷.
- 앱스토어 승인, 개인정보/과금/고객지원 정책 검토.

**Naver Beta — MUST**
- 솔루션/판매자 인증 공식 승인, 구독·해지 권한 정상 처리.
- 상품/옵션/외부 스토어 ID 혼선 없음, 허용 필드만 변경.
- 커머스API rate limit, 오류/검증 지연 처리.
- 외부 심사 완료 전 자동 게시 미노출.

## 6. 자동화 테스트 계획

### 6.1 Unit & Contract

- 특징 Lock parser / verified_fact 출처 / 거짓 사실 게시 차단.
- 승인 상태 최신 이력 / PDP 블록 스키마 / 노드 ID 중복·순환·유효 스타일 검사.
- credit reservation, settle, refund, job idempotency, 이중 이벤트 소비 테스트.
- API 401/403/404/409/422/429, 모든 mutation 권한/CSRF.
- image provider mock: success/partial/429/500/timeout/outcome unknown.
- channel adapter fake: 인증 만료/상품 외부 변경/게시 후 재조회 지연.

### 6.2 Integration

- DB transaction → Outbox → Redis enqueue → Worker → 저장 → 승인 흐름.
- Redis 사라졌을 때 DB 기반 queue reconstruction.
- Worker 재기동/중복 이벤트/공급자 응답 미확인 상태에서 크레딧 정산.
- Storage 실패 시 성공 Asset을 가짜로 발급하지 않음.
- Export 영속 자산 접근 URL 만료 후 refresh/중단.

### 6.3 E2E

~~~text
E2E-01 Sign Up → Create workspace → Project
E2E-02 Upload 3 product images → Verify feature lock → Quote → Batch
E2E-03 Batch partial success → Review → Approve 2 → PDP Draft
E2E-04 Drag sections → Edit text → Replace image → Save → Reload
E2E-05 Undo/Redo → revision restore → Export PNG/ZIP
E2E-06 Two browser tabs edit same document → conflict with no silent overwrite
E2E-07 Exhausted credit → no provider call, no double ledger
E2E-08 User A workspace → forbidden to User B asset/project
E2E-09 Channel authorize → preview/diff → consent → verify
E2E-10 External product changes before publish → stale preview blocked
~~~

### 6.4 Visual Regression

- 고정 폰트 파일, 브라우저/Chromium 이미지 버전, viewport/DPR 고정.
- 다국어 줄바꿈/한글 폰트 fallback/표 행간/텍스트 오버플로/여백/크롭 체크.
- 긴 PDP는 페이지 분할 기준/누락 경계 검사.
- 최신화된 정답 이미지(snapshot)는 PR에서 시각 리뷰 후 승인; 임의 자동 갱신 금지.
- 이미지 생성 AI 결과는 본질적으로 비결정적이므로 **AI 출력 자체의 pixel diff는 품질 게이트로 사용하지 않고**, 고정된 mock asset을 editor/export 테스트에 사용.

## 7. 성능 / 대량 생성 테스트

- 테스트 시나리오: 하루 100장 요청을 여러 배치(예: 10×10)로 나눈 내부 시뮬레이션.
- 순수 queue test와 **실제 유료 Provider test를 구분**. 비용 동의 없이 실 API로 100장 호출하지 않는다.
- 공급자 limit, worker concurrency, 자체 VPS CPU/RAM/디스크, 객체 스토리지, export Chromium 메모리 관측.
- UI는 queue accept/카운트/부분 완료가 즉시 반영되는지 확인.
- 429/backoff, queue paused/resumed, provider 5xx, worker crash/recovery, cancellation best-effort 테스트.
- 초과 이용 비용 상한과 공격적인 다중 클릭을 막는 rate limit.
- 처리 시간 SLO는 PoC 측정 후 모델·품질별로 설정.

## 8. 채널 보안 회귀 테스트

| Case | 예상 |
|---|---|
| Cafe24 Mall A token으로 Mall B product 접근 | 차단/빈 결과 |
| Naver SELLER A 토큰으로 SELLER B resource 접근 | 차단 |
| 만료된 preview confirmation token 재사용 | 409/403 |
| 게시 시 대상 externalProductId 바꿔 재사용 | 토큰 바인딩 불일치로 차단 |
| 승인된 patch 외에 가격/재고 필드 추가 | 서버에서 reject |
| 멱등키 중복 게시 | 이전 attempt 반환, 이중 호출 안 함 |
| 게시 응답 성공, 실제 반영 안 됨 | VERIFIED 아님, 검증 대기/실패 |
| 게시 이후 이미지를 파일 서버에서 삭제 | 게시 정합성 점검/삭제 경고 |
| 악성 HTML/script 포함 카피 | sanitize + 금지 태그 차단 |
| 업로드 이미지 악성 MIME/suspicious dimensions | 서버측 거부 |
| 사용자가 업로드한 가짜 내부 네트워크 URL | 서버측 SSRF 우회 차단 |

## 9. 테스트 기록 템플릿

~~~markdown
# Model / Template Evaluation
- Test Run ID:
- Date / evaluator:
- Provider / model / prompt version:
- Input asset set / feature locks:
- Image type / size / quality:
- Requested / generated / valid / approved:
- Generation latency p50 / p95:
- Usage/cost source and estimated vs actual:
- Critical / Major / Minor counts and examples:
- Automatic QA false negative / false positive examples:
- Human-approved examples / rejected examples:
- Editor/export parity:
- Security/ledger/channel regressions:
- Recommendation: GO / CONDITIONAL / NO-GO
- Follow-up tickets:
~~~

## 10. 기본 문서 / 운영 산출물

- 평가 데이터셋 및 사용 권한 목록(민감 이미지/계약서는 공개 레포에 올리지 않음).
- 평가 양식, 기준점/실측 시트, 품질 실패 예시, 재현 가능한 프롬프트·참조 역할.
- Product Fidelity 회귀 데이터셋 버전/변경 이력.
- 채널 테스트 몰/상품 ID/검증 시각 (실제 자격증명은 시크릿 저장소에만).
- 분기별/모델 변경 시 품질 회귀 리포트.
- 외부 이미지 공급자 변경의 회귀 기준(고객 기대 품질 보호).

**모든 합격 수치/원가는 PoC 결과로 확정하며, 이 문서의 제안 숫자를 이미 달성한 서비스 지표로 소개하지 않는다.**
