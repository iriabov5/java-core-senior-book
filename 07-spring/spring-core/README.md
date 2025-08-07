# Spring Core

## Обзор темы

**Spring Core** - это фундаментальный модуль Spring Framework, который предоставляет IoC (Inversion of Control) контейнер и систему Dependency Injection (DI) для управления объектами (beans) в Java приложениях.

### Что это такое?

Spring Core является основой всего Spring Framework и реализует паттерн Inversion of Control, где контроль над созданием и связыванием объектов передается контейнеру. Основные компоненты включают BeanFactory, ApplicationContext, различные способы конфигурации (XML, аннотации, Java код) и систему управления жизненным циклом beans.

Контейнер Spring Core читает метаданные конфигурации и создает полностью настроенные и готовые к использованию объекты. Он управляет зависимостями между компонентами, обеспечивает слабую связанность и упрощает тестирование приложений.

### Проблема, которую решает тема

В традиционных Java приложениях часто возникает ситуация, когда компоненты жестко связаны друг с другом, что затрудняет разработку и поддержку. Например:

1. **Сценарий**: Разработчик создает сервис, который напрямую создает экземпляры своих зависимостей
2. **Требование**: Нужно создать гибкую архитектуру с слабой связанностью компонентов
3. **Проблема**: Прямое создание зависимостей приводит к жесткой связанности, сложности тестирования и нарушению принципов SOLID
4. **Результат**: Spring Core решает эти проблемы через IoC контейнер, который управляет созданием и связыванием объектов

## Детальный анализ проблемы

### Сценарий 1: Прямое создание зависимостей

```java
public class UserService {
    
    // ПРОБЛЕМА ЗДЕСЬ!
    private UserRepository userRepository = new UserRepositoryImpl();
    private EmailService emailService = new EmailServiceImpl();
    
    public void createUser(User user) {
        // 1. Валидация пользователя
        
        // 2. Сохранение в базу данных (ПРОБЛЕМА ЗДЕСЬ!)
        userRepository.save(user);
        
        // 3. Отправка уведомления (ПРОБЛЕМА ЗДЕСЬ!)
        emailService.sendWelcomeEmail(user.getEmail());
    }
}
```

**Проблемы:**
- Жесткая связанность между классами
- Сложность тестирования (невозможно подменить зависимости)
- Нарушение принципа инверсии зависимостей
- Сложность конфигурации в разных окружениях

### Сценарий 2: Singleton паттерн

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;
    
    private DatabaseConnection() {}
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
}

public class UserService {
    private DatabaseConnection db = DatabaseConnection.getInstance();
    
    public void saveUser(User user) {
        // Использование глобального состояния
        db.execute("INSERT INTO users...");
    }
}
```

**Проблемы:**
- Глобальное состояние затрудняет тестирование
- Сложность управления жизненным циклом
- Потенциальные проблемы с многопоточностью
- Отсутствие гибкости в конфигурации

### Корень проблемы

Основная проблема заключается в том, что разработчики напрямую управляют созданием и связыванием объектов, что приводит к жесткой связанности компонентов. Это приводит к:

1. **Проблема тестируемости** - невозможно изолировать компоненты для unit-тестирования
2. **Проблема гибкости** - сложно изменить реализацию без изменения кода
3. **Проблема конфигурации** - настройки разбросаны по всему коду
4. **Проблема масштабируемости** - сложно добавлять новые компоненты

## Принцип работы

### Как работает Spring Core

1. **Шаг 1**: Конфигурация beans через XML, аннотации или Java код
2. **Шаг 2**: Создание IoC контейнера (BeanFactory или ApplicationContext)
3. **Шаг 3**: Сканирование и создание beans
4. **Шаг 4**: Внедрение зависимостей и управление жизненным циклом

## Детальный принцип работы

### Шаг 1: Конфигурация beans

```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }
    
    @Bean
    public EmailService emailService() {
        return new EmailServiceImpl();
    }
    
    @Bean
    public UserService userService(UserRepository userRepository, 
                                 EmailService emailService) {
        return new UserService(userRepository, emailService);
    }
}
```

**Ключевые моменты:**
- Использование аннотации `@Configuration` для обозначения класса конфигурации
- Методы с аннотацией `@Bean` создают beans
- Spring автоматически внедряет зависимости в конструктор или сеттеры

### Шаг 2: Создание ApplicationContext

```java
public class Application {
    public static void main(String[] args) {
        // Создание IoC контейнера
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        
        // Получение bean из контейнера
        UserService userService = context.getBean(UserService.class);
        
        // Использование сервиса
        userService.createUser(new User("john@example.com"));
    }
}
```

### Шаг 3: Жизненный цикл бина

```java
@Component
public class UserService implements InitializingBean, DisposableBean {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    // 1. Конструктор
    // 2. Внедрение зависимостей
    // 3. PostConstruct
    @PostConstruct
    public void init() {
        System.out.println("UserService initialized");
    }
    
