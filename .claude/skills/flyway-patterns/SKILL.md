---
name: flyway-patterns
description: Паттерны Flyway миграций - SQL, типы данных, индексы, constraints. Используй при создании миграций.
---

# Flyway Migration Patterns

## Формат имени файла

```
V{version}__{Description}.sql

Примеры:
V001__Create_Tables.sql
V002__Insert_Reference_Data.sql
V015__Add_New_Module.sql
V016__Alter_Table_Add_Column.sql
```

## Структура файла миграции

```sql
-- ============================================================================
-- V###: Описание миграции
-- ============================================================================

-- ============================================================================
-- ЧАСТЬ 1: ТАБЛИЦЫ
-- ============================================================================

CREATE TABLE IF NOT EXISTS table_name (
    -- поля
);

-- ============================================================================
-- ЧАСТЬ 2: ИНДЕКСЫ
-- ============================================================================

CREATE INDEX IF NOT EXISTS ... ;
CREATE UNIQUE INDEX IF NOT EXISTS ... ;

-- ============================================================================
-- ЧАСТЬ 3: КОММЕНТАРИИ
-- ============================================================================

COMMENT ON TABLE ... ;
COMMENT ON COLUMN ... ;
```

## Префиксы таблиц

| Префикс | Назначение | Примеры |
|---------|-----------|---------|
| hb_* | Справочники (handbook) | hb_categories, hb_types |
| sys_* | Системные | sys_users, sys_roles, sys_audit_log |
| (без) | Операционные | orders, transactions, history |

## Шаблон создания таблицы

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

## Типы данных

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

## Precision для DECIMAL

| Использование | Precision | Пример |
|--------------|-----------|--------|
| Объемы, суммы | (18,8) | 9999999999.99999999 |
| Плотность | (9,6) | 999.999999 |
| Температура | (5,2) | 999.99 |
| Координаты | (10,7) | 180.1234567 |
| Проценты | (5,2) | 100.00 |

## Индексы

```sql
-- Обычный индекс
CREATE INDEX IF NOT EXISTS idx_table_column
    ON table_name(column);

-- Составной индекс
CREATE INDEX IF NOT EXISTS idx_table_col1_col2
    ON table_name(col1, col2);

-- Уникальный индекс
CREATE UNIQUE INDEX IF NOT EXISTS uk_table_columns
    ON table_name(col1, col2);

-- Частичный индекс (только активные записи)
CREATE UNIQUE INDEX IF NOT EXISTS uk_table_code_active
    ON table_name(code)
    WHERE deleted = FALSE;

-- Индекс с сортировкой
CREATE INDEX IF NOT EXISTS idx_table_date
    ON table_name(created_at DESC);
```

## Обязательные индексы

```sql
-- На каждую таблицу:
CREATE INDEX IF NOT EXISTS idx_table_deleted ON table_name(deleted);

-- На каждый FK:
CREATE INDEX IF NOT EXISTS idx_table_fk_column ON table_name(fk_column_id);

-- На часто фильтруемые поля:
CREATE INDEX IF NOT EXISTS idx_table_status ON table_name(status);
CREATE INDEX IF NOT EXISTS idx_table_date ON table_name(operation_date DESC);
```

## Комментарии

```sql
COMMENT ON TABLE hb_domains IS 'СПРАВОЧНИК: Описание таблицы';

COMMENT ON COLUMN hb_domains.code IS 'Код записи (уникальный)';
COMMENT ON COLUMN hb_domains.status IS 'Статус: ACTIVE или INACTIVE';
```

## INSERT данных

```sql
-- Простая вставка
INSERT INTO hb_categories (code, name, created_at)
VALUES
    ('CAT1', 'Категория 1', CURRENT_TIMESTAMP),
    ('CAT2', 'Категория 2', CURRENT_TIMESTAMP)
ON CONFLICT (code) DO NOTHING;

-- С подзапросом для FK
INSERT INTO hb_types (code, name, category_id, created_at)
VALUES (
    'TYPE1',
    'Тип 1',
    (SELECT id FROM hb_categories WHERE code = 'CAT1'),
    CURRENT_TIMESTAMP
) ON CONFLICT (code) DO NOTHING;
```

## ALTER TABLE

```sql
-- Добавить колонку
ALTER TABLE table_name
    ADD COLUMN IF NOT EXISTS new_column VARCHAR(100);

-- Добавить колонку с default
ALTER TABLE table_name
    ADD COLUMN IF NOT EXISTS is_active BOOLEAN DEFAULT TRUE;

-- Изменить тип
ALTER TABLE table_name
    ALTER COLUMN column_name TYPE VARCHAR(200);

-- Добавить NOT NULL (с default)
ALTER TABLE table_name
    ALTER COLUMN column_name SET NOT NULL;

-- Добавить FK
ALTER TABLE table_name
    ADD CONSTRAINT fk_table_ref
    FOREIGN KEY (ref_id) REFERENCES ref_table(id);

-- Удалить колонку
ALTER TABLE table_name
    DROP COLUMN IF EXISTS old_column;
```

## Управление правами (DO блок)

```sql
DO $$
DECLARE
    v_role_id BIGINT;
BEGIN
    -- Получить ID роли
    SELECT id INTO v_role_id FROM sys_roles WHERE code = 'ADMIN';

    -- Назначить права
    INSERT INTO sys_role_routes (role_id, route_id, can_read, can_write)
    SELECT v_role_id, id, true, true
    FROM sys_routes
    WHERE code LIKE '/api/domain%'
    ON CONFLICT DO NOTHING;
END $$;
```

## Добавление API Routes

```sql
-- Добавить маршруты
INSERT INTO sys_available_routes (code, method, description, created_at)
VALUES
    ('/api/domain-name/{id}', 'GET', 'Получить по ID', CURRENT_TIMESTAMP),
    ('/api/domain-name/filter', 'POST', 'Фильтрация с пагинацией', CURRENT_TIMESTAMP),
    ('/api/domain-name', 'POST', 'Создать', CURRENT_TIMESTAMP),
    ('/api/domain-name', 'PUT', 'Обновить', CURRENT_TIMESTAMP),
    ('/api/domain-name/{id}', 'DELETE', 'Удалить', CURRENT_TIMESTAMP)
ON CONFLICT (code, method) DO NOTHING;

-- Назначить права ролям
DO $$
DECLARE
    v_admin_id BIGINT;
    v_operator_id BIGINT;
BEGIN
    SELECT id INTO v_admin_id FROM sys_roles WHERE code = 'ADMIN';
    SELECT id INTO v_operator_id FROM sys_roles WHERE code = 'OPERATOR';

    -- ADMIN: все права
    INSERT INTO sys_role_route_access (role_id, route_id, can_get, can_post, can_put, can_delete)
    SELECT v_admin_id, id, true, true, true, true
    FROM sys_available_routes
    WHERE code LIKE '/api/domain-name%'
    ON CONFLICT DO NOTHING;

    -- OPERATOR: только чтение
    INSERT INTO sys_role_route_access (role_id, route_id, can_get, can_post, can_put, can_delete)
    SELECT v_operator_id, id, true, true, false, false
    FROM sys_available_routes
    WHERE code LIKE '/api/domain-name%'
    ON CONFLICT DO NOTHING;
END $$;
```

## Правила

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
