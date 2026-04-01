# 플레이북 35: Git Worktree + AI 에이전트 병렬 작업

> 하나의 레포에서 여러 AI 코딩 에이전트를 동시에 돌리는 실전 플레이북

## 언제 쓰나요?

- 스프린트에 독립적인 작업이 3개 이상 있을 때
- 코드 리뷰 대기 중에 다른 작업을 시작하고 싶을 때
- 실험적인 변경을 안전하게 테스트하면서 안정 브랜치를 보호하고 싶을 때
- AI 에이전트에게 각각 다른 작업을 맡기고 동시에 진행할 때

## 전제 조건

- Git 2.15+ (worktree 기능)
- 독립적으로 분리 가능한 작업 목록
- 각 worktree에 충분한 디스크 공간 (프로젝트 크기 × N)

## 플레이북

### Phase 1: 작업 분석 및 분리

AI 에이전트에게 작업을 맡기기 전에 **의존성 분석**을 먼저 해요.

```
프롬프트:
"이 3개 작업의 파일 의존성을 분석해줘:
1. 사용자 인증 리팩토링
2. 대시보드 UI 구현
3. API 응답 캐싱

같은 파일을 수정하는 작업이 있으면 알려줘."
```

**판단 기준:**

| 조건 | 판정 |
|------|------|
| 수정 파일이 완전히 겹치지 않음 | ✅ 병렬 가능 |
| 공통 파일 1~2개 (설정 등) | ⚠️ 주의하며 병렬 가능 |
| 핵심 로직 파일이 겹침 | ❌ 순차 처리 권장 |

### Phase 2: Worktree 생성 및 환경 구성

```bash
#!/bin/bash
# setup-parallel-dev.sh

PROJECT=$(basename $(pwd))
TASKS=("feat/auth-refactor" "feat/dashboard-ui" "feat/api-caching")
DESCS=(
  "사용자 인증 시스템을 OAuth 2.0 + PKCE로 리팩토링"
  "React 대시보드 컴포넌트 구현"
  "Redis 기반 API 응답 캐싱 레이어 추가"
)

for i in "${!TASKS[@]}"; do
  SAFE=$(echo "${TASKS[$i]}" | tr '/' '-')
  DIR="../${PROJECT}-${SAFE}"

  # Worktree 생성
  git worktree add "$DIR" -b "${TASKS[$i]}"

  # 의존성 설치
  (cd "$DIR" && npm install --silent)

  # 작업별 CLAUDE.md 생성
  cat > "$DIR/CLAUDE.md" << EOF
# Task: ${TASKS[$i]}

## 목표
${DESCS[$i]}

## 범위 제한
- 이 worktree는 위 작업만 수행합니다
- 관련 없는 파일 수정 금지
- 완료 기준: 기능 구현 + 테스트 통과

## 프로젝트 규칙
- TypeScript strict mode
- 테스트 커버리지 80% 이상
- 린팅 통과 필수
EOF

  echo "✅ [$i] $DIR (${TASKS[$i]})"
done

echo ""
echo "📋 Worktrees:"
git worktree list
```

### Phase 3: AI 에이전트 병렬 실행

각 터미널에서 독립적으로 AI 에이전트를 실행해요.

**터미널 1:**
```bash
cd ../my-app-feat-auth-refactor
claude "CLAUDE.md를 읽고 인증 리팩토링을 시작해줘. 
기존 session 기반을 OAuth 2.0 PKCE 흐름으로 변경하고,
마이그레이션 스크립트도 만들어줘."
```

**터미널 2:**
```bash
cd ../my-app-feat-dashboard-ui
claude "CLAUDE.md를 읽고 대시보드를 구현해줘.
Shadcn/UI 컴포넌트를 사용하고, 반응형으로 만들어줘.
차트는 Recharts로."
```

**터미널 3:**
```bash
cd ../my-app-feat-api-caching
claude "CLAUDE.md를 읽고 API 캐싱 레이어를 구현해줘.
Redis를 캐시 백엔드로 쓰고, 캐시 무효화 전략도 포함해줘."
```

### Phase 4: 중간 점검

