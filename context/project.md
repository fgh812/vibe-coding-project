# BCD 농구 서비스

## 기술 설정

| 항목 | 값 |
|------|-----|
| DB | MySQL 8.x |
| PK 전략 | Long (AUTO_INCREMENT) |
| 테이블 네이밍 | snake_case, 단수형, 접두사 없음 |
| Base Package | com.bcd.api |
| 프로필 | local / dev / prod |

---

## 도메인 목록

| 도메인 | 영문 | 패키지 | 설명 |
|--------|------|--------|------|
| 팀 | Team | team | 팀(모임) 관리 |
| 멤버 | Member | member | 팀 소속 인원 |
| 경기 | Game | game | 경기 일정/기록 |
| 스쿼드 | Squad | squad | 경기 스쿼드 편성 |
| 통계 | Stats | stats | 선수/팀 통계 |
| 대시보드 | Dashboard | dashboard | 홈 대시보드 |

---

## 패키지 구조

```
com.bcd.api
├── team/
├── member/
├── game/
├── squad/
├── stats/
├── dashboard/
└── common/
    ├── config/
    ├── exception/
    ├── dto/
    ├── entity/
    └── util/
```

---

## Git Scope

| Scope | 도메인 |
|-------|--------|
| team | 팀 |
| member | 멤버 |
| game | 경기 |
| squad | 스쿼드 |
| stats | 통계 |
| dashboard | 대시보드 |
| common | 공통 |

---

## application.yml

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
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
    open-in-view: false

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

### 프로필별 설정

| 프로필 | 용도 | ddl-auto |
|--------|------|----------|
| local | 로컬 개발 | validate |
| dev | 개발 서버 | validate |
| prod | 운영 서버 | none |

---

## build.gradle.kts 주요 의존성

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
- 외부 라이브러리는 `gradle.properties` 또는 version catalog에서 관리
- SNAPSHOT 버전 사용 금지
