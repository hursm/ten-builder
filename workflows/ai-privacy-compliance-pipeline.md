# AI 코딩 프라이버시 컴플라이언스 파이프라인

> PR에서 민감 데이터 유출을 자동으로 탐지하고 AI 도구 설정을 팀 전체에 강제하는 CI/CD 워크플로우

## 이 워크플로우가 필요한 이유

AI 코딩 에이전트가 팀 전체에 보급되면서, **한 사람의 실수로 시크릿이 AI 학습 데이터에 포함**될 수 있습니다. 개인의 주의력에 의존하는 대신 **CI 파이프라인으로 자동 검증**하는 것이 확실합니다.

---

## 파이프라인 구조

```
PR 생성
  ↓
[Stage 1] 시크릿 스캔 ← gitleaks / trufflehog
  ↓
[Stage 2] AI 제외 파일 검증 ← .cursorignore, .copilotignore 존재 체크
  ↓
[Stage 3] 민감 패턴 탐지 ← 정규식 기반 커스텀 룰
  ↓
[Stage 4] AI 생성 코드 마킹 ← Co-authored-by 태그 확인
  ↓
✅ 통과 → 리뷰 가능  |  ❌ 실패 → 자동 코멘트 + 블록
```

---

## GitHub Actions 구현

### `.github/workflows/ai-privacy-check.yml`

```yaml
name: AI Privacy Compliance Check

on:
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  pull-requests: write

jobs:
  privacy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Stage 1: 시크릿 스캔
      - name: Secret Scan with gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      # Stage 2: AI 제외 파일 검증
      - name: Check AI ignore files
        run: |
          echo "🔍 AI 제외 파일 검증..."
          
          MISSING=()
          
          # 필수 제외 파일 목록
          REQUIRED_FILES=(".cursorignore" ".copilotignore")
          
          for f in "${REQUIRED_FILES[@]}"; do
            if [ ! -f "$f" ]; then
              MISSING+=("$f")
            fi
          done
          
          if [ ${#MISSING[@]} -gt 0 ]; then
            echo "⚠️  누락된 AI 제외 파일:"
            printf '  - %s\n' "${MISSING[@]}"
            echo ""
            echo "💡 프로젝트에 .cursorignore와 .copilotignore를 추가하세요."
            echo "   참고: cheatsheets/ai-coding-privacy-settings-cheatsheet.md"
          else
            echo "✅ 모든 AI 제외 파일 존재"
          fi
          
          # .env가 .gitignore에 포함되어 있는지
          if [ -f .gitignore ] && grep -q "^\.env" .gitignore; then
            echo "✅ .env가 .gitignore에 포함됨"
          else
            echo "⚠️  .env가 .gitignore에 없음"
          fi

      # Stage 3: 민감 패턴 탐지
      - name: Detect sensitive patterns
        run: |
          echo "🔍 민감 패턴 탐지..."
          
          # 이번 PR에서 변경된 파일만 검사
          CHANGED_FILES=$(git diff --name-only origin/main...HEAD)
          FOUND=0
          
          PATTERNS=(
            'password\s*[:=]\s*"[^"]{8,}"'
            'api[_-]?key\s*[:=]\s*"[^"]{16,}"'
            'secret\s*[:=]\s*"[^"]{8,}"'
            'AWS_ACCESS_KEY_ID\s*=\s*AK'
            'PRIVATE KEY-----'
            'mongodb\+srv://[^:]+:[^@]+@'
            'postgresql://[^:]+:[^@]+@'
            'redis://:[^@]+@'
          )
          
          for file in $CHANGED_FILES; do
            [ -f "$file" ] || continue
            # 바이너리 파일 스킵
            file -b --mime-type "$file" | grep -q "text/" || continue
            
            for pattern in "${PATTERNS[@]}"; do
              MATCHES=$(grep -nE "$pattern" "$file" 2>/dev/null || true)
              if [ -n "$MATCHES" ]; then
                echo "⚠️  $file:"
                echo "$MATCHES" | head -3
                FOUND=1
              fi
            done
          done
          
          if [ $FOUND -eq 1 ]; then
            echo ""
            echo "❌ 민감 패턴이 발견됐습니다. 환경변수로 분리하세요."
            exit 1
          fi
          
          echo "✅ 민감 패턴 미발견"

      # Stage 4: AI 생성 코드 추적
      - name: Track AI-generated code
        run: |
          echo "📊 AI 생성 코드 추적..."
          
          # AI 관련 커밋 태그 확인
          AI_COMMITS=$(git log origin/main...HEAD --format="%H %s" | \
            grep -iE "(ai|copilot|claude|cursor|generated)" || true)
          
          if [ -n "$AI_COMMITS" ]; then
            echo "ℹ️  AI 관련 커밋 발견:"
            echo "$AI_COMMITS" | head -10
            echo ""
            echo "💡 AI 생성 코드는 추가 리뷰가 권장됩니다."
          else
            echo "ℹ️  AI 관련 커밋 태그 없음"
          fi

      # 결과 PR 코멘트
      - name: Comment results
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🔐 AI Privacy Compliance Check 실패
              
            PR에서 잠재적 프라이버시 문제가 발견됐습니다.
            
            ### 체크리스트
            - [ ] 하드코딩된 시크릿을 환경변수로 분리
            - [ ] .cursorignore / .copilotignore 파일 추가
            - [ ] .env가 .gitignore에 포함되어 있는지 확인
            
            자세한 내용은 Actions 로그를 확인하세요.`
            })
