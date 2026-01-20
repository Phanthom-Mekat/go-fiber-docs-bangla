# লগিং ও অবজার্ভেবিলিটি (Logging & Observability)

একটি হাই-পারফরম্যান্স প্রোডাকশন সিস্টেমে শুধু "print" করা যথেষ্ট নয়। আপনার প্রয়োজন স্ট্রাকচার্ড লগিং (Structured Logging) এবং ট্রেসিং (Tracing) যা আপনাকে সিস্টেমের প্রতিটি রিকোয়েস্টের লাইফসাইকেল বুঝতে সাহায্য করবে।

## স্ট্রাকচার্ড লগিং (Structured Logging)

Go-তে বর্তমানে `slog` (Go 1.21+) বা `zerolog` (High Performance) ব্যবহার করা স্ট্যান্ডার্ড। Fiber-এর জন্য আমরা `zerolog` রিকমেন্ড করি কারণ এটি `fasthttp`-এর সাথে ভালোভাবে পারফর্ম করে।

### Zerolog সেটআপ

```go
package main

import (
    "os"
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func main() {
    // ১. গ্লোবাল লগার কনফিগারেশন
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    log.Logger = log.Output(zerolog.ConsoleWriter{Out: os.Stderr})

    app := fiber.New()

    // ২. Fiber Logger Middleware
    app.Use(logger.New(logger.Config{
        Format: "[${time}] ${status} - ${latency} ${method} ${path}\n",
        Output: os.Stdout,
    }))

    app.Get("/", func(c fiber.Ctx) error {
        // ৩. স্ট্রাকচার্ড লগিং ব্যবহার করুন
        log.Info().
            Str("path", c.Path()).
            Str("ip", c.IP()).
            Msg("Hello World Request")
            
        return c.SendString("Check logs")
    })

    app.Listen(":3000")
}
```

## ওপেনটেলিমেট্রি (OpenTelemetry / OTel)

ডিস্ট্রিবিউটেড ট্রেসিংয়ের জন্য OpenTelemetry ইন্ডাস্ট্রি স্ট্যান্ডার্ড। এটি আপনাকে রিকোয়েস্টের পুরো জার্নি ট্র্যাক করতে দেয় (যেমন: Client -> API -> Database -> Cache)।

### Fiber OTel Integration

```go
import "github.com/gofiber/contrib/otelfiber/v3"

// ... ট্রেসার প্রভাইডার ইনিশিয়ালাইজেশন কোড ...

app := fiber.New()

// মিডলওয়্যার হিসেবে সবার শুরুতে অ্যাড করুন
app.Use(otelfiber.New(otelfiber.Config{
    TracerProvider: tp, // আপনার কনফিগার করা ট্রেসার প্রভাইডার
    SpanNameFormatter: func(c fiber.Ctx) string {
        return fmt.Sprintf("%s %s", c.Method(), c.Route().Path)
    },
}))
```

## রিকোয়েস্ট আইডি (Request ID)

লগ এবং ট্রেসের মধ্যে সম্পর্ক স্থাপনের জন্য প্রতিটি রিকোয়েস্টে একটি ইউনিক আইডি থাকা জরুরি।

```go
import "github.com/gofiber/fiber/v3/middleware/requestid"

app.Use(requestid.New(requestid.Config{
    Header: "X-Trace-ID",
    Generator: func() string {
        return uuid.New().String()
    },
}))

// হ্যান্ডলারে আইডি নেওয়া
app.Get("/", func(c fiber.Ctx) error {
    rid := requestid.FromContext(c)
    log.Info().Str("request_id", rid).Msg("Processing request")
    return c.SendString(rid)
})
```

## বেস্ট প্র্যাকটিস

1.  **JSON Format:** প্রোডাকশনে লগের ফরম্যাট সবসময় JSON রাখুন যাতে Datadog/ELK স্ট্যাক সহজে পার্স করতে পারে।
2.  **Log Level:** `Debug`, `Info`, `Warn`, `Error` লেভেলগুলো সঠিক ভাবে ব্যবহার করুন। প্রোডাকশনে সাধারণত `Info` লেভেল অন থাকে।
3.  **No Sensitive Data:** পাসওয়ার্ড, টোকেন বা PII (Personal Identifiable Information) লগ করবেন না।
4.  **Contextual Logging:** সবসময় `request_id` সব লগে অ্যাড করুন।

---
[← আগের: টেস্টিং](01-testing.md) | [সূচি](../README.md) | [পরবর্তী: কনফিগারেশন →](03-configuration.md)
