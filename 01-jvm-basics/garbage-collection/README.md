# Garbage Collection

## Обзор темы

**Garbage Collection (GC)** - это автоматический процесс управления памятью в JVM, который освобождает память от неиспользуемых объектов без участия разработчика.

### Что это такое?

Garbage Collection - это один из ключевых механизмов Java, который автоматически управляет памятью в куче (heap). В отличие от языков программирования, где разработчик должен вручную освобождать память (например, C++), Java предоставляет автоматическое управление памятью через GC.

GC работает в фоновом режиме, отслеживая объекты в куче и освобождая память от тех объектов, которые больше не доступны из корневых ссылок. Это позволяет разработчикам сосредоточиться на бизнес-логике, не беспокоясь о ручном управлении памятью.

### Проблема, которую решает тема

В Java приложениях часто возникает ситуация, когда объекты создаются в больших количествах, но не все из них используются постоянно. Например:

1. **Сценарий**: Приложение обрабатывает большие объемы данных, создавая множество временных объектов
2. **Требование**: Необходимо эффективно управлять памятью, освобождая место от неиспользуемых объектов
3. **Проблема**: Без автоматического управления памятью приложение быстро исчерпает доступную память, что приведет к OutOfMemoryError
4. **Результат**: Garbage Collection автоматически освобождает память, позволяя приложению работать стабильно

## Детальный анализ проблемы

### Сценарий 1: Отсутствие автоматического управления памятью

```java
public class MemoryLeakExample {
    
    // ПРОБЛЕМА ЗДЕСЬ!
    private static List<String> cache = new ArrayList<>();
    
    public void processData(String data) {
        // 1. Обработка данных
        
        // 2. Добавление в кэш без ограничений (ПРОБЛЕМА ЗДЕСЬ!)
        cache.add(data);
        
        // 3. Память растет бесконечно
    }
}
```

**Проблемы:**
- Неограниченный рост потребления памяти
- Отсутствие освобождения неиспользуемых объектов
- Риск OutOfMemoryError при длительной работе
- Неэффективное использование ресурсов

### Корень проблемы

Основная проблема заключается в том, что без автоматического управления памятью разработчики должны вручную отслеживать и освобождать память. Это приводит к:

1. **Memory Leaks** - утечки памяти из-за забытых ссылок
2. **OutOfMemoryError** - исчерпание доступной памяти
3. **Сложность разработки** - необходимость ручного управления памятью
4. **Нестабильность приложений** - непредсказуемое поведение при нехватке памяти

## Принцип работы

### Как работает Garbage Collection

1. **Шаг 1**: Маркировка - GC определяет, какие объекты все еще используются
2. **Шаг 2**: Очистка - GC освобождает память от неиспользуемых объектов
3. **Шаг 3**: Компактификация - GC уплотняет память, устраняя фрагментацию
4. **Шаг 4**: Обновление ссылок - GC обновляет ссылки на перемещенные объекты

### Архитектурная схема

```
┌─────────────────────────────────────────────────────────────┐
│                    Garbage Collection                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │   Marking   │ -> │   Sweeping  │ -> │ Compaction  │    │
│  │    Phase    │    │    Phase    │    │    Phase    │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    Heap Memory                         │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │ │
│  │  │ Live    │ │ Dead    │ │ Live    │ │ Dead    │    │ │
│  │  │ Object  │ │ Object  │ │ Object  │ │ Object  │    │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘    │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Детальный принцип работы GC

### Фазы Garbage Collection

#### 1. Mark Phase (Фаза маркировки)

```java
public class MarkPhaseDemo {
    
    // Static переменная - это корневая ссылка
    private static Object staticRoot = new Object();
    
    public static void demonstrateMarking() {
        // Корневые ссылки (Root References)
        Object localRoot = new Object(); // Локальная переменная в активном методе
        
        // Создаем граф объектов
        Object obj1 = new Object();
        Object obj2 = new Object();
        Object obj3 = new Object();
        Object obj4 = new Object(); // Этот объект не связан с корнями
        
        // Связываем объекты
        localRoot = obj1;  // obj1 связан с корнем
        obj1 = obj2;       // obj2 связан с obj1
        obj2 = obj3;       // obj3 связан с obj2
        
        // obj4 НЕ связан с корневыми ссылками - будет помечен как garbage
        // obj1, obj2, obj3 связаны через цепочку: localRoot -> obj1 -> obj2 -> obj3
    }
}
```

**Процесс маркировки:**
1. **Начало с корневых ссылок** - static поля, локальные переменные активных методов
2. **Рекурсивный обход** - все объекты, достижимые из корней, помечаются как "живые"
3. **Использование битов** - каждый объект имеет mark bit для отслеживания состояния

**Корневые ссылки (Root References):**
- **Static поля** - переменные класса, объявленные как `static`
- **Локальные переменные** - переменные в активных методах (в стеке)
- **Параметры методов** - аргументы активных методов
- **JNI глобальные ссылки** - ссылки из нативного кода
- **Мониторы** - объекты, используемые в synchronized блоках

#### 2. Sweep Phase (Фаза очистки)

```java
public class SweepPhaseDemo {
    
