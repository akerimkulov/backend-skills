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

## Flyway Rules

```
✅ IF NOT EXISTS для таблиц и индексов
✅ ON CONFLICT DO NOTHING для INSERT
✅ BIGSERIAL для PRIMARY KEY
✅ DEFAULT FALSE для deleted
✅ DEFAULT CURRENT_TIMESTAMP для created_at
✅ Комментарии на русском
✅ Индекс на каждый FK
✅ Индекс на deleted поле

❌ НЕ использовать SERIAL (только BIGSERIAL)
❌ НЕ забывать deleted поле
❌ НЕ забывать audit поля (created_at, created_by, updated_at, updated_by)
❌ НЕ использовать CASCADE DELETE (soft delete)
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
List<Long> depotIds = CurrentUserUtils.getAccessibleDepotIds();

// Валидация доступа
AccessUtils.validateAccess(entityId);
List<Long> filtered = AccessUtils.applyFilter(requestIds);
```

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
