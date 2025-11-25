---
layout: default
title: JSON и сериализация
permalink: /interview-prep/go-deep-dive/json/
---

# Цель
---
JSON — это универсальный формат обмена данными. Ты должен знать, как Go работает с JSON, какие есть struct tags, как кастомизировать маршализацию, и как избежать типичных ошибок.

# Зачем это нужно на собеседовании?
---
Тебя спросят:
- Как работает json.Marshal и json.Unmarshal?
- Что такое struct tags?
- Как кастомизировать сериализацию?
- Как работает json.RawMessage?
- Что такое stream обработка JSON?

Это вопросы, которые показывают, работал ли ты с реальными данными.

# Теория

### Основы: Marshal и Unmarshal
---
**Marshal** — преобразовать Go структуру в JSON

```go
type Person struct {
    Name string
    Age  int
}

p := Person{Name: "Ivan", Age: 30}
data, err := json.Marshal(p)
// data = []byte(`{"Name":"Ivan","Age":30}`)
```

**Unmarshal** — преобразовать JSON в Go структуру

```go
jsonData := []byte(`{"Name":"Ivan","Age":30}`)
var p Person
err := json.Unmarshal(jsonData, &p)
// p = Person{Name: "Ivan", Age: 30}
```

**Главное:** Unmarshal требует **указателя** на структуру!

```go
// ❌ Плохо
json.Unmarshal(data, p)

// ✅ Хорошо
json.Unmarshal(data, &p)
```

### Struct tags
---
**Struct tags** позволяют контролировать, как поля маршализуются в JSON.

```go
type Person struct {
    Name      string `json:"name"`           // Переименовать в JSON
    Age       int    `json:"age"`
    Email     string `json:"email,omitempty"` // Пропустить если пусто
    Password  string `json:"-"`               // Игнорировать это поле
}

p := Person{Name: "Ivan", Age: 30, Email: ""}
data, _ := json.Marshal(p)
// {"name":"Ivan","age":30}
// Email не включен (omitempty)
// Password пропущен (-)
```

**Основные теги:**
- `json:"fieldName"` — имя в JSON
- `json:"fieldName,omitempty"` — пропустить если нулевое значение
- `json:"-"` — игнорировать поле
- `json:"fieldName,string"` — конвертировать в строку

### Типичные проблемы
---

#### Проблема 1: Экспортируемость полей
```go
// ❌ Плохо: поле не экспортируемо (начинается с малой буквы)
type Person struct {
    name string
}

// ✅ Хорошо: поле экспортируемо (начинается с большой буквы)
type Person struct {
    Name string `json:"name"`
}
```

JSON работает только с **экспортируемыми полями** (первая буква большая).

#### Проблема 2: Неправильный тип
```go
// ❌ Плохо: Age ожидает int, но JSON содержит строку
jsonData := []byte(`{"name":"Ivan","age":"30"}`)
var p Person
json.Unmarshal(jsonData, &p)  // Ошибка!

// ✅ Хорошо: типы совпадают
jsonData := []byte(`{"name":"Ivan","age":30}`)
```

#### Проблема 3: Поля не в JSON
```go
jsonData := []byte(`{"name":"Ivan","age":30}`)
var p Person
json.Unmarshal(jsonData, &p)  // OK, дополнительные поля игнорируются

// Но если в структуре есть поле, которого нет в JSON
var p Person  // Age будет 0
```

### Кастомная сериализация
---
Ты можешь контролировать, как структура маршализуется через методы.

#### MarshalJSON
```go
type Person struct {
    Name string
    Age  int
}

// Кастомный Marshal
func (p Person) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`{"person":"%s","years":%d}`, p.Name, p.Age)), nil
}

p := Person{Name: "Ivan", Age: 30}
data, _ := json.Marshal(p)
// {"person":"Ivan","years":30}
```

#### UnmarshalJSON
```go
// Кастомный Unmarshal
func (p *Person) UnmarshalJSON(data []byte) error {
    var raw map[string]interface{}
    if err := json.Unmarshal(data, &raw); err != nil {
        return err
    }
    
    p.Name, _ = raw["person"].(string)
    age, _ := raw["years"].(float64)
    p.Age = int(age)
    
    return nil
}

jsonData := []byte(`{"person":"Ivan","years":30}`)
var p Person
json.Unmarshal(jsonData, &p)
```

**Зачем это нужно?**
- Конвертировать форматы (например, timestamp → time.Time)
- Валидировать данные
- Обрабатывать вложенные структуры

