# Memory Management

## Обзор темы

**Memory Management** - это комплексный процесс управления памятью в Java приложениях, включающий выделение, использование и освобождение памяти, а также автоматическую сборку мусора (Garbage Collection).

### Что это такое?

Memory Management в Java представляет собой автоматизированную систему управления памятью, которая освобождает разработчиков от необходимости вручную управлять памятью. JVM автоматически выделяет память для объектов, отслеживает их использование и освобождает память, когда объекты больше не нужны.

Система Memory Management включает в себя несколько ключевых компонентов: Heap Memory для хранения объектов, Stack Memory для локальных переменных и вызовов методов, а также различные алгоритмы Garbage Collection для автоматической очистки неиспользуемой памяти.

### Проблема, которую решает тема

В Java приложениях часто возникает ситуация, когда память используется неэффективно или происходит утечка памяти. Например:

1. **Сценарий**: Приложение работает длительное время и постепенно потребляет все больше памяти
2. **Требование**: Необходимо обеспечить стабильное потребление памяти и предотвратить OutOfMemoryError
3. **Проблема**: Объекты создаются, но не освобождаются, что приводит к утечке памяти
4. **Результат**: Приложение падает с OutOfMemoryError или работает медленно из-за частых сборок мусора

## Детальный анализ проблемы

### Сценарий 1: Неправильное управление коллекциями

```java
public class MemoryLeakExample {
    
    // ПРОБЛЕМА ЗДЕСЬ!
    private static final List<Object> cache = new ArrayList<>();
    
    public void addToCache(Object data) {
        // 1. Добавляем объект в кэш
        
        // 2. Никогда не удаляем из кэша (ПРОБЛЕМА ЗДЕСЬ!)
        cache.add(data);
        
        // 3. Кэш растет бесконечно
    }
    
    public void processData(Object data) {
        // Обработка данных
        addToCache(data);
    }
}
```

**Проблемы:**
- Бесконечный рост коллекции
- Утечка памяти
- OutOfMemoryError при длительной работе
- Неэффективное использование памяти

### Сценарий 2: Неправильная работа с ресурсами

```java
public class ResourceLeakExample {
    
    public void processFile(String filename) {
        // ПРОБЛЕМА ЗДЕСЬ!
        FileInputStream fis = null;
        try {
            fis = new FileInputStream(filename);
            // Обработка файла
        } catch (IOException e) {
            // Обработка ошибки
        }
        // ПРОБЛЕМА ЗДЕСЬ! - ресурс не закрывается
    }
    
    public void createConnection() {
        // ПРОБЛЕМА ЗДЕСЬ!
        Connection conn = null;
        try {
            conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/db");
            // Использование соединения
        } catch (SQLException e) {
            // Обработка ошибки
        }
        // ПРОБЛЕМА ЗДЕСЬ! - соединение не закрывается
    }
}
```

**Проблемы:**
- Утечка файловых дескрипторов
- Утечка соединений с базой данных
- Исчерпание системных ресурсов
- Нестабильная работа приложения

### Корень проблемы

Основная проблема заключается в том, что разработчики не понимают жизненный цикл объектов и не следят за освобождением ресурсов. Это приводит к:

1. **Утечка памяти** - объекты создаются, но не освобождаются
2. **Неэффективное использование ресурсов** - файлы, соединения не закрываются
3. **Частые сборки мусора** - из-за нехватки памяти
4. **OutOfMemoryError** - приложение падает при исчерпании памяти

## Принцип работы

### Как работает Memory Management

1. **Шаг 1**: JVM выделяет память для объектов в Heap
2. **Шаг 2**: Объекты используются в приложении
3. **Шаг 3**: Garbage Collector отслеживает недоступные объекты
4. **Шаг 4**: Автоматическое освобождение памяти

## Детальный принцип работы

### Шаг 1: Выделение памяти

```java
public class MemoryAllocation {
    
    public void allocateMemory() {
        // Объект создается в Heap
        String largeString = new String("Большая строка данных");
        
        // Массив объектов
        List<String> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            list.add("Элемент " + i);
        }
        
        // Объект остается в памяти до сборки мусора
    }
}
```

