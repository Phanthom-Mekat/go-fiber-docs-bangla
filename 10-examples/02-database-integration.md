# ডাটাবেস ইন্টিগ্রেশন (PostgreSQL + GORM)

PostgreSQL ডাটাবেসের সাথে Fiber অ্যাপ্লিকেশন কানেক্ট করা শিখুন।

## প্রজেক্ট স্ট্রাকচার

```
blog-api/
├── main.go
├── config/
│   └── database.go
├── models/
│   ├── user.go
│   └── post.go
├── handlers/
│   ├── user.go
│   └── post.go
├── repository/
│   ├── user_repo.go
│   └── post_repo.go
├── middleware/
│   └── auth.go
└── .env
```

## ধাপ ১: ডিপেন্ডেন্সি ইনস্টল করুন

```bash
go get github.com/gofiber/fiber/v3
go get gorm.io/gorm
go get gorm.io/driver/postgres
go get github.com/joho/godotenv
go get golang.org/x/crypto/bcrypt
```

## ধাপ ২: Environment Configuration

**.env**

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your_password
DB_NAME=blog_db
DB_SSLMODE=disable

JWT_SECRET=your-secret-key-here
PORT=3000
```

## ধাপ ৩: ডাটাবেস কনফিগারেশন

**config/database.go**

```go
package config

import (
    "fmt"
    "log"
    "os"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

var DB *gorm.DB

func ConnectDB() {
    var err error

    // DSN (Data Source Name) তৈরি করুন
    dsn := fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=%s",
        os.Getenv("DB_HOST"),
        os.Getenv("DB_PORT"),
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASSWORD"),
        os.Getenv("DB_NAME"),
        os.Getenv("DB_SSLMODE"),
    )

    // ডাটাবেস কানেক্ট করুন
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })

    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }

    log.Println("Database connected successfully!")
}

func MigrateDB(models ...interface{}) {
    err := DB.AutoMigrate(models...)
    if err != nil {
        log.Fatal("Migration failed:", err)
    }
    log.Println("Database migrated successfully!")
}
```

## ধাপ ৪: মডেল তৈরি করুন

**models/user.go**

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Name      string         `gorm:"size:100;not null" json:"name"`
    Email     string         `gorm:"size:100;uniqueIndex;not null" json:"email"`
    Password  string         `gorm:"not null" json:"-"` // JSON-এ দেখাবে না
    Posts     []Post         `gorm:"foreignKey:UserID" json:"posts,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"` // Soft delete
}

type UserRegisterRequest struct {
    Name     string `json:"name" validate:"required,min=3"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=6"`
}

type UserLoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}
```

**models/post.go**

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type Post struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Title     string         `gorm:"size:200;not null" json:"title"`
    Content   string         `gorm:"type:text;not null" json:"content"`
    Published bool           `gorm:"default:false" json:"published"`
    UserID    uint           `gorm:"not null" json:"user_id"`
    User      User           `gorm:"foreignKey:UserID" json:"user,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

type CreatePostRequest struct {
    Title   string `json:"title" validate:"required,min=5"`
    Content string `json:"content" validate:"required,min=10"`
}

type UpdatePostRequest struct {
    Title     string `json:"title"`
    Content   string `json:"content"`
    Published *bool  `json:"published"`
}
```

## ধাপ ৫: Repository Pattern

**repository/user_repo.go**

```go
package repository

import (
    "blog-api/models"
    "gorm.io/gorm"
)

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(user *models.User) error {
    return r.db.Create(user).Error
}

func (r *UserRepository) FindByEmail(email string) (*models.User, error) {
    var user models.User
    err := r.db.Where("email = ?", email).First(&user).Error
    return &user, err
}

func (r *UserRepository) FindByID(id uint) (*models.User, error) {
    var user models.User
    err := r.db.Preload("Posts").First(&user, id).Error
    return &user, err
}

func (r *UserRepository) GetAll(page, limit int) ([]models.User, int64, error) {
    var users []models.User
    var total int64

    offset := (page - 1) * limit

    // মোট সংখ্যা গণনা করুন
    r.db.Model(&models.User{}).Count(&total)

    // Pagination সহ ডেটা পান
    err := r.db.Offset(offset).Limit(limit).Find(&users).Error

    return users, total, err
}