### JSON с динамической структурой
---
Иногда JSON структура неизвестна или динамическая.

#### map[string]interface{}
```go
jsonData := []byte(`{"name":"Ivan","age":30,"extra":"data"}`)

var data map[string]interface{}
json.Unmarshal(jsonData, &data)

fmt.Println(data["name"])  // "Ivan"
fmt.Println(data["age"])   // 30.0 (float64!)
```

**Проблема:** Числа становятся float64!

```go
age := data["age"].(float64)  // Нужно конвертировать
fmt.Println(int(age))  // 30
```

#### json.RawMessage
```go
type Message struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"`  // Сырые JSON данные
}

jsonData := []byte(`{"type":"user","data":{"name":"Ivan"}}`)
var msg Message
json.Unmarshal(jsonData, &msg)

// msg.Data = []byte(`{"name":"Ivan"}`)
// Теперь можешь обработать отдельно
```

### Stream обработка JSON
---
Когда JSON очень большой, лучше обработать его потоком, а не весь сразу.

#### Decoder (для чтения)
```go
import "encoding/json"

func processJSONStream(file *os.File) error {
    decoder := json.NewDecoder(file)
    
    for {
        var person Person
        if err := decoder.Decode(&person); err == io.EOF {
            break
        } else if err != nil {
            return err
        }
        
        // Обработка каждого объекта
        fmt.Println(person.Name)
    }
    
    return nil
}

// Файл содержит JSON array:
// [{"name":"Ivan"},{"name":"Maria"}]
```

#### Encoder (для записи)
```go
func writeJSONStream(file *os.File, people []Person) error {
    encoder := json.NewEncoder(file)
    
    for _, p := range people {
        if err := encoder.Encode(p); err != nil {
            return err
        }
    }
    
    return nil
}
```

### Типичные паттерны
---

#### Вложенные структуры
```go
type Address struct {
    City   string `json:"city"`
    Street string `json:"street"`
}

type Person struct {
    Name    string  `json:"name"`
    Age     int     `json:"age"`
    Address Address `json:"address"`
}

jsonData := []byte(`{
    "name":"Ivan",
    "age":30,
    "address":{"city":"Moscow","street":"Main St"}
}`)

var p Person
json.Unmarshal(jsonData, &p)
```

#### Массивы
```go
type Group struct {
    Name    string   `json:"name"`
    Members []Person `json:"members"`
}

jsonData := []byte(`{
    "name":"Team A",
    "members":[{"name":"Ivan"},{"name":"Maria"}]
}`)

var g Group
json.Unmarshal(jsonData, &g)
```

#### Null значения
```go
type Person struct {
    Name  string `json:"name"`
    Email *string `json:"email"`  // Может быть null
}

// {"name":"Ivan","email":null}
var p Person
json.Unmarshal(data, &p)
// p.Email = nil
```

## Практический пример: API обработка

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    Tags      []string  `json:"tags,omitempty"`
}

// Custom Marshal для форматирования даты
func (u User) MarshalJSON() ([]byte, error) {
    type UserAlias User
    return json.Marshal(&struct {
        CreatedAt string `json:"created_at"`
        *UserAlias
    }{
        CreatedAt: u.CreatedAt.Format("2006-01-02"),
        UserAlias: (*UserAlias)(&u),
    })
}

func main() {
    user := User{
        ID:        1,
        Name:      "Ivan",
        Email:     "ivan@example.com",
        CreatedAt: time.Now(),
        Tags:      []string{"developer", "gopher"},
    }
    
    // Marshal
    data, _ := json.Marshal(user)
    fmt.Println(string(data))
    
    // Unmarshal
    jsonData := []byte(`{"id":2,"name":"Maria","created_at":"2024-01-01"}`)
    var u User
    json.Unmarshal(jsonData, &u)
    fmt.Println(u)
}
```

# Правило для собеседования
---
**Когда спрашивают про JSON, отвечай:**

"JSON является стандартным форматом обмена данных. В Go я использую encoding/json пакет.

Важные моменты:
1. Структуры должны иметь экспортируемые поля (с большой буквы)
2. Struct tags контролируют маршализацию (json:\"field,omitempty\")
3. Unmarshal требует указателя на структуру
4. Я могу кастомизировать сериализацию через MarshalJSON и UnmarshalJSON

Для больших JSON файлов я использую decoder/encoder для stream обработки, чтобы не загружать всё в память сразу."

---

**[← Назад к Go углубленно](../)**
