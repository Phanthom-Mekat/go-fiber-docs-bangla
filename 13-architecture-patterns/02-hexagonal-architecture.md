# হেক্সাগোনাল আর্কিটেকচার (Hexagonal Architecture)

হেক্সাগোনাল আর্কিটেকচার বা **Ports and Adapters** প্যাটার্ন ক্লিন আর্কিটেকচারের মতোই, তবে এটি "Inside" (Core) এবং "Outside" (Adapters) এর মধ্যে স্পষ্ট পার্থক্য তৈরি করে।

## মূল ধারণা

অ্যাপ্লিকেশনকে একটি হেক্সাগন হিসেবে চিন্তা করুন:
*   **Core:** বিজনেস লজিক।
*   **Ports:** কোর এর সাথে যোগাযোগের ইন্টারফেস (Input/Output)।
*   **Adapters:** পোর্টের ইমপ্লিমেন্টেশন (যেমন: HTTP Adapter, SQL Adapter)।

## ইমপ্লিমেন্টেশন

### ১. Port (Interface)

ডোমেইন প্যাকেজে পোর্ট ডিফাইন করা থাকে।

```go
type UserServicePort interface {
    CreateUser(name string) error
}

type UserRepoPort interface {
    Save(user User) error
}
```

### ২. Core (Service)

কোর লজিক ইনকামিং পোর্ট ইমপ্লিমেন্ট করে এবং আউটগোয়িং পোর্ট ব্যবহার করে।

```go
type UserCore struct {
    repo UserRepoPort
}

func (u *UserCore) CreateUser(name string) error {
    // বিজনেস লজিক
    user := User{Name: name}
    return u.repo.Save(user)
}
```

### ৩. Driving Adapter (Primary)

ড্রাইভিং অ্যাডাপ্টার অ্যাপকে চালায় (যেমন: Fiber Handler)।

```go
type HttpAdapter struct {
    service UserServicePort
}

func (h *HttpAdapter) HandleCreateUser(c fiber.Ctx) error {
    name := c.FormValue("name")
    if err := h.service.CreateUser(name); err != nil {
        return c.SendStatus(500)
    }
    return c.SendStatus(201)
}
```

### ৪. Driven Adapter (Secondary)

ড্রিভেন অ্যাডাপ্টার অ্যাপ দ্বারা চালিত হয় (যেমন: PostgreSQL)।

```go
type PostgresAdapter struct {
    db *gorm.DB
}

func (p *PostgresAdapter) Save(user User) error {
    return p.db.Create(&user).Error
}
```

## মেইন ওয়্যারিং (Wiring)

```go
func main() {
    // ১. Driven Adapters
    repo := &PostgresAdapter{db: ConnectDB()}
    
    // ২. Core (Injecting Driven Adapters)
    core := &UserCore{repo: repo}
    
    // ৩. Driving Adapters (Injecting Core via Port)
    httpHandler := &HttpAdapter{service: core}
    
    // ৪. Framework
    app := fiber.New()
    app.Post("/users", httpHandler.HandleCreateUser)
    app.Listen(":8080")
}
```

এই প্যাটার্নটি বড় টিমের জন্য চমৎকার কারণ ডেভেলপাররা একে অপরের কোড ব্রেক না করেই আলাদা অ্যাডাপ্টারে কাজ করতে পারেন।

---
[← আগের: ক্লিন আর্কিটেকচার](01-clean-architecture.md) | [সূচি](../README.md) | [পরবর্তী: ডোমেইন ড্রিভেন ডিজাইন →](03-domain-driven-design.md)
