---
layout: default
title: "Работа с Git и GitHub"
permalink: /1. Погружение в основы/Работа с Git и GitHub/
---

*GIt - это обязательный инструмент, который поможет тебе смотреть и создавать разные версии проекта, контролировать их и откатываться к предыдущим версиям. На любой НОРМАЛЬНОЙ работе есть Git и Gitlab. Для начала лучше изучи Github, потому что Gitlab сложнее воспринимать, его мы изучим потом*

Также посмотри про git для новичков - https://learngitbranching.js.org/

## Регистрация и начальная настройка
---
### Создание аккаунта
1. Перейди на [github.com](https://github.com)
2. Зарегистрируйся, используя email
3. Подтверди email адрес
4. Настрой двухфакторную аутентификацию для безопасности

### Базовая конфигурация Git
```bash
# Установи свое имя и email
git config --global user.name "Твое Имя"
git config --global user.email "твой.email@example.com"

# Посмотреть настройки
git config --list
```

## Основные понятия Git
---
### Репозиторий (Repository)
- Папка проекта, которую отслеживает Git
- Может быть локальной (на твоем компьютере) или удаленной (на GitHub)

### Коммит (Commit)
- Снимок состояния файлов в определенный момент времени
- Каждый коммит имеет уникальный хеш (например, `a1b2c3d`)

### Ветка (Branch)
- Отдельная линия разработки
- По умолчанию создается ветка `main` или `master`
- Позволяет работать над разными функциями независимо

### Pull Request (PR)
- Запрос на объединение изменений из одной ветки в другую
- Основной инструмент для код-ревью и обсуждения изменений

## Базовый рабочий процесс
---
### 1. Создание нового репозитория

**На GitHub:**
1. Нажми "+" в правом верхнем углу → "New repository"
2. Укажи имя репозитория
3. Добавь описание (опционально)
4. Выбери публичный или приватный
5. Инициализируй с README файлом
6. Нажми "Create repository"

**Локально (если проект уже существует):**
```bash
# Перейди в папку проекта
cd your-project

# Инициализируй Git репозиторий
git init

# Добавь удаленный репозиторий
git remote add origin https://github.com/your-username/your-repo.git
```

### 2. Клонирование существующего репозитория
```bash
git clone https://github.com/username/repository-name.git
# или с SSH
git clone git@github.com:username/repository-name.git
```

### 3. Стандартный цикл работы

```bash
# 1. Проверь текущий статус
git status

# 2. Добавь изменения в staging area
git add .                          # все файлы
git add filename.go               # конкретный файл
git add folder/                   # папку

# 3. Создай коммит
git commit -m "Краткое описание изменений"

# 4. Получи последние изменения с сервера
git pull origin main

# 5. Отправь изменения на GitHub
git push origin main
```

## Эффективная работа с ветками
---
### Создание и переключение веток
```bash
# Создать новую ветку
git branch feature/new-authentication

# Переключиться на ветку
git checkout feature/new-authentication

# Создать и сразу переключиться (комбинированная команда)
git checkout -b feature/new-authentication

# Посмотреть все ветки
git branch -a
```

### Типичная структура веток
```
main (или master)          - стабильная версия
├── develop                - ветка разработки
├── feature/user-auth      - новая функция аутентификации
├── feature/payment-system - система оплаты
└── hotfix/critical-bug   - срочное исправление
```

## Решение распространенных проблем
---
### Конфликты при слиянии
```bash
# Если возник конфликт при git pull
git pull origin main

# Реши конфликты в файлах (они помечены <<<<<<<, =======, >>>>>>>)
# После решения:
git add .
git commit -m "Resolve merge conflicts"
git push origin your-branch
```

### Отмена изменений
```bash
# Отмена незакоммиченных изменений в файле
git checkout -- filename.go

# Отмена всех незакоммиченных изменений
git checkout -- .

# Отмена последнего коммита (локально)
git reset --soft HEAD~1

# Отмена коммита и удаление изменений
git reset --hard HEAD~1
```

### Работа с историей коммитов
```bash
# Посмотреть историю коммитов
git log --oneline

# Посмотреть изменения в коммите
git show commit-hash

# Изменить последний коммит (сообщение или добавить файлы)
git commit --amend
```

## Файлы конфигурации для Go проектов
---
### .gitignore для Go
```gitignore
# Бинарные файлы
*.exe
*.exe~
*.dll
*.so
*.dylib

# Файлы тестового покрытия
*.out
*.test

# Папка с зависимостями
vendor/

# IDE
.vscode/
.idea/

# Локальные конфиги
config.local.yaml
.env
*.local

# Логи
*.log
```

### Go модули и GitHub
```bash
# Инициализация модуля
go mod init github.com/your-username/your-repo

# Добавление зависимостей
go get github.com/gin-gonic/gin

# Обновление зависимостей
go mod tidy

# Версионирование (теги)
git tag v1.0.0
git push origin v1.0.0
```

## Полезные советы
---
### Коммиты
- **Делай маленькие коммиты** - одна логическая единица изменений
- **Пиши понятные сообщения** в настоящем времени:
  - Правильно - "Add user authentication middleware"
  - Неправильно - "Fixed stuff"
- **Используй conventional commits** (опционально):
  - `feat: add new payment method`
  - `fix: resolve memory leak`
  - `docs: update API documentation`

### Ветвление
- **Создавай feature-ветки** для новой функциональности
- **Названия веток** должны быть понятными:
  - `feature/user-profile`
  - `bugfix/login-error`
  - `hotfix/security-patch`
- **Часто синхронизируйся** с основной веткой

### Работа с GitHub
- **Часто делай push** - не оставляй код только на локальной машине
- **Используй Issues** для отслеживания задач

## Полезные команды для повседневной работы
---

```bash
# Временное сохранение изменений (когда нужно переключить ветку)
git stash
git stash pop

# Создание тега версии
git tag -a v1.0.1 -m "Release version 1.0.1"
git push origin v1.0.1
```

## Безопасность
---
- **Никогда не коммить** пароли, ключи API, токены
- Используй **environment variables** или **secrets** в GitHub
- Регулярно **обновляй зависимости** (`go mod tidy`)
- Используй **dependabot** для автоматического обновления

---

**[← Назад к Backend](1.%20Погружение%20в%20основы/Backend/)**
