---
name: fsd-backend
description: Generate backend code for Spring Boot projects. Creates controllers, services, entities, DTOs, mappers, and migrations. Just describe what you need - no architecture knowledge required.
---

# Spring Boot Backend Architecture Skill

## Simple User Commands

Users can request code generation with simple natural language:

- "Create a User entity with name, email, and role"
- "Create a Product with title, price, and category"
- "Create a controller for Orders"
- "Create a service for Payments"
- "Create a migration for products table"
- "Add filtering to Users"

**The skill handles all architecture decisions automatically.**

---

## Overview

This skill scaffolds Spring Boot backend architecture with layered design, JPA/Hibernate, MapStruct, and Flyway migrations.

## Package Structure

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

---

# Base Classes

## BaseEntity

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

## BasePageRequest

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

## BasePageResponse<T>

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

## BaseController

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

---

# REST Controller Patterns

## Mandatory Method Order

```
1. GET /{id}        - Получить по ID
2. POST /filter     - ЕДИНСТВЕННЫЙ метод поиска с пагинацией
3. POST             - Создать
4. PUT              - Обновить (id в теле запроса)
5. DELETE /{id}     - Удалить (soft delete)
```

## FORBIDDEN Endpoints

```
❌ GET /all           - использовать POST /filter
❌ GET /search        - использовать POST /filter
❌ GET /find-by-*     - использовать POST /filter
❌ GET /active        - использовать POST /filter с параметром
❌ POST /search       - использовать POST /filter
❌ GET /list          - использовать POST /filter
```

## Controller Template

```java
@RestController
@RequestMapping("${api.base-path}/domain-name")
@RequiredArgsConstructor
@SecurityRequirement(name = "bearerAuth")
@Tag(name = "КАТЕГОРИЯ: Название", description = "Описание API")
public class DomainController extends BaseController {

    private final DomainService service;

    // 1. GET BY ID
    @GetMapping("/{id}")
    @Operation(
        summary = "Получить по ID",
        responses = @ApiResponse(content = @Content(
            schema = @Schema(implementation = DomainResponse.class)))
    )
    public ResponseEntity<? extends BaseResponse<?>> findById(
            @Parameter(description = "ID", example = "1") @PathVariable Long id) {
        return createSuccessResponse(service.get(id));
    }

    // 2. FILTER (POST для сложных фильтров)
    @PostMapping("/filter")
    @Operation(
        summary = "Фильтрация с пагинацией",
        requestBody = @RequestBody(
            content = @Content(schema = @Schema(implementation = DomainFilter.class))
        ),
        responses = @ApiResponse(content = @Content(
            schema = @Schema(implementation = PageDomainResponse.class)))
    )
    public ResponseEntity<? extends BaseResponse<?>> findAllWithFilter(
            @org.springframework.web.bind.annotation.RequestBody(required = false)
            DomainFilter filter) {
        if (filter == null) {
            filter = new DomainFilter();
        }
        return createSuccessResponse(service.findAllWithFilter(filter));
    }

    // 3. CREATE
    @PostMapping
    @Operation(
        summary = "Создать",
        requestBody = @RequestBody(
            required = true,
            content = @Content(schema = @Schema(implementation = DomainRequest.class))
        ),
        responses = @ApiResponse(content = @Content(
            schema = @Schema(implementation = DomainResponse.class)))
    )
    public ResponseEntity<? extends BaseResponse<?>> create(
            @Valid @org.springframework.web.bind.annotation.RequestBody DomainRequest request,
            HttpServletRequest httpServletRequest) {
        return createSuccessResponse(service.create(request, httpServletRequest));
    }

    // 4. UPDATE
    @PutMapping
    @Operation(
        summary = "Обновить",
        requestBody = @RequestBody(
            required = true,
            content = @Content(schema = @Schema(implementation = DomainRequest.class))
        ),
        responses = @ApiResponse(content = @Content(
            schema = @Schema(implementation = DomainResponse.class)))
    )
    public ResponseEntity<? extends BaseResponse<?>> update(
            @Valid @org.springframework.web.bind.annotation.RequestBody DomainRequest request,
            HttpServletRequest httpServletRequest) {
        return createSuccessResponse(service.update(request, httpServletRequest));
    }

    // 5. DELETE
    @DeleteMapping("/{id}")
    @Operation(summary = "Удалить (soft delete)")
    public ResponseEntity<? extends BaseResponse<?>> delete(
            @PathVariable Long id,
            HttpServletRequest httpServletRequest) {
        service.delete(id, httpServletRequest);
        return createSuccessResponse(null);
    }
}
```

## Swagger @Tag Categories

```java
@Tag(name = "СПРАВОЧНИК: Название", description = "...")  // Справочники
@Tag(name = "ОПЕРАЦИИ: Название", description = "...")    // Операции
@Tag(name = "СИСТЕМА: Название", description = "...")     // Системные
@Tag(name = "МОНИТОРИНГ: Название", description = "...")  // Мониторинг
@Tag(name = "ОТЧЕТЫ: Название", description = "...")      // Отчеты
```

## @RequestBody Conflict Resolution

```java
// ВАЖНО: Два разных @RequestBody!

// 1. Swagger - для документации (в @Operation)
import io.swagger.v3.oas.annotations.parameters.RequestBody;

// 2. Spring - для binding (в параметрах метода)
import org.springframework.web.bind.annotation.RequestBody;

// Использование вместе:
@Operation(
    requestBody = @RequestBody(  // ← Swagger
        content = @Content(schema = @Schema(implementation = Request.class))
    )
)
public ResponseEntity<?> create(
    @Valid @org.springframework.web.bind.annotation.RequestBody Request request  // ← Spring (полный путь!)
) { ... }
```

---

# Service Implementation Patterns

## Mandatory Method Order

```
1. create()           - Создание
2. update()           - Обновление
3. delete()           - Удаление (soft delete)
4. get()              - Получить по ID → возвращает DTO
5. findById()         - Внутренний метод → возвращает Entity
6. findAllWithFilter() - ЕДИНСТВЕННЫЙ метод поиска
```

## FORBIDDEN Methods

```
❌ findAll()           - использовать findAllWithFilter()
❌ findByName()        - использовать findAllWithFilter()
❌ searchBy*()         - использовать findAllWithFilter()
❌ getAllActive()      - использовать findAllWithFilter() с фильтром
❌ getList()           - использовать findAllWithFilter()
```

## Service Interface Template

```java
public interface DomainService {

    DomainResponse create(DomainRequest request, HttpServletRequest httpServletRequest);

    DomainResponse update(DomainRequest request, HttpServletRequest httpServletRequest);

    void delete(Long id, HttpServletRequest httpServletRequest);

    DomainResponse get(Long id);

    Domain findById(Long id);  // Для внутреннего использования

    PageDomainResponse findAllWithFilter(DomainFilter filter);
}
```

