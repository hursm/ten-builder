# AI 코딩 도구 프라이버시 설정 치트시트

> 도구별 데이터 수집 끄는 법 한 장 정리 — 2026년 4월 기준

---

## ⚡ 3분 안에 끝내는 설정

### GitHub Copilot (🔴 4/24까지 필수)

```
github.com → Settings → Copilot → Copilot features
→ Privacy: "Allow GitHub to use my data for AI model training"
→ ❌ Disabled
```

| 수집 데이터 | 범위 |
|------------|------|
| 입력/출력, 승인 코드 | 모든 Copilot 세션 |
| 파일명, 레포 구조 | 작업 중인 프로젝트 |
| 탐색 패턴, 피드백 | IDE 내 행동 |
| 프라이빗 레포 (at rest) | ❌ 미수집 |
| 프라이빗 레포 (세션 중) | ⚠️ 수집 가능 |

**면제**: Business, Enterprise, 학생, 교사 계정

---

### Claude Code / Claude

```
claude.ai → Settings → Privacy
→ "Improve Claude for everyone"
→ ❌ 토글 OFF
```

| 플랜 | 기본 수집 | opt-out 후 보관 |
|------|----------|----------------|
| Free/Pro/Max | 사용자 선택 | 30일 후 삭제 |
| API | ❌ 미수집 | 7일 후 삭제 |
| Enterprise | ❌ 미수집 | ZDR 가능 |

---

### Cursor

```
Cursor → Settings → Privacy
→ Privacy Mode: ✅ ON
```

| Privacy Mode | 데이터 저장 | 학습 사용 | 모델 제공업체 보관 |
|-------------|-----------|----------|-----------------|
| ✅ ON | ❌ 없음 | ❌ 없음 | Zero Retention |
| ❌ OFF | ⚠️ 있음 | ⚠️ 가능 | 제공업체 정책 따름 |

**Team 플랜**: 기본 강제 ON

---

### Windsurf (Codeium)

```
Windsurf → Settings → Privacy
→ Telemetry level: minimal 또는 off
```

- Enterprise: 자체 호스팅 옵션으로 완전 격리 가능
- 코드 컨텍스트는 추론에만 사용, 학습에는 미사용 (공식 정책)

---

### Gemini CLI / Google AI Studio

```
AI Studio → Settings → Data Usage
→ "Help improve Google's products" → ❌ OFF
```

- API 사용 시: 유료 API는 학습에 미사용
- 무료 Tier: 학습에 사용될 수 있음 (opt-out 가능)

---

## 파일 제외 설정 모음

### `.cursorignore`

```plaintext
# 시크릿과 키
.env
.env.*
*.pem
*.key
*.p12
secrets/
credentials/

# 설정
config/production.*
config/secrets.*
docker-compose.prod.yml

# 데이터
*.sql
*.db
*.sqlite
backups/
```

### `.copilotignore`

```plaintext
# GitHub Copilot에서 제외할 파일
.env
*.pem
*.key
config/production.yaml
secrets/**
```

### `.aiignore` (범용)

```plaintext
# 대부분의 AI 도구가 인식하는 범용 형식
**/.env
**/*.key
**/*.pem
**/secrets/**
**/config/production.*
```

---

## 도구별 데이터 흐름 다이어그램

```
사용자 코드 → [AI 도구] → 어디로?

GitHub Copilot (opt-out 안 하면):
  코드 → GitHub 서버 → MS 계열사 → 모델 학습 데이터
  
Claude Code (opt-out 하면):
  코드 → Anthropic 서버 → 30일 보관 → 삭제 (학습 미사용)

Cursor (Privacy Mode ON):
  코드 → Cursor 서버 (일시 캐시) → 즉시 삭제 → 모델 제공업체 ZDR
```

---

## 빠른 점검 스크립트

```bash
#!/bin/bash
# ai-privacy-check.sh — AI 코딩 도구 프라이버시 점검

echo "🔍 AI 코딩 도구 프라이버시 빠른 점검"
echo "=================================="

# .env 파일 확인
if [ -f .env ]; then
  if [ -f .gitignore ] && grep -q ".env" .gitignore; then
    echo "✅ .env가 .gitignore에 포함됨"
  else
    echo "⚠️  .env가 .gitignore에 없음 — AI 도구가 읽을 수 있음"
  fi
fi

# 시크릿 패턴 탐지
echo ""
echo "🔑 시크릿 패턴 검사..."
SECRETS=$(grep -rn --include="*.ts" --include="*.js" --include="*.py" \
  -E '(password|secret|api_key|token)\s*=\s*["\x27][^"\x27]{8,}' . 2>/dev/null | \
  grep -v node_modules | grep -v .git | head -5)

if [ -n "$SECRETS" ]; then
  echo "⚠️  하드코딩된 시크릿 발견:"
  echo "$SECRETS"
else
  echo "✅ 하드코딩된 시크릿 없음"
fi

# AI 제외 파일 확인
echo ""
echo "📄 AI 제외 파일 확인..."
for f in .cursorignore .copilotignore .aiignore; do
  if [ -f "$f" ]; then
    echo "✅ $f 존재"
  else
    echo "💡 $f 없음 — 생성을 권장합니다"
  fi
done

echo ""
echo "🔗 설정 링크:"
echo "  Copilot: github.com/settings/copilot"
echo "  Claude:  claude.ai → Settings → Privacy"
echo "  Cursor:  Settings → Privacy → Privacy Mode"
```

---

## 핵심 체크리스트

```
□ GitHub Copilot: 데이터 학습 Disabled (4/24 마감!)
□ Claude Code: "Improve Claude" 토글 OFF
□ Cursor: Privacy Mode ON
□ 프로젝트에 .cursorignore / .copilotignore 추가
□ .env가 .gitignore에 포함되어 있는지 확인
□ 회사 코드는 Business/Enterprise/API 플랜만 사용
□ 팀원에게 이 치트시트 공유 완료
```

---

**관련 가이드:** [AI 코딩 도구 데이터 프라이버시 실전 가이드](../guides/45-ai-coding-data-privacy.md)

**뉴스레터:** [maily.so/tenbuilder](https://maily.so/tenbuilder)