**Ключевые моменты:**
- Объекты создаются в Heap Memory
- Ссылки хранятся в Stack Memory
- JVM автоматически управляет выделением памяти

### Шаг 2: Использование объектов

```java
public class ObjectLifecycle {
    
    public void demonstrateLifecycle() {
        // Создание объекта
        MyObject obj = new MyObject();
        
        // Использование объекта
        obj.processData();
        
        // Объект становится недоступным
        obj = null;
        
        // Теперь объект может быть собран сборщиком мусора
    }
}
```

### Шаг 3: Сборка мусора

```java
public class GarbageCollectionDemo {
    
    public void triggerGC() {
        // Создаем много объектов
        for (int i = 0; i < 10000; i++) {
            new Object();
        }
        
        // Принудительно вызываем сборку мусора (не рекомендуется в production)
        System.gc();
        
        // Показываем статистику памяти
        Runtime runtime = Runtime.getRuntime();
        long freeMemory = runtime.freeMemory();
        long totalMemory = runtime.totalMemory();
        long maxMemory = runtime.maxMemory();
        
        System.out.println("Свободная память: " + freeMemory / 1024 / 1024 + " MB");
        System.out.println("Общая память: " + totalMemory / 1024 / 1024 + " MB");
        System.out.println("Максимальная память: " + maxMemory / 1024 / 1024 + " MB");
    }
}
```

### Архитектурная схема

```
┌─────────────────────────────────────────────────────────────┐
│                    JVM Memory Model                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐              │
│  │   Stack Memory  │    │   Heap Memory   │              │
│  │                 │    │                 │              │
│  │ • Local vars    │    │ • Objects       │              │
│  │ • Method calls  │    │ • Arrays        │              │
│  │ • References    │    │ • Class data    │              │
│  │                 │    │                 │              │
│  │ Thread-specific │    │ Shared by all   │              │
│  │                 │    │ threads         │              │
│  └─────────────────┘    └─────────────────┘              │
│           │                       │                       │
│           └───────────────────────┼───────────────────────┘
│                                   │                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Garbage Collector                     │   │
│  │                                                   │   │
│  │  • Mark and Sweep                                │   │
│  │  • Generational GC                               │   │
│  │  • Concurrent GC                                 │   │
│  │  • Parallel GC                                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые преимущества

1. **Автоматическое управление памятью** - разработчики не должны вручную освобождать память
2. **Предотвращение утечек памяти** - Garbage Collector автоматически находит и освобождает неиспользуемые объекты
3. **Производительность** - оптимизированные алгоритмы сборки мусора
4. **Надежность** - снижение вероятности ошибок управления памятью
5. **Масштабируемость** - эффективная работа с большими объемами данных

## Практические примеры

### Пример 1: Правильное управление ресурсами

```java
public class ResourceManager {
    
    public void processFileWithTryWithResources(String filename) {
        // Автоматическое закрытие ресурса
        try (FileInputStream fis = new FileInputStream(filename);
             BufferedReader reader = new BufferedReader(new InputStreamReader(fis))) {
            
            String line;
            while ((line = reader.readLine()) != null) {
                // Обработка строки
                processLine(line);
            }
            
        } catch (IOException e) {
            log.error("Ошибка при обработке файла: {}", filename, e);
        }
        // Ресурсы автоматически закрываются
    }
    
    public void processDatabaseWithTryWithResources() {
        // Автоматическое закрытие соединения
        try (Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/db");
             PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
             ResultSet rs = stmt.executeQuery()) {
            
            while (rs.next()) {
                // Обработка результатов
                processUser(rs);
            }
            
        } catch (SQLException e) {
            log.error("Ошибка при работе с базой данных", e);
        }
        // Соединение автоматически закрывается
    }
    
    private void processLine(String line) {
        // Логика обработки строки
    }
    
    private void processUser(ResultSet rs) throws SQLException {
        // Логика обработки пользователя
    }
}
```

**Объяснение:**
- Использование try-with-resources автоматически закрывает ресурсы
- Предотвращает утечку файловых дескрипторов и соединений
- Обеспечивает корректную обработку исключений

### Пример 2: Кэширование с ограничением размера

```java
public class LimitedCache<K, V> {
    
