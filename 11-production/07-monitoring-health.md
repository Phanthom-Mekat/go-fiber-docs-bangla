# মনিটরিং ও হেলথ চেক (Monitoring & Health Checks)

একটি রিলায়েবল প্রোডাকশন অ্যাপ্লিকেশনের জন্য অ্যাপের ইন্টারনাল অবস্থা জানা জরুরি। "এটা কি চলছে?" - এই প্রশ্নের উত্তর দেয় হেলথ চেক, আর "কেমন চলছে?" - এর উত্তর দেয় ম্যাট্রিক্স (Metrics)।

## হেলথ চেক (Health Checks)

কুবারনেটিস বা লোড ব্যালান্সারের জন্য অ্যাপটি ট্রাফিক নেওয়ার জন্য প্রস্তুত কিনা তা জানতে `/health` বা `/ready` এন্ডপয়েন্ট ব্যবহার করা হয়।

```go
func main() {
    app := fiber.New()
    
    // Liveness Probe: অ্যাপ ক্র্যাশ করেছে কিনা
    app.Get("/healthz", func(c fiber.Ctx) error {
        return c.SendStatus(200)
    })
    
    // Readiness Probe: অ্যাপ ট্রাফিক হ্যান্ডেল করতে প্রস্তুত কিনা
    // (যেমন: DB কানেক্টেড কিনা)
    app.Get("/ready", func(c fiber.Ctx) error {
        if err := db.Ping(); err != nil {
            return c.Status(503).SendString("Database unavailable")
        }
        return c.SendStatus(200)
    })
}
```

## ম্যাট্রিক্স (Metrics with Prometheus)

অ্যাপ্লিকেশনের পারফরমেন্স গ্রাফ (Request Count, Latency, Error Rate) দেখার জন্য `Prometheus` এবং `Grafana` সেরা জুটি। Fiber-এর জন্য `monitor` বা `otelfiber` মিডলওয়্যার ব্যবহার করা হয়।

### Monitor Middleware (Basic)

```go
import "github.com/gofiber/fiber/v3/middleware/monitor"

app.Get("/metrics", monitor.New(monitor.Config{
    Title: "My App Metrics",
}))
```
এটি একটি সাধারণ HTML ড্যাশবোর্ড দেখাবে।

### Prometheus Integration (Advanced)

প্রোডাকশনের জন্য আমরা সাধারণত একটি এক্সপোর্টার ব্যবহার করি যা Prometheus স্ক্র্যাপ করতে পারে।

> **Note:** Fiber v2 তে `fiber-prometheus` ছিল। v3 তে আমরা সাধারণত ওপেনটেলিমেট্রি (OpenTelemetry) ব্যবহার করতে উৎসাহিত করি, অথবা কাস্টম অ্যাডাপ্টার।

যদি আপনি `expvar` (Go-এর বিল্ট-ইন ম্যাট্রিক্স) ব্যবহার করতে চান:

```go
import "github.com/gofiber/fiber/v3/middleware/expvar"

app.Use("/debug/vars", expvar.New())
```

## কি ট্র্যাক করবেন? (Golden Signals)

গুগলের SRE বই অনুযায়ী ৪টি গোল্ডেন সিগন্যাল মনিটর করা উচিত:

1.  **Latency:** রিকোয়েস্ট সার্ভ করতে কত সময় লাগছে।
2.  **Traffic:** কতগুলো রিকোয়েস্ট আসছে (RPS)।
3.  **Errors:** কতগুলো রিকোয়েস্ট ফেইল করছে (5xx errors)।
4.  **Saturation:** আপনার সার্ভার কতটা লোডেড (CPU/Memory usage)।

---
[← আগের: সিকিউরিটি](06-security-best-practices.md) | [সূচি](../README.md) | [পরবর্তী: ডকার কনটেইনার →](08-docker-container.md)
