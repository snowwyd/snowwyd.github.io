---
layout: default
title: Структуры данных Redis
permalink: /interview-prep/databases-cache/redis/data-structures/
---

# Цель
---
Понять все основные структуры данных в Redis, как они работают внутри, и главное — **когда использовать какую**. На собеседовании постоянно будут спрашивать: "Как бы ты реализовал X с помощью Redis?" Правильный ответ зависит от выбора структуры данных.

# Материалы
---
## Strings (Строки)
---
Самый базовый тип, но обманчиво мощный.

### Что это:

Просто последовательность байтов. Binary-safe, означает что может содержать:
- Текст: "Hello, World!"
- Числа: "42", "3.14"
- JSON: "{\"user_id\": 123}"
- Сериализованные объекты: Protocol Buffers, msgpack, и т.д.

### Команды:

```redis
SET key value                    # установить
GET key                          # получить
GETRANGE key 0 5                 # подстрока (символы 0-5)
SETRANGE key 5 "NEW"             # перезаписать часть
APPEND key "suffix"              # добавить в конец
STRLEN key                       # длина
```

### Арифметика (если значение — число):

```redis
SET counter 0
INCR counter                     # = 1 (атомарно!)
INCRBY counter 10                # = 11
DECR counter                     # = 10
DECRBY counter 5                 # = 5
INCRBYFLOAT counter 2.5          # = 7.5
```

**Важно**: INCR — это **атомарна операция**. Даже если 1000 клиентов одновременно вызовут `INCR counter`, результат будет правильный. Это в основе работает потому что Redis single-threaded.

### Use-cases:

- **Кэш результатов**: `SET user:1000:profile "{...json...}"` с EXPIRE
- **Счётчики**: page views, likes, follows (через INCR)
- **Сессии**: `SET session:abc123 "{...session_data...}" EX 3600`
- **Временные данные**: OTP коды, verification tokens

### Complexity:

| Операция | Complexity |
|----------|-----------|
| SET, GET | O(1) |
| APPEND | O(1) |
| GETRANGE | O(N) — где N это длина подстроки |
| STRLEN | O(1) |

## Lists (Списки)
---
Упорядоченная коллекция строк, реализованная как **linked list** (не array).

### Что это:

```
Head → [Value1] → [Value2] → [Value3] → Tail
```

Это означает:
- Быстрый доступ к началу и концу: O(1)
- Медленный доступ в середину: O(N)

### Команды:

```redis
# Добавление элементов
LPUSH list "first"               # в начало
RPUSH list "last"                # в конец
RPUSH list "a" "b" "c"           # много элементов за раз

# Удаление элементов
LPOP list                        # удалить с начала
RPOP list                        # удалить с конца

# Доступ к элементам
LLEN list                        # длина списка
LINDEX list 0                    # получить элемент по индексу
LRANGE list 0 -1                 # диапазон (0 to end)

# Обновление
LSET list 0 "new_value"          # изменить элемент по индексу

# Блокирующие операции
BLPOP list 5                     # ждать элемента 5 сек (для очередей)
BRPOP list 5                     # ждать с конца
```

### Важные операции для очередей:

```redis
# Очередь задач
RPUSH jobs:pending "task:1"
RPUSH jobs:pending "task:2"
LPOP jobs:pending                # Worker берёт задачу
```

### Use-cases:

- **Очереди задач**: worker pool pattern
- **Activity streams**: список последних действий юзера
- **Message queues**: простые очереди (для более сложных — используй Streams)
- **Recentlist**: последних N элементов

### Complexity:

| Операция | Complexity |
|----------|-----------|
| LPUSH, RPUSH, LPOP, RPOP | O(1) |
| LINDEX | O(N) |
| LRANGE | O(N) |
| LLEN | O(1) |

### ⚠️ Осторожно:

`LINDEX` с большим индексом это медленно. Если у тебя список из 1 миллиона элементов и ты хочешь элемент 999999 — это O(N).

## Sets (Множества)
---
Неупорядоченная коллекция **уникальных** строк.

### Что это:

```
Set = {value1, value2, value3}
```

Внутри реализовано через hash table, поэтому:
- Проверка наличия элемента: O(1)
- Добавление/удаление: O(1)
- Порядок не гарантирован

### Команды:

```redis
# Добавление/удаление
SADD set "member1" "member2"     # добавить элементы
SREM set "member1"               # удалить элемент
SPOP set                         # удалить случайный элемент

# Проверка и получение
SISMEMBER set "member1"          # есть ли элемент?
SMEMBERS set                     # все элементы (не гарантирован порядок!)
SCARD set                        # количество элементов

# Set операции (очень полезные!)
SINTER set1 set2                 # пересечение
SUNION set1 set2                 # объединение
SDIFF set1 set2                  # разность
SINTERSTORE dest set1 set2       # результат в dest
```

### Set операции — примеры:

```redis
# Друзья
SADD friends:user1 "alice" "bob" "charlie"
SADD friends:user2 "bob" "charlie" "david"

# Кто общие друзья?
SINTER friends:user1 friends:user2        # ["bob", "charlie"]

# Все друзья вместе
SUNION friends:user1 friends:user2        # ["alice", "bob", "charlie", "david"]

# Друзья user1 но не user2
SDIFF friends:user1 friends:user2         # ["alice"]
```

### Use-cases:

- **Теги**: `SADD post:1:tags "redis" "database"`
- **Дружба**: `SADD friends:user1 "user2" "user3"`
- **Уникальные посетители**: `SADD unique_visitors:2025-01-20 "ip1" "ip2"`
- **Чёрные списки**: `SADD blocked_users "user123"`

### Complexity:

| Операция | Complexity |
|----------|-----------|
| SADD, SREM, SISMEMBER | O(1) |
| SMEMBERS | O(N) |
| SINTER, SUNION, SDIFF | O(N*M) где N, M это размеры сетов |

## Hashes (Хеши)
---
Коллекция пар field-value. Похожий на Python dict или JSON object.

### Что это:

```
Hash = {
  field1: value1,
  field2: value2,
  field3: value3
}
```

Внутри это nested hash table.

### Команды:

```redis
# Установка и получение
HSET user:1000 name "Ivan" age 25 email "ivan@example.com"
HGET user:1000 name              # "Ivan"
HMGET user:1000 name age         # ["Ivan", 25]
HGETALL user:1000                # all fields and values

# Удаление
HDEL user:1000 email             # удалить field

# Проверка
HEXISTS user:1000 name           # есть ли field?
HLEN user:1000                   # количество fields

# Получение только keys или values
HKEYS user:1000                  # [name, age, email]
HVALS user:1000                  # [Ivan, 25, ivan@example.com]

# Итерация
HSCAN user:1000 0                # для больших хешей
```

### Арифметика в хешах:

```redis
HSET product:1 price 100
HINCRBY product:1 price 10       # price = 110
HINCRBYFLOAT product:1 price -5.5 # price = 104.5
```

### Use-cases:

- **User profiles**: `HSET user:1000 name "Ivan" email "ivan@..." age 25`
- **Product details**: `HSET product:123 price 99.99 stock 50 category "books"`
- **Конфигурация**: `HSET app:config redis_host "localhost" redis_port 6379`
- **Инкрементирующиеся метрики**: `HINCRBY stats:2025-01-20 page_views 1`

### Почему использовать Hash вместо отдельных Strings:

```redis
# ❌ Плохо (много ключей):
SET user:1000:name "Ivan"
SET user:1000:email "ivan@example.com"
SET user:1000:age 25

# ✅ Хорошо (один ключ):
HSET user:1000 name "Ivan" email "ivan@example.com" age 25
```

Hash экономит память и операции.

### Complexity:

| Операция | Complexity |
|----------|-----------|
| HSET, HGET, HDEL | O(1) |
| HGETALL | O(N) где N это количество fields |
| HSCAN | O(1) за итерацию |

## Sorted Sets (Упорядоченные множества)
---
Set где каждый элемент имеет **score** (число), и элементы упорядочены по score.

### Что это:

```
Sorted Set = {
  "alice": 100,
  "bob": 150,
  "charlie": 90
}

# Упорядочены по score:
charlie (90) < alice (100) < bob (150)
```

Внутри реализовано через skip list (особый тип сбалансированного дерева).

### Команды:

```redis
# Добавление элементов
ZADD leaderboard 100 "alice" 150 "bob" 90 "charlie"

# Получение
ZCARD leaderboard                # количество элементов
ZCOUNT leaderboard 90 150        # элементы со score 90-150
ZSCORE leaderboard "alice"       # score alice = 100
ZRANK leaderboard "alice"        # позиция (0-indexed)
ZREVRANK leaderboard "alice"     # позиция в обратном порядке

# Диапазоны
ZRANGE leaderboard 0 -1          # все элементы по возрастанию score
ZREVRANGE leaderboard 0 -1       # все элементы по убыванию
ZRANGE leaderboard 0 -1 WITHSCORES # с scores

# Обновление score
ZINCRBY leaderboard 10 "alice"   # alice score = 110

# Удаление
ZREM leaderboard "alice"
```