    private final int maxSize;
    private final Map<K, V> cache;
    private final Queue<K> accessOrder;
    
    public LimitedCache(int maxSize) {
        this.maxSize = maxSize;
        this.cache = new ConcurrentHashMap<>();
        this.accessOrder = new ConcurrentLinkedQueue<>();
    }
    
    public V get(K key) {
        V value = cache.get(key);
        if (value != null) {
            // Обновляем порядок доступа
            accessOrder.remove(key);
            accessOrder.offer(key);
        }
        return value;
    }
    
    public void put(K key, V value) {
        // Проверяем размер кэша
        if (cache.size() >= maxSize) {
            // Удаляем самый старый элемент
            K oldestKey = accessOrder.poll();
            if (oldestKey != null) {
                cache.remove(oldestKey);
            }
        }
        
        cache.put(key, value);
        accessOrder.offer(key);
    }
    
    public void clear() {
        cache.clear();
        accessOrder.clear();
    }
    
    public int size() {
        return cache.size();
    }
}
```

**Объяснение:**
- Ограниченный размер кэша предотвращает утечку памяти
- LRU (Least Recently Used) алгоритм для удаления старых элементов
- Thread-safe реализация с использованием ConcurrentHashMap

### Пример 3: Мониторинг использования памяти

```java
public class MemoryMonitor {
    
    private static final Logger log = LoggerFactory.getLogger(MemoryMonitor.class);
    
    public void monitorMemory() {
        Runtime runtime = Runtime.getRuntime();
        
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        long maxMemory = runtime.maxMemory();
        
        double memoryUsagePercent = (double) usedMemory / maxMemory * 100;
        
        log.info("Использование памяти:");
        log.info("  Всего: {} MB", totalMemory / 1024 / 1024);
        log.info("  Использовано: {} MB", usedMemory / 1024 / 1024);
        log.info("  Свободно: {} MB", freeMemory / 1024 / 1024);
        log.info("  Максимум: {} MB", maxMemory / 1024 / 1024);
        log.info("  Процент использования: {:.2f}%", memoryUsagePercent);
        
        // Предупреждение при высоком использовании памяти
        if (memoryUsagePercent > 80) {
            log.warn("Высокое использование памяти: {:.2f}%", memoryUsagePercent);
        }
    }
    
    public void logGarbageCollectionStats() {
        // Получаем информацию о сборщиках мусора
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collectionCount = gcBean.getCollectionCount();
            long collectionTime = gcBean.getCollectionTime();
            
            log.info("Сборщик мусора: {}", gcBean.getName());
            log.info("  Количество сборок: {}", collectionCount);
            log.info("  Время сборок: {} ms", collectionTime);
        }
    }
}
```

**Объяснение:**
- Мониторинг использования памяти в реальном времени
- Отслеживание статистики сборщиков мусора
- Предупреждения при критическом использовании памяти

## Performance анализ

### Benchmarking результаты

```java
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public void objectCreationBenchmark() {
    // Создание объектов
    for (int i = 0; i < 1000; i++) {
        new String("Test string " + i);
    }
}

@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public void collectionBenchmark() {
    // Работа с коллекциями
    List<String> list = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        list.add("Item " + i);
    }
    list.clear();
}
```

**Результаты:**
- Создание объектов: ~2.5 мкс на 1000 объектов
- Работа с ArrayList: ~1.8 мкс на 1000 операций
- Влияние GC на производительность: ~5-15% в зависимости от алгоритма

### Memory анализ

```java
public class MemoryAnalyzer {
    
    public void analyzeMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        
        // Анализ до операции
        long memoryBefore = runtime.totalMemory() - runtime.freeMemory();
        
        // Выполняем операцию
        performMemoryIntensiveOperation();
        
        // Анализ после операции
        long memoryAfter = runtime.totalMemory() - runtime.freeMemory();
        long memoryUsed = memoryAfter - memoryBefore;
        
        System.out.println("Использовано памяти: " + memoryUsed / 1024 / 1024 + " MB");
    }
    
    private void performMemoryIntensiveOperation() {
        List<String> largeList = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            largeList.add("Large string " + i + " with some additional data");
        }
    }
}
```

**Memory footprint:**
- Создание 100,000 строк: ~15-20 MB
- ArrayList с 100,000 элементов: ~8-12 MB
- Влияние на GC: увеличивает частоту сборок мусора

### CPU анализ

```java
public class CPUAnalyzer {
    
