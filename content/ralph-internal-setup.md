---
title: ralph-internal-setup — 사내 PC 자율 루프 개발 환경
date: 2026-04-19
description: 외부망 차단된 사내 PC에서 Ralph 패턴을 굴리기 위해 오프라인 아티팩트(자체 스킬 + 공식 플러그인 미러)를 USB 반입 가능하게 패키징한 인프라.
tags:
  - ralph
  - claude-code
  - internal-network
  - autonomous-loop
draft: false
---

# ralph-internal-setup — 사내 PC 자율 루프 개발 환경

## 한 줄

외부망이 막힌 사내 PC(Sonnet 4.6 1M)에서 Geoffrey Huntley의 랄프 패턴을 굴리기 위해, 두 갈래(자체 스킬 + 공식 플러그인 미러) 오프라인 아티팩트를 만들어 USB 반입할 수 있게 패키징한 인프라.

## 컨텍스트 / 동기

회사 PC는 다음 제약을 깐다.

- 외부 인터넷 차단. 런타임 `WebFetch`, `WebSearch`, context7 모두 사용 불가.
- npm은 내부 미러만. 공식 레지스트리·플러그인 마켓플레이스 막힘.
- 셸은 bash만. PowerShell 차단.
- Claude Code는 Sonnet 4.6만 프로비저닝됨. Opus 없음.

이 환경에서도 랄프 패턴 — 같은 `PROMPT.md`를 새 컨텍스트의 Claude Code 세션에 반복 투입해 목표가 수렴할 때까지 돌리는 unattended iteration loop — 을 굴리고 싶었다. 야간 무인 실행으로 리팩터·테스트 백필·스펙 구현 같은 기계적 sweep을 처리할 수 있으면, 사내 개발 생산성에서 가장 큰 비용인 "사람 시간"을 회수한다는 가정.

기존 옵션은 둘 다 어딘가가 안 맞았다.

- **Anthropic 공식 `ralph-wiggum` 플러그인**: 마켓플레이스 설치 전제 — 사내망에서 깔 수 없음. 그리고 Stop hook으로 *같은 세션* 안에서 prompt 재투입을 한다. 컨텍스트가 누적되는 게 도움이 될 때도 있지만 mechanical sweep에는 노이즈가 된다.
- **Huntley 원본 bash 한 줄 루프**: 새 컨텍스트 철학은 맞지만 stop condition이 `Ctrl+C` 외에 없고, 안전장치도 없고, 템플릿도 없다. 사내 코드베이스에서 야간 무인 가동시키기엔 무책임.

그래서 두 방향을 동시에 깔기로 했다 — 자체 스킬(자율 루프 본체)과 공식 플러그인 미러(필요 시 비교·전환용 백업).

## 핵심 결정

**1. 두 갈래 병행** — `ralph-skill`(자체)과 `official-ralph-mk`(공식 미러). 한쪽만 깔지 않은 이유는 두 패턴의 트레이드오프가 작업 종류마다 갈리기 때문. 사내 PC 한 대에서 둘을 다 시도해보고 수렴시키는 게 합리적.

**2. Pattern D — orchestrated fresh context** (`ralph-skill`이 채택). 부모 Claude Code 세션이 `Bash`로 `claude -p` 자식 세션을 띄우고, 자식은 매번 빈 컨텍스트에서 같은 `PROMPT.md`를 읽는다. Huntley 원본의 fresh-context 규율 + 부모가 stop 판단·진행 요약을 맡는 하이브리드.

**3. CLAUDE.md 룰을 스킬화** — 매 세션마다 Claude Code에게 "랄프 루프란 이런 거다"를 다시 설명하지 않게, `skills/ralph/SKILL.md` + `references/`로 박제. 외부망이 없어 context7으로 문서를 조회 못 하므로 Huntley·Willison·Anthropic headless 문서를 build time에 distill해서 `references/` 안에 넣었다.

**4. plugins / build / skills 디렉터리 분리** — `skills/`는 Claude Code 런타임이 읽는 자산, `build/`는 zip 패키징 + 설치 스크립트, `plugins/`(공식 미러 쪽)는 마켓플레이스 구조를 흉내낸 `.claude-plugin/marketplace.json` + `plugins/<name>/`. 역할별로 잘라야 사내 PC에서 어느 부분이 어디로 복사되는지 헷갈리지 않는다.

**5. 단일 오프라인 아티팩트** — `make build` 한 방으로 zip + sha256 생성. USB로 반입, 사내 PC에서 `bash install.sh` 한 번. pip·npm·플러그인 마켓 round-trip 0회.