### Практический пример: Leaderboard

```redis
# Player выиграл 10 очков
ZINCRBY leaderboard 10 "player1"

# Топ-10 игроков (по убыванию score)
ZREVRANGE leaderboard 0 9 WITHSCORES

# Позиция конкретного игрока
ZREVRANK leaderboard "player1"

# Количество игроков между score 1000 и 5000
ZCOUNT leaderboard 1000 5000
```

### Use-cases:

- **Leaderboards**: games, contests
- **Time-series data**: события с timestamp'ами как score
- **Rate limiting**: отслеживание количества запросов по time window
- **Priority queues**: задачи с priority как score
- **Ranking systems**: рейтинг продуктов, статей

### Complexity:

| Операция | Complexity |
|----------|-----------|
| ZADD, ZREM, ZINCRBY | O(log N) |
| ZRANGE, ZREVRANGE | O(log N + M) где M это количество элементов в диапазоне |
| ZCARD | O(1) |
| ZSCORE | O(1) |

## Специальные структуры (продвинуто)
---
### Streams

Redis streams — это log-like структура данных для message queues.

```redis
XADD mystream "*" field1 "value1" field2 "value2"
XRANGE mystream - +            # прочитать все
XREAD COUNT 2 STREAMS mystream 0
```

Streams лучше чем Lists для очередей потому что:
- Consumer groups
- Automatic ID generation
- Persistence

Но это продвинутый топик. Для базовых очередей Lists достаточно.

### Bitmaps

Для хранения bit arrays (флаги):

```redis
SETBIT visited:2025-01-20 100 1  # пользователь 100 посещал сегодня?
GETBIT visited:2025-01-20 100    # 1 если посещал, 0 если нет
BITCOUNT visited:2025-01-20      # сколько посещали
```

Use-case: отслеживание активности пользователей, online/offline статусы.

### HyperLogLog

Для подсчёта уникальных элементов с приблизительностью:

```redis
PFADD unique_visitors "ip1" "ip2" "ip3"
PFCOUNT unique_visitors           # примерно сколько уникальных?
```

Use-case: считать уникальных посетителей БЕЗ хранения всех IP адресов в памяти.

## Выбор структуры данных: Decision Tree
---

```
Что нужно хранить?

├─ Одно значение (string, число, JSON)?
│  └─ Используй STRING

├─ Упорядоченный список элементов?
│  └─ Используй LIST (если нужен порядок вставки)

├─ Неупорядоченная коллекция уникальных элементов?
│  └─ Используй SET

├─ Пара field-value (как JSON)?
│  └─ Используй HASH

├─ Элементы с оценками/рейтингом?
│  └─ Используй SORTED SET

├─ Лог/очередь со множеством консьюмеров?
│  └─ Используй STREAM

├─ Просто флаги (0/1)?
│  └─ Используй BITMAP

├─ Подсчёт уникальных (примерно)?
│  └─ Используй HYPERLOGLOG
```

## Ошибки которые делают на собеседовании
---
### ❌ Ошибка 1: Использовать List для случайного доступа

```redis
# ❌ Плохо! LINDEX O(N)
LINDEX my_list 999999

# ✅ Хорошо. ZRANGE O(log N + M)
ZRANGE my_sorted_set 0 10
```

### ❌ Ошибка 2: Не использовать Set операции

```redis
# ❌ Плохо! Две команды + логика на клиенте
SMEMBERS set1
SMEMBERS set2
# потом в коде делаем пересечение

# ✅ Хорошо! Одна атомарная команда
SINTER set1 set2
```

### ❌ Ошибка 3: Использовать String вместо Hash

```redis
# ❌ Плохо! Много ключей
SET user:1:name "Alice"
SET user:1:email "alice@..."
SET user:1:age 25

# ✅ Хорошо!
HSET user:1 name "Alice" email "alice@..." age 25
```

## Следующие шаги
---
Теперь что ты знаешь все структуры данных, пора научиться использовать их в реальных паттернах кэширования. Правильный выбор структуры — это половина успеха. Вторая половина — это правильный паттерн.

---

**[← Назад к Redis](../)**