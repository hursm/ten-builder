# Git Worktree + AI 병렬 개발 실전 가이드

> 하나의 저장소에서 여러 AI 에이전트가 동시에 독립적으로 작업하는 워크플로우

## 왜 Git Worktree인가?

AI 코딩 에이전트가 강력해지면서 "한 번에 하나의 작업"이라는 기존 개발 방식의 한계가 드러나고 있어요. Claude Code, Cursor, Copilot Workspace 같은 도구로 여러 작업을 동시에 돌리고 싶은데, 같은 디렉토리에서 브랜치를 오가면 충돌이 나고 작업이 뒤섞이죠.

Git Worktree는 **하나의 `.git` 저장소를 공유하면서 여러 독립된 작업 디렉토리**를 만들 수 있는 기능이에요. 각 worktree는 서로 다른 브랜치를 체크아웃하고, 완전히 독립된 파일 시스템을 가져요. AI 에이전트 하나하나에 전용 작업 공간을 줄 수 있다는 뜻이에요.

## 핵심 개념

```
my-project/                  ← main worktree (main 브랜치)
├── .git/
└── src/

my-project-feat-auth/        ← linked worktree (feat/auth 브랜치)
└── src/

my-project-fix-perf/         ← linked worktree (fix/perf 브랜치)
└── src/
```

### 기존 방식 vs Worktree 방식

| | 기존 (단일 디렉토리) | Worktree (다중 디렉토리) |
|---|---|---|
| 동시 작업 | ❌ 브랜치 전환 필요 | ✅ 각 worktree가 독립 |
| AI 에이전트 | 한 번에 1개 | 동시에 N개 |
| 디스크 사용 | 1x | ~1.2x (파일만 복사, .git 공유) |
| 컨텍스트 오염 | ⚠️ stash 필요 | ✅ 완전 격리 |
| node_modules | 1개 | worktree별 별도 필요 |

## 기본 명령어

### Worktree 생성

```bash
# 새 브랜치를 만들면서 worktree 생성
git worktree add ../my-project-feat-auth -b feat/auth

# 기존 브랜치로 worktree 생성
git worktree add ../my-project-hotfix hotfix/login-bug

# 특정 경로에 생성
git worktree add ~/worktrees/my-project-experiment -b experiment/new-api
```

### Worktree 관리

```bash
# 목록 확인
git worktree list

# 정리 (삭제된 디렉토리 참조 제거)
git worktree prune

# worktree 제거
git worktree remove ../my-project-feat-auth

# 잠금 (실수로 삭제 방지)
git worktree lock ../my-project-feat-auth
```

## AI 에이전트 병렬 개발 워크플로우

### Step 1: 프로젝트 구조 설계

```bash
# 메인 프로젝트
cd ~/projects/my-app

# AI 에이전트용 worktree 3개 생성
git worktree add ../my-app-agent-1 -b feat/user-dashboard
git worktree add ../my-app-agent-2 -b feat/notification-system
git worktree add ../my-app-agent-3 -b fix/api-performance

# 각 worktree에 의존성 설치
for wt in ../my-app-agent-{1,2,3}; do
  (cd "$wt" && npm install)
done
```

### Step 2: 각 에이전트에 전용 워크스페이스 할당

```bash
# 터미널 1: Agent 1 — 사용자 대시보드
cd ~/projects/my-app-agent-1
claude "사용자 대시보드 컴포넌트를 만들어줘. React + TailwindCSS로."

# 터미널 2: Agent 2 — 알림 시스템
cd ~/projects/my-app-agent-2
claude "실시간 알림 시스템을 구현해줘. WebSocket + Redis Pub/Sub으로."

# 터미널 3: Agent 3 — 성능 최적화
cd ~/projects/my-app-agent-3
claude "API 응답 시간이 느린 엔드포인트를 찾아서 최적화해줘."
```

### Step 3: 결과 통합

```bash
# 메인 브랜치로 돌아와서 머지
cd ~/projects/my-app

# 각 브랜치 PR 생성
gh pr create --head feat/user-dashboard --title "feat: 사용자 대시보드" --body "AI 에이전트가 생성한 대시보드 컴포넌트"
gh pr create --head feat/notification-system --title "feat: 알림 시스템" --body "WebSocket 기반 실시간 알림"
gh pr create --head fix/api-performance --title "fix: API 성능 개선" --body "병목 엔드포인트 최적화"

# 리뷰 후 머지하면 worktree 정리
git worktree remove ../my-app-agent-1
git worktree remove ../my-app-agent-2
git worktree remove ../my-app-agent-3
```

## 실전 패턴

### 패턴 1: Task Isolation (작업 격리)

가장 기본적인 패턴이에요. 각 AI 에이전트에 독립된 작업을 할당하고, 완료 후 PR로 통합해요.

```bash
#!/bin/bash
# parallel-dev.sh — AI 에이전트 병렬 실행 스크립트

PROJECT_DIR=$(pwd)
PROJECT_NAME=$(basename "$PROJECT_DIR")
TASKS=("feat/auth-refactor" "feat/api-v2" "fix/memory-leak")

for task in "${TASKS[@]}"; do
  SAFE_NAME=$(echo "$task" | tr '/' '-')
  WT_DIR="../${PROJECT_NAME}-${SAFE_NAME}"

  echo "🔧 Creating worktree: $WT_DIR ($task)"
  git worktree add "$WT_DIR" -b "$task" 2>/dev/null

  # 의존성 설치
  (cd "$WT_DIR" && npm install --silent)

  echo "✅ Ready: $WT_DIR"
done

echo ""
echo "📋 Active worktrees:"
git worktree list
```

### 패턴 2: Review + Fix 병렬 처리

코드 리뷰와 버그 수정을 동시에 진행할 때 유용해요.