    // 4. InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("UserService afterPropertiesSet");
    }
    
    // 5. Bean готов к использованию
    
    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user.getEmail());
    }
    
    // 6. PreDestroy
    @PreDestroy
    public void cleanup() {
        System.out.println("UserService cleanup");
    }
    
    // 7. DisposableBean.destroy()
    @Override
    public void destroy() throws Exception {
        System.out.println("UserService destroyed");
    }
}
```

### Архитектурная схема жизненного цикла бина

```
┌─────────────────────────────────────────────────────────────┐
│                    Жизненный цикл бина                     │
├─────────────────────────────────────────────────────────────┤
│  1. Создание экземпляра (Constructor)                     │
│  2. Внедрение зависимостей (Dependency Injection)         │
│  3. @PostConstruct                                        │
│  4. InitializingBean.afterPropertiesSet()                 │
│  5. Bean готов к использованию                            │
│  6. @PreDestroy                                          │
│  7. DisposableBean.destroy()                             │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые преимущества

1. **Слабая связанность**: Компоненты не зависят от конкретных реализаций
2. **Легкое тестирование**: Возможность подмены зависимостей через mock объекты
3. **Гибкость конфигурации**: Разные конфигурации для разных окружений
4. **Управление жизненным циклом**: Автоматическое создание и уничтожение объектов
5. **Стандартизация**: Единый подход к архитектуре приложения

## Практические примеры

### Пример 1: Базовый IoC контейнер

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    
    @Bean
    @Scope("singleton")
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}

@Component
public class UserRepository {
    
    private final JdbcTemplate jdbcTemplate;
    
    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    public void save(User user) {
        jdbcTemplate.update(
            "INSERT INTO users (email, name) VALUES (?, ?)",
            user.getEmail(), user.getName()
        );
    }
}
```

**Объяснение:**
- `@Configuration` обозначает класс как источник конфигурации beans
- `@ComponentScan` автоматически сканирует пакет на наличие компонентов
- `@Bean` методы создают beans в контейнере
- `@Scope("singleton")` определяет область видимости bean
- Spring автоматически внедряет зависимости через конструктор

### Пример 2: Жизненный цикл с событиями

```java
@Component
public class LifecycleBean implements BeanNameAware, BeanFactoryAware, 
                                   ApplicationContextAware, InitializingBean, DisposableBean {
    
    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;
    
    // BeanNameAware
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("Bean name set: " + name);
    }
    
    // BeanFactoryAware
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
        System.out.println("BeanFactory set");
    }
    
    // ApplicationContextAware
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        System.out.println("ApplicationContext set");
    }
    
    // InitializingBean
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean.afterPropertiesSet() called");
    }
    
    // DisposableBean
    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean.destroy() called");
    }
    
    @PostConstruct
    public void postConstruct() {
        System.out.println("@PostConstruct called");
    }
    
    @PreDestroy
    public void preDestroy() {
        System.out.println("@PreDestroy called");
    }
}
```

**Объяснение:**
- `BeanNameAware` - получает имя бина
- `BeanFactoryAware` - получает ссылку на BeanFactory
- `ApplicationContextAware` - получает ссылку на ApplicationContext
- `InitializingBean` - интерфейс для инициализации
- `DisposableBean` - интерфейс для очистки ресурсов

### Пример 3: Условная конфигурация

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @ConditionalOnProperty(name = "spring.profiles.active", havingValue = "dev")
    public DataSource devDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:testdb");
        config.setDriverClassName("org.h2.Driver");
        return new HikariDataSource(config);
    }
    
    @Bean
    @ConditionalOnProperty(name = "spring.profiles.active", havingValue = "prod")
    public DataSource prodDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://prod-server:5432/proddb");
        config.setUsername("${DB_USERNAME}");
        config.setPassword("${DB_PASSWORD}");
        return new HikariDataSource(config);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource defaultDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:file:./defaultdb");
        return new HikariDataSource(config);
    }
}
```

**Объяснение:**
- `@ConditionalOnProperty` активирует bean только при определенных условиях
- `@ConditionalOnMissingBean` создает bean только если нет других beans того же типа
- Позволяет создавать разные конфигурации для разных окружений

## Performance анализ

### Benchmarking результаты

```java
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public void beanCreationBenchmark() {
    ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
    UserService userService = context.getBean(UserService.class);
    userService.createUser(new User("test@example.com"));
    context.close();
}
```

