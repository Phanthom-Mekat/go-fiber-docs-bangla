# পারফরমেন্স টিউনিং (Performance Tuning)

Go Fiber ডিফল্টভাবেই অনেক দ্রুত, কিন্তু হাই-স্কেল প্রোডাকশন এনভায়রনমেন্টে আমরা আরও অপ্টিমাইজেশন করতে পারি।

## ১. JSON এনকোডার (Sonic)

Go-এর বিল্ট-ইন `encoding/json` কিছুটা ধীরগতির। Fiber v3 তে আমরা `sonic` বা `goccy/go-json` ব্যবহার করে ২-৩ গুণ বেশি স্পিড পেতে পারি।

```go
import "github.com/bytedance/sonic"

app := fiber.New(fiber.Config{
    JSONEncoder: sonic.Marshal,
    JSONDecoder: sonic.Unmarshal,
})
```

> **Note:** Sonic শুধুমাত্র `amd64` আর্কিটেকচারে কাজ করে (Linux/macOS/Windows)। ARM (M1/M2) চিপের জন্য `goccy/go-json` ব্যবহার করুন।

## ২. প্রিফর্ক (Prefork)

Fiber-এর `Prefork` ফিচার `SO_REUSEPORT` ব্যবহার করে মাল্টিপল প্রসেস স্পন করে, যা মাল্টি-কোর CPU-র সর্বোচ্চ ব্যবহার নিশ্চিত করে।

```go
app := fiber.New(fiber.Config{
    Prefork: true,
})
```

### Docker ও Prefork

কনটেইনারের ভেতরে Prefork ব্যবহারের সময় সাবধানতা অবলম্বন করতে হবে। লিনাক্সে PID 1 সমস্যা এড়াতে `dumb-init` বা `tini` ব্যবহার করা বাধ্যতামূলক।

**Dockerfile Example:**
```dockerfile
# ... build stage ...
FROM alpine:latest
RUN apk add --no-cache dumb-init
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/app/server"]
```

## ৩. কনকারেন্সি সেটিংস

আপনার সার্ভারের রিসোর্স অনুযায়ী কনফিগারেশন টিউন করুন:

```go
app := fiber.New(fiber.Config{
    Concurrency: 256 * 1024, // ডিফল্ট: 256k
    ReadTimeout: 5 * time.Second,
    WriteTimeout: 10 * time.Second,
})
```

## ৪. প্রফাইল করা (Profiling with pprof)

আপনার অ্যাপ কোথায় স্লো হচ্ছে তা বের করতে `pprof` সেরা টুল।

```go
import (
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/pprof"
)

func main() {
    app := fiber.New()
    
    // pprof মিডলওয়্যার (শুধুমাত্র ডেভ/প্রাইভেট নেটওয়ার্কে)
    app.Use(pprof.New()) 
    
    app.Listen(":3000")
}
```

**ব্যবহার:**
ব্রাউজারে `http://localhost:3000/debug/pprof/` এ যান অথবা টার্মিনালে রান করুন:
```bash
go tool pprof http://localhost:3000/debug/pprof/heap
```

## ৫. মেমরি টিউনিং

Go বা Fiber সাধারণত একটু বেশি মেমরি ব্যবহার করতে পারে যদি GC (Garbage Collector) ঘন ঘন রান না করে। আপনি `GOGC` এনভায়রনমেন্ট ভেরিয়েবল টিউন করে মেমরি বনাম CPU-র ট্রেড-অফ কন্ট্রোল করতে পারেন।

- `GOGC=100` (Default)
- `GOGC=50` (Less memory, more CPU usage)

এছাড়াও স্ট্রিং কনক্যাটেনেশনের বদলে `strings.Builder` বা `[]byte` বাফার ব্যবহার করুন।

---
[← আগের: গ্রেসফুল শাটডাউন](04-graceful-shutdown.md) | [সূচি](../README.md) | [পরবর্তী: সিকিউরিটি →](06-security-best-practices.md)
