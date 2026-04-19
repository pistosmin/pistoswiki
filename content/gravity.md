---
title: gravity
date: 2026-04-19
description: 모바일 퍼스트 커뮤니티 웹앱을 풀스택으로 짜보며 페이즈 분할 + 서브에이전트 라우팅 워크플로우를 실험한 학습·포트폴리오 프로젝트. Phase 1까지 동작 후 일시 중단.
tags:
  - fullstack
  - react
  - spring-boot
  - monorepo
  - jwt
  - docker
draft: false
---

# gravity

## 한 줄

모바일 퍼스트 커뮤니티 웹앱을 풀스택으로 직접 짜보는 학습·포트폴리오 프로젝트. 페이즈 분할 + 서브에이전트 라우팅 워크플로우를 굴려보는 게 진짜 동기였다. Phase 0~1 완료, Phase 2 착수 직전에서 멈춤.

## 컨텍스트·동기

2026-03-14에 빈 골격을 만들면서 시작했다. 표면 목표는 "사용자 게시글·댓글·추천이 있는 모바일 퍼스트 커뮤니티 웹앱"이지만, 실제 동기는 두 갈래였다.

1. **풀스택을 한 번 끝까지 짜본다** — 회사 일은 보통 한 레이어만 깊게 들어가게 되니까, 인프라·백엔드·프론트·배포까지 본인 손으로 묶어보는 경험이 필요했다.
2. **워크플로우를 실험한다** — ultraplan-ultrathink 패턴과 페이즈 분할 개발을 실제로 굴려보면서 어디서 막히고 어디서 빛나는지 보고 싶었다. Claude Code 서브에이전트 라우팅, Opus 에스컬레이션 규칙, gstack 같은 보조 스킬을 한 프로젝트에 묶어 실전 데이터를 모으는 자리.

도메인(커뮤니티)은 의도적으로 흔한 걸 골랐다. 도메인이 익숙해야 워크플로우 자체에 집중할 수 있다.

## 핵심 결정

### 1) pnpm 모노레포 + Docker Compose

`apps/web`(React) + `apps/api`(Spring Boot) + `infra/`(docker, monitoring, scripts)를 한 리포에 둔다. 대안으로 두 리포 분리도 있었지만, 학습용 단일 개발자 프로젝트에서 두 리포는 컨텍스트 스위칭 비용만 키운다. `make dev` 한 줄로 PostgreSQL·Redis·MinIO·Prometheus·Grafana·Loki·Nginx까지 일괄 기동. 모니터링 스택까지 미리 박아둔 건 "포트폴리오용으로 보여줄 때 그림이 잡힌다"는 판단.

### 2) JWT 인증 (Access 15분 + Refresh 7일)

세션 기반 대신 JWT를 골랐다. 이유는 SPA·모바일 클라이언트 친화 + Stateless 확장. AccessToken은 메모리/`localStorage`, RefreshToken은 DB 저장 + httpOnly 쿠키 또는 `localStorage`. 401 → `/auth/refresh` → 재시도 인터셉터 패턴. Spring Security 7의 `SecurityFilterChain` 빈 방식, jjwt 0.12.x 사용. 동시 토큰 갱신 경합은 Tech Advisor 에스컬레이션 항목으로 미리 적어둠.

### 3) 페이즈 8단계 분할 (Phase 0~7)

큰 그림을 한 번에 파지 않고 8개 페이즈로 자른다. Phase 0 부트스트랩 → 1 인증 → 2 게시글 → 3 댓글·추천 → 4 프로필·관리자 → 5 알림·미디어 → 6 완성 → 7 배포. 각 페이즈는 `prompt/phases/phase-N-*.md`에 plan + 진행 체크리스트 + 검증 기준이 들어 있다. 페이즈를 자른 기준은 "한 페이즈가 끝나면 무언가가 동작해야 한다"는 슬라이스 원칙.

### 4) 서브에이전트 라우팅 표 명시

