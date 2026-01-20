# বেসিক REST API

একটি সম্পূর্ণ CRUD (Create, Read, Update, Delete) REST API তৈরি করা শিখুন।

## প্রজেক্ট স্ট্রাকচার

```
todo-api/
├── main.go
├── handlers/
│   └── todo.go
├── models/
│   └── todo.go
├── database/
│   └── database.go
└── go.mod
```

## ধাপ ১: প্রজেক্ট সেটআপ

```bash
mkdir todo-api
cd todo-api
go mod init todo-api
go get github.com/gofiber/fiber/v3
go get github.com/google/uuid
```

## ধাপ ২: মডেল তৈরি করুন

**models/todo.go**

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type Todo struct {
    ID          string    `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Completed   bool      `json:"completed"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

// নতুন Todo তৈরি করার জন্য
type CreateTodoRequest struct {
    Title       string `json:"title" validate:"required,min=3"`
    Description string `json:"description"`
}

// Todo আপডেট করার জন্য
type UpdateTodoRequest struct {
    Title       string `json:"title"`
    Description string `json:"description"`
    Completed   *bool  `json:"completed"`
}
```

## ধাপ ৩: ইন-মেমোরি ডাটাবেস

**database/database.go**

```go
package database

import (
    "errors"
    "sync"
    "todo-api/models"
)

type Database struct {
    todos map[string]*models.Todo
    mu    sync.RWMutex
}

var (
    ErrNotFound = errors.New("todo not found")
)

func New() *Database {
    return &Database{
        todos: make(map[string]*models.Todo),
    }
}

// সব Todo পান
func (db *Database) GetAll() []*models.Todo {
    db.mu.RLock()
    defer db.mu.RUnlock()

    todos := make([]*models.Todo, 0, len(db.todos))
    for _, todo := range db.todos {
        todos = append(todos, todo)
    }
    return todos
}

// ID দিয়ে Todo পান
func (db *Database) GetByID(id string) (*models.Todo, error) {
    db.mu.RLock()
    defer db.mu.RUnlock()

    todo, exists := db.todos[id]
    if !exists {
        return nil, ErrNotFound
    }
    return todo, nil
}

// নতুন Todo তৈরি করুন
func (db *Database) Create(todo *models.Todo) {
    db.mu.Lock()
    defer db.mu.Unlock()
    db.todos[todo.ID] = todo
}

// Todo আপডেট করুন
func (db *Database) Update(id string, todo *models.Todo) error {
    db.mu.Lock()
    defer db.mu.Unlock()

    if _, exists := db.todos[id]; !exists {
        return ErrNotFound
    }
    db.todos[id] = todo
    return nil
}

// Todo ডিলিট করুন
func (db *Database) Delete(id string) error {
    db.mu.Lock()
    defer db.mu.Unlock()

    if _, exists := db.todos[id]; !exists {
        return ErrNotFound
    }
    delete(db.todos, id)
    return nil
}
```

## ধাপ ৪: হ্যান্ডলার তৈরি করুন

**handlers/todo.go**

```go
package handlers

import (
    "time"
    "todo-api/database"
    "todo-api/models"

    "github.com/gofiber/fiber/v3"
    "github.com/google/uuid"
)

type TodoHandler struct {
    db *database.Database
}

func NewTodoHandler(db *database.Database) *TodoHandler {
    return &TodoHandler{db: db}
}

// GET /api/todos - সব Todo পান
func (h *TodoHandler) GetAll(c fiber.Ctx) error {
    todos := h.db.GetAll()
    return c.JSON(fiber.Map{
        "success": true,
        "data":    todos,
        "count":   len(todos),
    })
}

// GET /api/todos/:id - একটি Todo পান
func (h *TodoHandler) GetByID(c fiber.Ctx) error {
    id := c.Params("id")

    todo, err := h.db.GetByID(id)
    if err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "success": false,
            "error":   "Todo not found",
        })
    }

    return c.JSON(fiber.Map{
        "success": true,
        "data":    todo,
    })
}

// POST /api/todos - নতুন Todo তৈরি করুন
func (h *TodoHandler) Create(c fiber.Ctx) error {
    var req models.CreateTodoRequest

    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "success": false,
            "error":   "Invalid request body",
        })
    }

    // ভ্যালিডেশন
    if req.Title == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "success": false,
            "error":   "Title is required",
        })
    }

    now := time.Now()
    todo := &models.Todo{
        ID:          uuid.New().String(),
        Title:       req.Title,
        Description: req.Description,
        Completed:   false,
        CreatedAt:   now,
        UpdatedAt:   now,
    }

    h.db.Create(todo)

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "success": true,
        "data":    todo,
        "message": "Todo created successfully",
    })
}

// PUT /api/todos/:id - Todo আপডেট করুন
func (h *TodoHandler) Update(c fiber.Ctx) error {
    id := c.Params("id")

    // চেক করুন Todo আছে কিনা
    todo, err := h.db.GetByID(id)
    if err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "success": false,
            "error":   "Todo not found",
        })
    }

    var req models.UpdateTodoRequest
    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "success": false,
            "error":   "Invalid request body",
        })
    }

    // আপডেট করুন
    if req.Title != "" {
        todo.Title = req.Title
    }
    if req.Description != "" {
        todo.Description = req.Description
    }
    if req.Completed != nil {
        todo.Completed = *req.Completed
    }
    todo.UpdatedAt = time.Now()

    h.db.Update(id, todo)

    return c.JSON(fiber.Map{
        "success": true,
        "data":    todo,
        "message": "Todo updated successfully",
    })
}

