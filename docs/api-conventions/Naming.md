# 네이밍 규칙

## 개요
BCD API 프로젝트의 클래스, 메서드, 변수 네이밍 규칙을 정의한다.

---

## 1. 패키지 구조

```
com.bcd.api
├── team/           # 팀(모임)
├── member/         # 멤버
├── game/           # 경기
├── squad/          # 스쿼드 편성
├── stats/          # 통계
├── dashboard/      # 대시보드
└── common/         # 공통
    ├── config/     # 설정
    ├── exception/  # 예외
    ├── util/       # 유틸리티
    ├── dto/        # 공통 DTO (페이징, 응답 래퍼 등)
    └── entity/     # 공통 엔티티 (BaseEntity 등)
```

### 도메인 패키지 내부 구조
```
com.bcd.api.game/
├── controller/
│   └── GameController.java
├── service/
│   └── GameService.java
├── dao/
│   ├── GameQueryDAO.java           # MyBatis 복잡 조회
│   └── GameCommandDAO.java         # MyBatis 복잡 변경 (필요시)
├── repository/
│   └── GameRepository.java         # JPA 기본 CRUD
├── entity/
│   └── Game.java
├── dto/
│   ├── GameCreateDTO.java
│   ├── GameUpdateDTO.java
│   └── GameDetailDTO.java
└── vo/
    └── GameScoreVO.java
```

---

## 2. 클래스 네이밍

### Controller
| 규칙 | 예시 |
|------|------|
| `{Domain}Controller` | `GameController`, `TeamController` |
| REST 리소스 단위 | 하나의 도메인 = 하나의 컨트롤러 |

### Service
| 규칙 | 예시 |
|------|------|
| `{Domain}Service` | `GameService`, `StatsService` |
| 인터페이스 없이 클래스로 | `GameService` (인터페이스 불필요) |

### Repository (JPA)
| 규칙 | 예시 |
|------|------|
| `{Domain}Repository` | `GameRepository`, `MemberRepository` |
| Spring Data JPA 인터페이스 | `extends JpaRepository<Game, Long>` |

### DAO (MyBatis - CQRS)
| 규칙 | 예시 |
|------|------|
| `{Domain}QueryDAO` | `GameQueryDAO`, `StatsQueryDAO` |
| `{Domain}CommandDAO` | `GameCommandDAO` (필요시) |
| `@Mapper` 어노테이션 사용 | MyBatis 매퍼 인터페이스 |

### Entity
| 규칙 | 예시 |
|------|------|
| `{Domain}` (접미사 없음) | `Game`, `Member`, `Team` |
| `@Entity` 어노테이션 | JPA 엔티티 |

### DTO
| 규칙 | 예시 | 용도 |
|------|------|------|
| `{Domain}CreateDTO` | `GameCreateDTO` | 생성 요청 |
| `{Domain}UpdateDTO` | `GameUpdateDTO` | 수정 요청 |
| `{Domain}DetailDTO` | `GameDetailDTO` | 상세 응답 |
| `{Domain}ListDTO` | `GameListDTO` | 목록 응답 |
| `{Domain}SearchDTO` | `GameSearchDTO` | 검색 조건 |

### VO
| 규칙 | 예시 | 용도 |
|------|------|------|
| `{Domain}{의미}VO` | `GameScoreVO` | 불변 값 객체 |
| | `PlayerStatVO` | 통계 값 |
| | `SeasonRecordVO` | 시즌 기록 |

### DTO vs VO 기준
| 구분 | DTO | VO |
|------|-----|-----|
| 용도 | 계층 간 데이터 전송 | 값 표현 |
| 가변성 | `@NoArgsConstructor` + `@Builder` | 불변 (record 또는 final 필드) |
| 동등성 | 참조 비교 | 값 비교 (equals/hashCode) |
| 예시 | 요청/응답 객체 | 스코어, 통계, 기록 |

---

## 3. 메서드 네이밍

### Controller
| HTTP Method | 접두어 | 예시 |
|-------------|--------|------|
| GET (단건) | `get*` | `getGameDetail()` |
| GET (목록) | `get*List` | `getGameList()` |
| POST | `create*` | `createGame()` |
| PUT | `update*` | `updateGame()` |
| DELETE | `delete*` | `deleteGame()` |

### Service
| 작업 | 접두어 | 예시 |
|------|--------|------|
| 단건 조회 | `get*` | `getGameDetail()` |
| 목록 조회 | `get*List` | `getGameListByTeamId()` |
| 생성 | `create*` | `createGame()` |
| 수정 | `update*` | `updateGameScore()` |
| 삭제 | `delete*` | `deleteGame()` |
| 검증 | `validate*` | `validateSquadSize()` |
| 계산 | `calculate*` | `calculatePlayerStats()` |

### QueryDAO (MyBatis 조회)
| 작업 | 접두어 | 예시 |
|------|--------|------|
| 단건 조회 | `find*` | `findGameById()` |
| 목록 조회 | `find*List` | `findGameListByTeamId()` |
| 건수 조회 | `count*` | `countGamesByTeamId()` |
| 존재 여부 | `exists*` | `existsGameOnDate()` |

### CommandDAO (MyBatis 변경)
| 작업 | 접두어 | 예시 |
|------|--------|------|
| 등록 | `insert*` | `insertGameStats()` |
| 수정 | `update*` | `updateGameScore()` |
| 삭제 | `delete*` | `deleteGameStats()` |
| 일괄 등록 | `insertBatch*` | `insertBatchPlayerStats()` |

### Repository (JPA)
Spring Data JPA 메서드 네이밍 컨벤션을 따른다.
```java
// 기본 제공
save(), findById(), deleteById(), findAll()

// 커스텀 쿼리 메서드
findByTeamIdAndGameDate()
findByTeamIdOrderByGameDateDesc()
existsByTeamIdAndGameDate()
```

---

## 4. 변수 네이밍

### 기본 규칙
| 규칙 | 예시 |
|------|------|
| camelCase | `gameDate`, `teamName` |
| boolean은 `is` 접두어 | `isActive`, `isAttending` |
| 컬렉션은 복수형 | `members`, `gameList` |
| 약어 금지 (일부 예외) | `id`, `dto`, `vo` 허용 |

### ID 필드
| 규칙 | 예시 |
|------|------|
| Entity | `id` (Long) |
| DTO/VO에서 참조 | `gameId`, `teamId`, `memberId` |

### 날짜/시간
| 규칙 | 예시 |
|------|------|
| 날짜 | `*Date` — `gameDate`, `createdDate` |
| 시간 | `*Time` — `startTime`, `endTime` |
| 일시 | `*DateTime` — `gameDateTime` |

### 상수
| 규칙 | 예시 |
|------|------|
| UPPER_SNAKE_CASE | `MAX_SQUAD_SIZE`, `DEFAULT_QUARTER_MINUTES` |

---

## 5. DB 네이밍 (테이블/컬럼)

### 테이블
| 규칙 | 예시 |
|------|------|
| snake_case, 단수형 | `team`, `member`, `game` |
| 연관 테이블 | `game_player_stat`, `squad_member` |

### 컬럼
| 규칙 | 예시 |
|------|------|
| snake_case | `team_id`, `game_date`, `created_at` |
| PK | `id` (BIGINT, AUTO_INCREMENT) |
| FK | `{참조테이블}_id` — `team_id`, `game_id` |
| boolean | `is_*` — `is_active`, `is_attending` |
| 생성/수정 | `created_at`, `updated_at`, `created_by`, `updated_by` |