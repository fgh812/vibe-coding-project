# REST API 설계 규칙

## 개요
BCD API 프로젝트의 URL 네이밍, 요청/응답 포맷, 에러 코드, 페이징 규칙을 정의한다.

---

## 1. URL 네이밍

### 기본 규칙
- 소문자 + 하이픈(`kebab-case`)
- 복수형 명사 사용
- 동사 금지 (HTTP Method로 표현)
- 리소스 계층을 URL로 표현

### URL 패턴
```
/api/{리소스}                          # 컬렉션
/api/{리소스}/{id}                     # 단건
/api/{리소스}/{id}/{하위리소스}          # 하위 리소스 (소유 관계가 명확한 경우만)
```

> **Flat 원칙**: 각 리소스의 ID가 UUID로 유니크하므로, 상위 리소스 ID를 URL에 포함하지 않는다.
> 팀 소속 검증은 세션 기반으로 서비스 레이어에서 처리한다.

### 프로젝트별 API 목록

> 도메인별 구체적인 API URL은 `context/features/*.md`에 정의한다.
> 아래는 패턴 예시이다.

| Method | URL | 설명 |
|--------|-----|------|
| GET | `/api/{resources}` | 목록 조회 |
| POST | `/api/{resources}` | 생성 |
| GET | `/api/{resources}/{id}` | 단건 조회 |
| PUT | `/api/{resources}/{id}` | 수정 |
| DELETE | `/api/{resources}/{id}` | 삭제 |
| GET | `/api/{resources}/{id}/{sub-resources}` | 하위 리소스 조회 |

---

## 2. HTTP Method

| Method | 용도 | 멱등성 | 응답 코드 |
|--------|------|--------|-----------|
| GET | 조회 | O | 200 |
| POST | 생성 | X | 201 |
| PUT | 전체 수정 | O | 200 |
| PATCH | 부분 수정 | O | 200 |
| DELETE | 삭제 | O | 204 |

---

## 3. 공통 응답 포맷

### ApiResponse 래퍼

```java
@Getter
@AllArgsConstructor
public class ApiResponse<T> {
    private boolean success;
    private T data;
    private ErrorInfo error;

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null);
    }

    public static ApiResponse<?> error(String code, String message) {
        return new ApiResponse<>(false, null, new ErrorInfo(code, message));
    }

    @Getter
    @AllArgsConstructor
    public static class ErrorInfo {
        private String code;
        private String message;
    }
}
```

### 성공 응답
```json
// 단건 조회
{
  "success": true,
  "data": {
    "gameId": 1,
    "gameDate": "2025-03-16",
    "location": "OO체육관"
  },
  "error": null
}

// 목록 조회 (페이징)
{
  "success": true,
  "data": {
    "content": [...],
    "page": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  },
  "error": null
}

// 생성 성공
{
  "success": true,
  "data": 1,
  "error": null
}
```

### 에러 응답
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "GAME_NOT_FOUND",
    "message": "경기를 찾을 수 없습니다."
  }
}
```

---

## 4. 에러 코드 체계

### 코드 네이밍
`{도메인}_{에러유형}` — UPPER_SNAKE_CASE

### 공통 에러 코드

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| `INVALID_INPUT` | 400 | 요청 파라미터 유효성 실패 |
| `UNAUTHORIZED` | 401 | 인증 필요 |
| `FORBIDDEN` | 403 | 권한 없음 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

### 도메인 에러 코드

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| `TEAM_NOT_FOUND` | 404 | 팀을 찾을 수 없음 |
| `MEMBER_NOT_FOUND` | 404 | 멤버를 찾을 수 없음 |
| `MEMBER_ALREADY_EXISTS` | 409 | 이미 가입된 멤버 |
| `GAME_NOT_FOUND` | 404 | 경기를 찾을 수 없음 |
| `GAME_ALREADY_EXISTS` | 409 | 해당 날짜에 이미 경기 존재 |
| `GAME_NOT_SCHEDULED` | 400 | 예정 상태가 아닌 경기 |
| `SQUAD_NOT_FORMED` | 400 | 스쿼드 미편성 |
| `SQUAD_INVALID_SIZE` | 400 | 스쿼드 인원 불균형 |
| `ATTENDANCE_ALREADY_RESPONDED` | 409 | 이미 참석 응답함 |
| `STATS_NOT_FOUND` | 404 | 통계 데이터 없음 |

---

## 5. 페이징

### 요청 파라미터
```
GET /api/games?page=0&size=20&sort=gameDate,desc
```

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `page` | 0 | 페이지 번호 (0부터 시작) |
| `size` | 20 | 페이지 크기 |
| `sort` | - | 정렬 (필드,방향) |

### 응답 포맷
```json
{
  "success": true,
  "data": {
    "content": [
      { "gameId": 1, "gameDate": "2025-03-16" },
      { "gameId": 2, "gameDate": "2025-03-09" }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  },
  "error": null
}
```

### PageResponse DTO
```java
@Getter
@Builder
public class PageResponse<T> {
    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;

    public static <T> PageResponse<T> from(Page<T> page) {
        return PageResponse.<T>builder()
                .content(page.getContent())
                .page(page.getNumber())
                .size(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .build();
    }
}
```

---

## 6. 날짜/시간 포맷

| 타입 | 포맷 | 예시 |
|------|------|------|
| 날짜 | `yyyy-MM-dd` | `2025-03-16` |
| 시간 | `HH:mm:ss` | `14:00:00` |
| 일시 | `yyyy-MM-dd'T'HH:mm:ss` | `2025-03-16T14:00:00` |

### Jackson 설정
```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd'T'HH:mm:ss
    time-zone: Asia/Seoul
    serialization:
      write-dates-as-timestamps: false
```

---

## 7. API 문서화 (Swagger)

### Controller 어노테이션
```java
@Tag(name = "Game", description = "경기 관리 API")
@RestController
@RequestMapping("/api/games")
public class GameController {

    @Operation(summary = "경기 상세 조회")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "성공"),
        @ApiResponse(responseCode = "404", description = "경기 없음")
    })
    @GetMapping("/{gameId}")
    public ApiResponse<GameDetailDTO> getGameDetail(...) { }
}
```

### 필수 문서화 항목
- 모든 Controller에 `@Tag`
- 모든 API 메서드에 `@Operation(summary = "...")`
- 주요 에러 응답에 `@ApiResponse`
- 요청 DTO 필드에 `@Schema(description = "...")` (선택)

---

## 8. 버전 관리

현재 버전 관리 없이 `/api/...` 사용.
향후 필요시 `/api/v1/...`, `/api/v2/...` 형태로 확장.