---
layout: default
title: Управление памятью и Eviction Policies
permalink: /interview-prep/databases-cache/redis/memory/
---
# Цель
---
Redis работает в памяти, и память конечна. Когда Redis займёт всю доступную память, нужна стратегия что делать. Это не просто теория — неправильная конфигурация приведёт либо к crash'у, либо к потере важных данных.

# Материалы
---
## Базовая проблема

```
Redis память
┌─────────────────────────────┐
│ Key1: value1                │ 5 MB
│ Key2: value2                │ 5 MB
│ Key3: value3                │ 5 MB
│ ...                         │
│ Key1000: value1000          │ 5 MB
│ [ПАМЯТИ НЕТУ!]              │ ← Что делать??
└─────────────────────────────┘
Используется: 5 GB (из 5 GB доступных)
```

Есть несколько вариантов:
1. **Ошибка**: Вернуть "out of memory" error
2. **Удалить старые данные**: какие ключи удалить?

## Maxmemory: Лимит памяти
---
Сначала устанавливаем максимум:

```redis
CONFIG SET maxmemory 1gb              # 1 ГБ максимум
CONFIG GET maxmemory                  # проверить текущее значение
```

Или в конфиг файле:

```
# redis.conf
maxmemory 1gb
```

Когда Redis достигнет этого лимита, срабатывает eviction policy.

## Eviction Policies: Какие ключи удалять?
---
Redis имеет несколько встроенных стратегий:

### 1. No Eviction (Default)

```redis
CONFIG SET maxmemory-policy noeviction
```

**Что происходит**: Когда память полна, Redis **отказывает** в новых write операциях.

```redis
SET newkey value        # ❌ OOM command not allowed
```

**Когда использовать**: Критичные данные где потеря недопустима.

**Проблема**: Приложение сломается если будет писать много.

### 2. LRU Policies (Least Recently Used)
---
Удаляет ключи что давно не читались.

#### 2.1 allkeys-lru

```redis
CONFIG SET maxmemory-policy allkeys-lru
```

Удаляет **любой** ключ (с TTL или без) что используется реже всего.

**Пример**:
```
Key1: 1 день назад читал
Key2: 5 дней назад читал ← УДАЛИТЬ (самый старый)
Key3: 2 дня назад читал
```

**Когда использовать**: General purpose cache (самый частый выбор!)

#### 2.2 volatile-lru

```redis
CONFIG SET maxmemory-policy volatile-lru
```

Удаляет **только ключи с TTL** что используются реже всего.

**Пример**:
```
Key1: (no TTL) - не трогаем
Key2: (TTL 1000) читали 5 дней назад ← УДАЛИТЬ
Key3: (TTL 2000) читали 2 дня назад
```

**Когда использовать**: Когда хочешь сохранить ключи без TTL.

### 3. LFU Policies (Least Frequently Used)
---
Удаляет ключи что используются реже всего (по частоте).

#### 3.1 allkeys-lfu

```redis
CONFIG SET maxmemory-policy allkeys-lfu
```

Удаляет **любой** ключ что используется реже всего по частоте.

**Пример**:
```
Key1: читали 10 раз
Key2: читали 2 раза ← УДАЛИТЬ (самый редкий)
Key3: читали 100 раз
```

**Отличие от LRU**: LRU смотрит КОГДА читали, LFU смотрит КАК ЧАСТО читали.

#### 3.2 volatile-lfu

```redis
CONFIG SET maxmemory-policy volatile-lfu
```

Удаляет только ключи с TTL что используются реже всего по частоте.

### 4. TTL Policy
---
```redis
CONFIG SET maxmemory-policy volatile-ttl
```

Удаляет ключи **с TTL** которые скоро истекут.

**Пример**:
```
Key1: TTL 10 сек ← УДАЛИТЬ (скоро истечёт)
Key2: TTL 5000 сек
```

### 5. Random Policy
---
```redis
CONFIG SET maxmemory-policy allkeys-random      # любой
CONFIG SET maxmemory-policy volatile-random     # только с TTL
```

Удаляет случайный ключ. Обычно не используется, но может быть полезно если данные uniform.

## Как выбрать Policy
---

