# Project: Vibe Coding Project

각 프로젝트들의 스펙을 참조해서 바이브 코딩을 진행하는 프로젝트

## Directory Structure

심볼릭 링크로 두 개의 독립된 프로젝트를 연결.
- `ui/`: React 프로젝트 (package.json)
- `api/`: Spring Boot 프로젝트 (build.gradle)

> 파일 경로 참조 시 `ui/` 또는 `api/` 접두사 필수 확인

## Context Routing

| 작업 유형 | 참조 파일 |
|----------|----------|
| 프론트엔드 | `ui/CLAUDE.md` |
| 백엔드 | `api/CLAUDE.md` |
| 아키텍처/DB | `context/` |
| 의사결정 기록 | `memories/decisions.md` |

## Workflow Protocol

1. **시작**: `memories/active_task.md` 읽고 이전 문맥 로드
2. **종료**: 작업 내용, 남은 과제, 이슈를 `active_task.md`에 업데이트

## Global Conventions

- **커밋**: `[FE]`, `[BE]`, `[COMMON]` 접두사
- **언어**: 변수명 영어, 주석 한국어
- **브랜치**: `feature/`, `fix/`, `refactor/`

## Available Agents

- `backend` - Spring Boot 구현
- `frontend` - React 구현
- `reviewer` - 코드 리뷰

<!-- ## Quick Commands

| 명령 | 실행 |
|-----|-----|
| 프론트 | `cd ui && npm run dev` |
| 백엔드 | `cd api && ./gradlew bootRun` |
| 전체 | `./scripts/run_all.sh` | -->