# ক্লিন আর্কিটেকচার (Clean Architecture)

বড় স্কেলের অ্যাপ্লিকেশনে কোড মেইনটেইনেবল রাখার জন্য "Clean Architecture" বা "The Onion Architecture" অনুসরণ করা জরুরি। এর মূল নীতি হলো **Dependency Rule**: ডিপেন্ডেন্সি সবসময় বাইরের লেয়ার থেকে ভেতরের দিকে নির্দেশ করবে।

## লেয়ারসমূহ (Layers)

1.  **Entities (Domain):** কোর বিজনেস লজিক এবং অবজেক্ট (যেমন User struct)। এটি কোনো ফ্রেমওয়ার্ক চেনে না।
2.  **Usecases (Service):** অ্যাপ্লিকেশনের স্পেসিফিক বিজনেস রুলস (যেমন CreateUser)।
3.  **Controllers (Delivery/Transport):** HTTP/gRPC হ্যান্ডলার (Fiber)।
4.  **Frameworks & Drivers (Infrastructure):** ডেটাবেস, এক্সটার্নাল API।

## প্রজেক্ট স্ট্রাকচার

```
/
├── cmd/api/main.go          # এন্ট্রি পয়েন্ট
├── internal/
│   ├── domain/              # Entities
│   │   └── user.go
│   ├── usecase/             # Business Logic
│   │   └── user_usecase.go
│   ├── repository/          # Data Access Interface
│   │   └── user_repo.go
│   └── deliver/http/        # Fiber Handlers
│       └── user_handler.go
└── pkg/                     # পাবলিক লাইব্রেরি
```

## কোড উদাহরণ

### ১. Domain (Entity)

```go
package domain

import "time"

type User struct {
    ID        int64     `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// Repository Interface (Contract)
type UserRepository interface {
    Store(user *User) error
    GetByID(id int64) (*User, error)
}
```

### ২. Usecase (Application Logic)

```go
package usecase

import "my-app/internal/domain"

type UserUsecase struct {
    repo domain.UserRepository
}

func NewUserUsecase(repo domain.UserRepository) *UserUsecase {
    return &UserUsecase{repo: repo}
}

func (u *UserUsecase) RegisterUser(name, email string) (*domain.User, error) {
    if email == "" {
        return nil, domain.ErrInvalidEmail
    }
    
    user := &domain.User{
        Name: name,
        Email: email,
        CreatedAt: time.Now(),
    }
    
    if err := u.repo.Store(user); err != nil {
        return nil, err
    }
    return user, nil
}
```

### ৩. Delivery (Fiber Handler)

```go
package http

import (
    "github.com/gofiber/fiber/v3"
    "my-app/internal/usecase"
)

type UserHandler struct {
    usecase *usecase.UserUsecase
}

func (h *UserHandler) Register(c fiber.Ctx) error {
    type req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    
    var r req
    if err := c.Bind().Body(&r); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    
    user, err := h.usecase.RegisterUser(r.Name, r.Email)
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    
    return c.Status(201).JSON(user)
}
```

## কেন এটি ব্যবহার করবেন?

*   **টেস্টেবল:** আপনি সহজেই `UserRepository` মক করে `UserUsecase` টেস্ট করতে পারেন।
*   **ফ্রেমওয়ার্ক ইন্ডিপেন্ডেন্ট:** আপনি চাইলে Fiber পাল্টে Gin বা Echo ব্যবহার করতে পারেন, বিজনেস লজিকে হাত না দিয়েই।
*   **ডেটাবেস ইন্ডিপেন্ডেন্ট:** আপনি MySQL থেকে MongolianDB তে শিফট করতে পারেন শুধু রিপোজিটরি লেয়ার পরিবর্তন করে।

---
[সূচি](../README.md) | [পরবর্তী: হেক্সাগোনাল আর্কিটেকচার →](02-hexagonal-architecture.md)
