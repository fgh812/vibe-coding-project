# BCD 기능 목록

## 대시보드

| 기능 | 설명 | 문서 | 상태 |
|------|------|------|------|
| 대시보드 | 팀 현황, 최근 경기, 일정, 스탯 | [dashboard.md](./dashboard.md) | 📋 설계 |

## 마스터 관리

| 기능 | 설명 | 문서 | 상태 |
|------|------|------|------|
| 팀 관리 | 팀 CRUD, 유니폼 관리 | [team.md](./team.md) | 📋 설계 |
| 선수 관리 | 선수 CRUD, 포지션/사이즈 | [player.md](./player.md) | 📋 설계 |

## 경기 관리

| 기능 | 설명 | 문서 | 상태 |
|------|------|------|------|
| 경기 관리 | 생성, 로스터, 실시간 기록, 결과 | [game.md](./game.md) | 📋 설계 |

## 통계

| 기능 | 설명 | 문서 | 상태 |
|------|------|------|------|
| 통계 | 선수/팀 통계, 순위표 | [stats.md](./stats.md) | 📋 설계 |

---

## 화면 구성
```
/team                        # 팀 목록
/team/new                    # 팀 등록
/team/{teamId}               # 팀 상세 (유니폼 포함)
/team/{teamId}/edit          # 팀 수정

/player                      # 선수 목록
/player/new                  # 선수 등록
/player/{playerId}           # 선수 상세 (포지션, 사이즈 포함)
/player/{playerId}/edit      # 선수 수정

/game                        # 경기 목록
/game/new                    # 경기 생성
/game/{gameId}               # 경기 상세/결과
/game/{gameId}/edit          # 경기 수정
/game/{gameId}/roster        # 로스터 관리
/game/{gameId}/live          # 실시간 기록
/game/{gameId}/scoreboard    # 전광판 뷰

/stats/players               # 선수 통계
/stats/players/{playerId}    # 선수 통계 상세
/stats/teams                 # 팀 통계
/stats/teams/{teamId}        # 팀 통계 상세
/stats/standings             # 순위표
```

---

## 우선순위

| 순서 | 기능 | 이유 |
|------|------|------|
| 1 | 팀 관리 | 기초 마스터 |
| 2 | 선수 관리 | 기초 마스터 |
| 3 | 경기 관리 | 핵심 기능 |
| 4 | 통계 | 데이터 축적 후 |

---

## 상태 범례

| 상태 | 설명 |
|------|------|
| 📋 설계 | 기능 정의 완료 |
| 🚧 개발 | 개발 진행 중 |
| ✅ 완료 | 개발 완료 |
| ⏳ 미정 | 미정의 |