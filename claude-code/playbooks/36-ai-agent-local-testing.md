# AI 에이전트 로컬 테스트 환경 구축 플레이북

> Docker + 샌드박스로 AI 에이전트를 안전하게 로컬 테스트하는 단계별 가이드

## 왜 로컬 테스트 환경이 필요한가?

AI 코딩 에이전트는 파일 시스템 쓰기, 명령어 실행, 네트워크 요청 등 강력한 권한을 가진다. 프로덕션 코드나 시스템에 영향을 주지 않으면서 에이전트의 동작을 검증하려면 **격리된 테스트 환경**이 필수다.

| 위험 | 예시 | 방지책 |
|------|------|--------|
| 파일 삭제/덮어쓰기 | `rm -rf /` 실행 | 컨테이너 격리 |
| 시크릿 유출 | `.env` 내용 출력 | 환경 변수 분리 |
| 네트워크 공격 | 외부 서버에 데이터 전송 | 네트워크 격리 |
| 무한 루프 | CPU/메모리 폭주 | 리소스 제한 |
| 의존성 오염 | 글로벌 패키지 변경 | 임시 볼륨 |

## 6단계 워크플로우

### Step 1: Docker 기반 샌드박스 생성

```dockerfile
# Dockerfile.ai-sandbox
FROM node:20-slim

# 기본 개발 도구 설치
RUN apt-get update && apt-get install -y \
    git curl jq python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 비루트 사용자 생성
RUN useradd -m -s /bin/bash sandbox
USER sandbox
WORKDIR /home/sandbox/workspace

# AI 에이전트가 수정할 프로젝트는 볼륨으로 마운트
VOLUME ["/home/sandbox/workspace"]
```

**빌드 & 실행:**

```bash
# 이미지 빌드
docker build -f Dockerfile.ai-sandbox -t ai-sandbox .

# 테스트 환경 실행 (리소스 제한 포함)
docker run -it --rm \
  --name ai-test \
  --memory=4g \
  --cpus=2 \
  --network=none \
  -v "$(pwd)/test-project:/home/sandbox/workspace" \
  ai-sandbox bash
```

**핵심 플래그 설명:**

| 플래그 | 역할 |
|--------|------|
| `--rm` | 종료 시 컨테이너 자동 삭제 |
| `--memory=4g` | 메모리 4GB 제한 |
| `--cpus=2` | CPU 2코어 제한 |
| `--network=none` | 네트워크 완전 차단 |
| `-v` | 테스트 프로젝트만 마운트 |

### Step 2: 프로젝트 스냅샷 관리

테스트 전 원본 코드를 스냅샷으로 저장하고, 테스트 후 복원한다.

```bash
#!/bin/bash
# scripts/sandbox-test.sh

PROJECT_DIR="$1"
SNAPSHOT_DIR="/tmp/ai-test-snapshot-$(date +%s)"

# 1. 스냅샷 생성
echo "📸 스냅샷 생성: $SNAPSHOT_DIR"
cp -r "$PROJECT_DIR" "$SNAPSHOT_DIR"

# 2. 테스트용 복사본 생성
TEST_DIR="/tmp/ai-test-workspace"
rm -rf "$TEST_DIR"
cp -r "$PROJECT_DIR" "$TEST_DIR"

# 3. Docker에서 AI 에이전트 실행
docker run --rm \
  --memory=4g --cpus=2 \
  --network=none \
  -v "$TEST_DIR:/home/sandbox/workspace" \
  ai-sandbox bash -c "
    cd /home/sandbox/workspace
    # 여기서 AI 에이전트 실행
    echo '에이전트 작업 완료'
  "

# 4. diff 확인
echo "📋 변경 사항:"
diff -rq "$SNAPSHOT_DIR" "$TEST_DIR" | head -20

# 5. 승인 후 반영 or 폐기
read -p "변경사항 적용? (y/N) " confirm
if [ "$confirm" = "y" ]; then
  cp -r "$TEST_DIR/"* "$PROJECT_DIR/"
  echo "✅ 적용 완료"
else
  echo "❌ 폐기"
fi

# 6. 정리
rm -rf "$SNAPSHOT_DIR" "$TEST_DIR"
```

### Step 3: 환경 변수 안전 관리

```bash
# .env.sandbox — 테스트용 더미 환경 변수
DATABASE_URL=postgresql://test:test@localhost:5432/testdb
API_KEY=sk-test-dummy-key-not-real
OPENAI_API_KEY=sk-test-dummy-key-not-real
AWS_ACCESS_KEY_ID=AKIATESTDUMMYKEY
AWS_SECRET_ACCESS_KEY=testdummysecretkey

# 절대 실제 키를 넣지 않는다
```

**Docker Compose로 환경 분리:**

```yaml
# docker-compose.sandbox.yml
services:
  sandbox:
    build:
      dockerfile: Dockerfile.ai-sandbox
    environment:
      - NODE_ENV=test
    env_file:
      - .env.sandbox
    volumes:
      - ./test-project:/home/sandbox/workspace
    mem_limit: 4g
    cpus: 2
    networks:
      - sandbox-net

  # 테스트용 DB (격리)
  test-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    networks:
      - sandbox-net

networks:
  sandbox-net:
    internal: true  # 외부 인터넷 차단
```

### Step 4: Claude Code 샌드박스 모드 활용

Claude Code의 내장 샌드박스 기능을 활용한다.

