---
layout: default
title: Транзакции и атомарность в Redis
permalink: /interview-prep/databases-cache/redis/transactions/
---

# Цель
---
Понять как Redis обеспечивает atomicity операций и как использовать транзакции. Это критично потому что на собеседовании точно спросят: "Как обновить два ключа так чтобы оба обновились или ни один?" На первый взгляд кажется просто, но есть множество подводных камней.

# Материалы
---
## Базовый факт о Redis
---
Redis — это **single-threaded**, то есть:

- **Одна команда** выполняется атомарно
- Если ты выполняешь `INCR counter`, ни одна другая команда не интерферирует
- Но если тебе нужна atomicity **нескольких команд**, одна команда уже не достаточно

```
Проблема:
Thread 1: GET x          # получил 10
Thread 2: GET x          # тоже получил 10
Thread 1: SET x (10+1)   # устанавливает 11
Thread 2: SET x (10+1)   # устанавливает 11
Result: x = 11 (должно быть 12!)
```

Решение: MULTI/EXEC транзакции.

## MULTI/EXEC: Базовые транзакции
---
### Как работает:

```redis
MULTI              # Начать транзакцию
SET key1 "value1"  # не выполняется, добавляется в очередь
SET key2 "value2"  # не выполняется, добавляется в очередь
INCR counter       # не выполняется, добавляется в очередь
EXEC               # ВСЕ команды выполняются АТОМАРНО, в порядке очереди
```

### Практический пример:

```redis
# Перевод денег от user1 к user2
MULTI
DECRBY user:1:balance 100   # снять с user1
INCRBY user:2:balance 100   # добавить user2
EXEC
```

Если Redis упадёт между MULTI и EXEC, ничего из этого не выполнится. Обе операции либо выполнятся либо нет.

### Код на Go:

```go
func TransferMoney(from, to int64, amount int) error {
    // Начинаем транзакцию
    pipe := redis.Pipeline()
    
    pipe.DecrBy(fmt.Sprintf("user:%d:balance", from), amount)
    pipe.IncrBy(fmt.Sprintf("user:%d:balance", to), amount)
    
    // Выполняем все команды атомарно
    results, err := pipe.Exec()
    if err != nil {
        return err
    }
    
    return nil
}
```

### Плюсы:

- ✅ Простой API
- ✅ Все команды выполняются атомарно
- ✅ Быстро

### Минусы:

- ❌ **Нельзя проверить условие перед выполнением**
  
  ```redis
  MULTI
  GET balance         # Получили значение
  DECRBY balance 100  # Но что если баланс был изменён?
  EXEC                # Мы этого не узнаем!
  ```

- ❌ **Нет откатов (rollback)**. Если команда ошибётся, остальные всё равно выполнятся

## WATCH: Optimistic Locking
---
Если тебе нужно проверить условие перед транзакцией, используй WATCH.

### Как работает:

```
1. WATCH key         # Начинаем следить за ключом
2. Читаем его значение в приложении
3. MULTI             # Начинаем транзакцию
4. Выполняем команды
5. EXEC              # Если key был изменён другим клиентом - отмена!
```

### Практический пример:

```
User1: Читает баланс = 100

User2: Изменяет баланс на 50 (теперь = 150)

User1: Пытается снять 60 (думает что баланс 100)
       WATCH balance
       MULTI
       DECRBY balance 60
       EXEC           # ❌ EXEC возвращает nil (transaction aborted)
                      # потому что balance был изменён User2
```

### Код:

```go
func DecrementWithCondition(key string, amount int, maxRetries int) error {
    for attempt := 0; attempt < maxRetries; attempt++ {
        // Шаг 1: Watch ключ
        redis.Watch(key)
        
        // Шаг 2: Прочитать текущее значение
        val := redis.Get(key)
        intVal, _ := strconv.Atoi(val)
        
        // Шаг 3: Проверить условие
        if intVal < amount {
            redis.Unwatch()
            return ErrInsufficientBalance
        }
        
        // Шаг 4: Попытаться обновить
        pipe := redis.Pipeline()
        pipe.Watch(key)
        pipe.Decrby(key, amount)
        
        results, err := pipe.Exec()
        
        if err == nil && len(results) > 0 {
            // ✓ Успех!
            return nil
        }
        
        // ❌ Ключ был изменён другим клиентом, retry
    }
    
    return ErrMaxRetriesExceeded
}
```

