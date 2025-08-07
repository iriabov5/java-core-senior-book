# JVM Архитектура

## Обзор темы

**JVM (Java Virtual Machine)** - это виртуальная машина, которая обеспечивает выполнение Java байткода и управляет памятью приложения.

### Что это такое?

JVM представляет собой абстрактную вычислительную машину, которая обеспечивает платформо-независимое выполнение Java программ. Она работает как слой абстракции между Java кодом и операционной системой, предоставляя единую среду выполнения для Java приложений независимо от того, на какой платформе они запускаются.

JVM состоит из нескольких ключевых компонентов: Class Loader, Memory Management (Heap, Stack, Method Area), Execution Engine и Native Interface. Каждый компонент играет важную роль в жизненном цикле Java приложения, от загрузки классов до управления памятью и выполнения байткода.

### Проблема, которую решает тема

В разработке программного обеспечения часто возникает ситуация, когда нужно обеспечить кроссплатформенность и эффективное управление ресурсами. Например:

1. **Сценарий**: Разработчик создает Java приложение на Windows, но оно должно работать на Linux серверах
2. **Требование**: Обеспечить одинаковое поведение приложения на разных платформах
3. **Проблема**: Без JVM пришлось бы компилировать код для каждой целевой платформы отдельно
4. **Результат**: JVM обеспечивает "Write Once, Run Anywhere" - один байткод работает везде

## Детальный анализ проблемы

### Сценарий 1: Нативная компиляция без JVM

```java
// ПРОБЛЕМА ЗДЕСЬ!
public class NativeCompilationExample {
    
    private static native void platformSpecificOperation();
    
    public static void main(String[] args) {
        // 1. Бизнес-логика приложения
        
        // 2. Вызов нативной функции (ПРОБЛЕМА ЗДЕСЬ!)
        platformSpecificOperation();
        
        // 3. Продолжение выполнения
    }
}
```

**Проблемы:**
- Код должен быть перекомпилирован для каждой платформы
- Сложность поддержки и развертывания
- Высокие затраты на разработку
- Несовместимость между платформами

### Сценарий 2: Прямое управление памятью

```java
public class MemoryManagementExample {
    
    // ПРОБЛЕМА ЗДЕСЬ!
    private static final int BUFFER_SIZE = 1024 * 1024; // 1MB
    private byte[] buffer = new byte[BUFFER_SIZE];
    
    public void processData() {
        // 1. Загрузка данных в буфер
        
        // 2. Обработка данных (ПРОБЛЕМА ЗДЕСЬ!)
        for (int i = 0; i < buffer.length; i++) {
            buffer[i] = (byte) (buffer[i] * 2);
        }
        
        // 3. Освобождение памяти вручную (ПРОБЛЕМА ЗДЕСЬ!)
        buffer = null;
        System.gc(); // Принудительный вызов сборщика мусора
    }
}
```

**Проблемы:**
- Утечки памяти при неправильном управлении
- Сложность отладки проблем с памятью
- Неэффективное использование ресурсов
- Возможность повреждения памяти

### Корень проблемы

Основная проблема заключается в том, что без JVM разработчики должны вручную управлять платформо-зависимыми аспектами и памятью. Это приводит к:

1. **Проблема кроссплатформенности** - необходимость компиляции для каждой платформы
2. **Проблема управления памятью** - ручное выделение и освобождение памяти
3. **Проблема безопасности** - прямой доступ к системным ресурсам
4. **Проблема производительности** - отсутствие оптимизаций на уровне виртуальной машины

## Принцип работы

### Как работает JVM

1. **Шаг 1**: Загрузка классов через Class Loader
2. **Шаг 2**: Размещение объектов в памяти (Heap, Stack, Method Area)
3. **Шаг 3**: Выполнение байткода через Execution Engine
4. **Шаг 4**: Управление памятью через Garbage Collector

## Детальный принцип работы

### Шаг 1: Загрузка классов

```java
public class ClassLoadingExample {
    
    public static void main(String[] args) {
        // JVM автоматически загружает классы при первом обращении
        MyClass obj = new MyClass(); // Class Loader загружает MyClass
        System.out.println("Class loaded: " + obj.getClass().getName());
    }
}

class MyClass {
    static {
        System.out.println("MyClass is being initialized");
    }
}
```

