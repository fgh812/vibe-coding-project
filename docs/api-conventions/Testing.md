# 테스트 규칙

## 개요
BCD API 프로젝트의 테스트 네이밍, 구조, 커버리지 기준을 정의한다.

---

## 1. 테스트 종류

| 종류 | 대상 | 어노테이션 | 속도 |
|------|------|-----------|------|
| Unit Test | Service, 유틸 | `@ExtendWith(MockitoExtension.class)` | 빠름 |
| Repository Test | JPA Repository | `@DataJpaTest` | 보통 |
| MyBatis Test | QueryDAO, CommandDAO | `@MybatisTest` | 보통 |
| Integration Test | Controller (API) | `@SpringBootTest` + `@AutoConfigureMockMvc` | 느림 |

---

## 2. 테스트 파일 구조

```
src/test/java/com/bcd/api/
├── game/
│   ├── controller/
│   │   └── GameControllerTest.java          # 통합 테스트
│   ├── service/
│   │   └── GameServiceTest.java             # 단위 테스트
│   ├── dao/
│   │   ├── GameQueryDAOTest.java            # MyBatis 테스트
│   │   └── GameCommandDAOTest.java
│   └── repository/
│       └── GameRepositoryTest.java          # JPA 테스트
├── stats/
│   ├── service/
│   │   └── StatsServiceTest.java
│   └── dao/
│       └── StatsQueryDAOTest.java
└── common/
    └── fixture/
        ├── GameFixture.java                 # 테스트 데이터 팩토리
        ├── MemberFixture.java
        └── TeamFixture.java
```

---

## 3. 테스트 네이밍

### 클래스명
```
{대상클래스}Test
```
예: `GameServiceTest`, `GameQueryDAOTest`, `GameControllerTest`

### 메서드명 — 한글 허용
```
{메서드명}_{시나리오}_{기대결과}
```

```java
@Test
void createGame_정상_경기생성성공() { }

@Test
void createGame_중복날짜_ConflictException발생() { }

@Test
void getGameDetail_존재하지않는경기_NotFoundException발생() { }

@Test
void findPlayerStatRanking_데이터없음_빈리스트반환() { }
```

### DisplayName (선택)
```java
@Test
@DisplayName("경기 생성 - 동일 날짜 경기 존재 시 예외 발생")
void createGame_중복날짜_ConflictException발생() { }
```

---

## 4. 테스트 패턴

### Service 단위 테스트
```java
@ExtendWith(MockitoExtension.class)
class GameServiceTest {

    @InjectMocks
    private GameService gameService;

    @Mock
    private GameRepository gameRepository;

    @Mock
    private GameQueryDAO gameQueryDAO;

    @Test
    void createGame_정상_경기생성성공() {
        // given
        GameCreateDTO dto = GameCreateDTO.builder()
                .gameDate(LocalDate.of(2025, 3, 16))
                .startTime(LocalTime.of(14, 0))
                .location("OO체육관")
                .build();

        given(gameRepository.save(any(Game.class)))
                .willReturn(GameFixture.createGame(1L));

        // when
        Long gameId = gameService.createGame(dto);

        // then
        assertThat(gameId).isEqualTo(1L);
        then(gameRepository).should().save(any(Game.class));
    }

    @Test
    void createGame_중복날짜_ConflictException발생() {
        // given
        GameCreateDTO dto = GameCreateDTO.builder()
                .gameDate(LocalDate.of(2025, 3, 16))
                .build();

        given(gameRepository.existsByTeamIdAndGameDate(any(), eq(dto.getGameDate())))
                .willReturn(true);

        // when & then
        assertThatThrownBy(() -> gameService.createGame(dto))
                .isInstanceOf(BcdConflictException.class);
    }
}
```

### JPA Repository 테스트
```java
@DataJpaTest
class GameRepositoryTest {

    @Autowired
    private GameRepository gameRepository;

    @Test
    void findByTeamIdOrderByGameDateDesc_정상_최신순정렬() {
        // given
        Game game1 = GameFixture.createGame(LocalDate.of(2025, 3, 9));
        Game game2 = GameFixture.createGame(LocalDate.of(2025, 3, 16));
        gameRepository.saveAll(List.of(game1, game2));

        // when
        List<Game> games = gameRepository.findByTeamIdOrderByGameDateDesc(1L);

        // then
        assertThat(games).hasSize(2);
        assertThat(games.get(0).getGameDate()).isAfter(games.get(1).getGameDate());
    }
}
```

