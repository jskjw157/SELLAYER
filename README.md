# SELLAYER

> **AI Commerce Creative Studio** — 상품 원본 이미지 한 장에서 모델 착용샷·상품 단독컷·편집 가능한 상세페이지까지. 카페24와 스마트스토어로 확장하는 독립 SaaS.

**Status:** PRD / 제품 요구사항 리뷰 단계 (코드 구현 및 배포 전)

## 제품 개요

SELLAYER는 한국 커머스 판매자가 상품 원본을 업로드하고 AI 상품 이미지 제작, 결과 검수, 상세페이지 블록 편집, 채널별 내보내기/게시를 하나의 워크플로우에서 수행하도록 설계한 서비스입니다.

- **원본 이미지 → AI 비주얼:** 모델 착용컷 / 단독컷 / 디테일컷 / 라이프스타일 컷.
- **제품 동일성 우선:** 잠금 특징 설정, 원본과 생성본 비교, 승인·재생성.
- **진짜 편집기:** Figma-like 3패널 UI, 텍스트/이미지/섹션/레이어 수정, 자동 저장/버전 관리.
- **채널 독립:** 공통 PDP 문서 원본, 카페24 및 네이버용 별도 커넥터.
- **판매자 보호:** 사전 비용/크레딧 확인, 외부 상품 변경 전 미리보기와 명시적 승인.

## 문서

| 문서 | 내용 | 상태 |
|---|---|---|
| [제품 요구사항 정의서 (PRD v0.1)](docs/PRD.md) | 목표, 페르소나, 사용자 여정, 기능 요구사항, UX, 인수 기준, 리스크 | 리뷰 |
| [출시 로드맵](docs/ROADMAP.md) | PoC → M0 독립 SaaS → M1 Cafe24 → M2 Naver → M3 Scale | 리뷰 |
| [커머스 채널 연동 요구사항](docs/CHANNEL_INTEGRATIONS.md) | Cafe24 OAuth, Naver 솔루션 인증, 게시 안전장치와 검증 | 리뷰 |

## 출시 방향

1. **Discovery / PoC:** 제품 원형 보존 성능, 편집기 렌더링, AI 원가 검증
2. **Private Alpha (M0):** 생성/검수/상세페이지 편집/다운로드
3. **Public Beta (M1):** 카페24 앱 연결 및 기존 상품 상세페이지 반영
4. **Channel Expansion (M2):** 공식 승인 후 네이버 스마트스토어 반영
5. **Scale (M3):** 대량 생성, 브랜드/모델 일관성, 멀티채널 확대

## 운영/배포 목표 (미구현)

- Hosting: **Hostinger VPS, Ubuntu 24.04**
- Planned delivery: **GitHub Actions CI/CD → Docker Compose → HTTPS**
- Planned service components: Next.js web/API, generation worker/queue, PostgreSQL, Redis, object storage
- AI: 서버 측 유료 생성 API/공급자 추상화. **ChatGPT Pro 구독으로 상용 고객에게 무제한 생성 제공을 가정하지 않습니다.**

> 현재 저장소는 **기획 문서 단계**입니다. 앱 실행 코드, 배포 파이프라인, 외부 채널 자동 게시 기능은 아직 존재하지 않습니다.

## 제품 원칙

- 원본은 덮어쓰지 않고 결과물은 리뷰 후 사용.
- 상품 설명의 재질·크기·인증·원산지 등 사실을 AI가 임의 생성하지 않음.
- 외부 채널 인증·승인 없이 상품 게시 기능 제공을 보장하지 않음.
- 변경 사항은 PRD의 `FR-*`, `NFR-*`, `AC-*` 코드에 연결해 개발.

Copyright © SELLAYER. All rights reserved unless otherwise noted.
