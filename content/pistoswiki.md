---
title: pistoswiki
date: 2026-04-19
description: Karpathy의 LLM Wiki 개념 + Graphify v4를 옵시디언 vault에 이식한 개인 지식 시스템. Claude에서 Codex로 작업 지침을 확장한 현재의 메타 프로젝트.
tags:
  - meta
  - wiki
  - graphify
  - quartz
draft: false
---

# pistoswiki

## 목적

Andrej Karpathy의 [LLM Wiki 개념](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)과 Graphify v4 지식 그래프를, 옵시디언 vault 위에서 하나로 묶은 개인 지식 시스템. 장기 축적과 AI 위임 둘 다 가능한 구조를 목표로 한다.

## 현재 단계

MVP 완료, 첫 콘텐츠 시드 중. Phase 0~7 — vault 스키마, 슬래시 커맨드, Graphify 통합, 더미 flow, Quartz 4 site repo, GitHub Pages 배포, `/publish` 검증 — 이 다 끝났고, Phase 8에서 이 프로젝트 자체의 설계·결정·엔티티를 위키 첫 콘텐츠로 채우는 중이다(dogfooding).

2026-05-05에는 Claude Code 중심이던 운영 지침을 Codex에서도 읽히도록 `AGENTS.md`로 이관했다. `CLAUDE.md`는 원본 기록으로 보존하고, Codex 세션에서는 `AGENTS.md`와 현재 사용 가능한 Codex 스킬·플러그인을 우선한다.

## 주요 결정

- **Quartz 4 + GitHub Pages 채택** (Next.js 대신). 정적 사이트 생성에 Obsidian vault 호환성이 좋고, Preact 컴포넌트 커스텀 여지도 확보.
- **파일명 ISO8601 timestamp prefix** (`YYYYMMDDTHHMMSS-<slug>.md`). 제목 변경에 흔들리지 않는 안정 ID.
- **플랫 + frontmatter 구조**. 폴더 위계 최소화, AI 위임 친화적.
- **LLM = 큐레이터·링커, 작성자 아님**. 생성을 맡기면 환각이 그래프 전체로 번진다는 경험 기반 룰.
- **에이전트 런타임 분리**. Claude 전용 규칙은 `CLAUDE.md`에 보존하고, Codex용 실행 규칙은 `AGENTS.md`로 옮겨 같은 위키 철학을 다른 도구에서도 적용한다.
- **듀얼 링크 시맨틱**(EXTRACTED/INFERRED/AMBIGUOUS). Graphify가 추출하는 엣지에 추적 가능한 의미를 붙여 신뢰도 유지.
- **한국어 기본**. 번역은 수동 `<id>.en.md`로 나중에.
- **하이브리드 Ingest** — 대화형 + Daily Queue 두 갈래.
- **publishable 기본 false, 선별 발행**. 공개 가능 판단은 사람이 명시적으로 true로 바꿔야 사이트에 올라간다.
- **저장소 이중 구조** — vault(원천, Google Drive) / site repo(파생, GitHub).

## 스택

- Obsidian — 원천 vault
- Graphify v4 — 지식 그래프 엔진
- Quartz 4 — 정적 사이트 생성
- GitHub Pages — 배포
- Claude Code — 초기 큐레이션·정리·발행 자동화
- Codex — `AGENTS.md` 기반 현재 작업 세션과 마이그레이션 이후 운영

## 로드맵

- [x] MVP (Phase 0~7): vault 스키마 + Graphify + Quartz 4 + GH Pages
- [x] Phase 5 GitHub Pages 배포 성공 (2026-04-19)
- [x] Phase 6 `/publish` flow 검증 (여는 글)
- [x] Phase 8: 프로젝트 회고 콘텐츠 시드
- [x] Codex 이관 노트 발행 (2026-05-05)
- [x] 사내 레거시 시스템 LLM 위키 워크벤치 공개 요약 발행 (2026-05-05)
- [ ] Graphify 첫 semantic 빌드 (콘텐츠 쌓인 뒤 Claude subagent로)
- [ ] 주간 `/lint` 루프 정착
- [ ] 선별 `/publish` (첫 90일 목표 5~10개)

## 미래 아이디어

- 모바일 덤프 → Telegram bot → MCP
- 커스텀 Preact 컴포넌트 (1년 후 재검토)
- 영어 번역 파이프라인 자동화
- MCP 서버 상시 운용
- `decision` / `area` 노트 타입 추가
