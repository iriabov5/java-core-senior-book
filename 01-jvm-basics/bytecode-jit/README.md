# Bytecode и JIT

## Обзор темы

**Байткод и JIT (Just‑In‑Time)** — ядро производительности JVM. Комpилятор `javac` превращает Java‑исходники в платформо‑независимый `.class` байткод, который интерпретатор и JIT компиляторы (C1/C2) исполняют и оптимизируют на лету. Понимание стека выполнения, структуры `.class`, профилирования, инлайнинга, деоптимизации и работы Code Cache — ключ к надёжным и быстрым системам.

### Что это такое?

- **Байткод**: стек‑ориентированный ISA виртуальной машины (операндный стек, локальные переменные, константный пул).
- **Интерпретация**: быстрый старт, сбор профилей ветвлений/вызовов.
- **JIT (C1/C2)**: по профилям компилирует «горячие» методы в нативный код, применяет инлайнинг, устранение виртуальности, scalar replacement, vectorization (SuperWord) и др.
- **OSR (On‑Stack Replacement)**: «подмена» уже выполняющегося фрейма на оптимизированную версию.
- **Деоптимизация**: откат к интерпретатору, если предположения JIT нарушены.

### Зачем это нужно?

- Стабильная низкая латентность под нагрузкой (после прогрева).
- Пропускная способность и энергоэффективность благодаря профилированным оптимизациям.
- Возможность точной диагностики (JFR, `-Xlog`, JITWatch) и детерминированного тюнинга.

См. также: раздел «[Принцип работы](#принцип-работы)», «[Практические примеры](#практические-примеры)», «[Инструменты и утилиты](#инструменты-и-утилиты)».

## Детальный анализ проблемы

Типичные проблемы производительности и стабильности связаны с тем, что «горячий» путь не стабилизируется: мегаморфизм интерфейсных вызовов, частые исключения, переполненный Code Cache, регулярные деоптимизации из‑за нестабильных типов, длинные safepoints.

1) Нестабильные типы на горячем вызове

```java
public interface PriceCalculator { double price(int qty); }

public final class BasicPrice implements PriceCalculator {
    public double price(int qty) { return qty * 1.0; }
}

public final class DiscountPrice implements PriceCalculator {
    public double price(int qty) { return qty * 0.9; }
}

// Проблема: подмешиваются десятки реализаций из плагинов → мегаморфизм
```

Последствия: затруднена де‑виртуализация и инлайнинг, хуже предсказуемость, больше ветвлений.

2) Исключения на горячем пути

```java
public final class Parser {
    public int parsePositiveInt(String s) {
        // Бросает NumberFormatException для нечисловых значений на горячем пути
        return Integer.parseInt(s);
    }
}
```

Последствия: блоки исключений мешают инлайнингу, увеличивают редкие пути и провоцируют деоптимизации.

3) Переполнение Code Cache

- Симптомы: снижение доли скомпилированного кода, WARN в логах, рост интерпретации.
- Причина: много горячих методов/инвалидций, маленький `ReservedCodeCacheSize`.

4) Длинные safepoints

- Симптомы: скачки latency, JFR “Long Safepoint”, `-Xlog:safepoint` показывает длительные стоп‑миры.
- Причины: большие фазовые операции (GC, deopt), тяжёлые нативные вызовы без polling, медленные страницы кучи.

