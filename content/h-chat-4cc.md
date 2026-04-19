---
title: h-chat-4cc
date: 2026-04-19
description: 외부망이 차단된 사내 PC에서 Claude Code를 굴리기 위한 사내 LLM ↔ Claude Code 브릿지. Python CLI + 스킬 래퍼, Playwright 백그라운드 자동화.
tags:
  - llm
  - internal-network
  - python-cli
  - claude-code
draft: false
---

# h-chat-4cc

## 한 줄

외부망이 차단된 사내 PC에서 Claude Code를 굴리기 위해, 사내 LLM 서비스를 Python CLI + Claude Code 스킬로 감싸 프로그래매틱 호출 가능하게 만든 브릿지 인프라.

## 컨텍스트 / 동기

사내 PC 환경 제약은 명확했다.

- 외부망 접속 불가 (외부 LLM API·웹검색·context7 모두 차단)
- 신규 패키지 설치 제약 (사내 미러만 사용 가능, 외부 PyPI/npm 불가)
- 정책상 막힌 건 아니지만 네트워크가 사실상 격벽
- MCP 서버 사내 배포도 금지 — 각자 로컬에서 띄워야 하는데 UX가 별로

그런데 사내에서 이미 자체 LLM 서비스가 굴러가고 있고, Sonnet 급 모델을 웹 UI로 쓸 수 있다. 이걸 Claude Code가 호출할 수 있게 뚫으면 사내 워크플로우와 랄프 루프를 사내 PC에서도 돌릴 수 있게 된다.

이게 개발 트리거였다. 사내 LLM 웹 UI는 사람이 클릭으로만 쓸 수 있는데, Claude Code의 Bash 도구가 호출할 수 있는 형태로 만들어야 한다. 즉, **사내 LLM ↔ Claude Code 브릿지**.

## 핵심 결정

**Python CLI + 얇은 스킬 래퍼로 간다 (MCP 아님).** MCP 서버 방식은 (1) 사내 배포 정책상 금지, (2) 각 PC에서 로컬 서버 상시 운용하는 UX가 별로여서 기각. CLI는 한 번 설치하면 `python -m src.hchat_cli ask "..."`로 호출 끝. Claude Code 스킬(`hchat`)은 트리거만 잡고 실제 실행은 CLI에 위임.

**브라우저는 백그라운드, 결정론적으로.** 사내망에서 우회로 간신히 쓰고 있는 Playwright를 활용. 사람에게 띄우지 않고 headless로 돌리고, 테스트할 때만 `--headed`로 확인. Claude가 직접 브라우저를 조종하면 비결정적이라 깨지므로 금지 룰을 스킬 SKILL.md에 박아둠.

**세션 재사용.** Playwright `storage_state`(`auth_state.json`)로 한 번 로그인 후 재사용. 매 호출마다 로그인 안 하니 빠름. 만료되면 종료 코드 3으로 신호 → 사용자에게 `login --headed` 안내.

**로컬 히스토리 자체 관리.** Q/A를 Python이 디스크(`history/<topic_id>/`)에 저장. Claude가 토큰 쓰는 건 "사용자가 읽으라고 시킬 때만". 사이트 전체 대화 증분 동기화(`sync-all`)도 지원.

**같은 토픽 / 새 토픽 / 임포트 모두 지원.** `--new-topic` (기본), `--continue-last`, `--topic <id>`. 백그라운드 브라우저는 호출 후 종료되므로 새 토픽이 자연스러운 디폴트. 토픽 ID 추출은 사이드바 셀렉터로 해결.

**마스킹된 단일 커밋만 원격 GitHub로.** 로컬 작업본은 실값 그대로, 원격 GitHub에는 사번·비밀번호·사내 호스트·회사명을 일괄 치환한 단일 커밋만 푸시한다. 포트폴리오 용도로 public 유지. 로컬·원격 분리는 파이프라인으로 고정해, 재push 시 동일 절차가 반복되도록 만들었다.

## 스택과 구조

- **Python 3.10+ + Playwright** — 결정론적 브라우저 자동화. argparse 기반 CLI 진입점(`src/hchat_cli.py`). 모듈 분리: `auth.py` / `model.py` / `search.py` / `topics.py` / `history.py` / `session.py` / `selectors.py` (DOM 셀렉터는 모두 한 파일에).
- **Claude Code 스킬 (`hchat`)** — `.claude/skills/hchat/SKILL.md`. 트리거("사내 LLM에 물어봐", "토픽 가져와", "전체 대화 동기화") → CLI 명령 라우팅 표 + 종료 코드 분기 + 절대 금지 룰.
- **사내 LLM API** — 사내 LLM 웹 UI를 Playwright로 자동 조작. 모델 리스트/선택(`list-models`, `set-default-model`), 응답 상태 감지(generating / completed / error / timeout), 웹검색 토글까지 다룸.
- **로컬 히스토리 디스크 저장** — `history/<topic_id>/`에 Q/A 누적. import / sync로 사람이 직접 사이트에서 한 대화도 끌어옴.
- **Phase 분할 + 프롬프트 패키지** — Phase 00~12까지 13단계로 쪼개고, 각 Phase마다 `dev/PHASE_XX_*.md` 지시서 + `prompts/PHASE_XX_prompt.md` 트리거 + `tests/TEST_PHASE_XX.md` 검증. 사내 Claude(Sonnet)가 한 Phase = 한 대화로 처리하도록 설계.

