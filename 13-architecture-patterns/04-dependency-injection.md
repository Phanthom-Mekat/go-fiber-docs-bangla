# ডিপেন্ডেন্সি ইনজেকশন (Dependency Injection - DI)

DI হলো এমন একটি টেকনিক যেখানে অবজেক্টগুলো তাদের ডিপেন্ডেন্সি নিজেরা তৈরি না করে বাইরে থেকে গ্রহণ করে। এটি টেস্টেবিলিটি এবং মডুলারিটি বাড়ায়।

## ১. ম্যানুয়াল DI (Constructor Injection)

Go-তে এটি সবচেয়ে জনপ্রিয় এবং সহজ পদ্ধতি।

```go
// Service যার Logger এবং Config দরকার
type UserService struct {
    logger *log.Logger
    config *Config
}

// Constructor দিয়ে ডিপেন্ডেন্সি পাস করা
func NewUserService(l *log.Logger, c *Config) *UserService {
    return &UserService{
        logger: l,
        config: c,
    }
}

func main() {
    // 1. ডিপেন্ডেন্সি তৈরি
    logger := log.New(os.Stdout, "", 0)
    config := &Config{DBUrl: "..."}

    // 2. ইনজেকশন
    service := NewUserService(logger, config)
    
    // 3. ব্যবহার
    app.Post("/users", service.HandleCreate)
}
```

## ২. Uber Fx (DI Framework)

বড় অ্যাপ্লিকেশনে ম্যানুয়াল ওয়্যারিং করা কঠিন হতে পারে। Uber-এর `fx` ফ্রেমওয়ার্ক এটি সহজ করে।

```go
import "go.uber.org/fx"

func main() {
    fx.New(
        // প্রোভাইডার (যিনি অবজেক্ট তৈরি করেন)
        fx.Provide(
            NewLogger,
            NewConfig,
            NewDatabase,
            NewUserService,
            fiber.New, // Fiber App তৈরি
        ),
        // ইনভৌকার (যিনি অবজেক্ট ব্যবহার করে অ্যাপ স্টার্ট করেন)
        fx.Invoke(registerRoutes),
    ).Run()
}

func registerRoutes(app *fiber.App, service *UserService) {
    app.Post("/users", service.HandleCreate)
    
    // Fx Lifecycle Hook ব্যবহার করে Fiber স্টার্ট করা
    // (নরমাল app.Listen এখানে ব্লকিং করতে পারে না, হুক লাগে)
}
```

## ৩. Google Wire (Code Generation)

`google/wire` রানটাইম রিফলেকশন ব্যবহার না করে কোড জেনারেট করে, তাই এটি `fx` এর চেয়ে ফাস্ট এবং নিরাপদ।

```go
// wire.go
func InitializeApp() (*App, error) {
    wire.Build(NewLogger, NewConfig, NewUserService, NewApp)
    return &App{}, nil
}
```
`wire` কমান্ড চালালে এটি `wire_gen.go` ফাইল তৈরি করবে যেখানে ম্যানুয়াল DI কোড লেখা থাকবে।

## রিকমেন্ডেশন

*   **ছোট/মাঝারি প্রজেক্ট:** ম্যানুয়াল DI ব্যবহার করুন। এটি ম্যাজিক-মুক্ত এবং সহজ।
*   **বড় এন্টারপ্রাইজ প্রজেক্ট:** Uber Fx বা Wire ব্যবহার করতে পারেন যদি ডিপেন্ডেন্সি গ্রাফ খুব জটিল হয়।

---
[← আগের: DDD](03-domain-driven-design.md) | [সূচি](../README.md) | [পরবর্তী: CQRS →](05-cqrs-pattern.md)