```
Какие данные в Redis?

├─ Cache (user profiles, API responses)?
│  └─ allkeys-lru или allkeys-lfu
│     (удаляем неиспользуемые)

├─ Mix: Sessions (с TTL) + Cache (без TTL)?
│  └─ volatile-lru
│     (sessions удаляются по TTL, cache вообще не трогаем)

├─ Sessions (все с TTL)?
│  └─ volatile-ttl
│     (удаляем сессии близкие к истечению)

├─ Critical данные (не должны потеряться)?
│  └─ noeviction
│     (ошибка если память полна)

├─ Критичные операции + cache?
│  └─ Раздели на разные instances!
└─  Critical: maxmemory-policy noeviction
    Cache: maxmemory-policy allkeys-lru
```

## Практический пример: Session + Cache
---

```go
// Sessions (с TTL, никогда не должны удаляться)
redis1 := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
})

// Cache (может быть удалён)
redis2 := redis.NewClient(&redis.Options{
    Addr: "localhost:6380",  // другой instance!
})
```

Конфиг для sessions:

```redis
# Instance 1 (sessions)
maxmemory 2gb
maxmemory-policy volatile-ttl    # удаляем старые sessions
```

Конфиг для cache:

```redis
# Instance 2 (cache)
maxmemory 4gb
maxmemory-policy allkeys-lru     # удаляем неиспользуемый cache
```

## Мониторинг памяти
---
### Проверить текущее использование:

```redis
INFO memory
```

Ключевые метрики:

```
used_memory: 100000000           # всего используется
used_memory_human: 95.37M
used_memory_rss: 120000000       # включая overhead
maxmemory: 1000000000            # лимит (1GB)
evicted_keys: 5000               # сколько ключей удалено
```

### Проверить hit rate кэша:

```go
// Считаем в приложении
hits := 0
misses := 0

for {
    val := redis.Get(key)
    if val != nil {
        hits++
    } else {
        misses++
    }
    
    rate := float64(hits) / float64(hits+misses)
    
    // Если <80% hit rate - проблема!
    if rate < 0.8 {
        log.Warn("Low cache hit rate:", rate)
    }
}
```

## Признаки проблем с памятью
---
### ⚠️ Признак 1: Много evictions

```redis
INFO stats
evicted_keys: 1000000  # слишком много удалено!
```

**Решение**: Увеличить maxmemory или оптимизировать что храним.

### ⚠️ Признак 2: Высокое used_memory_rss

```redis
used_memory: 100MB         # фактические данные
used_memory_rss: 500MB     # включая overhead
```

**Решение**: Фрагментация памяти. Перезагрузить Redis.

### ⚠️ Признак 3: Low cache hit rate

```
Cache hit rate: 40%  # очень низко!
```

**Решение**: Eviction слишком агрессивная или not enough memory.

## Когда всё ломается
---

### Scenario 1: maxmemory слишком маленький

```redis
maxmemory 100mb       # слишком мало
# Каждую секунду удаляются ключи
# Cache hit rate падает ниже 50%
```

**Fix**: Увеличить maxmemory или переложить данные на другой instance.

### Scenario 2: Неправильная policy

```redis
maxmemory-policy noeviction
# Redis упадёт когда память полна
# Приложение не сможет писать
```

**Fix**: Выбрать правильную policy для use case.

### Scenario 3: Out of memory на всех instances

```
Session Redis: Полна
Cache Redis: Полна
Application: Не может ни читать ни писать
```

**Fix**: Срочно добавить ещё памяти или сократить данные.

## Best Practices
---

1. **Мониторь memory usage** постоянно
2. **Раздели критичные и некритичные данные** на разные instances
3. **Используй allkeys-lru для cache**, volatile-lru для смеси
4. **Настрой monitoring alerts**:
   - Если used_memory > 80% maxmemory
   - Если evicted_keys растёт быстро
   - Если hit_rate < 80%
5. **Регулярно проверяй что в памяти** (KEYS * в production хорошо!):
   ```
   KEYS *            # сколько ключей?
   DBSIZE            # быстро посчитать
   ```

## Следующие шаги
---
Теперь когда ты понимаешь как Redis управляет памятью, пора разобраться с Lua scripting — самый мощный способ выполнять atomically сложные операции.

---

**[← Назад к Redis](../)**