**Ключевые моменты:**
- Bootstrap Class Loader загружает основные Java классы
- Extension Class Loader загружает расширения
- Application Class Loader загружает пользовательские классы

### Шаг 2: Управление памятью

```java
public class MemoryManagementExample {
    
    // Объекты размещаются в Heap
    private List<String> dataList = new ArrayList<>();
    
    // Примитивы и ссылки размещаются в Stack
    public void processData(int count) {
        String localVar = "local"; // Stack
        dataList.add(localVar);    // Heap
    }
    
    // Статические данные размещаются в Method Area
    private static final String CONSTANT = "constant";
}
```

### Шаг 3: Выполнение байткода

```java
public class BytecodeExecutionExample {
    
    public int calculate(int a, int b) {
        // Байткод: iload_1, iload_2, iadd, ireturn
        return a + b;
    }
    
    public void loopExample() {
        // Байткод: iconst_0, istore_1, goto, iinc, if_icmplt
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```

### Архитектурная схема

```
┌─────────────────────────────────────────────────────────────┐
│                    Java Application                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    JVM Architecture                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │Class Loader │  │Memory Mgmt  │  │Execution    │       │
│  │             │  │             │  │Engine       │       │
│  │• Bootstrap  │  │• Heap       │  │• Interpreter│       │
│  │• Extension  │  │• Stack      │  │• JIT        │       │
│  │• Application│  │• Method Area│  │• GC         │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    Operating System                         │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые преимущества

1. **Платформо-независимость** - один байткод работает на всех платформах
2. **Автоматическое управление памятью** - Garbage Collector освобождает память
3. **Безопасность** - изоляция приложений и контроль доступа к ресурсам
4. **Оптимизация производительности** - JIT компиляция и различные оптимизации
5. **Отладка и профилирование** - встроенные инструменты для анализа

## Практические примеры

### Пример 1: Мониторинг использования памяти

```java
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;

public class MemoryMonitoringExample {
    
    public static void main(String[] args) {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        // Получение информации о heap памяти
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        System.out.println("Heap Memory Usage:");
        System.out.println("  Used: " + formatBytes(heapUsage.getUsed()));
        System.out.println("  Max: " + formatBytes(heapUsage.getMax()));
        System.out.println("  Committed: " + formatBytes(heapUsage.getCommitted()));
        
        // Получение информации о non-heap памяти
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        System.out.println("Non-Heap Memory Usage:");
        System.out.println("  Used: " + formatBytes(nonHeapUsage.getUsed()));
        System.out.println("  Max: " + formatBytes(nonHeapUsage.getMax()));
    }
    
    private static String formatBytes(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.2f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format("%.2f MB", bytes / (1024.0 * 1024.0));
        return String.format("%.2f GB", bytes / (1024.0 * 1024.0 * 1024.0));
    }
}
```

**Объяснение:**
- Используем ManagementFactory для получения информации о памяти
- Мониторим как heap, так и non-heap память
- Форматируем размеры в читаемом виде

### Пример 2: Настройка JVM параметров

```java
public class JVMParametersExample {
    
    public static void main(String[] args) {
        // Вывод текущих JVM параметров
        System.out.println("JVM Parameters:");
        System.out.println("  Max Heap Size: " + 
            Runtime.getRuntime().maxMemory() / (1024 * 1024) + " MB");
        System.out.println("  Total Memory: " + 
            Runtime.getRuntime().totalMemory() / (1024 * 1024) + " MB");
        System.out.println("  Free Memory: " + 
            Runtime.getRuntime().freeMemory() / (1024 * 1024) + " MB");
        
        // Демонстрация работы с большими объектами
        createLargeObjects();
    }
    
