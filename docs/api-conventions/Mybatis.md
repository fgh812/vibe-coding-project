# MyBatis 규칙

## 개요
BCD API 프로젝트의 MyBatis XML 매퍼, DAO 인터페이스 작성 규칙을 정의한다.
MyBatis는 **복잡 조회(통계, 다중 조인, 동적 쿼리)**와 **벌크 처리**에 사용한다.

---

## 1. 역할 범위

### MyBatis를 사용하는 경우
- 3개 이상 테이블 조인
- 동적 WHERE 조건 (검색, 필터)
- 통계/집계 쿼리 (GROUP BY, 서브쿼리)
- 벌크 INSERT/UPDATE
- 네이티브 SQL 최적화가 필요한 경우

### JPA를 사용하는 경우 (MyBatis 사용 안 함)
- 단순 CRUD (save, findById, delete)
- 단일 테이블 조회
- Spring Data JPA 메서드 네이밍으로 해결 가능한 쿼리

---

## 2. 파일 구조

### XML 매퍼 위치
```
resources/mapper/
├── game/
│   ├── GameQuery.xml               # 조회 SQL
│   └── GameCommand.xml             # 변경 SQL (필요시)
├── stats/
│   ├── StatsQuery.xml
│   └── StatsCommand.xml
├── squad/
│   └── SquadQuery.xml
└── dashboard/
    └── DashboardQuery.xml
```

### DAO 인터페이스 위치
```
com.bcd.api.{domain}/dao/
├── {Domain}QueryDAO.java
└── {Domain}CommandDAO.java         # 필요시
```

---

## 3. XML 매퍼 규칙

### namespace
```xml
<!-- namespace = DAO 인터페이스 풀 패키지 -->
<mapper namespace="com.bcd.api.stats.dao.StatsQueryDAO">
```

### SQL 주석 패턴
모든 SQL 문의 첫 줄에 주석을 작성한다.

```xml
<!-- 형식: /* bcd.{도메인}.{Mapper파일명}.{메서드명} */ -->
<select id="findPlayerStatRanking" resultType="PlayerStatVO">
    /* bcd.stats.StatsQuery.findPlayerStatRanking */
    SELECT
        m.id AS member_id,
        m.member_nm,
        AVG(ps.points) AS avg_points
    FROM player_stat ps
    JOIN member m ON m.id = ps.member_id
    WHERE ps.team_id = #{teamId}
    GROUP BY m.id, m.member_nm
    ORDER BY avg_points DESC
</select>
```

### id 규칙
- `<select>` id = QueryDAO 메서드명과 일치
- `<insert>`, `<update>`, `<delete>` id = CommandDAO 메서드명과 일치

---

## 4. 파라미터 바인딩

### #{} 사용 (기본)
```xml
<!-- ✅ PreparedStatement — SQL Injection 방지 -->
<select id="findGameById" resultType="GameDetailDTO">
    /* bcd.game.GameQuery.findGameById */
    SELECT * FROM game WHERE id = #{gameId}
</select>
```

### ${} 사용 (제한적)
```xml
<!-- ⚠️ 동적 컬럼/테이블명에만 허용, 반드시 화이트리스트 검증 -->
<select id="findPlayerStatRanking" resultType="PlayerStatVO">
    /* bcd.stats.StatsQuery.findPlayerStatRanking */
    SELECT *
    FROM player_stat
    ORDER BY ${sortColumn} ${sortDirection}
</select>
```

**${} 사용 시 필수 조건:**
- Service 레이어에서 화이트리스트 검증 후 전달
- 사용자 입력값 직접 바인딩 금지
- 허용 값: 컬럼명, 정렬 방향 (ASC/DESC)

---

## 5. 동적 SQL

