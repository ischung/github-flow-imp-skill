---
description: /implement 단축어 — GitHub Flow 이슈 자동 구현
argument-hint: "[#이슈번호 | --inline 이슈내용]"
---

인자: $ARGUMENTS

아래 모드를 판별하여 즉시 실행한다.

---

## 모드 A: 인자 없음 (자동 선택)

1. `git remote get-url origin` 으로 OWNER/REPO 확인
2. `gh project list --owner OWNER --format json` 으로 PROJECT_NUMBER 확인
3. github-flow-impl 스킬의 **Step 0.5** 실행 — 칸반 자동화 워크플로우 설정 확인 및 초기화
   (PR 오픈 → Review 이동 / PR 머지 → Done 이동 GitHub Actions 설정)
4. `gh project item-list PROJECT_NUMBER --owner OWNER --format json` 으로 Todo 첫 이슈 선택
5. 이후 github-flow-impl 스킬의 Step 2~7 순서대로 실행
   - **Step 6 포함**: PR 생성 후 반드시 이슈를 Review 열로 이동
   - **PR 머지 후**: GitHub Actions(kanban-auto-done.yml)가 자동으로 Done 이동

---

## 모드 B: `#42` 형태 (번호 지정)

1. `git remote get-url origin` 으로 OWNER/REPO 확인
2. `gh project list --owner OWNER --format json` 으로 PROJECT_NUMBER 확인
3. github-flow-impl 스킬의 **Step 0.5** 실행 — 칸반 자동화 워크플로우 설정 확인 및 초기화
   (PR 오픈 → Review 이동 / PR 머지 → Done 이동 GitHub Actions 설정)
4. `gh issue view 42 --repo OWNER/REPO` 로 이슈 조회
5. 이후 github-flow-impl 스킬의 Step 2~7 순서대로 실행
   - **Step 6 포함**: PR 생성 후 반드시 이슈를 Review 열로 이동
   - **PR 머지 후**: GitHub Actions(kanban-auto-done.yml)가 자동으로 Done 이동

---

## 모드 C: `--inline` (직접 입력)

`--inline` 뒤의 내용을 파싱하여 **지금 즉시 순서대로 실행**한다.

### 1단계: 환경 확인 (실행)
```
git remote get-url origin
```
출력에서 OWNER와 REPO를 추출한다.

### 2단계: 프로젝트 번호 확인 (실행)
```
gh project list --owner OWNER --format json
```

### 3단계: 입력 파싱
- 티켓 번호: `[T-006]`, `T-006`, `티켓 번호: [T-006]` 등 어떤 형식이든 추출
- 제목: `제목:` 뒤 텍스트 또는 첫 번째 줄
- 본문: 상세 설명 전체
- AC: `수용 기준`, `AC:`, `- [ ]` 패턴 항목들 (없으면 본문 전체를 기준으로 사용)
- 티켓 번호가 있으면 제목 앞에 붙임: `[T-006] 제목`

### 4단계: GitHub 이슈 생성 (실행)
```
gh issue create --repo OWNER/REPO --title "제목" --body "본문 전체"
```
명령 실행 후 출력된 URL 마지막 숫자가 ISSUE_NUMBER다.

### 5단계: 보드 등록 (실행)
```
gh project item-add PROJECT_NUMBER --owner OWNER --url ISSUE_URL
```
2초 대기 후 아이템 ID 조회:
```
gh project item-list PROJECT_NUMBER --owner OWNER --format json --limit 100
```

### 6단계: To Do 상태 설정 (실행)
Status 필드와 Todo 옵션 ID 조회:
```
gh project field-list PROJECT_NUMBER --owner OWNER --format json
gh project list --owner OWNER --format json
```
조회한 ID로 상태 변경:
```
gh project item-edit --id ITEM_ID --field-id STATUS_FIELD_ID --project-id PROJECT_ID --single-select-option-id TODO_OPTION_ID
```


### 7단계 이후
github-flow-impl 스킬의 Step 2~7(In Progress 이동, 브랜치 생성, 구현, 테스트, PR, 완료 보고)을 순서대로 실행한다.
- **Step 6 포함**: PR 생성 후 반드시 이슈를 Review 열로 이동
- **PR 머지 후**: GitHub Actions(kanban-auto-done.yml)가 자동으로 Done 이동

