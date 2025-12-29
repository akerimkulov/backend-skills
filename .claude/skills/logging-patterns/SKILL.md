---
name: logging-patterns
description: Паттерны логирования - @Slf4j, LogService, AuditLog, уровни логирования. Используй при добавлении логирования.
---

# Logging Patterns

## Двухуровневая система логирования

| Уровень | Назначение | Хранение | Поиск |
|---------|------------|----------|-------|
| **Файловое** (Logback + @Slf4j) | Все события приложения | 30 дней | grep, ELK |
| **Табличное** (LogService) | Бизнес-операции | 90 дней | SQL запросы |

## Обязательные компоненты класса

```java
@Service
@RequiredArgsConstructor
@Slf4j                                    // Файловое логирование
public class DomainServiceImpl implements DomainService {

    private final LogService logService;  // Табличное логирование (опционально)

    // ...
}
```

## Уровни логирования

```java
// DEBUG - детальная отладочная информация
log.debug("Finding entity by id={}", id);
log.debug("Calculating value: param1={}, param2={}", param1, param2);

// INFO - важные события (успешные операции)
log.info("Entity created: id={}, code={}", id, code);
log.info("User logged in: username={}", username);

// WARN - потенциальные проблемы
log.warn("Task {} rejected, queue is full", taskName);
log.warn("User not found: {}", username);

// ERROR - серьёзные ошибки
log.error("Failed to save: {}", exception.getMessage());
log.error("Database connection failed", exception);
```

## Паттерн: ФАЙЛ → БД

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

## LogService — интерфейс

```java
public interface LogService {

    // Успешная операция (без DTO)
    void saveSuccess(String className, String message, HttpServletRequest request);

    // Успешная операция (с DTO)
    void saveSuccess(String className, String message, Object requestDto, HttpServletRequest request);

    // Ошибка (ФАЙЛ + БД)
    void saveException(String className, Exception e, String methodName, HttpServletRequest request);
}
```

## LogServiceImpl — реализация

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class LogServiceImpl implements LogService {

    private final LogRepository logRepository;

    @Override
    @Async("logExecutor")
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveSuccess(String className, String message, HttpServletRequest request) {
        try {
            LogEntry entry = LogEntry.builder()
                .className(className)
                .message(truncate(message, 4500))
                .status("SUCCESS")
                .remoteIp(extractIp(request))
                .userAgent(request.getHeader("User-Agent"))
                .createdAt(LocalDateTime.now())
                .build();
            logRepository.save(entry);
        } catch (Exception e) {
            log.error("Failed to save log: {}", e.getMessage());
        }
    }

    @Override
    public void saveException(String className, Exception e, String methodName, HttpServletRequest request) {
        // 1. СНАЧАЛА файловое логирование
        log.error("[{}] {} - {}", className, methodName, e.getMessage(), e);

        // 2. ПОТОМ в БД (асинхронно)
        saveExceptionAsync(className, e, methodName, request);
    }

    @Async("logExecutor")
    protected void saveExceptionAsync(String className, Exception e, String methodName, HttpServletRequest request) {
        try {
            LogEntry entry = LogEntry.builder()
                .className(className)
                .methodName(methodName)
                .message(truncate(e.getMessage(), 4500))
                .status("ERROR")
                .remoteIp(extractIp(request))
                .createdAt(LocalDateTime.now())
                .build();
            logRepository.save(entry);
        } catch (Exception ex) {
            log.error("Failed to save exception log: {}", ex.getMessage());
        }
    }

    private String extractIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getHeader("X-Real-IP");
        }
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }

    private String truncate(String text, int maxLength) {
        if (text == null) return null;
        return text.length() > maxLength ? text.substring(0, maxLength) : text;
    }
}
```

## Примеры в сервисах

### Create операция

```java
@Override
@Transactional
public EntityResponse create(EntityRequest request, HttpServletRequest httpRequest) {
    // Валидация и создание...
    Entity saved = repository.save(entity);

    // Файловое логирование
    log.info("Entity created: id={}, name={}", saved.getId(), saved.getName());

    // Табличное логирование (опционально)
    logService.saveSuccess(
        this.getClass().getSimpleName(),
        "Created entity: id=" + saved.getId(),
        request,
        httpRequest
    );

    return mapper.toResponse(saved);
}
```

### Update операция

```java
@Override
@Transactional
public EntityResponse update(Long id, EntityRequest request, HttpServletRequest httpRequest) {
    Entity entity = findById(id);
    // Обновление...
    Entity updated = repository.save(entity);

    log.info("Entity updated: id={}", updated.getId());

    logService.saveSuccess(
        this.getClass().getSimpleName(),
        "Updated entity: id=" + id,
        request,
        httpRequest
    );

    return mapper.toResponse(updated);
}
```

### Delete операция

```java
@Override
@Transactional
public void delete(Long id, HttpServletRequest httpRequest) {
    Entity entity = findById(id);

    entity.setDeleted(true);
    entity.setUpdatedAt(LocalDateTime.now());
    repository.save(entity);

    log.info("Entity deleted: id={}", id);

    // БЕЗ request DTO — 3 параметра
    logService.saveSuccess(
        this.getClass().getSimpleName(),
        "Deleted entity: id=" + id,
        httpRequest
    );
}
```

## Конфигурация Logback

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <property name="LOG_PATTERN"
        value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%X{username:-SYSTEM}] [%thread] %logger{40} : %msg%n"/>

    <!-- Консоль -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- Файл с ротацией -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- Асинхронный аппендер -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
    </appender>

    <!-- Уровни логирования -->
    <logger name="com.yourcompany" level="INFO"/>
    <logger name="org.springframework.web" level="ERROR"/>
    <logger name="org.hibernate" level="ERROR"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </root>

</configuration>
```

