---
layout: default
title: HTTP и networking
permalink: /interview-prep/go-deep-dive/http/
---

# Цель
---
HTTP — это протокол, на котором построена вся веб. Ты должен знать, как Go работает с HTTP, как создавать серверы, делать запросы, и понимать, как работает net пакет.

# Зачем это нужно на собеседовании?
---
Тебя спросят:
- Как создать HTTP сервер в Go?
- Как сделать HTTP запрос?
- Что такое Handler и Middleware?
- Как работает net.Conn?
- Как обработать таймауты?

Это базовые вопросы для backend разработчика.

# Теория

### Создание простого HTTP сервера
---

#### Базовый сервер
```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    // Регистрируем обработчик
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })
    
    // Запускаем сервер на порту 8080
    http.ListenAndServe(":8080", nil)
}

// Тест: curl http://localhost:8080/
```

#### Несколько маршрутов
```go
func main() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/users", usersHandler)
    http.HandleFunc("/users/{id}", userHandler)
    
    http.ListenAndServe(":8080", nil)
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintf(w, `{"message":"Home"}`)
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        fmt.Fprintf(w, `[{"id":1,"name":"Ivan"}]`)
    } else if r.Method == http.MethodPost {
        w.WriteHeader(http.StatusCreated)
        fmt.Fprintf(w, `{"id":2,"name":"Maria"}`)
    }
}
```

### Request и Response
---

#### Читать данные из Request
```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Метод
    fmt.Println(r.Method)  // GET, POST, PUT, DELETE
    
    // URL
    fmt.Println(r.URL.Path)      // /users/123
    fmt.Println(r.URL.Query())   // ?page=1&limit=10
    
    // Query параметры
    page := r.URL.Query().Get("page")      // "1"
    limit := r.URL.Query().Get("limit")    // "10"
    
    // Headers
    contentType := r.Header.Get("Content-Type")
    
    // Body (для POST/PUT)
    body := make([]byte, r.ContentLength)
    r.Body.Read(body)
    
    // Формальные данные
    r.ParseForm()
    name := r.FormValue("name")
    
    // JSON
    var user User
    json.NewDecoder(r.Body).Decode(&user)
}
```

#### Писать Response
```go
func handleResponse(w http.ResponseWriter, r *http.Request) {
    // Status код
    w.WriteHeader(http.StatusOK)  // 200
    // w.WriteHeader(http.StatusNotFound)  // 404
    // w.WriteHeader(http.StatusInternalServerError)  // 500
    
    // Headers
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom-Header", "value")
    
    // Body
    fmt.Fprintf(w, `{"message":"Success"}`)
    
    // Или JSON
    json.NewEncoder(w).Encode(map[string]string{
        "message": "Success",
    })
}
```

### HTTP Client (делать запросы)
---

#### Простой GET запрос
```go
resp, err := http.Get("http://example.com")
if err != nil {
    // Ошибка при выполнении запроса
}
defer resp.Body.Close()

// Прочитать Body
body, _ := io.ReadAll(resp.Body)
fmt.Println(string(body))

// Проверить статус
if resp.StatusCode != http.StatusOK {
    fmt.Println("Error:", resp.StatusCode)
}
```

#### POST запрос
```go
payload := map[string]string{"name": "Ivan"}
jsonData, _ := json.Marshal(payload)

resp, err := http.Post(
    "http://example.com/users",
    "application/json",
    bytes.NewBuffer(jsonData),
)

body, _ := io.ReadAll(resp.Body)
defer resp.Body.Close()
```

#### Кастомный запрос с headers
```go
client := &http.Client{}

req, _ := http.NewRequest("GET", "http://example.com", nil)
req.Header.Add("Authorization", "Bearer token123")
req.Header.Add("User-Agent", "MyApp/1.0")

resp, _ := client.Do(req)
defer resp.Body.Close()
```