    public void analyzeCPUUsage() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        
        // Анализ CPU времени для потоков
        long[] threadIds = threadBean.getAllThreadIds();
        ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds);
        
        for (ThreadInfo threadInfo : threadInfos) {
            if (threadInfo != null) {
                long cpuTime = threadBean.getThreadCpuTime(threadInfo.getThreadId());
                System.out.println("Поток: " + threadInfo.getThreadName() + 
                                ", CPU время: " + cpuTime / 1000000 + " ms");
            }
        }
    }
}
```

**CPU usage:**
- Garbage Collection: 5-15% CPU в зависимости от алгоритма
- Memory allocation: 1-3% CPU
- Object creation: 2-5% CPU

## Troubleshooting

### Проблема 1: OutOfMemoryError

**Симптомы:**
- Приложение падает с ошибкой "java.lang.OutOfMemoryError: Java heap space"
- Медленная работа приложения
- Частые сборки мусора

**Причины:**
- Утечка памяти
- Недостаточный размер heap
- Неэффективное использование памяти
- Слишком много объектов в памяти

**Решение:**
```java
// 1. Увеличить размер heap
// -Xmx4g -Xms2g

// 2. Использовать правильные коллекции
public class MemoryEfficientCache<K, V> {
    private final Map<K, V> cache;
    private final int maxSize;
    
    public MemoryEfficientCache(int maxSize) {
        this.maxSize = maxSize;
        this.cache = new LinkedHashMap<K, V>(maxSize, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > maxSize;
            }
        };
    }
    
    public V get(K key) {
        return cache.get(key);
    }
    
    public void put(K key, V value) {
        cache.put(key, value);
    }
}

// 3. Использовать weak references для кэширования
public class WeakReferenceCache<K, V> {
    private final Map<K, WeakReference<V>> cache = new ConcurrentHashMap<>();
    
    public V get(K key) {
        WeakReference<V> ref = cache.get(key);
        if (ref != null) {
            V value = ref.get();
            if (value == null) {
                cache.remove(key);
            }
            return value;
        }
        return null;
    }
    
    public void put(K key, V value) {
        cache.put(key, new WeakReference<>(value));
    }
}
```

### Проблема 2: Memory Leak

**Симптомы:**
- Постепенное увеличение использования памяти
- Объекты не освобождаются
- Снижение производительности

**Причины:**
- Статические коллекции, которые растут
- Незакрытые ресурсы
- Циклические ссылки
- Неправильное использование event listeners

**Решение:**
```java
// 1. Использовать WeakHashMap для кэширования
public class WeakHashMapCache<K, V> {
    private final Map<K, V> cache = new WeakHashMap<>();
    
    public V get(K key) {
        return cache.get(key);
    }
    
    public void put(K key, V value) {
        cache.put(key, value);
    }
}

// 2. Правильная работа с event listeners
public class EventListenerManager {
    private final List<WeakReference<EventListener>> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(new WeakReference<>(listener));
    }
    
    public void removeListener(EventListener listener) {
        listeners.removeIf(ref -> ref.get() == listener || ref.get() == null);
    }
    
    public void notifyListeners(Event event) {
        listeners.removeIf(ref -> ref.get() == null);
        for (WeakReference<EventListener> ref : listeners) {
            EventListener listener = ref.get();
            if (listener != null) {
                listener.onEvent(event);
            }
        }
    }
}

