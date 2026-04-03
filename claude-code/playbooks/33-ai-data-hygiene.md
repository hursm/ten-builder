# AI 코딩 데이터 위생 플레이북

> AI 코딩 에이전트를 쓸 때 민감 데이터가 유출되지 않도록 하는 6단계 워크플로우

## 왜 데이터 위생이 중요한가?

AI 코딩 에이전트는 파일 시스템에 접근하고, 터미널 명령을 실행하고, 코드 전체를 읽습니다. 이 과정에서 의도치 않게 **API 키, 데이터베이스 비밀번호, 고객 데이터**가 AI 서비스로 전송될 수 있습니다.

특히 2026년 4월 GitHub Copilot의 기본 데이터 수집 전환 이후, 개발자가 **자신의 데이터 위생 상태를 능동적으로 관리**해야 하는 시대가 됐습니다.

---

## 6단계 데이터 위생 워크플로우

### Step 1: 프로젝트 민감도 분류

프로젝트 시작 전에 민감도를 분류하세요:

```yaml
# .ai-classification.yaml (프로젝트 루트에 추가)
project:
  name: my-project
  sensitivity: high  # low | medium | high | critical
  
  contains:
    customer_data: true
    payment_info: false
    auth_secrets: true
    internal_api: true
    
  ai_tools:
    allowed:
      - claude-code  # API 모드
      - cursor       # Privacy Mode ON
    restricted:
      - copilot      # Business 플랜만
    blocked:
      - free-tier-tools
```

**분류 기준:**

| 민감도 | 기준 | AI 도구 사용 |
|--------|------|-------------|
| Low | 오픈소스, 학습용 | 제한 없음 |
| Medium | 개인 프로젝트, 사이드 프로젝트 | opt-out 권장 |
| High | 회사 코드, 클라이언트 프로젝트 | Business/API만 |
| Critical | 금융, 의료, 정부 | 자체 호스팅 또는 미사용 |

### Step 2: 제외 파일 설정

```bash
# 자동 생성 스크립트
cat > .cursorignore << 'EOF'
# === 시크릿 ===
.env
.env.*
*.pem
*.key
*.p12
*.pfx
secrets/
credentials/
.vault/

# === 프로덕션 설정 ===
config/production.*
config/staging.*
docker-compose.prod.yml
k8s/secrets/

# === 데이터 ===
*.sql
*.db
*.sqlite
*.dump
backups/
migrations/data/

# === 인증 ===
**/auth/tokens.*
**/oauth/config.*
EOF

# .copilotignore도 동일하게
cp .cursorignore .copilotignore
```

### Step 3: 시크릿 탐지 pre-commit 훅

```bash
#!/bin/bash
# .git/hooks/pre-commit — AI 세션 전 시크릿 유출 방지

set -e

echo "🔍 AI 데이터 위생 검사 중..."

# 1. 하드코딩된 시크릿 패턴
PATTERNS=(
  'password\s*[:=]\s*["\x27][^"\x27]{8,}'
  'api[_-]?key\s*[:=]\s*["\x27][^"\x27]{16,}'
  'secret\s*[:=]\s*["\x27][^"\x27]{8,}'
  'token\s*[:=]\s*["\x27][A-Za-z0-9_\-\.]{20,}'
  'AWS_ACCESS_KEY_ID\s*=\s*AK'
  'PRIVATE KEY-----'
)

FOUND=0
for pattern in "${PATTERNS[@]}"; do
  MATCHES=$(git diff --cached --diff-filter=ACM -U0 | \
    grep -E "$pattern" 2>/dev/null || true)
  if [ -n "$MATCHES" ]; then
    echo "⚠️  시크릿 패턴 발견: $pattern"
    echo "$MATCHES" | head -3
    FOUND=1
  fi
done

# 2. .env 파일이 커밋에 포함되는지
ENV_FILES=$(git diff --cached --name-only | grep -E '\.env($|\.)' || true)
if [ -n "$ENV_FILES" ]; then
  echo "⚠️  .env 파일이 커밋에 포함됨: $ENV_FILES"
  FOUND=1
fi

if [ $FOUND -eq 1 ]; then
  echo ""
  echo "❌ 데이터 위생 검사 실패. 시크릿을 제거하고 다시 시도하세요."
  echo "   건너뛰려면: git commit --no-verify"
  exit 1
fi

echo "✅ 데이터 위생 검사 통과"
```

### Step 4: AI 세션 시작 전 체크리스트

AI 코딩 에이전트를 시작하기 전에 확인:

