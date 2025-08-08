# JPA/Hibernate - Компактная версия

## 🎯 Ключевые концепции (20/80)

### 1. Что такое ORM

**ORM (Object-Relational Mapping)** - технология, которая позволяет работать с базой данных через объекты Java вместо SQL запросов.

```java
// ❌ Без ORM - ручная работа с JDBC
public User findUserById(Long id) {
    String sql = "SELECT id, name, email FROM users WHERE id = ?";
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement(sql)) {
        stmt.setLong(1, id);
        ResultSet rs = stmt.executeQuery();
        if (rs.next()) {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setName(rs.getString("name"));
            user.setEmail(rs.getString("email"));
            return user;
        }
    }
    return null;
}

// ✅ С ORM - декларативный подход
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name")
    private String name;
    
    @Column(name = "email")
    private String email;
    
    // getters, setters
}

// Использование
User user = entityManager.find(User.class, id);
```

### 2. Основные аннотации JPA

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "email", unique = true)
    private String email;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at")
    private Date createdAt;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    private UserStatus status;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;
    
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "department_id")
    private Department department;
}
```

### 3. Типы связей

#### One-to-One
```java
@Entity
public class User {
    @Id
    private Long id;
    
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id
    private Long id;
    
    @OneToOne(mappedBy = "profile")
    private User user;
}
```

#### One-to-Many
```java
@Entity
public class Department {
    @Id
    private Long id;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees;
}

@Entity
public class Employee {
    @Id
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```

#### Many-to-Many
```java
@Entity
public class Student {
    @Id
    private Long id;
    
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses;
}

@Entity
public class Course {
    @Id
    private Long id;
    
    @ManyToMany(mappedBy = "courses")
    private List<Student> students;
}
```

## ⚡ Критичные моменты для Senior

### 1. Fetch Strategies

```java
// EAGER - загружается сразу
@ManyToOne(fetch = FetchType.EAGER)
private Department department;

// LAZY - загружается при обращении
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;

// Проблема N+1 запросов
// ❌ Плохо
List<User> users = userRepository.findAll();
for (User user : users) {
    System.out.println(user.getOrders().size()); // N+1 запросов
}

// ✅ Хорошо - JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### 2. Cascade Types

```java
// PERSIST - каскадное сохранение
@OneToMany(cascade = CascadeType.PERSIST)
private List<Order> orders;

// REMOVE - каскадное удаление
@OneToMany(cascade = CascadeType.REMOVE)
private List<Order> orders;

// ALL - все операции
@OneToMany(cascade = CascadeType.ALL)
private List<Order> orders;
```

### 3. Entity States

```java
// Transient - не управляется JPA
User user = new User();

// Managed - управляется JPA
entityManager.persist(user); // Transient → Managed
User found = entityManager.find(User.class, 1L); // Managed

// Detached - не управляется JPA
entityManager.detach(user); // Managed → Detached

// Removed - помечен для удаления
entityManager.remove(user); // Managed → Removed
```

### 4. Transaction Management

```java
@Service
@Transactional
public class UserService {
    
    @Transactional(readOnly = true)
    public User findUser(Long id) {
        return userRepository.findById(id);
    }
    
    @Transactional
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @Transactional
    public void updateUser(User user) {
        userRepository.save(user);
    }
}
```

## 🔧 Практические примеры

### 1. Repository Pattern

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простые запросы
    List<User> findByName(String name);
    List<User> findByEmailContaining(String email);
    Optional<User> findByEmail(String email);
    
    // Сложные запросы
    @Query("SELECT u FROM User u WHERE u.createdAt > :date")
    List<User> findRecentUsers(@Param("date") Date date);
    
    @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
    
    // Нативные запросы
    @Query(value = "SELECT * FROM users WHERE status = :status", nativeQuery = true)
    List<User> findByStatus(@Param("status") String status);
}
```

### 2. Auditing

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class User {
    
    @Id
    private Long id;
    
    @CreatedDate
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(name = "created_by")
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}

@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                .map(SecurityContext::getAuthentication)
                .map(Authentication::getName);
    }
}
```

### 3. Performance Optimization

```java
// Batch Processing
@Transactional
public void batchInsert(List<User> users) {
    int batchSize = 50;
    for (int i = 0; i < users.size(); i++) {
        entityManager.persist(users.get(i));
        if (i % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}

// Second Level Cache
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    // ...
}

// Query Cache
@QueryHints(@QueryHint(name = HINT_CACHEABLE, value = "true"))
@Query("SELECT u FROM User u WHERE u.status = :status")
List<User> findByStatus(@Param("status") String status);
```

### 4. Уровни кэширования Hibernate

#### First Level Cache (Session Cache)
```java
// Автоматический кэш на уровне сессии
@Transactional
public void firstLevelCacheExample() {
    // 1. Загружается из БД
    User user1 = userRepository.findById(1L);
    
    // 2. Берется из кэша сессии (без запроса к БД)
    User user2 = userRepository.findById(1L);
    
    // 3. user1 и user2 - это один и тот же объект
    System.out.println(user1 == user2); // true
}
```

#### Second Level Cache (Application Cache)
```java
// Настройка Second Level Cache
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    @Id
    private Long id;
    
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

// Конфигурация
@Configuration
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new EhCacheCacheManager(ehCacheManager());
    }
    
    @Bean
    public EhCacheManagerFactoryBean ehCacheManager() {
        EhCacheManagerFactoryBean factory = new EhCacheManagerFactoryBean();
        factory.setConfigLocation(new ClassPathResource("ehcache.xml"));
        return factory;
    }
}
```