에이전트가 작업하는 동안 진행 상황을 확인해요.

```bash
# 각 worktree의 변경사항 요약
for dir in ../my-app-feat-*; do
  echo "=== $(basename $dir) ==="
  (cd "$dir" && git diff --stat)
  echo ""
done
```

### Phase 5: 테스트 및 PR 생성

```bash
# 각 worktree에서 테스트 실행
for dir in ../my-app-feat-*; do
  echo "🧪 Testing $(basename $dir)..."
  (cd "$dir" && npm test) || echo "❌ FAILED: $dir"
done

# 통과한 작업에 대해 PR 생성
for dir in ../my-app-feat-*; do
  cd "$dir"
  BRANCH=$(git branch --show-current)

  git add -A
  git commit -m "feat: $(cat CLAUDE.md | head -1 | sed 's/# Task: //')"
  git push origin "$BRANCH"

  gh pr create \
    --title "$(git log -1 --format=%s)" \
    --body "AI 에이전트가 병렬로 생성한 PR입니다.

## 작업 내용
$(cat CLAUDE.md | sed -n '/## 목표/,/## 범위/p' | head -n -1)

## 테스트
- [x] 단위 테스트 통과
- [ ] 코드 리뷰 완료
- [ ] QA 검증"

  cd -
done
```

### Phase 6: 정리

```bash
# 머지 완료된 worktree 정리
for dir in ../my-app-feat-*; do
  BRANCH=$(cd "$dir" && git branch --show-current)
  # PR이 머지되었는지 확인
  STATE=$(gh pr list --head "$BRANCH" --json state --jq '.[0].state')
  if [ "$STATE" = "MERGED" ]; then
    git worktree remove "$dir"
    git branch -d "$BRANCH"
    echo "🗑️  Removed: $dir ($BRANCH)"
  fi
done

git worktree prune
```

## 충돌 방지 전략

### 사전 분석 프롬프트

```
"다음 3개 작업이 수정할 파일 목록을 예측해줘:
1. OAuth 인증 리팩토링 — src/auth/, src/middleware/
2. 대시보드 UI — src/components/dashboard/
3. API 캐싱 — src/services/, src/cache/

겹치는 파일이 있으면 충돌 방지 전략을 제안해줘."
```

### 공통 파일 처리 규칙

| 파일 유형 | 전략 |
|-----------|------|
| `package.json` | 마지막에 한 번만 업데이트 |
| 라우트 정의 | 각 작업에서 추가만 (삭제/수정 금지) |
| 공통 타입 | 한 worktree에서만 수정 |
| 설정 파일 | 변경이 필요하면 순차 처리 |
| DB 마이그레이션 | 타임스탬프로 자동 정렬 |

## 체크리스트

```markdown
## 시작 전
- [ ] 작업 간 파일 의존성 분석 완료
- [ ] 겹치는 파일 없음 (또는 처리 전략 수립)
- [ ] 디스크 공간 충분 (프로젝트 × worktree 수)

## 실행 중
- [ ] 각 worktree에 CLAUDE.md 또는 작업 지시서 배치
- [ ] 의존성 설치 완료 (npm install / pip install 등)
- [ ] 각 에이전트가 범위를 벗어나지 않는지 확인

## 완료 후
- [ ] 각 worktree에서 테스트 통과
- [ ] PR 생성 및 리뷰 요청
- [ ] 머지 완료 후 worktree 정리
- [ ] `git worktree prune` 실행
```

## 성과 지표

| 지표 | 순차 처리 | 병렬 처리 (3 agents) |
|------|-----------|---------------------|
| 총 작업 시간 | 3~4시간 | 1~1.5시간 |
| 컨텍스트 스위칭 | 높음 | 없음 |
| 충돌 위험 | 중간 | 낮음 (격리) |
| 리뷰 부담 | 큰 PR 1개 | 작은 PR 3개 |

---

**관련 가이드:** [Git Worktree + AI 병렬 개발 가이드](../../guides/48-git-worktree-ai-parallel-dev.md)

**치트시트:** [Git Worktree 치트시트](../../cheatsheets/git-worktree-cheatsheet.md)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
