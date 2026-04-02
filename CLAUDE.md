# Project: Vibe Coding Project

각 프로젝트들의 스펙을 참조해서 바이브 코딩을 진행하는 프로젝트

## Directory Structure

심볼릭 링크로 두 개의 독립된 프로젝트를 연결.
- `_ui/`: React 프로젝트 (package.json)
- `_api/`: Spring Boot 프로젝트 (build.gradle)

> 파일 경로 참조 시 `_ui/` 또는 `_api/` 접두사 필수 확인

## Context Routing

### Base 규칙 (프로젝트 공통, 고정)
| 참조 파일 | 내용 |
|----------|------|
| `docs/api-conventions/Patterns.md` | 계층 구조, CQRS, 코드 패턴 |
| `docs/api-conventions/Naming.md` | 네이밍 규칙 |
| `docs/api-conventions/Api-spec.md` | REST API 설계 규칙, 응답 포맷 |
| `docs/api-conventions/Exception.md` | 예외 처리 구조 |
| `docs/api-conventions/Jpa.md` | JPA/Entity 규칙 |
| `docs/api-conventions/Mybatis.md` | MyBatis XML/DAO 규칙 |
| `docs/api-conventions/Testing.md` | 테스트 작성 규칙 |
| `docs/api-conventions/Tech-stack.md` | 기술 스택 규칙 |
| `docs/api-conventions/Git.md` | 커밋/브랜치/PR 규칙 |

### Dynamic 규칙 (프로젝트별, 가변)
| 참조 파일 | 내용 |
|----------|------|
| `context/project.md` | DB, PK전략, 도메인 목록, 패키지, 설정 |
| `context/database.md` | 테이블/컬럼 스키마 |
| `context/error-codes.md` | 에러 코드 목록 |
| `context/features/*.md` | 기능 스펙, API URL, 비즈니스 규칙 |

### 프로젝트별 규칙
| 작업 유형 | 참조 파일 |
|----------|----------|
| 프론트엔드 | `_ui/CLAUDE.md` |
| 백엔드 | `_api/CLAUDE.md` |

## Global Conventions

- **커밋**: `docs/api-conventions/Git.md` 참조 — `{type}({scope}): {subject}` 형식
- **언어**: 변수명 영어, 주석 한국어
- **브랜치**: `feature/`, `fix/`, `refactor/`, `docs/`

## Available Agents

- `backend` - Spring Boot 구현
- `frontend` - React 구현
- `reviewer` - 코드 리뷰

## 참조 우선순위

코드 생성 시 충돌이 있으면 아래 순서로 우선:
1. `context/` (프로젝트 스키마/스펙이 최우선)
2. `docs/api-conventions/` (코드 패턴/규칙)
3. `_api/CLAUDE.md`, `_ui/CLAUDE.md` (프로젝트별 오버라이드)