## 일화·구체 디테일

**왜 Phase 13단계로 쪼갰나.** 사내 Claude가 Sonnet 4.6 1M 토큰 버전이라 플래닝·오케스트레이션 능력이 떨어진다. 충분히 디테일하게 쓰되 잘게 쪼개서 컨텍스트 길이 안 터지게. Phase 간엔 새 대화 권장. 각 Phase 문서는 6섹션 공통 포맷(`0. 시작 전 확인 / 1. 목표 / 2. 사전 준비 / 3. 구현 / 4. 자가 검증 / 5. 커밋 / 6. 다음 Phase 조건`).

**마스킹 정책.** 원격 push 전에 임시 디렉터리에서 `sed` 기반 일괄 치환으로 사내 호스트·회사명·계정·비밀번호를 일반 플레이스홀더로 바꾼다. **절대 로컬 작업본은 마스킹하지 않는다** — 로컬은 실값으로, 원격은 마스킹으로 분리한다는 룰을 스킬 SKILL.md에 고정. 프로젝트/폴더/repo 이름(`h-chat_4cc`), CLI 서브커맨드·모듈명(`hchat`), 환경변수 prefix(`HCHAT_*`)는 유지한다.

**재push 절차.** 임시 디렉터리로 `rsync`(`.git`·`.env`·`auth_state.json`·`history`·`debug`·`dist` 제외) → `sed` 치환 → `git init` → 단일 커밋 → `git remote add` → `git push --force origin main` → 임시 디렉터리·스크립트 삭제. 전부 일시 산출물.

**SKILL.md 절대 금지 룰.** Claude가 CLI 우회해서 Playwright MCP로 직접 브라우저 조종 금지(결정론 깨짐), `.env` 출력 금지(비밀번호 들어있음), `auth_state.json` 수정·삭제 금지, 여러 명령 병렬 호출 금지(브라우저 자원·세션 충돌), 스킬 호출 없이 사내 LLM 응답 흉내내기 금지.

## 결과·현황

- 2026-04-17: 본격 설계. Sonnet 4.6 사내 PC 환경 제약 분석 → MCP 기각, CLI 채택, Phase 13단계 쪼개기.
- 2026-04-18: 사내 PC 이관 프롬프트 패키지 작성(`prompts/START.md` + Phase 별 트리거).
- 2026-04-19 기준 산출물: `PLAN.md` / `USAGE.md` 스텁(Phase 12에서 본문 채울 예정) / `prompts/` 13개 / `dev/` Phase 지시서 13개 / `tests/` Phase 별 체크리스트 + `SMOKE_TEST.md` / `src/` 모듈 스캐폴딩 / `.claude/skills/hchat/SKILL.md` 완성.
- GitHub 마스킹 단일 커밋 푸시 완료: <https://github.com/pistosmin/h-chat_4cc> (public, 포트폴리오).

## 회고

**잘된 것:**

- MCP vs CLI 의사결정이 명확. 사내 정책 + UX 두 축으로 잘랐고, 두 번 흔들리지 않음.
- 마스킹 규칙을 한 번 정해서 메모리에 박아두니 재push가 단순 작업이 됨. 로컬 실값 / 원격 마스킹 분리도 깔끔.
- Phase 분할 + 6섹션 공통 포맷이 사내 Sonnet 향 설계로 적절. "0. 시작 전 확인" 게이트가 있어서 헛발질 방지.

**못된 것 / 다시 한다면:**

- 사내 PC에서 실제로 Phase 00부터 굴려본 결과가 아직 이 노트에 없음. 진짜 동작 확인 전에는 "설계까지 완료"가 정확한 표현.
- Playwright 셀렉터가 사내 사이트 UI 변경에 취약. `selectors.py` 한 파일로 모은 건 잘했지만, 사이트 리뉴얼 한 번 들어가면 Phase 03~08 거의 다 깨질 가능성. 스냅샷 테스트 같은 회복 장치 없음.
- 로그인 자동화는 일부러 빼고 `login --headed` 수동에 맡겼는데, 팀 배포 시 동료가 매번 OTP 등 끼면 마찰 클 듯. 다시 한다면 OTP 가정한 자동화까지 한 단계 더.