См. «[Troubleshooting](#troubleshooting)» и «[Best Practices](#best-practices)».

## Принцип работы

### Архитектура исполнения JVM

```
┌────────────────────────────────────────────────────────────────┐
│                        Java Application                        │
└───────────────┬───────────────────────────┬────────────────────┘
                │                           │
        ┌───────▼────────┐        ┌─────────▼────────┐
        │ Class Loading   │        │   Runtime Data   │
        │  • load/link    │        │    Areas         │
        │  • verification │        │  • Heap          │
        └───────┬─────────┘        │  • Stacks        │
                │                  │  • Metaspace     │
        ┌───────▼─────────┐        │  • Code Cache    │
        │ Execution Engine│        └─────────┬────────┘
        │  • Interpreter  │                  │
        │  • JIT C1/C2    │          ┌───────▼───────────┐
        │  • OSR/Deopt    │          │   GC/Safepoints   │
        └─────────────────┘          └────────────────────┘
```

### Модель исполнения (stack‑based)

- Операции работают со стеком операндов: `iload`, `iadd`, `invokestatic`, `areturn` и т. п.
- Локальные переменные — отдельный массив слотов кадра вызова.
- Константный пул — ссылки на литералы, типы, поля/методы, bootstrap‑данные.

### Tiered Compilation (C1/C2)

1) Интерпретатор исполняет и собирает профили ветвлений, частоты вызовов, типизации.
2) C1 компилирует быстро с лёгкими оптимизациями (inlining по размеру, базовая LICM), может вставлять профилирующие пробы.
3) C2 компилирует «очень горячие» места с агрессивными оптимизациями: де‑виртуализация, SuperWord, scalar replacement, предикации, loop opts.
4) OSR позволяет вбросить оптимизированный фрейм прямо в середину «горячего» цикла.

### Code Cache

- Разделяется на области: non‑profiled nmethods, profiled nmethods, non‑nmethods (stubs, adapters), intrinsics.
- Тюнинг: `-XX:ReservedCodeCacheSize=256m`, мониторинг: `-Xlog:codecache=debug`.

### Safepoints, GC и JIT

- Safepoint polling встроен в скомпилированный код. Долгие нативные секции без polling задерживают стоп‑мир.
- GC влияет на latency и на моменты деоптимизации/OSR. Современные GC (G1/ZGC/Shenandoah) уменьшают паузы и воздействие на JIT.

## Детальный принцип работы

### Структура .class и константного пула

```
ClassFile {
  u4 magic;                // 0xCAFEBABE
  u2 minor_version;
  u2 major_version;
  u2 constant_pool_count;
  cp_info constant_pool[constant_pool_count-1];
  u2 access_flags;
  u2 this_class;
  u2 super_class;
  u2 interfaces_count; u2 interfaces[interfaces_count];
  u2 fields_count;      field_info fields[fields_count];
  u2 methods_count;     method_info methods[methods_count];
  u2 attributes_count;  attribute_info attributes[attributes_count];
}
```

Основные типы пула констант: `Utf8`, `Integer/Long/Float/Double`, `Class`, `String`, `Fieldref/Methodref/InterfaceMethodref`, `NameAndType`, `InvokeDynamic`.

### Мини‑пример байткода и разбор `javap`

```java
// src/main/java/org/example/Adder.java (Java 17+)
package org.example;

import java.util.logging.Level;
import java.util.logging.Logger;

public final class Adder {
    private static final Logger LOG = Logger.getLogger(Adder.class.getName());

    public static int add(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        try {
            int res = add(2, 3);
            LOG.info(() -> "Result: " + res);
        } catch (RuntimeException e) {
            LOG.log(Level.SEVERE, "Unexpected error", e);
        }
    }
}
```

Команда дизассемблирования:

```bash
javac -d out src/main/java/org/example/Adder.java
javap -c -p -v -classpath out org.example.Adder | sed -n '1,120p'
```

Ожидаемые ключевые инструкции:

```
  public static int add(int, int);
    Code:
       0: iload_0        // загрузка a на стек
       1: iload_1        // загрузка b на стек
       2: iadd           // сложение двух верхних операндов стека
       3: ireturn        // возврат значения из метода
```

Stack‑based модель: `iload_0` → [a], `iload_1` → [a, b], `iadd` → [a+b], `ireturn` возвращает верх стека.

### Ключевые инструкции байткода

