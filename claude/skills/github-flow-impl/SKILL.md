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

### Step 0.5 — 칸반 자동화 워크플로우 설정 확인 (최초 1회)

> **이 단계를 생략하면 PR 오픈/머지 후 칸반 카드가 자동으로 이동하지 않는다. 반드시 실행한다.**
>
> ⚠️ **사전 조건**: GitHub Actions가 `gh project` 명령을 실행하려면 기본 `GITHUB_TOKEN`이 아닌
> **`project` 스코프를 포함한 PAT**가 필요하다. 아래 순서로 한 번만 설정한다:
> 1. https://github.com/settings/tokens 에서 PAT 생성 (`project` 스코프 체크)
> 2. 저장소 Settings → Secrets and variables → Actions → **`KANBAN_TOKEN`** 이름으로 등록

마커 파일 `.github/.kanban-auto-done-configured`가 없으면 아래 자동화를 설정한다.

> **핵심 원칙**: 워크플로우 파일은 **feature 브랜치가 아닌 main에 직접 커밋**해야 한다.
> GitHub Actions는 base 브랜치(main)에 있는 워크플로우만 실행하기 때문에,
> feature 브랜치에 커밋하면 해당 PR이 머지될 때는 아직 main에 파일이 없어 첫 번째 PR부터 동작하지 않는다.

생성할 파일 3개:
- `_kanban-move.yml` — 공통 이슈 이동 로직 (reusable workflow)
- `kanban-auto-review.yml` — PR 오픈 시 Review 이동
- `kanban-auto-done.yml` — PR 머지 시 Done 이동