    public static void demonstrateSweeping() {
        // После маркировки все непомеченные объекты считаются garbage
        List<Object> heap = new ArrayList<>();
        
        // Симулируем кучу
        heap.add(new Object()); // Live object
        heap.add(new Object()); // Dead object (не связан с корнями)
        heap.add(new Object()); // Live object
        
        // Sweep phase - освобождаем память от dead объектов
        heap.removeIf(obj -> !isMarked(obj));
    }
    
    private static boolean isMarked(Object obj) {
        // Проверка mark bit
        return obj != null && obj.hashCode() % 2 == 0; // Упрощенная логика
    }
}
```

#### Подробный пример маркировки

```java
public class DetailedMarkingExample {
    
    // Static поле - корневая ссылка
    private static Object staticObject = new Object();
    
    public static void demonstrateDetailedMarking() {
        // Локальная переменная - корневая ссылка
        Object localObject = new Object();
        
        // Создаем граф объектов
        Object objA = new Object();
        Object objB = new Object();
        Object objC = new Object();
        Object objD = new Object();
        Object objE = new Object();
        
        // Связываем объекты
        localObject = objA;        // objA связан с корнем
        objA = objB;              // objB связан с objA
        objB = objC;              // objC связан с objB
        
        staticObject = objD;       // objD связан с static корнем
        
        // objE НЕ связан ни с чем - будет garbage
        
        // Результат маркировки:
        // ЖИВЫЕ объекты: staticObject, localObject, objA, objB, objC, objD
        // МЕРТВЫЕ объекты: objE
    }
}
```

#### 3. Compact Phase (Фаза компактификации)

```java
public class CompactPhaseDemo {
    
    public static void demonstrateCompaction() {
        // До компактификации: фрагментированная память
        Object[] heap = {
            new Object(), // Live
            null,         // Freed
            new Object(), // Live
            null,         // Freed
            new Object()  // Live
        };
        
        // После компактификации: уплотненная память
        Object[] compactedHeap = {
            new Object(), // Live
            new Object(), // Live
            new Object(), // Live
            null,         // Free space
            null          // Free space
        };
    }
}
```

### Поколения в Heap Memory

```java
public class GenerationsDemo {
    
    public static void demonstrateGenerations() {
        // Young Generation (Eden + Survivor spaces)
        Object youngObject = new Object(); // Создается в Eden
        
        // После первой сборки мусора
        if (youngObject != null) {
            // Объект перемещается в Survivor space
            youngObject = promoteToSurvivor(youngObject);
        }
        
        // После нескольких сборок мусора
        if (getAge(youngObject) > TENURE_THRESHOLD) {
            // Объект перемещается в Old Generation
            youngObject = promoteToOldGen(youngObject);
        }
    }
    
    private static final int TENURE_THRESHOLD = 15;
    
    private static Object promoteToSurvivor(Object obj) {
        // Логика перемещения в Survivor
        return obj;
    }
    
    private static Object promoteToOldGen(Object obj) {
        // Логика перемещения в Old Generation
        return obj;
    }
    
    private static int getAge(Object obj) {
        // Получение возраста объекта
        return obj.hashCode() % 20; // Упрощенная логика
    }
}
```

### Типы сборок мусора

#### 1. Minor GC (Young Generation)

```java
public class MinorGCDemo {
    
    public static void demonstrateMinorGC() {
        // Minor GC собирает только Young Generation
        List<Object> youngObjects = new ArrayList<>();
        
        // Создаем много молодых объектов
        for (int i = 0; i < 100000; i++) {
            youngObjects.add(new byte[1024]); // 1KB объекты
        }
        
        // Когда Eden заполняется, запускается Minor GC
        // Выжившие объекты перемещаются в Survivor
        // Мертвые объекты освобождаются
    }
}
```

#### 2. Major GC (Old Generation)

```java
public class MajorGCDemo {
    
