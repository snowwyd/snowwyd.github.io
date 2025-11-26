---
layout: default
title: "Распределённые паттерны: Locks, Rate Limiting, Leaderboards"
permalink: /interview-prep/databases-cache/redis/distributed-patterns/
---
# Цель
---
Redis не только для кэша. Это мощный инструмент для решения распределённых проблем. На собеседовании часто просят реализовать distributed lock или rate limiter. Вот как это делается.

# Материалы
---
## Паттерн 1: Distributed Lock (Распределённая блокировка)

### Проблема

Несколько сервисов нужно синхронизировать доступ к ресурсу:

```
Service1: Обновляю userprofile
Service2: Обновляю userprofile  ← race condition!

Кто последний — его версия выигрывает.
```

### Решение: Distributed Lock

```
Service1: LOCK user:123
         ├─ обновляю
         └─ UNLOCK user:123

Service2: ждёт пока Service1 отпустит lock
         ├─ получает lock
         └─ обновляю
```

### Реализация (простая):

```go
func UpdateUserWithLock(userID int64, newData User) error {
    lockKey := fmt.Sprintf("lock:user:%d", userID)
    lockValue := generateUUID()  // уникальный ID
    
    // Пытаемся получить lock
    ok, err := redis.SetNX(lockKey, lockValue, 10*time.Second)
    if !ok {
        return ErrLocked  // Кто-то уже обновляет
    }
    
    defer redis.Del(lockKey)  // Освобождаем lock
    
    // Критичная секция
    user := db.GetUser(userID)
    user.Update(newData)
    db.SaveUser(user)
    
    return nil
}
```

**Важно**: используем `SetNX` (set if not exists) с TTL. TTL предотвращает deadlocks если сервис упадёт.

### Проблема: Race condition при release

```
Service1:
├─ GET lock → "my-uuid"  ← это мой lock?
├─ ... проходит время ...
├─ TTL истекает и Service2 получает lock!
├─ SET lock новый-uuid (Service2 обновляет)
└─ DEL lock ← удаляем чужой lock! ❌
```

### Правильная реализация (с Lua):

```go
func ReleaseLocksafely(lockKey, lockValue string) error {
    script := `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    `
    
    result, err := redis.Eval(script, []string{lockKey}, lockValue)
    return err
}
```

Или используй Redis Streams с TTL:

```go
// Redis 6.2+ имеет MODULE RedisTimeSeries
// или используй Lua script как выше
```

## Паттерн 2: Rate Limiting
---
### Проблема

Нужно ограничить количество запросов от пользователя:

```
User может делать 100 запросов в час
После 100 → HTTP 429 Too Many Requests
```

### Решение 1: Counter с TTL

```go
func RateLimitSimple(userID string, limit int, window int) bool {
    key := fmt.Sprintf("rate_limit:%s", userID)
    
    count := redis.Incr(key)
    if count == 1 {
        redis.Expire(key, window)
    }
    
    return count <= limit
}
```

**Как работает**:
```
Запрос 1: INCR → 1 ≤ 100 ✓
Запрос 2: INCR → 2 ≤ 100 ✓
...
Запрос 100: INCR → 100 ≤ 100 ✓
Запрос 101: INCR → 101 > 100 ❌ заблокировано!
```

### Решение 2: Sliding Window (более точное)

```go
func RateLimitSlidingWindow(userID string, limit int, window int) bool {
    key := fmt.Sprintf("rate_limit:%s", userID)
    now := time.Now().Unix()
    
    // Удаляем старые запросы
    redis.ZRemRangeByScore(key, 0, now-int64(window))
    
    // Считаем текущие запросы
    count := redis.ZCard(key)
    
    if count >= limit {
        return false  // blocked
    }
    
    // Добавляем текущий запрос
    redis.ZAdd(key, float64(now), fmt.Sprintf("req:%d", now))
    redis.Expire(key, window+1)
    
    return true
}
```

**Плюсы**: более точный (не пропускает запросы)

### Решение 3: Token Bucket

```go
func RateLimitTokenBucket(userID string, refillRate int) bool {
    key := fmt.Sprintf("bucket:%s", userID)
    
    // Получаем текущие tokens и последний refill
    data := redis.HGetAll(key)
    tokens := parseInt(data["tokens"])
    lastRefill := parseInt(data["lastRefill"])
    
    now := time.Now().Unix()
    
    // Добавляем tokens за прошедшее время
    elapsedSeconds := now - lastRefill
    tokensToAdd := elapsedSeconds * int64(refillRate)
    tokens = min(tokens+int(tokensToAdd), BUCKET_SIZE)
    
    // Проверяем можем ли выполнить запрос
    if tokens >= 1 {
        tokens--
        redis.HSet(key, "tokens", tokens, "lastRefill", now)
        return true
    }
    
    return false
}
```

## Паттерн 3: Leaderboard (Рейтинг)
---
### Use-case

Game leaderboard, contest rankings, etc.

### Решение: Sorted Sets

```go
// Обновить score
func UpdateScore(playerID string, score int) {
    redis.ZAdd("leaderboard", float64(score), playerID)
}

// Топ-10
func GetTop10() []Player {
    results := redis.ZRevRange("leaderboard", 0, 9)
    // ZRevRange возвращает от высокого к низкому score
    return convertToPlayers(results)
}

// Позиция конкретного игрока
func GetRank(playerID string) int {
    // 0-indexed, поэтому +1
    rank := redis.ZRevRank("leaderboard", playerID)
    return rank + 1
}

// Игроки с score между X и Y
func GetPlayersInScoreRange(minScore, maxScore int) []Player {
    results := redis.ZRangeByScore("leaderboard", 
        float64(minScore), 
        float64(maxScore),
    )
    return convertToPlayers(results)
}
```

