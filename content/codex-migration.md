---
title: Claude에서 Codex로 옮긴 개인 에이전트 작업 환경
date: 2026-05-05
description: Claude Code 중심으로 굴리던 개인 프로젝트 지침과 스킬 운용을 Codex의 AGENTS.md, 플러그인, 스킬 구조로 옮긴 기록.
tags:
  - codex
  - claude-code
  - migration
  - agents
  - workflow
draft: false
---

# Claude에서 Codex로 옮긴 개인 에이전트 작업 환경

## 한 줄

Claude Code 중심으로 굴리던 개인 프로젝트 지침과 스킬 운용을 Codex에서도 읽히도록 `AGENTS.md`와 Codex 스킬·플러그인 구조로 옮겼다. 기존 `CLAUDE.md`는 원본 기록으로 보존하고, Codex 세션에서는 `AGENTS.md`가 우선 진입점이 된다.

## 왜 옮겼나

pistoswiki와 여러 개발 프로젝트는 원래 Claude Code의 `CLAUDE.md`, 슬래시 커맨드, 스킬, 플러그인 전제를 중심으로 정리되어 있었다. 그런데 작업 런타임이 Codex로도 확장되면서 같은 운영 의도를 새 에이전트가 놓치지 않게 해야 했다.

중요한 건 단순 파일명 교체가 아니었다. Claude 전용 slash command나 Claude Agent tool 호출은 그대로 실행할 수 없으니, 의도만 보존하고 Codex의 내장 도구·플러그인·스킬에 맞게 해석하는 층이 필요했다.

## 결정

### `CLAUDE.md`는 보존, `AGENTS.md`를 추가

기존 Claude 작업 메모리를 지우지 않았다. 대신 Codex가 읽을 수 있는 `AGENTS.md`를 새로 두고, "원본 `CLAUDE.md`는 보존한다"는 규칙을 명시했다.

이렇게 하면 Claude로 돌아가도 이전 문맥이 살아 있고, Codex에서는 같은 운영 철학을 자기 도구 체계로 적용할 수 있다.

### Codex에서는 현재 사용 가능한 기능을 우선

Claude 플러그인 전체를 무조건 흉내내지 않는다. Codex에 이미 있는 기능, 설치된 플러그인, 현재 세션의 MCP 도구, Codex 스킬을 먼저 사용한다.

예를 들어 브라우저 작업은 Codex의 in-app browser나 Playwright 플러그인을 쓰고, 문서·스프레드시트·프레젠테이션은 Codex에 연결된 전용 스킬을 우선한다. Claude 전용 명령은 의도를 읽어 현재 세션 기능으로 번역한다.

### LLM 위키에서의 역할은 그대로

pistoswiki에서 에이전트의 역할은 바뀌지 않았다. Claude든 Codex든 여기서는 작성자가 아니라 큐레이터·링커·정리자다.

새 사실을 꾸며내지 않고, 원천 자료를 읽고, 공개 가능한 것만 선별하고, 기존 노트와 연결한다. Daily·Journal·Inbox는 읽기 전용, Sources는 write-once, log는 append-only라는 보호 규칙도 그대로 유지한다.

## 바뀐 점

- 개인 전역 운영 의도는 Codex용 `AGENTS.md`로 옮겼다.
- pistoswiki vault에도 Codex 우선 지침을 담은 `AGENTS.md`가 추가됐다.
- 프로젝트별 `CLAUDE.md`는 참고 자료로 남기고, 실제 현재 세션 지침은 각 저장소의 `AGENTS.md`를 우선한다.
- Claude 전용 slash command와 Agent tool 호출은 그대로 실행하지 않고, Codex 도구와 현재 세션 기능에 맞게 해석한다.
- Claude에서 쓰던 `superpowers`, `compound-engineering`, `ralph` 계열은 Codex에 연결된 스킬이 있을 때 선별적으로 쓴다.

## pistoswiki에 남는 의미

이전의 pistoswiki는 Claude Code가 주 실행자였다. 이제는 특정 에이전트 제품 하나에 묶인 위키가 아니라, `AGENTS.md`를 통해 여러 에이전트 런타임이 같은 작업 규율을 공유하는 쪽으로 이동했다.

이 변화는 작지만 중요하다. LLM 위키의 핵심은 "어떤 모델이 쓰느냐"보다 "사람이 만든 원천과 판단을 어떻게 누적·연결하느냐"에 있기 때문이다.

## 관련

- [pistoswiki](pistoswiki)
- [Claude Code effort 레벨과 thinking 토큰](claude-code-effort-levels)
- [ralph-internal-setup](ralph-internal-setup)
