---
layout: default
title: EXPLAIN и EXPLAIN ANALYZE
permalink: /interview-prep/databases-cache/postgresql/explain-analyze/
---

# Цель
---
Здесь ты научишься читать планы выполнения запросов — это главный инструмент для понимания того, почему запрос медленный и как его оптимизировать.

# Материалы
---
Рекомендую прочитать документацию по EXPLAIN.

[Документация по EXPLAIN](https://postgrespro.ru/docs/postgresql/current/using-explain)


## Команда EXPLAIN
---

EXPLAIN показывает план выполнения запроса — какие операции выполняются, в каком порядке, и какова их стоимость.

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;

-- Результат:
-- QUERY PLAN
-- ────────────────────────────────────────────────
-- Seq Scan on users  (cost=0.00..35.50 rows=1 width=244)
--   Filter: (id = 1)
```

**Что означает вывод:**

- **Seq Scan** — последовательное сканирование таблицы (читаем все строки)
- **cost=0.00..35.50** — оценка стоимости: от начала до конца
- **rows=1** — оценка, сколько строк вернёт этот узел
- **width=244** — средний размер одной строки в байтах
- **Filter: (id = 1)** — условие фильтрации


## EXPLAIN ANALYZE: выполнение и реальные цифры
---

EXPLAIN ANALYZE **выполняет запрос** и показывает реальные времена и количество строк.

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1;

-- Результат:
-- QUERY PLAN
-- ────────────────────────────────────────────────────────────
-- Seq Scan on users  (cost=0.00..35.50 rows=1 width=244) 
--                    (actual time=0.123..0.456 rows=1 loops=1)
--   Filter: (id = 1)
-- Planning Time: 0.102 ms
-- Execution Time: 0.567 ms
```

**Новые колонки:**

- **actual time=0.123..0.456** — реальное время выполнения
- **rows=1** — реальное количество строк
- **loops=1** — сколько раз выполнялся этот узел
- **Planning Time** — время на планирование запроса
- **Execution Time** — реальное время выполнения


## Основные типы узлов плана
---

**1. Seq Scan (последовательное сканирование)**

Читает ВСЕ строки таблицы последовательно. Медленно для больших таблиц.

```sql
EXPLAIN SELECT * FROM users;
-- Seq Scan on users  (cost=0.00..35.50 rows=1000 width=244)
```

**2. Index Scan (сканирование индекса)**

Использует индекс для быстрого поиска. Хорошо!

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;
-- Index Scan using users_id_idx on users  
--   (cost=0.29..8.30 rows=1 width=244)
--   Index Cond: (id = 1)
```

**3. Bitmap Scan (растровое сканирование)**

Комбинирует несколько индексов или эффективен при большом диапазоне.

```sql
EXPLAIN SELECT * FROM users WHERE id > 100 AND id < 1000;
-- Bitmap Index Scan on users_id_idx
-- Bitmap Heap Scan on users
```

**4. Типы соединений (JOIN)**

- **Nested Loop** — для маленьких соединений
- **Hash Join** — быстро, требует памяти
- **Merge Join** — для больших отсортированных данных


## EXPLAIN с дополнительными опциями
---

```sql
-- Показать максимум информации
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) 
SELECT * FROM users WHERE id = 1;

-- Результат:
-- Index Scan using users_pkey on users  
--   (cost=0.29..8.30 rows=1 width=244) 
--   (actual time=0.123..0.234 rows=1 loops=1)
--   Output: id, name, email, created_at
--   Index Cond: (id = 1)
--   Buffers: shared hit=3
-- Planning Time: 0.102 ms
-- Execution Time: 0.345 ms
```

**Buffers:**
- **shared hit** — страницы, прочитанные из кеша (хорошо!)
- **shared read** — страницы, прочитанные с диска (медленнее)


## Практические примеры
---

**Пример 1: Обнаружение отсутствующего индекса**

```sql
-- Медленный запрос
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
-- Seq Scan on users  (actual time=0.234..5.456 rows=1)  -- МЕДЛЕННО!

-- Решение: создать индекс
CREATE INDEX users_email_idx ON users(email);

-- Снова проверяем
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
-- Index Scan using users_email_idx  (actual time=0.012..0.023 rows=1)  -- БЫСТРО!
```

**Пример 2: Неправильная оценка строк**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';
-- Seq Scan on users  
--   (cost=0.00..35.50 rows=500 width=244)   -- Оценка: 500
--   (actual time=0.234..5.456 rows=50000)   -- Реальность: 50000!

-- Решение: обновить статистику
ANALYZE users;

-- Теперь оценка будет ближе к реальности
```


## Советы для оптимизации
---

1. **Seq Scan — плохо, Index Scan — хорошо** (обычно)
2. **Если actual rows сильно отличается от rows** — нужен ANALYZE
3. **Если Buffers: shared read большой** — нужен больший shared_buffers
4. **Если Execution Time растёт экспоненциально** — возможно N+1 проблема


## Вопросы для самопроверки
---

1. В чём разница между EXPLAIN и EXPLAIN ANALYZE?
2. Что значит cost=0.00..35.50 в выводе EXPLAIN?
3. Какой узел плана хуже: Seq Scan или Index Scan? Почему?
4. Что означает, если actual rows сильно отличается от оценки?
5. Как улучшить неправильные оценки в EXPLAIN?
6. Какие три типа JOIN-ов использует PostgreSQL?

---

**[← Назад к PostgreSQL](../)**