```markdown
## AI 세션 시작 전 체크리스트

### 환경 확인
- [ ] AI 도구의 프라이버시 설정이 올바른가?
- [ ] 현재 프로젝트의 민감도 분류를 확인했는가?
- [ ] .cursorignore / .copilotignore가 최신 상태인가?

### 데이터 확인
- [ ] .env 파일이 .gitignore에 포함되어 있는가?
- [ ] 하드코딩된 시크릿이 없는가?
- [ ] 테스트 데이터에 실제 고객 정보가 포함되어 있지 않은가?

### 작업 범위
- [ ] AI에게 전달할 컨텍스트에 민감 정보가 없는가?
- [ ] 프롬프트에 실제 URL, IP, 계정 정보를 마스킹했는가?
```

### Step 5: 실시간 모니터링

Claude Code 세션에서 어떤 파일이 읽히는지 모니터링:

```bash
# Claude Code 사용 시 접근된 파일 로그 확인
# ~/.claude/projects/*/session*.jsonl에서 파일 접근 패턴 추출

# 최근 세션에서 접근된 파일 목록
find ~/.claude/projects -name "*.jsonl" -newer /tmp/session-start \
  -exec grep -l "Read\|Write\|Edit" {} \; 2>/dev/null | head -10

# 민감 파일 접근 알림 (Claude Code Hooks 활용)
# .claude/settings.json에 추가:
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Edit",
        "hook": "bash -c 'echo $TOOL_INPUT | jq -r .file_path | grep -E \"\\.(env|pem|key)$\" && echo \"⚠️ 민감 파일 접근\" >&2 && exit 1 || exit 0'"
      }
    ]
  }
}
```

### Step 6: 사후 감사

작업 완료 후 감사:

```bash
#!/bin/bash
# ai-session-audit.sh — AI 세션 후 데이터 위생 감사

echo "📋 AI 세션 후 감사 리포트"
echo "========================"
echo "시간: $(date)"
echo ""

# git diff에서 새로 추가된 시크릿 확인
echo "🔑 새로 추가된 의심 패턴:"
git diff HEAD --diff-filter=A -U0 | \
  grep -E "(password|secret|key|token)" | head -10

# 큰 파일 추가 확인 (AI가 불필요한 파일 생성할 수 있음)
echo ""
echo "📦 큰 파일 변경:"
git diff --stat HEAD | sort -t'|' -k2 -rn | head -5

# .gitignore 변경 확인
echo ""
if git diff HEAD --name-only | grep -q ".gitignore"; then
  echo "⚠️  .gitignore가 변경됨 — 의도적인지 확인하세요"
else
  echo "✅ .gitignore 변경 없음"
fi
```

---

## CLAUDE.md에 추가할 데이터 위생 규칙

```markdown
## Data Hygiene Rules

### Forbidden
- 하드코딩된 시크릿 절대 금지
- .env 파일 직접 수정 금지
- 실제 고객 데이터를 테스트 코드에 사용 금지
- 프로덕션 DB URL을 코드에 포함 금지

### Required
- 환경변수는 반드시 process.env에서 읽기
- 테스트 데이터는 faker 라이브러리로 생성
- 민감 설정은 config/secrets/ 디렉토리에 (gitignored)
- 새 시크릿 추가 시 .env.example도 업데이트
```

---

## 팀에서 사용할 때

### Slack/Teams 리마인더 템플릿

```
🔐 AI 코딩 도구 프라이버시 리마인더

4월 24일부터 GitHub Copilot이 기본적으로 데이터를 수집합니다.

✅ 지금 확인하세요:
1. github.com/settings/copilot → 데이터 학습 Disabled
2. Cursor → Privacy Mode ON
3. Claude → "Improve Claude" 토글 OFF

5분이면 끝납니다. 회사 코드를 보호합시다.
```

---

## 핵심 요약

```
데이터 위생 = 예방이 치료보다 100배 쉽다

1. 분류 — 프로젝트 민감도를 먼저 정하기
2. 차단 — .cursorignore + .copilotignore로 민감 파일 제외
3. 탐지 — pre-commit 훅으로 시크릿 유출 자동 차단
4. 체크 — 세션 시작 전 체크리스트 확인
5. 감시 — Hooks로 민감 파일 접근 실시간 모니터링
6. 감사 — 작업 후 변경사항 감사
```

---

**관련 자료:**
- [AI 코딩 도구 데이터 프라이버시 가이드](../../guides/45-ai-coding-data-privacy.md)
- [AI 코딩 프라이버시 설정 치트시트](../../cheatsheets/ai-coding-privacy-settings-cheatsheet.md)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