## ServiceImpl Template

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DomainServiceImpl implements DomainService {

    // Dependencies
    private final DomainRepository repository;
    private final DomainMapper mapper;
    private final RelatedService relatedService;  // для связей
    private final SysLogsRequestService logService;  // для audit

    // ========== CREATE ==========
    @Override
    @Transactional
    public DomainResponse create(DomainRequest request, HttpServletRequest httpServletRequest) {
        // 1. Валидация уникальности
        if (repository.existsByCodeAndDeletedFalse(request.getCode())) {
            throw new DuplicateResourceException("Запись с кодом уже существует: " + request.getCode());
        }

        // 2. Маппинг Request → Entity
        Domain entity = mapper.toEntity(request);

        // 3. Установка связей (получаем из других сервисов)
        if (request.getRelatedId() != null) {
            entity.setRelated(relatedService.findById(request.getRelatedId()));
        }

        // 4. Сохранение
        Domain saved = repository.save(entity);

        // 5. Файловое логирование
        log.info("Created entity: id={}, code={}", saved.getId(), saved.getCode());

        // 6. Табличное логирование (audit)
        logService.saveSuccessToDb(
            this.getClass().getSimpleName(),
            "Created entity: id=" + saved.getId(),
            request,
            httpServletRequest
        );

        return mapper.toResponse(saved);
    }

    // ========== UPDATE ==========
    @Override
    @Transactional
    public DomainResponse update(DomainRequest request, HttpServletRequest httpServletRequest) {
        // 1. Получить существующую сущность
        Domain entity = findById(request.getId());

        // 2. Валидация уникальности при изменении
        if (!entity.getCode().equals(request.getCode()) &&
            repository.existsByCodeAndDeletedFalse(request.getCode())) {
            throw new DuplicateResourceException("Запись с кодом уже существует");
        }

        // 3. Обновление через mapper
        mapper.updateEntityFromRequest(entity, request);

        // 4. Обновление связей
        if (request.getRelatedId() != null) {
            entity.setRelated(relatedService.findById(request.getRelatedId()));
        }

        // 5. Установка audit полей
        entity.setUpdatedAt(LocalDateTime.now());

        // 6. Сохранение
        Domain updated = repository.save(entity);

        // 7. Логирование
        log.info("Updated entity: id={}", updated.getId());
        logService.saveSuccessToDb(
            this.getClass().getSimpleName(),
            "Updated entity: id=" + updated.getId(),
            request,
            httpServletRequest
        );

        return mapper.toResponse(updated);
    }

    // ========== DELETE ==========
    @Override
    @Transactional
    public void delete(Long id, HttpServletRequest httpServletRequest) {
        Domain entity = findById(id);

        // Soft delete
        entity.markAsDeleted();
        entity.setUpdatedAt(LocalDateTime.now());
        repository.save(entity);

        log.info("Deleted entity: id={}", id);

        // БЕЗ request DTO - 3 параметра
        logService.saveSuccessToDb(
            this.getClass().getSimpleName(),
            "Deleted entity: id=" + id,
            httpServletRequest
        );
    }

    // ========== GET (DTO) ==========
    @Override
    public DomainResponse get(Long id) {
        return mapper.toResponse(findById(id));
    }

    // ========== FIND BY ID (Entity) ==========
    @Override
    public Domain findById(Long id) {
        return repository.findByIdAndDeletedFalse(id)
            .orElseThrow(() -> new ResourceNotFoundException("Запись не найдена: " + id));
    }

    // ========== FILTER ==========
    @Override
    public PageDomainResponse findAllWithFilter(DomainFilter filter) {
        Pageable pageable = PageRequest.of(
            BaseController.getPage(filter.getPage()),
            filter.getSize() != null ? filter.getSize() : 15,
            Sort.by("createdAt").descending()
        );

        Specification<Domain> spec = DomainSpecifications.filterBy(filter);
        Page<Domain> page = repository.findAll(spec, pageable);

        return mapper.toPageResponse(page);
    }
}
```

---

# Entity & Repository Patterns

## Entity Template

```java
@Entity
@Table(name = "table_name")  // БЕЗ schema! Schema из application.properties
@Comment("Описание таблицы")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Domain extends AuditableEntity {

    // Простые поля
    @Column(nullable = false, length = 50)
    private String code;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(length = 500)
    private String description;

    // Числовые с точностью
    @Column(nullable = false, precision = 18, scale = 8)
    private BigDecimal amount;

    @Column(precision = 9, scale = 6)
    private BigDecimal density;

    // Enum
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private StatusEnum status;

    // Связь Many-to-One (LAZY!)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "related_id", nullable = false)
    private Related related;

    // Связь One-to-Many
    @OneToMany(mappedBy = "domain", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<Child> children = new ArrayList<>();

    // Default значения
    @Column(nullable = false)
    @Builder.Default
    private Long counter = 0L;
}
```

## Entity Rules

```
✅ extends AuditableEntity     - наследует id, audit поля, deleted
✅ @Table(name = "...")        - БЕЗ schema параметра
✅ @ManyToOne(fetch = LAZY)    - всегда LAZY для оптимизации
✅ @Builder.Default            - для значений по умолчанию
✅ Precision для BigDecimal    - (18,8) для сумм, (9,6) для плотности
✅ @Comment("...")             - описание таблицы

❌ НЕ указывать schema в @Table
❌ НЕ использовать валидацию (@NotNull, @Size) в Entity - только в DTO
❌ НЕ добавлять JavaDoc комментарии
❌ НЕ использовать @Data - только @Getter @Setter
```

## Precision for BigDecimal

| Использование | Precision | Пример |
|--------------|-----------|--------|
| Объемы, массы, суммы | (18,8) | 9999999999.99999999 |
| Плотность | (9,6) | 999.999999 |
| Температура | (5,2) | 999.99 |
| Координаты | (10,7) | 180.1234567 |
| Проценты | (5,2) | 100.00 |

## Repository Template

```java
@Repository
public interface DomainRepository extends
        JpaRepository<Domain, Long>,
        JpaSpecificationExecutor<Domain> {

    // Soft delete lookup (обязательный)
    Optional<Domain> findByIdAndDeletedFalse(Long id);

    // Проверка существования
    boolean existsByCodeAndDeletedFalse(String code);

    // Поиск по уникальному полю
    Optional<Domain> findByCodeAndDeletedFalse(String code);

    // Список по IDs
    List<Domain> findByIdInAndDeletedFalse(List<Long> ids);

    // С JOIN FETCH для избежания LazyInitializationException
    @Query("SELECT d FROM Domain d LEFT JOIN FETCH d.related WHERE d.id = :id AND d.deleted = false")
    Optional<Domain> findByIdWithRelatedAndDeletedFalse(@Param("id") Long id);

    // Кастомный запрос
    @Query("SELECT d FROM Domain d WHERE d.status = :status AND d.deleted = false ORDER BY d.name")
    List<Domain> findActiveByStatus(@Param("status") StatusEnum status);

    // Подсчет
    long countByDeletedFalse();
}
```

## Specifications Template

```java
public class DomainSpecifications {

    // Обязательный базовый фильтр
    public static Specification<Domain> notDeleted() {
        return (root, query, cb) -> cb.isFalse(root.get("deleted"));
    }

    // Текстовое поле (LIKE, case-insensitive)
    public static Specification<Domain> byName(String name) {
        return (root, query, cb) -> {
            if (name == null || name.isBlank()) {
                return cb.conjunction();  // нет условия
            }
            return cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%");
        };
    }

    // Точное совпадение
    public static Specification<Domain> byStatus(StatusEnum status) {
        return (root, query, cb) -> {
            if (status == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("status"), status);
        };
    }

    // Список ID (IN)
    public static Specification<Domain> byIds(List<Long> ids) {
        return (root, query, cb) -> {
            if (ids == null || ids.isEmpty()) {
                return cb.conjunction();
            }
            return root.get("id").in(ids);
        };
    }

    // Связанная сущность
    public static Specification<Domain> byRelatedId(Long relatedId) {
        return (root, query, cb) -> {
            if (relatedId == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("related").get("id"), relatedId);
        };
    }

    // Диапазон дат
    public static Specification<Domain> byDateRange(LocalDateTime from, LocalDateTime to) {
        return (root, query, cb) -> {
            if (from == null && to == null) {
                return cb.conjunction();
            }
            if (from != null && to != null) {
                return cb.between(root.get("createdAt"), from, to);
            }
            if (from != null) {
                return cb.greaterThanOrEqualTo(root.get("createdAt"), from);
            }
            return cb.lessThanOrEqualTo(root.get("createdAt"), to);
        };
    }

    // Главный метод - комбинация через .and()
    public static Specification<Domain> filterBy(DomainFilter filter) {
        Specification<Domain> spec = notDeleted();

        if (filter == null) {
            return spec;
        }

        return spec
            .and(byIds(filter.getIds()))
            .and(byName(filter.getName()))
            .and(byStatus(filter.getStatus()))
            .and(byRelatedId(filter.getRelatedId()))
            .and(byDateRange(filter.getCreatedAtFrom(), filter.getCreatedAtTo()));
    }
}
```

## Specifications Rules

```
✅ Всегда начинать с notDeleted()
✅ Использовать цепочку .and()
✅ Возвращать cb.conjunction() для null/empty значений
✅ isBlank() для строк
✅ Название главного метода: filterBy()

❌ НЕ использовать List<Predicate> + stream().reduce()
❌ НЕ использовать Specification.where()
❌ НЕ называть withFilter(), byFilter()
```

---

# DTO Patterns

## DTO Types

| Тип | Назначение | Наследование |
|-----|-----------|--------------|
| Filter | Поиск + пагинация | extends BasePageRequest |
| Request | Создание/обновление | - |
| Response | Ответ API | - |
| PageResponse | Список с пагинацией | extends BasePageResponse<Response> |

## Filter DTO

```java
@Schema(description = "Фильтр для поиска")
@Data
@EqualsAndHashCode(callSuper = true)
public class DomainFilter extends BasePageRequest {

    @ArraySchema(schema = @Schema(description = "Список ID", example = "[1, 2, 3]"))
    private List<Long> ids;

    @Schema(description = "Название (частичное совпадение)", example = "текст")
    private String name;

    @Schema(description = "Код", example = "CODE")
    private String code;

    @Schema(description = "ID связанной сущности", example = "1")
    private Long relatedId;

    @Schema(description = "Статус", example = "ACTIVE")
    private StatusEnum status;

    @Schema(description = "Активность", example = "true")
    private Boolean isActive;

    @Schema(description = "Дата создания с", example = "2024-01-01T00:00:00")
    private LocalDateTime createdAtFrom;

    @Schema(description = "Дата создания до", example = "2024-12-31T23:59:59")
    private LocalDateTime createdAtTo;
}
```

## Request DTO

```java
@Schema(description = "Запрос на создание/обновление")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainRequest {

    @Schema(description = "ID (null для создания, заполнен для обновления)", example = "1")
    private Long id;

    @Schema(description = "Код", example = "CODE", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank(message = "Код обязателен")
    @Size(max = 50, message = "Код не должен превышать 50 символов")
    private String code;

    @Schema(description = "Название", example = "Название", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank(message = "Название обязательно")
    @Size(max = 100, message = "Название не должно превышать 100 символов")
    private String name;

    @Schema(description = "ID связанной сущности", example = "1", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotNull(message = "Связанная сущность обязательна")
    private Long relatedId;  // ID, не полный объект!

    @Schema(description = "Сумма", example = "1000.50")
    @NotNull(message = "Сумма обязательна")
    @Positive(message = "Сумма должна быть положительной")
    @Digits(integer = 10, fraction = 2, message = "Неверный формат суммы")
    private BigDecimal amount;

    @Schema(description = "Описание", example = "Текст описания")
    @Size(max = 500, message = "Описание не должно превышать 500 символов")
    private String description;
}
```

## Response DTO

```java
@Schema(description = "Ответ с данными")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainResponse {

    @Schema(description = "ID", example = "1")
    private Long id;

    @Schema(description = "Код", example = "CODE")
    private String code;

    @Schema(description = "Название", example = "Название")
    private String name;

    // Связанный объект - ПОЛНЫЙ Response DTO, не ID!
    @Schema(description = "Связанная сущность", implementation = RelatedResponse.class)
    private RelatedResponse related;

    @Schema(description = "Сумма", example = "1000.50")
    private BigDecimal amount;

    @Schema(description = "Описание")
    private String description;

    // Audit поля - только createdAt, updatedAt
    @Schema(description = "Дата создания", example = "2024-01-15T10:30:00")
    private LocalDateTime createdAt;

    @Schema(description = "Дата обновления", example = "2024-02-20T14:45:00")
    private LocalDateTime updatedAt;

    // ❌ НЕ включать: createdBy, updatedBy, deleted
}
```

## PageResponse DTO

```java
@Setter
@Getter
public class PageDomainResponse extends BasePageResponse<DomainResponse> {

    // Опциональная аналитика
    @Schema(description = "Аналитические данные")
    private Analytics analytics;

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Analytics {
        @Schema(description = "Общее количество", example = "100")
        private Integer total;

        @Schema(description = "Активных", example = "85")
        private Integer active;
    }
}
```

## Response DTO Rules

```
✅ ВКЛЮЧАТЬ:
   - id
   - Все бизнес-поля
   - createdAt, updatedAt
   - Связанные объекты как ПОЛНЫЕ Response DTO

❌ НЕ ВКЛЮЧАТЬ:
   - createdBy, updatedBy (служебные)
   - deleted (служебное)

❌ НЕ РАЗВОРАЧИВАТЬ связи:
   ❌ relatedId, relatedName, relatedCode  // отдельные поля
   ✅ related: RelatedResponse             // полный объект
```

## Validation Annotations

```java
// Строки
@NotBlank(message = "Поле обязательно")
@Size(max = 100, message = "Не должно превышать 100 символов")
@Pattern(regexp = "^[A-Z0-9]+$", message = "Только заглавные буквы и цифры")

// Числа
@NotNull(message = "Поле обязательно")
@Positive(message = "Должно быть положительным")
@PositiveOrZero(message = "Должно быть >= 0")
@Min(value = 1, message = "Минимум 1")
@Max(value = 100, message = "Максимум 100")
@Digits(integer = 10, fraction = 2, message = "Неверный формат")

// Даты
@NotNull(message = "Дата обязательна")
@Past(message = "Должна быть в прошлом")
@Future(message = "Должна быть в будущем")

// Коллекции
@NotEmpty(message = "Список не может быть пустым")
@Size(min = 1, max = 10, message = "От 1 до 10 элементов")

// Email
@Email(message = "Неверный формат email")
```

---

# Swagger/OpenAPI Documentation Patterns

## Enum Documentation

### Enum Template

```java
@Schema(description = "Статус записи")
@Getter
@RequiredArgsConstructor
public enum StatusEnum {
    @Schema(description = "Активная запись")
    ACTIVE("Активный"),

    @Schema(description = "Неактивная запись")
    INACTIVE("Неактивный"),

    @Schema(description = "В ожидании подтверждения")
    PENDING("Ожидание"),

    @Schema(description = "Запись отменена")
    CANCELLED("Отменён");

    private final String displayName;
}
```

### Enum в DTO

```java
@Schema(description = "Статус",
        example = "ACTIVE",
        allowableValues = {"ACTIVE", "INACTIVE", "PENDING", "CANCELLED"})
private StatusEnum status;
```

## DTO Class Documentation

### Request DTO — полный пример

```java
@Schema(description = "Запрос на создание/обновление",
        requiredProperties = {"code", "name"})
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainRequest {

    @Schema(description = "ID (null для создания, заполнен для обновления)",
            example = "1",
            nullable = true)
    private Long id;

    @Schema(description = "Код",
            example = "CODE_001",
            requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank(message = "Код обязателен")
    @Size(max = 50, message = "Код не должен превышать 50 символов")
    private String code;

    @Schema(description = "Название",
            example = "Название записи",
            requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank(message = "Название обязательно")
    @Size(max = 100, message = "Название не должно превышать 100 символов")
    private String name;

    @Schema(description = "ID связанной сущности",
            example = "1",
            requiredMode = Schema.RequiredMode.REQUIRED)
    @NotNull(message = "Связанная сущность обязательна")
    private Long relatedId;

    @Schema(description = "Сумма",
            example = "1000.50",
            minimum = "0.01")
    @NotNull(message = "Сумма обязательна")
    @Positive(message = "Сумма должна быть положительной")
    @Digits(integer = 10, fraction = 2, message = "Неверный формат суммы")
    private BigDecimal amount;

    @Schema(description = "Описание",
            example = "Текст описания",
            maxLength = 500)
    @Size(max = 500, message = "Описание не должно превышать 500 символов")
    private String description;
}
```

### Response DTO с вложенными объектами

```java
@Schema(description = "Ответ с данными")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainResponse {

    @Schema(description = "ID", example = "1")
    private Long id;

    @Schema(description = "Код", example = "CODE_001")
    private String code;

    @Schema(description = "Название", example = "Название записи")
    private String name;

    // Связанный объект — ПОЛНЫЙ Response DTO
    @Schema(description = "Связанная сущность",
            implementation = RelatedResponse.class)
    private RelatedResponse related;

    // Список объектов
    @Schema(description = "Список элементов",
            implementation = ItemResponse.class)
    private List<ItemResponse> items;

    @Schema(description = "Статус",
            example = "ACTIVE",
            allowableValues = {"ACTIVE", "INACTIVE", "PENDING", "CANCELLED"})
    private StatusEnum status;

    @Schema(description = "Сумма",
            example = "1000.50",
            format = "decimal")
    private BigDecimal amount;

    @Schema(description = "Дата создания",
            example = "2024-01-15T10:30:00")
    private LocalDateTime createdAt;

    @Schema(description = "Дата обновления",
            example = "2024-02-20T14:45:00")
    private LocalDateTime updatedAt;
}
```

## Controller Documentation

### Полный пример контроллера с документацией

```java
@RestController
@RequestMapping("${api.base-path}/domain-name")
@RequiredArgsConstructor
@SecurityRequirement(name = "bearerAuth")
@Tag(name = "ОПЕРАЦИИ: Название",
     description = "API для управления: создание, обновление, фильтрация, удаление")
public class DomainController extends BaseController {

    private final DomainService service;

    @GetMapping("/{id}")
    @Operation(
        summary = "Получить по ID",
        description = "Возвращает полную информацию включая связанные сущности",
        responses = {
            @ApiResponse(responseCode = "200", description = "Успешно",
                content = @Content(schema = @Schema(implementation = DomainResponse.class))),
            @ApiResponse(responseCode = "404", description = "Не найдено",
                content = @Content(schema = @Schema(implementation = BaseResponse.class)))
        }
    )
    public ResponseEntity<? extends BaseResponse<?>> findById(
            @Parameter(description = "ID записи", example = "1", required = true)
            @PathVariable Long id) {
        return createSuccessResponse(service.get(id));
    }

    @PostMapping("/filter")
    @Operation(
        summary = "Фильтрация с пагинацией",
        description = "Поиск по критериям с пагинацией. Все параметры опциональны.",
        requestBody = @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "Параметры фильтрации",
            content = @Content(schema = @Schema(implementation = DomainFilter.class))
        ),
        responses = {
            @ApiResponse(responseCode = "200", description = "Успешно",
                content = @Content(schema = @Schema(implementation = PageDomainResponse.class)))
        }
    )
    public ResponseEntity<? extends BaseResponse<?>> filter(
            @org.springframework.web.bind.annotation.RequestBody(required = false)
            DomainFilter filter) {
        return createSuccessResponse(service.findAllWithFilter(
            filter != null ? filter : new DomainFilter()));
    }

    @PostMapping
    @Operation(
        summary = "Создать",
        description = "Создание новой записи",
        requestBody = @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = @Content(schema = @Schema(implementation = DomainRequest.class))
        ),
        responses = {
            @ApiResponse(responseCode = "200", description = "Успешно создано",
                content = @Content(schema = @Schema(implementation = DomainResponse.class))),
            @ApiResponse(responseCode = "400", description = "Ошибка валидации",
                content = @Content(schema = @Schema(implementation = BaseResponse.class))),
            @ApiResponse(responseCode = "409", description = "Дублирование",
                content = @Content(schema = @Schema(implementation = BaseResponse.class)))
        }
    )
    public ResponseEntity<? extends BaseResponse<?>> create(
            @Valid @org.springframework.web.bind.annotation.RequestBody DomainRequest request,
            HttpServletRequest httpServletRequest) {
        return createSuccessResponse(service.create(request, httpServletRequest));
    }
}
```

## Swagger Annotations Cheatsheet

| Аннотация | Уровень | Назначение |
|-----------|---------|------------|
| `@Tag` | Класс | Группировка endpoints |
| `@Operation` | Метод | Описание endpoint |
| `@Parameter` | Параметр | Описание path/query параметра |
| `@RequestBody` | Метод | Описание тела запроса (Swagger) |
| `@ApiResponse` | Метод | Описание возможных ответов |
| `@Schema` | Класс/Поле | Описание модели/поля |
| `@ArraySchema` | Поле | Описание массива |
| `@SecurityRequirement` | Класс | Требование авторизации |

## Swagger Rules

```
✅ @Schema(description, example) на КАЖДОМ поле DTO
✅ @Schema на каждом значении Enum с описанием
✅ allowableValues для Enum полей в DTO
✅ implementation для вложенных объектов
✅ requiredMode = REQUIRED для обязательных полей
✅ @Tag с категорией (СПРАВОЧНИК/ОПЕРАЦИИ/СИСТЕМА) на контроллере
✅ @Operation(summary, description) на каждом endpoint
✅ @ApiResponse для 200, 400, 404, 409 где применимо

❌ НЕ оставлять поля без @Schema
❌ НЕ оставлять Enum без описаний значений
❌ НЕ забывать example для примитивных типов
```

---

# MapStruct Mapper Patterns

## Mapper Template

```java
@Mapper(componentModel = "spring", uses = {RelatedMapper.class})
public interface DomainMapper {

    // 1. REQUEST → ENTITY (для create)
    @Mapping(target = "related", ignore = true)  // связи в Service!
    Domain toEntity(DomainRequest request);

    // 2. ENTITY → RESPONSE
    DomainResponse toResponse(Domain entity);

    // 3. PAGE → PAGE RESPONSE (default метод)
    default PageDomainResponse toPageResponse(Page<Domain> page) {
        PageDomainResponse response = new PageDomainResponse();
        response.setContent(page.getContent().stream()
            .map(this::toResponse)
            .collect(Collectors.toList()));
        response.setTotalElements(page.getTotalElements());
        response.setTotalPages(page.getTotalPages());
        response.setPage(page.getNumber() + 1);  // 0-based → 1-based
        response.setSize(page.getSize());
        return response;
    }

    // 4. UPDATE ENTITY (для update)
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "related", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    void updateEntityFromRequest(@MappingTarget Domain entity, DomainRequest request);
}
```

## 4 Mandatory Methods

| Метод | Назначение | Возвращает |
|-------|-----------|------------|
| toEntity(Request) | Request → Entity | Entity |
| toResponse(Entity) | Entity → Response | Response |
| toPageResponse(Page) | Page → PageResponse | PageResponse |
| updateEntityFromRequest() | Update Entity in-place | void |

## updateEntityFromRequest() - ALWAYS Ignore

```java
@BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
@Mapping(target = "id", ignore = true)           // никогда не менять ID
@Mapping(target = "related", ignore = true)      // связи через Service
@Mapping(target = "createdAt", ignore = true)    // audit
@Mapping(target = "createdBy", ignore = true)    // audit
@Mapping(target = "updatedAt", ignore = true)    // audit (устанавливается в Service)
@Mapping(target = "updatedBy", ignore = true)    // audit
@Mapping(target = "deleted", ignore = true)      // soft delete
```

## Mapper Prohibitions

```
❌ Repository в маппере
❌ @Component класс вместо @Mapper interface
❌ Бизнес-логика в маппере
❌ Запросы к БД в маппере
❌ Установка связей в маппере

✅ Связи устанавливаются ТОЛЬКО в Service:
   entity.setRelated(relatedService.findById(request.getRelatedId()));
```

---

# Error Handling Patterns

## Exception Hierarchy

| Класс | HTTP код | Назначение |
|-------|----------|------------|
| `ResourceNotFoundException` | 404 | Ресурс не найден |
| `BadRequestException` | 400 | Некорректные данные запроса |
| `BusinessException` | 400 | Бизнес-логические ошибки |
| `AuthenticationException` | 401 | Ошибки аутентификации |
| `AuthorizationException` | 403 | Недостаток прав доступа |
| `DuplicateResourceException` | 409 | Нарушение уникальности |

## ExceptionFactory Usage

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

## BaseResponse Structure

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

## HTTP Status Codes Table

| Код | Статус | Исключения | Когда использовать |
|-----|--------|-----------|-------------------|
| **400** | Bad Request | BusinessException, BadRequestException | Ошибка в данных или бизнес-логике |
| **401** | Unauthorized | AuthenticationException | Неверный пароль, токен истек |
| **403** | Forbidden | AuthorizationException | Недостаточно прав |
| **404** | Not Found | ResourceNotFoundException | Ресурс не найден |
| **405** | Method Not Allowed | HttpRequestMethodNotSupportedException | Неверный HTTP метод |
| **409** | Conflict | DuplicateResourceException | Нарушение уникальности |
| **500** | Internal Server Error | Exception | Непредвиденная ошибка |

---

# Flyway Migration Patterns

## File Naming Format

```
V{version}__{Description}.sql

Examples:
V001__Create_Tables.sql
V002__Insert_Reference_Data.sql
V015__Add_New_Module.sql
V016__Alter_Table_Add_Column.sql
```

## Table Prefixes

| Префикс | Назначение | Примеры |
|---------|-----------|---------|
| hb_* | Справочники (handbook) | hb_categories, hb_types |
| sys_* | Системные | sys_users, sys_roles, sys_audit_log |
| (без) | Операционные | orders, transactions, history |

## Table Creation Template

```sql
CREATE TABLE IF NOT EXISTS hb_domains (
    -- Primary key
    id BIGSERIAL PRIMARY KEY,

    -- Unique fields
    code VARCHAR(50) NOT NULL UNIQUE,

    -- Required fields
    name VARCHAR(100) NOT NULL,

    -- Optional fields
    description VARCHAR(500),

    -- Numeric with precision
    amount DECIMAL(18,8) NOT NULL,
    density DECIMAL(9,6),

    -- Enum as string
    status VARCHAR(20) NOT NULL CHECK (status IN ('ACTIVE', 'INACTIVE')),

    -- Foreign keys
    category_id BIGINT NOT NULL REFERENCES hb_categories(id),

    -- Dates
    operation_date DATE NOT NULL,
    operation_time TIME,

    -- Audit fields (обязательные)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by BIGINT,
    deleted BOOLEAN DEFAULT FALSE
);
```

## Data Types Mapping

| Java Type | PostgreSQL | Примечание |
|-----------|-----------|-----------|
| Long | BIGINT / BIGSERIAL | BIGSERIAL для PK |
| Integer | INTEGER | - |
| String | VARCHAR(N) | N по размеру данных |
| String (большой) | TEXT | Неограниченный |
| BigDecimal | DECIMAL(P,S) | P-precision, S-scale |
| LocalDate | DATE | Только дата |
| LocalTime | TIME | Только время |
| LocalDateTime | TIMESTAMP | Дата+время |
| Boolean | BOOLEAN | DEFAULT FALSE |
| Enum | VARCHAR(N) + CHECK | С ограничением |
| byte[] | BYTEA | Бинарные данные |

## Mandatory Indexes

```sql
-- На каждую таблицу:
CREATE INDEX IF NOT EXISTS idx_table_deleted ON table_name(deleted);

-- На каждый FK:
CREATE INDEX IF NOT EXISTS idx_table_fk_column ON table_name(fk_column_id);

-- На часто фильтруемые поля:
CREATE INDEX IF NOT EXISTS idx_table_status ON table_name(status);
CREATE INDEX IF NOT EXISTS idx_table_date ON table_name(operation_date DESC);
```

## Table and Column Comments

### Comment Template

```sql
-- Комментарии к таблице
COMMENT ON TABLE hb_domains IS 'Справочник доменов';

-- Комментарии к колонкам
COMMENT ON COLUMN hb_domains.id IS 'Уникальный идентификатор';
COMMENT ON COLUMN hb_domains.code IS 'Код записи (уникальный)';
COMMENT ON COLUMN hb_domains.name IS 'Наименование';
COMMENT ON COLUMN hb_domains.description IS 'Описание';
COMMENT ON COLUMN hb_domains.amount IS 'Сумма';
COMMENT ON COLUMN hb_domains.category_id IS 'ID категории (FK)';
COMMENT ON COLUMN hb_domains.status IS 'Статус записи (ACTIVE, INACTIVE)';
COMMENT ON COLUMN hb_domains.created_at IS 'Дата и время создания записи';
COMMENT ON COLUMN hb_domains.created_by IS 'ID пользователя, создавшего запись';
COMMENT ON COLUMN hb_domains.updated_at IS 'Дата и время последнего обновления';
COMMENT ON COLUMN hb_domains.updated_by IS 'ID пользователя, обновившего запись';
COMMENT ON COLUMN hb_domains.deleted IS 'Признак мягкого удаления';
```

### Full Migration Example

```sql
-- =====================================================
-- V001__Create_Domains_Table.sql
-- Справочник доменов
-- =====================================================

CREATE TABLE IF NOT EXISTS hb_domains (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    description VARCHAR(500),
    amount DECIMAL(18,8),
    category_id BIGINT NOT NULL REFERENCES hb_categories(id),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE')),

    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by BIGINT,
    deleted BOOLEAN DEFAULT FALSE
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_hb_domains_deleted ON hb_domains(deleted);
CREATE INDEX IF NOT EXISTS idx_hb_domains_code ON hb_domains(code);
CREATE INDEX IF NOT EXISTS idx_hb_domains_category ON hb_domains(category_id);
CREATE INDEX IF NOT EXISTS idx_hb_domains_status ON hb_domains(status);

-- Table comment
COMMENT ON TABLE hb_domains IS 'Справочник доменов';

-- Column comments
COMMENT ON COLUMN hb_domains.id IS 'Уникальный идентификатор';
COMMENT ON COLUMN hb_domains.code IS 'Код записи (уникальный)';
COMMENT ON COLUMN hb_domains.name IS 'Наименование';
COMMENT ON COLUMN hb_domains.description IS 'Описание';
COMMENT ON COLUMN hb_domains.amount IS 'Сумма';
COMMENT ON COLUMN hb_domains.category_id IS 'ID категории (FK)';
COMMENT ON COLUMN hb_domains.status IS 'Статус записи (ACTIVE, INACTIVE)';
COMMENT ON COLUMN hb_domains.created_at IS 'Дата и время создания записи';
COMMENT ON COLUMN hb_domains.created_by IS 'ID пользователя, создавшего запись';
COMMENT ON COLUMN hb_domains.updated_at IS 'Дата и время последнего обновления';
COMMENT ON COLUMN hb_domains.updated_by IS 'ID пользователя, обновившего запись';
COMMENT ON COLUMN hb_domains.deleted IS 'Признак мягкого удаления';

-- Initial data (if needed)
INSERT INTO hb_domains (code, name, description, category_id, status)
VALUES
    ('DEFAULT', 'По умолчанию', 'Запись по умолчанию', 1, 'ACTIVE')
ON CONFLICT (code) DO NOTHING;
```

## Migration Structure Order

```
1. CREATE TABLE (поля в порядке: PK → unique → required → optional → FK → audit)
2. CREATE INDEX (deleted → FK → фильтруемые поля)
3. COMMENT ON TABLE
4. COMMENT ON COLUMN (в порядке полей таблицы)
5. INSERT начальных данных (если нужно)
```

## Flyway Rules

```
✅ IF NOT EXISTS для таблиц и индексов
✅ ON CONFLICT DO NOTHING для INSERT
✅ BIGSERIAL для PRIMARY KEY
✅ DEFAULT FALSE для deleted
✅ DEFAULT CURRENT_TIMESTAMP для created_at
✅ COMMENT ON TABLE и COMMENT ON COLUMN для документации
✅ Комментарии на русском
✅ Индекс на каждый FK
✅ Индекс на deleted поле

❌ НЕ использовать SERIAL (только BIGSERIAL)
❌ НЕ забывать deleted поле
❌ НЕ забывать audit поля (created_at, created_by, updated_at, updated_by)
❌ НЕ использовать CASCADE DELETE (soft delete)
❌ НЕ забывать COMMENT ON TABLE и COMMENT ON COLUMN
```

## Naming Conventions

```
Таблицы:        snake_case, множественное число (hb_fuel_types, sys_users)
Колонки:        snake_case (created_at, fuel_type_id)
Индексы:        idx_{table}_{column} (idx_orders_status)
Unique:         uk_{table}_{columns} (uk_users_username)
FK constraint:  fk_{table}_{ref_table} (fk_orders_users)
Check:          chk_{table}_{column} (chk_orders_status)
```

---

# Logging Patterns

## Two-Level Logging System

| Уровень | Назначение | Хранение | Поиск |
|---------|------------|----------|-------|
| **Файловое** (Logback + @Slf4j) | Все события приложения | 30 дней | grep, ELK |
| **Табличное** (LogService) | Бизнес-операции | 90 дней | SQL запросы |

## Mandatory Class Components

```java
@Service
@RequiredArgsConstructor
@Slf4j                                    // Файловое логирование
public class DomainServiceImpl implements DomainService {

    private final LogService logService;  // Табличное логирование (опционально)

    // ...
}
```

## Logging Levels

```java
// DEBUG - детальная отладочная информация
log.debug("Finding entity by id={}", id);

// INFO - важные события (успешные операции)
log.info("Entity created: id={}, code={}", id, code);

// WARN - потенциальные проблемы
log.warn("Task {} rejected, queue is full", taskName);

// ERROR - серьёзные ошибки
log.error("Failed to save: {}", exception.getMessage());
log.error("Database connection failed", exception);
```

## Pattern: FILE → DB

**ВАЖНО:** Порядок логирования: СНАЧАЛА файл, ПОТОМ БД (асинхронно)

```java
@Override
@Transactional
public DomainResponse create(DomainRequest request, HttpServletRequest httpRequest) {
    // ... создание сущности ...

    Domain saved = repository.save(entity);

    // 1. ФАЙЛОВОЕ логирование (синхронно, надёжно)
    log.info("Created entity: id={}, code={}", saved.getId(), saved.getCode());

    // 2. ТАБЛИЧНОЕ логирование (асинхронно, для анализа) — опционально
    logService.saveSuccess(
        this.getClass().getSimpleName(),           // className
        "Created entity: id=" + saved.getId(),     // message
        request,                                   // requestDto (→ JSON)
        httpRequest                                // для IP, User-Agent
    );

    return mapper.toResponse(saved);
}
```

## Logging Rules

```
1. @Slf4j обязателен на всех Service классах
2. LogService для CUD операций (опционально)
3. ФАЙЛ → БД — порядок логирования
4. Асинхронность — табличные логи не блокируют основной поток
5. Плейсхолдеры — использовать {}, не конкатенацию строк
6. INFO для успеха — log.info() после успешной операции
7. ERROR для ошибок — log.error() с exception объектом
```

---

# Authentication Utilities

```java
// Получить username текущего пользователя
String username = AuthenticationUtils.getCurrentUsername();

// Проверить аутентификацию
boolean isAuth = AuthenticationUtils.isAuthenticated();

// Получить текущего пользователя
User user = CurrentUserUtils.getCurrentUser();

// Получить доступные ID (multi-tenant)
List<Long> organizationIds = CurrentUserUtils.getAccessibleOrganizationIds();

// Валидация доступа
AccessUtils.validateOrganizationAccess(organizationId);
AccessUtils.validateResourceAccess(entity.getOrganizationId());
List<Long> filtered = AccessUtils.applyOrganizationFilter(requestIds);
```

---

# Security & Authorization

## Архитектура безопасности

Авторизация реализуется на **уровне Service**, а не через аннотации на контроллерах:

| Уровень | Что проверяется |
|---------|----------------|
| SecurityConfig | Публичные endpoints, базовая аутентификация |
| JwtAuthenticationFilter | Валидация токена, загрузка ролей |
| Service Layer | Бизнес-логика доступа, multi-tenant фильтрация |
| Utility Classes | Проверка ролей, получение текущего пользователя |

## Database Schema

### sys_users Table

```sql
CREATE TABLE IF NOT EXISTS sys_users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE,
    full_name VARCHAR(200),
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMP,

    -- Multi-tenant (опционально)
    organization_id BIGINT REFERENCES hb_organizations(id),
    owner_id BIGINT,

    -- Audit
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by BIGINT,
    deleted BOOLEAN DEFAULT FALSE
);

CREATE INDEX IF NOT EXISTS idx_sys_users_username ON sys_users(username);
CREATE INDEX IF NOT EXISTS idx_sys_users_deleted ON sys_users(deleted);
CREATE INDEX IF NOT EXISTS idx_sys_users_organization ON sys_users(organization_id);

COMMENT ON TABLE sys_users IS 'Системные пользователи';
COMMENT ON COLUMN sys_users.organization_id IS 'ID организации для multi-tenant фильтрации';
COMMENT ON COLUMN sys_users.owner_id IS 'ID владельца для дополнительной фильтрации (READONLY роль)';
```

### sys_roles Table

```sql
CREATE TABLE IF NOT EXISTS sys_roles (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    description VARCHAR(500),
    priority INTEGER NOT NULL DEFAULT 100,
    is_system BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by BIGINT,
    deleted BOOLEAN DEFAULT FALSE
);

CREATE INDEX IF NOT EXISTS idx_sys_roles_code ON sys_roles(code);
CREATE INDEX IF NOT EXISTS idx_sys_roles_deleted ON sys_roles(deleted);

COMMENT ON TABLE sys_roles IS 'Справочник ролей';
COMMENT ON COLUMN sys_roles.priority IS 'Приоритет роли (меньше = выше полномочия)';
COMMENT ON COLUMN sys_roles.is_system IS 'Системная роль (нельзя удалить/изменить код)';

-- Базовые роли с иерархией приоритетов
INSERT INTO sys_roles (code, name, description, priority, is_system) VALUES
    ('SUPERADMIN', 'Суперадминистратор', 'Полный доступ к системе', 1, true),
    ('ADMIN', 'Администратор', 'Администрирование', 10, true),
    ('MANAGER', 'Менеджер', 'Управление операциями', 20, false),
    ('OPERATOR', 'Оператор', 'Выполнение операций', 30, false),
    ('AUDITOR', 'Аудитор', 'Аудит и контроль', 35, false),
    ('READONLY', 'Только просмотр', 'Просмотр данных', 40, false)
ON CONFLICT (code) DO NOTHING;
```

### sys_user_roles Table

```sql
CREATE TABLE IF NOT EXISTS sys_user_roles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES sys_users(id),
    role_id BIGINT NOT NULL REFERENCES sys_roles(id),
    is_active BOOLEAN DEFAULT TRUE,
    assigned_by BIGINT REFERENCES sys_users(id),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,

    UNIQUE(user_id, role_id)
);

CREATE INDEX IF NOT EXISTS idx_sys_user_roles_user ON sys_user_roles(user_id);
CREATE INDEX IF NOT EXISTS idx_sys_user_roles_role ON sys_user_roles(role_id);
CREATE INDEX IF NOT EXISTS idx_sys_user_roles_active ON sys_user_roles(is_active);

COMMENT ON TABLE sys_user_roles IS 'Связь пользователей и ролей (M:N)';
COMMENT ON COLUMN sys_user_roles.is_active IS 'Активна ли роль у пользователя';
COMMENT ON COLUMN sys_user_roles.assigned_by IS 'Кто назначил роль';
```

## User Entity

```java
@Entity
@Table(name = "sys_users")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
@Builder
public class User extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    @Column(length = 100)
    private String email;

    @Column(name = "full_name", length = 200)
    private String fullName;

    @Column(name = "is_active")
    @Builder.Default
    private Boolean isActive = true;

    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;

    // Multi-tenant
    @Column(name = "organization_id")
    private Long organizationId;

    @Column(name = "owner_id")
    private Long ownerId;

    // Связь с ролями
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @Builder.Default
    private List<UserRole> userRoles = new ArrayList<>();

    // ========== Helper Methods ==========

    public boolean hasRole(String roleCode) {
        return userRoles.stream()
            .filter(ur -> ur.getIsActive() && !ur.getRole().getDeleted())
            .anyMatch(ur -> ur.getRole().getCode().equals(roleCode));
    }

    public boolean hasAnyRole(String... roleCodes) {
        Set<String> codes = Set.of(roleCodes);
        return userRoles.stream()
            .filter(ur -> ur.getIsActive() && !ur.getRole().getDeleted())
            .anyMatch(ur -> codes.contains(ur.getRole().getCode()));
    }

    public List<String> getActiveRoleCodes() {
        return userRoles.stream()
            .filter(ur -> ur.getIsActive() && !ur.getRole().getDeleted())
            .map(ur -> ur.getRole().getCode())
            .toList();
    }

    public List<Role> getActiveRoles() {
        return userRoles.stream()
            .filter(ur -> ur.getIsActive() && !ur.getRole().getDeleted())
            .map(UserRole::getRole)
            .toList();
    }
}
```

## Role Entity

```java
@Entity
@Table(name = "sys_roles")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
@Builder
public class Role extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String code;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(length = 500)
    private String description;

    @Column(nullable = false)
    @Builder.Default
    private Integer priority = 100;

    @Column(name = "is_system")
    @Builder.Default
    private Boolean isSystem = false;
}
```

## SecurityConfig (минимальный)

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Public
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

                // Admin only для actuator
                .requestMatchers("/actuator/**").hasRole("ADMIN")

                // Всё остальное — просто authenticated (логика в Service)
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

## JwtAuthenticationFilter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtils jwtUtils;
    private final UserRepository userRepository;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = parseJwt(request);

            if (jwt != null && jwtUtils.validateJwtToken(jwt)) {
                String username = jwtUtils.getUsernameFromJwtToken(jwt);

                // Проверка активности пользователя
                if (!userRepository.existsByUsernameAndIsActiveTrueAndDeletedFalse(username)) {
                    log.warn("User is not active: {}", username);
                    filterChain.doFilter(request, response);
                    return;
                }

                // Загрузка ролей отдельным запросом (избегаем LazyInitializationException)
                List<String> roleCodes = userRepository.findActiveRoleCodesByUsername(username);

                List<SimpleGrantedAuthority> authorities = roleCodes.stream()
                    .map(code -> new SimpleGrantedAuthority("ROLE_" + code))
                    .toList();

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(username, null, authorities);
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            log.error("Cannot set user authentication: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }

    private String parseJwt(HttpServletRequest request) {
        String headerAuth = request.getHeader("Authorization");
        if (StringUtils.hasText(headerAuth) && headerAuth.startsWith("Bearer ")) {
            return headerAuth.substring(7);
        }
        return null;
    }
}
```

## Security Utilities

### SecurityUtils — проверка ролей

```java
@Component
@RequiredArgsConstructor
public class SecurityUtils {

    public boolean hasAnyRole(String... roles) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return false;
        }

        Set<String> userRoles = auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .map(r -> r.replace("ROLE_", ""))
            .collect(Collectors.toSet());

        return Arrays.stream(roles).anyMatch(userRoles::contains);
    }

    public boolean isSuperAdmin() {
        return hasAnyRole("SUPERADMIN");
    }

    public boolean isAdmin() {
        return hasAnyRole("SUPERADMIN", "ADMIN");
    }

    public boolean isManager() {
        return hasAnyRole("SUPERADMIN", "ADMIN", "MANAGER");
    }

    public boolean isOperator() {
        return hasAnyRole("SUPERADMIN", "ADMIN", "MANAGER", "OPERATOR");
    }

    public boolean isReadOnly() {
        return hasAnyRole("READONLY");
    }
}
```

### CurrentUserUtils — получение текущего пользователя

```java
@Component
@RequiredArgsConstructor
public class CurrentUserUtils {

    private final UserRepository userRepository;

    public User getCurrentUser() {
        String username = getCurrentUsername();
        return userRepository.findByUsernameAndDeletedFalse(username)
            .orElseThrow(() -> new AuthenticationException("Пользователь не найден"));
    }

    public String getCurrentUsername() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            throw new AuthenticationException("Пользователь не аутентифицирован");
        }
        return auth.getName();
    }

    public Long getCurrentUserId() {
        return getCurrentUser().getId();
    }

    /**
     * Возвращает список ID организаций, к которым у пользователя есть доступ.
     * Пустой список означает доступ ко всем организациям.
     */
    public List<Long> getAccessibleOrganizationIds() {
        User user = getCurrentUser();

        // SUPERADMIN/ADMIN — доступ ко всем
        if (user.hasAnyRole("SUPERADMIN", "ADMIN")) {
            return Collections.emptyList(); // пустой = все
        }

        // Остальные — только своя организация
        if (user.getOrganizationId() != null) {
            return List.of(user.getOrganizationId());
        }

        return Collections.emptyList();
    }
}
```

### AccessUtils — multi-tenant фильтрация

```java
@Component
@RequiredArgsConstructor
public class AccessUtils {

    private final CurrentUserUtils currentUserUtils;
    private final SecurityUtils securityUtils;

    /**
     * Применяет фильтр по организации к списку ID.
     * Возвращает пересечение запрошенных ID и доступных пользователю.
     */
    public List<Long> applyOrganizationFilter(List<Long> requestedIds) {
        List<Long> accessibleIds = currentUserUtils.getAccessibleOrganizationIds();

        // Пустой список = доступ ко всем
        if (accessibleIds.isEmpty()) {
            return requestedIds;
        }

        // Если не запросили конкретные — возвращаем доступные
        if (requestedIds == null || requestedIds.isEmpty()) {
            return accessibleIds;
        }

        // Пересечение
        return requestedIds.stream()
            .filter(accessibleIds::contains)
            .toList();
    }

    /**
     * Проверяет доступ к конкретной организации.
     */
    public void validateOrganizationAccess(Long organizationId) {
        List<Long> accessibleIds = currentUserUtils.getAccessibleOrganizationIds();

        if (accessibleIds.isEmpty()) {
            return; // доступ ко всем
        }

        if (!accessibleIds.contains(organizationId)) {
            throw new AuthorizationException("Нет доступа к организации: " + organizationId);
        }
    }

    /**
     * Проверяет доступ к ресурсу по его organizationId.
     */
    public void validateResourceAccess(Long resourceOrganizationId) {
        validateOrganizationAccess(resourceOrganizationId);
    }

    /**
     * Проверяет, может ли текущий пользователь выполнять операцию.
     */
    public void validateOperation(String operation) {
        switch (operation) {
            case "CREATE", "UPDATE" -> {
                if (!securityUtils.isOperator()) {
                    throw new AuthorizationException("Недостаточно прав для операции: " + operation);
                }
            }
            case "DELETE" -> {
                if (!securityUtils.isManager()) {
                    throw new AuthorizationException("Недостаточно прав для удаления");
                }
            }
        }
    }
}
```

## RoleHierarchyValidator — валидация иерархии ролей

```java
@Component
@RequiredArgsConstructor
public class RoleHierarchyValidator {

    private final RoleRepository roleRepository;

    /**
     * Проверяет, может ли текущий пользователь назначить указанную роль.
     * Правило: можно назначать только роли с БОЛЬШИМ priority (ниже в иерархии).
     */
    public void validateRoleAssignment(User currentUser, Role roleToAssign) {
        Role highestRole = getHighestPriorityRole(currentUser);

        if (highestRole == null) {
            throw new AuthorizationException("У пользователя нет активных ролей");
        }

        // SUPERADMIN может назначать любые роли
        if ("SUPERADMIN".equals(highestRole.getCode())) {
            return;
        }

        // Остальные могут назначать только роли ниже своей
        if (roleToAssign.getPriority() <= highestRole.getPriority()) {
            throw new RoleHierarchyViolationException(
                String.format("Нельзя назначить роль %s (priority=%d). " +
                    "Ваша роль %s (priority=%d) не позволяет это.",
                    roleToAssign.getCode(), roleToAssign.getPriority(),
                    highestRole.getCode(), highestRole.getPriority())
            );
        }
    }

    /**
     * Возвращает роль с наименьшим priority (высшие полномочия).
     */
    public Role getHighestPriorityRole(User user) {
        return user.getActiveRoles().stream()
            .min(Comparator.comparing(Role::getPriority))
            .orElse(null);
    }

    /**
     * Проверяет, может ли пользователь управлять другими пользователями.
     */
    public boolean canManageUsers(User user) {
        return user.hasAnyRole("SUPERADMIN", "ADMIN", "MANAGER");
    }
}
```

## Service-Level Authorization Pattern

### Контроллер (БЕЗ аннотаций безопасности)

```java
@RestController
@RequestMapping("${api.base-path}/domain-name")
@RequiredArgsConstructor
@Tag(name = "ОПЕРАЦИИ: Название")
public class DomainController extends BaseController {

    private final DomainService service;

    @PostMapping
    public ResponseEntity<? extends BaseResponse<?>> create(
            @Valid @RequestBody DomainRequest request,
            HttpServletRequest httpServletRequest) {
        // Авторизация внутри сервиса
        return createSuccessResponse(service.create(request, httpServletRequest));
    }

    @GetMapping("/{id}")
    public ResponseEntity<? extends BaseResponse<?>> findById(@PathVariable Long id) {
        // Авторизация внутри сервиса
        return createSuccessResponse(service.get(id));
    }

    @PostMapping("/filter")
    public ResponseEntity<? extends BaseResponse<?>> filter(
            @RequestBody(required = false) DomainFilter filter) {
        // Автоматическая фильтрация по организациям внутри сервиса
        return createSuccessResponse(service.findAllWithFilter(
            filter != null ? filter : new DomainFilter()));
    }
}
```

### Service с проверкой доступа

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DomainServiceImpl implements DomainService {

    private final DomainRepository repository;
    private final DomainMapper mapper;
    private final CurrentUserUtils currentUserUtils;
    private final AccessUtils accessUtils;
    private final SecurityUtils securityUtils;

    @Override
    @Transactional
    public DomainResponse create(DomainRequest request, HttpServletRequest httpServletRequest) {
        // 1. Проверка прав на операцию
        accessUtils.validateOperation("CREATE");

        // 2. Проверка доступа к организации
        accessUtils.validateOrganizationAccess(request.getOrganizationId());

        // 3. Бизнес-логика создания
        Domain entity = mapper.toEntity(request);
        Domain saved = repository.save(entity);

        log.info("Entity created: id={}, by={}", saved.getId(), currentUserUtils.getCurrentUsername());

        return mapper.toResponse(saved);
    }

    @Override
    public DomainResponse get(Long id) {
        Domain entity = findById(id);

        // Проверка доступа к организации сущности
        accessUtils.validateResourceAccess(entity.getOrganizationId());

        return mapper.toResponse(entity);
    }

    @Override
    public PageDomainResponse findAllWithFilter(DomainFilter filter) {
        // Автоматическая фильтрация по доступным организациям
        List<Long> orgIds = accessUtils.applyOrganizationFilter(filter.getOrganizationIds());
        filter.setOrganizationIds(orgIds.isEmpty() ? null : orgIds);

        // Для READONLY — дополнительная фильтрация по owner
        if (securityUtils.isReadOnly()) {
            User user = currentUserUtils.getCurrentUser();
            if (user.getOwnerId() != null) {
                filter.setOwnerId(user.getOwnerId());
            }
        }

        Pageable pageable = PageRequest.of(
            BaseController.getPage(filter.getPage()),
            filter.getSize() != null ? filter.getSize() : 15,
            Sort.by("createdAt").descending()
        );

        Specification<Domain> spec = DomainSpecifications.filterBy(filter);
        Page<Domain> page = repository.findAll(spec, pageable);

        return mapper.toPageResponse(page);
    }

    @Override
    @Transactional
    public void delete(Long id, HttpServletRequest httpServletRequest) {
        // Проверка прав на удаление
        accessUtils.validateOperation("DELETE");

        Domain entity = findById(id);
        accessUtils.validateResourceAccess(entity.getOrganizationId());

        entity.markAsDeleted();
        repository.save(entity);

        log.info("Entity deleted: id={}, by={}", id, currentUserUtils.getCurrentUsername());
    }
}
```

## Security Rules (Service-Level подход)

```
✅ SecurityConfig — минимальный (только public endpoints и authenticated)
✅ Авторизация в Service через AccessUtils, SecurityUtils
✅ Автоматическая multi-tenant фильтрация в findAllWithFilter()
✅ RoleHierarchyValidator для назначения ролей
✅ Username в JWT (не user ID) — лучше для аудита
✅ Отдельный запрос для ролей в JwtFilter (избегаем LazyInitializationException)

❌ НЕ использовать @PreAuthorize на контроллерах
❌ НЕ хардкодить роли в бизнес-логике (использовать SecurityUtils)
❌ НЕ забывать validateResourceAccess() для single-entity операций
❌ НЕ забывать applyOrganizationFilter() для list операций
```

## Сравнение подходов

| Аспект | @PreAuthorize | Service-Level |
|--------|---------------|---------------|
| Место проверки | Контроллер | Сервис |
| Гибкость | Средняя | Высокая |
| Multi-tenant | Сложно | Естественно |
| Тестирование | Mock Security Context | Mock Utils |
| Читаемость | Аннотации | Явный код |
| Рекомендация | Простые проекты | Enterprise/Multi-tenant |

---

# Configuration

## application.properties

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

## build.gradle Dependencies

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

## HikariCP Connection Pool

### Production Profile (application-prod.yml)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000        # 5 min
      connection-timeout: 20000   # 20 sec
      max-lifetime: 1200000       # 20 min
      validation-timeout: 5000    # 5 sec
      leak-detection-threshold: 60000  # 60 sec
```

### Dev / Stage Profile (application-dev.yml, application-stage.yml)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      idle-timeout: 120000        # 2 min
      connection-timeout: 10000   # 10 sec
      max-lifetime: 600000        # 10 min
      validation-timeout: 3000    # 3 sec
      leak-detection-threshold: 30000  # 30 sec
```

### HikariCP Rules

```
✅ Prod и dev/stage в РАЗНЫХ профилях (application-prod.yml, application-dev.yml)
✅ maximum-pool-size = кол-во ядер × 2 + кол-во дисков (формула HikariCP)
✅ max-lifetime < PostgreSQL idle_in_transaction_session_timeout
✅ leak-detection-threshold > самый долгий запрос
✅ validation-timeout < connection-timeout

❌ НЕ настраивать HikariCP в основном application.yml (только в профилях)
❌ НЕ ставить maximum-pool-size > 30 (PostgreSQL default max_connections = 100)
❌ НЕ ставить minimum-idle = maximum-pool-size (нет смысла в пуле)
```

---

# Spring Actuator Configuration

## Dependencies

```groovy
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

## application.yml Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      show-components: when_authorized
      probes:
        enabled: true
  health:
    db:
      enabled: true
    diskspace:
      enabled: true
  info:
    env:
      enabled: true

info:
  app:
    name: ${spring.application.name}
    version: @project.version@
    description: Service description
```

## Security Configuration for Actuator

```java
// В SecurityConfig добавить:
.authorizeHttpRequests(auth -> auth
    // Actuator endpoints
    .requestMatchers("/actuator/health/**").permitAll()
    .requestMatchers("/actuator/info").permitAll()
    .requestMatchers("/actuator/prometheus").permitAll()
    .requestMatchers("/actuator/**").hasRole("ADMIN")
    // ... остальные правила
)
```

## Custom Health Indicator

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.down().build();
    }
}
```

## Actuator Endpoints Reference

| Endpoint | Назначение | Доступ |
|----------|-----------|--------|
| `/actuator/health` | Статус приложения | Public |
| `/actuator/health/liveness` | Kubernetes liveness probe | Public |
| `/actuator/health/readiness` | Kubernetes readiness probe | Public |
| `/actuator/info` | Информация о приложении | Public |
| `/actuator/metrics` | Метрики приложения | Admin |
| `/actuator/prometheus` | Метрики для Prometheus | Public/Internal |

## Actuator Rules

```
✅ health и info — публичные (для мониторинга)
✅ metrics, env, beans — только для ADMIN
✅ show-details: when_authorized (не выдавать детали анонимам)
✅ probes.enabled: true для Kubernetes
✅ Использовать в Docker healthcheck

❌ НЕ открывать /actuator/env публично (содержит секреты)
❌ НЕ открывать /actuator/beans публично (раскрывает структуру)
❌ НЕ использовать include: "*" в production
```

---

# Docker Configuration

## Dockerfile Template

```dockerfile
FROM gradle:8.5.0-jdk21 AS build

WORKDIR /app

COPY build.gradle settings.gradle ./

RUN gradle --no-daemon dependencies

COPY src/ src/

RUN gradle --no-daemon bootJar -x test

FROM eclipse-temurin:21-jre-jammy
WORKDIR /app

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

RUN addgroup --gid 1000 spring && \
    adduser --uid 1000 --gid 1000 --disabled-password --gecos "" spring

RUN mkdir -p /app/logs && \
    chown -R spring:spring /app

COPY --from=build /app/build/libs/*.jar app.jar

USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## docker-compose.yml Template

```yaml
services:
  service-name:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: service-name
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE:-prod}
      - JAVA_TOOL_OPTIONS=${JAVA_TOOL_OPTIONS:--XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:InitialRAMPercentage=50.0 -Dfile.encoding=UTF-8}
      - TZ=Asia/Bishkek
    ports:
      - "${APP_PORT:-8080}:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: ${MEMORY_LIMIT:-900M}
        reservations:
          memory: ${MEMORY_RESERVATION:-450M}
