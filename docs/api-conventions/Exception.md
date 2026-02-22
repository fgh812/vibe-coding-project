# 예외 처리 규칙

## 개요
BCD API 프로젝트의 커스텀 예외 계층, 에러 응답 구조, GlobalExceptionHandler 규칙을 정의한다.

---

## 1. 예외 계층 구조

```
RuntimeException
└── BcdException (추상 - 프로젝트 공통 예외)
    ├── BcdNotFoundException         # 404 - 리소스 없음
    ├── BcdBadRequestException       # 400 - 잘못된 요청
    ├── BcdConflictException         # 409 - 중복/충돌
    ├── BcdForbiddenException        # 403 - 권한 없음
    └── BcdInternalException         # 500 - 내부 오류
```

---

## 2. 공통 예외 클래스

### BcdException (추상 부모)
```java
@Getter
public abstract class BcdException extends RuntimeException {

    private final String errorCode;
    private final HttpStatus httpStatus;

    protected BcdException(String errorCode, String message, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }
}
```

### 구체 예외 클래스

```java
// 404 Not Found
public class BcdNotFoundException extends BcdException {
    public BcdNotFoundException(String errorCode, String message) {
        super(errorCode, message, HttpStatus.NOT_FOUND);
    }
}

// 400 Bad Request
public class BcdBadRequestException extends BcdException {
    public BcdBadRequestException(String errorCode, String message) {
        super(errorCode, message, HttpStatus.BAD_REQUEST);
    }
}

// 409 Conflict
public class BcdConflictException extends BcdException {
    public BcdConflictException(String errorCode, String message) {
        super(errorCode, message, HttpStatus.CONFLICT);
    }
}

// 403 Forbidden
public class BcdForbiddenException extends BcdException {
    public BcdForbiddenException(String errorCode, String message) {
        super(errorCode, message, HttpStatus.FORBIDDEN);
    }
}

// 500 Internal Server Error
public class BcdInternalException extends BcdException {
    public BcdInternalException(String errorCode, String message) {
        super(errorCode, message, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

---

## 3. 에러 코드 상수

### ErrorCode Enum
```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    // 공통
    INVALID_INPUT("INVALID_INPUT", "잘못된 입력입니다.", HttpStatus.BAD_REQUEST),
    UNAUTHORIZED("UNAUTHORIZED", "인증이 필요합니다.", HttpStatus.UNAUTHORIZED),
    FORBIDDEN("FORBIDDEN", "권한이 없습니다.", HttpStatus.FORBIDDEN),
    INTERNAL_ERROR("INTERNAL_ERROR", "서버 내부 오류가 발생했습니다.", HttpStatus.INTERNAL_SERVER_ERROR),

    // Team
    TEAM_NOT_FOUND("TEAM_NOT_FOUND", "팀을 찾을 수 없습니다.", HttpStatus.NOT_FOUND),

    // Member
    MEMBER_NOT_FOUND("MEMBER_NOT_FOUND", "멤버를 찾을 수 없습니다.", HttpStatus.NOT_FOUND),
    MEMBER_ALREADY_EXISTS("MEMBER_ALREADY_EXISTS", "이미 가입된 멤버입니다.", HttpStatus.CONFLICT),

    // Game
    GAME_NOT_FOUND("GAME_NOT_FOUND", "경기를 찾을 수 없습니다.", HttpStatus.NOT_FOUND),
    GAME_ALREADY_EXISTS("GAME_ALREADY_EXISTS", "해당 날짜에 이미 경기가 존재합니다.", HttpStatus.CONFLICT),
    GAME_NOT_SCHEDULED("GAME_NOT_SCHEDULED", "예정 상태가 아닌 경기입니다.", HttpStatus.BAD_REQUEST),

    // Squad
    SQUAD_NOT_FORMED("SQUAD_NOT_FORMED", "스쿼드가 편성되지 않았습니다.", HttpStatus.BAD_REQUEST),
    SQUAD_INVALID_SIZE("SQUAD_INVALID_SIZE", "스쿼드 인원이 균형이 맞지 않습니다.", HttpStatus.BAD_REQUEST),

    // Attendance
    ATTENDANCE_ALREADY_RESPONDED("ATTENDANCE_ALREADY_RESPONDED", "이미 참석 응답을 했습니다.", HttpStatus.CONFLICT),

    // Stats
    STATS_NOT_FOUND("STATS_NOT_FOUND", "통계 데이터가 없습니다.", HttpStatus.NOT_FOUND);

    private final String code;
    private final String message;
    private final HttpStatus httpStatus;
}
```

---

## 4. Service에서 예외 발생

### 사용 패턴
```java
@Service
@RequiredArgsConstructor
public class GameService {

