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

### Ключевые преимущества

1. **Автоматическое управление памятью** - разработчики не должны вручную освобождать память
2. **Предотвращение утечек памяти** - GC автоматически находит и освобождает неиспользуемые объекты
3. **Упрощение разработки** - можно сосредоточиться на бизнес-логике
4. **Стабильность приложений** - предотвращение OutOfMemoryError
5. **Оптимизация производительности** - различные алгоритмы GC для разных сценариев

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