# github-flow-impl

GitHub Projects 보드의 이슈를 자동으로 구현하는 Claude Code 스킬 + 슬래시 커맨드.

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

브라우저 로그인 후 `gh project` 명령어를 쓰려면 `project` 스코프를 추가해야 한다:

```bash
gh auth refresh -s project
```

#### 방법 B — Personal Access Token (PAT) 사용

브라우저 인증이 불가한 환경(서버, CI 등)에서 사용.

**1단계 — PAT 발급**

1. https://github.com/settings/tokens 접속
2. `Generate new token (classic)` 클릭
3. Note(이름) 입력 예: `github-flow-impl`
4. 아래 스코프 체크:

| 스코프 | 용도 |
|--------|------|
| `repo` | 이슈 조회/수정, 브랜치 push, PR 생성 |
| `read:org` | 조직 저장소 접근 |
| `project` | GitHub Projects 보드 읽기/쓰기 (이슈 상태 변경) |

> ⚠️ `repo`, `read:org`만으로는 **부족합니다**.
> `gh project item-list`, `gh project item-edit` 등 Projects 관련 명령어는
> 반드시 `project` 스코프가 있어야 합니다.

5. `Generate token` 클릭 후 토큰 값 복사 (페이지를 벗어나면 다시 볼 수 없음)

**2단계 — PAT로 gh 인증**

```bash
echo YOUR_TOKEN | gh auth login --with-token
```

**3단계 — 인증 및 스코프 확인**

```bash
gh auth status
# ✓ Logged in to github.com as [username]
# ✓ Token scopes: repo, read:org, project
```

`project` 스코프가 없으면:

```bash
gh auth refresh -s project
```

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
