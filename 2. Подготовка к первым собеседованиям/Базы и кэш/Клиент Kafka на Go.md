---
layout: default
title: Практика с segmentio/kafka-go
permalink: /interview-prep/databases-cache/kafka/go-client/
---

# Цель
---
Это момент, когда теория встречается с практикой. Вы уже знаете архитектуру Kafka, теперь давайте писать код на Go.

`segmentio/kafka-go` — это лучшая библиотека для Kafka на Go. Она использует стандартную библиотеку Go и концепции.

# Материалы
---
## Установка и первый производитель
---

```bash
go get github.com/segmentio/kafka-go
```

### Простой производитель

```go
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/segmentio/kafka-go"
)

func main() {
    // Создаем писатель (производитель)
    писатель := &kafka.Writer{
        Addr:     kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
        Topic:    "заказы",
        Acks:     kafka.RequireAll, // Ждем всех replicas (acks=all)
        Balancer: &kafka.LeastBytes{}, // Распределяем по размеру
    }
    defer писатель.Close()

    ctx := context.Background()

    // Отправляем сообщение
    err := писатель.WriteMessages(ctx, kafka.Message{
        Key:   []byte("пользователь-123"),
        Value: []byte(`{"заказ_id": "12345", "сумма": 99.99}`),
    })
    
    if err != nil {
        log.Fatal("write failed:", err)
    }
    
    fmt.Println("Сообщение успешно отправлено!")
}
```

**Параметры Writer**:
- `Addr`: Адреса брокеров (можно несколько)
- `Topic`: В какую тему писать
- `Acks`: Подтверждения (0, 1, all)
- `Balancer`: Как распределять (LeastBytes, Hash, и т.д.)
- `Compression`: Сжатие (snappy, lz4, zstd, gzip)

## Продвинутые возможности производителя
---

### Батчинг и асинхронная отправка

```go
писатель := &kafka.Writer{
    Addr:           kafka.TCP("localhost:9092"),
    Topic:          "заказы",
    BatchSize:      1000,           // Батчим 1000 сообщений
    BatchBytes:     1000000,        // или 1МБ, что первое
    WriteBackoffMin: 10 * time.Millisecond,
    WriteBackoffMax: 100 * time.Millisecond,
    MaxAttempts:    5,              // Попытки повтора
}

// Отправляем много сообщений
for i := 0; i < 10000; i++ {
    err := писатель.WriteMessages(ctx, kafka.Message{
        Key:   []byte(fmt.Sprintf("ключ-%d", i)),
        Value: []byte(fmt.Sprintf(`{"id": %d}`, i)),
    })
    if err != nil {
        log.Printf("Ошибка при отправке сообщения %d: %v", i, err)
    }
}
```

### Пользовательский распределитель

```go
// Используем пользовательский ключ раздела
идентификаторПользователя := "пользователь-123"
писатель := &kafka.Writer{
    Addr:  kafka.TCP("localhost:9092"),
    Topic: "события-пользователя",
}

// Все события одного пользователя в один раздел
for i := 0; i < 100; i++ {
    писатель.WriteMessages(ctx, kafka.Message{
        Key:   []byte(идентификаторПользователя), // Ключ определяет раздел
        Value: []byte(fmt.Sprintf(`{"событие": "действие-%d"}`, i)),
    })
}
```

## Первый потребитель
---

### Простой потребитель

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    "github.com/segmentio/kafka-go"
)

func main() {
    // Создаем читатель (потребитель)
    читатель := kafka.NewReader(kafka.ReaderConfig{
        Brokers:        []string{"localhost:9092", "localhost:9093", "localhost:9094"},
        Topic:          "заказы",
        GroupID:        "обработчики-заказов", // ID группы потребителей
        StartOffset:    kafka.LastOffset,   // Начать с новых сообщений
        CommitInterval: time.Second,        // Фиксируем каждую секунду
        MaxBytes:       10e6,               // 10МБ за раз
    })
    defer читатель.Close()

    ctx := context.Background()

    for {
        сообщение, ошибка := читатель.ReadMessage(ctx)
        if ошибка != nil {
            log.Fatal("read error:", ошибка)
        }

        fmt.Printf("Сообщение в смещении %d: %s = %s\n", сообщение.Offset, сообщение.Key, сообщение.Value)
    }
}
```

### Ручное фиксирование (production pattern)

```go
читатель := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"localhost:9092"},
    Topic:          "заказы",
    GroupID:        "обработчики-заказов",
    CommitInterval: 0, // ОТКЛЮЧИТЬ автофиксирование
})