## Асинхронный Executor

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "logExecutor")
    public Executor logExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(6);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("Log-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
}
```

## Entity для логов

```java
@Entity
@Table(name = "sys_logs")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor @Builder
public class LogEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String className;

    private String methodName;

    @Column(nullable = false)
    private String status;  // SUCCESS, ERROR

    @Column(columnDefinition = "TEXT")
    private String message;

    @Column(columnDefinition = "TEXT")
    private String requestBody;

    @Column(length = 45)
    private String remoteIp;

    @Column(length = 500)
    private String userAgent;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
}
```

## SQL миграция для таблицы логов

```sql
CREATE TABLE IF NOT EXISTS sys_logs (
    id BIGSERIAL PRIMARY KEY,
    class_name VARCHAR(255) NOT NULL,
    method_name VARCHAR(255),
    status VARCHAR(50) NOT NULL,
    message TEXT,
    request_body TEXT,
    remote_ip VARCHAR(45),
    user_agent VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sys_logs_created_at ON sys_logs(created_at);
CREATE INDEX idx_sys_logs_class_name ON sys_logs(class_name);
CREATE INDEX idx_sys_logs_status ON sys_logs(status);
```

## Правила

1. **@Slf4j** обязателен на всех Service классах
2. **LogService** для CUD операций (опционально)
3. **ФАЙЛ → БД** — порядок логирования
4. **Асинхронность** — табличные логи не блокируют основной поток
5. **Плейсхолдеры** — использовать `{}`, не конкатенацию строк
6. **INFO для успеха** — `log.info()` после успешной операции
7. **ERROR для ошибок** — `log.error()` с exception объектом

## Расположение файлов

```
{root.package}/
├── config/
│   └── AsyncConfig.java
├── service/
│   ├── LogService.java
│   └── impl/
│       └── LogServiceImpl.java
├── db/
│   ├── entity/
│   │   └── LogEntry.java
│   └── repository/
│       └── LogRepository.java
└── resources/
    └── logback-spring.xml
```