    public static void demonstrateMajorGC() {
        // Major GC собирает Old Generation
        List<Object> oldObjects = new ArrayList<>();
        
        // Объекты, пережившие несколько Minor GC
        for (int i = 0; i < 10000; i++) {
            Object obj = new Object();
            // Симулируем объекты, которые пережили Minor GC
            if (isLongLived(obj)) {
                oldObjects.add(obj);
            }
        }
        
        // Когда Old Generation заполняется, запускается Major GC
        // Это более дорогая операция
    }
    
    private static boolean isLongLived(Object obj) {
        return obj.hashCode() % 10 == 0; // 10% объектов долгоживущие
    }
}
```

#### 3. Full GC (Вся куча)

```java
public class FullGCDemo {
    
    public static void demonstrateFullGC() {
        // Full GC собирает всю кучу (Young + Old)
        // Это самая дорогая операция
        
        // Ситуации, когда запускается Full GC:
        // 1. Нехватка памяти в Old Generation
        // 2. Явный вызов System.gc() (не рекомендуется)
        // 3. Нехватка памяти в Metaspace
        
        System.out.println("Full GC - самая дорогая операция");
        System.out.println("Старайтесь избегать её в production");
    }
}
```

### Алгоритмы сборки мусора

#### Mark-Sweep

```java
public class MarkSweepDemo {
    
    public static void demonstrateMarkSweep() {
        // Простой алгоритм: Mark -> Sweep
        Set<Object> reachable = new HashSet<>();
        
        // 1. Mark phase
        markReachableObjects(reachable);
        
        // 2. Sweep phase
        sweepUnreachableObjects(reachable);
    }
    
    private static void markReachableObjects(Set<Object> reachable) {
        // Начинаем с корневых ссылок
        Object root = new Object();
        reachable.add(root);
        
        // Рекурсивно помечаем все достижимые объекты
        markObject(root, reachable);
    }
    
    private static void markObject(Object obj, Set<Object> reachable) {
        if (obj == null || reachable.contains(obj)) {
            return;
        }
        
        reachable.add(obj);
        
        // Рекурсивно помечаем связанные объекты
        // (в реальности это делается через reflection или специальные структуры)
    }
    
    private static void sweepUnreachableObjects(Set<Object> reachable) {
        // Освобождаем память от недостижимых объектов
        System.out.println("Освобождаем память от " + 
            (getTotalObjects() - reachable.size()) + " объектов");
    }
    
    private static int getTotalObjects() {
        return 1000; // Упрощенная логика
    }
}
```

#### Mark-Compact

```java
public class MarkCompactDemo {
    
    public static void demonstrateMarkCompact() {
        // Алгоритм: Mark -> Compact
        List<Object> heap = new ArrayList<>();
        
        // 1. Mark phase (как в Mark-Sweep)
        Set<Object> reachable = new HashSet<>();
        markReachableObjects(reachable);
        
        // 2. Compact phase - перемещаем живые объекты
        compactLiveObjects(heap, reachable);
    }
    
    private static void compactLiveObjects(List<Object> heap, Set<Object> reachable) {
        // Перемещаем живые объекты в начало кучи
        List<Object> compactedHeap = new ArrayList<>();
        
        for (Object obj : heap) {
            if (reachable.contains(obj)) {
                compactedHeap.add(obj);
            }
        }
        
        // Освобождаем оставшуюся память
        System.out.println("Куча уплотнена: " + 
            compactedHeap.size() + " живых объектов");
    }
}
```

#### Copying

```java
public class CopyingDemo {
    
    public static void demonstrateCopying() {
        // Алгоритм: копирование живых объектов в новое пространство
        Object[] fromSpace = new Object[1000];
        Object[] toSpace = new Object[1000];
        
        // Заполняем fromSpace
        for (int i = 0; i < 100; i++) {
            fromSpace[i] = new Object();
        }
        
        // Копируем живые объекты в toSpace
        int toIndex = 0;
        for (int i = 0; i < fromSpace.length; i++) {
            if (fromSpace[i] != null && isLive(fromSpace[i])) {
                toSpace[toIndex++] = fromSpace[i];
                fromSpace[i] = null; // Освобождаем fromSpace
            }
        }
        
        // Меняем роли пространств
        System.out.println("Скопировано " + toIndex + " живых объектов");
    }
    