```bash
# Claude Code 안전 모드 실행
# 1. 허용 명령어 제한
claude --allowedTools "Edit,Read,Bash(git diff),Bash(npm test)"

# 2. 읽기 전용 모드로 분석만
claude --allowedTools "Read,Bash(find),Bash(grep),Bash(wc)"

# 3. Git worktree로 격리된 브랜치 작업
git worktree add ../test-workspace -b ai-test
cd ../test-workspace
claude  # 여기서 자유롭게 작업
# 결과 확인 후 머지 or 삭제
git worktree remove ../test-workspace
```

### Step 5: 자동 검증 파이프라인

에이전트 작업 후 자동으로 품질을 검증한다.

```bash
#!/bin/bash
# scripts/validate-agent-output.sh

echo "🔍 AI 에이전트 출력 검증 시작"
ERRORS=0

# 1. 타입 체크
echo "  📝 TypeScript 타입 체크..."
npx tsc --noEmit 2>/dev/null
if [ $? -ne 0 ]; then
  echo "  ❌ 타입 오류 발견"
  ERRORS=$((ERRORS + 1))
fi

# 2. 린트
echo "  🧹 ESLint 검사..."
npx eslint . --quiet 2>/dev/null
if [ $? -ne 0 ]; then
  echo "  ❌ 린트 오류 발견"
  ERRORS=$((ERRORS + 1))
fi

# 3. 테스트
echo "  🧪 테스트 실행..."
npm test -- --watchAll=false 2>/dev/null
if [ $? -ne 0 ]; then
  echo "  ❌ 테스트 실패"
  ERRORS=$((ERRORS + 1))
fi

# 4. 시크릿 스캔
echo "  🔐 시크릿 스캔..."
grep -rn "sk-\|AKIA\|password=" --include="*.ts" --include="*.js" . 2>/dev/null
if [ $? -eq 0 ]; then
  echo "  ⚠️  시크릿 유출 가능성"
  ERRORS=$((ERRORS + 1))
fi

# 5. 파일 크기 체크
echo "  📏 비정상 파일 체크..."
find . -name "*.ts" -size +100k -not -path "*/node_modules/*" | while read f; do
  echo "  ⚠️  큰 파일: $f ($(du -h "$f" | cut -f1))"
  ERRORS=$((ERRORS + 1))
done

# 결과
echo ""
if [ $ERRORS -eq 0 ]; then
  echo "✅ 모든 검증 통과"
else
  echo "❌ $ERRORS개 문제 발견 — 리뷰 필요"
fi
exit $ERRORS
```

### Step 6: 테스트 결과 기록 & 반복

```bash
# scripts/log-test-run.sh
LOG_DIR="$HOME/.ai-test-logs"
mkdir -p "$LOG_DIR"

LOG_FILE="$LOG_DIR/$(date +%Y%m%d_%H%M%S).json"

cat > "$LOG_FILE" << EOF
{
  "timestamp": "$(date -Iseconds)",
  "agent": "${AGENT_NAME:-claude-code}",
  "model": "${MODEL:-sonnet}",
  "task": "$1",
  "files_changed": $(git diff --stat | tail -1 | grep -oE '[0-9]+' | head -1 || echo 0),
  "tests_passed": $(npm test -- --watchAll=false 2>&1 | grep -oE '[0-9]+ passed' | grep -oE '[0-9]+' || echo 0),
  "type_errors": $(npx tsc --noEmit 2>&1 | grep -c "error TS" || echo 0),
  "validation_passed": ${VALIDATION_RESULT:-false}
}
EOF

echo "📝 로그 저장: $LOG_FILE"
```

## 환경별 테스트 매트릭스

| 테스트 레벨 | 격리 수준 | 네트워크 | 용도 |
|------------|----------|---------|------|
| **Level 0** | Git worktree | 허용 | 빠른 반복 테스트 |
| **Level 1** | Docker (bridge) | 제한적 허용 | 의존성 설치 필요 시 |
| **Level 2** | Docker (none) | 차단 | 보안 민감 코드 |
| **Level 3** | VM (Firecracker) | 완전 격리 | 프로덕션급 검증 |

## 체크리스트

### 테스트 전
- [ ] 프로젝트 스냅샷 생성됨
- [ ] 더미 환경 변수 설정됨 (`.env.sandbox`)
- [ ] Docker 컨테이너 리소스 제한 설정됨
- [ ] 네트워크 격리 레벨 결정됨
- [ ] 테스트 DB/서비스 준비됨

### 테스트 후
- [ ] `diff`로 변경 사항 리뷰 완료
- [ ] 타입 체크 통과
- [ ] 테스트 스위트 통과
- [ ] 시크릿 스캔 통과
- [ ] 불필요한 파일 변경 없음 확인
- [ ] 테스트 로그 저장됨

## 안티패턴 🚫

| 안티패턴 | 위험 | 대안 |
|---------|------|------|
| 프로덕션 환경에서 직접 테스트 | 데이터 손실, 장애 | 반드시 격리 환경 사용 |
| 실제 API 키를 샌드박스에 넣기 | 키 유출, 과금 | 더미 키 또는 mock |
| 리소스 제한 없이 실행 | 시스템 행업 | `--memory`, `--cpus` 설정 |
| 변경사항 리뷰 없이 적용 | 버그 코드 유입 | `diff` 확인 후 수동 승인 |
| 테스트 로그 미보관 | 재현 불가 | 매 실행 결과 기록 |

---

*이 플레이북은 텐빌더의 AI 코딩 도구 실전 시리즈의 일부입니다.*
*에이전트 샌드박싱 심화 → [AI 코딩 에이전트 샌드박싱 실전 가이드](../../guides/27-ai-agent-sandboxing.md)*
*자율 실행 설계 → [AI 에이전트 자율 실행 설계 플레이북](22-autonomous-execution.md)*
