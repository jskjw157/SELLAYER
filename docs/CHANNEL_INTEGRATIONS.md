# SELLAYER — 커머스 채널 연동 요구사항

> **문서 ID:** SL-INT-001 · v0.1 · 2026-10-15  
> **상태:** 연동 설계 기준 (채널 개발사/솔루션 승인 전)  
> **관계 문서:** [PRD](./PRD.md) · [출시 로드맵](./ROADMAP.md)

## 1. 원칙

**SELLAYER는 독립 SaaS가 본체, 커머스 채널은 어댑터**다. 초기 고객이 카페24 앱스토어를 통해 들어와도 데이터의 원본은 SELLAYER 프로젝트이며, 같은 콘텐츠를 다른 채널용으로 다시 출력할 수 있어야 한다.

- 채널 연결 전에도 **생성 → 편집 → 다운로드** 가능.
- **기존 상품의 상세 정보/이미지 수정**을 최초 목표로 설정; 신규 상품 등록 및 재고·판매가 관리는 후순위.
- 업데이트 전에 **채널/스토어/상품 식별 + 변경 필드 + 미리보기 + 승인** 필수.
- 업데이트 후 대상 플랫폼의 데이터를 **재조회**하여 반영 여부를 검증한다.
- 원본 PDP JSON 문서는 유지하고, 각 채널에 업로드되는 것은 별도의 **채널 렌더링 결과물**이다.
- 연동 규격과 심사 요구사항은 2026-10-15 기준 확인했으며 실제 출시 직전 재검증해야 한다.

## 2. 공통 채널 커넥터 계약 (개념)

| 기능 | 동작 | 실패 시 |
|---|---|---|
| `connect(account)` | 정식 판매자 위임/솔루션 활성화 흐름 | 연결 실패 상태를 남기고 작업 중단 |
| `getProducts(cursor, filters)` | 승인된 판매자 상품 목록/검색 | 재로그인 또는 재시도 유도 |
| `getProduct(externalId)` | 단일 기존 상품 필드·버전 조회 | 존재/권한/상태 변경 알림 |
| `validate(document, target)` | 필수 규격·제약 및 HTML/이미지 검증 | 게시 전 차단, 수정 가이드 |
| `render(document, target)` | 채널별 설명/이미지 산출물 생성 | 원본 문서 불변 |
| `previewDiff(externalId, rendered)` | 현재 상품과 변경점 비교 | 비교 불가 시 쓰기 허용 금지 |
| `publish(externalId, patch, idempotencyKey)` | **승인된 필드만** 패치 | 에러/부분 실패 기록 |
| `verify(externalId)` | 수정 내용 재조회·대조 | 반영 미확인/지연 상태 |
| `disconnect(account)` | 토큰 폐기/연동 해제/사용자 안내 | 토큰 삭제 재시도, 관리 경고 |

### 공통 상태

`DISCONNECTED → CONNECTING → CONNECTED → DEGRADED | EXPIRED → DISCONNECTED`

게시: `DRAFT → READY_TO_REVIEW → APPROVED → PUBLISHING → VERIFIED | FAILED | PARTIALLY_VERIFIED`

`PUBLISHING` 상태는 외부 API 응답 성공만으로 `VERIFIED` 처리하면 안 된다.

## 3. Cafe24 (M1)

### 3.1 지원 목표

- 카페24 쇼핑몰 관리자/앱스토어에서 설치 및 권한 위임.
- 상품 검색/가져오기, 원본 콘텐츠 미리보기.
- 기존 상품의 **상세설명 및 사용자 선택 이미지** 업데이트.
- 별도 동의 없이 **가격·재고·옵션·배송/판매 상태** 변경하지 않음.

### 3.2 인증과 권한

- Cafe24 공식 **OAuth 2.0 Authorization Code** 플로우 사용.
- `mall_id`, `shop_no`, 외부 상품 번호와 연결된 계정 저장.
- 필요한 `mall.read_product` / `mall.write_product` 등 최소 scope만 요청 (실제 API별 scope 개발 중 확인).
- 토큰/리프레시 로테이션, 기한 만료, 연결 해제, 권한 변경 처리.
- `redirect_uri`는 승인된 HTTPS 도메인만 허용하며 OAuth state 검증.