    private static boolean isLive(Object obj) {
        return obj.hashCode() % 3 != 0; // 2/3 объектов живые
    }
}
```

### Ключевые преимущества

1. **Автоматическое управление памятью** - разработчики не должны вручную освобождать память
2. **Предотвращение утечек памяти** - GC автоматически находит и освобождает неиспользуемые объекты
3. **Упрощение разработки** - можно сосредоточиться на бизнес-логике
4. **Стабильность приложений** - предотвращение OutOfMemoryError
5. **Оптимизация производительности** - различные алгоритмы GC для разных сценариев

## Типы Garbage Collectors

### Обзор алгоритмов GC

В Java существует несколько различных алгоритмов Garbage Collection, каждый из которых оптимизирован для определенных сценариев использования:

1. **Serial GC** - однопоточный, для небольших приложений
2. **Parallel GC** - многопоточный, для high-throughput приложений
3. **CMS (Concurrent Mark Sweep)** - low-latency, но deprecated
4. **G1GC (Garbage First)** - современный, универсальный
5. **ZGC (Z Garbage Collector)** - ultra-low-latency
6. **Shenandoah** - low-latency, альтернатива ZGC

### 1. Serial Garbage Collector

**Принцип работы:**
```java
// Serial GC работает в одном потоке
public class SerialGCExample {
    
    public static void main(String[] args) {
        // Запуск с Serial GC
        // java -XX:+UseSerialGC
        
        // Создаем объекты для демонстрации
        for (int i = 0; i < 100000; i++) {
            new LargeObject();
        }
    }
}
```

**Характеристики:**
- **Тип**: Stop-the-world, однопоточный
- **Память**: Young + Old generation
- **Алгоритм**: Mark-Sweep-Compact
- **Паузы**: Длительные, но редкие

**Плюсы:**
- Простота реализации
- Низкое потребление CPU
- Подходит для небольших приложений

**Минусы:**
- Длительные паузы
- Не подходит для больших куч
- Плохая производительность в многопоточных приложениях

**Когда использовать:**
- Небольшие приложения (< 100MB heap)
- Однопоточные приложения
- Системы с ограниченными ресурсами

### 2. Parallel Garbage Collector

**Принцип работы:**
```java
// Parallel GC использует несколько потоков
public class ParallelGCExample {
    
    public static void main(String[] args) {
        // Запуск с Parallel GC
        // java -XX:+UseParallelGC -XX:ParallelGCThreads=4
        
        // Создаем нагрузку для демонстрации
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                for (int j = 0; j < 100000; j++) {
                    new byte[1024]; // Создаем временные объекты
                }
            });
        }
        
        executor.shutdown();
    }
}
```

**Характеристики:**
- **Тип**: Stop-the-world, многопоточный
- **Память**: Young + Old generation
- **Алгоритм**: Parallel Mark-Sweep-Compact
- **Паузы**: Средние, но быстрее Serial GC

**Плюсы:**
- Высокая пропускная способность
- Эффективное использование CPU
- Хорошая производительность для batch-приложений

**Минусы:**
- Паузы все еще заметны
- Не подходит для low-latency приложений
- Может блокировать все потоки

**Когда использовать:**
- Batch-приложения
- High-throughput системы
- Серверные приложения с большими кучами

### 3. CMS (Concurrent Mark Sweep) - DEPRECATED

**Принцип работы:**
```java
// CMS GC (устарел, но для понимания)
public class CMSExample {
    
    public static void main(String[] args) {
        // Запуск с CMS (больше не рекомендуется)
        // java -XX:+UseConcMarkSweepGC
        
        // CMS работал параллельно с приложением
        // но имел проблемы с фрагментацией
    }
}
```

**Характеристики:**
- **Тип**: Concurrent, low-latency
- **Память**: Young + Old generation
- **Алгоритм**: Concurrent Mark-Sweep
- **Паузы**: Короткие, но частые

**Плюсы:**
- Короткие паузы
- Работа параллельно с приложением
- Хорошо подходил для low-latency приложений

**Минусы:**
- Фрагментация памяти
- Непредсказуемая производительность
- Устарел в Java 9, удален в Java 14

### 4. G1GC (Garbage First) - РЕКОМЕНДУЕМЫЙ

**Принцип работы:**
```java
// G1GC - современный и универсальный GC
public class G1GCExample {
    
    public static void main(String[] args) {
        // Запуск с G1GC (рекомендуется)
        // java -XX:+UseG1GC -XX:MaxGCPauseMillis=200
        
        // G1GC разделяет кучу на регионы
        // и собирает мусор в регионах с наибольшим количеством мусора
        
        // Создаем нагрузку для демонстрации
        createLoad();
    }
    
