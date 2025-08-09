# Class Loading

## Обзор темы

**Class Loading** — это механизм JVM, отвечающий за поиск, загрузку, связывание и инициализацию классов перед их выполнением. Понимание загрузки классов критично для разработки надёжных, модульных и расширяемых Java‑приложений (плагины, контейнеры, app servers, фреймворки).

### Что это такое?

Загрузка классов включает три крупных этапа жизненного цикла класса в JVM:

1. **Loading (Загрузка)**: поиск байткода и создание объектного представления `java.lang.Class`.
2. **Linking (Связывание)**: верификация, подготовка и (ленивая) резолвинг ссылок.
3. **Initialization (Инициализация)**: выполнение статических инициализаторов и присвоение значения `static` полям.

Работу выполняют **загрузчики классов (Class Loaders)** в иерархии делегирования: Bootstrap → Platform (ранее Extension) → Application → Custom.

### Проблема, которую решает тема

Типичные проблемы в продакшене связаны с неверным класс‑путём, конфликтующими версиями библиотек и изоляцией модулей:

1. `ClassNotFoundException`/`NoClassDefFoundError` при неправильном `classpath`/`module-path`.
2. Конфликты версий зависимостей и «скрытые» классы в разных загрузчиках.
3. `ClassCastException` между «одинаковыми» классами, загруженными разными `ClassLoader`.
4. Ошибки инициализации: `ExceptionInInitializerError`, порядок инициализаторов, циклы.

Эта глава покажет, как устроена загрузка классов, как диагностировать и исправлять такие ситуации, и как безопасно внедрять динамическую загрузку (плагины).

## Детальный анализ проблем

### Сценарий 1: ClassNotFoundException vs NoClassDefFoundError

```java
public class LoaderDemo {
    public static void main(String[] args) throws Exception {
        // Инициализирует класс: выполнит статические блоки
        Class<?> c1 = Class.forName("com.example.Missing"); // ClassNotFoundException, если не найден

        // НЕ инициализирует класс, только загружает описание
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        Class<?> c2 = cl.loadClass("com.example.Missing"); // ClassNotFoundException, если не найден
    }
}
```

```java
public class BadInit {
    static {
        if (true) throw new RuntimeException("init failed");
    }
}

class UseBadInit {
    public static void main(String[] args) {
        try {
            Class.forName("BadInit"); // ExceptionInInitializerError при первой инициализации
        } catch (Throwable e) {
            e.printStackTrace();
        }
        // На последующих обращениях может прилететь NoClassDefFoundError,
        // т.к. класс уже помечен как проваливший инициализацию.
        new BadInit(); // NoClassDefFoundError
    }
}
```

Ключевая разница:
- **ClassNotFoundException** — байткод не найден загрузчиком.
- **NoClassDefFoundError** — класс был найден ранее, но не может быть использован (часто из‑за ошибки инициализации или изменений после компиляции).

### Сценарий 2: «Одинаковые» классы из разных ClassLoader

```java
public class TypeIdentityDemo {
    public static void main(String[] args) throws Exception {
        var jarUrl = new java.io.File("./plugins/sample.jar").toURI().toURL();
        var loaderA = new java.net.URLClassLoader(new java.net.URL[]{jarUrl}, null);
        var loaderB = new java.net.URLClassLoader(new java.net.URL[]{jarUrl}, null);

        Class<?> a = loaderA.loadClass("plugin.Api");
        Class<?> b = loaderB.loadClass("plugin.Api");

        System.out.println(a == b); // false — разные загрузчики, разные типы

        // Приведение типов между разными загрузчиками вызовет ClassCastException,
        // даже если имя класса совпадает.
    }
}
```

Вывод: идентичность типа в JVM — это пара (имя класса, загрузчик). Разные загрузчики → разные типы.

### Сценарий 3: Порядок инициализации и циклы

```java
class A { static { B.touch(); } static void touch() {} }
class B { static { A.touch(); } static void touch() {} }

public class InitOrder {
    public static void main(String[] args) throws Exception {
        Class.forName("A"); // может привести к сложным сценариям инициализации
    }
}
```

Рекомендуется минимизировать «работу» в статических инициализаторах, избегать циклической инициализации.

## Принцип работы

### Иерархия загрузчиков и делегирование

```
┌──────────────────────────────────────────────┐
│                  JVM                        │
│  ┌────────────┐  ┌─────────────┐  ┌───────┐ │
│  │ Bootstrap  │→ │  Platform   │→ │ App   │→ … Custom
│  │ (native)   │  │ (JDK libs)  │  │(AppCL)│ │
│  └────────────┘  └─────────────┘  └───────┘ │
└──────────────────────────────────────────────┘
```