```bash
# Worktree 1: 코드 리뷰 중인 PR의 브랜치
git worktree add ../review-pr-42 pr/42

# Worktree 2: 리뷰에서 발견된 이슈 수정
git worktree add ../fix-from-review -b fix/review-42-findings

# AI 에이전트 1: 리뷰 계속
cd ../review-pr-42
claude "이 PR에서 보안 취약점과 성능 이슈를 찾아줘"

# AI 에이전트 2: 수정 작업
cd ../fix-from-review
claude "다음 이슈들을 수정해줘: [리뷰 결과 복사]"
```

### 패턴 3: Experiment + Stable 분리

실험적인 변경을 안전하게 진행하면서 안정 브랜치는 건드리지 않는 패턴이에요.

```bash
# 안정 브랜치 (배포 가능 상태 유지)
cd ~/projects/my-app  # main 브랜치

# 실험 브랜치들
git worktree add ../my-app-exp-graphql -b experiment/graphql-migration
git worktree add ../my-app-exp-rust -b experiment/rust-wasm-module

# 실험 실패하면 worktree만 삭제 — 메인 프로젝트는 깨끗
git worktree remove ../my-app-exp-rust
git branch -D experiment/rust-wasm-module
```

## CLAUDE.md 통합

각 worktree에 맞춤형 CLAUDE.md를 두면 AI 에이전트가 해당 작업에 더 집중할 수 있어요.

### 메인 프로젝트의 CLAUDE.md

```markdown
# CLAUDE.md — Main Branch

## Project
- Next.js 15 / TypeScript 5.5
- Prisma ORM + PostgreSQL
- TailwindCSS 4

## Rules
- main 브랜치에 직접 커밋하지 마세요
- 모든 변경은 feature 브랜치 → PR 경로
```

### Worktree용 CLAUDE.md (자동 생성)

```bash
#!/bin/bash
# generate-worktree-claude-md.sh

TASK_BRANCH=$(git branch --show-current)
TASK_DESC="$1"

cat > CLAUDE.md << EOF
# CLAUDE.md — Worktree: $TASK_BRANCH

## Task
$TASK_DESC

## Scope
- 이 worktree는 "$TASK_BRANCH" 작업 전용입니다
- 다른 기능 영역의 파일은 수정하지 마세요
- 작업 완료 후 PR을 생성할 예정입니다

## Base Branch Rules
$(cat ../my-app/CLAUDE.md 2>/dev/null || echo "메인 프로젝트 CLAUDE.md를 참조하세요")
EOF

echo "✅ CLAUDE.md generated for $TASK_BRANCH"
```

## 주의사항과 팁

### ⚠️ 주의할 점

1. **같은 브랜치를 두 worktree에서 체크아웃할 수 없어요**
   - worktree당 하나의 브랜치만 가능

2. **node_modules / venv 등 의존성은 각 worktree에서 별도 설치 필요**
   - 심볼릭 링크로 공유하면 문제가 생길 수 있어요

3. **IDE 설정 충돌**
   - `.vscode/settings.json`은 각 worktree에서 독립적으로 관리

4. **디스크 공간**
   - 소스 파일은 복사되지만 `.git`은 공유하므로 overhead는 적음
   - `node_modules`가 주요 디스크 소비원

### 💡 생산성 팁

```bash
# 1. 자주 쓰는 worktree 경로를 alias로 등록
alias wt-list="git worktree list"
alias wt-add="git worktree add"
alias wt-rm="git worktree remove"

# 2. fzf로 worktree 빠르게 전환
wt() {
  local selected
  selected=$(git worktree list | fzf --height 40% | awk '{print $1}')
  [ -n "$selected" ] && cd "$selected"
}

# 3. 모든 worktree에서 명령 실행
wt-exec() {
  git worktree list --porcelain | grep "^worktree " | sed 's/worktree //' | while read -r dir; do
    echo "=== $dir ==="
    (cd "$dir" && eval "$@")
  done
}

# 모든 worktree에서 테스트 실행
wt-exec "npm test"
```

## 실전 시나리오: 3개 에이전트 동시 작업

```
시간    Agent 1 (Dashboard)    Agent 2 (Notifications)    Agent 3 (Performance)
────    ──────────────────    ─────────────────────────    ───────────────────────
0:00    worktree 생성          worktree 생성               worktree 생성
0:02    컴포넌트 설계           WebSocket 서버 구현         프로파일링 시작
0:10    UI 구현                이벤트 핸들러 구현           병목 지점 분석
0:20    API 연동               Redis Pub/Sub 통합          쿼리 최적화
0:30    테스트 작성             테스트 작성                 벤치마크 작성
0:35    PR 생성                PR 생성                     PR 생성
```

**총 소요 시간:** 순차 처리 시 ~1.5시간 → 병렬 처리 시 ~35분 (약 60% 단축)

## 마무리

Git Worktree + AI 에이전트 조합은 개발 속도를 비약적으로 높여주는 실전 패턴이에요. 핵심은:

- **격리**: 각 에이전트에 독립된 작업 공간
- **병렬성**: 동시에 여러 작업 진행
- **안전성**: 메인 브랜치는 항상 깨끗하게 유지
- **통합**: PR 기반으로 체계적인 머지

한 번에 3~5개 에이전트를 돌리면서 하루에 처리할 수 있는 작업량이 2~3배 늘어나는 걸 경험해 보세요.

---

**관련 콘텐츠:**
- [멀티 에이전트 오케스트레이션 실전 패턴](40-multi-agent-orchestration.md)
- [AI 백그라운드 코딩 에이전트 가이드](46-background-coding-agents.md)
- [서브에이전트 병렬 개발 예제](../examples/subagent-parallel-dev/)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