### Real-time обновления

Игрок зарабатывает очки:

```go
func OnPlayerScores(playerID string, points int) {
    // Обновить leaderboard
    redis.ZIncrBy("leaderboard", float64(points), playerID)
    
    // Отправить notification
    redis.Publish("leaderboard:updates", 
        fmt.Sprintf("%s scored %d points", playerID, points),
    )
}
```

### Производительность

```
Complexity:
ZADD:        O(log N)
ZREVRANGE:   O(log N + K) где K это количество элементов
ZREVRANK:    O(log N)
```

Можешь иметь миллион игроков, операции остаются быстрыми!

## Паттерн 4: Counting (Статистика)
---
### Проблема

Считать:
- Page views
- Likes
- Downloads
- Etc.

### Решение: Simple Counter

```redis
INCR page_views:home          # 1
INCR page_views:home          # 2
...
GET page_views:home           # 10000
```

### Более сложное: Daily Counters

```go
func IncrementDailyCounter(key string) {
    today := time.Now().Format("2006-01-02")
    dailyKey := fmt.Sprintf("%s:%s", key, today)
    
    redis.Incr(dailyKey)
    redis.Expire(dailyKey, 24*3600)  // Удалится через день
}

func GetTodayCount(key string) int {
    today := time.Now().Format("2006-01-02")
    dailyKey := fmt.Sprintf("%s:%s", key, today)
    
    count := redis.Get(dailyKey)
    return parseInt(count)
}

func GetWeeklyCount(key string) int {
    total := 0
    for i := 0; i < 7; i++ {
        date := time.Now().AddDate(0, 0, -i).Format("2006-01-02")
        dailyKey := fmt.Sprintf("%s:%s", key, date)
        count := redis.Get(dailyKey)
        total += parseInt(count)
    }
    return total
}
```

### HyperLogLog для unique counts

```go
func AddUniqueVisitor(userIP string, date string) {
    key := fmt.Sprintf("unique_visitors:%s", date)
    redis.PFAdd(key, userIP)  // Add to HyperLogLog
}

func GetUniqueVisitorCount(date string) int {
    key := fmt.Sprintf("unique_visitors:%s", date)
    return redis.PFCount(key)  // Approximate count
}
```

**Плюсы**: O(1) память для миллионов уникальных элементов!
**Минусы**: Результат приблизительный (~2% ошибка)

## Паттерн 5: Pub/Sub для Events
---

```go
// Publisher (когда событие случается)
func OnOrderShipped(orderID int64) {
    event := map[string]interface{}{
        "type": "order_shipped",
        "order_id": orderID,
        "timestamp": time.Now(),
    }
    
    jsonData := json.Marshal(event)
    redis.Publish("order:events", string(jsonData))
}

// Subscriber (слушает события)
func SubscribeToOrderEvents() {
    pubsub := redis.Subscribe("order:events")
    
    for msg := range pubsub.Channel() {
        var event map[string]interface{}
        json.Unmarshal([]byte(msg.Payload), &event)
        
        if event["type"] == "order_shipped" {
            notifyCustomer(event["order_id"])
        }
    }
}
```

## Паттерн 6: Cache Warming
---
Pre-populate кэш важными данными:

```go
func WarmCache() {
    // Загрузить популярные пользователей из БД
    users := db.GetTopUsers(1000)
    
    for _, user := range users {
        jsonData := json.Marshal(user)
        redis.Set(
            fmt.Sprintf("user:%d", user.ID),
            jsonData,
            24*3600,  // 1 day TTL
        )
    }
}
```

Запустить при startup или периодически в фоне.

## Паттерн 7: Session Storage
---
```go
func StoreSession(sessionID string, sessionData Session) {
    jsonData := json.Marshal(sessionData)
    redis.Set(
        fmt.Sprintf("session:%s", sessionID),
        jsonData,
        30*60,  // 30 minute TTL
    )
}

func GetSession(sessionID string) (Session, error) {
    jsonData := redis.Get(fmt.Sprintf("session:%s", sessionID))
    
    var session Session
    json.Unmarshal(jsonData, &session)
    
    return session, nil
}
```

## Real-world: Комбинирование паттернов
---
```go
// Rate limiting + leaderboard + cache

func ProcessGameScore(playerID string, score int) error {
    // 1. Rate limiting: макс 100 запросов в минуту
    if !RateLimitTokenBucket(fmt.Sprintf("player:%s", playerID), 100) {
        return ErrRateLimited
    }
    
    // 2. Обновить leaderboard
    redis.ZIncrBy("leaderboard", float64(score), playerID)
    
    // 3. Кэшировать top-10
    top10 := redis.ZRevRange("leaderboard", 0, 9, "WITHSCORES")
    redis.Set("cached:top10", top10, 5*60)  // 5 min cache
    
    // 4. Отправить event
    redis.Publish("game:events", fmt.Sprintf("player %s scored", playerID))
    
    return nil
}
```

## Best Practices
---
1. **Distributed locks**: Всегда используй TTL и Lua для safe release
2. **Rate limiting**: Выбери алгоритм базированный на use-case
3. **Leaderboards**: Sorted sets — идеальный выбор
4. **Counting**: Используй HyperLogLog для unique counts
5. **Комбинируй паттерны**: Lock + cache, rate limit + counter, etc.

## Следующие шаги
---
Теперь ты знаешь основные распределённые паттерны в Redis. Пора разобраться с дебугингом и мониторингом — как узнать что происходит когда что-то сломалось.

---

**[← Назад к Redis](../)**