### if
```xml
<select id="findGameList" resultType="GameListDTO">
    /* bcd.game.GameQuery.findGameList */
    SELECT g.*
    FROM game g
    WHERE g.team_id = #{teamId}
    <if test="status != null">
        AND g.status = #{status}
    </if>
    <if test="startDate != null and endDate != null">
        AND g.game_date BETWEEN #{startDate} AND #{endDate}
    </if>
    ORDER BY g.game_date DESC
</select>
```

### choose / when / otherwise
```xml
<choose>
    <when test="sortType == 'POINTS'">
        ORDER BY avg_points DESC
    </when>
    <when test="sortType == 'REBOUNDS'">
        ORDER BY avg_rebounds DESC
    </when>
    <otherwise>
        ORDER BY avg_points DESC
    </otherwise>
</choose>
```

### foreach (벌크 처리)
```xml
<insert id="insertBatchPlayerStats">
    /* bcd.stats.StatsCommand.insertBatchPlayerStats */
    INSERT INTO player_stat (game_id, member_id, squad_name, points, rebounds, assists)
    VALUES
    <foreach collection="stats" item="stat" separator=",">
        (#{stat.gameId}, #{stat.memberId}, #{stat.squadName},
         #{stat.points}, #{stat.rebounds}, #{stat.assists})
    </foreach>
</insert>
```

### where 태그
```xml
<!-- ✅ 자동으로 AND/OR 처리 -->
<select id="findMemberList" resultType="MemberListDTO">
    /* bcd.member.MemberQuery.findMemberList */
    SELECT *
    FROM member
    <where>
        <if test="teamId != null">
            AND team_id = #{teamId}
        </if>
        <if test="isActive != null">
            AND is_active = #{isActive}
        </if>
        <if test="keyword != null and keyword != ''">
            AND member_nm LIKE CONCAT('%', #{keyword}, '%')
        </if>
    </where>
</select>
```

---

## 6. resultType / resultMap

### resultType (기본)
DTO/VO의 풀 패키지 또는 typeAlias 사용.
```xml
<select id="findGameDetailWithSquads" resultType="com.bcd.api.game.dto.GameDetailDTO">
```

### typeAlias 설정 (application.yml)
```yaml
mybatis:
  type-aliases-package: com.bcd.api
  mapper-locations: classpath:mapper/**/*.xml
  configuration:
    map-underscore-to-camel-case: true
```

### resultMap (복잡한 매핑)
```xml
<resultMap id="gameDetailMap" type="GameDetailDTO">
    <id property="gameId" column="game_id"/>
    <result property="gameDate" column="game_date"/>
    <collection property="squads" ofType="SquadDTO">
        <id property="squadName" column="squad_name"/>
        <collection property="members" ofType="SquadMemberDTO">
            <id property="memberId" column="member_id"/>
            <result property="memberNm" column="member_nm"/>
        </collection>
    </collection>
</resultMap>
```

---

## 7. 페이징

```xml
<select id="findGameListPaged" resultType="GameListDTO">
    /* bcd.game.GameQuery.findGameListPaged */
    SELECT g.*
    FROM game g
    WHERE g.team_id = #{teamId}
    ORDER BY g.game_date DESC
    LIMIT #{size} OFFSET #{offset}
</select>
```

별도 count 쿼리:
```xml
<select id="countGamesByTeamId" resultType="int">
    /* bcd.game.GameQuery.countGamesByTeamId */
    SELECT COUNT(*)
    FROM game
    WHERE team_id = #{teamId}
</select>
```

---

## 8. 금지 사항

| 항목 | 이유 |
|------|------|
| SELECT * 사용 | 필요 컬럼만 명시 |
| SQL 주석 누락 | 슬로우 쿼리 추적 불가 |
| QueryDAO에 INSERT/UPDATE/DELETE | CQRS 위반 |
| CommandDAO에 SELECT | CQRS 위반 |
| 사용자 입력 `${}` 직접 바인딩 | SQL Injection 위험 |
| XML 내 비즈니스 로직 | Service에서 처리 |