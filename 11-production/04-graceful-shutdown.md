# গ্রেসফুল শাটডাউন (Graceful Shutdown)

সার্ভার হঠাৎ বন্ধ হলে যাতে চলমান রিকোয়েস্টগুলো ফেইল না করে বা ডাটা লস না হয়, তার জন্য "Graceful Shutdown" জরুরি। Go Fiber v3 তে এটি ইমপ্লিমেন্ট করা খুবই সহজ।

## বেসিক কনসেপ্ট

যখন আমরা `CTRL+C` বা কুবারনেটিস থেকে কোনো টার্মিনেশন সিগন্যাল (SIGINT/SIGTERM) পাই, তখন আমরা:
1.  নতুন রিকোয়েস্ট নেওয়া বন্ধ করব।
2.  চলমান রিকোয়েস্টগুলো শেষ হওয়ার জন্য অপেক্ষা করব (Timeout সহ)।
3.  ডাটাবেস কানেকশন ক্লোজ করব।
4.  অ্যাপ বন্ধ করব।

## v3 ইমপ্লিমেন্টেশন

Fiber v3 এর নতুন হুক সিস্টেম এবং `app.ShutdownWithContext` এর মাধ্যমে আমরা এটি করতে পারি।

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gofiber/fiber/v3"
)

func main() {
    app := fiber.New()

    app.Get("/", func(c fiber.Ctx) error {
        time.Sleep(2 * time.Second) // লং রানিং প্রসেস সিমুলেশন
        return c.SendString("Done")
    })

    // ১. শাটডাউন হুক রেজিস্টার করা (অপশনাল কিন্তু রেকমেন্ডেড)
    app.Hooks().OnShutdown(func() error {
        log.Println("Shutting down... Cleaning up resources")
        // এখানে ডাটাবেস বা Redis কানেকশন ক্লোজ করুন
        // db.Close()
        return nil
    })

    // ২. সার্ভার রান করা (goroutine-এ)
    go func() {
        if err := app.Listen(":3000"); err != nil {
            log.Panic(err)
        }
    }()

    // ৩. সিগন্যাল শোনা (Listen for syscall signals)
    c := make(chan os.Signal, 1)
    // SIGINT (Ctrl+C) এবং SIGTERM (Docker/K8s stop) শোনা
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)

    // ব্লকিং - সিগন্যাল আসা পর্যন্ত অপেক্ষা
    <-c 
    log.Println("Graceful shutdown started...")

    // ৪. টাইমআউট সহ শাটডাউন
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := app.ShutdownWithContext(ctx); err != nil {
        log.Printf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exiting")
}
```

## কুবারনেটিস ও ডকার

কুবারনেটিসে (Kubernetes) যখন পড টার্মিনেট হয়, তখন এটি প্রথমে `SIGTERM` পাঠায়। আমাদের অ্যাপ যদি এই সিগন্যাল হ্যান্ডেল না করে, তাহলে কুবারনেটিস হার্ড কিল (`SIGKILL`) করবে যা ডেটা লস ঘটাতে পারে।

উপরের কোডটি নিশ্চিত করে যে আপনার অ্যাপ কুবারনেটিস এনভায়রনমেন্টে নিরাপদে বন্ধ হবে।

---
[← আগের: কনফিগারেশন](03-configuration.md) | [সূচি](../README.md) | [পরবর্তী: পারফরমেন্স টিউনিং →](05-performance-tuning.md)
