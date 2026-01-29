# FSD Backend Skill

A Claude Code skill that generates production-ready backend code for Spring Boot projects. Just describe what you need in plain language - no architecture knowledge required.

---

## Installation / Установка

### English

Clone directly to your skills folder:

```bash
git clone https://github.com/akerimkulov/backend-skills.git ~/.claude/skills/fsd-backend
```

Restart Claude Code to load the skill.

### Русский

Клонируйте напрямую в папку скиллов:

```bash
git clone https://github.com/akerimkulov/backend-skills.git ~/.claude/skills/fsd-backend
```

Перезапустите Claude Code для загрузки скилла.

---

## Usage / Использование

### English

Just describe what you need in plain language. The skill will generate all necessary files following best practices.

#### Create Entities

```
Create a User entity with name, email, and role
```

```
Create a Product entity with title, price, description, and categoryId
```

```
Create an Order entity with items, totalAmount, status, and customerId
```

#### Create Full CRUD

```
Create full CRUD for Product with controller, service, repository, DTOs, and mapper
```

```
Create a complete API for Orders with filtering and pagination
```

#### Create Migrations

```
Create a Flyway migration for products table with name, price, and category_id
```

```
Create a migration to add description column to orders table
```

#### Create Specific Layers

```
Create only the service layer for Payments
```

```
Create a controller for Users with Swagger documentation
```

```
Create DTOs (Filter, Request, Response) for Products
```

### Русский

Просто опишите что вам нужно на обычном языке. Скилл сгенерирует все необходимые файлы следуя лучшим практикам.

#### Создание сущностей

```
Создай сущность User с полями name, email и role
```

```
Создай сущность Product с полями title, price, description и categoryId
```

```
Создай сущность Order с полями items, totalAmount, status и customerId
```

#### Создание полного CRUD

```
Создай полный CRUD для Product с контроллером, сервисом, репозиторием, DTO и маппером
```

```
Создай полное API для Orders с фильтрацией и пагинацией
```

#### Создание миграций

```
Создай Flyway миграцию для таблицы products с полями name, price и category_id
```

```
Создай миграцию для добавления колонки description в таблицу orders
```

#### Создание отдельных слоев

```
Создай только сервисный слой для Payments
```

```
Создай контроллер для Users со Swagger документацией
```

```
Создай DTO (Filter, Request, Response) для Products
```

---

## What Gets Generated / Что генерируется

### English

When you create an entity (e.g., "Create a Product with name and price"), you get:

```
src/main/java/com/example/
├── controller/
│   └── ProductController.java      # REST API with Swagger docs
├── service/
│   ├── ProductService.java         # Interface
│   └── impl/
│       └── ProductServiceImpl.java # Implementation with logging
├── db/
│   ├── entity/
│   │   └── Product.java            # JPA Entity with audit fields
│   ├── repository/
│   │   ├── ProductRepository.java  # Spring Data JPA
│   │   └── specifications/
│   │       └── ProductSpecifications.java  # Dynamic filters
│   └── enums/
│       └── ProductStatus.java      # If needed
├── dto/
│   └── product/
│       ├── filter/
│       │   └── ProductFilter.java  # Search filters + pagination
│       ├── request/
│       │   └── ProductRequest.java # Create/Update DTO with validation
│       └── response/
│           ├── ProductResponse.java
│           └── PageProductResponse.java
├── mapper/
│   └── ProductMapper.java          # MapStruct mapper
└── resources/
    └── db/migration/
        └── V###__Create_Products.sql  # Flyway migration
```

### Русский

Когда вы создаете сущность (например, "Создай Product с name и price"), вы получаете:

```
src/main/java/com/example/
├── controller/
│   └── ProductController.java      # REST API со Swagger документацией
├── service/
│   ├── ProductService.java         # Интерфейс
│   └── impl/
│       └── ProductServiceImpl.java # Реализация с логированием
├── db/
│   ├── entity/
│   │   └── Product.java            # JPA Entity с audit полями
│   ├── repository/
│   │   ├── ProductRepository.java  # Spring Data JPA
│   │   └── specifications/
│   │       └── ProductSpecifications.java  # Динамические фильтры
│   └── enums/
│       └── ProductStatus.java      # При необходимости
├── dto/
│   └── product/
│       ├── filter/
│       │   └── ProductFilter.java  # Фильтры поиска + пагинация
│       ├── request/
│       │   └── ProductRequest.java # DTO создания/обновления с валидацией
│       └── response/
│           ├── ProductResponse.java
│           └── PageProductResponse.java
├── mapper/
│   └── ProductMapper.java          # MapStruct маппер
└── resources/
    └── db/migration/
        └── V###__Create_Products.sql  # Flyway миграция
```