ctx := context.Background()

for {
    сообщение, ошибка := читатель.ReadMessage(ctx)
    if ошибка != nil {
        break
    }

    // Обрабатываем сообщение
    if ошибка := обработатьЗаказ(сообщение); ошибка != nil {
        log.Printf("Ошибка обработки: %v", ошибка)
        continue // Не фиксируем если ошибка
    }

    // ТОЛЬКО если успешно обработано
    if ошибка := читатель.CommitMessages(ctx, сообщение); ошибка != nil {
        log.Printf("Ошибка фиксирования: %v", ошибка)
        // Может быть нужен повтор или выход
    }
}
```

## Группы потребителей и переназначение
---

### Простая группа потребителей

```go
// Потребитель 1
читатель1 := kafka.NewReader(kafka.ReaderConfig{
    Brokers: []string{"localhost:9092"},
    Topic:   "заказы",
    GroupID: "процессоры", // ОДИНАКОВЫЙ group ID!
})

// Потребитель 2 (в отдельном процессе/горутине)
читатель2 := kafka.NewReader(kafka.ReaderConfig{
    Brokers: []string{"localhost:9092"},
    Topic:   "заказы",
    GroupID: "процессоры", // ОДИНАКОВЫЙ group ID!
})

// Kafka АВТОМАТИЧЕСКИ распределит разделы между читатель1 и читатель2
```

Когда вы запустите обоих потребителей:
- Если 4 раздела: читатель1 получит раздел 0,1; читатель2 получит 2,3
- Если добавить читатель3: Переназначение! Перераспределение

## Управление смещениями
---

### Чтение с определенного смещения

```go
читатель := kafka.NewReader(kafka.ReaderConfig{
    Brokers:     []string{"localhost:9092"},
    Topic:       "заказы",
    GroupID:     "процессоры",
    StartOffset: 0, // Начать с начала (если группа новая)
})

// Или после запуска
читатель.SetOffset(1000) // Перейти на смещение 1000
```

### Мониторинг отставания

```go
отставание, ошибка := читатель.ReadLag(ctx) // Только для одного раздела при GroupID=null
if ошибка != nil {
    log.Fatal("lag error:", ошибка)
}
fmt.Printf("Отставание потребителя: %d\n", отставание)
```

## Обработка ошибок
---

```go
// Ошибки сети
ошибка := писатель.WriteMessages(ctx, сообщение)
if ошибка != nil {
    if ошибка == context.Canceled {
        log.Println("Контекст отменен")
    } else if ошибка == context.DeadlineExceeded {
        log.Println("Таймаут")
    } else {
        log.Printf("Ошибка записи: %v", ошибка)
    }
}

// Ошибки читателя
сообщение, ошибка := читатель.ReadMessage(ctx)
if ошибка != nil {
    if ошибка == io.EOF {
        log.Println("Конец темы")
    } else {
        log.Printf("Ошибка чтения: %v", ошибка)
    }
}
```

## Production Checklist
---

- ✅ Используйте `acks=all` для критичных данных
- ✅ Реализуйте ручное фиксирование с обработкой ошибок
- ✅ Мониторьте отставание потребителя
- ✅ Используйте context для таймаутов
- ✅ Обработайте все возможные ошибки
- ✅ Корректное выключение: `defer читатель.Close()`
- ✅ Батчинг и сжатие для пропускной способности
- ✅ Логируйте все сбои для отладки

---

**[← Назад к Kafka](../)**