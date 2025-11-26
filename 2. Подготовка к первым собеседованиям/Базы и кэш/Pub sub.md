---
layout: default
title: "Pub/Sub: Real-time коммуникация"
permalink: /interview-prep/databases-cache/redis/pub-sub/
---
# Цель
---
Понять как Redis Pub/Sub работает и когда его использовать. На собеседовании часто спрашивают: "Как реализовать real-time notifications?" Redis Pub/Sub — частый ответ. Но есть подводные камни: сообщения могут потеряться, нет гарантий доставки, и это не подходит для очередей где критична надёжность.


# Материалы
---
## Базовая идея

Redis Pub/Sub — это **один-ко-многим коммуникация** без состояния.

```
Publisher                 Channel              Subscribers
   |                        |                      |
   ├──── PUBLISH message ──→ |                      |
   |                        ├──── message ────────→ Subscriber1
   |                        |                      |
   |                        ├──── message ────────→ Subscriber2
   |                        |                      |
   |                        └──── message ────────→ Subscriber3
```

**Ключные особенности:**

- 📤 **Publisher** отправляет сообщение в channel
- 👂 **Subscribers** слушают channel и получают сообщения
- 🔥 **Fire-and-forget**: если нет subscribers — сообщение теряется!
- 💨 **Decoupled**: publisher не знает про subscribers

## Базовые команды
---
### Subscribe (слушать)

```redis
# Подписаться на один channel
SUBSCRIBE news

# Подписаться на несколько channels
SUBSCRIBE news sports weather
```

После SUBSCRIBE клиент переходит в **subscriber mode** и может только:
- Получать сообщения
- Подписаться на больше channels
- Отписаться

Не может выполнять обычные команды типа SET, GET!

### Publish (отправить)

```redis
# Отправить сообщение в channel
PUBLISH news "Breaking: Redis is awesome!"

# Результат: (integer) 3
# Это количество subscribers которые получили сообщение
```

### Unsubscribe (перестать слушать)

```redis
UNSUBSCRIBE news    # отписаться от одного
UNSUBSCRIBE         # отписаться от всех
```

## Pattern Subscriptions
---
Можешь подписаться на паттерны с wildcards:

```redis
# Подписаться на все channels начинающиеся с "chat:"
PSUBSCRIBE chat:*

# Получит сообщения из chat:general, chat:random, chat:dev, etc.
```

```redis
# Похожие паттерны
PSUBSCRIBE user:*:notification    # user:123:notification, user:456:notification
PSUBSCRIBE news.*                 # news.sports, news.politics, news.tech

# Символы паттернов:
#  *  - любые символы
#  ?  - один символ
#  [] - набор символов (типа regex)
```

## Реальный пример: Chat приложение
---
### Subscriber (слушатель):

```go
func SubscribeToChat(roomID string) {
    channel := fmt.Sprintf("chat:%s", roomID)
    
    pubsub := redis.Subscribe(channel)
    
    for msg := range pubsub.Channel() {
        fmt.Printf("New message: %s\n", msg.Payload)
    }
}

// Запуск в отдельной goroutine
go SubscribeToChat("chat-room-1")
```

### Publisher (отправитель):

```go
func SendMessage(roomID, message string) {
    channel := fmt.Sprintf("chat:%s", roomID)
    
    count := redis.Publish(channel, message)
    if count == 0 {
        log.Warn("No one listening to", channel)
    }
}
```

### Использование:

```
User1: Подписывается на "chat:room1"
User2: Подписывается на "chat:room1"
User3: Публикует сообщение в "chat:room1"
↓
User1 и User2 получают сообщение
```

## Pub/Sub vs Message Queues
---
Это **очень важное** различие!

| Аспект | Pub/Sub | Message Queue (Streams/Lists) |
|--------|---------|-----|
| **Модель** | One-to-many broadcast | One-to-one delivery |
| **Доставка** | Fire-and-forget | Guaranteed |
| **Если нет consumers** | Сообщение теряется | Сообщение ждёт |
| **Использование** | Notifications, events | Task queues |
| **Пример** | Chat, notifications | Job processing |

### ❌ Не используй Pub/Sub для:

```
Критичных задач где сообщение не должно теряться!
```

Если worker упадёт при обработке задачи, сообщение потеряется:

```go
// ❌ Плохо для queues
redis.Publish("task:queue", taskData)

// ✅ Хорошо для queues
redis.RPush("task:queue", taskData)
```

## Когда использовать Pub/Sub
---
### ✅ Хорошие use-cases:

1. **Real-time notifications**
   ```
   User получает лайк → PUBLISH user:123:notifications "new_like"
   ```

2. **Live updates**
   ```
   Stock price изменилась → PUBLISH stock:AAPL:updates "250.50"
   ```

3. **Chat/messaging (если потеря сообщений OK)**
   ```
   User отправляет сообщение → PUBLISH chat:room1 "message"
   ```

