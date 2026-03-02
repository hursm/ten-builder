# 텐빌더

> AI로 10배 빠르게 빌드하는 방법을 알려드려요.

[![Newsletter](https://img.shields.io/badge/뉴스레터-구독-blue)](https://maily.so/tenbuilder)
[![YouTube](https://img.shields.io/badge/YouTube-텐빌더-red)](https://youtube.com/@ten-builder)

---

실무에서 바로 쓸 수 있는 AI 활용법을 다루고 있어요.

글로벌 IT 회사에서 6년간 2억+ 유저 서비스를 담당했던 엔지니어가,
Claude Code, Gemini 등 AI를 실무에서 직접 써보고 검증한 내용을 공유합니다.

- **직접 써보고 검증한 AI 리뷰**
- **AI를 10배 더 활용할 수 있는 실전 노하우**
- **2억명+ 서비스를 다뤄본 경험의 실전 노하우**

---

## 무엇이 있나요?

| 폴더 | 내용 | 난이도 |
|------|------|--------|
| [`/guides`](./guides) | 01~10 단계별 실전 가이드 | ⭐⭐⭐ |
| [`/templates`](./templates) | 복사해서 바로 쓰는 설정 파일 | ⭐ |
| [`/examples`](./examples) | 프로젝트별 CLAUDE.md 예시 | ⭐⭐ |
| [`/cheatsheets`](./cheatsheets) | 원페이저 치트시트 | ⭐ |

## Quick Start

**1분 안에 Claude Code 프로젝트 설정:**

```bash
# CLAUDE.md 템플릿 복사
curl -O https://raw.githubusercontent.com/ten-builder/ten-builder/main/templates/CLAUDE.md.template

# 프로젝트 루트에 배치
mv CLAUDE.md.template CLAUDE.md

# 프로젝트에 맞게 수정 후 사용
```

**AI 코딩 환경 한 번에 세팅:**

```bash
# macOS 원클릭 설정
curl -sSL https://raw.githubusercontent.com/ten-builder/ten-builder/main/templates/macos-setup.sh | bash
```

## 가이드 목차

| # | 가이드 | 설명 |
|---|--------|------|
| 01 | [환경 세팅](./guides/01-environment-setup.md) | AI 코딩 도구 설치 & 설정 |
| 02 | [프로젝트 초기 설정](./guides/02-project-setup.md) | CLAUDE.md부터 첫 커밋까지 |
| 03 | [일일 코딩 루틴](./guides/03-daily-workflow.md) | 매일 AI와 코딩하는 워크플로 |
| 04 | [코드 리뷰](./guides/04-code-review.md) | AI 코드 리뷰 & PR 워크플로 |
| 05 | [디버깅](./guides/05-debugging.md) | AI와 체계적으로 버그 잡기 |
| 06 | [리팩토링](./guides/06-refactoring.md) | AI와 안전하게 코드 개선 |
| 07 | [TDD](./guides/07-tdd.md) | AI와 테스트 주도 개발 |
| 08 | [MCP 도구](./guides/08-mcp-tools.md) | 외부 도구 연결 (DB, GitHub 등) |
| 09 | [보안](./guides/09-security.md) | AI 코딩 도구 보안 설정 |
| 10 | [Hooks](./guides/10-hooks.md) | 자동 검사/포맷/알림 설정 |

## 이 레포는 어떻게 업데이트 되나요?

- **매주** — 새로운 가이드와 패턴 추가
- **Release** — ⭐ Star 누르면 새 콘텐츠 추가 시 알림

## 더 알아보기

이 레포가 도움이 됐다면, 매주 보내는 AI 코딩 인사이트도 좋아할 거예요:

- 직접 써보고 검증한 AI 도구 리뷰
- 실무에서 바로 적용 가능한 설정과 워크플로
- 실패 사례와 트레이드오프

**뉴스레터 구독:** [maily.so/tenbuilder](https://maily.so/tenbuilder)

## License

MIT
