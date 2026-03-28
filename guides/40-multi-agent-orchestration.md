# 가이드 40: 멀티 에이전트 오케스트레이션 실전 패턴

> 여러 AI 코딩 에이전트를 동시에 운영하고, 작업을 분배하고, 결과를 통합하는 구체적인 방법

## 왜 멀티 에이전트가 필요한가

단일 AI 에이전트가 할 수 있는 일에는 한계가 있습니다. 컨텍스트 윈도우는 유한하고, 한 에이전트가 프론트엔드 작업을 하는 동안 백엔드는 대기 상태가 됩니다. 2026년 현재, 실무에서 체감되는 가장 큰 생산성 차이는 **에이전트 하나를 잘 쓰는 것**이 아니라 **여러 에이전트를 동시에 굴리는 것**에서 나옵니다.

멀티 에이전트 오케스트레이션은 이런 상황에서 효과적입니다:

- 프론트/백엔드가 분리된 풀스택 프로젝트
- 기능 구현과 테스트를 동시에 진행해야 할 때
- 대규모 리팩토링에서 여러 모듈을 병렬 처리할 때
- 코드 리뷰, 문서화, 보안 검사를 파이프라인으로 자동화할 때

## 핵심 패턴 5가지

### 1. Fan-Out / Fan-In (병렬 분산)

가장 기본적이고 효과적인 패턴입니다. 하나의 작업을 여러 서브태스크로 나누고, 각각을 별도 에이전트에게 할당한 후 결과를 통합합니다.

```
[오케스트레이터]
    ├── Agent A: frontend/components → 완료
    ├── Agent B: backend/api → 완료
    └── Agent C: tests/ → 완료
         ↓
    [결과 통합 → PR 생성]
```

**실전 구현 (Claude Code + worktree):**

```bash
# Step 1: worktree로 독립 작업 공간 생성
git worktree add ../project-frontend feature/frontend
git worktree add ../project-backend feature/backend
git worktree add ../project-tests feature/tests

# Step 2: 각 worktree에서 에이전트 실행
cd ../project-frontend && claude "components 폴더의 UI 리팩토링" &
cd ../project-backend && claude "API 엔드포인트 최적화" &
cd ../project-tests && claude "통합 테스트 추가" &

# Step 3: 모든 작업 완료 후 merge
wait
git merge feature/frontend feature/backend feature/tests
```

**적합한 경우:** 서로 의존성이 낮은 독립 작업, 대규모 리팩토링
**주의점:** 같은 파일을 수정하면 merge conflict 발생 → 작업 범위를 명확히 분리

### 2. Pipeline (순차 파이프라인)

이전 에이전트의 출력이 다음 에이전트의 입력이 되는 직렬 구조입니다. 코드 품질 게이트로 특히 효과적입니다.

```
[스펙 작성 Agent]
    → [구현 Agent]
    → [테스트 Agent]
    → [리뷰 Agent]
    → [문서화 Agent]
```

**실전 구현:**

```bash
# Phase 1: 스펙 생성
claude "이 요구사항을 기술 스펙으로 변환" --output spec.md

# Phase 2: 구현
claude "spec.md를 기반으로 구현" --allowedTools Edit,Write

# Phase 3: 테스트
claude "구현된 코드에 대한 테스트 작성 및 실행"

# Phase 4: 리뷰
claude "코드 리뷰: 보안, 성능, 코드 스타일 검사"
```

**적합한 경우:** 코드 품질이 중요한 프로덕션 코드, 규제가 있는 도메인
**주의점:** 파이프라인이 길어질수록 전체 시간 증가 → 꼭 필요한 단계만 포함

### 3. Supervisor (감독자 패턴)

하나의 에이전트가 전체를 감독하면서, 필요에 따라 전문 에이전트를 호출합니다.

```
[Supervisor Agent]
    ├── "이 버그는 DB 관련" → DB 전문 Agent
    ├── "이건 프론트엔드" → UI 전문 Agent
    └── "보안 이슈 감지" → Security Agent
```

**실전 구현 (Claude Code subagent):**

