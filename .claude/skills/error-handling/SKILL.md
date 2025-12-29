---
name: error-handling
description: Паттерны обработки ошибок - исключения, ExceptionFactory, GlobalExceptionHandler, HTTP коды. Используй при создании/обработке ошибок.
---

# Error Handling Patterns

## Иерархия исключений

| Класс | HTTP код | Назначение |
|-------|----------|------------|
| `ResourceNotFoundException` | 404 | Ресурс не найден |
| `BadRequestException` | 400 | Некорректные данные запроса |
| `BusinessException` | 400 | Бизнес-логические ошибки |
| `AuthenticationException` | 401 | Ошибки аутентификации |
| `AuthorizationException` | 403 | Недостаток прав доступа |
| `DuplicateResourceException` | 409 | Нарушение уникальности |

## Структура классов исключений

```java
// Базовое бизнес-исключение
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Ресурс не найден (наследует BusinessException)
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// Остальные — наследуют RuntimeException
public class BadRequestException extends RuntimeException { ... }
public class AuthenticationException extends RuntimeException { ... }
public class AuthorizationException extends RuntimeException { ... }
public class DuplicateResourceException extends RuntimeException { ... }
```

## ExceptionFactory — утилита создания исключений

```java
public class ExceptionFactory {

    // ========== ResourceNotFoundException (404) ==========

    public static ResourceNotFoundException notFound(String entityName, Long id) {
        return new ResourceNotFoundException(entityName + " с ID " + id + " не найден(а)");
    }

    public static ResourceNotFoundException notFound(String message) {
        return new ResourceNotFoundException(message);
    }

    // ========== BusinessException (400) ==========

    public static BusinessException alreadyExists(String entityName, String field, String value) {
        return new BusinessException(entityName + " с " + field + " '" + value + "' уже существует");
    }

    public static BusinessException businessError(String message) {
        return new BusinessException(message);
    }

    // ========== BadRequestException (400) ==========

    public static BadRequestException badRequest(String message) {
        return new BadRequestException(message);
    }

    public static BadRequestException requiredFieldMissing(String fieldName) {
        return new BadRequestException("Обязательное поле '" + fieldName + "' отсутствует");
    }

    public static BadRequestException invalidField(String fieldName, String reason) {
        return new BadRequestException("Некорректное значение поля '" + fieldName + "': " + reason);
    }

    // ========== AuthenticationException (401) ==========

    public static AuthenticationException invalidCredentials() {
        return new AuthenticationException("Неверные учетные данные");
    }

    public static AuthenticationException tokenExpired() {
        return new AuthenticationException("Токен истек. Пожалуйста, войдите заново");
    }

    public static AuthenticationException invalidToken() {
        return new AuthenticationException("Невалидный токен");
    }

    // ========== AuthorizationException (403) ==========

    public static AuthorizationException accessDenied() {
        return new AuthorizationException("Доступ запрещен");
    }

    public static AuthorizationException insufficientPermissions(String resource) {
        return new AuthorizationException("Недостаточно прав для доступа к: " + resource);
    }
}
```

## Использование в сервисах

```java
// 1. Ресурс не найден → ResourceNotFoundException
User user = userRepository.findByIdAndDeletedFalse(id)
    .orElseThrow(() -> ExceptionFactory.notFound("Пользователь", id));

// 2. Бизнес-ошибка → BusinessException
if (tank.getCapacity().compareTo(requiredVolume) < 0) {
    throw new BusinessException("Недостаточно места в резервуаре");
}

// 3. Некорректный запрос → BadRequestException
if (request.getAmount() == null) {
    throw ExceptionFactory.requiredFieldMissing("amount");
}

// 4. Ошибка аутентификации → AuthenticationException
if (!passwordEncoder.matches(password, user.getPasswordHash())) {
    throw ExceptionFactory.invalidCredentials();
}

// 5. Ошибка авторизации → AuthorizationException
if (!AccessUtils.hasAccess(resourceId)) {
    throw ExceptionFactory.accessDenied();
}

// 6. Дублирование → DuplicateResourceException
if (repository.existsByCodeAndDeletedFalse(request.getCode())) {
    throw new DuplicateResourceException("Код уже используется: " + request.getCode());
}
```

## Структура JSON ответа об ошибке

```java
@Getter @Setter
@AllArgsConstructor @NoArgsConstructor @Builder
public class BaseResponse<T> {
    private boolean success;      // true = успех, false = ошибка
    private Object message;       // Сообщение об ошибке
    private T result;             // Данные (null при ошибке)

    @Builder.Default
    private String time = LocalDateTime.now()
        .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
}
```