- Загрузка/сохранение: `iload`, `istore`, `aload`, `astore`.
- Арифметика: `iadd`, `isub`, `imul`, `idiv`, `iinc`.
- Управление потоком: `if_icmp<cond>`, `goto`, `tableswitch`, `lookupswitch`.
- Вызовы: `invokestatic`, `invokevirtual`, `invokespecial`, `invokeinterface`, `invokedynamic`.
- Объекты: `new`, `getfield/putfield`, `getstatic/putstatic`, `checkcast`, `instanceof`.

### Профилирование, инлайнинг и деоптимизация

- Профили ветвлений и типизации направляют inlining и де‑виртуализацию.
- Ограничители инлайнинга: размер целевого метода, исключения, мегаморфизм, нестабильные профили.
- Деоптимизация происходит при изменении профиля типов/веток; наблюдаема в `-Xlog:deopt=debug` и JFR.

### Vectorization (SuperWord)

- Автоматическое слияние независимых операций над массивами в SIMD‑инструкции процессора.
- Тюнинг и диагностика: `-XX:+UseSuperWord`, `-XX:LoopMaxUnroll=16`, логи `-Xlog:jit+compilation=debug`.

## Практические примеры

### Полиморфизм и инлайнинг

```java
package org.example.inline;

import java.util.List;
import java.util.ArrayList;
import java.util.logging.Logger;

public final class PricingService {
    private static final Logger LOG = Logger.getLogger(PricingService.class.getName());

    interface PriceCalculator { double price(int qty); }

    static final class Basic implements PriceCalculator {
        public double price(int qty) { return qty * 1.0; }
    }
    static final class Discount implements PriceCalculator {
        public double price(int qty) { return qty * 0.9; }
    }

    public double total(List<PriceCalculator> calculators) {
        double sum = 0.0;
        for (PriceCalculator c : calculators) {
            sum += c.price(10);
        }
        return sum;
    }

    public static void main(String[] args) {
        List<PriceCalculator> list = new ArrayList<>();
        list.add(new Basic());
        list.add(new Discount());
        double res = new PricingService().total(list);
        LOG.info(() -> "Total: " + res);
    }
}
```