    private static void createLoad() {
        List<byte[]> objects = new ArrayList<>();
        
        // Создаем объекты разного размера
        for (int i = 0; i < 1000; i++) {
            if (i % 100 == 0) {
                // Большие объекты
                objects.add(new byte[1024 * 1024]); // 1MB
            } else {
                // Малые объекты
                objects.add(new byte[1024]); // 1KB
            }
        }
    }
}
```

**Характеристики:**
- **Тип**: Concurrent, low-latency
- **Память**: Region-based heap
- **Алгоритм**: Garbage First
- **Паузы**: Предсказуемые, настраиваемые

**Плюсы:**
- Предсказуемые паузы
- Эффективная работа с большими кучами
- Автоматическая оптимизация
- Поддержка больших объектов

**Минусы:**
- Более сложная настройка
- Может потреблять больше CPU
- Не всегда оптимален для очень маленьких куч

**Когда использовать:**
- **РЕКОМЕНДУЕТСЯ для большинства приложений**
- Серверные приложения
- Приложения с большими кучами (> 4GB)
- Low-latency приложения

**Настройки G1GC:**
```bash
# Базовые настройки
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=16m \
     -XX:G1NewSizePercent=30 \
     -XX:G1MaxNewSizePercent=40 \
     -XX:G1MixedGCLiveThresholdPercent=85 \
     -XX:G1MixedGCCountTarget=8 \
     -XX:G1OldCSetRegionThresholdPercent=10 \
     -jar application.jar
```

### 5. ZGC (Z Garbage Collector) - УЛЬТРА-СОВРЕМЕННЫЙ

**Принцип работы:**
```java
// ZGC - ультра-low-latency GC
public class ZGCExample {
    
    public static void main(String[] args) {
        // Запуск с ZGC (Java 11+)
        // java -XX:+UseZGC -Xmx16g
        
        // ZGC использует colored pointers и load barriers
        // для достижения ultra-low-latency
        
        // Создаем нагрузку для демонстрации
        createIntensiveLoad();
    }
    
    private static void createIntensiveLoad() {
        // Создаем множество объектов для демонстрации ZGC
        List<Object> objects = new ArrayList<>();
        
        for (int i = 0; i < 1000000; i++) {
            objects.add(new byte[1024]);
            
            if (i % 100000 == 0) {
                System.out.println("Создано объектов: " + i);
            }
        }
    }
}
```

**Характеристики:**
- **Тип**: Concurrent, ultra-low-latency
- **Память**: Region-based heap
- **Алгоритм**: Concurrent Mark-Compact
- **Паузы**: < 1ms (обычно < 10ms)

**Плюсы:**
- Ультра-короткие паузы
- Масштабируемость до терабайтных куч
- Полностью concurrent
- Предсказуемая производительность

**Минусы:**
- Требует Java 11+
- Может потреблять больше CPU
- Не подходит для очень маленьких куч
- Более сложная настройка

**Когда использовать:**
- Критически важные low-latency приложения
- Приложения с большими кучами (> 8GB)
- Системы реального времени
- Торговые платформы, игровые серверы

**Настройки ZGC:**
```bash
# Настройки для production
java -XX:+UseZGC \
     -Xmx16g \
     -Xms16g \
     -XX:+UnlockExperimentalVMOptions \
     -XX:+UseZGC \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogVMOutput \
     -XX:LogFile=zgc.log \
     -jar application.jar
```

### 6. Shenandoah GC - АЛЬТЕРНАТИВА ZGC

**Принцип работы:**
```java
// Shenandoah - альтернатива ZGC
public class ShenandoahExample {
    
    public static void main(String[] args) {
        // Запуск с Shenandoah
        // java -XX:+UseShenandoahGC
        
        // Shenandoah также обеспечивает ultra-low-latency
        // но с другим подходом к реализации
        
        createLoad();
    }
}
```

**Характеристики:**
- **Тип**: Concurrent, low-latency
- **Память**: Region-based heap
- **Алгоритм**: Concurrent Mark-Compact
- **Паузы**: < 10ms

**Плюсы:**
- Короткие паузы
- Хорошая производительность
- Альтернатива ZGC
- Поддержка в OpenJDK

**Минусы:**
- Менее зрелый чем ZGC
- Может потреблять больше CPU
- Ограниченная поддержка

### Сравнение GC алгоритмов

| GC Алгоритм | Паузы | Throughput | Память | Сложность | Рекомендация |
|-------------|-------|------------|--------|-----------|--------------|
| Serial GC | Длинные | Низкий | Низкое | Простая | Только для маленьких приложений |
| Parallel GC | Средние | Высокий | Среднее | Простая | Batch приложения |
| CMS | Короткие | Средний | Высокое | Средняя | **DEPRECATED** |
| G1GC | Короткие | Высокий | Среднее | Средняя | **РЕКОМЕНДУЕТСЯ** |
| ZGC | Ультра-короткие | Высокий | Высокое | Сложная | Low-latency приложения |
| Shenandoah | Короткие | Высокий | Высокое | Сложная | Альтернатива ZGC |

### Какой GC актуален сейчас?

**Для большинства приложений (2024):**

1. **G1GC** - **ОСНОВНОЙ РЕКОМЕНДУЕМЫЙ** GC
   - По умолчанию в Java 9+
   - Хороший баланс между latency и throughput
   - Подходит для большинства приложений

2. **ZGC** - для критически важных low-latency приложений
   - Требует Java 11+
   - Ультра-короткие паузы
   - Для больших куч (> 8GB)

3. **Parallel GC** - для batch-приложений
   - Высокий throughput
   - Подходит для обработки данных

**Выбор GC по сценарию:**

```java
// Для web-приложений
// java -XX:+UseG1GC -XX:MaxGCPauseMillis=200