func (r *UserRepository) Update(user *models.User) error {
    return r.db.Save(user).Error
}

func (r *UserRepository) Delete(id uint) error {
    return r.db.Delete(&models.User{}, id).Error
}
```

**repository/post_repo.go**

```go
package repository

import (
    "blog-api/models"
    "gorm.io/gorm"
)

type PostRepository struct {
    db *gorm.DB
}

func NewPostRepository(db *gorm.DB) *PostRepository {
    return &PostRepository{db: db}
}

func (r *PostRepository) Create(post *models.Post) error {
    return r.db.Create(post).Error
}

func (r *PostRepository) FindByID(id uint) (*models.Post, error) {
    var post models.Post
    err := r.db.Preload("User").First(&post, id).Error
    return &post, err
}

func (r *PostRepository) GetAll(page, limit int, published *bool) ([]models.Post, int64, error) {
    var posts []models.Post
    var total int64

    offset := (page - 1) * limit
    query := r.db.Model(&models.Post{})

    // ফিল্টার করুন (published)
    if published != nil {
        query = query.Where("published = ?", *published)
    }

    query.Count(&total)

    err := query.Preload("User").
        Offset(offset).
        Limit(limit).
        Order("created_at DESC").
        Find(&posts).Error

    return posts, total, err
}

func (r *PostRepository) GetByUserID(userID uint) ([]models.Post, error) {
    var posts []models.Post
    err := r.db.Where("user_id = ?", userID).
        Order("created_at DESC").
        Find(&posts).Error
    return posts, err
}

func (r *PostRepository) Update(post *models.Post) error {
    return r.db.Save(post).Error
}

func (r *PostRepository) Delete(id uint) error {
    return r.db.Delete(&models.Post{}, id).Error
}
```

## ধাপ ৬: হ্যান্ডলার তৈরি করুন

**handlers/user.go**

```go
package handlers

import (
    "blog-api/models"
    "blog-api/repository"
    "strconv"

    "github.com/gofiber/fiber/v3"
    "golang.org/x/crypto/bcrypt"
)

type UserHandler struct {
    repo *repository.UserRepository
}

func NewUserHandler(repo *repository.UserRepository) *UserHandler {
    return &UserHandler{repo: repo}
}

// POST /api/users/register
func (h *UserHandler) Register(c fiber.Ctx) error {
    var req models.UserRegisterRequest

    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // পাসওয়ার্ড হ্যাশ করুন
    hashedPassword, err := bcrypt.GenerateFromPassword(
        []byte(req.Password), 
        bcrypt.DefaultCost,
    )
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to hash password",
        })
    }

    user := &models.User{
        Name:     req.Name,
        Email:    req.Email,
        Password: string(hashedPassword),
    }

    if err := h.repo.Create(user); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Email already exists",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "message": "User registered successfully",
        "data":    user,
    })
}

// GET /api/users
func (h *UserHandler) GetAll(c fiber.Ctx) error {
    page, _ := strconv.Atoi(c.Query("page", "1"))
    limit, _ := strconv.Atoi(c.Query("limit", "10"))

    users, total, err := h.repo.GetAll(page, limit)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to fetch users",
        })
    }

    return c.JSON(fiber.Map{
        "data": users,
        "meta": fiber.Map{
            "page":  page,
            "limit": limit,
            "total": total,
        },
    })
}

// GET /api/users/:id
func (h *UserHandler) GetByID(c fiber.Ctx) error {
    id, err := strconv.ParseUint(c.Params("id"), 10, 32)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid user ID",
        })
    }

    user, err := h.repo.FindByID(uint(id))
    if err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "User not found",
        })
    }

    return c.JSON(fiber.Map{
        "data": user,
    })
}

// DELETE /api/users/:id
func (h *UserHandler) Delete(c fiber.Ctx) error {
    id, err := strconv.ParseUint(c.Params("id"), 10, 32)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid user ID",
        })
    }

    if err := h.repo.Delete(uint(id)); err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to delete user",
        })
    }

    return c.JSON(fiber.Map{
        "message": "User deleted successfully",
    })
}
```

## ধাপ ৭: মেইন অ্যাপ্লিকেশন

**main.go**

```go
package main