Orchestrator(Opus) / Implementer(Sonnet) / Reviewer(Opus) / Tech Advisor(Opus) / Verifier·Tester·Explorer(Sonnet) — 7개 페르소나로 역할을 나누고 트리거 조건을 표로 못 박았다. 트랜잭션 전파, 동시성, 캐시 무효화 같은 시나리오는 Sonnet이 즉시 멈추고 Opus에 에스컬레이션. 과잉 에스컬레이션을 막는 self-check도 같이.

## 스택과 구조

### 프론트엔드

- **React 19.2 + TypeScript 5.7 + Vite 7.3** — 최신 안정 라인. Vite는 ESBuild + HMR로 dev 루프가 빠르다.
- **MUI 7.3** — 디자인 시스템 빠르게 올리려고. v7에서 `Grid2`가 `Grid`로 승격된 것을 반영. 토스 TDS 톤을 입히려고 `apps/web/src/app/theme/tokens.ts`에 디자인 토큰 분리.
- **Zustand 5 + TanStack Query 5.90** — 클라 상태와 서버 상태 분리. Redux 안 쓴 이유는 보일러플레이트 비용 대비 학습 가치가 낮다고 봤기 때문. TanStack Query는 캐시·재시도·역시리얼라이즈를 다 해주니까 `useEffect` fetching을 금지 사항에 박아둠.
- **React Router 7** — `react-router-dom`이 `react-router`로 통합된 거 한 번 발에 채여서 CLAUDE.md 금지 사항에 명시.
- **React Hook Form + Zod, Axios, Framer Motion** — 폼·HTTP·모션 표준 조합.

### 백엔드

- **Spring Boot 4.0.3 + Java 21 LTS** — Spring Boot 4.0이 `javax.*` 완전 폐기·`jakarta.*` 단일화한 첫 메이저. 한 번 옮겨두면 다음 5년 편하다는 베팅.
- **Spring Data JPA + QueryDSL** — JPA로 단순 CRUD, QueryDSL로 동적 쿼리. N+1은 `@ManyToOne(fetch = LAZY)` 강제 + 케이스별 fetch join/`@BatchSize`/`@EntityGraph` 선택.
- **PostgreSQL 18.3 + Flyway 11** — Postgres 18의 AIO·`uuidv7()` 내장이 매력. 마이그레이션은 Flyway V1~V7 미리 깔아둠.
- **Redis 7.4** — JWT 블랙리스트, 응답 캐시, Rate Limiting.
- **Gradle Kotlin DSL** — 빌드 스크립트 타입 안전.

### 인프라

- **Docker Compose** — 7개 서비스 한 번에. healthcheck로 `minio-init` 같은 의존 컨테이너 직렬화.
- **Nginx** — 리버스 프록시, 호스트 랩탑의 프론트·백엔드로 분배.
- **MinIO** — S3 호환 오브젝트 스토리지. 로컬에서 S3 연동 코드를 그대로 짤 수 있음.
- **Grafana + Prometheus + Loki** — Spring Actuator 메트릭 + 앱 로그.

### 디렉터리

```
gravity/
├── apps/
│   ├── web/  (Feature-Sliced Design: app/features/shared)
│   └── api/  (DDD-lite: global/domain)
├── infra/    (docker, monitoring, scripts)
├── prompt/   (architecture-plan-v2.md, phases/)
└── docs/superpowers/  (plans, specs)
```

## 일화·디테일

### 워크플로우가 진짜 소재

구현보다 더 신경 쓴 건 `CLAUDE.md`(프로젝트 루트, 22KB). 페이즈 표·서브에이전트 라우팅·Opus 에스컬레이션 트리거·자주 발생하는 에러 표·금지 사항·gstack 스킬 사용법·Git 전략까지 한 파일에 꾹꾹 눌러 담았다. 이 파일이 사실상 본 프로젝트의 핵심 산출물이다. 코드는 그 결과물.

### 구체적으로 박힌 규칙

- **한국어 주석 강제** — 클래스·메서드·주요 로직마다 "왜 이렇게 하는지" 한국어로. AI 어시스턴트가 다음 세션에서 컨텍스트 빨리 잡으라고.
- **`@Setter` 금지, 비즈니스 메서드로 상태 변경** — User에 `updateProfile`, `ban`, `unban`, `changeRole` 같은 도메인 의미 가진 메서드만.
- **DTO는 record** — Java 21이니까.
- **`any` 금지, `unknown` + 타입 가드** — TS strict.