```javascript
// CLAUDE.md에 서브에이전트 규칙 정의
// Supervisor는 Task를 분석하고 적절한 전문 에이전트에 위임

// claude-code/commands/supervisor.md
// "다음 작업을 분석하고 적절한 전문 에이전트에게 위임하세요:
//  - frontend 관련: /frontend-agent
//  - backend 관련: /backend-agent  
//  - infra 관련: /infra-agent"
```

**적합한 경우:** 작업 유형이 다양하고 예측하기 어려운 경우
**주의점:** Supervisor의 판단력이 전체 품질을 좌우 → 명확한 라우팅 규칙 필요

### 4. Critique-Refine (비판-개선 루프)

한 에이전트가 생성하고, 다른 에이전트가 비판하는 반복 루프입니다. 코드 품질을 단계적으로 끌어올립니다.

```
[Generator Agent] → 코드 생성
    ↓
[Critic Agent] → 문제점 지적
    ↓
[Generator Agent] → 피드백 반영 수정
    ↓
[Critic Agent] → 재검토
    ↓ (품질 기준 충족 시)
[완료]
```

**실전 구현:**

```bash
# Round 1: 생성
claude "사용자 인증 모듈 구현" --output auth-module.ts

# Round 2: 비판 (다른 모델 사용 가능)
claude "auth-module.ts를 리뷰하고 보안 취약점과 개선점을 목록으로 정리" \
  --output review.md

# Round 3: 개선
claude "review.md의 피드백을 반영해서 auth-module.ts 수정"

# Round 4: 최종 검증
claude "수정된 auth-module.ts가 review.md의 모든 피드백을 반영했는지 확인"
```

**적합한 경우:** 보안 코드, 인프라 설정, 공개 API 설계
**주의점:** 무한 루프 방지를 위해 최대 반복 횟수 설정 필요 (보통 2~3회)

### 5. Swarm (자율 협업)

각 에이전트가 독립적으로 작업하면서, 공유 상태를 통해 협업합니다. 가장 고급 패턴입니다.

```
[Agent A] ←→ [공유 상태: TODO.md, PROGRESS.md]
[Agent B] ←→ [공유 상태: TODO.md, PROGRESS.md]  
[Agent C] ←→ [공유 상태: TODO.md, PROGRESS.md]
```

**실전 구현:**

```bash
# 공유 상태 파일 생성
cat > TODO.md << 'EOF'
- [ ] API 엔드포인트 구현 (assigned: none)
- [ ] 프론트엔드 폼 구현 (assigned: none)
- [ ] 데이터베이스 스키마 (assigned: none)
- [ ] E2E 테스트 (assigned: none, depends: API, Frontend)
EOF

# 각 에이전트가 TODO.md를 읽고, 할당되지 않은 작업을 선택
# → 작업 시작 시 assigned 업데이트
# → 완료 시 체크 표시
# → 의존성 있는 작업은 선행 작업 완료 후 시작
```

**적합한 경우:** 장기 프로젝트, 에이전트 수가 3개 이상일 때
**주의점:** 동시 수정으로 인한 충돌 관리가 핵심 → 파일 단위 락 또는 영역 분리

## 도구별 멀티 에이전트 지원 현황

| 도구 | 병렬 실행 | 서브에이전트 | 공유 상태 | 난이도 |
|------|-----------|------------|----------|--------|
| **Claude Code** | worktree + tmux | Task tool 지원 | 파일 시스템 | ⭐⭐ |
| **Cursor** | 다중 Composer | Agent Mode | .cursorrules | ⭐ |
| **Aider** | 다중 인스턴스 | 미지원 | git repo | ⭐⭐⭐ |
| **Devin** | 내장 병렬 | 내장 에이전트 | Devin workspace | ⭐ |
| **Gemini CLI** | 다중 세션 | 미지원 | 파일 시스템 | ⭐⭐⭐ |
| **Copilot Agent** | Workspace 내 | 코딩 에이전트 | VS Code | ⭐ |

## 실전 시나리오: 풀스택 기능 구현

새로운 "대시보드" 페이지를 구현한다고 가정합니다.

### Step 1: 작업 분해

```markdown
## 대시보드 구현 태스크
- Backend: GET /api/dashboard 엔드포인트
- Frontend: Dashboard 컴포넌트 + 차트
- Test: API 테스트 + 컴포넌트 테스트
- Docs: API 문서 업데이트
```