### Как это работает в деталях:

1. `WATCH balance`: Redis следит за ключом
2. Если другой клиент изменит `balance`, метка "watched" устанавливается в true
3. Когда мы выполним `EXEC`, Redis проверит все watched ключи
4. Если хоть один был изменён — `EXEC` возвращает `nil` и ничего не выполняется

### DISCARD: Отмена транзакции

```redis
MULTI
SET key1 "value1"
SET key2 "value2"
DISCARD             # Очистить очередь, ничего не выполняется
```

### Плюсы WATCH:

- ✅ Оптимистичный лок (хороший для read-heavy)
- ✅ Нет блокировки, не блокируешь других клиентов
- ✅ Можешь проверить условие перед выполнением

### Минусы WATCH:

- ❌ Если часто бывают конфликты, много retry'ев
- ❌ Нужна логика для обработки nil результата

## Типичные ошибки с транзакциями
---
### ❌ Ошибка 1: Думать что Redis откатывает на ошибку

```redis
MULTI
SET key "value"
INCR not_a_number      # ошибка! не число
EXEC
```

Redis выполнит обе команды! `SET` успешно, `INCR` ошибку, но обе будут в результате.

Правильно обработать:

```go
results, err := pipe.Exec()
for _, result := range results {
    if err := result.Err(); err != nil {
        // Обработать ошибку
    }
}
```

### ❌ Ошибка 2: Dual Write Problem

```go
// ❌ Плохо!
redis.Set("user:1", userData)
db.UpdateUser(userData)      // Если упадёт - рассинхрон

// ✅ Хорошо! (если speed позволяет)
tx := db.Begin()
tx.UpdateUser(userData)
tx.Commit()
redis.Del("user:1")          // invalidate cache
```

### ❌ Ошибка 3: Не использовать WATCH когда нужно

```redis
# ❌ Плохо - что если balance изменился?
balance = GET balance
MULTI
DECRBY balance 100
EXEC

# ✅ Хорошо
WATCH balance
balance = GET balance
MULTI
DECRBY balance 100
EXEC
```

## Когда использовать что
---

| Ситуация | Что использовать |
|----------|------------------|
| Просто несколько команд подряд | MULTI/EXEC |
| Нужно проверить значение перед обновлением | MULTI/EXEC + WATCH |
| Очень частые конфликты | Lua Script (EVAL) |
| Сложная логика с условиями | Lua Script |

## Сравнение с Lua Scripting
---
Redis также поддерживает Lua скрипты для더 сложных операций:

```lua
-- script.lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('SET', KEYS[1], ARGV[2])
else
    return nil
end
```

```redis
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('SET', KEYS[1], ARGV[2]) else return nil end" 1 key1 "oldvalue" "newvalue"
```

**Плюсы vs WATCH:**
- ✅ Более сложная логика
- ✅ Не нужны retry'и
- ✅ Дешевле сетевых обращений

**Минусы:**
- ❌ Сложнее писать и дебугить
- ❌ Медленнее чем simple commands

## Real-world: Rate Limiter через WATCH
---

```go
func RateLimitUser(userID string, limit int, window int) bool {
    key := fmt.Sprintf("rate_limit:%s", userID)
    
    redis.Watch(key)
    count := redis.Get(key)
    intCount, _ := strconv.Atoi(count)
    
    if intCount >= limit {
        redis.Unwatch()
        return false
    }
    
    pipe := redis.Pipeline()
    pipe.Watch(key)
    pipe.Incr(key)
    pipe.Expire(key, window)
    
    results, err := pipe.Exec()
    
    if err == nil && len(results) > 0 {
        return true
    }
    
    return false  // Retry логика
}
```

## Практические советы:
---
1. **Используй WATCH для read-check-then-update patterns**
2. **Обрабатывай nil результат от EXEC (retry)**
3. **Не злоупотребляй транзакциями — часто одна команда достаточно**
4. **Для критичных операций рассмотри Lua**
5. **Мониторь количество абортов транзакций**

## Следующие шаги
---
Теперь когда ты понимаешь как гарантировать atomicity, пора разобраться с Pub/Sub для real-time коммуникации между сервисами.

---

**[← Назад к Redis](../)**