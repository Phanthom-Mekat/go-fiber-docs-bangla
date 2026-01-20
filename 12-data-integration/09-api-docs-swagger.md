# Swagger API ডকুমেন্টেশন

একটি ভালো API এর অন্যতম বৈশিষ্ট্য হলো ভালো ডকুমেন্টেশন। Fiber v3 এর জন্য আমরা `swaggo/swag` এবং `gofiber/contrib/v3/swagger` ব্যবহার করব।

> **লক্ষ্য করুন:** v3 তে `gofiber/swagger` প্যাকেজটি `gofiber/contrib/v3/swagger` এবং `swaggo` তে স্থানান্তরিত হয়েছে।

## ১. সেটআপ

Swag CLI ইনস্টল করুন:
```bash
go install github.com/swaggo/swag/cmd/swag@latest
```

## ২. মেইন ফাইলে অ্যানোটেশন

`main.go` এর শুরুতে API এর সাধারণ তথ্য দিন:

```go
// @title           My Fiber API
// @version         1.0
// @description     This is a sample server for Fiber v3.
// @host            localhost:3000
// @BasePath        /api/v1
func main() { ... }
```

## ৩. হ্যান্ডলারে অ্যানোটেশন

প্রতিটি হ্যান্ডলারের উপরে কমেন্ট যোগ করুন:

```go
// CreateUser godoc
// @Summary      Create a new user
// @Description  Create a new user with the input payload
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        user  body      model.User  true  "User JSON"
// @Success      201   {object}  model.User
// @Failure      400   {object}  fiber.Error
// @Router       /users [post]
func CreateUser(c fiber.Ctx) error { ... }
```

## ৪. ডক জেনারেট এবং সার্ভ করা

টার্মিনালে রান করুন:
```bash
swag init
```
এটি `docs/` ফোল্ডার তৈরি করবে।

এখন Fiber অ্যাপে এটি কানেক্ট করুন:

```go
import (
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/contrib/v3/swagger" // v3 প্যাকেজ
    
    _ "my-app/docs" // এটি ইমপোর্ট করা জরুরি যাতে init() ফাংশন রান করে
)

func main() {
    app := fiber.New()

    // Swagger UI সার্ভ করার জন্য কনফিগ
    cfg := swagger.Config{
        BasePath: "/api/v1", // সোয়েরগার UI এর জন্য বেইস পাথ
        FilePath: "./docs/swagger.json",
        Path:     "docs", // ইউজার ব্রাউজারে /docs এ যাবে
        Title:    "Swagger API Docs",
    }

    app.Use(swagger.New(cfg))

    app.Listen(":3000")
}
```

এখন `http://localhost:3000/docs` এ গেলে আপনি Swagger UI দেখতে পাবেন।

---
[← আগের: Casbin](08-rbac-casbin.md) | [সূচি](../README.md) | [পরবর্তী: GraphQL →](10-graphql-gqlgen.md)