// DELETE /api/todos/:id - Todo ডিলিট করুন
func (h *TodoHandler) Delete(c fiber.Ctx) error {
    id := c.Params("id")

    if err := h.db.Delete(id); err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "success": false,
            "error":   "Todo not found",
        })
    }

    return c.JSON(fiber.Map{
        "success": true,
        "message": "Todo deleted successfully",
    })
}
```

## ধাপ ৫: মেইন অ্যাপ্লিকেশন

**main.go**

```go
package main

import (
    "log"
    "todo-api/database"
    "todo-api/handlers"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/gofiber/fiber/v3/middleware/recover"
)

func main() {
    // Fiber অ্যাপ তৈরি করুন
    app := fiber.New(fiber.Config{
        AppName: "Todo API v1.0",
        ErrorHandler: func(c fiber.Ctx, err error) error {
            code := fiber.StatusInternalServerError
            if e, ok := err.(*fiber.Error); ok {
                code = e.Code
            }
            return c.Status(code).JSON(fiber.Map{
                "success": false,
                "error":   err.Error(),
            })
        },
    })

    // মিডলওয়্যার
    app.Use(recover.New())
    app.Use(logger.New())
    app.Use(cors.New())

    // ডাটাবেস ইনিশিয়ালাইজ করুন
    db := database.New()

    // হ্যান্ডলার ইনিশিয়ালাইজ করুন
    todoHandler := handlers.NewTodoHandler(db)

    // রাউট
    api := app.Group("/api")

    // Health check
    api.Get("/health", func(c fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status": "ok",
            "message": "Server is running",
        })
    })

    // Todo রাউট
    todos := api.Group("/todos")
    todos.Get("/", todoHandler.GetAll)
    todos.Get("/:id", todoHandler.GetByID)
    todos.Post("/", todoHandler.Create)
    todos.Put("/:id", todoHandler.Update)
    todos.Delete("/:id", todoHandler.Delete)

    // সার্ভার শুরু করুন
    log.Fatal(app.Listen(":3000"))
}
```

## চালান

```bash
go run main.go
```

## API টেস্ট করুন

### 1. Health Check

```bash
curl http://localhost:3000/api/health
```

### 2. নতুন Todo তৈরি করুন

```bash
curl -X POST http://localhost:3000/api/todos \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Go শিখুন",
    "description": "Fiber ফ্রেমওয়ার্ক মাস্টার করুন"
  }'
```

### 3. সব Todo দেখুন

```bash
curl http://localhost:3000/api/todos
```

### 4. একটি Todo দেখুন

```bash
curl http://localhost:3000/api/todos/{id}
```

### 5. Todo আপডেট করুন

```bash
curl -X PUT http://localhost:3000/api/todos/{id} \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Go মাস্টার করুন",
    "completed": true
  }'
```

### 6. Todo ডিলিট করুন

```bash
curl -X DELETE http://localhost:3000/api/todos/{id}
```

## রেসপন্স উদাহরণ

### সফল রেসপন্স (Create)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Go শিখুন",
    "description": "Fiber ফ্রেমওয়ার্ক মাস্টার করুন",
    "completed": false,
    "created_at": "2024-01-20T10:30:00Z",
    "updated_at": "2024-01-20T10:30:00Z"
  },
  "message": "Todo created successfully"
}
```

### এরর রেসপন্স

```json
{
  "success": false,
  "error": "Todo not found"
}
```

## বেস্ট প্র্যাকটিস

### 1. Error Handling
```go
// কাস্টম এরর হ্যান্ডলার ব্যবহার করুন
app := fiber.New(fiber.Config{
    ErrorHandler: customErrorHandler,
})
```

### 2. Response Format
```go
// সব রেসপন্সে একই ফরম্যাট ব্যবহার করুন
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Message string      `json:"message,omitempty"`
}
```

### 3. Validation
```go
// ইনপুট ভ্যালিডেশন সবসময় করুন
if req.Title == "" {
    return c.Status(400).JSON(fiber.Map{
        "success": false,
        "error": "Title is required",
    })
}
```

### 4. Thread Safety
```go
// শেয়ার্ড ডেটায় Mutex ব্যবহার করুন
type Database struct {
    todos map[string]*Todo
    mu    sync.RWMutex  // Read-Write Mutex
}
```

## পরবর্তী ধাপ

এই বেসিক API-তে আরও ফিচার যোগ করুন:

1. ✅ Pagination যোগ করুন
2. ✅ Filtering ও Sorting
3. ✅ Validation লাইব্রেরি (go-playground/validator)
4. ✅ Database ইন্টিগ্রেশন (PostgreSQL/MongoDB)
5. ✅ Authentication যোগ করুন

---

[← আগের: উদাহরণ তালিকা](README.md) | [পরবর্তী: ডাটাবেস ইন্টিগ্রেশন →](02-database-integration.md)
