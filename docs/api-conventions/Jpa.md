# JPA 규칙

## 개요
BCD API 프로젝트의 JPA 엔티티, Repository, 연관관계 매핑 규칙을 정의한다.
JPA는 **기본 CRUD**와 **단순 조회**에 사용한다.

---

## 1. 역할 범위

### JPA를 사용하는 경우
- 단건 저장/조회/삭제 (save, findById, delete)
- 단일 테이블 또는 간단한 조건 조회
- Spring Data JPA 쿼리 메서드로 해결 가능한 쿼리
- 엔티티 상태 변경 (dirty checking)

### MyBatis를 사용하는 경우 (JPA 사용 안 함)
- 3개 이상 테이블 조인
- 통계/집계 쿼리
- 동적 WHERE 조건이 복잡한 경우
- 벌크 INSERT/UPDATE

---

## 2. Entity 규칙

### 기본 구조
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

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private GameStatus status;

    // 정적 팩토리 메서드로 생성
    public static Game create(GameCreateDTO dto, Long teamId) { ... }

    // 비즈니스 메서드로 상태 변경
    public void complete() { ... }
}
```

### Entity 필수 규칙

| 규칙 | 설명 |
|------|------|
| `@Setter` 금지 | 비즈니스 메서드로 상태 변경 |
| `@NoArgsConstructor(PROTECTED)` | JPA 전용, 외부 생성 방지 |
| 정적 팩토리 메서드 | `create()`, `of()` 등으로 생성 |
| `BaseEntity` 상속 | 공통 감사(audit) 필드 |
| `@Enumerated(STRING)` | 숫자(ORDINAL) 사용 금지 |

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

## 3. 연관관계 매핑

### 기본 원칙
- **지연 로딩(LAZY) 기본** — `@ManyToOne(fetch = FetchType.LAZY)`
- **양방향 매핑 최소화** — 단방향 우선, 필요시에만 양방향
- **복잡한 조인은 MyBatis로** — JPA 연관관계 남용 금지

### FK를 직접 들고 있는 패턴 (권장)
```java
// ✅ 단순한 FK 참조 — 엔티티 연관관계 대신 ID 직접 보유
@Entity
@Table(name = "game")
public class Game extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long teamId;           // Team 엔티티 연관관계 대신 ID

    // ...
}
```

### 엔티티 연관관계 패턴 (필요시)
```java
// 연관관계가 꼭 필요한 경우에만
@Entity
@Table(name = "squad_member")
public class SquadMember extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "squad_id", nullable = false)
    private Squad squad;

    @Column(nullable = false)
    private Long memberId;
}
```

### 연관관계 판단 기준

| 상황 | 선택 | 이유 |
|------|------|------|
| 단순 FK 참조 | ID 직접 보유 | 불필요한 조인 방지 |
| 같은 aggregate 내 | 엔티티 연관관계 | 일관성 보장 |
| 복잡한 조회 필요 | MyBatis로 처리 | N+1 문제 회피 |

---

## 4. Repository 규칙

### 기본 구조
```java
@Repository
public interface GameRepository extends JpaRepository<Game, Long> {

    // Spring Data JPA 쿼리 메서드
    List<Game> findByTeamIdOrderByGameDateDesc(Long teamId);

    Optional<Game> findByIdAndTeamId(Long id, Long teamId);

    boolean existsByTeamIdAndGameDate(Long teamId, LocalDate gameDate);

    List<Game> findByTeamIdAndStatus(Long teamId, GameStatus status);
}
```

### 쿼리 메서드 네이밍
```java
// 조회
findBy{필드}()                          // 조건 조회
findBy{필드}OrderBy{필드}Desc()         // 정렬
findFirstByTeamIdOrderByGameDateDesc()  // 최신 1건

// 존재 여부
existsBy{필드}()

// 삭제
deleteBy{필드}()

// 개수
countBy{필드}()
```

### @Query 사용 (제한적)
간단한 JPQL만 허용. 복잡하면 MyBatis로 이동.
```java
// ✅ 간단한 JPQL — 허용
@Query("SELECT g FROM Game g WHERE g.teamId = :teamId AND g.status = 'COMPLETED' ORDER BY g.gameDate DESC")
List<Game> findRecentCompletedGames(@Param("teamId") Long teamId, Pageable pageable);

// ❌ 복잡한 조인 — MyBatis로 이동
@Query("SELECT g FROM Game g JOIN Squad s ON ... JOIN SquadMember sm ON ... WHERE ...")
// → StatsQueryDAO.findGameDetailWithSquads()로 이동
```

### @Query 사용 기준

| 복잡도 | 방법 |
|--------|------|
| 메서드 네이밍으로 가능 | 쿼리 메서드 |
| 단순 JPQL (조인 1개 이하) | @Query |
| 복잡 쿼리 (조인 2개+, 서브쿼리, 동적 조건) | MyBatis QueryDAO |

---

## 5. 트랜잭션

### 기본 원칙
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)     // 클래스 레벨: 읽기 전용
public class GameService {

    // 조회 메서드 — 클래스 레벨 readOnly 적용
    public GameDetailDTO getGameDetail(Long teamId, Long gameId) { ... }

    // 변경 메서드 — 쓰기 트랜잭션으로 오버라이드
    @Transactional
    public Long createGame(Long teamId, GameCreateDTO dto) { ... }

    @Transactional
    public void updateGameScore(Long gameId, GameScoreVO score) { ... }
}
```

### 트랜잭션 규칙
| 규칙 | 설명 |
|------|------|
| 클래스 레벨 `readOnly = true` | 기본 조회 최적화 |
| 변경 메서드만 `@Transactional` | 명시적 쓰기 트랜잭션 |
| Controller에 트랜잭션 금지 | Service 레이어에서만 |

---

## 6. N+1 방지

### 문제
```java
// ❌ N+1 발생
List<Squad> squads = squadRepository.findByGameId(gameId);
squads.forEach(s -> s.getMembers().size());  // 스쿼드마다 추가 쿼리
```

### 해결 방법

| 방법 | 적용 |
|------|------|
| fetch join (@Query) | 간단한 1:N |
| MyBatis로 전환 | 복잡한 조회 |
| @BatchSize | 부분 최적화 |

```java
// ✅ fetch join
@Query("SELECT s FROM Squad s JOIN FETCH s.members WHERE s.gameId = :gameId")
List<Squad> findByGameIdWithMembers(@Param("gameId") Long gameId);

// ✅ 복잡하면 MyBatis
// → SquadQueryDAO.findSquadListWithMembers(gameId)
```

---

## 7. 금지 사항

| 항목 | 이유 |
|------|------|
| `@Setter` on Entity | 무분별한 상태 변경 |
| `@Data` on Entity | Setter + equals/hashCode 문제 |
| `FetchType.EAGER` | N+1, 성능 저하 |
| `@Enumerated(ORDINAL)` | 순서 변경 시 데이터 꼬임 |
| 양방향 매핑 남용 | 순환참조, 복잡도 증가 |
| Repository에서 복잡 조인 | MyBatis로 처리 |
| `cascade = ALL` 남용 | 의도치 않은 삭제/저장 |