    private static void createLargeObjects() {
        List<byte[]> objects = new ArrayList<>();
        
        try {
            // Создаем объекты по 1MB каждый
            for (int i = 0; i < 10; i++) {
                byte[] largeObject = new byte[1024 * 1024]; // 1MB
                objects.add(largeObject);
                System.out.println("Created object " + (i + 1) + 
                    ", Free memory: " + 
                    Runtime.getRuntime().freeMemory() / (1024 * 1024) + " MB");
                
                Thread.sleep(100); // Пауза для наблюдения
            }
        } catch (OutOfMemoryError e) {
            System.err.println("Out of memory: " + e.getMessage());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Объяснение:**
- Демонстрируем мониторинг памяти в реальном времени
- Показываем как JVM управляет памятью при создании больших объектов
- Обрабатываем OutOfMemoryError

### Пример 3: Анализ байткода

```java
import java.io.FileOutputStream;
import java.io.IOException;

public class BytecodeAnalysisExample {
    
    public static void main(String[] args) {
        // Простой метод для анализа байткода
        int result = calculateSum(5, 10);
        System.out.println("Result: " + result);
        
        // Создание класса динамически
        createDynamicClass();
    }
    
    public static int calculateSum(int a, int b) {
        // Этот метод будет скомпилирован в байткод
        int sum = a + b;
        return sum * 2;
    }
    
    private static void createDynamicClass() {
        // Демонстрация работы Class Loader
        try {
            Class<?> clazz = Class.forName("java.lang.String");
            System.out.println("Loaded class: " + clazz.getName());
            System.out.println("Class loader: " + clazz.getClassLoader());
        } catch (ClassNotFoundException e) {
            System.err.println("Class not found: " + e.getMessage());
        }
    }
}
```

**Объяснение:**
- Показываем как методы компилируются в байткод
- Демонстрируем работу Class Loader
- Анализируем загрузку классов

## Performance анализ

### Benchmarking результаты

```java
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
public class JVMPerformanceBenchmark {
    
    @Benchmark
    public void objectCreationBenchmark() {
        // Создание объектов для тестирования производительности
        for (int i = 0; i < 1000; i++) {
            new String("test" + i);
        }
    }
    
    @Benchmark
    public void memoryAllocationBenchmark() {
        // Тестирование выделения памяти
        byte[] buffer = new byte[1024 * 1024]; // 1MB
        for (int i = 0; i < buffer.length; i++) {
            buffer[i] = (byte) i;
        }
    }
    
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JVMPerformanceBenchmark.class.getSimpleName())
                .forks(1)
                .build();
        new Runner(opt).run();
    }
}
```

**Результаты:**
- Object Creation: ~2.5 microseconds per 1000 objects
- Memory Allocation: ~15 microseconds per 1MB
- GC Overhead: ~5% of total execution time

### Memory анализ

```java
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;
import java.util.List;

public class MemoryAnalysisExample {
    
    public static void analyzeMemory() {
        // Анализ Garbage Collector
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("GC Name: " + gcBean.getName());
            System.out.println("  Collection Count: " + gcBean.getCollectionCount());
            System.out.println("  Collection Time: " + gcBean.getCollectionTime() + " ms");
        }
        
        // Анализ использования памяти
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Memory Analysis:");
        System.out.println("  Total Memory: " + formatBytes(totalMemory));
        System.out.println("  Used Memory: " + formatBytes(usedMemory));
        System.out.println("  Free Memory: " + formatBytes(freeMemory));
        System.out.println("  Memory Usage: " + 
            String.format("%.2f%%", (double) usedMemory / totalMemory * 100));
    }
    
    private static String formatBytes(long bytes) {
        return String.format("%.2f MB", bytes / (1024.0 * 1024.0));
    }
}
```

**Memory footprint:**
- Heap usage: 50-80% от доступной памяти
- Non-heap usage: 10-20% от доступной памяти
- GC overhead: 2-5% от времени выполнения

### CPU анализ

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

public class CPUAnalysisExample {
    
    public static void analyzeCPU() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        
        // Анализ CPU времени по потокам
        long[] threadIds = threadBean.getAllThreadIds();
        
        System.out.println("CPU Analysis by Thread:");
        for (long threadId : threadIds) {
            ThreadInfo threadInfo = threadBean.getThreadInfo(threadId);
            if (threadInfo != null) {
                long cpuTime = threadBean.getThreadCpuTime(threadId);
                long userTime = threadBean.getThreadUserTime(threadId);
                
                System.out.println("  Thread: " + threadInfo.getThreadName());
                System.out.println("    CPU Time: " + cpuTime / 1000000 + " ms");
                System.out.println("    User Time: " + userTime / 1000000 + " ms");
                System.out.println("    State: " + threadInfo.getThreadState());
            }
        }
        
        // Общая статистика CPU
        System.out.println("CPU Statistics:");
        System.out.println("  Available Processors: " + 
            Runtime.getRuntime().availableProcessors());
        System.out.println("  Total CPU Time: " + 
            threadBean.getCurrentThreadCpuTime() / 1000000 + " ms");
    }
}
```

**CPU usage:**
- JVM threads: 5-10% CPU в idle состоянии
- Application threads: зависит от нагрузки
- GC threads: 1-3% CPU во время сборки мусора

## Troubleshooting

### Проблема 1: OutOfMemoryError

**Симптомы:**
- Приложение падает с ошибкой "java.lang.OutOfMemoryError: Java heap space"
- Медленная работа приложения
- Частые вызовы Garbage Collector

**Причины:**
- Недостаточно heap памяти
- Утечки памяти
- Неправильная настройка JVM параметров

**Решение:**
```java
// Настройка JVM параметров для увеличения heap
// -Xmx4g -Xms2g -XX:+UseG1GC

public class MemoryLeakFix {
    
    private static final Map<String, Object> cache = new WeakHashMap<>();
    
    public void processData(String key, Object data) {
        // Используем WeakHashMap для автоматической очистки
        cache.put(key, data);
        
        // Периодическая очистка кэша
        if (cache.size() > 1000) {
            cache.clear();
        }
    }
    
    public void monitorMemory() {
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        
        if ((double) usedMemory / maxMemory > 0.8) {
            System.gc(); // Принудительная сборка мусора
        }
    }
}
```

### Проблема 2: ClassNotFoundException

**Симптомы:**
- Ошибка "java.lang.ClassNotFoundException"
- Приложение не запускается
- Отсутствуют зависимости

**Причины:**
- Класс не найден в classpath
- Проблемы с Class Loader
- Отсутствующие JAR файлы

**Решение:**
```java
public class ClassLoadingFix {
    
    public static void loadClassSafely(String className) {
        try {
            // Попытка загрузки класса
            Class<?> clazz = Class.forName(className);
            System.out.println("Class loaded successfully: " + clazz.getName());
            
        } catch (ClassNotFoundException e) {
            System.err.println("Class not found: " + className);
            
            // Попытка загрузки через custom Class Loader
            try {
                ClassLoader customLoader = new URLClassLoader(new URL[0]);
                Class<?> clazz = customLoader.loadClass(className);
                System.out.println("Class loaded via custom loader: " + clazz.getName());
                
            } catch (Exception ex) {
                System.err.println("Failed to load class: " + ex.getMessage());
            }
        }
    }
    
    public static void checkClassPath() {
        String classPath = System.getProperty("java.class.path");
        System.out.println("Classpath: " + classPath);
        
        // Проверка доступности основных классов
        String[] essentialClasses = {
            "java.lang.String",
            "java.util.List",
            "java.io.File"
        };
        
        for (String className : essentialClasses) {
            try {
                Class.forName(className);
                System.out.println("✓ " + className + " is available");
            } catch (ClassNotFoundException e) {
                System.err.println("✗ " + className + " is missing");
            }
        }
    }
}
```

### Проблема 3: StackOverflowError

**Симптомы:**
- Ошибка "java.lang.StackOverflowError"
- Бесконечная рекурсия
- Переполнение стека

**Причины:**
- Глубокая рекурсия без базового случая
- Неправильная настройка размера стека
- Циклические вызовы методов

**Решение:**
```java
public class StackOverflowFix {
    
    // Правильная рекурсия с базовым случаем
    public static int factorial(int n) {
        if (n <= 1) {
            return 1; // Базовый случай
        }
        return n * factorial(n - 1);
    }
    
    // Итеративное решение для избежания переполнения стека
    public static int factorialIterative(int n) {
        int result = 1;
        for (int i = 2; i <= n; i++) {
            result *= i;
        }
        return result;
    }
    
    // Tail recursion optimization
    public static int factorialTailRec(int n) {
        return factorialTailRecHelper(n, 1);
    }
    
    private static int factorialTailRecHelper(int n, int acc) {
        if (n <= 1) {
            return acc;
        }
        return factorialTailRecHelper(n - 1, n * acc);
    }
    
    public static void checkStackSize() {
        // Проверка размера стека
        int stackSize = Thread.currentThread().getStackTrace().length;
        System.out.println("Current stack depth: " + stackSize);
        
        // Рекомендуемый размер стека: 1MB для большинства приложений
        // -Xss1m
    }
}
```

## Best Practices

### ✅ DO

1. **Правильная настройка JVM параметров**
   ```java
   // Рекомендуемые параметры для production
   // -Xmx4g -Xms2g -XX:+UseG1GC -XX:+UseStringDeduplication
   // -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumps
   ```
   - Устанавливайте одинаковые значения для -Xmx и -Xms в production
   - Используйте современные GC алгоритмы (G1GC, ZGC)
   - Включайте heap dump при OOM для анализа

2. **Мониторинг производительности**
   ```java
   public class PerformanceMonitor {
       
       private static final Logger logger = LoggerFactory.getLogger(PerformanceMonitor.class);
       
       public void monitorApplication() {
           Runtime runtime = Runtime.getRuntime();
           long usedMemory = runtime.totalMemory() - runtime.freeMemory();
           long maxMemory = runtime.maxMemory();
           
           double memoryUsage = (double) usedMemory / maxMemory;
           logger.info("Memory usage: {:.2f}%", memoryUsage * 100);
           
           if (memoryUsage > 0.8) {
               logger.warn("High memory usage detected");
           }
       }
   }
   ```
   - Регулярно мониторьте использование памяти
   - Устанавливайте алерты при высоком использовании ресурсов
   - Логируйте метрики производительности

3. **Правильное управление ресурсами**
   ```java
   public class ResourceManager {
       
       public void processWithResources() {
           try (InputStream input = new FileInputStream("data.txt");
                OutputStream output = new FileOutputStream("result.txt")) {
               
               byte[] buffer = new byte[8192];
               int bytesRead;
               
               while ((bytesRead = input.read(buffer)) != -1) {
                   output.write(buffer, 0, bytesRead);
               }
               
           } catch (IOException e) {
               logger.error("Error processing file", e);
           }
       }
   }
   ```
   - Используйте try-with-resources для автоматического закрытия ресурсов
   - Освобождайте ресурсы явно при необходимости
   - Обрабатывайте исключения корректно

### ❌ DON'T

1. **Игнорирование утечек памяти**
   ```java
   // НЕПРАВИЛЬНО - статический кэш без ограничений
   public class MemoryLeakExample {
       
       private static final Map<String, Object> cache = new HashMap<>();
       
       public void addToCache(String key, Object value) {
           cache.put(key, value); // ПРОБЛЕМА: кэш растет бесконечно
       }
   }
   ```
   - Не используйте неограниченные кэши
   - Не забывайте очищать коллекции
   - Не игнорируйте предупреждения о памяти

2. **Неправильная обработка исключений**
   ```java
   // НЕПРАВИЛЬНО - подавление исключений
   public class ExceptionHandlingExample {
       
       public void processData() {
           try {
               // Код, который может выбросить исключение
               riskyOperation();
           } catch (Exception e) {
               // ПРОБЛЕМА: исключение игнорируется
               e.printStackTrace(); // Только для отладки
           }
       }
   }
   ```
   - Не подавляйте исключения без логирования
   - Не используйте printStackTrace() в production
   - Не игнорируйте checked exceptions

3. **Неправильная настройка GC**
   ```java
   // НЕПРАВИЛЬНО - принудительный вызов GC
   public class GCMisuseExample {
       
       public void optimizeMemory() {
           // ПРОБЛЕМА: принудительный вызов GC
           System.gc(); // Неэффективно и может замедлить приложение
           
           // ПРОБЛЕМА: неправильная настройка памяти
           Runtime.getRuntime().freeMemory(); // Не влияет на производительность
       }
   }
   ```
   - Не вызывайте System.gc() вручную
   - Не полагайтесь на freeMemory() для оптимизации
   - Не игнорируйте GC логи

## Инструменты и утилиты

### Инструмент 1: JVisualVM

**Назначение:** Графический интерфейс для мониторинга JVM

**Использование:**
```bash
# Запуск JVisualVM
jvisualvm

# Подключение к удаленному процессу
jvisualvm --openpid <PID>
```

**Пример вывода:**
```
JVisualVM - Java VisualVM
Version: 1.8.0_301
Java Version: 1.8.0_301-b09
```

### Инструмент 2: JConsole

**Назначение:** Мониторинг и управление JVM через JMX

**Использование:**
```bash
# Запуск JConsole
jconsole

# Подключение к процессу
jconsole <PID>
```

**Пример вывода:**
```
JConsole - Java Monitoring and Management Console
Connected to: <PID>@<hostname>
```

### Инструмент 3: jstack

**Назначение:** Анализ стеков потоков

**Использование:**
```bash
# Получение thread dump
jstack <PID>

# Получение thread dump с дополнительной информацией
jstack -l <PID>
```

**Пример вывода:**
```
"main" #1 prio=5 os_prio=0 tid=0x00007f8c0c001000 nid=0x1234 runnable [0x00007f8c0e123000]
   java.lang.Thread.State: RUNNABLE
        at java.io.FileInputStream.read0(Native Method)
        at java.io.FileInputStream.read(FileInputStream.java:243)
        at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
```

### Инструмент 4: jmap

**Назначение:** Анализ памяти и создание heap dump

**Использование:**
```bash
# Создание heap dump
jmap -dump:format=b,file=heap.hprof <PID>

# Анализ использования памяти
jmap -histo <PID>
```

**Пример вывода:**
```
num     #instances         #bytes  class name
----------------------------------------------
   1:        100000      20000000  java.lang.String
   2:         50000      10000000  java.util.HashMap$Node
   3:         25000       5000000  java.lang.Object[]
```

### Инструмент 5: jstat

**Назначение:** Мониторинг статистики JVM

**Использование:**
```bash
# Мониторинг GC статистики
jstat -gc <PID> 1000

# Мониторинг классов
jstat -class <PID> 1000
```

**Пример вывода:**
```
S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
1024.0 1024.0  0.0   1024.0  8192.0   8192.0   20480.0    2048.0   4480.0 4480.0  512.0  512.0      1    0.010   0      0.000    0.010
```

## Дополнительные ресурсы

### Документация
- [Oracle JVM Documentation](https://docs.oracle.com/javase/specs/jvms/se17/html/)
- [OpenJDK Documentation](https://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html)
- [JVM Specification](https://docs.oracle.com/javase/specs/jvms/se17/html/)

### Книги
- "Inside the Java Virtual Machine" - Bill Venners
- "Java Performance: The Definitive Guide" - Scott Oaks
- "Optimizing Java: Practical Techniques for Improving JVM Application Performance" - Benjamin J. Evans

### Статьи и блоги
- [JVM Internals](https://blog.jamesdbloom.com/JVMInternals.html) - James D Bloom
- [Understanding JVM Memory Model](https://www.baeldung.com/jvm-memory) - Baeldung
- [JVM Garbage Collection](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) - Oracle

### Видео
- [JVM Architecture Deep Dive](https://www.youtube.com/watch?v=ZBJ0u9KaK0M) - Oracle
- [Understanding JVM Memory Management](https://www.youtube.com/watch?v=UwB0OSmkOtQ) - Java Brains
- [JVM Performance Tuning](https://www.youtube.com/watch?v=1_4Ok0MVdEg) - Oracle

### Инструменты
- [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html) - Профессиональный профилировщик Java
- [YourKit](https://www.yourkit.com/) - Java Profiler
- [MAT (Memory Analyzer Tool)](https://www.eclipse.org/mat/) - Анализ heap dump
- [JFR (Java Flight Recorder)](https://docs.oracle.com/javacomponents/jdk-11-jfr/) - Встроенный профилировщик
- [JMH (Java Microbenchmark Harness)](https://openjdk.java.net/projects/code-tools/jmh/) - Фреймворк для бенчмаркинга 