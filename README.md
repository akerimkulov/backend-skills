# Backend Skills (Spring Boot)

Скиллы для Claude Code — паттерны и шаблоны разработки Spring Boot приложений.

## Установка

```bash
# 1. Клонировать репозиторий
git clone git@github.com:YOUR_ORG/backend-skills.git ~/IdeaProjects/backend-skills

# 2. В backend проекте создать симлинк
cd ~/IdeaProjects/your-backend-project
ln -s ~/IdeaProjects/backend-skills/.claude/skills .claude/skills

# 3. Добавить в .gitignore (чтобы не коммитить симлинк)
echo ".claude/skills" >> .gitignore
```

## Доступные скиллы

| Скилл | Описание |
|-------|----------|
| `controller-patterns` | REST контроллеры — порядок методов, Swagger аннотации, обработка запросов |
| `dto-patterns` | DTO классы — Filter, Request, Response, PageResponse |
| `entity-repository` | JPA Entity, Repository и Specifications |
| `error-handling` | Обработка ошибок — исключения, ExceptionFactory, GlobalExceptionHandler |
| `flyway-patterns` | Flyway миграции — SQL, типы данных, индексы, constraints |
| `logging-patterns` | Логирование — @Slf4j, LogService, двухуровневая система |
| `mapper-patterns` | MapStruct маппинг Entity ↔ DTO |
| `service-patterns` | Service/ServiceImpl — порядок методов, транзакции, логирование |
| `spring-boot-architecture` | Архитектура — структура пакетов, базовые классы, конфигурация |

## Использование

После установки скиллы автоматически доступны в Claude Code. Используйте команды:

```
/controller-patterns  - при создании контроллеров
/dto-patterns         - при создании DTO классов
/entity-repository    - при создании сущностей
/error-handling       - при обработке ошибок
/flyway-patterns      - при создании миграций
/logging-patterns     - при добавлении логирования
/mapper-patterns      - при создании мапперов
/service-patterns     - при создании сервисов
/spring-boot-architecture - при проектировании архитектуры
```

## Обновление

```bash
cd ~/IdeaProjects/backend-skills && git pull
```

Один `git pull` — скиллы обновляются во всех проектах автоматически.

## Структура

```
backend-skills/
├── .claude/
│   └── skills/
│       ├── controller-patterns/
│       │   └── SKILL.md
│       ├── dto-patterns/
│       │   └── SKILL.md
│       ├── entity-repository/
│       │   └── SKILL.md
│       ├── error-handling/
│       │   └── SKILL.md
│       ├── flyway-patterns/
│       │   └── SKILL.md
│       ├── logging-patterns/
│       │   └── SKILL.md
│       ├── mapper-patterns/
│       │   └── SKILL.md
│       ├── service-patterns/
│       │   └── SKILL.md
│       └── spring-boot-architecture/
│           └── SKILL.md
└── README.md
```

## Технологии

- Spring Boot 3.x
- Java 21+
- PostgreSQL
- Flyway
- MapStruct
- Lombok
- Swagger/OpenAPI
