---
name: entity-repository
description: Паттерны JPA Entity, Repository и Specifications. Используй при создании сущностей и репозиториев.
---

# Entity & Repository Patterns

## Entity шаблон

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

## Правила Entity

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

## JPA аннотации - только отличия от default

```java
// @Column defaults: length=255, nullable=true, unique=false
@Column(name = "code")                    // только имя
@Column(nullable = false)                  // только nullable
@Column(length = 50)                       // только length
@Column(nullable = false, unique = true)   // несколько отличий

// @ManyToOne defaults: fetch=EAGER, optional=true
@ManyToOne(fetch = FetchType.LAZY)         // только fetch (LAZY отличается!)
@JoinColumn(name = "related_id", nullable = false)

// @OneToMany defaults: fetch=LAZY
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
```

## Precision для BigDecimal

| Использование | Precision | Пример |
|--------------|-----------|--------|
| Объемы, массы, суммы | (18,8) | 9999999999.99999999 |
| Плотность | (9,6) | 999.999999 |
| Температура | (5,2) | 999.99 |
| Координаты | (10,7) | 180.1234567 |
| Проценты | (5,2) | 100.00 |

## Repository шаблон

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

## Specifications шаблон

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

    // Boolean фильтр
    public static Specification<Domain> byIsActive(Boolean isActive) {
        return (root, query, cb) -> {
            if (isActive == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("isActive"), isActive);
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
            .and(byDateRange(filter.getCreatedAtFrom(), filter.getCreatedAtTo()))
            .and(byIsActive(filter.getIsActive()));
    }
}
```

## Правила Specifications

```
✅ Всегда начинать с notDeleted()
✅ Использовать цепочку .and()
✅ Возвращать cb.conjunction() для null/empty значений
✅ isBlank() для строк
✅ Название главного метода: filterBy()
✅ Отдельный метод для каждого поля фильтра

❌ НЕ использовать List<Predicate> + stream().reduce()
❌ НЕ использовать Specification.where()
❌ НЕ называть withFilter(), byFilter()
❌ НЕ использовать StringUtils.hasText() - использовать isBlank()
```

## Fetch Strategy

```java
// Для избежания N+1 queries при загрузке связей

// Вариант 1: JOIN FETCH в Repository
@Query("SELECT d FROM Domain d " +
       "LEFT JOIN FETCH d.category " +
       "LEFT JOIN FETCH d.type " +
       "WHERE d.id = :id AND d.deleted = false")
Optional<Domain> findByIdWithDetailsAndDeletedFalse(@Param("id") Long id);

// Вариант 2: EntityGraph
@EntityGraph(attributePaths = {"category", "type"})
Optional<Domain> findByIdAndDeletedFalse(Long id);
```
