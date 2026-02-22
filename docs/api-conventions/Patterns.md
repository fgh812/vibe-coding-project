# 코드 패턴

## 개요
BCD API 프로젝트의 아키텍처 패턴, CQRS, 코드 구조 규칙을 정의한다.

---

## 1. 계층 구조

```
Client (React)
    ↓
Controller          — 요청 검증, 응답 변환
    ↓
Service             — 비즈니스 로직, 트랜잭션
    ↓ ↘
Repository    QueryDAO / CommandDAO
(JPA)         (MyBatis)
    ↓              ↓
        Database (MySQL)
```

### 계층별 책임

| 계층 | 책임 | 금지 |
|------|------|------|
| Controller | 요청 파라미터 검증, DTO 변환, 응답 포맷 | 비즈니스 로직, 직접 DB 접근 |
| Service | 비즈니스 로직, 트랜잭션 관리, 도메인 간 조합 | 직접 SQL 작성, HttpServletRequest 접근 |
| Repository | JPA 기본 CRUD | 복잡한 조인 쿼리 |
| QueryDAO | MyBatis 복잡 조회 (통계, 조인, 동적 쿼리) | 데이터 변경 |
| CommandDAO | MyBatis 복잡 변경 (벌크 insert 등) | 데이터 조회 |

### 의존성 규칙
```
Controller → Service (O)
Service → Repository, QueryDAO, CommandDAO (O)
Service → Service (X) — 직접 의존 금지. 도메인 간 조합이 필요하면 별도 유틸, 도메인 서비스, 또는 Aggregator로 분리
Controller → Repository / DAO (X) — 금지
```

---

## 2. CQRS 패턴

### JPA + MyBatis 역할 분담

| 용도 | 기술 | 이유 |
|------|------|------|
| 기본 CRUD (단건 저장/조회/삭제) | JPA Repository | 코드 간결, 엔티티 매핑 자동 |
| 복잡 조회 (통계, 다중 조인, 동적 조건) | MyBatis QueryDAO | SQL 직접 제어, 최적화 용이 |
| 복잡 변경 (벌크 insert, 배치) | MyBatis CommandDAO | 대량 처리 성능 |

### 사용 기준

```java
// ✅ JPA — 단순 CRUD
@Repository
public interface GameRepository extends JpaRepository<Game, Long> {
    List<Game> findByTeamIdOrderByGameDateDesc(Long teamId);
    Optional<Game> findByIdAndTeamId(Long id, Long teamId);
    boolean existsByTeamIdAndGameDate(Long teamId, LocalDate gameDate);
}

// ✅ MyBatis QueryDAO — 복잡 조회
@Mapper
public interface StatsQueryDAO {
    List<PlayerStatVO> findPlayerStatRanking(StatsSearchDTO search);
    GameDetailDTO findGameDetailWithSquads(Long gameId);
    List<WeeklyResultVO> findWeeklyResultList(Long teamId, String season);
}

// ✅ MyBatis CommandDAO — 복잡 변경
@Mapper
public interface StatsCommandDAO {
    void insertBatchPlayerStats(List<PlayerStatVO> stats);
    void updateBulkAttendance(Long gameId, List<Long> memberIds);
}
```

### CQRS 위반 예시
```java
// ❌ QueryDAO에 변경 메서드
@Mapper
public interface GameQueryDAO {
    List<GameDTO> findGameList(...);
    void insertGame(...);           // ERROR: CommandDAO로 이동
}

// ❌ Service에서 다른 Service 직접 의존
@Service
public class GameService {
    private final SquadService squadService;  // ERROR: 직접 의존 금지
}
```

---

## 3. Controller 패턴

```java
@RestController
@RequestMapping("/api/games")
@RequiredArgsConstructor
@Tag(name = "Game", description = "경기 관리 API")
public class GameController {

    private final GameService gameService;

    @GetMapping("/{gameId}")
    @Operation(summary = "경기 상세 조회")
    public ApiResponse<GameDetailDTO> getGameDetail(
            @PathVariable Long gameId) {
        return ApiResponse.success(gameService.getGameDetail(gameId));
    }

    @PostMapping
    @Operation(summary = "경기 생성")
    public ApiResponse<Long> createGame(
            @RequestBody @Valid GameCreateDTO dto) {
        return ApiResponse.success(gameService.createGame(dto));
    }
}
```

### Controller 규칙
- `@Valid`로 요청 DTO 검증
- 반환 타입은 `ApiResponse<T>` 래퍼 사용
- Swagger `@Operation` 필수
- 비즈니스 로직 금지 (Service에 위임)

---

