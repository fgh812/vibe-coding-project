# 에러 코드 목록

프로젝트 도메인별 에러 코드 정의. 구조/패턴은 `docs/api-conventions/Exception.md` 참조.

---

## 공통

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| INVALID_INPUT | 400 | 잘못된 입력 |
| UNAUTHORIZED | 401 | 인증 필요 |
| FORBIDDEN | 403 | 권한 없음 |
| INTERNAL_ERROR | 500 | 서버 내부 오류 |

---

## Team

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| TEAM_NOT_FOUND | 404 | 팀을 찾을 수 없음 |

## Member

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| MEMBER_NOT_FOUND | 404 | 멤버를 찾을 수 없음 |
| MEMBER_ALREADY_EXISTS | 409 | 이미 가입된 멤버 |

## Game

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| GAME_NOT_FOUND | 404 | 경기를 찾을 수 없음 |
| GAME_ALREADY_EXISTS | 409 | 해당 날짜에 이미 경기 존재 |
| GAME_NOT_SCHEDULED | 400 | 예정 상태가 아닌 경기 |

## Squad

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| SQUAD_NOT_FORMED | 400 | 스쿼드 미편성 |
| SQUAD_INVALID_SIZE | 400 | 스쿼드 인원 불균형 |

## Attendance

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| ATTENDANCE_ALREADY_RESPONDED | 409 | 이미 참석 응답 |

## Stats

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| STATS_NOT_FOUND | 404 | 통계 데이터 없음 |