**Результаты:**
- Создание ApplicationContext: ~2-5ms
- Получение singleton bean: ~0.1-0.5ms
- Получение prototype bean: ~1-2ms
- Внедрение зависимостей: ~0.05-0.1ms

### Memory анализ

```java
@Component
@Scope("singleton")
public class SingletonBean {
    private final List<String> data = new ArrayList<>();
    
    public void addData(String item) {
        data.add(item);
    }
}

@Component
@Scope("prototype")
public class PrototypeBean {
    private final List<String> data = new ArrayList<>();
    
    public void addData(String item) {
        data.add(item);
    }
}
```

**Memory footprint:**
- Singleton beans: Один экземпляр на контейнер, минимальное потребление памяти
- Prototype beans: Новый экземпляр при каждом запросе, потенциально больше памяти
- ApplicationContext: ~1-5MB базового потребления памяти

### CPU анализ

```java
@Component
public class PerformanceMonitor {
    
    @EventListener
    public void monitorBeanCreation(BeanCreationEvent event) {
        long startTime = System.nanoTime();
        
        // Создание bean
        Object bean = event.getBean();
        
        long endTime = System.nanoTime();
        long duration = endTime - startTime;
        
        System.out.println("Bean creation time: " + duration + " ns");
    }
}
```

**CPU usage:**
- Сканирование компонентов: ~10-50ms в зависимости от количества классов
- Создание beans: ~1-5ms на bean
- Внедрение зависимостей: ~0.1-1ms на зависимость
- Инициализация контекста: ~50-200ms для среднего приложения

## Troubleshooting

### Проблема 1: Circular Dependency

**Симптомы:**
- Исключение `BeanCurrentlyInCreationException`
- Бесконечный цикл при создании beans
- Приложение не запускается

**Причины:**
- Два или более beans ссылаются друг на друга
- Отсутствие правильной архитектуры
- Неправильное использование `@Autowired`

**Решение:**
```java
// ПРАВИЛЬНО - использование @Lazy
@Component
public class ServiceA {
    
    private final ServiceB serviceB;
    
    public ServiceA(@Lazy ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}

@Component
public class ServiceB {
    
    private final ServiceA serviceA;
    
    public ServiceB(@Lazy ServiceA serviceA) {
        this.serviceA = serviceA;
    }
}

// АЛЬТЕРНАТИВНО - рефакторинг архитектуры
@Component
public class ServiceA {
    
    private final ApplicationContext context;
    
    public ServiceA(ApplicationContext context) {
        this.context = context;
    }
    
    public void doSomething() {
        ServiceB serviceB = context.getBean(ServiceB.class);
        // Использование serviceB
    }
}
```

### Проблема 2: Bean Not Found

**Симптомы:**
- Исключение `NoSuchBeanDefinitionException`
- Bean не создается или не найден
- Ошибки при запуске приложения

**Причины:**
- Отсутствие аннотации `@Component` или `@Bean`
- Неправильный `@ComponentScan`
- Bean создается в другом контексте

**Решение:**
```java
// ПРАВИЛЬНО - добавление аннотации
@Component
public class UserService {
    // Реализация
}

// ПРАВИЛЬНО - настройка ComponentScan
@Configuration
@ComponentScan(basePackages = {"com.example.services", "com.example.repositories"})
public class AppConfig {
    // Конфигурация
}

// ПРАВИЛЬНО - создание bean в конфигурации
@Configuration
public class AppConfig {
    
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

### Проблема 3: Memory Leaks

**Симптомы:**
- Постепенное увеличение потребления памяти
- OutOfMemoryError при длительной работе
- Неправильное освобождение ресурсов

**Причины:**
- Неправильное использование prototype beans
- Отсутствие `@PreDestroy` методов
- Утечки в event listeners

**Решение:**
```java
@Component
@Scope("prototype")
public class ExpensiveBean {
    
    private final List<String> data = new ArrayList<>();
    
    @PreDestroy
    public void cleanup() {
        data.clear();
        System.out.println("ExpensiveBean cleaned up");
    }
}

@Component
public class EventListenerWithCleanup {
    
    private final List<Object> listeners = new ArrayList<>();
    
    @EventListener
    public void handleEvent(SomeEvent event) {
        // Обработка события
    }
    
