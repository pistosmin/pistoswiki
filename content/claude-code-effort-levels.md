---
title: Claude Code effort 레벨과 thinking 토큰
date: 2026-04-19
description: Claude Code의 effort 레벨(low/medium/high/xhigh/max)이 실제로 thinking 토큰을 얼마나 쓰는지. 고정 버짓이 아니라 '노력 신호'다.
tags:
  - claude-code
  - effort
  - thinking
  - tokens
draft: false
---

# Claude Code effort 레벨과 thinking 토큰

Claude Code는 모델이 생각(thinking)에 얼마나 시간을 쓸지 고르게 해준다. `low` · `medium` · `high` · `xhigh` · `max` 다섯 단계다. 이름만 보면 `xhigh`가 `high`의 몇 배쯤 토큰을 쓸 것 같지만, Anthropic 공식 문서에는 배수가 명시돼 있지 않다.

## 고정 버짓이 아니라 "노력 신호"

effort 파라미터는 고정된 thinking 토큰 버짓이 아니다. 모델이 문제 난이도에 따라 thinking 길이를 동적으로 조절하도록 주는 힌트다. 그래서 "xhigh = high의 N배" 같은 스펙 보장이 없다. 같은 `xhigh` 설정이라도 쉬운 질문엔 짧게, 복잡한 agentic 작업엔 길게 생각한다.

## 레벨의 위치

- `high` — 기본값. 복잡한 추론에 최적화돼 있고 대부분 작업에 충분하다.
- `xhigh` — Opus 4.7 전용. 30분 이상 걸리는 긴 코딩·agentic 작업용으로 튜닝된 레벨이다.
- `max` — 상한이 없다. 모델 능력의 꼭대기를 쓰는 대신 토큰을 과다 소비할 위험이 있다.

체감상 `xhigh`·`max`에서 thinking 블록이 훨씬 길어지는 경향은 있다. 수 배까지도 벌어질 수 있다. 다만 이건 경험적 관찰이지 스펙으로 보장된 값은 아니다.

## 실용 팁

`xhigh`나 `max`를 쓸 땐 `max_tokens`을 64k 이상으로 올리는 편이 좋다. Anthropic 공식 권장이기도 하고, thinking이 중간에 잘려 낭비되는 걸 막아준다.

간단한 질문·리팩토링엔 `medium`/`high`로 내려두면 토큰을 꽤 아낀다. 세션 중에도 `/effort`로 즉시 바꿀 수 있으니 작업 성격에 따라 옮겨 다니면 된다.

## 참고

- [Claude Code — Model Configuration](https://code.claude.com/docs/en/model-config)
- [Anthropic — Effort Parameter Guide](https://platform.claude.com/docs/en/build-with-claude/effort)
