# Type-safe SQL with sqlc

যারা ORM পছন্দ করেন না এবং raw SQL এর পারফরম্যান্স চান, তাদের জন্য `sqlc` সেরা টুল। এটি আপনার SQL স্কিমা এবং কোয়েরি থেকে **Type-safe Go code** জেনারেট করে।

## কেন sqlc?

*   **রানটাইম এরর নেই:** কোয়েরি ভুল হলে কম্পাইল টাইমেই ধরা পড়ে।
*   **সুপার ফাস্ট:** `gorm` এর তুলনায় অনেক দ্রুত কারণ রিফলেকশন ব্যবহার করে না।
*   **Raw SQL:** আপনি সরাসরি SQL লিখছেন, তাই ফুল কন্ট্রোল থাকে।

## ১. সেটআপ

`sqlc` CLI ইনস্টল করুন এবং `sqlc.yaml` কনফিগার করুন:

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "query.sql"
    schema: "schema.sql"
    gen:
      go:
        package: "db"
        out: "db"
        sql_package: "pgx/v5" # আমরা pgx ড্রাইভার ব্যবহার করব
```

## ২. স্কিমা ও কোয়েরি লেখা

`query.sql` ফাইলে কোয়েরি লিখুন:

```sql
-- name: GetUser :one
SELECT * FROM users
WHERE id = $1 LIMIT 1;

-- name: ListUsers :many
SELECT * FROM users
ORDER BY name;

-- name: CreateUser :one
INSERT INTO users (
  name, email, bio
) VALUES (
  $1, $2, $3
)
RETURNING *;
```

এরপর `sqlc generate` কমান্ড রান করুন। এটি `db` প্যাকেজে Go কোড জেনারেট করবে।

## ৩. Fiber হ্যান্ডলারে ব্যবহার

```go
package main

import (
    "context"
    "github.com/gofiber/fiber/v3"
    "github.com/jackc/pgx/v5/pgxpool"
    "my-project/db" // জেনারেটেড প্যাকেজ
)

type ApiServer struct {
    store *db.Queries
}

func main() {
    // ১. pgx কানেকশন পুল
    pool, _ := pgxpool.New(context.Background(), "postgres://user:pass@localhost:5432/mydb")
    
    // ২. sqlc স্টোর ইনিশিয়ালাইজ
    queries := db.New(pool)
    server := &ApiServer{store: queries}

    app := fiber.New()

    app.Post("/users", server.CreateUserHandler)
    app.Get("/users/:id", server.GetUserHandler)

    app.Listen(":3000")
}

// হ্যান্ডলার ইমপ্লিমেন্টেশন
func (s *ApiServer) CreateUserHandler(c fiber.Ctx) error {
    // ইনপুট বাইন্ডিং (D TO)
    type CreateUserReq struct {
        Name  string `json:"name"`
        Email string `json:"email"`
        Bio   string `json:"bio"`
    }
    
    req := new(CreateUserReq)
    if err := c.Bind().Body(req); err != nil {
        return c.Status(400).SendString(err.Error())
    }

    // sqlc মেথড কল (জেনারেটেড)
    // CreateUserParams স্ট্রাক্ট অটোমেটিক তৈরি হয়েছে
    user, err := s.store.CreateUser(c.UserContext(), db.CreateUserParams{
        Name:  req.Name,
        Email: req.Email,
        Bio:   &req.Bio, // Nullable string handling
    })
    
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }

    return c.Status(201).JSON(user)
}
```

দেখুন কত ক্লিন এবং টাইপ-সেফ কোড! ORM এর ম্যাজিক নেই, আবার Raw SQL এরboilerplate কোডও লিখতে হচ্ছে না।

---
[← আগের: GORM](01-gorm-integration.md) | [সূচি](../README.md) | [পরবর্তী: MongoDB →](03-mongodb-official.md)
