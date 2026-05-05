---
title: 사내 레거시 시스템 LLM 위키 워크벤치
date: 2026-05-05
description: 레거시 업무 시스템을 LLM 위키로 점진 문서화하기 위해 집에서 템플릿·프롬프트·샌드박스를 먼저 만든 워크벤치 기록.
tags:
  - llm-wiki
  - legacy-system
  - ralph
  - obsidian
  - graphify
draft: false
---

# 사내 레거시 시스템 LLM 위키 워크벤치

## 한 줄

레거시 업무 시스템을 회사 환경에서 LLM 위키로 자동 문서화하기 위한 집 워크벤치. 실데이터를 가져오지 않고, 템플릿·프롬프트·스킬 명세·샌드박스를 먼저 깎아 둔 뒤 회사 환경으로 단방향 반입하는 구조다.

## 배경

레거시 시스템은 한 화면의 동작을 이해하려 해도 화면, 자바 클래스, DAO, 쿼리, 프로시저, 함수, 테이블이 서로 얽힌다. 한 번에 전체를 리팩터링하기보다, LLM이 한 객체씩 분석해 노트로 남기고 링크를 쌓아가면 시간이 지날수록 시스템 지도가 만들어진다.

이 접근을 회사 환경에서 바로 실험하기에는 보안·망 분리·실데이터 문제가 있다. 그래서 집에서는 가짜 샌드박스와 반입용 vault 템플릿만 만들고, 실제 회사 식별자나 데이터는 절대 들여오지 않는 방향으로 워크벤치를 구성했다.

## 구조

- `llm-wiki/` — 회사 반입용 Obsidian vault 템플릿
- `_system/schema.md` — 노트 타입과 필드 규칙 요약
- `_system/queue.md` — Todo / Done / Blocked 단일 큐
- `_system/prompts/ralph-cycle.md` — 한 사이클 분석 프롬프트
- `_system/templates/` — procedure, function, package, table, view, query, facade, session, dao, screen 템플릿
- `skills/sync-and-queue/` — 실제 회사 환경에서 구현할 동기화 스킬 명세
- `sandbox/` — 집에서 검증하기 위한 가짜 소스와 가짜 DB 객체

## 핵심 결정

### 집에서 회사로만 단방향

가장 중요한 규칙은 역방향 금지다. 집 워크벤치에서 만든 템플릿과 프롬프트는 회사로 가져갈 수 있지만, 회사 소스·실 DB 객체·실 쿼리·내부 식별자는 집으로 가져오지 않는다.

### 한 사이클, 한 노트

LLM이 한 번에 여러 객체를 요약하면 링크 품질이 금방 흐려진다. 그래서 한 사이클은 하나의 객체를 분석하고, 하나의 노트를 만들고, 기존 노트와의 forward link를 남기는 흐름으로 제한했다.

### `source_hash`와 `last_analyzed`

노트가 언제 어떤 원천을 보고 만들어졌는지 추적해야 한다. 원천이 바뀌면 같은 노트를 다시 볼 수 있도록 `source_hash`와 `last_analyzed`를 핵심 필드로 두었다.

### Graphify 우선, grep 폴백

연관성 탐색은 Graphify를 기본으로 두되, 회사 환경에서 설치나 권한이 막힐 수 있으므로 `rg` 기반 폴백을 함께 둔다. 위키가 도구 하나에 막혀 멈추지 않게 하기 위한 결정이다.

## 검증 상태

집 샌드박스에서는 procedure, function, table, query, facade, session, dao, screen 등 10개 타입 템플릿과 샘플을 만들고, 가짜 데이터 기준으로 수동 시뮬레이션과 서브에이전트 검증을 진행했다. 핵심 원칙은 한 사이클 한 노트, 링크 일관성, `source_hash`·`last_analyzed` 기록, 프롬프트 인젝션 방어다.

## 남은 일

회사 환경에서는 첫 사이클 전에 파일시스템 권한, Graphify 실행 가능 여부, 실제 pull 경로와 프롬프트 경로 테이블 일치를 확인해야 한다. 이후 첫 주에는 자바 타입 판별 규칙, 화면 규약, 팀 vault 공유 방식을 확정하고, 첫 달에는 동기화 스킬과 링크 점검 루프를 붙이는 것이 다음 단계다.

## 관련

- [pistoswiki](pistoswiki)
- [ralph-internal-setup](ralph-internal-setup)
- [Claude에서 Codex로 옮긴 개인 에이전트 작업 환경](codex-migration)
