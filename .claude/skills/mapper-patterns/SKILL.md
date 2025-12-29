---
name: mapper-patterns
description: Паттерны MapStruct маппинга. Используй при создании мапперов.
---

# MapStruct Mapper Patterns

## Шаблон Mapper

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

## 4 обязательных метода

| Метод | Назначение | Возвращает |
|-------|-----------|------------|
| toEntity(Request) | Request → Entity | Entity |
| toResponse(Entity) | Entity → Response | Response |
| toPageResponse(Page) | Page → PageResponse | PageResponse |
| updateEntityFromRequest() | Update Entity in-place | void |

## Правила @Mapping

### toEntity() - что игнорировать:

```java
@Mapping(target = "related", ignore = true)      // все связи ManyToOne
@Mapping(target = "children", ignore = true)     // все связи OneToMany
// audit поля НЕ нужно игнорировать - они наследуются и устанавливаются автоматически
```

### updateEntityFromRequest() - ВСЕГДА игнорировать:

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

## Переименование полей

```java
// Если имена в Entity и DTO отличаются
@Mapping(source = "densityKgPerCubicMeter", target = "density")
DomainResponse toResponse(Domain entity);

@Mapping(source = "density", target = "densityKgPerCubicMeter")
Domain toEntity(DomainRequest request);
```

## uses = {} для вложенных объектов

```java
@Mapper(componentModel = "spring", uses = {
    RelatedMapper.class,
    CategoryMapper.class,
    TypeMapper.class
})
public interface DomainMapper { ... }
```

## Маппинг вложенных объектов

```java
// Автоматический маппинг если есть маппер в uses
// Entity.related → Response.related автоматически через RelatedMapper

// Для списков
@Mapping(target = "items", source = "children")
DomainResponse toResponse(Domain entity);
```

## toPageResponse с аналитикой

```java
default PageDomainResponse toPageResponse(Page<Domain> page) {
    PageDomainResponse response = new PageDomainResponse();
    response.setContent(page.getContent().stream()
        .map(this::toResponse)
        .collect(Collectors.toList()));
    response.setTotalElements(page.getTotalElements());
    response.setTotalPages(page.getTotalPages());
    response.setPage(page.getNumber() + 1);  // 0-based → 1-based
    response.setSize(page.getSize());

    // Опциональная аналитика
    PageDomainResponse.Analytics analytics = new PageDomainResponse.Analytics();
    analytics.setTotal((int) page.getTotalElements());
    response.setAnalytics(analytics);

    return response;
}
```

## ЗАПРЕТЫ

```
❌ Repository в маппере
❌ @Component класс вместо @Mapper interface
❌ Бизнес-логика в маппере
❌ Запросы к БД в маппере
❌ Установка связей в маппере

✅ Связи устанавливаются ТОЛЬКО в Service:
   entity.setRelated(relatedService.findById(request.getRelatedId()));
```

## Abstract class Mapper (для сложной логики)

```java
@Mapper(componentModel = "spring", uses = {RelatedMapper.class})
public abstract class DomainMapper {

    @Autowired
    protected CalculationUtil calculationUtil;

    @Mapping(target = "related", ignore = true)
    public abstract Domain toEntity(DomainRequest request);

    @Mapping(target = "calculatedField", expression = "java(calculationUtil.calculate(entity))")
    public abstract DomainResponse toResponse(Domain entity);

    public PageDomainResponse toPageResponse(Page<Domain> page) {
        PageDomainResponse response = new PageDomainResponse();
        response.setContent(page.getContent().stream()
            .map(this::toResponse)
            .collect(Collectors.toList()));
        response.setTotalElements(page.getTotalElements());
        response.setTotalPages(page.getTotalPages());
        response.setPage(page.getNumber() + 1);
        response.setSize(page.getSize());
        return response;
    }

    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "related", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    public abstract void updateEntityFromRequest(@MappingTarget Domain entity, DomainRequest request);
}
```

## Expression для вычисляемых полей

```java
// Простое выражение
@Mapping(target = "fullName", expression = "java(entity.getFirstName() + \" \" + entity.getLastName())")

// С утилитой
@Mapping(target = "formattedDate", expression = "java(DateUtils.format(entity.getCreatedAt()))")

// Условное значение
@Mapping(target = "status", expression = "java(entity.isActive() ? \"ACTIVE\" : \"INACTIVE\")")
```

## Constant и Default values

```java
// Константа
@Mapping(target = "type", constant = "DEFAULT")

// Значение по умолчанию если source = null
@Mapping(target = "count", defaultValue = "0")
```