import (
    "log"
    "os"

    "blog-api/config"
    "blog-api/handlers"
    "blog-api/models"
    "blog-api/repository"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/gofiber/fiber/v3/middleware/recover"
    "github.com/joho/godotenv"
)

func main() {
    // .env ফাইল লোড করুন
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found")
    }

    // ডাটাবেস কানেক্ট করুন
    config.ConnectDB()

    // মাইগ্রেশন চালান
    config.MigrateDB(&models.User{}, &models.Post{})

    // Fiber অ্যাপ তৈরি করুন
    app := fiber.New(fiber.Config{
        AppName: "Blog API v1.0",
    })

    // মিডলওয়্যার
    app.Use(recover.New())
    app.Use(logger.New())
    app.Use(cors.New())

    // Repository ইনিশিয়ালাইজ করুন
    userRepo := repository.NewUserRepository(config.DB)
    postRepo := repository.NewPostRepository(config.DB)

    // Handler ইনিশিয়ালাইজ করুন
    userHandler := handlers.NewUserHandler(userRepo)
    postHandler := handlers.NewPostHandler(postRepo)

    // রাউট
    api := app.Group("/api")

    // User রাউট
    users := api.Group("/users")
    users.Post("/register", userHandler.Register)
    users.Get("/", userHandler.GetAll)
    users.Get("/:id", userHandler.GetByID)
    users.Delete("/:id", userHandler.Delete)

    // Post রাউট
    posts := api.Group("/posts")
    posts.Get("/", postHandler.GetAll)
    posts.Get("/:id", postHandler.GetByID)
    posts.Post("/", postHandler.Create)
    posts.Put("/:id", postHandler.Update)
    posts.Delete("/:id", postHandler.Delete)

    // সার্ভার শুরু করুন
    port := os.Getenv("PORT")
    if port == "" {
        port = "3000"
    }

    log.Fatal(app.Listen(":" + port))
}
```

## ডাটাবেস সেটআপ

### PostgreSQL ইনস্টল করুন

```bash
# Ubuntu/Debian
sudo apt install postgresql postgresql-contrib

# macOS (Homebrew)
brew install postgresql

# Windows - PostgreSQL ওয়েবসাইট থেকে ডাউনলোড করুন
```

### ডাটাবেস তৈরি করুন

```sql
CREATE DATABASE blog_db;
```

## চালান

```bash
go run main.go
```

## API টেস্ট করুন

### 1. ইউজার রেজিস্টার করুন

```bash
curl -X POST http://localhost:3000/api/users/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "রহিম আহমেদ",
    "email": "rahim@example.com",
    "password": "secure123"
  }'
```

### 2. সব ইউজার দেখুন (Pagination)

```bash
curl "http://localhost:3000/api/users?page=1&limit=10"
```

### 3. নতুন পোস্ট তৈরি করুন

```bash
curl -X POST http://localhost:3000/api/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Go Fiber শিখুন",
    "content": "Fiber একটি দ্রুত ওয়েব ফ্রেমওয়ার্ক...",
    "user_id": 1
  }'
```

## GORM বেস্ট প্র্যাকটিস

### 1. Connection Pooling

```go
sqlDB, err := config.DB.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

### 2. Preloading (N+1 সমস্যা এড়ান)

```go
// খারাপ - N+1 query
db.Find(&users)
for _, user := range users {
    db.Model(&user).Association("Posts").Find(&user.Posts)
}

// ভালো - একটি query
db.Preload("Posts").Find(&users)
```

### 3. Transaction ব্যবহার করুন

```go
err := config.DB.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&post).Error; err != nil {
        return err
    }
    return nil
})
```

### 4. Soft Delete

```go
// DeletedAt ফিল্ড যোগ করুন
type User struct {
    ID        uint
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// Soft delete
db.Delete(&user) // UPDATE users SET deleted_at = NOW()

// Permanently delete
db.Unscoped().Delete(&user) // DELETE FROM users
```

---

[← আগের: বেসিক REST API](01-basic-rest-api.md) | [পরবর্তী: JWT Authentication →](03-jwt-authentication.md)
