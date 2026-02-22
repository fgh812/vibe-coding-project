# Git 컨벤션

## 개요
BCD API 프로젝트의 커밋 메시지, 브랜치 전략, PR 규칙을 정의한다.

---

## 1. 커밋 메시지

### 형식
```
{type}({scope}): {subject}

{body}       ← 선택
{footer}     ← 선택
```

### Type

| Type | 설명 | 예시 |
|------|------|------|
| `feat` | 새 기능 | `feat(game): 경기 생성 API 추가` |
| `fix` | 버그 수정 | `fix(stats): 평균 계산 오류 수정` |
| `refactor` | 리팩토링 | `refactor(squad): 스쿼드 편성 로직 분리` |
| `docs` | 문서 변경 | `docs: API 컨벤션 문서 추가` |
| `test` | 테스트 추가/수정 | `test(game): GameService 단위 테스트 추가` |
| `chore` | 빌드, 설정 변경 | `chore: Gradle 의존성 업데이트` |
| `style` | 코드 포맷 (기능 변경 없음) | `style: 불필요한 import 제거` |
| `perf` | 성능 개선 | `perf(stats): 통계 쿼리 인덱스 최적화` |

### Scope (선택)
도메인 또는 모듈명. 생략 가능.

| Scope | 예시 |
|-------|------|
| `team` | `feat(team): 팀 생성 API` |
| `member` | `fix(member): 멤버 중복 검증 누락` |
| `game` | `feat(game): 경기 생성 API` |
| `squad` | `feat(squad): 스쿼드 편성 API` |
| `stats` | `feat(stats): 개인 랭킹 조회` |
| `dashboard` | `feat(dashboard): 대시보드 API` |
| `common` | `refactor(common): 예외 계층 재설계` |

### Subject 규칙
- 한글 사용 가능
- 50자 이내
- 마침표 없음
- 명령형 ("추가", "수정", "삭제")

### 예시
```bash
# 기능 추가
feat(game): 경기 생성 API 추가

# 버그 수정
fix(stats): 0으로 나누기 에러 수정

경기 미참여 선수의 평균 스탯 계산 시
게임 수가 0인 경우 ArithmeticException 발생하는 문제 수정

# 리팩토링
refactor(squad): CQRS 패턴 적용

SquadRepository에서 QueryDAO/CommandDAO로 분리
- SquadQueryDAO: 스쿼드 조회
- SquadCommandDAO: 스쿼드 편성/수정

# 문서
docs: MyBatis 컨벤션 문서 추가
```

---

## 2. 브랜치 전략

### Git Flow (간소화)

```
main ─────────────────────────────── 운영 배포
  │
  └── develop ────────────────────── 개발 통합
        │
        ├── feature/game-create ──── 기능 개발
        ├── feature/stats-ranking
        ├── fix/stats-divide-zero ── 버그 수정
        └── hotfix/login-error ───── 긴급 수정 (main에서 분기)
```

### 브랜치 네이밍

| 브랜치 | 패턴 | 예시 |
|--------|------|------|
| 기능 개발 | `feature/{domain}-{기능}` | `feature/game-create` |
| 버그 수정 | `fix/{domain}-{설명}` | `fix/stats-divide-zero` |
| 긴급 수정 | `hotfix/{설명}` | `hotfix/login-error` |
| 리팩토링 | `refactor/{설명}` | `refactor/cqrs-pattern` |
| 문서 | `docs/{설명}` | `docs/api-conventions` |

### 브랜치 규칙
- `main`: 직접 push 금지, PR만 허용
- `develop`: 기능 브랜치 머지 대상
- feature/fix 브랜치: develop에서 분기 → develop으로 PR
- hotfix: main에서 분기 → main + develop 양쪽 머지

---

## 3. PR (Pull Request)

### PR 제목
커밋 메시지와 동일한 형식.
```
feat(game): 경기 생성 API 추가
```

### PR 템플릿
```markdown
## 변경 사항
- 경기 생성 API 구현 (POST /api/games)
- GameService, GameController, GameRepository 추가
- 중복 날짜 검증 로직 포함

## 변경 유형
- [x] 새 기능
- [ ] 버그 수정
- [ ] 리팩토링
- [ ] 문서 변경

## 테스트
- [x] GameServiceTest 추가
- [x] GameControllerTest 추가

## 참고 사항
- 스쿼드 편성 API는 다음 PR에서 진행
```

### PR 규칙
| 규칙 | 설명 |
|------|------|
| 1 PR = 1 기능 | 하나의 PR에 여러 기능 금지 |
| 테스트 포함 | 기능 변경 시 관련 테스트 필수 |
| 셀프 리뷰 | 머지 전 본인 코드 리뷰 |
| 컨벤션 검수 | `/bcd-review --diff` 실행 권장 |

---

## 4. 태그/릴리즈

### 버전 규칙 (Semantic Versioning)
```
v{MAJOR}.{MINOR}.{PATCH}
```

| 구분 | 변경 시점 | 예시 |
|------|----------|------|
| MAJOR | 호환성 깨지는 변경 | `v2.0.0` |
| MINOR | 기능 추가 | `v1.1.0` |
| PATCH | 버그 수정 | `v1.0.1` |

### 태그 생성
```bash
git tag -a v1.0.0 -m "v1.0.0 - 초기 릴리즈"
git push origin v1.0.0
```

---

## 5. .gitignore

```gitignore
# IDE
.idea/
*.iml
.vscode/

# Build
build/
out/
*.class

# Gradle
.gradle/
gradle/wrapper/gradle-wrapper.jar

# Env
.env
*.env.local
application-local.yml

# Logs
logs/
*.log

# OS
.DS_Store
Thumbs.db

# Review reports
*_code_review_report.md
```

---

## 6. 금지 사항

| 항목 | 이유 |
|------|------|
| main 직접 push | PR 통한 머지만 허용 |
| `git push --force` (main, develop) | 히스토리 꼬임 |
| 커밋에 비밀정보 포함 | DB 비밀번호, API 키 등 |
| 거대 커밋 (파일 30개+) | 리뷰 불가, 분할 커밋 |
| 의미 없는 커밋 메시지 | "수정", "fix", "update" 등 |