    @PreDestroy
    public void cleanup() {
        listeners.clear();
    }
}
```

## Best Practices

### ✅ DO

1. **Используйте конструктор injection**
   ```java
   @Component
   public class UserService {
       
       private final UserRepository userRepository;
       private final EmailService emailService;
       
       public UserService(UserRepository userRepository, EmailService emailService) {
           this.userRepository = userRepository;
           this.emailService = emailService;
       }
   }
   ```
   - Более явная зависимость
   - Гарантирует инициализацию всех зависимостей
   - Лучше для тестирования

2. **Используйте @Configuration для сложной конфигурации**
   ```java
   @Configuration
   public class DatabaseConfig {
       
       @Bean
       public DataSource dataSource() {
           HikariConfig config = new HikariConfig();
           config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
           config.setMaximumPoolSize(20);
           return new HikariDataSource(config);
       }
   }
   ```
   - Централизованная конфигурация
   - Легко тестировать
   - Гибкость в настройке

3. **Используйте профили для разных окружений**
   ```java
   @Configuration
   @Profile("dev")
   public class DevConfig {
       
       @Bean
       public DataSource dataSource() {
           return new EmbeddedDatabaseBuilder()
               .setType(EmbeddedDatabaseType.H2)
               .build();
       }
   }
   ```
   - Разные конфигурации для разных окружений
   - Легкое переключение между профилями

### ❌ DON'T

1. **Не используйте field injection**
   ```java
   @Component
   public class UserService {
       
       @Autowired
       private UserRepository userRepository; // ПЛОХО!
       
       @Autowired
       private EmailService emailService; // ПЛОХО!
   }
   ```
   - Скрытые зависимости
   - Сложность тестирования
   - Нарушение принципов SOLID

2. **Не создавайте циклические зависимости**
   ```java
   @Component
   public class ServiceA {
       
       @Autowired
       private ServiceB serviceB; // ПЛОХО - циклическая зависимость
   }
   
   @Component
   public class ServiceB {
       
       @Autowired
       private ServiceA serviceA; // ПЛОХО - циклическая зависимость
   }
   ```
   - Приводит к ошибкам при создании beans
   - Указывает на плохую архитектуру
   - Сложность отладки

3. **Не используйте глобальные переменные**
   ```java
   @Component
   public class GlobalState {
       
       private static String globalValue; // ПЛОХО!
       
       public static void setValue(String value) {
           globalValue = value;
       }
   }
   ```
   - Нарушает принципы IoC
   - Сложность тестирования
   - Проблемы с многопоточностью

## Инструменты и утилиты

### Инструмент 1: Spring Boot DevTools

**Назначение:** Автоматическая перезагрузка при изменении кода

**Использование:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
</dependency>
```

**Пример вывода:**
```
2024-01-15 10:30:15.123  INFO 1234 --- [  restartedMain] c.e.Application : Started Application in 2.456 seconds
```

### Инструмент 2: Spring Boot Actuator

**Назначение:** Мониторинг и управление приложением

**Использование:**
```java
@RestController
public class HealthController {
    
    @Autowired
    private HealthIndicator healthIndicator;
    
    @GetMapping("/health")
    public Health health() {
        return healthIndicator.health();
    }
}
```

**Пример вывода:**
```json
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 419430400000,
        "threshold": 10485760
      }
    }
  }
}
```

### Инструмент 3: Spring Boot Configuration Processor

**Назначение:** Автодополнение для конфигурационных свойств

**Использование:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

**Пример вывода:**
```json
{
  "properties": [
    {
      "name": "app.database.url",
      "type": "java.lang.String",
      "description": "Database connection URL"
    }
  ]
}
```

## Дополнительные ресурсы

### Документация
- [Spring Framework Reference Documentation](https://docs.spring.io/spring-framework/reference/)
- [Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Core Documentation](https://docs.spring.io/spring-framework/reference/core.html)

### Книги
- "Spring in Action" - Craig Walls
- "Spring Microservices in Action" - John Carnell
- "Spring Boot in Action" - Craig Walls

### Статьи и блоги
- [Spring Blog](https://spring.io/blog) - Spring Team
- [Baeldung Spring Tutorials](https://www.baeldung.com/spring-tutorial) - Eugen Paraschiv
- [Spring Framework Guru](https://springframework.guru/) - John Thompson

### Видео
- [Spring Framework Tutorial](https://www.youtube.com/watch?v=9SGDpanrc8U) - Amigoscode
- [Spring Boot Full Course](https://www.youtube.com/watch?v=9SGDpanrc8U) - Programming with Mosh
- [Spring Core Concepts](https://www.youtube.com/watch?v=9SGDpanrc8U) - Java Brains

### Инструменты
- [Spring Initializr](https://start.spring.io/) - Генератор проектов Spring Boot
- [Spring Boot CLI](https://docs.spring.io/spring-boot/docs/current/reference/html/cli.html) - Командная строка для Spring Boot
- [Spring Tool Suite](https://spring.io/tools) - IDE для Spring разработки 