### MyBatis DAO 테스트
```java
@MybatisTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class StatsQueryDAOTest {

    @Autowired
    private StatsQueryDAO statsQueryDAO;

    @Test
    @Sql("/sql/stats-test-data.sql")
    void findPlayerStatRanking_정상_득점순정렬() {
        // given
        StatsSearchDTO search = StatsSearchDTO.builder()
                .teamId(1L)
                .sortType("POINTS")
                .build();

        // when
        List<PlayerStatVO> ranking = statsQueryDAO.findPlayerStatRanking(search);

        // then
        assertThat(ranking).isNotEmpty();
        assertThat(ranking.get(0).getAvgPoints())
                .isGreaterThanOrEqualTo(ranking.get(1).getAvgPoints());
    }
}
```

### Controller 통합 테스트
```java
@SpringBootTest
@AutoConfigureMockMvc
class GameControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void createGame_정상_201반환() throws Exception {
        // given
        GameCreateDTO dto = GameCreateDTO.builder()
                .gameDate(LocalDate.of(2025, 3, 16))
                .startTime(LocalTime.of(14, 0))
                .location("OO체육관")
                .build();

        // when & then
        mockMvc.perform(post("/api/games")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(dto)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").isNumber());
    }

    @Test
    void createGame_날짜누락_400반환() throws Exception {
        // given
        GameCreateDTO dto = GameCreateDTO.builder()
                .location("OO체육관")
                .build();

        // when & then
        mockMvc.perform(post("/api/games")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(dto)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.error.code").value("INVALID_INPUT"));
    }
}
```

---

## 5. 테스트 Fixture

### Fixture 클래스
테스트 데이터 생성을 위한 팩토리 클래스.

```java
public class GameFixture {

    public static Game createGame(Long id) {
        return Game.builder()
                .id(id)
                .gameDate(LocalDate.of(2025, 3, 16))
                .startTime(LocalTime.of(14, 0))
                .location("OO체육관")
                .status(GameStatus.SCHEDULED)
                .build();
    }

    public static Game createGame(LocalDate gameDate) {
        return Game.builder()
                .gameDate(gameDate)
                .startTime(LocalTime.of(14, 0))
                .location("OO체육관")
                .status(GameStatus.SCHEDULED)
                .build();
    }

    public static GameCreateDTO createDTO() {
        return GameCreateDTO.builder()
                .gameDate(LocalDate.of(2025, 3, 16))
                .startTime(LocalTime.of(14, 0))
                .location("OO체육관")
                .build();
    }
}
```

### SQL 테스트 데이터
MyBatis 테스트용:
```
src/test/resources/sql/
├── stats-test-data.sql
├── game-test-data.sql
└── member-test-data.sql
```

---

## 6. 테스트 작성 규칙

### given-when-then
모든 테스트는 given-when-then 구조를 따른다.
```java
@Test
void methodName_시나리오_기대결과() {
    // given — 테스트 데이터 준비

    // when — 실행

    // then — 검증
}
```

### 필수 테스트 케이스

| 계층 | 필수 테스트 |
|------|-----------|
| Service | 정상 케이스, 예외 케이스 (Not Found, Conflict 등) |
| Repository | 커스텀 쿼리 메서드 |
| QueryDAO | 조회 결과 정합성, 동적 조건 |
| Controller | 성공 응답, 유효성 실패(400), 리소스 없음(404) |

### Assertion 라이브러리
AssertJ 사용 권장.
```java
// ✅ AssertJ
assertThat(result).isNotNull();
assertThat(result.getGameDate()).isEqualTo(LocalDate.of(2025, 3, 16));
assertThatThrownBy(() -> service.getGame(999L))
        .isInstanceOf(BcdNotFoundException.class);

// ❌ JUnit 기본 (비권장)
assertEquals(expected, actual);
assertNotNull(result);
```

---

## 7. 커버리지 기준

| 계층 | 목표 커버리지 |
|------|-------------|
| Service | 80% 이상 |
| Repository / DAO | 커스텀 메서드 100% |
| Controller | 주요 API 엔드포인트 |
| Entity | 비즈니스 메서드가 있는 경우 |
| DTO / VO | 불필요 (Lombok 생성 코드) |

---

## 8. 금지 사항

| 항목 | 이유 |
|------|------|
| `@SpringBootTest` 남용 | 느림, 단위 테스트 우선 |
| 테스트 간 순서 의존 | 독립적으로 실행 가능해야 함 |
| 하드코딩된 테스트 데이터 | Fixture 클래스 사용 |
| `Thread.sleep()` | `Awaitility` 등 사용 |
| `System.out.println` | 로거 또는 assertion 사용 |
| 프로덕션 DB 접근 | 테스트 전용 DB/H2 사용 |