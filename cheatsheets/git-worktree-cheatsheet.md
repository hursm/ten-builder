# Git Worktree 치트시트

> AI 에이전트 병렬 개발을 위한 Git Worktree 핵심 명령어 한 페이지 요약

## 기본 명령어

| 명령어 | 설명 |
|--------|------|
| `git worktree add <경로> -b <브랜치>` | 새 브랜치 + worktree 생성 |
| `git worktree add <경로> <브랜치>` | 기존 브랜치로 worktree 생성 |
| `git worktree list` | 활성 worktree 목록 |
| `git worktree remove <경로>` | worktree 삭제 |
| `git worktree prune` | 유효하지 않은 참조 정리 |
| `git worktree lock <경로>` | worktree 잠금 (삭제 방지) |
| `git worktree unlock <경로>` | worktree 잠금 해제 |
| `git worktree move <경로> <새경로>` | worktree 이동 |

## 빠른 시작 (30초)

```bash
# 1. 현재 프로젝트에서 worktree 생성
git worktree add ../my-app-feat -b feat/new-feature

# 2. worktree로 이동
cd ../my-app-feat

# 3. 의존성 설치 후 작업 시작
npm install && claude "새 기능 구현해줘"

# 4. 작업 완료 후 PR 생성 & 정리
gh pr create && cd - && git worktree remove ../my-app-feat
```

## AI 병렬 개발 패턴

### 3-Agent 동시 작업

```bash
# 한 번에 3개 작업 공간 생성
git worktree add ../app-feat-1 -b feat/auth-v2
git worktree add ../app-feat-2 -b feat/dashboard
git worktree add ../app-fix-1  -b fix/memory-leak

# 각 worktree에서 AI 에이전트 실행
(cd ../app-feat-1 && claude "인증 시스템 리팩토링해줘") &
(cd ../app-feat-2 && claude "대시보드 구현해줘") &
(cd ../app-fix-1  && claude "메모리 누수 수정해줘") &
wait
```

### 일괄 정리

```bash
# 모든 linked worktree 삭제
git worktree list | tail -n +2 | awk '{print $1}' | xargs -I{} git worktree remove {}
git worktree prune
```

## 자주 쓰는 셸 함수

```bash
# worktree 생성 + npm install + 디렉토리 이동
wt-new() {
  local branch="$1"
  local dir="../$(basename $(pwd))-$(echo $branch | tr '/' '-')"
  git worktree add "$dir" -b "$branch"
  (cd "$dir" && npm install --silent 2>/dev/null)
  echo "✅ $dir ready — run: cd $dir"
}

# fzf로 worktree 선택 후 이동
wt-go() {
  local dir=$(git worktree list | fzf --height 40% | awk '{print $1}')
  [ -n "$dir" ] && cd "$dir"
}

# 모든 worktree에서 명령 실행
wt-all() {
  git worktree list --porcelain | grep "^worktree " | cut -d' ' -f2 | while read d; do
    echo "── $d ──"
    (cd "$d" && eval "$@")
  done
}
```

## 주의사항

| 항목 | 내용 |
|------|------|
| 같은 브랜치 | ❌ 두 worktree에서 동일 브랜치 체크아웃 불가 |
| node_modules | 📦 각 worktree에서 별도 `npm install` 필요 |
| .git 디렉토리 | 🔗 main worktree의 .git을 참조 (linked) |
| IDE 열기 | 💡 각 worktree를 별도 프로젝트로 열기 |
| stash | ⚠️ stash는 모든 worktree에서 공유됨 |
| submodule | ⚠️ 각 worktree에서 `git submodule update` 필요 |

## 네이밍 컨벤션

```
프로젝트명-작업유형-설명

my-app-feat-auth         # 기능: 인증
my-app-fix-perf          # 수정: 성능
my-app-exp-graphql       # 실험: GraphQL
my-app-review-pr-42      # 리뷰: PR #42
my-app-agent-1           # AI 에이전트 1번
```

## 한눈에 보기

```
                    ┌─ worktree-1 (feat/auth)     ← Agent 1
                    │
main worktree ──────┼─ worktree-2 (feat/dashboard) ← Agent 2
(.git 공유)          │
                    └─ worktree-3 (fix/perf)       ← Agent 3

    ↓ 완료 후

    worktree-1 → PR #101 → Review → Merge
    worktree-2 → PR #102 → Review → Merge
    worktree-3 → PR #103 → Review → Merge

    → git worktree prune (정리)
```

---

**가이드:** [Git Worktree + AI 병렬 개발 가이드](../guides/48-git-worktree-ai-parallel-dev.md)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
