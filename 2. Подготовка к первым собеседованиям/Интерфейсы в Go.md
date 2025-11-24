---
layout: default
title: Интерфейсы и типизация
permalink: /interview-prep/go-deep-dive/interfaces/
---

# Цель
---
Понять, как работают интерфейсы в Go, как Go реализует полиморфизм, и почему интерфейсы — это одна из самых мощных фишек языка. Интерфейсы позволяют писать **гибкий и переиспользуемый код**.

# Зачем это нужно на собеседовании?
---
Интервьюер обязательно спросит:
- Что такое интерфейс в Go?
- Как работает неявная реализация интерфейсов?
- Чем Go отличается от классического ООП?
- Что такое пустой интерфейс `interface{}`?
- Как работает type assertion?

Твой ответ покажет, насколько глубоко ты понимаешь архитектуру кода.

# Теория

### Что такое интерфейс?
---
**Интерфейс** — это **контракт**, который описывает, какие методы должна иметь переменная. Это вроде того, как договор между тобой и твоей функцией.

**Простой пример:**
```go
// Интерфейс описывает, что "что-то" должно уметь читать
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Любой тип, у которого есть метод Read() — автоматически Reader!
// Не нужно писать "implements" как в Java
```

**Главное отличие Go от других языков:** **Ты не говоришь явно, что твой тип реализует интерфейс. Go сам проверяет это автоматически.**

```go
// Java (явная реализация)
class Dog implements Animal {
    public void speak() { ... }
}

// Go (неявная реализация)
type Dog struct {}
func (d Dog) Speak() { }
// Если где-то есть интерфейс Animal с методом Speak,
// Dog автоматически его реализует!
```

### Пример для начинающих
---
```go
package main

import "fmt"

// Интерфейс: что-то должно уметь издавать звук
type Animal interface {
    Sound() string
}

// Собака реализует Animal
type Dog struct {
    Name string
}

func (d Dog) Sound() string {
    return "Woof!"
}

// Кот реализует Animal
type Cat struct {
    Name string
}

func (c Cat) Sound() string {
    return "Meow!"
}

// Эта функция работает с ЛЮБЫм Animal!
func makeNoise(a Animal) {
    fmt.Println(a.Sound())
}

func main() {
    dog := Dog{Name: "Rex"}
    cat := Cat{Name: "Whiskers"}
    
    makeNoise(dog)  // Woof!
    makeNoise(cat)  // Meow!
    
    // Я могу создать новое животное, и makeNoise сработает,
    // даже если я не менял код функции!
}
```

**Почему это крутая штука?**
- Функция `makeNoise` не знает о Dog и Cat конкретно
- Она знает только об интерфейсе Animal
- Когда я добавлю Parrot или Lion, мне не нужно менять makeNoise!

### Пустой интерфейс `interface{}`
---
**Пустой интерфейс** — это интерфейс, который не имеет никаких методов.

```go
interface{}
```

**Главное:** Каждый тип реализует пустой интерфейс! (Потому что у каждого типа "есть" ноль методов)

```go
var x interface{}

x = 42              // ✅ OK, int реализует interface{}
x = "hello"         // ✅ OK, string реализует interface{}
x = []int{1, 2, 3}  // ✅ OK, []int реализует interface{}
x = Dog{}           // ✅ OK, Dog реализует interface{}
```

**Зачем это нужно?**
```go
// Функция может принимать ЛЮБОЙ тип
func printAnything(x interface{}) {
    fmt.Println(x)
}

printAnything(42)
printAnything("hello")
printAnything(3.14)
```

**Но внимание!** Если ты используешь `interface{}`, ты теряешь type safety. Это вроде того, как во многих язык использовать `any`:

```go
func process(x interface{}) {
    // Что-то делать с x?
    // Но я не знаю, какой это тип!
    // x. <- IDE не сможет подсказать методы
}
```

### Type Assertion (проверка типа)
---
Когда ты имеешь `interface{}` и хочешь узнать, какой конкретно тип внутри, нужна **type assertion**.

```go
var x interface{} = "hello"

// Способ 1: небезопасная проверка (panic если неправильно)
s := x.(string)
fmt.Println(s)  // hello

// Способ 2: безопасная проверка
s, ok := x.(string)
if ok {
    fmt.Println("It's a string:", s)
} else {
    fmt.Println("It's not a string")
}
```