## 4. Service 패턴

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class GameService {

    private final GameRepository gameRepository;
    private final GameQueryDAO gameQueryDAO;
    private final GameCommandDAO gameCommandDAO;

    // 조회 — readOnly 트랜잭션 (클래스 레벨)
    public GameDetailDTO getGameDetail(Long gameId) {
        return gameQueryDAO.findGameDetailWithSquads(gameId);
    }

    // 변경 — 쓰기 트랜잭션 오버라이드
    @Transactional
    public Long createGame(GameCreateDTO dto) {
        Game game = Game.create(dto);
        return gameRepository.save(game).getId();
    }
}
```

### Service 규칙
- 클래스 레벨: `@Transactional(readOnly = true)`
- 변경 메서드만: `@Transactional` 오버라이드
- Service → Service 직접 호출 금지

---

## 5. Entity 패턴

```java
@Entity
@Table(name = "game")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Game extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long teamId;

    @Column(nullable = false)
    private LocalDate gameDate;

    @Column(nullable = false)
    private LocalTime startTime;

    private String location;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private GameStatus status;  // SCHEDULED, IN_PROGRESS, COMPLETED

    // 정적 팩토리 메서드
    public static Game create(GameCreateDTO dto) {
        Game game = new Game();
        game.gameDate = dto.getGameDate();
        game.startTime = dto.getStartTime();
        game.location = dto.getLocation();
        game.status = GameStatus.SCHEDULED;
        return game;
    }

    // 비즈니스 메서드
    public void start() {
        this.status = GameStatus.IN_PROGRESS;
    }

    public void complete() {
        this.status = GameStatus.COMPLETED;
    }
}
```

### Entity 규칙
- `@Setter` 금지 — 비즈니스 메서드로 상태 변경
- `@NoArgsConstructor(access = AccessLevel.PROTECTED)` — JPA 전용
- 생성은 **정적 팩토리 메서드** 사용
- `BaseEntity` 상속 (createdAt, updatedAt, createdBy, updatedBy)

### BaseEntity
```java
@MappedSuperclass
@Getter
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

---

## 6. DTO 패턴

```java
// 요청 DTO — @Valid 검증
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class GameCreateDTO {

    @NotNull(message = "경기 날짜는 필수입니다")
    private LocalDate gameDate;

    @NotNull(message = "시작 시간은 필수입니다")
    private LocalTime startTime;

    private String location;
}

// 응답 DTO — 정적 팩토리 메서드
@Getter
@Builder
public class GameDetailDTO {

    private Long gameId;
    private LocalDate gameDate;
    private LocalTime startTime;
    private String location;
    private String status;
    private List<SquadDTO> squads;

    public static GameDetailDTO from(Game game) {
        return GameDetailDTO.builder()
                .gameId(game.getId())
                .gameDate(game.getGameDate())
                .startTime(game.getStartTime())
                .location(game.getLocation())
                .status(game.getStatus().name())
                .build();
    }
}
```

### DTO 규칙
- 요청 DTO: `@Valid` + Bean Validation 어노테이션
- 응답 DTO: `from()` 정적 팩토리 메서드로 Entity → DTO 변환
- `@Builder` 활용 권장

---

## 7. VO 패턴

```java
// VO — 불변, 값 비교
@Getter
@EqualsAndHashCode
@AllArgsConstructor
public class GameScoreVO {
    private final String squadName;
    private final int score;
}
```

### VO 규칙
- 모든 필드 `final`
- `@Setter` 금지
- `@EqualsAndHashCode` 필수
- MyBatis 조회 결과 매핑에 주로 사용

---

## 8. Enum 패턴

```java
@Getter
@RequiredArgsConstructor
public enum GameStatus {
    SCHEDULED("예정"),
    IN_PROGRESS("진행중"),
    COMPLETED("완료"),
    CANCELLED("취소");

    private final String description;
}

@Getter
@RequiredArgsConstructor
public enum SquadName {
    A("A팀"),
    B("B팀"),
    C("C팀");

    private final String displayName;
}
```

---

## 9. Lombok 사용 규칙

| 어노테이션 | 사용처 | 비고 |
|-----------|--------|------|
| `@Getter` | 모든 클래스 | 필수 |
| `@Setter` | 사용 자제 | Entity, VO 금지. 요청 DTO는 `@NoArgsConstructor` + `@Builder`로 대체 |
| `@NoArgsConstructor` | Entity, DTO | Entity는 `PROTECTED` |
| `@AllArgsConstructor` | DTO, VO | 필요시 |
| `@RequiredArgsConstructor` | Service, Controller | DI용 |
| `@Builder` | DTO | 권장 |
| `@EqualsAndHashCode` | VO | 필수 |
| `@Data` | 사용 금지 | Setter 포함되므로 |
| `@ToString` | 필요시 | 순환참조 주의 |