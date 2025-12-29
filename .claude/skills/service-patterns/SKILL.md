---
name: service-patterns
description: Паттерны Service/ServiceImpl - порядок методов, логирование, транзакции. Используй при создании сервисов.
---

# Service Implementation Patterns

## Обязательный порядок методов

```
1. create()           - Создание
2. update()           - Обновление
3. delete()           - Удаление (soft delete)
4. get()              - Получить по ID → возвращает DTO
5. findById()         - Внутренний метод → возвращает Entity
6. findAllWithFilter() - ЕДИНСТВЕННЫЙ метод поиска
```

## ЗАПРЕЩЕННЫЕ методы

```
❌ findAll()           - использовать findAllWithFilter()
❌ findByName()        - использовать findAllWithFilter()
❌ searchBy*()         - использовать findAllWithFilter()
❌ getAllActive()      - использовать findAllWithFilter() с фильтром
❌ getList()           - использовать findAllWithFilter()
```

## Шаблон Service Interface

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

## Шаблон ServiceImpl

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

## Двойное логирование

```java
// Обязательные компоненты класса:
@Slf4j                                           // файловое логирование
private final SysLogsRequestService logService;  // табличное логирование

// После успешной операции:

// 1. Файловое логирование
log.info("Created entity: id={}, code={}", id, code);

// 2. Табличное логирование
// Create/Update - 4 параметра:
logService.saveSuccessToDb(className, message, request, httpServletRequest);

// Delete - 3 параметра (БЕЗ request):
logService.saveSuccessToDb(className, message, httpServletRequest);
```

## Multi-tenant фильтрация

```java
@Override
public PageDomainResponse findAllWithFilter(DomainFilter filter) {
    // Применить фильтр доступных depot IDs
    List<Long> accessibleDepotIds = AccessUtils.applyFilter(filter.getDepotIds());
    filter.setDepotIds(accessibleDepotIds);

    // ... остальная логика
}
```

## Валидация доступа

```java
@Override
public DomainResponse get(Long id) {
    Domain entity = findById(id);

    // Проверить доступ к depot
    AccessUtils.validateAccess(entity.getDepot().getId());

    return mapper.toResponse(entity);
}
```

## Правила

1. **@Transactional** на методах изменения данных
2. **HttpServletRequest** в сигнатуре CUD методов
3. **findById()** возвращает Entity (для внутреннего использования)
4. **get()** возвращает DTO (для внешнего API)
5. **Маппер** для конвертации Entity ↔ DTO
6. **Связи устанавливаются в Service**, не в Mapper
7. **Двойное логирование** - @Slf4j + SysLogsRequestService
8. **Soft delete** - markAsDeleted() + setUpdatedAt()

## Исключения

```java
// Не найдено
throw new ResourceNotFoundException("Запись не найдена: " + id);

// Дубликат
throw new DuplicateResourceException("Запись с кодом уже существует: " + code);

// Бизнес-ошибка
throw new BusinessException("Невозможно выполнить операцию: причина");

// Нет доступа
throw new AuthorizationException("Нет доступа к данному ресурсу");
```
