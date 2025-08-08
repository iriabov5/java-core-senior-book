# Git Workflow для создания курса по информационной безопасности

## Начальная настройка проекта

### Шаг 1: Создание структуры проекта

```bash
# Создаем основную папку проекта
mkdir info-security-course
cd info-security-course

# Создаем структуру папок
mkdir -p 01-basics
mkdir -p 02-organizational
mkdir -p 03-legal-regulatory
mkdir -p 04-threat-models
mkdir -p 05-risk-management
mkdir -p 06-incident-management
mkdir -p 07-awareness-training
mkdir -p 08-tools-technologies
mkdir -p 09-audit-compliance
mkdir -p 10-best-practices
mkdir -p 11-real-cases
mkdir -p 12-conclusion

# Создаем основные файлы
touch README.md
touch SUMMARY.md
touch .gitignore
```

### Шаг 2: Создание .gitignore

```bash
# Создаем .gitignore файл
cat > .gitignore << EOF
# Временные файлы
*.tmp
*.temp
*~

# Системные файлы
.DS_Store
Thumbs.db

# IDE файлы
.vscode/
.idea/
*.swp
*.swo

# Логи
*.log

# Временные папки
temp/
tmp/
EOF
```

### Шаг 3: Инициализация Git

```bash
# Инициализируем git репозиторий
git init

# Добавляем все файлы
git add .

# Первый коммит
git commit -m "feat: Initial project structure

- Created main directories for course sections
- Added README.md with course overview
- Added SUMMARY.md placeholder
- Added .gitignore for common files
- Set up basic project structure"

# Создаем репозиторий на GitHub/GitLab и добавляем remote
git remote add origin https://github.com/your-username/info-security-course.git
git push -u origin main
```

## Workflow для создания контента

### Этап 1: Создание SUMMARY.md

```bash
# Создаем SUMMARY.md с помощью промпта
# (используйте промпт 1 из PROMPTS.md)

# После создания SUMMARY.md
git add SUMMARY.md
git commit -m "feat: Add course table of contents

- Added comprehensive SUMMARY.md structure
- Included all 12 main sections
- Added subsections for detailed topics
- Created logical navigation structure"
git push origin main
```

### Этап 2: Создание основного раздела

```bash
# Создаем раздел "Основы информационной безопасности"
# (используйте промпт 2 из PROMPTS.md)

# После создания раздела
git add 01-basics/
git commit -m "feat: Add basics of information security

- Added overview of information security concepts
- Included CIA triad explanation
- Added threat and vulnerability analysis
- Included security models overview
- Added practical examples and templates
- Added checklists and procedures"
git push origin main
```

### Этап 3: Создание подразделов

```bash
# Создаем подраздел "Модели угроз"
# (используйте промпт 5 из PROMPTS.md)

git add 01-basics/threat-models.md
git commit -m "feat: Add threat modeling section

- Added STRIDE methodology explanation
- Included MITRE ATT&CK framework overview
- Added practical threat modeling examples
- Included templates and checklists
- Added tools and resources section
- Added real-world case studies"
git push origin main
```

## Шаблоны коммитов

### Для новых разделов:
```bash
git commit -m "feat: Add [название раздела]

- Added [основные компоненты]
- Included [ключевые элементы]
- Added [практические материалы]
- Included [шаблоны/чек-листы]
- Added [примеры/кейсы]"
```

### Для обновлений:
```bash
git commit -m "fix: Update [название раздела]

- Fixed [исправления]
- Updated [обновления]
- Added [новые материалы]
- Improved [улучшения]
- Updated [ссылки/документы]"
```

### Для исправлений:
```bash
git commit -m "fix: Fix [описание проблемы]

- Fixed [конкретное исправление]
- Updated [обновленный контент]
- Corrected [исправленная ошибка]
- Improved [улучшение]"
```

## Рекомендуемый порядок создания

### Неделя 1: Основы
```bash
# День 1-2: SUMMARY.md
git add SUMMARY.md
git commit -m "feat: Add course table of contents"

# День 3-5: Основы информационной безопасности
git add 01-basics/
git commit -m "feat: Add basics of information security"

# День 6-7: Организационные аспекты
git add 02-organizational/
git commit -m "feat: Add organizational aspects"
```

### Неделя 2: Регуляторы
```bash
# День 1-3: Юридические и нормативные требования
git add 03-legal-regulatory/
git commit -m "feat: Add legal and regulatory requirements"

# День 4-7: Модели угроз
git add 04-threat-models/
git commit -m "feat: Add threat modeling section"
```

