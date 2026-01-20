# সিকিউরিটি বেস্ট প্র্যাকটিস (Security Best Practices)

ওয়েব অ্যাপ্লিকেশন সিকিউরিটি অবহেলা করার কোনো সুযোগ নেই। Fiber অনেকগুলো বিল্ট-ইন মিডলওয়্যার প্রদান করে যা আপনার অ্যাপকে সুরক্ষিত রাখতে সাহায্য করে।

## ১. হেলমেট (Helmet)

`Helmet` মিডলওয়্যার বিভিন্ন HTTP সিকিউরিটি হেডার সেট করে যা XSS (Cross-Site Scripting), Clickjacking ইত্যাদি আক্রমণ থেকে রক্ষা করে।

```go
import "github.com/gofiber/fiber/v3/middleware/helmet"

app.Use(helmet.New())
```

এটি ডিফল্টভাবে `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection` ইত্যাদি হেডার সেট করে।

## ২. CORS (Cross-Origin Resource Sharing)

আপনার API যদি অন্য ডোমেইন থেকে অ্যাক্সেস করা হয়, তাহলে `CORS` কনফিগার করতে হবে।

> **সতর্কতা:** কখনোই প্রোডাকশনে `AllowOrigins: "*"` এবং `AllowCredentials: true` একসাথে ব্যবহার করবেন না।

```go
import "github.com/gofiber/fiber/v3/middleware/cors"

app.Use(cors.New(cors.Config{
    AllowOrigins: []string{"https://example.com", "https://app.example.com"},
    AllowHeaders: []string{"Origin", "Content-Type", "Accept", "Authorization"},
    AllowMethods: []string{"GET", "POST", "PUT", "DELETE"},
}))
```

## ৩. রেট লিমিটিং (Rate Limiting)

DDoS অ্যাটাক বা ব্রুট ফোর্স অ্যাটাক ঠেকাতে রেট লিমিটিং জরুরি।

```go
import "github.com/gofiber/fiber/v3/middleware/limiter"

app.Use(limiter.New(limiter.Config{
    Max:          100,             // ১০০ রিকোয়েস্ট
    Expiration:   1 * time.Minute, // প্রতি মিনিটে
    KeyGenerator: func(c fiber.Ctx) string {
        return c.IP() // IP অনুযায়ী লিমিট
    },
    LimitReached: func(c fiber.Ctx) error {
        return c.Status(429).SendString("Too many requests, please try again later.")
    },
}))
```

ডিস্ট্রিবিউটেড সিস্টেমের জন্য (যেমন কুবারনেটিস), ইন-মেমরি স্টোরেজের বদলে **Redis** স্টোরেজ ব্যবহার করা উচিত যাতে সব পডের মধ্যে লিমিট শেয়ার হয়।

## ৪. CSRF প্রোটেকশন

যদি আপনার অ্যাপ সেশন বা কুকি বেসড অথেন্টিকেশন ব্যবহার করে, তাহলে CSRF (Cross-Site Request Forgery) প্রোটেকশন মাস্ট।

```go
import "github.com/gofiber/fiber/v3/middleware/csrf"

app.Use(csrf.New(csrf.Config{
    KeyLookup:      "header:X-Csrf-Token", // ক্লায়েন্ট এই হেডার পাঠাবে
    CookieName:     "csrf_",
    CookieSameSite: "Lax",
    Expiration:     1 * time.Hour,
}))
```

## ৫. ইনপুট ভ্যালিডেশন

কখনোই ইউজারের ইনপুট বিশ্বাস করবেন না। SQL Injection বা NoSQL Injection ঠেকাতে সবসময় প্যারামিটারাইজড কোয়েরি বা ORM ব্যবহার করুন। ইনপুট ডেটা ভ্যালিডেট করার জন্য `go-playground/validator` ব্যবহার করুন (যেটি আমরা এরর হ্যান্ডলিং অধ্যায়ে দেখেছি)।

## ৬. সিকিউর কুকি (Secure Cookies)

Fiber v3 অটোমেটিক্যালি `Secure: true` ফ্ল্যাগ সেট করে যদি `SameSite: None` থাকে। প্রোডাকশনে সবসময় HTTPS ব্যবহার করুন।

---
[← আগের: পারফরমেন্স](05-performance-tuning.md) | [সূচি](../README.md) | [পরবর্তী: হেলথ ও মনিটরিং →](07-monitoring-health.md)