Комментарий: 1–2 реализации → likely monomorphic/polymorphic с де‑виртуализацией и инлайнингом. 10+ разных реализаций → мегаморфизм и отсутствие инлайнинга, см. «[Best Practices](#best-practices)».

### Исключения на горячем пути vs fast‑path

```java
package org.example.exceptions;

import java.util.OptionalInt;
import java.util.logging.Level;
import java.util.logging.Logger;

public final class SafeParser {
    private static final Logger LOG = Logger.getLogger(SafeParser.class.getName());

    public OptionalInt parsePositive(String s) {
        try {
            int v = Integer.parseInt(s);
            return v > 0 ? OptionalInt.of(v) : OptionalInt.empty();
        } catch (NumberFormatException e) {
            // Исключение остаётся редким путём — не на горячем пути
            LOG.log(Level.FINE, "Bad number: {0}", s);
            return OptionalInt.empty();
        }
    }

    public OptionalInt parsePositiveFast(String s) {
        // Избегаем исключения в типичном случае: предварительная проверка
        int len = s.length();
        if (len == 0) return OptionalInt.empty();
        for (int i = 0; i < len; i++) {
            char ch = s.charAt(i);
            if (ch < '0' || ch > '9') return OptionalInt.empty();
        }
        int v = Integer.parseInt(s);
        return v > 0 ? OptionalInt.of(v) : OptionalInt.empty();
    }
}
```

### Дизассемблирование цикла и OSR

```java
package org.example.osr;

public final class OsrDemo {
    public static long sumUpTo(int n) {
        long s = 0L;
        for (int i = 0; i < n; i++) s += i;
        return s;
    }

    public static void main(String[] args) {
        System.out.println(sumUpTo(100_000_000));
    }
}
```

Команды наблюдения:

```bash
java -Xlog:jit+compilation=debug -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation \
     -XX:+LogCompilation -XX:LogFile=comp.xml -cp out org.example.osr.OsrDemo | sed -n '1,60p'
```

Сигналы OSR видны как компиляции с `@` (например, `sumUpTo @ 2` — компиляция петли уровня OSR bci).

## Performance анализ

### JMH: интерпретация vs прогретый код

Maven зависимости:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.openjdk.jmh</groupId>
      <artifactId>jmh-bom</artifactId>
      <version>1.37</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<dependencies>
  <dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
  </dependency>
  <dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <scope>provided</scope>
  </dependency>
</dependencies>
```

Бенчмарки:

```java
package org.example.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5, time = 500, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 10, time = 500, timeUnit = TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgsAppend = {"-Xms512m", "-Xmx512m", "-XX:+AlwaysPreTouch"})
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class JitBasicsBench {

    interface C { int f(int x); }
    static final class A implements C { public int f(int x) { return x + 1; } }
    static final class B implements C { public int f(int x) { return x + 2; } }

    @State(Scope.Thread)
    public static class S {
        C c1 = new A();
        C c2 = new B();
        int v = 0;
    }

    @Benchmark
    public int monomorphic(S s) { // легко инлайнится
        return s.c1.f(s.v);
    }

    @Benchmark
    public int bimorphic(S s) { // всё ещё хорошо оптимизируется
        return (s.v & 1) == 0 ? s.c1.f(s.v) : s.c2.f(s.v);
    }

    @Benchmark
    public int megamorphic(S s) { // ухудшение из-за множества реализаций
        C c = ((s.v & 3) == 0) ? s.c1 : (s.v & 1) == 0 ? s.c2 : (x -> x + 3);
        return c.f(s.v);
    }

    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
                .include(JitBasicsBench.class.getSimpleName())
                .detectJvmArgs()
                .build();
        new Runner(opt).run();
    }
}
```

Ожидаемые результаты (на типичном x86_64, Java 17+):

- Монопольный вызов — минимальное время, инлайнинг.
- Биморфный — близко к монопольному.
- Мегаморфный — заметно хуже из‑за отсутствия де‑виртуализации.

### Влияние исключений

```java
package org.example.bench;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@Warmup(iterations = 5)
@Measurement(iterations = 10)
@Fork(1)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public class ExceptionBench {

    @Param({"42", "bad"})
    public String input;

    @Benchmark
    public int withException() {
        try {
            return Integer.parseInt(input);
        } catch (NumberFormatException e) {
            return -1;
        }
    }

    @Benchmark
    public int withoutException() {
        String s = input;
        for (int i = 0, n = s.length(); i < n; i++) {
            char ch = s.charAt(i);
            if (ch < '0' || ch > '9') return -1;
        }
        return Integer.parseInt(s);
    }
}
```

Ожидание: профиль без исключений стабилен и быстрее при «грязных» данных.

### Наблюдение профилей JIT/JFR

```java
package org.example.observe;

import jdk.jfr.Recording;
import jdk.jfr.RecordingState;
import jdk.jfr.consumer.RecordedEvent;
import jdk.jfr.consumer.RecordingFile;

import java.nio.file.Path;
import java.time.Duration;
import java.util.List;

public final class JfrObserveDemo {
    public static void main(String[] args) throws Exception {
        Path file = Path.of("jit.jfr");
        try (Recording r = new Recording()) {
            r.enable("jdk.Compilation").withThreshold(Duration.ofMillis(0));
            r.enable("jdk.CodeCacheConfiguration");
            r.enable("jdk.CodeCacheFull");
            r.start();

            busyLoop();

            r.stop();
            r.dump(file);
        }

        List<RecordedEvent> events = RecordingFile.readAllEvents(file);
        long compilations = events.stream().filter(e -> e.getEventType().getName().contains("Compilation")).count();
        System.out.println("Compilations: " + compilations);
    }