Модель по умолчанию — «parent‑first»: каждый загрузчик сначала делегирует поиск родителю, и лишь затем пытается загрузить сам. Это предотвращает подмену базовых классов JDK.

Основные загрузчики:
- **Bootstrap**: загружает базовые классы JDK (например, `java.*`). Реализован на нативном уровне.
- **Platform**: библиотеки платформы (JDK вне базового ядра).
- **Application**: классы приложения из `classpath`.
- **Custom**: пользовательские (для плагинов, изоляции модулей и т.п.).

### Жизненный цикл класса

1. **Loading**: поиск ресурса `.class` и создание `Class<?>` через `defineClass`.
2. **Linking**:
   - Verification (верификация байткода);
   - Preparation (подготовка статических полей, выделение памяти, значения по умолчанию);
   - Resolution (ленивое разрешение символьных ссылок при обращении).
3. **Initialization**: выполнение `<clinit>` — статических блоков и присваиваний `static`.

С Java 8 классы и метаданные хранятся в **Metaspace** (вместо устаревшего PermGen).

## Детальный принцип работы

### Различия Class.forName и ClassLoader#loadClass

```java
// Инициализирует класс (выполнит статические блоки)
Class<?> c1 = Class.forName("com.example.Foo");

// Только загрузит (без инициализации); инициализация произойдёт при первом обращении к статике
Class<?> c2 = Thread.currentThread().getContextClassLoader().loadClass("com.example.Foo");
```

### Пользовательский ClassLoader (под плагины)

```java
public class PluginClassLoader extends ClassLoader {
    private final java.nio.file.Path root;

    public PluginClassLoader(ClassLoader parent, java.nio.file.Path root) {
        super(parent);
        this.root = root;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            String rel = name.replace('.', '/') + ".class";
            byte[] bytes = java.nio.file.Files.readAllBytes(root.resolve(rel));
            return defineClass(name, bytes, 0, bytes.length);
        } catch (java.io.IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

По умолчанию будет работать делегирование «parent‑first». Если требуется «child‑first» (как в некоторых контейнерах/OSGi), меняют стратегию поиска: сначала `findClass`, затем делегирование (используйте осторожно из‑за безопасности и совместимости).

### Контекстный ClassLoader

Многие фреймворки используют `Thread.currentThread().getContextClassLoader()` для разрешения классов/ресурсов во время выполнения. Библиотекам следует предпочитать контекстный загрузчик вместо `ClassName.class.getClassLoader()` для поддержки контейнеров и плагинов.

### Модули (Java 9+)

Система модулей добавляет `module-path` и правила экспорта/доступа. Классы могут отсутствовать не только из‑за `classpath`, но и из‑за ограничения модульной доступности. Для диагностики применяйте `jdeps`, флаги `--add-opens`, `--add-exports` при необходимости.

## Практические примеры

### Пример 1: Динамическая загрузка класса без инициализации

```java
public class LazyLoadExample {
    public static void main(String[] args) throws Exception {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        Class<?> clazz = cl.loadClass("com.example.Service"); // без инициализации
        System.out.println("Loaded: " + clazz.getName());
    }
}
```

### Пример 2: Динамическая загрузка JAR из директории плагинов

```java
public class PluginLoaderExample {
    public static void main(String[] args) throws Exception {
        java.nio.file.Path plugins = java.nio.file.Paths.get("./plugins");
        java.util.List<java.net.URL> urls = new java.util.ArrayList<>();
        try (var stream = java.nio.file.Files.list(plugins)) {
            stream.filter(p -> p.toString().endsWith(".jar")).forEach(p -> {
                try { urls.add(p.toUri().toURL()); } catch (Exception ignored) {}
            });
        }
        try (var cl = new java.net.URLClassLoader(urls.toArray(java.net.URL[]::new),
                PluginLoaderExample.class.getClassLoader())) {
            Class<?> pluginApi = cl.loadClass("plugin.Api");
            Object plugin = pluginApi.getDeclaredConstructor().newInstance();
            System.out.println(plugin.getClass() + " via " + plugin.getClass().getClassLoader());
        }
    }
}
```

### Пример 3: Диагностика ресурсов и classpath

```java
public class ResourceCheck {
    public static void main(String[] args) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        var res = cl.getResource("config/application.yaml");
        System.out.println(res != null ? res : "resource not found");