### 3.3 구현·검증 요구

- 상세 설명/이미지/상품 속성의 **실제 수정 가능한 필드**를 확인하고 필드 화이트리스트 적용.
- 멀티쇼핑몰(shop_no), 다국어/PC·모바일 상세설명, 이미지 CDN/URL 수명 정책 검증.
- 앱 설치/해제 이벤트와 구독 상태, 심사 기준/수수료/결제 정책은 **판매 채널 출시 전** 최신 문서로 검증.
- 샌드박스 또는 테스트 쇼핑몰에서 ① 업로드 ② 상품 조회 ③ 변경 미리보기 ④ 게시 ⑤ 재조회 ⑥ 연결 해제 테스트.

### 3.4 참고 문서

- [Cafe24 Admin Products API](https://developers.cafe24.com/docs/ko/api/admin/)
- [Cafe24 OAuth 흐름](https://developers.cafe24.com/app/front/app/develop/oauth/process)
- [Cafe24 토큰](https://developers.cafe24.com/app/front/app/develop/oauth/token)
- [Cafe24 필수 정보 및 HTTPS 리디렉션](https://developers.cafe24.com/app/front/app/develop/oauth/put)

## 4. Naver 스마트스토어 (M2)

### 4.1 지원 목표

- **네이버 커머스솔루션마켓에 공식 솔루션으로 입점**하는 경로를 우선 검토.
- 판매자 구독/동의, 솔루션 승인 및 판매자 권한 범위 내에서만 사용.
- 기존 상품 조회, 상세 설명/상품 이미지 필드 갱신, 게시 결과 검증.

### 4.2 인증과 솔루션 제약

- 네이버 커머스API는 기본적으로 **OAuth2 Client Credentials** 방식.
- 솔루션 통합은 `SELF`(솔루션사 자체 작업)와 `SELLER`(구독 중인 판매자 자원) 토큰을 구분한다.
- `SELLER` 인증은 승인된 솔루션 권한 및 판매자 UID/account_id에 종속. 카페24 OAuth와 같은 구조라고 가정 금지.
- 커머스솔루션마켓의 구독, 사용 승인/해지, 심사/정산 체계를 조사하고 구현.
- 각 판매자의 **개인용 앱 Client ID/Secret을 수집해 SaaS에서 사용하도록 요구하지 않는다.**

### 4.3 구현·검증 요구

- 원상품/채널상품의 구분 및 동일 상품 ID 매핑을 올바르게 처리.
- `detailContent` 및 이미지 업로드 형식/URL/태그 제약, 카테고리별 필드 의존성 확인.
- **기존 상품 일부 콘텐츠 수정**만 MVP 출시 범위; 신규 등록은 카테고리/배송/과금/정책 필수 필드 요구가 커 별도 계획으로 관리.
- 심사 승인 전에는 고객에게 네이버 자동 게시를 판매하거나 제공 가능하다고 표시하지 않음.
- 네이버 솔루션 등록/테스트/심사 결과를 출시 게이트로 정의.

### 4.4 참고 문서

- [커머스솔루션마켓 소개](https://apicenter.commerce.naver.com/docs/solution-doc/1000/%EC%BB%A4%EB%A8%B8%EC%8A%A4%EC%86%94%EB%A3%A8%EC%85%98%EB%A7%88%EC%BC%93-%EC%86%8C%EA%B0%9C)
- [기본 연동 요소: SELF/SELLER](https://apicenter.commerce.naver.com/docs/solution-doc/3000/%EA%B8%B0%EB%B3%B8-%EC%97%B0%EB%8F%99-%EC%9A%94%EC%86%8C-%EA%B0%80%EC%9D%B4%EB%93%9C)
- [Naver 인증](https://apicenter.commerce.naver.com/docs/auth)
- [Naver 상품 API](https://apicenter.commerce.naver.com/docs/commerce-api/current/%EC%83%81%ED%92%88)
- [Naver 판매자 커머스 아이디 인증](https://apicenter.commerce.naver.com/docs/solution-doc/3000/%ED%8C%90%EB%A7%A4%EC%9E%90-%EC%BB%A4%EB%A8%B8%EC%8A%A4-%EC%95%84%EC%9D%B4%EB%94%94-%EC%9D%B8%EC%A6%9D)

## 5. 채널 게시 안전 장치

**반드시 적용**

1. **Read-before-write:** 대상 상품의 현재 상세설명/이미지/수정 버전을 조회하여 스냅샷 보관.
2. **Field allowlist:** `detailContent` 등 명시적으로 허용된 필드만 전송. 채널별 업데이트 API가 전체 리소스를 요구하면 무관한 필드를 최신 조회 값으로 보존하며 회귀 테스트.
3. **Human confirmation:** 서비스에서 변경점 미리보기, 외부 상품 링크, 변경 설명, 비용/규정 경고를 제시.
4. **Idempotency:** 고객 재클릭/네트워크 재시도에도 중복 게시·이중 과금 방지.
5. **Re-read and compare:** 작업 응답 성공과 **실제 적용 성공**을 구분.
6. **Audit:** 누가 어떤 문서 revision을 어떤 스토어/상품에 어떤 필드로 반영했는지 로그.
7. **Rollback limits:** 스냅샷은 남기되 채널 제약상 **자동 완전 복원 보장 금지**; 지원 가능한 범위·실패 이유 표시.
8. **Auth isolation:** 업체 A의 연결 토큰으로 업체 B 상품 접근 불가.
9. **Secure HTML:** 스크립트/이벤트 핸들러 제거, 이미지 호스팅 URL 접근성 확인, 채널별 허용 HTML 검증.
10. **Consumer accuracy:** 오인성 실물 착용 주장, 미확인 인증/효능/원산지 등 자동 게시 금지.

## 6. 테스트 매트릭스

| 시나리오 | 독립 SaaS | Cafe24 | Naver |
|---|---|---|---|
| 원본 상품 이미지 → 생성 → PDP 편집 | M0 필수 | 동일 문서 재사용 | 동일 문서 재사용 |
| ZIP/PNG/긴 이미지 Export | M0 필수 | 대체 경로 | 대체 경로 |
| 계정 연결·해제 | 해당 없음 | M1 필수 | M2 필수/승인 후 |
| 기존 상품 가져오기 | 해당 없음 | M1 필수 | M2 필수 |
| 기존 상품 상세/선택 이미지 반영 | 해당 없음 | M1 필수 | M2 필수 |
| 새 상품 등록 | 대상 아님 | M3 별도 | M3 별도 |
| 가격·재고·주문 관리 | 대상 아님 | 지원하지 않음 | 지원하지 않음 |
| 게시본 검증·실패 처리 | 해당 없음 | M1 필수 | M2 필수 |

## 7. Release Gate

- **Cafe24:** 앱 등록/설치·권한 승인 완료, 데모몰 read/write 테스트, 권한 철회 및 오류 복구 검증, 비용·정산 정책 확인.
- **Naver:** 솔루션마켓 요건·사업자/보안/심사 검토, `SELLER` 데이터 접근 승인, 실제 테스트 판매자 상품 반영 확인.
- **공통:** 관련 채널 승인이 없거나 정책이 바뀌면 **파일 내보내기 기능은 유지**하되 자동 게시 메뉴는 비활성화/베타 대기 상태로 안내.

## 8. 오픈 질문

- Cafe24 상품 이미지 추가/교체와 상세 설명 업데이트를 **동일 원자적 작업**처럼 UX에 제공할 수 있는가?
- 플랫폼에서 외부 호스팅 이미지를 허용하는 조건, 파일 용량/HTML 제약은 무엇인가?
- Naver 솔루션 등록·심사 일정 및 구독/결제 모델이 자체 SaaS 과금과 어떤 충돌을 만드는가?
- 기존 상품이 SELLAYER 밖에서 바뀌었을 때 충돌 탐지·선택 병합/덮어쓰기 UX는 무엇인가?
- 플랫폼별 이미지 컨텐츠 정책에 맞춰 AI 생성 이미지 사실성/고지/워터마크를 어떻게 처리해야 하는가?