    private static void busyLoop() {
        long s = 0L;
        for (int j = 0; j < 10; j++) {
            for (int i = 0; i < 10_000_000; i++) s += i;
        }
        System.out.println(s);
    }
}
```

## Troubleshooting

### Частые деоптимизации

- Симптомы: `-Xlog:deopt*=debug` показывает частые deopt, JFR события Deoptimization.
- Причины: нестабильные типы, исключения на горячем пути, мегаморфизм, большие методы мешают инлайнингу.
- Решения:
  - Стабилизировать типы, упрощать иерархии; sealed‑типы/records.
  - Убрать исключения с горячего пути (предикаты/валидации).
  - Тюнинг инлайнинга: `-XX:MaxInlineSize=60`, `-XX:FreqInlineSize=120` (с осторожностью).

### Переполнение Code Cache

- Симптомы: WARN в `-Xlog:codecache=debug`, падение компиляций, рост интерпретации.
- Причины: недостаточный `ReservedCodeCacheSize`, большое количество прогретых методов.
- Решения: увеличить `-XX:ReservedCodeCacheSize=256m` или выше, следить за фрагментацией (`-Xlog:codecache=trace`).

### Длинные safepoints

- Симптомы: пики latency, `-Xlog:safepoint=info` и JFR Long Safepoint.
- Причины: тяжёлые нативные секции, большие stop‑the‑world фазы GC, медленные page faults.
- Решения: избегать длительных JNI без safepoint polling, тюнинг GC под SLA (например, G1/ZGC), `-XX:+AlwaysPreTouch`.

### Мегаморфизм вызовов

- Симптомы: отсутствие инлайнинга, низкий профиль де‑виртуализации, просадка в бенчах.
- Причины: слишком много реализаций интерфейса в горячей точке.
- Решения: стабилизировать call‑site (кеширование конкретной реализации, sealed иерархии, ручной диспетчинг switch по enum).

### Исключения на hot path

- Симптомы: горячий метод видит частые exceptions, `-Xlog:exceptions=trace` шумит.
- Решения: выносить исключения за горячий путь, предварительная валидация, fast‑path и редкий slow‑path с исключениями.

## Best Practices

- Стабилизируйте профили: разнесите «шумные» ветки, уменьшайте количество реализаций на горячем вызове.
- Избегайте исключений на hot path: используйте проверку входных данных и коды результатов/Optional.
- Управляйте инлайнингом осознанно: маленькие, чистые методы; не перегружайте огромными try/catch.
- Прогревайте систему: JIT нуждается в времени для профилей — прогрев в JMH и в проде (pre‑warm endpoints, CDS/AppCDS).
- Следите за Code Cache: `-Xlog:codecache`, тюнинг `-XX:ReservedCodeCacheSize`.
- Используйте JFR + JITWatch: просматривайте цепочки инлайнинга, причины отказа, места деоптимизаций.
- Оптимизируйте векторизацию: простые, предсказуемые циклы над массивами; избегайте зависимостей между итерациями.

## Инструменты и утилиты

### javap

```bash
javap -c -p -v -classpath out org.example.Adder | sed -n '1,80p'
```

### JITWatch

- Сбор логов: `-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:LogFile=jit.xml`.
- Анализ `jit.xml` и просмотра графов инлайнинга, причин отказа.

### JFR

Запуск с профилем:

```bash
java -XX:StartFlightRecording=name=jit,settings=profile,filename=jit.jfr -XX:FlightRecorderOptions=stackdepth=128 -jar app.jar
```

### jcmd

```bash
jcmd <pid> VM.version | cat
jcmd <pid> Compiler.codecache | cat
jcmd <pid> Compiler.queue | cat
jcmd <pid> JFR.check | cat
```

### Логи JVM

```bash
java -Xlog:jit+compilation=debug,safepoint=info,gc=info,class+load=info -jar app.jar | sed -n '1,120p'
```

### jstat, jmap

```bash
jstat -compiler <pid> 1s 5 | cat
jmap -clstats <pid> | cat
```

## Дополнительные ресурсы

- Официальная спецификация JVM (JVMS): `https://docs.oracle.com/javase/specs/jvms/se17/html/`
- HotSpot Internals и JIT: `https://wiki.openjdk.org/display/HotSpot/`
- JITWatch: `https://github.com/AdoptOpenJDK/jitwatch`
- JFR Руководство: `https://docs.oracle.com/javacomponents/jmc-8-2/jfr-runtime-guide/`
- JMH: `https://openjdk.org/projects/code-tools/jmh/`
- Safepoints и GC: `https://wiki.openjdk.org/display/HotSpot/Safepoint`