### Phase 1 회원가입 UX 개선

3월 27일에 `BUGFIX.md`에 정리된 회원가입 UX 개선이 마지막에 가까운 작업이었다. 이메일 중복 시 단순 Alert만 띄우던 걸 `setError`로 인라인 필드 에러로 바꿈. `DUPLICATE_EMAIL`/`DUPLICATE_NICKNAME` 코드 분기. 로그인 실패는 보안상 어느 쪽이 틀렸는지 명시 안 함. 작은 변화지만 "JWT 끝났으니까 다음"으로 넘어가지 않고 UX까지 다듬은 흔적.

### 마지막 세션의 메타 작업

3월 28일 마지막 세션에서는 코드를 짠 게 아니라 Phase 0·1을 superpowers plan + spec 형식으로 다시 정리하는 메타 작업을 했다. "이미 구현됐지만 다시 점검하고 부족한 부분을 채울 수 있도록"이라고 직접 적어둠. `docs/superpowers/plans/`와 `docs/superpowers/specs/`에 재구성 산출물. 그게 마지막. 이후 커밋 0건.

### 14개 세션

3월 25일 첫 세션부터 3월 29일까지 5일간 14개 세션. 짧은 시간에 굵게 몰아친 패턴.

## 결과·현황

- **Phase 0 (부트스트랩)** — 완료. Docker 7개 서비스 healthy, `make web`/`make api` 정상.
- **Phase 1 (인증)** — 완료. 회원가입·로그인·JWT·토큰 갱신 인터셉터·라우터 가드·블러 시 실시간 중복 확인까지.
- **Phase 2 (게시글)** — 미착수. plan 파일은 있으나 코드 없음.
- **Phase 3~7** — 대기.
- **마지막 커밋**: 2026-03-28 21:17 `docs: add superpowers phase plans and specs`. 이후 약 3주간 신규 커밋 0건.
- **사실상 일시 중단**.

## 회고

### 잘된 것

- **워크플로우 문서가 실제로 두꺼워졌다** — 프로젝트 루트 CLAUDE.md는 다른 프로젝트로 이식 가능한 자산이 됐다. 서브에이전트 라우팅 표와 에스컬레이션 트리거 표는 다음 풀스택에서 그대로 재활용할 수 있다.
- **Phase 0~1까지의 슬라이스가 동작한다** — `make dev` → `make api` → `make web`으로 회원가입·로그인이 끝까지 돈다. 학습용으론 이미 한 사이클을 닫았다.
- **인프라를 넉넉히 깔아둔 것** — Grafana·Prometheus·Loki까지 미리 들어가 있어서 나중에 재개하면 관측 코드 추가가 즉시 가능.

### 못된 것

- **페이즈가 너무 많다** — 7단계는 학습 프로젝트치고 과욕. 동기가 빠지기 전에 "보이는 게시글 1개"까지 가야 했는데 Phase 1(인증)에서 회원가입 UX 다듬다가 Phase 2 진입을 못 했다. 인증부터 완벽하게 가는 패턴은 학습용에서는 함정.
- **메타 작업 비중이 컸다** — 마지막 세션이 코드가 아니라 plan/spec 재정리였다는 게 상징적. 워크플로우 자체가 더 재밌어지면 도메인 코드는 뒤로 밀린다.
- **모니터링 스택은 시기상조** — 기능이 없는데 메트릭부터 깔았다. YAGNI 위반.

### 다시 한다면

- Phase를 3개로 줄인다: (1) 인증 + 게시글 1개 보이기, (2) 댓글·추천, (3) 배포. 각 페이즈 끝에 무조건 데모 가능한 화면.
- 모니터링·관리자·알림은 (4) 이후로 미룬다.
- CLAUDE.md는 처음부터 두껍게 쓰지 말고, Phase 1 끝나고 한 번 회고하면서 추출한다. 프리미엄 첫 세션부터 22KB는 과적합 위험.
- 서브에이전트 라우팅은 유지. 트랜잭션 시나리오 Tech Advisor 에스컬레이션은 실전에서 효과를 봤다.
