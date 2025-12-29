---
name: controller-patterns
description: Паттерны REST контроллеров - порядок методов, Swagger аннотации, обработка запросов. Используй при создании контроллеров.
---

# REST Controller Patterns

## Обязательный порядок методов

```
1. GET /{id}        - Получить по ID
2. POST /filter     - ЕДИНСТВЕННЫЙ метод поиска с пагинацией
3. POST             - Создать
4. PUT              - Обновить (id в теле запроса)
5. DELETE /{id}     - Удалить (soft delete)
```

## ЗАПРЕЩЕННЫЕ endpoints

```
❌ GET /all           - использовать POST /filter
❌ GET /search        - использовать POST /filter
❌ GET /find-by-*     - использовать POST /filter
❌ GET /active        - использовать POST /filter с параметром
❌ POST /search       - использовать POST /filter
❌ GET /list          - использовать POST /filter
```

## Шаблон контроллера

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

## Swagger @Tag категории

```java
// Справочники
@Tag(name = "СПРАВОЧНИК: Название", description = "...")

// Операции
@Tag(name = "ОПЕРАЦИИ: Название", description = "...")

// Системные
@Tag(name = "СИСТЕМА: Название", description = "...")

// Мониторинг
@Tag(name = "МОНИТОРИНГ: Название", description = "...")

// Отчеты
@Tag(name = "ОТЧЕТЫ: Название", description = "...")
```

## Конфликт @RequestBody

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

## Дополнительные endpoints

```java
// История изменений (опционально)
@GetMapping("/{id}/history")
@Operation(summary = "История изменений")
public ResponseEntity<? extends BaseResponse<?>> getHistory(@PathVariable Long id) {
    return createSuccessResponse(service.getHistory(id));
}

// Активные записи для dropdown (опционально)
@GetMapping("/active")
@Operation(summary = "Получить активные записи для выпадающего списка")
public ResponseEntity<? extends BaseResponse<?>> getActive() {
    return createSuccessResponse(service.getActiveList());
}
```

## Правила

1. **Наследовать BaseController** - для стандартных методов ответа
2. **@RequiredArgsConstructor** - constructor injection
3. **HttpServletRequest** - в параметрах CUD методов для audit
4. **@Valid** - на request body для валидации
5. **Возвращать ResponseEntity<? extends BaseResponse<?>>** - единый формат
6. **filter == null check** - создавать пустой фильтр если null
7. **Порядок методов** - GET/{id} → POST/filter → POST → PUT → DELETE/{id}

## Типы возвращаемых значений

```java
// Единичная запись
ResponseEntity<? extends BaseResponse<?>>  // data = DomainResponse

// Список с пагинацией
ResponseEntity<? extends BaseResponse<?>>  // data = PageDomainResponse

// Без данных (delete)
ResponseEntity<? extends BaseResponse<?>>  // data = null
```

## API Path Structure

```
${api.base-path}/domain-name

Примеры:
/api/oil-depot/tanks
/api/oil-depot/fuel-types
/api/users
/api/roles
```