**Пример ответа об ошибке:**
```json
{
  "success": false,
  "message": "Пользователь с ID 999 не найден(а)",
  "result": null,
  "time": "2025-01-15 14:30:45"
}
```

## GlobalExceptionHandler

```java
@RestControllerAdvice
@RequiredArgsConstructor
@Slf4j
public class GlobalExceptionHandler {

    private final LogService logService;  // Опционально

    // ========== Пользовательские исключения ==========

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(errorResponse(ex.getMessage()));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            BusinessException ex, HttpServletRequest request) {
        log.warn("Business error: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(errorResponse(ex.getMessage()));
    }

    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            BadRequestException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(errorResponse(ex.getMessage()));
    }

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            AuthenticationException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(errorResponse(ex.getMessage()));
    }

    @ExceptionHandler(AuthorizationException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            AuthorizationException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(errorResponse(ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            DuplicateResourceException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(errorResponse(ex.getMessage()));
    }

    // ========== Spring MVC ошибки ==========

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<BaseResponse<Void>> handle(MethodArgumentNotValidException ex) {
        String errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(errorResponse(errors));
    }

    // Неверный тип параметра (например, "abc" вместо числа)
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<BaseResponse<Void>> handle(MethodArgumentTypeMismatchException ex) {
        String message = "Некорректный параметр '" + ex.getName() + "': ожидается " +
            (ex.getRequiredType() != null ? ex.getRequiredType().getSimpleName() : "другой тип");
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(errorResponse(message));
    }

    // Неподдерживаемый HTTP метод (например, POST вместо GET)
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<BaseResponse<Void>> handle(HttpRequestMethodNotSupportedException ex) {
        String message = "Метод " + ex.getMethod() + " не поддерживается";
        return ResponseEntity.status(HttpStatus.METHOD_NOT_ALLOWED)
            .body(errorResponse(message));
    }

    // Неверный Content-Type (например, text/plain вместо application/json)
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    public ResponseEntity<BaseResponse<Void>> handle(HttpMediaTypeNotSupportedException ex) {
        String message = "Тип контента '" + ex.getContentType() + "' не поддерживается";
        return ResponseEntity.status(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
            .body(errorResponse(message));
    }

    // ========== JPA/Database ошибки ==========

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<BaseResponse<Void>> handle(
            DataIntegrityViolationException ex, HttpServletRequest request) {
        log.error("Data integrity violation: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(errorResponse("Нарушение целостности данных"));
    }

    // ========== Generic ошибки ==========

    @ExceptionHandler(Exception.class)
    public ResponseEntity<BaseResponse<Void>> handleGeneric(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(errorResponse("Внутренняя ошибка сервера"));
    }

    private BaseResponse<Void> errorResponse(String message) {
        return BaseResponse.<Void>builder()
            .success(false)
            .message(message)
            .build();
    }
}
```

## Таблица HTTP кодов

| Код | Статус | Исключения | Когда использовать |
|-----|--------|-----------|-------------------|
| **400** | Bad Request | BusinessException, BadRequestException, MethodArgumentTypeMismatchException | Ошибка в данных или бизнес-логике |
| **401** | Unauthorized | AuthenticationException | Неверный пароль, токен истек |
| **403** | Forbidden | AuthorizationException | Недостаточно прав |
| **404** | Not Found | ResourceNotFoundException | Ресурс не найден |
| **405** | Method Not Allowed | HttpRequestMethodNotSupportedException | Неверный HTTP метод |
| **409** | Conflict | DuplicateResourceException | Нарушение уникальности |
| **415** | Unsupported Media Type | HttpMediaTypeNotSupportedException | Неверный Content-Type |
| **500** | Internal Server Error | Exception | Непредвиденная ошибка |

## Правила

1. **Всегда использовать ExceptionFactory** для типовых ошибок
2. **Сообщения на русском языке** для бизнес-ошибок
3. **Не раскрывать детали** технических ошибок пользователю (500)
4. **Логировать все ошибки** в GlobalExceptionHandler
5. **Проверять доступ** через AccessUtils перед операциями
6. **Валидировать данные** на уровне DTO через @Valid

## Расположение файлов

```
{root.package}/
├── exception/
│   ├── ResourceNotFoundException.java
│   ├── BadRequestException.java
│   ├── BusinessException.java
│   ├── AuthenticationException.java
│   ├── AuthorizationException.java
│   └── DuplicateResourceException.java
├── util/
│   └── ExceptionFactory.java
├── config/
│   └── GlobalExceptionHandler.java
└── dto/common/
    └── BaseResponse.java
```