```bash
# 이 코드는 Step 3(브랜치 생성) 이전, main 브랜치 위에서 실행한다.
# 마커 파일이 없거나, 워크플로우가 구버전(repositoryOwner 미사용)이면 재생성한다.
if [ ! -f ".github/.kanban-auto-done-configured" ] || \
   ! grep -q "repositoryOwner" .github/workflows/_kanban-move.yml 2>/dev/null; then
  echo "🔧 칸반 자동화 설정 중 (신규 또는 구버전 업그레이드)..."

  mkdir -p .github/workflows

  # ── 1) 공통 reusable workflow ──────────────────────────────────────
  cat > .github/workflows/_kanban-move.yml << 'GHACTIONS'
# 재사용 가능한 공통 워크플로우 — 칸반 이슈 상태 이동
# Secret 이름을 바꾸려면 이 파일의 secrets.KANBAN_TOKEN 한 곳만 수정하면 됩니다.
#
# 필수 사전 설정:
#   저장소 Settings → Secrets → KANBAN_TOKEN 에
#   'project' 스코프를 포함한 PAT를 등록해야 합니다.
#   (GITHUB_TOKEN은 project 스코프가 없어 gh project 명령이 실패합니다)
name: Kanban — Move Issue Status (Reusable)

on:
  workflow_call:
    inputs:
      status:
        description: "이동할 칸반 컬럼 이름 (예: Review, Done)"
        required: true
        type: string
      pr_body:
        description: "PR 본문 (closes/fixes #N 파싱용)"
        required: true
        type: string
    secrets:
      KANBAN_TOKEN:
        required: true

jobs:
  move:
    runs-on: ubuntu-latest
    steps:
      - name: Move linked issues to ${{ inputs.status }}
        env:
          GH_TOKEN: ${{ secrets.KANBAN_TOKEN }}
          OWNER: ${{ github.repository_owner }}
          PR_BODY: ${{ inputs.pr_body }}
          TARGET_STATUS: ${{ inputs.status }}
          KANBAN_PROJECT_NUMBER: ${{ vars.KANBAN_PROJECT_NUMBER }}
        run: |
          set -euo pipefail
          unset GITHUB_OUTPUT GITHUB_ENV

          if [ -z "$GH_TOKEN" ]; then
            echo "::error::KANBAN_TOKEN secret이 설정되지 않았습니다."
            echo "::error::저장소 Settings → Secrets → KANBAN_TOKEN 에 project 스코프 PAT를 등록하세요."
            exit 1
          fi

          # PROJECT_NUMBER: Variable이 있으면 사용, 없으면 첫 번째 프로젝트 자동 탐색
          # repositoryOwner 를 사용하면 org/user 타입 무관하게 동작 ("unknown owner type" 방지)
          if [ -n "$KANBAN_PROJECT_NUMBER" ]; then
            PROJECT_NUMBER="$KANBAN_PROJECT_NUMBER"
          else
            PROJECT_NUMBER=$(gh api graphql \
              -f query='query($owner:String!){repositoryOwner(login:$owner){projectsV2(first:1,orderBy:{field:UPDATED_AT,direction:DESC}){nodes{number}}}}' \
              -f owner="$OWNER" | jq -r '.data.repositoryOwner.projectsV2.nodes[0].number')
          fi
          echo "PROJECT_NUMBER=$PROJECT_NUMBER"

          PROJECT_ID=$(gh api graphql \
            -f query='query($owner:String!,$num:Int!){repositoryOwner(login:$owner){projectV2(number:$num){id}}}' \
            -f owner="$OWNER" -F num="$PROJECT_NUMBER" | \
            jq -r '.data.repositoryOwner.projectV2.id')

          if [ -z "$PROJECT_ID" ]; then
            echo "::error::프로젝트 ID를 찾을 수 없습니다. OWNER=$OWNER, PROJECT_NUMBER=$PROJECT_NUMBER"
            exit 1
          fi
          echo "PROJECT_ID=$PROJECT_ID"

          FIELD_JSON=$(gh api graphql \
            -f query='query($owner:String!,$num:Int!){repositoryOwner(login:$owner){projectV2(number:$num){fields(first:30){nodes{...on ProjectV2SingleSelectField{id name options{id name}}}}}}}' \
            -f owner="$OWNER" -F num="$PROJECT_NUMBER")

          STATUS_FIELD_ID=$(echo "$FIELD_JSON" | \
            jq -r '.data.repositoryOwner.projectV2.fields.nodes[] | select(.name=="Status") | .id')
          TARGET_OPTION_ID=$(echo "$FIELD_JSON" | \
            jq -r --arg s "$TARGET_STATUS" \
              '.data.repositoryOwner.projectV2.fields.nodes[] | select(.name=="Status") | .options[] | select(.name==$s) | .id')

          if [ -z "$STATUS_FIELD_ID" ] || [ -z "$TARGET_OPTION_ID" ]; then
            echo "::error::Status 필드 또는 '$TARGET_STATUS' 옵션을 찾을 수 없습니다."
            echo "STATUS_FIELD_ID=$STATUS_FIELD_ID, TARGET_OPTION_ID=$TARGET_OPTION_ID"
            exit 1
          fi
          echo "STATUS_FIELD_ID=$STATUS_FIELD_ID, TARGET_OPTION_ID=$TARGET_OPTION_ID"

          # PR 본문에서 이슈 번호 추출 (closes/fixes/resolves #N)
          ISSUE_NUMBERS=$(echo "$PR_BODY" | \
            grep -oiE '(closes?|fixes?|resolves?)[[:space:]]+#[0-9]+' | \
            grep -oE '[0-9]+$' || true)

          if [ -z "$ISSUE_NUMBERS" ]; then
            echo "링크된 이슈 없음. 종료."
            exit 0
          fi
          echo "추출된 이슈 번호: $ISSUE_NUMBERS"

          for ISSUE_NUM in $ISSUE_NUMBERS; do
            echo "이슈 #$ISSUE_NUM → $TARGET_STATUS 이동 중..."

            # repositoryOwner 로 조회 — org/user 분기 불필요
            ITEM_ID=$(
              gh api graphql \
                -f query='query($owner:String!,$num:Int!){repositoryOwner(login:$owner){projectV2(number:$num){items(first:200){nodes{id content{...on Issue{number}}}}}}}' \
                -f owner="$OWNER" -F num="$PROJECT_NUMBER" | \
              jq -r --argjson n "$ISSUE_NUM" \
                '(.data.repositoryOwner.projectV2.items.nodes // [])[] | select(.content.number == $n) | .id'
            )

            if [ -z "$ITEM_ID" ]; then
              echo "  ⚠️  이슈 #$ISSUE_NUM 가 프로젝트에 없음, 건너뜀"
              continue
            fi
            echo "  ITEM_ID=$ITEM_ID"

            gh project item-edit \
              --project-id "$PROJECT_ID" \
              --id "$ITEM_ID" \
              --field-id "$STATUS_FIELD_ID" \
              --single-select-option-id "$TARGET_OPTION_ID"
            echo "  ✅ 이슈 #$ISSUE_NUM $TARGET_STATUS 이동 완료"
          done
GHACTIONS

  # ── 2) PR 오픈 → Review 이동 ──────────────────────────────────────
  cat > .github/workflows/kanban-auto-review.yml << 'GHACTIONS'
name: Kanban — Move to Review on PR Open

on:
  pull_request:
    types: [opened, reopened, ready_for_review]

jobs:
  move-to-review:
    if: "!github.event.pull_request.draft"
    uses: ./.github/workflows/_kanban-move.yml
    with:
      status: Review
      pr_body: ${{ github.event.pull_request.body }}
    secrets: inherit
GHACTIONS

  # ── 3) PR 머지 → Done 이동 ────────────────────────────────────────
  cat > .github/workflows/kanban-auto-done.yml << 'GHACTIONS'
name: Kanban — Move to Done on PR Merge

on:
  pull_request:
    types: [closed]

jobs:
  move-to-done:
    if: github.event.pull_request.merged == true
    uses: ./.github/workflows/_kanban-move.yml
    with:
      status: Done
      pr_body: ${{ github.event.pull_request.body }}
    secrets: inherit
GHACTIONS

  # ── 마커 파일 생성 (중복 설정 방지) ─────────────────────────────
  echo "configured $(date -u +%Y-%m-%dT%H:%M:%SZ)" > .github/.kanban-auto-done-configured

  # ── main에 직접 커밋 & 푸시 (feature 브랜치 생성 전에 반드시 실행) ──
  git add .github/workflows/_kanban-move.yml \
          .github/workflows/kanban-auto-review.yml \
          .github/workflows/kanban-auto-done.yml \
          .github/.kanban-auto-done-configured
  git commit -m "chore: add kanban automation workflows (reusable pattern)"
  git push origin main

  echo ""
  echo "✅ 칸반 자동화 설정 완료 (main에 직접 커밋됨)"
  echo "   · _kanban-move.yml       → 공통 이슈 이동 로직"
  echo "   · kanban-auto-review.yml → PR 오픈 시 Review 이동"
  echo "   · kanban-auto-done.yml   → PR 머지 시 Done 이동"
  echo "   💡 PR 본문에 'Closes #이슈번호'를 포함하면 자동으로 이동합니다."
  echo "   ⚠️  secrets.KANBAN_TOKEN 미설정 시 Actions 실행 시 오류 발생합니다."
fi
```

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

> **중요**: PR 본문에 반드시 `Closes #ISSUE_NUMBER` 형식을 포함한다.
> 이 키워드가 없으면 PR 머지 시 이슈가 자동으로 닫히지 않아 Done 자동 이동이 동작하지 않는다.

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
🤖 자동화: PR 머지 시 'Closes #ISSUE_NUMBER' 키워드에 의해 Done으로 자동 이동됩니다
```

---

## Constraints

- 한 번에 하나의 이슈만 처리
- `gh` CLI와 `git`만 사용
- 테스트 미통과 시 PR 생성 금지

## Prerequisites

- `gh auth login` 완료
- `gh` CLI 2.x 이상, `jq` 설치
