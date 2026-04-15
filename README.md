# github-flow-impl

GitHub Projects 보드의 이슈를 자동으로 구현하는 Claude Code 스킬 + 슬래시 커맨드.

---

## 시작 전 체크리스트

새 프로젝트에서 `/impl`을 처음 사용하기 전에 아래 항목을 순서대로 완료해야 합니다.

```
[ ] 1. gh CLI 설치 (2.x 이상)
[ ] 2. jq 설치
[ ] 3. gh auth login + project 스코프 추가
[ ] 4. KANBAN_TOKEN PAT 발급 (4가지 스코프)
[ ] 5. 저장소 Secret에 KANBAN_TOKEN 등록
[ ] 6. 칸반 보드 생성 (컬럼명 정확히 일치)
```

---

## 사전 요구사항

### 1. gh CLI 설치

```bash
# macOS
brew install gh

# 설치 확인
gh --version  # 2.x 이상 필요
```

### 2. jq 설치

```bash
brew install jq
```

### 3. GitHub 인증

#### 방법 A — 브라우저 로그인 (권장, 가장 간단)

```bash
gh auth login
```

프롬프트 순서:
1. `GitHub.com` 선택
2. `HTTPS` 선택
3. `Login with a web browser` 선택
4. 터미널에 표시된 one-time code 복사
5. 브라우저에서 https://github.com/login/device 접속 후 코드 붙여넣기
6. Authorize 클릭 → 인증 완료

브라우저 로그인 후 `gh project` 명령어를 쓰려면 `project` 스코프를 추가해야 합니다:

```bash
gh auth refresh -s project
```

#### 방법 B — Personal Access Token (PAT) 사용

브라우저 인증이 불가한 환경(서버, CI 등)에서 사용.

1. https://github.com/settings/tokens 접속
2. `Generate new token (classic)` 클릭
3. Note(이름) 입력 예: `github-flow-impl`
4. 아래 스코프 체크:

| 스코프 | 용도 |
|--------|------|
| `repo` | 이슈 조회/수정, 브랜치 push, PR 생성 |
| `read:org` | 조직 저장소 접근 |
| `project` | GitHub Projects 보드 읽기/쓰기 |

5. `Generate token` 클릭 후 토큰 값 복사

```bash
echo YOUR_TOKEN | gh auth login --with-token

# 확인
gh auth status
# ✓ Token scopes: repo, read:org, project
```

### 4. KANBAN_TOKEN 등록 (칸반 자동화 필수)

GitHub Actions가 PR 오픈/머지 시 칸반 이슈를 자동으로 이동합니다.
기본 `GITHUB_TOKEN`은 Projects 권한이 없으므로 별도 PAT가 필요합니다.

> **흐름 요약**
> PR 오픈 → `kanban-auto-review.yml` → 이슈를 **Review**로 이동
> PR 머지 → `kanban-auto-done.yml` → 이슈를 **Done**으로 이동

#### Step 1 — PAT 발급

1. https://github.com/settings/tokens 접속
2. **Generate new token (classic)** 클릭
3. Note: 아무 이름이나 입력 (예: `kanban-token`)
4. Expiration: 원하는 기간 선택
5. 아래 **4가지 스코프** 모두 체크:

| 스코프 | 필요한 이유 |
|--------|------------|
| `repo` | 저장소 읽기/쓰기 |
| `read:org` | 오너 타입(org/user) 판별 및 조직 접근 |
| `read:discussion` | GitHub Discussions 읽기 (gh CLI 내부 요구) |
| `project` | GitHub Projects 보드 읽기/쓰기 |

> ⚠️ 4가지 중 하나라도 빠지면 Actions 실행 시 아래 에러가 발생합니다:
> - `project` 누락 → `Resource not accessible by integration`
> - `read:org` 누락 → `your authentication token is missing required scopes [read:org]`
> - `read:discussion` 누락 → `your authentication token is missing required scopes [read:discussion]`

6. **Generate token** 클릭 후 토큰 복사 (페이지를 벗어나면 다시 볼 수 없음)

#### Step 2 — 저장소 Secret 등록

1. 저장소 → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** 클릭
3. 입력:
   - **Name**: `KANBAN_TOKEN` (워크플로우 코드와 반드시 일치)
   - **Secret**: Step 1에서 복사한 PAT
4. **Add secret** 클릭

> ⚠️ PAT의 메모 이름(Note)과 Secret Name은 **별개**입니다.
> Secret Name `KANBAN_TOKEN`만 워크플로우에서 `secrets.KANBAN_TOKEN`으로 참조됩니다.

#### Step 3 — 동작 확인

```bash
GH_TOKEN=<발급한 PAT> gh project list --owner <your-username>
# 프로젝트 목록이 출력되면 정상
```

### 5. 칸반 보드 생성

`/kanban-create` 스킬로 자동 생성하거나, 수동 생성 시 **Status 컬럼명이 정확히 일치**해야 합니다.

| 컬럼명 | 비고 |
|--------|------|
| `Todo` | 대소문자 정확히 일치 |
| `In Progress` | 공백 포함 |
| `Review` | |
| `Done` | |

---

## 설치 방법

### A. 모든 프로젝트에서 사용 (글로벌 스코프, 권장)

```bash
cd ~/Downloads
unzip github-flow-impl.zip -d github-flow-impl
cd github-flow-impl

# claude → .claude 로 이름 변경 후 글로벌 설치
mkdir -p ~/.claude/commands
mkdir -p ~/.claude/skills/github-flow-impl

cp claude/commands/implement.md ~/.claude/commands/
cp claude/commands/impl.md      ~/.claude/commands/
cp claude/skills/github-flow-impl/SKILL.md ~/.claude/skills/github-flow-impl/
```

### B. 특정 프로젝트에만 적용 (프로젝트 스코프)

```bash
cd ~/Downloads
unzip github-flow-impl.zip -d github-flow-impl
cd github-flow-impl

# claude → .claude 로 이름 변경 후 프로젝트에 설치
mkdir -p /your-project/.claude/commands
mkdir -p /your-project/.claude/skills/github-flow-impl

cp claude/commands/implement.md /your-project/.claude/commands/
cp claude/commands/impl.md      /your-project/.claude/commands/
cp claude/skills/github-flow-impl/SKILL.md /your-project/.claude/skills/github-flow-impl/
```

---

## 사용법

```bash
# Todo 최우선 이슈 자동 선택 후 구현
/implement

# 특정 이슈 번호 지정
/implement #42

# 이슈 내용 직접 입력 (번호·AC 필수)
/implement --inline #47 로그인 구현 ... AC: - [ ] 항목1

# 단축어
/impl
/impl #42
/impl --inline ...
```

---

## 파일 구조

```
.claude/
├── commands/
│   ├── implement.md   # /implement 슬래시 커맨드
│   └── impl.md        # /impl 단축어
└── skills/
    └── github-flow-impl/
        └── SKILL.md   # 스킬 본체 (지침 전체 포함)
```