---

## Supported Technologies / Поддерживаемые технологии

| Category / Категория | Technologies / Технологии |
|---------------------|---------------------------|
| Framework | Spring Boot 3.x |
| Language | Java 21+ |
| Database | PostgreSQL |
| Migrations | Flyway |
| ORM | Spring Data JPA / Hibernate |
| Mapping | MapStruct |
| Validation | Jakarta Validation |
| Documentation | Swagger / OpenAPI (springdoc) |
| Utilities | Lombok |
| Security | Spring Security + JWT |

---

## Examples / Примеры

### Example 1: E-commerce Product

**English:**
```
Create a Product entity with:
- code (string, unique)
- name (string, required)
- price (BigDecimal)
- description (string, optional)
- categoryId (reference to Category)
- status (enum: ACTIVE, INACTIVE)
```

**Русский:**
```
Создай сущность Product с полями:
- code (строка, уникальный)
- name (строка, обязательный)
- price (BigDecimal)
- description (строка, опциональный)
- categoryId (ссылка на Category)
- status (enum: ACTIVE, INACTIVE)
```

### Example 2: User Management

**English:**
```
Create full CRUD for User with username, email, passwordHash, role, and isActive
```

**Русский:**
```
Создай полный CRUD для User с полями username, email, passwordHash, role и isActive
```

### Example 3: Migration Only

**English:**
```
Create a Flyway migration V015 for orders table with:
- id (bigserial, primary key)
- order_number (varchar 50, unique)
- customer_id (bigint, foreign key to users)
- total_amount (decimal 18,2)
- status (varchar 20, check constraint)
- created_at, updated_at, deleted (audit fields)
```

**Русский:**
```
Создай Flyway миграцию V015 для таблицы orders с:
- id (bigserial, primary key)
- order_number (varchar 50, unique)
- customer_id (bigint, foreign key на users)
- total_amount (decimal 18,2)
- status (varchar 20, check constraint)
- created_at, updated_at, deleted (audit поля)
```

---

## API Patterns / Паттерны API

### Standard Endpoints

Every entity gets these endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/{id}` | Get by ID |
| POST | `/filter` | Search with filters + pagination |
| POST | `/` | Create |
| PUT | `/` | Update (id in body) |
| DELETE | `/{id}` | Soft delete |

### Response Format

```json
{
  "success": true,
  "message": null,
  "result": { ... },
  "time": "2025-01-29 12:00:00"
}
```

---

## Tips / Советы

### English

1. **Be specific about field types** - "Create User with name (String), age (Integer), salary (BigDecimal)"
2. **Specify relationships** - "Product has categoryId (many-to-one to Category)"
3. **Request only what you need** - "Create only the service layer for Orders"
4. **Mention constraints** - "code field should be unique"
5. **Specify enum values** - "status enum with ACTIVE, INACTIVE, PENDING"

### Русский

1. **Указывайте типы полей** - "Создай User с name (String), age (Integer), salary (BigDecimal)"
2. **Указывайте связи** - "Product имеет categoryId (many-to-one на Category)"
3. **Запрашивайте только нужное** - "Создай только сервисный слой для Orders"
4. **Упоминайте ограничения** - "поле code должно быть уникальным"
5. **Указывайте значения enum** - "status enum со значениями ACTIVE, INACTIVE, PENDING"

---

## Architecture / Архитектура

This skill follows **layered architecture**:

```
Controller (REST API)
    ↓
Service (Business Logic)
    ↓
Repository (Data Access)
    ↓
Entity (JPA/Database)
```

**Key principles / Ключевые принципы:**

- Constructor injection (`@RequiredArgsConstructor`)
- Soft delete (no physical deletion)
- Audit fields on all entities (createdAt, createdBy, updatedAt, updatedBy, deleted)
- Validation in DTOs, not in Entities
- Single search method: `POST /filter` with JPA Specifications
- Relationships set in Service, not in Mapper

---

## Update / Обновление

```bash
cd ~/.claude/skills/fsd-backend && git pull
```

---

## License / Лицензия

MIT
