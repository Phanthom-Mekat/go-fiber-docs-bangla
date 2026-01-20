# GORM ইন্টিগ্রেশন (GORM Integration)

Go-এর সবচেয়ে জনপ্রিয় ORM হলো GORM। Fiber-এর সাথে GORM ব্যবহার করে আপনি সহজেই ডেটাবেস অপারেশন হ্যান্ডেল করতে পারেন।

## ১. ইনস্টলেশন

প্রথমে GORM এবং আপনার ডেটাবেস ড্রাইভার (যেমন PostgreSQL) ইনস্টল করুন:

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
```

## ২. ডেটাবেস কানেকশন সেটআপ

আমরা একটি `database` প্যাকেজ তৈরি করব যা কানেকশন পুল ম্যানেজ করবে।

```go
package database

import (
    "log"
    "os"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

// গ্লোবাল DB ভেরিয়েবল (অথবা ডিপেন্ডেন্সি ইনজেকশন ব্যবহার করুন)
var DB *gorm.DB

func Connect() {
    dsn := "host=localhost user=gorm password=gorm dbname=gorm port=9920 sslmode=disable"

    var err error
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // SQL কোয়েলি লগ দেখার জন্য
    })

    if err != nil {
        log.Fatal("Failed to connect to database. \n", err)
    }

    log.Println("Database connected")
    
    // মাইগ্রেশন রান করা (শুধুমাত্র ডেভ এনভায়রনমেন্টে)
    // DB.AutoMigrate(&model.User{})
}
```

## ৩. মডেল তৈরি

`User` মডেল ডিফাইন করা:

```go
package model

import "gorm.io/gorm"

type User struct {
    gorm.Model // ID, CreatedAt, UpdatedAt, DeletedAt ফিল্ড যোগ করে
    Name  string `json:"name"`
    Email string `json:"email" gorm:"unique"`
    Age   int    `json:"age"`
}
```

## ৪. হ্যান্ডলার ইমপ্লিমেন্টেশন

Fiber হ্যান্ডলারে GORM ব্যবহার করা:

```go
package handler

import (
    "github.com/gofiber/fiber/v3"
    "my-app/database"
    "my-app/model"
)

// সকল ইউজার পাওয়া
func GetUsers(c fiber.Ctx) error {
    var users []model.User
    
    // কনটেক্সট পাস করা জরুরি (টাইমআউটের জন্য)
    result := database.DB.WithContext(c.UserContext()).Find(&users)
    
    if result.Error != nil {
        return c.Status(500).JSON(fiber.Map{"error": result.Error.Error()})
    }
    
    return c.JSON(users)
}

// নতুন ইউজার তৈরি
func CreateUser(c fiber.Ctx) error {
    user := new(model.User)
    
    if err := c.Bind().Body(user); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": err.Error()})
    }
    
    result := database.DB.WithContext(c.UserContext()).Create(&user)
    
    if result.Error != nil {
        return c.Status(500).JSON(fiber.Map{"error": result.Error.Error()})
    }
    
    return c.Status(201).JSON(user)
}
```

## ৫. মেইন ফাংশনে ইন্টিগ্রেশন

```go
func main() {
    // ১. ডিবি কানেক্ট
    database.Connect()

    app := fiber.New()

    // ২. রাউট রেজিস্টার
    app.Get("/users", handler.GetUsers)
    app.Post("/users", handler.CreateUser)

    app.Listen(":3000")
}
```

## টিপস

*   **Connection Pool:** প্রোডাকশনে `sql.DB` অবজেক্ট নিয়ে `SetMaxIdleConns` এবং `SetMaxOpenConns` কনফিগার করুন।
*   **Hooks:** GORM এর `BeforeCreate`/`BeforeSave` হুক ব্যবহার করে পাসওয়ার্ড হ্যাশ করা বা ভ্যালিডেশন করা যায়।

---
[সূচি](../README.md) | [পরবর্তী: SQLC (Type-safe SQL) →](02-sql-builder-sqlc.md)