```

---

## 커스텀 룰 설정

### `gitleaks.toml` — 추가 룰

```toml
title = "AI Privacy Custom Rules"

# 기본 룰에 추가
[[rules]]
  id = "hardcoded-db-url"
  description = "하드코딩된 데이터베이스 URL"
  regex = '''(postgres|mysql|mongodb|redis)://[^:]+:[^@]{8,}@[^\s'"]+'''
  tags = ["secret", "database"]

[[rules]]
  id = "jwt-secret"
  description = "하드코딩된 JWT 시크릿"
  regex = '''JWT_SECRET\s*[:=]\s*['"][^'"]{16,}['"]'''
  tags = ["secret", "auth"]

[[rules]]
  id = "private-ip-exposure"
  description = "프라이빗 IP 주소 노출"
  regex = '''(10\.\d{1,3}\.\d{1,3}\.\d{1,3}|172\.(1[6-9]|2[0-9]|3[01])\.\d{1,3}\.\d{1,3}|192\.168\.\d{1,3}\.\d{1,3})'''
  tags = ["network", "internal"]

# 허용 목록 (false positive 방지)
[allowlist]
  paths = [
    '''test/.*''',
    '''\.example$''',
    '''\.md$'''
  ]
```

---

## 팀 설정 자동 동기화

### AI 도구 설정을 레포에 포함

```bash
# 프로젝트 설정 자동 적용 스크립트
# scripts/setup-ai-tools.sh

#!/bin/bash
echo "🔧 AI 코딩 도구 설정 적용..."

# 1. Git hooks 설치
if [ -d .git ]; then
  cp hooks/pre-commit .git/hooks/pre-commit
  chmod +x .git/hooks/pre-commit
  echo "✅ pre-commit 훅 설치"
fi

# 2. AI 제외 파일 검증
for f in .cursorignore .copilotignore; do
  if [ ! -f "$f" ]; then
    echo "⚠️  $f 없음 — 템플릿에서 복사합니다"
    cp templates/ai-ignore-template "$f"
  fi
done

# 3. 안내 메시지
echo ""
echo "📋 AI 도구 프라이버시 설정을 확인하세요:"
echo "  Copilot: github.com/settings/copilot"
echo "  Claude:  claude.ai → Settings → Privacy"
echo "  Cursor:  Settings → Privacy → Privacy Mode"
```

### `package.json`에 postinstall로 연결

```json
{
  "scripts": {
    "postinstall": "bash scripts/setup-ai-tools.sh",
    "privacy:check": "bash scripts/privacy-check.sh",
    "privacy:audit": "bash scripts/ai-session-audit.sh"
  }
}
```

---

## Slack/Discord 알림 연동

```yaml
# GitHub Actions에 추가
- name: Notify on privacy violation
  if: failure()
  run: |
    curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
      -H 'Content-Type: application/json' \
      -d '{
        "text": "🔐 AI Privacy Check 실패",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*PR #${{ github.event.pull_request.number }}*에서 프라이버시 문제 발견\n<${{ github.event.pull_request.html_url }}|PR 확인하기>"
            }
          }
        ]
      }'
```

---

## 핵심 요약

| 단계 | 자동화 도구 | 목적 |
|------|-----------|------|
| 시크릿 스캔 | gitleaks / trufflehog | 하드코딩된 시크릿 탐지 |
| 제외 파일 검증 | 커스텀 스크립트 | .cursorignore 등 존재 확인 |
| 민감 패턴 탐지 | 정규식 매칭 | DB URL, IP 등 패턴 탐지 |
| AI 코드 추적 | 커밋 메시지 분석 | AI 생성 코드 리뷰 강화 |
| 알림 | Slack/Discord | 위반 시 즉시 팀 알림 |

```
핵심: 사람의 주의력에 의존하지 말고, CI 파이프라인으로 자동 검증하자.
한 번 설정하면 모든 PR에서 자동으로 작동한다.
```

---

**관련 자료:**
- [AI 코딩 도구 데이터 프라이버시 가이드](../guides/45-ai-coding-data-privacy.md)
- [AI 코딩 프라이버시 설정 치트시트](../cheatsheets/ai-coding-privacy-settings-cheatsheet.md)
- [AI 코딩 데이터 위생 플레이북](../claude-code/playbooks/33-ai-data-hygiene.md)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
