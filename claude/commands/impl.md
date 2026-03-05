---
description: /implement 단축어 — GitHub Flow 이슈 자동 구현
argument-hint: "[#이슈번호 | --inline 이슈내용]"
allowed-tools: Bash(git *), Bash(gh *)
---

`github-flow-impl` 스킬의 지침에 따라 아래 인자를 처리하라.

인자: $ARGUMENTS

- 인자가 없으면 → **자동 선택 모드**: GitHub Projects Todo 열 최상단 이슈 선택
- 인자가 `#숫자` 형태이면 → **번호 지정 모드**: 해당 이슈 번호로 구현
- 인자가 `--inline`으로 시작하면 → **직접 입력 모드**: `--inline` 뒤의 내용을 이슈로 파싱