#### Query Cache
```java
// Кэширование результатов запросов
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @QueryHints(@QueryHint(name = HINT_CACHEABLE, value = "true"))
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findByStatus(@Param("status") String status);
}

// Включение Query Cache
@Configuration
public class HibernateConfig {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return hibernateProperties -> {
            hibernateProperties.put("hibernate.cache.use_second_level_cache", "true");
            hibernateProperties.put("hibernate.cache.use_query_cache", "true");
            hibernateProperties.put("hibernate.cache.region.factory_class", 
                "org.hibernate.cache.ehcache.EhCacheRegionFactory");
        };
    }
}
```

#### Стратегии кэширования
```java
@Entity
public class User {
    
    // READ_ONLY - только чтение, не изменяется
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
    private String name;
    
    // READ_WRITE - чтение и запись с блокировками
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private String email;
    
    // NONSTRICT_READ_WRITE - без блокировок, возможны грязные данные
    @Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
    private String status;
    
    // TRANSACTIONAL - полная транзакционность
    @Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)
    private BigDecimal balance;
}
```

### 4. Типичные проблемы и решения

#### N+1 Query Problem
```java
// ❌ Плохо - N+1 запросов
List<User> users = userRepository.findAll();
for (User user : users) {
    System.out.println(user.getOrders().size()); // Lazy loading
}

// ✅ Хорошо - JOIN FETCH
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// ✅ Хорошо - @EntityGraph
@EntityGraph(attributePaths = {"orders"})
List<User> findAllWithOrdersGraph();
```

#### LazyInitializationException
```java
// ❌ Плохо - исключение вне транзакции
@Transactional(readOnly = true)
public List<String> getUserOrderTitles(Long userId) {
    User user = userRepository.findById(userId);
    return user.getOrders().stream() // LazyInitializationException
            .map(Order::getTitle)
            .collect(Collectors.toList());
}

// ✅ Хорошо - загрузка в транзакции
@Transactional(readOnly = true)
public List<String> getUserOrderTitles(Long userId) {
    User user = userRepository.findByIdWithOrders(userId);
    return user.getOrders().stream()
            .map(Order::getTitle)
            .collect(Collectors.toList());
}
```

#### Memory Leaks
```java
// ❌ Плохо - утечка памяти
@Transactional
public void processUsers() {
    List<User> users = userRepository.findAll();
    for (User user : users) {
        // обработка пользователя
        processUser(user);
    }
    // Session остается открытым, объекты в памяти
}

// ✅ Хорошо - очистка сессии
@Transactional
public void processUsers() {
    List<User> users = userRepository.findAll();
    for (User user : users) {
        processUser(user);
        entityManager.detach(user); // очистка из сессии
    }
}
```

## 📋 Чек-лист для Senior

- [ ] Понимаю принципы ORM и их преимущества
- [ ] Умею использовать основные аннотации JPA (@Entity, @Id, @Column, @OneToMany, etc.)
- [ ] Знаю типы связей между сущностями (One-to-One, One-to-Many, Many-to-Many)
- [ ] Понимаю fetch strategies (EAGER vs LAZY) и их влияние на производительность
- [ ] Умею избегать N+1 проблем с помощью JOIN FETCH и @EntityGraph
- [ ] Знаю entity states (Transient, Managed, Detached, Removed)
- [ ] Умею работать с транзакциями и их уровнями изоляции
- [ ] Могу оптимизировать производительность через batch processing и кэширование
- [ ] Умею диагностировать и решать LazyInitializationException
- [ ] Знаю как использовать auditing для отслеживания изменений
- [ ] Понимаю уровни кэширования Hibernate (First Level, Second Level, Query Cache)
- [ ] Умею настраивать стратегии кэширования (READ_ONLY, READ_WRITE, etc.)
- [ ] Могу диагностировать проблемы с кэшированием и их решения

## 🎯 Что спрашивают на интервью

1. **"Объясни принципы ORM и преимущества JPA/Hibernate"**
   - Платформо-независимость, снижение boilerplate кода, автоматическое управление связями

2. **"Разница между EAGER и LAZY loading?"**
   - EAGER загружает сразу, LAZY при обращении. Влияние на производительность и N+1 проблемы

3. **"Что такое N+1 проблема и как её решить?"**
   - Множественные запросы при lazy loading, решения через JOIN FETCH, @EntityGraph, batch fetching

4. **"Какие типы связей знаешь и когда их использовать?"**
   - One-to-One, One-to-Many, Many-to-Many с примерами и каскадными операциями

5. **"Как работает кэширование в Hibernate?"**
   - First Level Cache (Session), Second Level Cache, Query Cache, стратегии кэширования

6. **"Что такое LazyInitializationException и как избежать?"**
   - Исключение при обращении к lazy коллекциям вне транзакции, решения через @Transactional или JOIN FETCH

7. **"Как оптимизировать производительность JPA приложений?"**
    - Batch processing, правильные fetch strategies, кэширование, мониторинг SQL запросов

8. **"Объясни уровни кэширования в Hibernate"**
    - First Level Cache (Session), Second Level Cache (Application), Query Cache, стратегии кэширования

9. **"Когда использовать разные стратегии кэширования?"**
    - READ_ONLY для неизменяемых данных, READ_WRITE для изменяемых, NONSTRICT_READ_WRITE для редко изменяемых

## 📚 Дополнительно (если время есть)

- **Criteria API** - динамические запросы
- **Hibernate Envers** - аудит изменений сущностей
- **Second Level Cache** - настройка EhCache, Redis
- **Query Optimization** - анализ execution plans
- **Spring Data JPA** - дополнительные возможности
- **Database Migration** - Flyway, Liquibase
- **Performance Monitoring** - Hibernate Statistics, SQL logging 