```

## .env.example Template

```bash
# =============================================================================
# Service Name - Environment Configuration
# Copy this file to .env and fill in actual values
# =============================================================================
#
# SECURITY NOTICE:
# - NEVER commit .env file to version control (.env is in .gitignore)
# - Use strong, unique passwords for all secrets (min 12 chars, mixed case, digits, special chars)
# - Rotate secrets periodically
# - For production: consider using Docker Secrets, HashiCorp Vault, or cloud secret managers

# Database
# For production with TLS: jdbc:postgresql://host:5432/db?sslmode=require
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mydb
DB_USERNAME=postgres
DB_PASSWORD=  # Required. Use a strong password (min 12 chars)

# Security
JWT_SECRET=  # Required. Min 256-bit (32+ chars), e.g.: openssl rand -base64 32

# Application
SPRING_PROFILES_ACTIVE=prod
APP_PORT=8080

# JVM (optional, auto-picked by JVM via JAVA_TOOL_OPTIONS)
# JAVA_TOOL_OPTIONS=-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:InitialRAMPercentage=50.0 -Dfile.encoding=UTF-8

# Docker Resources (optional)
# MEMORY_LIMIT=900M
# MEMORY_RESERVATION=450M
```

## JVM Container Configuration

```
JAVA_TOOL_OPTIONS=-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:InitialRAMPercentage=50.0 -Dfile.encoding=UTF-8
```

| Флаг | Назначение |
|------|-----------|
| `+UseContainerSupport` | JVM видит лимиты контейнера, а не хоста |
| `MaxRAMPercentage=75.0` | JVM берёт 75% от memory limit контейнера |
| `+UseG1GC` | G1 сборщик мусора (оптимален для контейнеров) |
| `InitialRAMPercentage=50.0` | Начальный heap = 50% от лимита |
| `Dfile.encoding=UTF-8` | Кодировка по умолчанию |

### Рекомендуемые Memory Limits

| Тип сервиса | Memory Limit | Memory Reservation | JVM MaxRAM (75%) |
|-------------|-------------|-------------------|------------------|
| Микросервис | 512M | 256M | ~384M |
| Стандартный | 768M–900M | 384M–450M | ~576M–675M |
| Тяжёлый (AI, отчёты) | 1.5G | 768M | ~1.1G |

## Docker Rules

```
✅ Multi-stage build: gradle → eclipse-temurin:*-jre-jammy
✅ Non-root user (spring:spring, UID/GID 1000)
✅ Exec-form ENTRYPOINT: ["java", "-jar", "app.jar"]
✅ JAVA_TOOL_OPTIONS вместо JAVA_OPTS (авто-подхват JVM, не нужен shell)
✅ env_file: .env для секретов
✅ environment: только для несекретных (SPRING_PROFILES_ACTIVE, TZ, JAVA_TOOL_OPTIONS)
✅ healthcheck на /api/actuator/health
✅ .env в .gitignore
✅ .env.example с SECURITY NOTICE и требованиями к паролям