### Неделя 3: Управление
```bash
# День 1-3: Управление рисками
git add 05-risk-management/
git commit -m "feat: Add risk management section"

# День 4-7: Инцидент-менеджмент
git add 06-incident-management/
git commit -m "feat: Add incident management section"
```

### Неделя 4: Практика
```bash
# День 1-3: Обучение и осведомленность
git add 07-awareness-training/
git commit -m "feat: Add awareness and training section"

# День 4-7: Инструменты и технологии
git add 08-tools-technologies/
git commit -m "feat: Add tools and technologies overview"
```

### Неделя 5: Завершение
```bash
# День 1-2: Аудит и соответствие
git add 09-audit-compliance/
git commit -m "feat: Add audit and compliance section"

# День 3-4: Лучшие практики
git add 10-best-practices/
git commit -m "feat: Add best practices section"

# День 5-6: Реальные кейсы
git add 11-real-cases/
git commit -m "feat: Add real-world cases"

# День 7: Заключение
git add 12-conclusion/
git commit -m "feat: Add course conclusion"
```

## Полезные команды Git

### Просмотр статуса и истории
```bash
# Просмотр статуса
git status

# Просмотр истории коммитов
git log --oneline

# Просмотр изменений в файле
git diff filename.md

# Просмотр изменений в последнем коммите
git show
```

### Работа с ветками
```bash
# Создание новой ветки
git checkout -b feature/new-section

# Переключение между ветками
git checkout main

# Слияние веток
git merge feature/new-section

# Удаление ветки
git branch -d feature/new-section

# Просмотр всех веток
git branch -a
```

### Отмена изменений
```bash
# Отмена последнего коммита (сохраняя изменения)
git reset --soft HEAD~1

# Отмена последнего коммита (удаляя изменения)
git reset --hard HEAD~1

# Отмена изменений в рабочей директории
git checkout -- filename.md

# Отмена добавленных файлов
git reset HEAD filename.md
```

### Синхронизация с удаленным репозиторием
```bash
# Получение изменений
git pull origin main

# Отправка изменений
git push origin main

# Принудительная отправка (осторожно!)
git push --force origin main
```

## Советы по работе

### 1. Регулярность коммитов
- Делайте коммиты после каждого значимого изменения
- Не накапливайте много изменений в одном коммите
- Пишите понятные сообщения коммитов

### 2. Структура сообщений коммитов
```
тип: Краткое описание

- Детальное описание изменений
- Список добавленных компонентов
- Ссылки на связанные материалы
```

### 3. Типы коммитов
- `feat:` - новые функции/разделы
- `fix:` - исправления
- `docs:` - документация
- `style:` - форматирование
- `refactor:` - рефакторинг
- `test:` - тесты
- `chore:` - обслуживание

### 4. Работа с большими изменениями
```bash
# Создание feature ветки
git checkout -b feature/major-section

# Работа над изменениями
# ... создание контента ...

# Коммиты по частям
git add section1/
git commit -m "feat: Add section 1"

git add section2/
git commit -m "feat: Add section 2"

# Слияние с основной веткой
git checkout main
git merge feature/major-section
```

### 5. Резервное копирование
```bash
# Создание тега для важных версий
git tag -a v1.0.0 -m "First complete version"

# Отправка тегов
git push origin --tags

# Создание архива
git archive --format=zip --output=course-v1.0.0.zip v1.0.0
```

## Автоматизация

### Создание скрипта для быстрого коммита
```bash
# Создаем скрипт commit.sh
cat > commit.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: ./commit.sh <section-name> <description>"
    exit 1
fi

SECTION=$1
DESCRIPTION=$2

git add .
git commit -m "feat: Add $SECTION

- Added $DESCRIPTION
- Included practical examples
- Added templates and checklists
- Added real-world cases"

git push origin main
EOF

chmod +x commit.sh

# Использование
./commit.sh "threat modeling" "comprehensive threat modeling section"
```

### Создание скрипта для обновления
```bash
# Создаем скрипт update.sh
cat > update.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: ./update.sh <section-name> <description>"
    exit 1
fi

SECTION=$1
DESCRIPTION=$2

git add .
git commit -m "fix: Update $SECTION

- Updated $DESCRIPTION
- Fixed typos and errors
- Improved examples and templates
- Updated links and references"

git push origin main
EOF

chmod +x update.sh
```

Этот workflow поможет вам систематически создавать курс по информационной безопасности с правильным управлением версиями в Git. 