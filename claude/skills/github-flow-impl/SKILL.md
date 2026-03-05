---
name: github-flow-impl
description: >
  GitHub Projects 보드에서 우선순위 이슈를 자동으로 선택하거나, 사용자가 직접 이슈 내용을
  입력하여 GitHub Flow에 따라 브랜치 생성, 코드 구현, 로컬 테스트, PR 생성까지
  end-to-end로 처리하는 AI 개발자 스킬.
  사용자가 "/implement", "/impl", "이슈 처리해줘", "다음 이슈 구현해줘",
  "GitHub 보드 작업 시작", "Todo 이슈 자동 처리", "이슈 직접 입력해서 구현" 등을
  언급하면 반드시 이 스킬을 사용할 것.
  특정 이슈 번호("/implement #42") 또는 이슈 내용을 직접 입력하는
  "/implement --inline" 형태 요청 시에도 이 스킬을 사용할 것.
---

# GitHub Flow Auto-Implementation Skill

## Commands

| 커맨드 | 이슈 소스 | 설명 |
|--------|-----------|------|
| `/implement` | GitHub Projects | Todo 열 최우선 이슈 자동 선택 후 구현 |
| `/implement #42` | GitHub Issues | 특정 이슈 번호 지정 후 구현 |
| `/implement --inline` | 직접 입력 | 이슈 내용 입력 → GitHub 이슈 생성 → 보드 등록 → 구현 |
| `/impl` | — | `/implement` 단축어 |

---

## Role

너는 이 프로젝트의 **구현 담당 AI 개발자**다. 한 번에 **하나의 이슈만** 처리한다.

---

## Step 0 — 환경 감지 (모든 모드 공통, 가장 먼저 실행)

아래 명령을 실제로 실행하여 OWNER, REPO, PROJECT_NUMBER를 확인한다.

```bash
git remote get-url origin
```

위 출력에서 `github.com/OWNER/REPO` 형식으로 OWNER와 REPO를 추출한다.
실패하면 `gh repo view --json owner,name` 으로 대체한다.

```bash
gh project list --owner OWNER --format json
```

프로젝트 목록을 출력하고 PROJECT_NUMBER를 확인한다.
- 1개: 자동 사용
- 2개 이상: 사용자에게 선택 요청
- 0개: "연결된 프로젝트가 없습니다" 안내 후 중단

---

## Step 1 — 이슈 선정

### 모드 A: `/implement` (자동 선택)

```bash
gh project item-list PROJECT_NUMBER --owner OWNER --format json
```

Status가 "Todo"인 첫 번째 아이템을 선택한다. 없으면 중단.

### 모드 B: `/implement #42` (번호 지정)

```bash
gh issue view 42 --repo OWNER/REPO
```

### 모드 C: `/implement --inline` (직접 입력)

사용자 입력에서 아래 항목을 파싱한다:

- **티켓 번호**: `[T-006]`, `T-006`, `티켓 번호: [T-006]` 등 어떤 형식이든 추출. 없으면 무시.
- **제목**: `제목:` 레이블 뒤 텍스트, 또는 첫 번째 의미 있는 줄.
- **본문**: 상세 설명 전체.
- **AC**: `수용 기준`, `AC:`, `- [ ]` 패턴 뒤 항목들. 없으면 본문 전체를 구현 기준으로 사용.

파싱 후 **즉시** 아래 명령을 실행하여 GitHub 이슈를 생성한다:

```bash
gh issue create \
  --repo OWNER/REPO \
  --title "ISSUE_TITLE" \
  --body "ISSUE_BODY"
```

- 티켓 번호가 있으면 제목 앞에 붙인다: `[T-006] TimerDisplay 컴포넌트 구현`
- 명령 실행 후 출력된 URL에서 이슈 번호를 추출한다 (URL 마지막 숫자)

이슈 생성 직후 보드에 등록한다:

```bash
gh project item-add PROJECT_NUMBER --owner OWNER --url ISSUE_URL
```

등록 후 2초 대기, 이후 아이템 ID를 조회하여 To Do 상태로 설정한다:

```bash
gh project item-list PROJECT_NUMBER --owner OWNER --format json --limit 100
```

조회된 아이템 중 방금 생성된 이슈 번호와 일치하는 항목의 ID를 찾는다.

```bash
gh project item-edit \
  --id ITEM_ID \
  --field-id STATUS_FIELD_ID \
  --project-id PROJECT_ID \
  --single-select-option-id TODO_OPTION_ID
```

STATUS_FIELD_ID, PROJECT_ID, TODO_OPTION_ID는 아래로 조회한다:

```bash
gh project field-list PROJECT_NUMBER --owner OWNER --format json
gh project list --owner OWNER --format json
```

---

## Step 2 — In Progress 이동 및 담당자 할당

모드 A/B의 경우 이슈가 보드에 없으면 Step 1 모드 C의 보드 등록 과정과 동일하게 추가한다.

```bash
gh project item-edit \
  --id ITEM_ID \
  --field-id STATUS_FIELD_ID \
  --project-id PROJECT_ID \
  --single-select-option-id IN_PROGRESS_OPTION_ID

gh issue edit ISSUE_NUMBER --add-assignee @me --repo OWNER/REPO
```

---

## Step 3 — 브랜치 생성

```bash
git checkout main && git pull origin main
git checkout -b BRANCH_NAME
```

브랜치 이름 규칙:
- 티켓 번호 있음: `feature/t-006-timer-display`
- 티켓 번호 없음: `feature/issue-42-login-page`

---

## Step 4 — 코드 구현

먼저 프로젝트 구조를 파악한다:

```bash
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.py" \) \
  | grep -v node_modules | grep -v .git | head -30
cat package.json 2>/dev/null || true
```

AC가 있으면 AC 항목 기준으로, 없으면 이슈 설명 전체를 기준으로 구현한다.
기존 코드 컨벤션(파일 구조, 네이밍, import 스타일)을 엄격히 준수한다.

---

## Step 5 — 로컬 테스트

```bash
npm run build 2>/dev/null || yarn build 2>/dev/null || true
npm test 2>/dev/null || yarn test 2>/dev/null || true
```

테스트 실패 시 코드 수정 후 재실행. 1회 재시도 후에도 실패 시 사용자에게 보고 후 중단.

---

## Step 6 — PR 생성 및 Review 이동

```bash
git add -A
git commit -m "feat: SUMMARY (closes #ISSUE_NUMBER)"
git push origin BRANCH_NAME

gh pr create \
  --repo OWNER/REPO \
  --base main \
  --title "ISSUE_TITLE (#ISSUE_NUMBER)" \
  --body "## Summary
SUMMARY

## Changes
- CHANGE_1
- CHANGE_2

Closes #ISSUE_NUMBER"
```

PR 생성 후 Review 열로 이동:

```bash
gh project item-edit \
  --id ITEM_ID \
  --field-id STATUS_FIELD_ID \
  --project-id PROJECT_ID \
  --single-select-option-id REVIEW_OPTION_ID
```

---

## Step 7 — 완료 보고

```
✅ 작업 완료

📌 이슈: #ISSUE_NUMBER ISSUE_TITLE
🌿 브랜치: BRANCH_NAME
🔗 PR: PR_URL
✔️ 테스트: 통과
📬 상태: Review로 이동 완료
```

---

## Constraints

- 한 번에 하나의 이슈만 처리
- `gh` CLI와 `git`만 사용
- 테스트 미통과 시 PR 생성 금지

## Prerequisites

- `gh auth login` 완료
- `gh` CLI 2.x 이상, `jq` 설치