        System.out.println("classpath = " + System.getProperty("java.class.path"));
    }
}
```

### Пример 4: Демонстрация проблем с идентичностью типа

```java
// Предположим, интерфейс plugin.Api загружается из двух разных JAR/загрузчиков
public class CastProblem {
    public static void main(String[] args) throws Exception {
        java.net.URL jar = new java.io.File("./plugins/api.jar").toURI().toURL();
        var a = new java.net.URLClassLoader(new java.net.URL[]{jar}, null);
        var b = new java.net.URLClassLoader(new java.net.URL[]{jar}, null);

        Class<?> apiA = a.loadClass("plugin.Api");
        Class<?> apiB = b.loadClass("plugin.Api");

        Object o = apiA.getDeclaredConstructor().newInstance();
        // Следующая строка кинет ClassCastException, т.к. apiA != apiB
        System.out.println(apiB.cast(o));
    }
}
```

## Performance и Optimization

- **Кэширование**: JVM кэширует загруженные классы в рамках загрузчика; повторные запросы дешевы.
- **Стоимость поиска**: сканирование `classpath`/`module-path` и JAR‑файлов; избегайте больших, неиспользуемых JAR.
- **CDS/AppCDS**: Class Data Sharing позволяет шарить прединициализированные метаданные классов между процессами, ускоряя старт и уменьшая потребление памяти.

Примеры запуска с CDS (версии JDK могут отличаться):

```bash
# Генерация архива AppCDS (примерная последовательность)
java -Xshare:dump
java -XX:ArchiveClassesAtExit=app-cds.jsa -cp app.jar com.example.Main
java -Xshare:on -XX:SharedArchiveFile=app-cds.jsa -cp app.jar com.example.Main
```

Мониторинг классов:

```bash
# Счётчики загрузки классов
jstat -class <PID> 1000

# Потребление Metaspace
jcmd <PID> VM.native_memory summary | cat

# Статистика классов (если доступно)
jcmd <PID> GC.class_stats | cat
```

## Troubleshooting

### Частые ошибки и причины

- **ClassNotFoundException**: отсутствует класс в `classpath`/`module-path`; опечатка в FQN.
- **NoClassDefFoundError**: класс найден ранее, но недоступен (ошибка инициализации, несовместимая версия, удаление JAR в рантайме).
- **ExceptionInInitializerError**: ошибка в статическом блоке/инициализации `static` полей.
- **ClassCastException (разные загрузчики)**: «одинаковые» классы из разных `ClassLoader` — изоляция модулей нарушена.
- **IncompatibleClassChangeError/NoSuchMethodError/NoSuchFieldError**: версия бинарной совместимости библиотеки несовместима с скомпилированным кодом.

### Чек‑лист диагностики

1. Выведите classpath/module‑path: `System.getProperty("java.class.path")`.
2. Проверьте версию класса: `javap -verbose MyClass | grep "major version"`.
3. Посмотрите загрузчик класса: `obj.getClass().getClassLoader()`.
4. Проверьте доступность ресурса: `cl.getResource(name)`.
5. Для модулей — `jdeps app.jar` и флаги `--add-opens/--add-exports` при необходимости.
6. Посмотрите Metaspace и количество классов: `jstat -class`, `jcmd VM.native_memory`.

### Быстрые исправления

- Убедитесь, что все необходимые JAR в `classpath` и соответствуют версиям компиляции.
- Исключите дубли библиотек; используйте shading/relocation при сборке плагинов.
- Для плагинов — чёткая изоляция: отдельные `ClassLoader` и узкий контракт API, известный родителю.
- Не держите сильные ссылки на классы/объекты из выгружаемых загрузчиков (иначе утечки Metaspace).

## Best Practices

### ✅ DO

1. Явно управляйте иерархией `ClassLoader` в контейнерах/плагинах.
2. Используйте контекстный `ClassLoader` в библиотеках для загрузки ресурсов/классов.
3. Минимизируйте логику в статических инициализаторах; избегайте циклов.
4. Изолируйте плагины: API в родительском загрузчике, реализации — в дочерних.
5. Контролируйте версии зависимостей; используйте `jdeps` для анализа.

### ❌ DON'T

1. Не подменяйте классы JDK (child‑first без крайней необходимости).
2. Не полагайтесь на порядок JAR при конфликтующих версиях — разрешайте конфликты явно.
3. Не удерживайте ссылки на классы из выгружаемых загрузчиков (утечки Metaspace).
4. Не используйте `System.gc()` как «починку» проблем инициализации — это не связано.

## Инструменты и утилиты

- **jstat -class**: счётчики загрузок/разгрузок классов.
- **jcmd VM.native_memory summary**: оценка Metaspace.
- **jcmd GC.class_stats**: статистика по классам (если поддерживается).
- **jdeps**: анализ зависимостей и модульной доступности.
- **javap**: инспекция байткода и версии классов.
- **jar tf**: просмотр содержимого JAR.

## Дополнительные ресурсы

- Документация JVM Class File & Class Loading: [JVM Spec, JVMS 5](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-5.html)
- Ресурсы по Metaspace: [Metaspace Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
- Анализ модулей и доступности: [JDeps Guide](https://docs.oracle.com/javase/9/tools/jdeps.htm)