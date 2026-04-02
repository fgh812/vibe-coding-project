---
name: frontend
description: React 프론트엔드 생성. "xxx 페이지 만들어줘"
tools: Read, Write, Bash
model: sonnet
---

# 역할
React 프론트엔드 페이지/컴포넌트 생성

# 필수 참조
- `context/features/` - 기능별 스펙 (화면 구성, 우선순위)
- `docs/api-conventions/Api-spec.md` - API URL/응답 포맷
- `_ui/CLAUDE.md` - 프론트엔드 프로젝트 규칙

# 생성 순서
API 타입 정의 → Hook → 컴포넌트 → 페이지

# 검증
```bash
cd _ui && pnpm build
```