4. **Event broadcasting**
   ```
   Server перезагружается → PUBLISH system:events "server_restart"
   ```

### ❌ Плохие use-cases:

- Очереди задач где важна гарантия доставки
- Финансовые операции
- Что-угодно где потеря сообщения критична

## Fire-and-Forget Проблема
---
**Самая важная особенность Pub/Sub:**

Если нет никого слушающего, сообщение **теряется**.

```
Scenario 1: Есть listeners
PUBLISH message → ✓ Все получат

Scenario 2: Нет listeners
PUBLISH message → 💨 Потеряется

Scenario 3: Listener подписывается ПОСЛЕ publish
Publisher: PUBLISH message
Subscriber: SUBSCRIBE (слишком поздно!)
Result: Subscriber не получит сообщение
```

Решение для критичного broadcasting:

```go
// Использовать Streams вместо Pub/Sub
redis.XAdd("notifications", "*", "user_id", userId, "type", "like")

// Или использовать separate cache для "последнего сообщения"
redis.Publish("notifications", message)
redis.Set("last_notification", message)  // backup
```

## Pattern Subscriptions: Практический пример
---

```
Система уведомлений для разных типов событий
```

### Subscriber отслеживает разные события:

```go
func SubscribeToNotifications() {
    pubsub := redis.PSubscribe(
        "notification:*:user",
        "notification:*:system",
    )
    
    for msg := range pubsub.Channel() {
        switch msg.Channel {
        case "notification:order:user":
            handleUserNotification(msg.Payload)
        case "notification:payment:system":
            handleSystemNotification(msg.Payload)
        }
    }
}
```

### Publisher отправляет разные события:

```go
// Пользовательское уведомление
redis.Publish("notification:order:user", "Your order shipped!")

// Системное уведомление
redis.Publish("notification:payment:system", "Payment received")
```

## PUBSUB команды (для мониторинга)
---
```redis
# Какие channels есть активные?
PUBSUB CHANNELS

# Какие channels активные с паттерном?
PUBSUB CHANNELS "chat:*"

# Сколько subscribers на channel?
PUBSUB NUMSUB chat:room1

# Сколько подписок на паттерны?
PUBSUB NUMPAT
```

## Limitations и когда это проблема
---
### Проблема 1: Потеря сообщений

```go
// Если Redis упадёт, все в памяти теряется
redis.Publish("notifications", message)
```

Решение: использовать Redis Streams или отдельное хранилище.

### Проблема 2: Нет истории

```redis
# Если subscriber подписался секунду назад, он не видит старые сообщения
SUBSCRIBE chat:room1
# Не получит сообщения что были отправлены в прошлом!
```

Решение: Redis Streams сохраняют историю!

### Проблема 3: Нет гарантии обработки

```
Publisher: PUBLISH message
Subscriber: Получил сообщение
            (процесс упал при обработке)
Result: Сообщение потеряется
```

Решение: для критичных задач используй message queues с acknowledgments.

## Real-world: Notification System
---

```go
// Publisher (когда случается событие)
func OnNewLike(postID, userID int64) {
    notification := map[string]interface{}{
        "type": "like",
        "post_id": postID,
        "user_id": userID,
        "timestamp": time.Now(),
    }
    
    jsonData := json.Marshal(notification)
    count := redis.Publish("notifications", jsonData)
    
    if count == 0 {
        log.Warn("No subscribers for notifications")
        // Fallback: сохранить в БД для уведомления позже
    }
}

// Subscriber (слушает notifications)
func StreamNotifications() {
    pubsub := redis.Subscribe("notifications")
    
    for msg := range pubsub.Channel() {
        var notification map[string]interface{}
        json.Unmarshal([]byte(msg.Payload), &notification)
        
        // Send to WebSocket или другой транспорт
        sendToClient(notification)
    }
}
```

## Сравнение: Pub/Sub vs Streams
---

| Фича | Pub/Sub | Streams |
|------|---------|---------|
| History | ❌ | ✅ |
| Guaranteed delivery | ❌ | ✅ |
| Consumer groups | ❌ | ✅ |
| Real-time | ✅ | ✅ |
| Persistence | ❌ | ✅ (configurable) |
| Use case | Notifications | Job queues |

**Давай выбирать:**

- Notifications, live updates → **Pub/Sub**
- Reliable message queues → **Streams**

## Best practices
---
1. **Не полагайся на Pub/Sub для критичных данных**
2. **Используй Streams если нужна история**
3. **Мониторь количество subscribers** (PUBSUB NUMSUB)
4. **Обрабатывай случай когда нет listeners**
5. **Для WebSocket интеграции — используй Pub/Sub как backbone**

## Следующие шаги
---
Теперь когда ты понимаешь real-time коммуникацию, пора разобраться с управлением памятью и eviction policies. Redis работает в памяти, и когда память закончится, нужна стратегия что удалять.

---

**[← Назад к Redis](../)**