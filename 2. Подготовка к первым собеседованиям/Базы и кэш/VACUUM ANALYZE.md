---
layout: default
title: VACUUM и ANALYZE
permalink: /interview-prep/databases-cache/postgresql/vacuum-analyze/
---

# Цель
---
Здесь ты разберёшься с VACUUM и ANALYZE — критичными операциями обслуживания PostgreSQL. VACUUM удаляет мёртвые версии строк, ANALYZE обновляет статистику. Без них база деградирует и замедляется.

### Материалы
---
Рекомендую прочитать документацию по обслуживанию БД.

[Документация по VACUUM](https://postgrespro.ru/docs/postgresql/current/sql-vacuum)


## VACUUM: удаление мёртвых строк
---

Помни MVCC? Когда ты выполняешь UPDATE, PostgreSQL не перезаписывает строку, а создаёт новую версию. Старая версия помечается как «мёртвая». VACUUM удаляет эти мёртвые версии.

**Что происходит без VACUUM:**

```
1. INSERT: строка создана (xmin=100, xmax=NULL)
2. UPDATE: новая версия (xmin=200), старая мёртвая
3. UPDATE: новая версия (xmin=300), предыдущая мёртвая
4. UPDATE: новая версия (xmin=400), предыдущая мёртвая

Таблица раздувается! На диске лежат 4 версии, нужна только одна.
```


## Команды VACUUM
---

```sql
-- Удалить мёртвые строки из таблицы
VACUUM users;

-- Удалить мёртвые строки из всех таблиц БД
VACUUM;

-- Удалить мёртвые строки и вернуть место операционной системе
VACUUM FULL users;  -- МЕДЛЕННО! Требует эксклюзивной блокировки
```

**VACUUM** выполняется онлайн (не блокирует чтение):

```sql
-- Это работает во время VACUUM
SELECT * FROM users;          -- Работает
INSERT INTO users VALUES (...);  -- Работает
UPDATE users SET ...;         -- Работает
```

**VACUUM FULL** требует эксклюзивной блокировки:

```sql
-- ОЧЕНЬ МЕДЛЕННО! Таблица полностью заблокирована
VACUUM FULL users;  -- Используй только в окне обслуживания
```


## Autovacuum
---

PostgreSQL имеет **autovacuum** — фоновый процесс, который автоматически запускает VACUUM.

```sql
-- Посмотреть настройки autovacuum
SHOW autovacuum;                         -- Включён ли
SHOW autovacuum_vacuum_threshold;        -- Порог для триггера
SHOW autovacuum_vacuum_scale_factor;     -- Процент таблицы для триггера
```

Для таблиц с высокой нагрузкой можно настроить более агрессивный autovacuum:

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,   -- 5% таблицы
  autovacuum_analyze_scale_factor = 0.02   -- 2% таблицы
);
```


## ANALYZE: обновление статистики
---

PostgreSQL использует **оптимизатор** для выбора плана выполнения запроса. Оптимизатор полагается на **статистику** о таблицах.

**ANALYZE** сканирует таблицу и собирает статистику:
- Количество строк
- Уникальные значения в каждой колонке
- Распределение значений

**Проблема без ANALYZE:**

```sql
-- БЕЗ статистики оптимизатор гадает:
EXPLAIN SELECT * FROM users WHERE age > 30;
-- Seq Scan (cost=0.00..35000.00 rows=500000)  <- неправильная оценка!

-- Выполним ANALYZE
ANALYZE users;

-- Теперь оптимизатор знает правду:
EXPLAIN SELECT * FROM users WHERE age > 30;
-- Index Scan (cost=0.29..8.30 rows=2000)  <- правильная оценка!
```


## Команды ANALYZE
---

```sql
-- Обновить статистику для таблицы
ANALYZE users;

-- Обновить статистику для колонки
ANALYZE users(age);

-- Обновить статистику для всех таблиц БД
ANALYZE;

-- С подробным выводом
ANALYZE VERBOSE users;
```


## VACUUM ANALYZE: комбо
---

```sql
-- Выполнить оба одновременно (рекомендуется)
VACUUM ANALYZE users;
```


## Мониторинг статистики
---

```sql
-- Посмотреть статистику для таблицы
SELECT 
  schemaname,
  tablename,
  n_dead_tup,        -- Количество мёртвых строк
  n_live_tup,        -- Количество живых строк
  last_vacuum,       -- Когда был последний VACUUM
  last_autovacuum    -- Когда был последний автоматический VACUUM
FROM pg_stat_user_tables
WHERE tablename = 'users';

-- Таблицы с большим bloat
SELECT 
  tablename,
  ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as bloat_pct,
  n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY bloat_pct DESC;

-- Если bloat > 20%, пора делать VACUUM FULL
```


## Практические рекомендации
---

**Для production-систем:**

```sql
-- Агрессивный autovacuum для нагруженных таблиц
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_cost_delay = 2
);
```

**После массовых операций:**

```sql
-- После массового DELETE или UPDATE
DELETE FROM users WHERE status = 'inactive';  -- Удалили миллион строк

-- Освобождаем место
VACUUM FULL users;
ANALYZE users;
REINDEX TABLE users;  -- Пересоздать индексы
```


## Специальные операции обслуживания
---

**REINDEX** — пересоздаёт индексы:

```sql
REINDEX TABLE users;
REINDEX INDEX users_email_idx;
```

**CLUSTER** — физически переорганизует таблицу по индексу:

```sql
CLUSTER users USING users_pkey;
-- Удаляет bloat, улучшает локальность данных
-- Но медленно и требует эксклюзивной блокировки
```


## Вопросы для самопроверки
---

1. Что происходит без VACUUM? Почему таблица раздувается?
2. В чём разница между VACUUM и VACUUM FULL?
3. Когда и зачем выполнять ANALYZE?
4. Как работает autovacuum? Можно ли его отключить?
5. Как посмотреть количество мёртвых строк в таблице?
6. Что означает bloat в БД и как его измерить?

---

**[← Назад к PostgreSQL](../)**