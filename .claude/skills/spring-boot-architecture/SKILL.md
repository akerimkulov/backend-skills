---
name: spring-boot-architecture
description: Архитектура Spring Boot приложения - структура пакетов, базовые классы, конфигурация. Используй при создании нового сервиса или модуля.
---

# Архитектура Spring Boot Backend

## Структура пакетов

```
{root.package}
├── controller/           # REST контроллеры
│   └── hb/               # Справочники (handbooks)
├── service/              # Интерфейсы сервисов
│   ├── hb/               # Справочники
│   │   └── impl/         # Реализации
│   └── impl/             # Реализации операций
├── db/
│   ├── entity/           # JPA сущности
│   │   ├── base/         # Базовые классы (BaseEntity, AuditableEntity)
│   │   ├── hb/           # Справочники
│   │   └── sys/          # Системные (User, Role)
│   ├── repository/       # Spring Data репозитории
│   │   ├── hb/
│   │   └── specifications/  # JPA Specifications
│   └── enums/            # Перечисления
├── dto/
│   ├── common/           # Базовые DTO (BasePageRequest, BasePageResponse)
│   └── {domain}/         # По доменам
│       ├── filter/       # Фильтры
│       ├── request/      # Запросы
│       └── response/     # Ответы
├── mapper/               # MapStruct мапперы
├── config/               # Конфигурация Spring
├── exception/            # Исключения
├── util/                 # Утилиты
└── interceptor/          # HTTP интерцепторы
```

## Базовые классы

### BaseEntity
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @Column
    private LocalDateTime updatedAt;

    @Column
    private Long updatedBy;

    @Column(nullable = false)
    private Boolean deleted = false;

    public void markAsDeleted() {
        this.deleted = true;
    }
}
```

### BasePageRequest
```java
@Data
@SuperBuilder
@NoArgsConstructor
public class BasePageRequest {
    @Schema(description = "Номер страницы (1-based)", example = "1")
    @Builder.Default
    Integer page = 1;

    @Schema(description = "Размер страницы", example = "15")
    @Builder.Default
    Integer size = 15;
}
```

### BasePageResponse<T>
```java
@Data
@SuperBuilder
@NoArgsConstructor
@AllArgsConstructor
public class BasePageResponse<T> {
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private List<T> content;
}
```

### BaseController
```java
@Slf4j
public class BaseController {

    // Преобразование 1-based → 0-based для JPA
    public static Integer getPage(Integer page) {
        return (page != null && page > 0) ? page - 1 : 0;
    }

    protected <T> ResponseEntity<BaseResponse<T>> createSuccessResponse(T data) {
        return ResponseEntity.ok(BaseResponse.<T>builder()
            .success(true)
            .result(data)
            .build());
    }

    protected <T> ResponseEntity<BaseResponse<T>> createErrorResponse(Object message, HttpStatus status) {
        return ResponseEntity.status(status).body(BaseResponse.<T>builder()
            .success(false)
            .message(message)
            .build());
    }
}
```

## Утилиты аутентификации

```java
// Получить username текущего пользователя
String username = AuthenticationUtils.getCurrentUsername();

// Проверить аутентификацию
boolean isAuth = AuthenticationUtils.isAuthenticated();

// Получить текущего пользователя
User user = CurrentUserUtils.getCurrentUser();

// Получить доступные ID (multi-tenant)
List<Long> depotIds = CurrentUserUtils.getAccessibleDepotIds();

// Валидация доступа
AccessUtils.validateAccess(entityId);
List<Long> filtered = AccessUtils.applyFilter(requestIds);
```

## Исключения

| Исключение | HTTP Status | Использование |
|-----------|------------|---------------|
| ResourceNotFoundException | 404 | Сущность не найдена |
| BadRequestException | 400 | Некорректные данные |
| DuplicateResourceException | 409 | Дубликат (unique constraint) |
| AuthenticationException | 401 | Ошибка аутентификации |
| AuthorizationException | 403 | Нет прав доступа |
| BusinessException | 400 | Бизнес-логическая ошибка |

## Конфигурация

```properties
# Обязательные свойства
server.servlet.context-path=/api
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.default_schema={schema_name}

# JWT
jwt.secret=${JWT_SECRET}
jwt.expiration=432000000

# Audit
app.audit.enabled=true
```

## Зависимости (build.gradle)

```groovy
dependencies {
    // Spring Boot
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-security'

    // Database
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'

    // MapStruct
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

    // Swagger/OpenAPI
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'

    // JWT
    implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.3'
}
```

## Правила

```
✅ Разделять слои: Controller → Service → Repository
✅ Использовать constructor injection (@RequiredArgsConstructor)
✅ Валидация в DTO, не в Entity
✅ Soft delete вместо физического удаления
✅ Audit поля в каждой сущности (createdAt, createdBy, updatedAt, updatedBy, deleted)
✅ Multi-tenant изоляция через AccessUtils

❌ НЕ смешивать слои (Repository в Controller)
❌ НЕ использовать @Autowired на полях
❌ НЕ хранить бизнес-логику в Controller
❌ НЕ использовать физическое удаление
```