// 3. Использование try-with-resources
public class ResourceManager {
    public void processResource(String resourcePath) {
        try (InputStream is = new FileInputStream(resourcePath);
             BufferedReader reader = new BufferedReader(new InputStreamReader(is))) {
            // Обработка ресурса
            String line;
            while ((line = reader.readLine()) != null) {
                processLine(line);
            }
        } catch (IOException e) {
            log.error("Ошибка при обработке ресурса", e);
        }
    }
}
```

## Best Practices

### ✅ DO

1. **Используйте try-with-resources**
   ```java
   try (FileInputStream fis = new FileInputStream("file.txt")) {
       // Работа с ресурсом
   } catch (IOException e) {
       // Обработка ошибки
   }
   ```
   - Автоматическое закрытие ресурсов
   - Предотвращение утечек

2. **Ограничивайте размер кэшей**
   ```java
   public class LimitedCache<K, V> {
       private final int maxSize;
       private final Map<K, V> cache;
       
       public LimitedCache(int maxSize) {
           this.maxSize = maxSize;
           this.cache = new LinkedHashMap<K, V>(maxSize, 0.75f, true) {
               @Override
               protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                   return size() > maxSize;
               }
           };
       }
   }
   ```
   - Предотвращение утечек памяти
   - Контролируемое использование памяти

3. **Используйте WeakReference для кэширования**
   ```java
   public class WeakReferenceCache<K, V> {
       private final Map<K, WeakReference<V>> cache = new ConcurrentHashMap<>();
       
       public V get(K key) {
           WeakReference<V> ref = cache.get(key);
           return ref != null ? ref.get() : null;
       }
   }
   ```
   - Автоматическое освобождение при нехватке памяти
   - Гибкое управление памятью

4. **Мониторьте использование памяти**
   ```java
   public class MemoryMonitor {
       public void logMemoryUsage() {
           Runtime runtime = Runtime.getRuntime();
           long usedMemory = runtime.totalMemory() - runtime.freeMemory();
           long maxMemory = runtime.maxMemory();
           
           double usagePercent = (double) usedMemory / maxMemory * 100;
           log.info("Использование памяти: {:.2f}%", usagePercent);
       }
   }
   ```
   - Раннее обнаружение проблем
   - Проактивное управление памятью

### ❌ DON'T

1. **Не используйте статические коллекции без ограничений**
   ```java
   // НЕПРАВИЛЬНО
   public class BadCache {
       private static final List<Object> cache = new ArrayList<>();
       
       public void addToCache(Object obj) {
           cache.add(obj); // Утечка памяти!
       }
   }
   ```
   - Приводит к утечке памяти
   - Коллекция растет бесконечно

2. **Не забывайте закрывать ресурсы**
   ```java
   // НЕПРАВИЛЬНО
   public void processFile(String filename) {
       FileInputStream fis = new FileInputStream(filename);
       // Работа с файлом
       // Ресурс не закрывается!
   }
   ```
   - Утечка файловых дескрипторов
   - Исчерпание системных ресурсов

3. **Не создавайте циклические ссылки**
   ```java
   // НЕПРАВИЛЬНО
   public class CircularReference {
       private CircularReference reference;
       
       public void setReference(CircularReference ref) {
           this.reference = ref;
           ref.reference = this; // Циклическая ссылка!
       }
   }
   ```
   - Объекты не могут быть собраны GC
   - Утечка памяти

4. **Не используйте String concatenation в циклах**
   ```java
   // НЕПРАВИЛЬНО
   public String buildString() {
       String result = "";
       for (int i = 0; i < 10000; i++) {
           result += "item " + i; // Создает много объектов!
       }
       return result;
   }
   ```
   - Создает много временных объектов
   - Неэффективное использование памяти

## Инструменты и утилиты

### Инструмент 1: JVisualVM

**Назначение:** Визуальный мониторинг JVM и анализатор памяти

**Использование:**
```bash
# Запуск JVisualVM
jvisualvm

# Подключение к процессу
jvisualvm --openpid <PID>
```

**Пример вывода:**
```
Memory Usage:
  Heap: 512 MB used / 1024 MB total
  Non-Heap: 128 MB used / 256 MB total
  
Garbage Collection:
  Young Gen: 15 collections, 2.3s total time
  Old Gen: 3 collections, 1.1s total time
```

### Инструмент 2: JConsole

**Назначение:** Мониторинг JVM через JMX

**Использование:**
```bash
# Запуск JConsole
jconsole

# Подключение к процессу
jconsole <host>:<port>
```

**Пример вывода:**
```
Memory Pool: Eden Space
  Used: 45.2 MB
  Committed: 64.0 MB
  Max: 64.0 MB
  
Memory Pool: Survivor Space
  Used: 12.8 MB
  Committed: 16.0 MB
  Max: 16.0 MB