// Для batch-обработки
// java -XX:+UseParallelGC

// Для low-latency систем
// java -XX:+UseZGC -Xmx16g

// Для маленьких приложений
// java -XX:+UseSerialGC
```

## Мониторинг и настройка GC

### Мониторинг работы GC

#### 1. Включение GC логов

```bash
# Базовые GC логи
java -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log

# Подробные логи с метриками
java -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps \
     -XX:+PrintGCApplicationStoppedTime \
     -XX:+PrintGCApplicationConcurrentTime \
     -Xloggc:gc.log

# Логи для G1GC
java -XX:+UseG1GC \
     -XX:+PrintGC \
     -XX:+PrintGCDetails \
     -XX:+PrintGCTimeStamps \
     -XX:+PrintGCApplicationStoppedTime \
     -Xloggc:g1gc.log

# Логи для ZGC
java -XX:+UseZGC \
     -Xloggc:zgc.log \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogVMOutput
```

#### 2. Анализ GC логов

```java
public class GCLogAnalyzer {
    
    public static void analyzeGCLog(String logFile) {
        try (BufferedReader reader = new BufferedReader(new FileReader(logFile))) {
            String line;
            int minorGCCount = 0;
            int majorGCCount = 0;
            long totalGCTime = 0;
            
            while ((line = reader.readLine()) != null) {
                if (line.contains("[GC")) {
                    minorGCCount++;
                    // Парсим время GC
                    long gcTime = parseGCTime(line);
                    totalGCTime += gcTime;
                } else if (line.contains("[Full GC")) {
                    majorGCCount++;
                    long gcTime = parseGCTime(line);
                    totalGCTime += gcTime;
                }
            }
            
            System.out.println("Minor GC count: " + minorGCCount);
            System.out.println("Major GC count: " + majorGCCount);
            System.out.println("Total GC time: " + totalGCTime + "ms");
            System.out.println("Average GC time: " + 
                (totalGCTime / (minorGCCount + majorGCCount)) + "ms");
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private static long parseGCTime(String logLine) {
        // Упрощенный парсинг времени GC
        if (logLine.contains("secs]")) {
            String timeStr = logLine.split("secs]")[0].split(" ")[1];
            return (long) (Double.parseDouble(timeStr) * 1000);
        }
        return 0;
    }
}
```

#### 3. Программный мониторинг GC

```java
public class GCMonitor {
    
    public static void startMonitoring() {
        // Получаем MXBean для GC
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("GC Name: " + gcBean.getName());
            System.out.println("GC Collections: " + gcBean.getCollectionCount());
            System.out.println("GC Time: " + gcBean.getCollectionTime() + "ms");
        }
        
        // Мониторинг памяти
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        
        System.out.println("Heap Used: " + 
            (heapUsage.getUsed() / 1024 / 1024) + "MB");
        System.out.println("Heap Max: " + 
            (heapUsage.getMax() / 1024 / 1024) + "MB");
    }
}
```

### Настройка GC параметров

#### 1. Размер кучи

```bash
# Базовые настройки размера кучи
java -Xms2g -Xmx4g -jar application.jar

# Для production серверов
java -Xms4g -Xmx8g -jar application.jar

# Для больших приложений
java -Xms8g -Xmx16g -jar application.jar
```

#### 2. Настройки G1GC

```bash
# Оптимизированные настройки G1GC
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=16m \
     -XX:G1NewSizePercent=30 \
     -XX:G1MaxNewSizePercent=40 \
     -XX:G1MixedGCLiveThresholdPercent=85 \
     -XX:G1MixedGCCountTarget=8 \
     -XX:G1OldCSetRegionThresholdPercent=10 \
     -XX:+G1PrintRegionLivenessInfo \
     -jar application.jar
```

#### 3. Настройки ZGC

```bash
# Production настройки ZGC
java -XX:+UseZGC \
     -Xmx16g \
     -Xms16g \
     -XX:+UnlockExperimentalVMOptions \
     -XX:+UseZGC \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogVMOutput \
     -XX:LogFile=zgc.log \
     -jar application.jar
```

### Оптимизация GC

#### 1. Уменьшение пауз GC

```java
public class GCPauseOptimization {
    
