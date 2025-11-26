---
layout: default
title: JOIN соединения
permalink: /interview-prep/databases-cache/postgresql/joins/
---

# Цель
---
Здесь ты разберёшься с типами JOIN — это одна из самых важных операций в SQL. Если ты не понимаешь JOIN хорошо, ты будешь писать медленные запросы и получать неправильные результаты.

# Материалы
---
Рекомендую практиковаться на реальных запросах.

[Документация по SELECT](https://postgrespro.ru/docs/postgresql/current/sql-select)


## Основной синтаксис
---

```sql
SELECT <колонки>
FROM таблица_1
[INNER | LEFT | RIGHT | FULL | CROSS] JOIN таблица_2
ON условие_соединения
WHERE фильтр;
```


## 1. INNER JOIN (внутреннее соединение)
---

**Определение:** возвращает только строки, где есть совпадение в ОБЕИХ таблицах.

```sql
SELECT u.id, u.name, o.order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

**Диаграмма:**

```
Таблица users:              Таблица orders:
┌─────────────┐            ┌──────────────────┐
│  id | name  │            │ user_id | сумма  │
├─────────────┤            ├──────────────────┤
│ 1  | Алиса  │            │  1      | 100    │
│ 2  | Боб    │ INNER JOIN │  1      | 200    │
│ 3  | Катя   │ ═════════> │  2      | 150    │
│ 4  | Дима   │            │  5      | 300    │
└─────────────┘            └──────────────────┘

Результат: только совпадающие (1, 2)
Катя и Дима не появятся — у них нет заказов
Заказ от user_id=5 не появится — нет такого пользователя
```


## 2. LEFT JOIN (левое внешнее соединение)
---

**Определение:** возвращает ВСЕ строки из левой таблицы + совпадающие из правой. Если совпадения нет, правые колонки будут NULL.

```sql
SELECT u.id, u.name, o.order_id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- Результат:
-- id | name  | order_id | amount
-- ---|-------|----------|--------
--  1 | Алиса | 100      | 100
--  1 | Алиса | 200      | 200
--  2 | Боб   | 150      | 150
--  3 | Катя  | NULL     | NULL     <- Нет заказов
--  4 | Дима  | NULL     | NULL     <- Нет заказов
```

**Практическое применение — найти пользователей БЕЗ заказов:**

```sql
SELECT u.id, u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.order_id IS NULL;

-- Результат: Катя и Дима
```


## 3. RIGHT JOIN (правое внешнее соединение)
---

**Определение:** это LEFT JOIN с поменянными местами таблицами.

```sql
SELECT u.id, u.name, o.order_id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```

Эквивалентно:

```sql
SELECT u.id, u.name, o.order_id
FROM orders o
LEFT JOIN users u ON u.id = o.user_id;
```

**На практике RIGHT JOIN почти не используется** — можно просто переставить таблицы и использовать LEFT.


## 4. FULL OUTER JOIN (полное внешнее соединение)
---

**Определение:** возвращает ВСЕ строки из ОБЕИХ таблиц. Если совпадения нет, недостающие колонки будут NULL.

```sql
SELECT u.id, u.name, o.order_id, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- Результат:
-- id   | name  | order_id | amount
-- -----|-------|----------|--------
--  1   | Алиса | 100      | 100
--  1   | Алиса | 200      | 200
--  2   | Боб   | 150      | 150
--  3   | Катя  | NULL     | NULL
--  4   | Дима  | NULL     | NULL
-- NULL | NULL  | 300      | 300     <- Заказ без пользователя
```

**Практическое применение — найти расхождения:**

```sql
SELECT u.id, u.name, o.order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id
WHERE u.id IS NULL OR o.order_id IS NULL;

-- Пользователи без заказов ИЛИ заказы без пользователей
```


## 5. CROSS JOIN (декартово произведение)
---

**Определение:** возвращает все возможные комбинации строк из обеих таблиц.

**Внимание:** если в users 1000 строк, а в categories 10, результат будет 10 000 строк!

```sql
SELECT u.name, c.name
FROM users u
CROSS JOIN categories c;

-- Если users: Алиса, Боб
-- И categories: Электроника, Книги, Одежда
-- Результат:
-- name  | name
-- ------|----------
-- Алиса | Электроника
-- Алиса | Книги
-- Алиса | Одежда
-- Боб   | Электроника
-- Боб   | Книги
-- Боб   | Одежда
```


## Множественные JOIN
---

Можно соединять больше чем две таблицы:

```sql
SELECT 
  u.name,
  o.order_id,
  oi.product_id,
  p.product_name
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id;
```


## Разница между WHERE и ON в JOIN
---

- **ON**: определяет условие соединения таблиц
- **WHERE**: фильтрует результат после соединения

```sql
-- НЕПРАВИЛЬНО: условие в WHERE исключит строки из LEFT JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed';  -- Это исключит пользователей БЕЗ заказов!

-- ПРАВИЛЬНО: условие в ON
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'completed';
-- Теперь пользователи без заказов останутся (с NULL в колонках orders)
```


## Производительность JOIN
---

```sql
-- ХОРОШО: используется индекс
SELECT * FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- ПЛОХО: функция на колонке — индекс не используется
SELECT * FROM users u
INNER JOIN orders o ON LOWER(u.email) = LOWER(o.user_email);
```


## Вопросы для самопроверки
---

1. Объясни разницу между INNER JOIN и LEFT JOIN.
2. Как найти пользователей БЕЗ заказов? Напиши запрос.
3. Когда использовать FULL OUTER JOIN?
4. Почему CROSS JOIN опасен?
5. В чём разница между WHERE и ON в JOIN?
6. Как соединять больше чем две таблицы?

---

**[← Назад к PostgreSQL](../)**