❌ НЕ использовать shell-form ENTRYPOINT (command injection risk)
❌ НЕ передавать секреты через environment: в docker-compose
❌ НЕ использовать JAVA_OPTS (требует shell для подстановки)
❌ НЕ коммитить .env файл
❌ НЕ хардкодить секреты в application.yml (использовать ${ENV_VAR:})
```

## application.yml — Secrets Pattern

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:mydb}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:}

jwt:
  secret: ${JWT_SECRET:}
```

```
✅ Все секреты через ${ENV_VAR:default}
✅ Пустой default для обязательных секретов (приложение упадёт без них)
✅ Разумный default для несекретных (localhost, 5432, postgres)

❌ НЕ хардкодить реальные пароли, токены, ключи
❌ НЕ ставить дефолтные JWT секреты для production
```

---

# Global Rules

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

---

# Quick Reference: Generate Entity

When asked to create a new entity, generate these files:

1. `db/entity/{Name}.java` - JPA Entity
2. `db/repository/{Name}Repository.java` - Repository interface
3. `db/repository/specifications/{Name}Specifications.java` - JPA Specifications
4. `dto/{name}/filter/{Name}Filter.java` - Filter DTO
5. `dto/{name}/request/{Name}Request.java` - Request DTO
6. `dto/{name}/response/{Name}Response.java` - Response DTO
7. `dto/{name}/response/Page{Name}Response.java` - Page Response DTO
8. `mapper/{Name}Mapper.java` - MapStruct Mapper
9. `service/{Name}Service.java` - Service interface
10. `service/impl/{Name}ServiceImpl.java` - Service implementation
11. `controller/{Name}Controller.java` - REST Controller
12. `resources/db/migration/V###__{Description}.sql` - Flyway migration

## Supported Technologies

- Spring Boot 3.x
- Java 21+
- PostgreSQL
- Flyway
- MapStruct
- Lombok
- Swagger/OpenAPI