## Вопросы для собеседования (Senior)

- **Объясните разницу между интерпретацией, C1 и C2, и зачем нужен tiered compilation?**
  - **Ответ**: Интерпретатор быстро стартует и собирает профили. C1 компилирует оперативно с лёгкими оптимизациями и, при необходимости, продолжает профилировать. C2 применяет агрессивные оптимизации на стабилизированных профилях. Tiered даёт быстрый старт и высокий steady‑state performance.
  - **Follow‑ups**: Как влияет OSR? Как warmup в JMH отражает реальный прод?
  - **Red flags**: «JIT только замедляет старт», «C2 всегда лучше C1».
  - См. разделы: «[Принцип работы](#принцип-работы)», «[Performance анализ](#performance-анализ)».

- **Что такое деоптимизация, когда она происходит и как её диагностировать?**
  - **Ответ**: Откат из JIT‑кода к интерпретатору при нарушении предположений (типы/ветки), invalidation зависимостей. Диагностируется через `-Xlog:deopt=debug`, JFR события Deoptimization, JITWatch.
  - **Follow‑ups**: Как уменьшить частоту deopt? Влияет ли исключение на hot path?
  - **Red flags**: «Деоптимизация — это баг JVM», «Её нельзя увидеть».
  - См.: «[Детальный принцип работы](#детальный-принцип-работы)», «[Troubleshooting](#troubleshooting)».

- **Как работает инлайнинг и какие параметры его ограничивают?**
  - **Ответ**: Инлайнинг разворачивает целевой метод в вызывающий, уменьшая вызовы и раскрывая оптимизации. Ограничивается размером, исключениями, мегаморфизмом, нестабильными профилями. Параметры: `-XX:MaxInlineSize`, `-XX:FreqInlineSize`.
  - **Follow‑ups**: Почему большие try/catch мешают инлайнингу? Как sealed‑типы помогают?
  - **Red flags**: «JIT всегда инлайнит всё», «Inline = микрооптимизация без последствий».
  - См.: «[Профилирование, инлайнинг и деоптимизация](#профилирование-инлайнинг-и-деоптимизация)».

- **Объясните мегаморфные вызовы и их влияние на оптимизации.**
  - **Ответ**: Когда call‑site видит много реализаций, JIT не может эффективно де‑виртуализировать и инлайнить, ухудшается предсказуемость и производительность. Лечат стабилизацией типов, переразбиением иерархий, sealed, кешированием.
  - **Follow‑ups**: Чем мономорфизм отличается от биморфизма в оптимизациях?
  - **Red flags**: «Кол‑во реализаций не важно», «Интерфейсы всегда бесплатны».
  - См.: «[Best Practices](#best-practices)», «[Практические примеры](#практические-примеры)».

- **Что такое OSR и где он возникает?**
  - **Ответ**: On‑Stack Replacement — компиляция/подмена фрейма цикла, уже выполняющегося, чтобы продолжить его в JIT‑коде. Возникает в долгих циклах и помогает быстро получить выгоду от оптимизаций без перезапуска метода.
  - **Follow‑ups**: Как увидеть OSR в логах? Как влияет на бенчмарки?
  - **Red flags**: «OSR — это только для исключений», «Его нельзя контролировать».
  - См.: «[Практические примеры](#практические-примеры)», «[Инструменты и утилиты](#инструменты-и-утилиты)».

- **Как устроен Code Cache, как диагностировать переполнение и что тюнить?**
  - **Ответ**: Области nmethods/stubs/intrinsics. Переполнение снижает компиляции, растёт интерпретация. Диагностика `-Xlog:codecache=debug`, jcmd `Compiler.codecache`. Тюнинг `-XX:ReservedCodeCacheSize`.
  - **Follow‑ups**: Что с фрагментацией? Как влияет динамическая генерация байткода?
  - **Red flags**: «Code Cache бесконечен», «Нет влияния на latency».
  - См.: «[Принцип работы](#принцип-работы)», «[Troubleshooting](#troubleshooting)».

- **Где и как использовать JFR/JITWatch для анализа производительности JIT?**
  - **Ответ**: JFR фиксирует события компиляций, деоптимизаций, safepoints; JITWatch объясняет инлайнинг/деоптимизации по `LogCompilation`. Используются совместно для root‑cause анализа.
  - **Follow‑ups**: Какие JFR события включить? Как читать `jit.xml`?
  - **Red flags**: «JFR слишком тяжёлый для продакшена» (профиль `continuous` дешев).
  - См.: «[Инструменты и утилиты](#инструменты-и-утилиты)».

- **Как исключения на hot path влияют на JIT и как этого избежать?**
  - **Ответ**: Исключения создают редкие пути и мешают инлайнингу, могут вызывать деоптимизации. Следует валидировать вход заранее, переносить исключения в slow‑path.
  - **Follow‑ups**: Когда исключение оправдано? Как это проверить в JMH?
  - **Red flags**: «Исключения бесплатны», «try/catch ускоряет код».
  - См.: «[Практические примеры](#практические-примеры)», «[Best Practices](#best-practices)».

- **Какие JVM флаги полезны для диагностики JIT/байткода/компиляции?**
  - **Ответ**: `-Xlog:jit*,safepoint,gc,class+load`, `-XX:+PrintCompilation`, `-XX:+LogCompilation -XX:LogFile=comp.xml`, `-XX:+UnlockDiagnosticVMOptions`, `-XX:ReservedCodeCacheSize`.
  - **Follow‑ups**: Что включить в проде? Как уменьшить шум логов?
  - **Red flags**: «Достаточно только PrintCompilation».
  - См.: «[Инструменты и утилиты](#инструменты-и-утилиты)», «[Troubleshooting](#troubleshooting)».

- **Что такое SuperWord и когда происходит авто‑векторизация?**
  - **Ответ**: SuperWord сливает независимые операции над массивами в SIMD, когда отсутствуют зависимые записи/чтения, выровнены шаги цикла и понятны границы. Включено по умолчанию на HotSpot.
  - **Follow‑ups**: Как помочь векторизации кодом? Как увидеть эффект?
  - **Red flags**: «Всегда срабатывает», «Нужно руками писать intrinsics».
  - См.: «[Детальный принцип работы](#детальный-принцип-работы)», «[Performance анализ](#performance-анализ)».

- **Как GC влияет на JIT и latency?**
  - **Ответ**: GC инициирует safepoints и может задерживать выполнение JIT‑кода; некоторые GC минимизируют паузы (ZGC/Shenandoah). Параметры кучи/пред‑толчок страниц влияют на stop‑the‑world и прогрев.
  - **Follow‑ups**: Как измерить safepoint time? Что даёт `-XX:+AlwaysPreTouch`?
  - **Red flags**: «GC не влияет на JIT», «Safepoints только для GC».
  - См.: «[Принцип работы](#принцип-работы)», «[Инструменты и утилиты](#инструменты-и-утилиты)».

- **Какие практики помогут стабилизировать профили под высокую нагрузку?**
  - **Ответ**: Уменьшение полиморфизма, прогрев, CDS/AppCDS, предсказуемые ветви, отказ от исключений на hot path, контроль inlining, мониторинг Code Cache/JFR.
  - **Follow‑ups**: Как это проверить до релиза? Какие метрики JFR критичны?
  - **Red flags**: «Достаточно один раз прогреть локально», «Инлайнинг не важен».
  - См.: «[Best Practices](#best-practices)», «[Performance анализ](#performance-анализ)».