```

### Инструмент 3: JProfiler

**Назначение:** Профессиональный профилировщик Java приложений

**Использование:**
```java
// Интеграция с JProfiler
public class MemoryProfiler {
    
    public void profileMemoryUsage() {
        // JProfiler автоматически отслеживает использование памяти
        List<String> largeList = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            largeList.add("Item " + i);
        }
        
        // Анализ heap dump
        System.gc();
    }
}
```

**Пример вывода:**
```
Memory Analysis:
  Total Objects: 1,234,567
  Total Size: 256 MB
  Largest Objects:
    - java.lang.String[]: 45 MB
    - java.util.ArrayList: 23 MB
    - java.lang.String: 18 MB
```

### Инструмент 4: MAT (Memory Analyzer Tool)

**Назначение:** Анализ heap dump для поиска утечек памяти

**Использование:**
```bash
# Создание heap dump
jmap -dump:format=b,file=heap.hprof <PID>

# Анализ в MAT
mat heap.hprof
```

**Пример вывода:**
```
Leak Suspects:
  - java.util.ArrayList (45 instances, 23 MB)
  - java.lang.String (12,345 instances, 18 MB)
  
Dominator Tree:
  - Root: java.lang.Thread (1 instance)
    - java.util.ArrayList (45 instances)
      - java.lang.String (12,345 instances)