### Step 2: worktree 생성 및 에이전트 할당

```bash
# 메인 브랜치에서 분기
git worktree add ../dash-backend feature/dash-backend
git worktree add ../dash-frontend feature/dash-frontend

# 백엔드 에이전트 (터미널 1)
cd ../dash-backend
claude "GET /api/dashboard 엔드포인트 구현. 
  응답: { stats: {...}, charts: [...], recentActivity: [...] }
  미들웨어: auth, rateLimit 적용"

# 프론트엔드 에이전트 (터미널 2)  
cd ../dash-frontend
claude "Dashboard 페이지 컴포넌트 구현.
  API: GET /api/dashboard (mock 데이터로 시작)
  차트: recharts 사용, 반응형 레이아웃"
```

### Step 3: 통합 및 테스트

```bash
# 두 브랜치 merge
git checkout feature/dashboard
git merge feature/dash-backend feature/dash-frontend

# 통합 테스트 에이전트
claude "백엔드 API와 프론트엔드를 연동 테스트.
  mock 데이터를 실제 API 호출로 교체.
  E2E 테스트 작성."
```

### Step 4: 리뷰 에이전트

```bash
claude "전체 대시보드 구현을 리뷰:
  - API 보안 (인증, 입력 검증)
  - 프론트엔드 접근성
  - 에러 핸들링
  - 성능 (불필요한 리렌더링, N+1 쿼리)"
```

## 비용과 속도 트레이드오프

멀티 에이전트는 빠르지만 비용도 배수로 늘어납니다.

| 전략 | 속도 | 비용 | 권장 상황 |
|------|------|------|----------|
| 단일 에이전트 순차 | 1x | 1x | 간단한 기능, 학습 |
| 2-에이전트 병렬 | 1.6x | 1.8x | 프론트/백 분리 |
| 3-에이전트 병렬 | 2.2x | 2.5x | 풀스택 + 테스트 |
| Pipeline (4단계) | 0.8x | 3x | 품질 중심 |
| Critique-Refine (2회) | 0.6x | 2.2x | 보안/인프라 |

**비용 최적화 팁:**

- Fan-Out 작업에는 저비용 모델 사용 (Haiku, GPT-4o-mini)
- Critique에만 고급 모델 사용 (Opus, o3)
- 캐시 가능한 컨텍스트는 프롬프트 캐싱 활용
- 불필요한 파이프라인 단계 제거

## 실패 처리 전략

멀티 에이전트 시스템에서 가장 중요한 건 **하나가 실패했을 때 전체가 죽지 않는 것**입니다.

```bash
# 타임아웃 설정
timeout 300 claude "작업 내용" || echo "Agent timeout - 수동 확인 필요"

# 재시도 로직
for i in 1 2 3; do
  claude "작업 내용" && break
  echo "Retry $i/3..."
  sleep 10
done

# 결과 검증
claude "생성된 코드가 다음 기준을 충족하는지 검사:
  1. 빌드 성공
  2. 테스트 통과
  3. lint 에러 없음" || {
    echo "품질 기준 미달 - 수동 리뷰 필요"
    exit 1
}
```

## 체크리스트: 멀티 에이전트 도입 전 확인

- [ ] 작업이 독립적으로 분리 가능한가?
- [ ] 에이전트 간 공유 상태 관리 방안이 있는가?
- [ ] merge conflict 해결 전략이 있는가?
- [ ] 개별 에이전트 실패 시 롤백 방안이 있는가?
- [ ] 비용 한도를 설정했는가?
- [ ] 최종 통합 테스트 단계가 있는가?

## 다음 단계

- **가이드 11: Agent Teams** — 에이전트 팀 구성의 기초
- **가이드 15: Subagent Orchestration** — 서브에이전트 활용법
- **가이드 33: AI Delegation Patterns** — 위임 패턴 심화
- **워크플로우: AI Agent Pipeline** — 파이프라인 자동화 실전

---

> 에이전트 하나를 잘 쓰는 건 시작입니다. 여러 에이전트를 하나의 팀처럼 운영하는 게 진짜 10배 생산성의 시작이에요.