    public static void optimizeForLowLatency() {
        // 1. Используем G1GC или ZGC
        // java -XX:+UseG1GC -XX:MaxGCPauseMillis=200
        
        // 2. Избегаем создания больших объектов
        List<byte[]> objects = new ArrayList<>();
        
        // Плохо - создаем большие объекты
        // objects.add(new byte[1024 * 1024 * 100]); // 100MB
        
        // Хорошо - создаем объекты по частям
        for (int i = 0; i < 100; i++) {
            objects.add(new byte[1024 * 1024]); // 1MB каждый
        }
        
        // 3. Используем object pooling для дорогих объектов
        ObjectPool<ExpensiveObject> pool = new ObjectPool<>();
        
        ExpensiveObject obj = pool.borrow();
        try {
            obj.process();
        } finally {
            pool.release(obj);
        }
    }
}

class ExpensiveObject {
    private byte[] data = new byte[1024 * 1024]; // 1MB
    
    public void process() {
        // Обработка данных
    }
}

class ObjectPool<T> {
    private final Queue<T> pool = new ConcurrentLinkedQueue<>();
    
    public T borrow() {
        T obj = pool.poll();
        return obj != null ? obj : createNew();
    }
    
    public void release(T obj) {
        pool.offer(obj);
    }
    
    private T createNew() {
        return (T) new ExpensiveObject();
    }
}
```

#### 2. Оптимизация throughput

```java
public class ThroughputOptimization {
    
    public static void optimizeForThroughput() {
        // 1. Используем Parallel GC для batch-приложений
        // java -XX:+UseParallelGC
        
        // 2. Увеличиваем размер кучи
        // java -Xmx8g -Xms8g
        
        // 3. Оптимизируем создание объектов
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10000; i++) {
            sb.append("item").append(i);
        }
        String result = sb.toString();
        
        // 4. Используем примитивы вместо объектов где возможно
        int[] numbers = new int[1000000]; // Вместо Integer[]
        
        // 5. Избегаем autoboxing
        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i; // Вместо Integer.valueOf(i)
        }
    }
}
```

### Инструменты для анализа GC

#### 1. JVisualVM

```bash
# Запуск JVisualVM
jvisualvm

# Подключение к процессу
jvisualvm --openpid <PID>
```

#### 2. JConsole

```bash
# Запуск JConsole
jconsole

# Подключение к удаленному процессу
jconsole hostname:port
```

#### 3. GCViewer

```bash
# Анализ GC логов
java -jar gcviewer.jar gc.log
```

#### 4. JMH для бенчмаркинга

```java
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public void benchmarkWithGC() {
    // Создаем нагрузку для тестирования GC
    List<String> objects = new ArrayList<>();
    
    for (int i = 0; i < 10000; i++) {
        objects.add("Object " + i);
    }
    
    // Симулируем работу с объектами
    objects.clear();
}
```

## Практические примеры

### Пример 1: Демонстрация работы GC

```java
public class GCDemo {
    
    public static void main(String[] args) {
        // Создаем много объектов
        for (int i = 0; i < 1000000; i++) {
            new LargeObject();
        }
        
        // Принудительно вызываем GC (не рекомендуется в production)
        System.gc();
        
        // Проверяем свободную память
        Runtime runtime = Runtime.getRuntime();
        long freeMemory = runtime.freeMemory();
        long totalMemory = runtime.totalMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Используемая память: " + usedMemory / 1024 / 1024 + " MB");
        System.out.println("Свободная память: " + freeMemory / 1024 / 1024 + " MB");
    }
}

class LargeObject {
    private byte[] data = new byte[1024]; // 1KB
}
```

**Объяснение:**
- Создаем множество объектов для демонстрации работы GC
- Используем System.gc() для принудительного вызова сборщика мусора
- Показываем статистику использования памяти

### Пример 2: Настройка GC параметров

```java
public class GCConfigurationDemo {
    
    public static void main(String[] args) {
        // Показываем текущие настройки GC
        System.out.println("GC Algorithm: " + 
            System.getProperty("java.vm.name"));
        
        // Демонстрируем работу с большими объектами
        createLargeObjects();
    }
    
