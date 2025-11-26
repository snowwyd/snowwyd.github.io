---
layout: default
title: "Репликация: High Availability"
permalink: /interview-prep/databases-cache/redis/replica/
---

# Цель
---
Когда Redis instance упадёт — вся система рухнет. Репликация решает эту проблему: данные копируются на другие узлы (replicas), и если master упадёт, мы можем переключиться на replica.

# Материалы
---
## Базовая архитектура

```
Master Redis              Replica1 Redis           Replica2 Redis
┌──────────────┐         ┌──────────────┐        ┌──────────────┐
│ SET key val  │ ────────→ │ SET key val  │      │ SET key val  │
│ INCR counter │ ────────→ │ INCR counter │      │ INCR counter │
│              │ (async)   │              │      │              │
└──────────────┘         └──────────────┘        └──────────────┘
(Primary)                 (Read-only)             (Read-only)
```

**Что происходит**:
- Все WRITE операции идут в master
- Все READ операции могут идти в replicas
- Master отправляет updates в replicas (асинхронно)

## Как это работает
---
### 1. Replica подключается к Master

```redis
# На replica'е:
REPLICAOF master_ip master_port    # или SLAVEOF (старая команда)

# На master'е:
INFO replication
# Output:
# role: master
# connected_slaves: 1
# slave0: ip=..., state=online
```

### 2. Master отправляет данные Replica'е

```
Master:
├─ Полный снимок (если первое подключение)
├─ Потом отправляет все новые команды
└─ Replica'е просто replay'ит их
```

### 3. Replica читает команды и применяет их

```
Replica:
├─ SET key val      (из master)
├─ INCR counter     (из master)
└─ Синхронизирована!
```

## Практическая установка
---
### На Master (ничего не нужно менять):

```redis
redis-server
```

### На Replica:

```redis
redis-server --port 6380 --replicaof localhost 6379
```

Или в конфиг файле:

```
# replica.conf
port 6380
replicaof localhost 6379
```

## Асинхронная репликация
---

```
Master                        Replica
│                             │
SET key1 ─────────────────────→ устанавливается
SET key2 ─────────────────────→ устанавливается
│ (может быть lag!)
SET key3
├─ Клиент: получает OK
├─ Master: выполнил
└─ Replica: ещё не знает (lag ~1ms)

Если в момент lag master упадёт:
→ key3 потеряется на replica'е
```

**Следствие**: Asynchronous replication гарантирует HA но не гарантирует что все данные будут на replica.

## Replication Backlog
---
Когда replica временно отключается и потом переподключается:

```
Time: 00:00 - Replica упал

Time: 00:05 - Redis commands выполнялись (в master)
SET key1, SET key2, ... SET key100

Time: 00:10 - Replica снова онлайн
Master: "Вот что ты пропустил" (из backlog)
Replica: Sync'ит себя
```

Конфигурация:

```redis
# redis.conf
repl-backlog-size 1mb           # Сколько команд помнить
repl-backlog-ttl 3600           # На сколько времени
```

**Важно**: Если replica была offline дольше чем repl-backlog-ttl или backlog переполнился, понадобится FULL RESYNC (медленнее).

## Чтение с Replica
---

```go
// Master - write
masterRedis := redis.NewClient(&redis.Options{
    Addr: "master:6379",
})
masterRedis.Set("key", "value")

// Replica - read (не забыть про потенциальный lag!)
replicaRedis := redis.NewClient(&redis.Options{
    Addr: "replica:6380",
})
val := replicaRedis.Get("key")  // может быть старое значение!
```

**⚠️ Внимание**: Replica может быть не синхронизирована. Если нужна strong consistency, нужна синхронная репликация (дороже).

## Масштабирование reads
---

```
One master, many replicas:

Master              Replica1    Replica2    Replica3
Write ops    ───→  Read ops    Read ops    Read ops
│                  │           │           │
└──────────────────────────────────────────┘
```

**Плюсы**:
- ✅ Распределяем read нагрузку
- ✅ Читаем локально (если replica в other datacenter)
- ✅ Cheap масштабирование (добавляем replicas)

**Минусы**:
- ❌ Может быть lag (eventual consistency)
- ❌ Нужно управлять несколькими instance'ами

## Chain Replication
---

```
Master ─→ Replica1 ─→ Replica2 ─→ Replica3

Replica1 сам становится "master" для Replica2, etc.
```

Команда на Replica1:

```redis
REPLICAOF master_ip master_port
```

Replica1 теперь replica master'а И master для Replica2.

**Плюсы**: Может уменьшить нагрузку на master
**Минусы**: Больше lag через цепь

## Типичные проблемы
---
### Issue 1: High Replication Lag

```
Master: SET key value
Replica: (спустя 5 секунд) устанавливает
```

**Решение**:
- Оптимизировать сеть
- Использовать SSD для master
- Меньше нагрузки на master

### Issue 2: Replica отключилась и не переподключается

```redis
# Проверить статус на master
INFO replication

# Output:
# slave0: ip=..., state=offline, port=6380
```

**Решение**: Переподключить replica.

### Issue 3: Потеря данных при failover

```
Master упал
Replica не знает про последние 100ms команд
```

**Решение**: Использовать Sentinel для управления failover'ом (следующий модуль).

## Monitoring Replication
---

```redis
# На master'е
INFO replication
# role: master
# connected_slaves: 2
# slave0: ip=..., state=online, offset=...
# slave1: ip=..., state=online, offset=...

# На replica'е
INFO replication
# role: slave
# master_host: master_ip
# master_port: 6379
# master_link_status: up
# slave_repl_offset: ...
```

**Важные метрики**:
- `connected_slaves`: сколько replicas connected
- `slave_repl_offset`: как далеко replica позади
- `master_link_status`: up или down?

## Real-world: Master + 2 Replicas
---

```
Architecture:
Master (primary datacenter)
├─ Replica1 (same datacenter, hot standby)
└─ Replica2 (other datacenter, backup)

Используем:
├─ Master для write'ов
├─ Replica1 для local read'ов
├─ Replica2 для восстановления если центр упадёт
```

```go
// Write operations
writeConn := redis.Connect("master:6379")
writeConn.Set("user:1000", userData)

// Read operations (load balanced)
readConn1 := redis.Connect("replica1:6380")
readConn2 := redis.Connect("replica2:6380")

user := readConn1.Get("user:1000")
if user == nil {
    user = readConn2.Get("user:1000")  // fallback
}
```

## Best Practices
---
1. **Всегда используй at least 1 replica** для HA
2. **Мониторь replication lag** (SLOWLOG и INFO)
3. **Для критичных данных** используй Sentinel
4. **Тестируй failover** сценарии
5. **Backup'и от replica'е** чтобы не замедлять master
6. **Используй replica'е для analytics queries** (не критичных)

## Следующие шаги
---
Теперь когда ты понимаешь как реплицировать данные, пора разобраться с Sentinel — автоматическим failover'ом и мониторингом высокой доступности.

---

**[← Назад к Redis](../)**