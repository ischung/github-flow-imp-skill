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
| `/implement #[번호]` | GitHub Issues | 특정 이슈 번호 지정 후 구현 (예: `/implement #42`) |
| `/implement --inline` | 직접 입력 | 이슈 내용(번호·AC 포함)을 커맨드 뒤에 바로 붙여 입력하여 구현 |
| `/impl` | GitHub Projects | `/implement` 단축어 |
| `/impl #[번호]` | GitHub Issues | `/implement #[번호]` 단축어 |
| `/impl --inline` | 직접 입력 | `/implement --inline` 단축어 |


---

## Role

너는 이 프로젝트의 **구현 담당 AI 개발자**다.
GitHub Projects 보드에서 가장 높은 우선순위의 이슈를 가져와서 GitHub Flow에 따라 작업을 완료한다.
한 번에 **하나의 이슈만** 처리한다.

---

## Task Sequence

### Step 0 — 프로젝트 컨텍스트 자동 감지

모든 모드 실행 전 가장 먼저 수행한다. `git`과 `gh`를 이용해 OWNER와 PROJECT_NUMBER를 자동으로 감지한다.

**OWNER 감지**
```bash
# 1순위: git remote에서 추출
git remote get-url origin \
  | sed -E 's|.*github\.com[:/]([^/]+)/.*|\1|'

# 위 명령이 실패할 경우 2순위: gh 인증 계정 사용
gh api user --jq '.login'
```

**PROJECT_NUMBER 감지**
```bash
# 현재 repo에 연결된 Projects 목록 조회
gh project list --owner $OWNER --format json \
  | jq '.projects[] | {number: .number, title: .title}'
```

- 프로젝트가 **1개**이면 → 자동으로 해당 번호 사용
- 프로젝트가 **2개 이상**이면 → 목록을 출력하고 사용자에게 선택 요청:
  > "여러 프로젝트가 감지됐습니다. 사용할 프로젝트 번호를 선택해주세요:"
- 프로젝트가 **0개**이면 → 중단 후 안내:
  > "이 저장소에 연결된 GitHub Project를 찾을 수 없습니다."

감지된 값은 이후 모든 단계에서 `[OWNER]`, `[PROJECT_NUMBER]`로 재사용한다.

---

### Step 1 — 이슈 선정

**모드 판별표:**

| 입력 | 모드 | 동작 |
|------|------|------|
| `/implement` | 자동 선택 | Projects Todo 열 최상단 이슈 조회 |
| `/implement #42` | 번호 지정 | 해당 이슈 직접 조회 |
| `/implement --inline` | 직접 입력 | 템플릿 제시 → 사용자 입력 → GitHub 이슈 생성 |

---

**[자동 선택 모드]** `/implement`
```bash
# Step 0에서 감지된 OWNER, PROJECT_NUMBER 자동 사용
gh project item-list $PROJECT_NUMBER --owner $OWNER --format json \
  | jq '[.items[] | select(.status == "Todo")] | first'
```
Todo 열이 비어있으면 → "처리할 이슈가 없습니다." 출력 후 중단.

---

**[번호 지정 모드]** `/implement #42`
```bash
gh issue view 42
```

---

**[직접 입력 모드]** `/implement --inline [이슈 내용]`

사용자가 `/implement --inline` 뒤에 이슈 내용을 자유 형식으로 바로 붙여 입력한다.
Claude는 템플릿을 제시하지 않고 입력된 내용을 즉시 파싱하여 처리한다.

**입력 예시:**
```
/implement --inline
#47 로그인 페이지 구현

사용자가 이메일/비밀번호로 로그인할 수 있어야 한다.

AC:
- [ ] 이메일 형식 유효성 검사
- [ ] 로그인 실패 시 에러 메시지 표시
- [ ] 로그인 성공 시 /dashboard로 리다이렉트
```

**Claude의 파싱 및 검증 절차:**

1. **이슈 번호 확인** — 입력에서 `#숫자` 패턴을 찾는다.
   - 있으면 → 해당 번호를 `issue_number`로 사용
   - 없으면 → **중단 후 사용자에게 요청:**
     > "이슈 번호가 없습니다. `#번호`를 입력에 포함해주세요. (예: `#47`)"

2. **AC(수용 기준) 확인** — `AC`, `Acceptance Criteria`, `수용 기준`, `- [ ]` 등의 패턴을 찾는다.
   - 있으면 → AC 항목 목록 추출
   - 없으면 → **중단 후 사용자에게 요청:**
     > "수용 기준(AC)이 없습니다. 구현 완료 조건을 항목으로 추가해주세요."

3. 번호와 AC 모두 확인되면 → 이슈 번호와 파싱된 내용으로 이후 단계 진행
   (GitHub에 별도 이슈 생성 없이 입력된 내용을 그대로 사용)

