# Claude Code 실전 가이드

> 직접 써보고 검증한 Claude Code 설정과 워크플로 — 2억+ 유저 서비스 경험 기반

## 이 폴더에 있는 것

| 파일/폴더 | 설명 |
|-----------|------|
| `CLAUDE.md.template` | 프로젝트에 바로 적용할 수 있는 CLAUDE.md 템플릿 |
| `agents.md.template` | 멀티 에이전트 설정 템플릿 |
| `cursorrules.template` | Cursor용 설정 (Claude Code와 병행 시) |
| `playbooks/` | 상황별 실전 플레이북 |
| `examples/` | 프로젝트 유형별 적용 예시 |

## Quick Start

```bash
# 1. 템플릿 복사
cp CLAUDE.md.template /your/project/CLAUDE.md

# 2. 프로젝트에 맞게 수정
# - [PROJECT_NAME] → 실제 프로젝트명
# - [TECH_STACK] → 사용 기술
# - 불필요한 섹션 삭제

# 3. Claude Code 실행
claude
```

## "CLAUDE.md 필요 없다" vs "필수다" — 내 입장

CLAUDE.md 논쟁의 핵심은 **크기**가 아니라 **품질**입니다:

- ❌ 500줄짜리 장황한 CLAUDE.md → Claude가 무시하는 노이즈
- ❌ CLAUDE.md 없음 → 매 세션마다 같은 실수 반복
- ✅ **50-100줄의 정확한 CLAUDE.md** → 일관된 코드 품질

이 템플릿은 글로벌 IT 회사 6년 + 현업 CTO 경험에서 검증한 구조입니다.

## 플레이북

1. [프로젝트 초기 설정](./playbooks/01-project-setup.md) — CLAUDE.md부터 첫 커밋까지
2. [AI 코드 리뷰](./playbooks/02-code-review.md) — PR 리뷰를 Claude에게 맡기기
3. [리팩토링 워크플로](./playbooks/03-refactoring.md) — 기존 코드를 안전하게 개선
4. [AI 디버깅 워크플로](./playbooks/04-debugging.md) — 에러를 체계적으로 해결
5. [MCP 서버 활용](./playbooks/05-mcp-tools.md) — 외부 도구 연결로 능력 확장
