# CQRS প্যাটার্ন (Command Query Responsibility Segregation)

CQRS প্যাটার্নে আমরা ডেটা রিড (Query) এবং রাইট (Command) অপারেশনকে আলাদা মডেলে ভাগ করি। এটি হাই-স্কেল অ্যাপ্লিকেশনের জন্য খুব কার্যকর।

## ধারণা (Concepts)

*   **Command:** ডেটা পরিবর্তন করে (Create, Update, Delete)। যেমন: `RegisterUserCmd`. কোনো ডেটা রিটার্ন করে না (শুধু এরর বা ID ছাড়া)।
*   **Query:** ডেটা রিড করে (Get, List)। ডেটা পরিবর্তন করে না। যেমন: `GetUserByIdQuery`.

## সাধারণ ইমপ্লিমেন্টেশন

আমরা `MediatR` (C#) স্টাইলে Go তে চ্যানেল বা ইন্টারফেস ব্যবহার করে এটি করতে পারি।

### ১. কমান্ড এবং হ্যান্ডলার

```go
// Command
type CreateUserCmd struct {
    Name  string
    Email string
}

// Handler Interface (Generic)
type CommandHandler[C any] interface {
    Handle(ctx context.Context, cmd C) error
}

// Implementation
type CreateUserHandler struct {
    repo UserRepository
}

func (h *CreateUserHandler) Handle(ctx context.Context, cmd CreateUserCmd) error {
    user := &User{Name: cmd.Name, Email: cmd.Email}
    return h.repo.Save(user)
}
```

### ২. কুয়েরি এবং হ্যান্ডলার

```go
// Query
type GetUserQuery struct {
    ID string
}

// Result (Query Model - DTO)
type UserDTO struct {
    Name string
    Role string
}

type QueryHandler[Q any, R any] interface {
    Handle(ctx context.Context, query Q) (R, error)
}
```

## Fiber এ ইন্টিগ্রেশন

Fiber হ্যান্ডলার ডেসপ্যাচার হিসেবে কাজ করে।

```go
func (api *UserAPI) CreateUser(c fiber.Ctx) error {
    var cmd CreateUserCmd
    if err := c.Bind().Body(&cmd); err != nil {
        return err
    }
    
    // কমান্ড হ্যান্ডলার কল করা
    if err := api.createHandler.Handle(c.UserContext(), cmd); err != nil {
        return c.Status(500).SendString(err.Error())
    }
    
    return c.SendStatus(201)
}

func (api *UserAPI) GetUser(c fiber.Ctx) error {
    query := GetUserQuery{ID: c.Params("id")}
    
    // কুয়েরি হ্যান্ডলার কল করা
    // এখানে আমরা সরাসরি অপ্টিমাইজড রিড-মডেল (যেমন Redis বা Elastic) থেকে ডেটা আনতে পারি
    userView, err := api.queryHandler.Handle(c.UserContext(), query)
    if err != nil {
        return err
    }
    
    return c.JSON(userView)
}
```

## সুবিধা

*   **অপ্টিমাইজেশন:** রিড এবং রাইট এর জন্য আলাদা ডাটাবেস ব্যবহার করা সম্ভব (যেমন: Write to Postgres, Read from ElasticSearch)।
*   **স্কেলিং:** রিড এবং রাইট অংশ আলাদাভাবে স্কেল করা যায়।

---
[← আগের: DI](04-dependency-injection.md) | [সূচি](../README.md) | [পরবর্তী: ইভেন্ট সোর্সিং →](06-event-sourcing-basics.md)
