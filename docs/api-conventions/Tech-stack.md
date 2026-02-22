# 기술 스택

## 개요
BCD API 프로젝트의 기술 스택과 버전 규칙을 정의한다.

---

## 1. 핵심 기술 스택

| 카테고리 | 기술 | 버전 |
|---------|------|------|
| Language | Java | 17+ |
| Framework | Spring Boot | 3.x |
| ORM | Spring Data JPA (Hibernate) | - |
| SQL Mapper | MyBatis | 3.x |
| Database | MySQL | 8.x |
| Build | Gradle (Kotlin DSL) | 8.x |
| API Docs | Springdoc OpenAPI (Swagger) | 2.x |
| Logging | SLF4J + Logback | - |
| Test | JUnit 5 + Mockito | - |

---

## 2. Spring Boot 3.x 호환성

### Jakarta EE 패키지
Spring Boot 3.x는 Jakarta EE를 사용한다. `javax.*` → `jakarta.*`

```java
// ✅ 올바른 import
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.validation.constraints.NotNull;
import jakarta.servlet.http.HttpServletRequest;

// ❌ 사용 금지
import javax.persistence.Entity;
import javax.validation.constraints.NotNull;
```

### 주요 변경 패키지

| 기존 (javax) | 변경 (jakarta) |
|-------------|---------------|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.annotation.*` | `jakarta.annotation.*` |

---

## 3. Java 17 기능 활용

### Record (제한적 사용)
JPA Entity, MyBatis VO, 요청/응답 DTO에는 사용하지 않는다. 순수 내부 값 전달에만 허용.
```java
// ✅ 내부 값 전달용
public record GameEvent(Long gameId, String eventType) {}

// ❌ VO, DTO에는 클래스 사용 (MyBatis 매핑, @Valid 등 필요)
```

### Text Block
```java
// ✅ 긴 문자열
String message = """
    클럽 '%s'의 경기가 생성되었습니다.
    날짜: %s
    장소: %s
    """.formatted(teamName, gameDate, location);
```

### Sealed Class (필요시)
```java
// 이벤트 타입 제한
public sealed interface GameEvent permits GameCreated, GameCompleted, GameCancelled {}
public record GameCreated(Long gameId, Long teamId) implements GameEvent {}
public record GameCompleted(Long gameId, GameScoreVO score) implements GameEvent {}
```

### Switch Expression
```java
String label = switch (status) {
    case SCHEDULED -> "예정";
    case IN_PROGRESS -> "진행중";
    case COMPLETED -> "완료";
    case CANCELLED -> "취소";
};
```

### Pattern Matching
```java
if (obj instanceof GameCreateDTO dto) {
    // dto 바로 사용
    return dto.getGameDate();
}
```

---

## 4. 의존성 관리

### build.gradle.kts 주요 의존성
```kotlin
dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")

    // MyBatis
    implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0+")

    // Database
    runtimeOnly("com.mysql:mysql-connector-j")

    // Swagger
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.x")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.mybatis.spring.boot:mybatis-spring-boot-starter-test:3.0+")
}
```

### 버전 관리 원칙
- Spring Boot BOM이 관리하는 의존성은 버전 명시 안 함
- 외부 라이브러리는 버전을 `gradle.properties` 또는 version catalog에서 관리
- SNAPSHOT 버전 사용 금지

---

## 5. 설정 파일

### application.yml 구조
```yaml
spring:
  profiles:
    active: local

  datasource:
    url: jdbc:mysql://localhost:3306/bcd?useSSL=false&serverTimezone=Asia/Seoul
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: validate        # 운영: none, 개발: validate
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
    open-in-view: false          # OSIV 끄기

mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.bcd.api
  configuration:
    map-underscore-to-camel-case: true
    default-fetch-size: 100
    default-statement-timeout: 30

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui
```

### 프로필 구성

| 프로필 | 용도 | ddl-auto |
|--------|------|----------|
| `local` | 로컬 개발 | `validate` |
| `dev` | 개발 서버 | `validate` |
| `prod` | 운영 서버 | `none` |

---

## 6. 프로젝트 구조

```
bcd-api/
├── src/
│   ├── main/
│   │   ├── java/com/bcd/api/
│   │   │   ├── BcdApiApplication.java
│   │   │   ├── team/
│   │   │   ├── member/
│   │   │   ├── game/
│   │   │   ├── squad/
│   │   │   ├── stats/
│   │   │   ├── dashboard/
│   │   │   └── common/
│   │   │       ├── config/
│   │   │       ├── exception/
│   │   │       ├── dto/
│   │   │       ├── entity/
│   │   │       └── util/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-local.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── mapper/
│   │           ├── game/
│   │           ├── stats/
│   │           └── ...
│   └── test/
│       └── java/com/bcd/api/
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── CLAUDE.md
└── docs/
    └── bcd-api-conventions/
```

---

## 7. 금지 사항

| 항목 | 이유 |
|------|------|
| `javax.*` import | Spring Boot 3.x는 `jakarta.*` |
| Java 8 스타일 (익명 클래스 등) | Java 17 기능 활용 |
| `application.properties` | yml 통일 |
| `spring.jpa.open-in-view: true` | 지연 로딩 이슈 |
| `ddl-auto: create/update` (운영) | 데이터 유실 위험 |
| SNAPSHOT 의존성 | 빌드 안정성 |