---
name: dto-patterns
description: Паттерны DTO - Filter, Request, Response, PageResponse. Используй при создании DTO классов.
---

# DTO Patterns

## Типы DTO

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

    @ArraySchema(schema = @Schema(description = "Список ID связанных", example = "[1, 2]"))
    private List<Long> relatedIds;

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

## Правила Response DTO

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

## Правила Request DTO

```
✅ Для CREATE: id = null
✅ Для UPDATE: id заполнен
✅ Связи через ID: relatedId, не полный объект
✅ Валидация: @NotNull, @NotBlank, @Size, @Positive, @Digits
✅ requiredMode = REQUIRED для обязательных полей в Swagger
```

## Валидационные аннотации

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
@PastOrPresent(message = "Не может быть в будущем")

// Коллекции
@NotEmpty(message = "Список не может быть пустым")
@Size(min = 1, max = 10, message = "От 1 до 10 элементов")

// Email
@Email(message = "Неверный формат email")
```

## Swagger аннотации

```java
// Для полей
@Schema(description = "Описание", example = "пример", requiredMode = Schema.RequiredMode.REQUIRED)

// Для списков
@ArraySchema(schema = @Schema(description = "Описание элемента", example = "1"))

// Для вложенных объектов
@Schema(description = "Описание", implementation = NestedResponse.class)

// Скрыть поле
@Schema(hidden = true)
```

## Структура пакетов DTO

```
dto/
├── common/
│   ├── BasePageRequest.java
│   └── BasePageResponse.java
└── {domain}/
    ├── filter/
    │   └── DomainFilter.java
    ├── request/
    │   └── DomainRequest.java
    └── response/
        ├── DomainResponse.java
        └── PageDomainResponse.java
```