    private static void createLargeObjects() {
        List<byte[]> objects = new ArrayList<>();
        
        try {
            while (true) {
                // Создаем объекты по 1MB
                objects.add(new byte[1024 * 1024]);
                
                if (objects.size() % 100 == 0) {
                    System.out.println("Создано объектов: " + objects.size());
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError после создания " + 
                objects.size() + " объектов");
        }
    }
}
```

## Performance анализ

### Benchmarking результаты

```java
@Benchmark
public void gcBenchmark() {
    // Создаем объекты для тестирования GC
    List<String> objects = new ArrayList<>();
    
    for (int i = 0; i < 10000; i++) {
        objects.add("Object " + i);
    }
    
    // Симулируем работу с объектами
    objects.clear();
}
```

**Результаты:**
- Время выполнения: ~2-5ms на 10,000 объектов
- Частота GC: зависит от размера кучи и алгоритма
- Паузы GC: от 1ms до нескольких секунд в зависимости от алгоритма

## Troubleshooting

### Проблема 1: OutOfMemoryError

**Симптомы:**
- Исключение OutOfMemoryError
- Медленная работа приложения
- Высокое потребление памяти

**Причины:**
- Утечки памяти
- Недостаточный размер кучи
- Неэффективные алгоритмы GC
- Создание слишком больших объектов

**Решение:**
```java
// Увеличиваем размер кучи
// -Xmx4g -Xms2g

// Используем правильные коллекции
public class MemoryEfficientExample {
    
    private final Map<String, String> cache = new WeakHashMap<>();
    
    public void addToCache(String key, String value) {
        cache.put(key, value);
    }
    
    // Используем try-with-resources для автоматического освобождения ресурсов
    public void processFile(String filename) {
        try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
            String line;
            while ((line = reader.readLine()) != null) {
                processLine(line);
            }
        } catch (IOException e) {
            // Обработка ошибок
        }
    }
}
```

## Best Practices

### ✅ DO

1. **Используйте подходящий алгоритм GC**
   ```java
   // Для low-latency приложений
   // -XX:+UseG1GC
   
   // Для high-throughput приложений
   // -XX:+UseParallelGC
   ```
   - Выбирайте алгоритм GC в зависимости от требований приложения

2. **Настраивайте размер кучи правильно**
   ```java
   // Устанавливайте начальный и максимальный размер кучи
   // -Xms2g -Xmx4g
   ```
   - Избегайте динамического изменения размера кучи

3. **Избегайте создания временных объектов в циклах**
   ```java
   // Правильно
   StringBuilder sb = new StringBuilder();
   for (String item : items) {
       sb.append(item);
   }
   String result = sb.toString();
   ```
   - Используйте StringBuilder для конкатенации строк

### ❌ DON'T

1. **Не вызывайте System.gc() в production коде**
   ```java
   // НЕПРАВИЛЬНО
   public void cleanup() {
       System.gc(); // Не делайте этого!
   }
   ```
   - Позволяйте JVM самой решать, когда вызывать GC

2. **Не держите ссылки на объекты дольше необходимого**
   ```java
   // ПРАВИЛЬНО
   public class Cache {
       private static final Map<String, Object> cache = new WeakHashMap<>();
       
       public void add(String key, Object value) {
           cache.put(key, value); // Автоматически освобождается при нехватке памяти
       }
   }
   ```
   - Используйте WeakReference для кэшей

## Инструменты и утилиты

### Инструмент 1: JVisualVM

**Назначение:** Визуальный мониторинг JVM, включая GC

**Использование:**
```bash
# Запуск JVisualVM
jvisualvm
```

### Инструмент 2: JConsole

**Назначение:** Мониторинг производительности JVM

**Использование:**
```bash
# Запуск JConsole
jconsole
```

### Инструмент 3: GC Logs

**Назначение:** Детальный анализ работы GC

**Использование:**
```bash
# Включение GC логов
java -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```

## Дополнительные ресурсы

### Документация
- [Oracle GC Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
- [OpenJDK GC Documentation](https://openjdk.java.net/jeps/333)

### Книги
- "Java Performance: The Definitive Guide" - Scott Oaks
- "Optimizing Java" - Benjamin J. Evans, James Gough, Chris Newland

### Инструменты
- [JVisualVM](https://visualvm.github.io/) - Визуальный мониторинг JVM
- [GCViewer](https://github.com/chewiebug/GCViewer) - Анализ GC логов
- [JMH](https://openjdk.java.net/projects/code-tools/jmh/) - Microbenchmark Harness 