---

추출 항목: `issue_number`, `issue_title`, `item_id`

---

### Step 2 — 상태 변경 및 담당자 할당

STATUS_FIELD_ID, PROJECT_ID, 각 열의 OPTION_ID는 아래 명령으로 자동 조회한다:

```bash
# 프로젝트 ID 및 상태 필드 정보 조회
gh project field-list $PROJECT_NUMBER --owner $OWNER --format json \
  | jq '.fields[] | select(.name == "Status") | {id: .id, options: .options}'
```

조회 결과에서 `In Progress` 와 `Review` 의 option ID를 추출하여 아래 명령에 사용한다:

```bash
# In Progress로 이동 (자동 감지된 값 사용)
gh project item-edit \
  --id $ITEM_ID \
  --field-id $STATUS_FIELD_ID \
  --project-id $PROJECT_ID \
  --single-select-option-id $IN_PROGRESS_OPTION_ID

# AI(현재 인증된 계정)를 담당자로 할당
gh issue edit $ISSUE_NUMBER --add-assignee @me
```

---

### Step 3 — 브랜치 생성

```bash
# main 최신화 후 브랜치 생성
git checkout main && git pull origin main
git checkout -b feature/issue-[NUMBER]-[한두단어-요약]
```

브랜치 명명 규칙: `feature/issue-42-add-login-page`

| 모드 | `[NUMBER]` 출처 |
|------|----------------|
| 자동 선택 | Projects 보드에서 조회한 이슈 번호 |
| 번호 지정 | 커맨드에 입력한 `#번호` |
| 직접 입력 | 사용자가 이슈 내용에 포함한 `#번호` |

> **직접 입력 모드**에서는 `gh issue view`를 호출하지 않고,
> 사용자가 입력한 내용을 이슈 본문으로 그대로 사용한다.

---

### Step 4 — 코드 구현

- `tasks.md` (프로젝트 루트)를 읽어 전체 맥락 파악
- 이슈의 **수용 기준(AC, Acceptance Criteria)** 을 기준으로 구현
- 프로젝트 기존 컨벤션 **엄격히** 준수 (파일 구조, 네이밍, 포맷, import 스타일)
- CI/CD 없이 **로컬 환경 최적화** 기준으로 작성

`tasks.md`가 없으면 → 이슈 내용만으로 진행, 누락 사실을 사용자에게 고지.

---

### Step 5 — 로컬 테스트

```bash
# 빌드
npm run build       # 또는 yarn build / pnpm build 등 프로젝트에 맞게

# 테스트 실행
npm test

# AC 항목 전체 충족 여부 수동 검증
```

테스트 실패 시 → 코드 수정 후 재실행. 통과 전까지 PR 생성 금지.
1회 재시도 후에도 실패 시 → 오류 내용과 함께 사용자에게 보고 후 중단.

---

### Step 6 — PR 생성

```bash
git add -A
git commit -m "feat: [간략한 설명] (closes #[ISSUE_NUMBER])"
git push origin feature/issue-[NUMBER]-[요약]

gh pr create \
  --base main \
  --title "[이슈 제목] (#[ISSUE_NUMBER])" \
  --body "## Summary
[구현 내용 요약]

## Changes
- [변경사항 1]
- [변경사항 2]

Closes #[ISSUE_NUMBER]"
```

PR 생성 후 이슈를 Review 열로 이동 (Step 2에서 조회한 값 재사용):
```bash
gh project item-edit \
  --id $ITEM_ID \
  --field-id $STATUS_FIELD_ID \
  --project-id $PROJECT_ID \
  --single-select-option-id $REVIEW_OPTION_ID
```

---

### Step 7 — 완료 보고

작업 완료 후 아래 형식으로 보고:

```
✅ 작업 완료 보고

📌 이슈: #[NUMBER] [TITLE]
🌿 브랜치: feature/issue-[NUMBER]-[요약]
🔗 PR: [PR_URL]

📋 작업 내용:
- [주요 변경사항 1]
- [주요 변경사항 2]

✔️ 테스트: 통과
📬 이슈 상태: Review로 이동 완료
```

---

## Constraints

- 한 번에 **하나의 이슈만** 처리
- `gh` CLI와 `git` 명령어만 사용 (GitHub 웹 UI 사용 금지)
- 프로젝트 기존 컨벤션 엄격히 준수
- 테스트 미통과 시 PR 생성 금지
- CI/CD 파이프라인 설정 변경 금지

---

## Prerequisites

실행 전 확인사항:
- `gh auth login` 완료
- `gh` CLI 버전 2.x 이상
- `jq` 설치 (`brew install jq` 또는 `apt install jq`)
