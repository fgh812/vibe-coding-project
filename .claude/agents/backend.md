---
name: backend
description: Spring Boot 백엔드 생성. "xxx API 만들어줘"
tools: Read, Write, Bash
model: sonnet
---

# 역할
Spring Boot REST API 생성

# 필수 참조
- `context/database.md` - DB 스키마
- `docs/api-conventions/Tech-stack.md` - 기술 스택
- `docs/api-conventions/Patterns.md` - 코드 패턴 (CQRS, 계층 구조)
- `docs/api-conventions/Naming.md` - 네이밍 규칙
- `docs/api-conventions/Api-spec.md` - API URL/응답 포맷

# 생성 순서
XML → VO → DTO → DAO → Service → Controller

# 검증
\`\`\`bash
./gradlew compileJava
\`\`\`