#### С таймаутом
```go
client := &http.Client{
    Timeout: 10 * time.Second,  // 10 сек максимум
}

resp, err := client.Get("http://example.com")
if err != nil {
    if err == context.DeadlineExceeded {
        fmt.Println("Request timeout")
    }
}
```

### Middleware
---
**Middleware** — это функция, которая обрабатывает request до и/или после основного handler'а.

#### Простой middleware
```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("[%s] %s %s\n", time.Now().Format("15:04:05"), r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello")
    })
    
    // Обернуть в middleware
    handler := loggingMiddleware(mux)
    
    http.ListenAndServe(":8080", handler)
}
```

#### Middleware с обработкой ошибок
```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                fmt.Fprintf(w, "Internal Server Error")
                w.WriteHeader(http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

#### Middleware для authentication
```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        if token == "" {
            w.WriteHeader(http.StatusUnauthorized)
            fmt.Fprintf(w, "Missing token")
            return
        }
        
        // Проверить token
        if !isValidToken(token) {
            w.WriteHeader(http.StatusForbidden)
            fmt.Fprintf(w, "Invalid token")
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/users", usersHandler)
    
    // Применить middleware
    handler := authMiddleware(mux)
    
    http.ListenAndServe(":8080", handler)
}
```

### net пакет: низкоуровневая работа с сетью
---

#### TCP сервер
```go
import "net"

func main() {
    listener, _ := net.Listen("tcp", ":9000")
    defer listener.Close()
    
    for {
        conn, _ := listener.Accept()
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()
    
    buffer := make([]byte, 1024)
    n, _ := conn.Read(buffer)
    fmt.Println(string(buffer[:n]))
    
    conn.Write([]byte("Hello from server"))
}
```

#### TCP клиент
```go
conn, _ := net.Dial("tcp", "localhost:9000")
defer conn.Close()

conn.Write([]byte("Hello from client"))

buffer := make([]byte, 1024)
n, _ := conn.Read(buffer)
fmt.Println(string(buffer[:n]))
```

#### UDP (без соединения)
```go
// UDP сервер
addr, _ := net.ResolveUDPAddr("udp", ":9001")
conn, _ := net.ListenUDP("udp", addr)
defer conn.Close()

buffer := make([]byte, 1024)
n, remoteAddr, _ := conn.ReadFromUDP(buffer)
fmt.Printf("Received from %s: %s\n", remoteAddr, string(buffer[:n]))

conn.WriteToUDP([]byte("Echo: "+string(buffer[:n])), remoteAddr)
```

### Полный пример: простое REST API

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

var users = []User{
    {1, "Ivan", 30},
    {2, "Maria", 25},
}

func listUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.URL.Query().Get("id")
    userID, _ := strconv.Atoi(id)
    
    for _, user := range users {
        if user.ID == userID {
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(user)
            return
        }
    }
    
    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "User not found")
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    json.NewDecoder(r.Body).Decode(&user)
    
    user.ID = len(users) + 1
    users = append(users, user)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func main() {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        if r.Method == http.MethodGet {
            listUsers(w, r)
        } else if r.Method == http.MethodPost {
            createUser(w, r)
        }
    })
    
    mux.HandleFunc("/users/", getUser)
    
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

# Правило для собеседования
---
**Когда спрашивают про HTTP, отвечай:**

"HTTP в Go — это просто. Есть встроенный пакет net/http с хорошим API.

Для сервера я использую http.HandleFunc для регистрации маршрутов. Важно помнить, что request handler должен писать в ResponseWriter и читать из Request.

Для клиента я использую http.Client с таймаутом, чтобы не зависнуть на долгих запросах.

Middleware используется для добавления логирования, аутентификации и обработки ошибок.

Для сложных случаев (много маршрутов, параметры в path) я использую фреймворк как Gin, но основы — это всегда стандартный net/http пакет."

---

**[← Назад к Go углубленно](../)**