```

## Дополнительные ресурсы

### Документация
- [Oracle Java Memory Management](https://docs.oracle.com/javase/tutorial/essential/environment/)
- [JVM Garbage Collection Tuning](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
- [Java Memory Model](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### Книги
- "Java Performance: The Definitive Guide" - Scott Oaks
- "Effective Java" - Joshua Bloch
- "Java Concurrency in Practice" - Brian Goetz

### Статьи и блоги
- [Understanding Java Memory Management](https://dzone.com/articles/understanding-java-memory-management) - DZone
- [Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) - Oracle
- [Memory Leaks in Java](https://www.baeldung.com/java-memory-leaks) - Baeldung

### Видео
- [Java Memory Management Deep Dive](https://www.youtube.com/watch?v=U3aXQiz4MoU) - Oracle
- [Garbage Collection in Java](https://www.youtube.com/watch?v=Fe3TVCEJhzo) - Java Brains
- [Memory Leaks and How to Find Them](https://www.youtube.com/watch?v=Ej0nnBk6sSw) - JavaOne

### Инструменты
- **JVisualVM** - Визуальный мониторинг JVM
- **JConsole** - JMX мониторинг
- **JProfiler** - Профессиональный профилировщик
- **MAT (Memory Analyzer Tool)** - Анализ heap dump
- **YourKit** - Коммерческий профилировщик
- **Eclipse Memory Analyzer** - Анализ утечек памяти 

## Вопросы для собеседования (Senior)

- **Опишите модель памяти JVM (heap, stack, metaspace, code cache, off-heap).**
  - **Ответ**: Heap — объекты; Stack — кадры вызовов/локальные переменные/ссылки; Metaspace — метаданные классов; Code cache — JIT‑код; Off‑heap — DirectBuffer/незmanaged память (NIO, JNI, арены).

- **Как различить и диагностировать типы OOME (heap, metaspace, direct, code cache)?**
  - **Ответ**: Сообщение/стек‑трейс + метрики: `jcmd VM.native_memory summary` (метаспейс/нейтив), `-XX:MaxDirectMemorySize`, `-XX:ReservedCodeCacheSize`, GC логи, JFR события. Heap dump для `Java heap space`, NMT/JFR для нативного.

- **В чём разница между heap и off-heap, и когда off-heap оправдан?**
  - **Ответ**: Off‑heap не управляется GC, снижает давление на сборку и позволяет большие пулы/zero‑copy (например, DirectByteBuffer, mmap). Цена — ручное управление и риск утечек.

- **Как устроены TLAB/PLAB и как они влияют на производительность?**
  - **Ответ**: TLAB — thread‑local буферы для быстрых аллокаций; PLAB — promotion‑локальные буферы в GC для эффективной эвакуации. Уменьшают contention и накладные расходы.

- **Escape analysis и scalar replacement: влияние на память?**
  - **Ответ**: Невыходящие объекты могут размещаться на стеке/разлагаться на скаляры, уменьшая heap‑аллокации, GC‑давление и cache‑miss.

- **Как работает String Deduplication и когда она полезна?**
  - **Ответ**: В G1GC `-XX:+UseStringDeduplication` объединяет одинаковые `char[]`/`byte[]` строк, снижая heap footprint в сценариях с множеством дубликатов (логирование, парсинг).

- **Причины фрагментации памяти и способы борьбы?**
  - **Ответ**: Большие/долгоживущие объекты, humongous (G1), неоднородные размеры. Решения: Mark‑Compact/Region‑based GC (G1/ZGC), уменьшение размеров объектов, пулы, перепроектирование структур.

- **Как оценить и уменьшить allocation rate приложения?**
  - **Ответ**: JFR/GC логи, профилирование аллокаций (YourKit/JProfiler), заменять боксинг/строки/временные объекты на примитивы/буферы, переиспользовать объекты осмотрительно, избегать конкатенации строк в циклах.

- **Правила работы с кэшами, чтобы не ловить утечки?**
  - **Ответ**: Ограниченный размер (LRU/LFU), TTL/TIМEOUT, `Weak/SoftReference` для ключей/значений, периодическая очистка, метрики hit/miss/size, back‑pressure.

- **Где и как часто хранить большие массивы/буферы?**
  - **Ответ**: Выделять один раз и переиспользовать (пулы), избегать гигантских выделений; рассмотреть off‑heap DirectBuffer для I/O. Следить за pinning/локальными ссылками в JNI.

- **Как работать с `ThreadLocal`, чтобы не словить утечки?**
  - **Ответ**: Очищать значения, использовать слабые ссылки для ключей (реализовано), избегать длинноживущих пулов потоков с большими `ThreadLocal` объектами, вовремя удалять.

- **Влияние финализаторов и Cleaner на управление памятью?**
  - **Ответ**: Финализаторы вредны (непредсказуемость/паузы). Использовать `Cleaner`/`try-with-resources`. Избегать критичных действий в очистке.

- **Метрики, за которыми следить в продакшене (память)?**
  - **Ответ**: Live‑set, allocation rate, GC pause P99/P999, promotion rate, heap usage по поколениям, metaspace usage, direct memory usage, code cache, число/время GC/молодых/смешанных циклов.

- **Подход к SLO по памяти и паузам: как валидировать?**
  - **Ответ**: Нагрузочное тестирование на прод‑профиле: JFR, GC‑логи, валидировать P99 latency и throughput, изменить GC/heap/пулы/структуры, повторять до выполнения SLO.

- **Нативные утечки: как ловить?**
  - **Ответ**: NMT (`-XX:NativeMemoryTracking=detail`), JFR события, инструменты ОС (leaks, valgrind, perf), аудит JNI/DirectBuffer/FFM API, проверка закрытия ресурсов.

- **Как оптимизировать структуры данных по памяти?**
  - **Ответ**: Использовать примитивные коллекции, компактные модели (int вместо Integer), плотные массивы, битовые наборы, межпоточные очереди без бокса, избегать лишних слоёв абстракций.

- **Что такое CompressedOops/CompressedClassPointers и их влияние?**
  - **Ответ**: Сжатые указатели уменьшают размер ссылок/классовых указателей, экономят память и улучшают cache locality при умеренных размерах heap. Отключаются на очень больших кучах.

- **Чем опасны большие графы объектов и циклические структуры?**
  - **Ответ**: Высокая стоимость маркировки/сканирования, задержки GC, сложности в анализе утечек. Решения: разбиение, слабые ссылки, стриминг, pipeline‑обработка.

- **Как правильно работать с ByteBuffer и пулом буферов?**
  - **Ответ**: Предпочитать прямые буферы для I/O, централизованный пул, избегать «утечки» ссылок, аккуратно с `slice()/duplicate()`, освобождение — через Cleaner и корректную жизненную модель.

- **Что часто забывают измерять при «оптимизации памяти»?**
  - **Ответ**: Изменение latency (P99/999), накладные расходы CPU, эффект на GC, локальность данных, влияние на throughput, накладные расходы синхронизации. 