**6. 안전장치** — `max_iter`, `max_wall`, `DONE.md` + 실행 가능한 `verify_cmd`, git HEAD 기반 `no_progress`, `consec_err`. 모두 mock `claude` 바이너리로 단위 테스트. Willison이 기록한 2025년 12월 `rm -rf ~/` 사고를 반면교사로, `--dangerously-skip-permissions`보다 `.claude/settings.json`의 `permissions.allow`를 우선 권고하도록 SKILL.md에 박았다.

## 스택과 구조

- **Claude Code 2.1+** — 사내 PC에서 사용 가능한 헤드리스 모드(`claude -p`)가 자식 세션 spawn의 핵심.
- **bash + Makefile** — 사내 환경에서 보장되는 최소 의존성. PowerShell 안 됨.
- **zip + sha256** — 반입 아티팩트. 무결성 검증을 USB 반입 후 `shasum -a 256 -c`로 확인.
- **두 디렉터리 차이 (중요)**:
  - `ralph-skill/` — **자체 작성한 오프라인 스킬**. Huntley 원본 철학(fresh context per iteration) + 안전장치 + 5개 템플릿(`bug-hunt`, `refactor-sweep`, `spec-to-impl`, `test-backfill`, `doc-sweep`). zip 21K, 27 파일.
  - `official-ralph-mk/` — **Anthropic 공식 `ralph-wiggum` 플러그인 미러**. 마켓플레이스에 접근 못 하는 사내망용으로 구조(`marketplace.json` + `plugins/ralph-wiggum/`)를 그대로 복제해뒀다. Stop hook으로 같은 세션에서 prompt 재투입 — `ralph-skill`과 정반대 메커니즘.

## 일화·구체 디테일

- 두 디렉터리 둘 다 2026-04-17 하루에 만들어졌다. ralph-skill은 14:26 LICENSE 커밋부터 시작해 15:10에 README 정리까지. official mirror는 15:04에 한 번에 카피.
- 사내망 제약을 푸는 핵심 트릭은 **build-time 사전 흡수**. 외부 문서(Huntley 블로그, Willison, HumanLayer, Anthropic headless docs)를 `research/`에 5개 md로 모아두고, 그중 핵심을 `skills/ralph/references/` 5개로 distill했다. 런타임에 Claude Code가 외부 호출 없이 이걸 그대로 읽는다.
- `/ralph` 슬래시 커맨드와 SKILL.md `description` 양쪽에 동일 트리거 표현을 박아서, 사용자가 명시 호출하지 않아도 "overnight refactor", "keep iterating until X" 같은 자연어로 자동 트리거되게 했다.
- 폴백 모드로 `orchestrator/run.sh` 순수 bash 스크립트도 함께 들어 있다. 부모 Claude Code 세션 없이 터미널에서 단독 실행 가능 — 사내 PC가 야간에 Claude Code UI를 안 띄워도 cron으로 굴릴 수 있게.

## 결과·현황

- 로컬에서 `make test` 9개 테스트 그린, `make build` 정상, `make install-local` 정상.
- 두 zip 아티팩트와 sha256 생성됨(`dist/`).
- 사내 PC 이관 가이드(`build/INSTALL.md`, `docs/release-2026-04-17.md`) 완성.
- **미확인**: 실제 사내 PC에 USB 반입 후 설치·구동 결과. 첫 unattended sweep이 성공적으로 수렴했는지, Claude Code Sonnet 4.6에서 fresh-context 패턴이 실전에서 잘 도는지. 회고는 그 데이터가 들어와야 정확해진다.

## 회고

**잘된 것**

- 사내망 제약(외부 차단·내부 npm·tgz 반입)이 오히려 패키징 규율을 강제했다 — single zip + sha256 + bash-only install. 일반 환경이었으면 plugin marketplace나 npm install로 적당히 넘겼을 것.
- "공식 플러그인 vs 자체 스킬" 양자택일을 안 한 게 다행. 두 패턴의 적합 영역이 갈리는데 사내 PC에 둘 다 깔리는 비용은 zip 두 개뿐이다.
- build-time 문서 distill 전략은 외부망 막힌 환경의 표준 답이 됐다. context7을 못 쓰는 만큼, 필요한 외부 지식을 빌드 단계에서 흡수해 박제하는 게 유일한 길.

**못된 것 / 회의 지점**

- 사내 PC에서 실제로 굴려본 결과가 아직 없다. 로컬 테스트는 mock `claude` 바이너리 — 실전 검증 아님.
- 두 아티팩트 모두 같은 날 1회 빌드 후 갱신 흐름이 없다. 버전 정책·갱신 trigger가 명시 안 됨.

**다시 한다면**

- ralph-skill 자체보다 사내 PC에서 도는 첫 sweep 1건을 먼저 set up하고 그 경험으로 스킬을 다듬었을 것. 사내 환경에 발 붙이지 않은 채로 추상 수준에서 안전장치를 설계한 게 과한 부분(`consec_err` 임계값 등)을 만들었을 가능성.