**Type switch** (как switch, но для типов):
```go
func describe(x interface{}) {
    switch v := x.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %v\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

describe(42)       // Integer: 42
describe("hello")  // String: hello
describe(true)     // Boolean: true
```

### Reader и Writer интерфейсы
---
В Go есть два очень важных интерфейса в пакете `io`:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**Это один из красивых примеров интерфейсов в Go:**

```go
// Файлы
*os.File — это Reader и Writer

// Буферы
*bytes.Buffer — это Reader и Writer

// Сетевые соединения
net.Conn — это Reader и Writer

// Благодаря интерфейсам, одна функция может работать с файлами, буферами, сетью!
func copyData(dst Writer, src Reader) error {
    _, err := io.Copy(dst, src)
    return err
}

// Копировать файл в память
copyData(buffer, file)

// Копировать буфер на сетевое соединение
copyData(connection, buffer)
```

### Практический пример: JSON marshaller
---
```go
// Любой тип, у которого есть MarshalJSON(), 
// кастомизирует свою сериализацию
type MarshalJSON interface {
    MarshalJSON() ([]byte, error)
}

type Person struct {
    Name string
    Age  int
}

// Если я хочу, чтобы Person сериализовалась специальным образом
func (p Person) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`{"person":"%s","years":%d}`, p.Name, p.Age)), nil
}

person := Person{Name: "Ivan", Age: 30}
data, _ := json.Marshal(person)
fmt.Println(string(data))  // {"person":"Ivan","years":30}
```

### Композиция интерфейсов
---
Интерфейсы можно комбинировать!

```go
// Читатель, который может быть закрыт
type ReadCloser interface {
    Reader
    Closer
}

// Это означает, что ReadCloser должен иметь методы:
// - Read() (от Reader)
// - Close() (от Closer)
```

### Когда использовать интерфейсы
---

#### ✅ ИСПОЛЬЗУЙ интерфейсы когда:

**1. Разные типы должны делать одно и то же:**
```go
type PaymentMethod interface {
    Pay(amount float64) error
}

type CreditCard struct { ... }
func (cc CreditCard) Pay(amount float64) error { ... }

type PayPal struct { ... }
func (pp PayPal) Pay(amount float64) error { ... }

// Функция работает с любым способом оплаты
func checkout(pm PaymentMethod, amount float64) error {
    return pm.Pay(amount)
}
```

**2. Хочешь делать dependency injection:**
```go
type Logger interface {
    Log(msg string)
}

func processUser(logger Logger, user User) {
    logger.Log("Processing user: " + user.Name)
    // ...
}

// В production передаю настоящий logger
// В тестах передаю mock logger
```

**3. Хочешь писать переиспользуемый код:**
```go
// Функция работает с любым Reader!
func readAll(r io.Reader) ([]byte, error) {
    return ioutil.ReadAll(r)
}

// Она работает с файлами, сетью, буферами...
```

#### ❌ НЕ ИСПОЛЬЗУЙ интерфейсы когда:

**1. Только один тип реализует интерфейс:**
```go
// ❌ Плохо: оверинженеринг
type UserGetter interface {
    GetUser(id int) *User
}

type UserService struct { ... }
func (us UserService) GetUser(id int) *User { ... }

// ✅ Хорошо: просто использовать тип
type UserService struct { ... }
func (us UserService) GetUser(id int) *User { ... }
```

**2. Интерфейс очень большой (много методов):**
```go
// ❌ Плохо
type HugeInterface interface {
    Method1() ...
    Method2() ...
    Method3() ...
    ... (30 методов)
}

// ✅ Хорошо: разбей на маленькие интерфейсы
type Reader interface {
    Read() ...
}

type Writer interface {
    Write() ...
}
```

# Правило для собеседования
---
**Когда спрашивают про интерфейсы, отвечай:**

"Интерфейсы в Go — это контракты, описывающие, какие методы должен иметь тип. Главное отличие от других языков: в Go реализация интерфейса **неявная** — ты не пишешь `implements`. Это позволяет писать гибкий код, потому что функции работают с интерфейсами, а не с конкретными типами.

Пустой интерфейс `interface{}` позволяет работать с любым типом, но я использую его редко, потому что теряется type safety. Вместо этого я стараюсь определить нужный мне интерфейс.

Интерфейсы особенно полезны для dependency injection и когда разные типы должны делать одно и то же."

---

**[← Назад к Go углубленно](../)**