    private final GameRepository gameRepository;

    public GameDetailDTO getGameDetail(Long gameId) {
        Game game = gameRepository.findById(gameId)
                .orElseThrow(() -> new BcdNotFoundException(
                        ErrorCode.GAME_NOT_FOUND.getCode(),
                        ErrorCode.GAME_NOT_FOUND.getMessage()
                ));
        return GameDetailDTO.from(game);
    }

    @Transactional
    public Long createGame(GameCreateDTO dto) {
        // 중복 검증
        if (gameRepository.existsByTeamIdAndGameDate(dto.getTeamId(), dto.getGameDate())) {
            throw new BcdConflictException(
                    ErrorCode.GAME_ALREADY_EXISTS.getCode(),
                    ErrorCode.GAME_ALREADY_EXISTS.getMessage()
            );
        }

        Game game = Game.create(dto);
        return gameRepository.save(game).getId();
    }
}
```

### 헬퍼 메서드 (선택)
반복 코드를 줄이기 위한 유틸:
```java
public class ExceptionUtil {

    public static BcdNotFoundException notFound(ErrorCode errorCode) {
        return new BcdNotFoundException(errorCode.getCode(), errorCode.getMessage());
    }

    public static BcdConflictException conflict(ErrorCode errorCode) {
        return new BcdConflictException(errorCode.getCode(), errorCode.getMessage());
    }

    public static BcdBadRequestException badRequest(ErrorCode errorCode) {
        return new BcdBadRequestException(errorCode.getCode(), errorCode.getMessage());
    }
}

// 사용
Game game = gameRepository.findById(gameId)
        .orElseThrow(() -> ExceptionUtil.notFound(ErrorCode.GAME_NOT_FOUND));
```

---

## 5. GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // BCD 비즈니스 예외
    @ExceptionHandler(BcdException.class)
    public ResponseEntity<ApiResponse<?>> handleBcdException(BcdException e) {
        log.warn("Business exception: {} - {}", e.getErrorCode(), e.getMessage());
        return ResponseEntity
                .status(e.getHttpStatus())
                .body(ApiResponse.error(e.getErrorCode(), e.getMessage()));
    }

    // Bean Validation 예외 (@Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<?>> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.joining(", "));

        log.warn("Validation failed: {}", message);
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("INVALID_INPUT", message));
    }

    // 타입 불일치 (PathVariable, RequestParam)
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<ApiResponse<?>> handleTypeMismatch(MethodArgumentTypeMismatchException e) {
        log.warn("Type mismatch: {}", e.getMessage());
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("INVALID_INPUT", "잘못된 파라미터 형식입니다."));
    }

    // 기타 예외 (예상치 못한 오류)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleException(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("INTERNAL_ERROR", "서버 내부 오류가 발생했습니다."));
    }
}
```

---

## 6. 예외 처리 규칙

| 규칙 | 설명 |
|------|------|
| checked 예외 금지 | RuntimeException 기반만 사용 |
| Controller에서 try-catch 금지 | GlobalExceptionHandler가 처리 |
| Service에서 비즈니스 예외 발생 | 적절한 BcdException 하위 클래스 |
| 에러 로깅은 Handler에서 | Service에서 중복 로깅 금지 |
| 에러 코드는 ErrorCode enum 사용 | 하드코딩 금지 |
| 사용자에게 내부 정보 노출 금지 | 스택트레이스, SQL 에러 등 |

---

## 7. 패키지 구조

```
com.bcd.api.common.exception/
├── BcdException.java                  # 추상 부모
├── BcdNotFoundException.java          # 404
├── BcdBadRequestException.java        # 400
├── BcdConflictException.java          # 409
├── BcdForbiddenException.java         # 403
├── BcdInternalException.java          # 500
├── ErrorCode.java                     # 에러 코드 enum
├── ExceptionUtil.java                 # 헬퍼 (선택)
└── GlobalExceptionHandler.java        # 전역 핸들러
```