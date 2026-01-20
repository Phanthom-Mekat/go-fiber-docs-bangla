# রেজিলিয়েন্স: সার্কিট ব্রেকার (Circuit Breaker)

মাইক্রোসারভিসে একটি সার্ভিস ডাউন থাকলে যাতে পুরো সিস্টেম ধসে না পড়ে, তার জন্য "Circuit Breaker" প্যাটার্ন ব্যবহৃত হয়।

Fiber এর একটি অফিশিয়াল কন্ট্রিবিউশন আছে: `github.com/gofiber/contrib/circuitbreaker`। কিন্তু এটি মূলত ইনকামিং রিকোয়েস্টের জন্য। আউটগোয়িং রিকোয়েস্টের জন্য (যেখানে আমাদের আসলে সার্কিট ব্রেকার দরকার) `sony/gobreaker` জনপ্রিয়।

## ১. ইনকামিং রিকোয়েস্টের জন্য (Fiber Middleware)

যদি আপনার API তে অতিরিক্ত এরর হতে থাকে, আপনি সাময়িকভাবে রিকোয়েস্ট নেওয়া বন্ধ করতে পারেন।

```go
import "github.com/gofiber/contrib/circuitbreaker"

app.Use(circuitbreaker.New(circuitbreaker.Config{
    Next: func(c fiber.Ctx) bool {
        // কোন কোন পাথে সার্কিট ব্রেকার কাজ করবে না
        return c.Path() == "/health"
    },
    MaxFailures: 3,         // ৩ বার ফেইল করলে সার্কিট ওপেন হবে
    Timeout:     10 * time.Second, // ১০ সেকেন্ড পর আবার ট্রাই করবে (Half-Open)
    Handler: func(c fiber.Ctx) error {
        return c.Status(503).SendString("Service unavailable currently")
    },
}))
```

## ২. আউটগোয়িং রিকোয়েস্টের জন্য (gobreaker)

ধরা যাক আপনার Fiber অ্যাপ `PaymentService` কল করছে।

```go
import "github.com/sony/gobreaker"

var cb *gobreaker.CircuitBreaker

func InitCB() {
    settings := gobreaker.Settings{
        Name:    "PaymentService",
        Timeout: 5 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            // যদি ফেইলিউর রেশিও ৫০% এর বেশি হয়
            return counts.FailureRatio > 0.5
        },
    }
    cb = gobreaker.NewCircuitBreaker(settings)
}

func CallPaymentAPI() error {
    _, err := cb.Execute(func() (interface{}, error) {
        // আসল API কল
        resp, err := http.Get("http://payment-service/charge")
        if err != nil {
            return nil, err
        }
        if resp.StatusCode >= 500 {
            return nil, errors.New("server error")
        }
        return resp, nil
    })
    
    return err
}
```

যখন সার্কিট "Open" অবস্থায় থাকবে, তখন `CallPaymentAPI` কল করলে সাথে সাথে এরর দিবে, নেটওয়ার্ক কল করবে না। এতে আপনার অ্যাপ ফাস্ট ফেইল করবে এবং পেমেন্ট সার্ভিস রিকভার হওয়ার সুযোগ পাবে।

---
[← আগের: ইভেন্ট সোর্সিং](06-event-sourcing-basics.md) | [সূচি](../README.md) | [পরবর্তী: রিট্রাই মেকানিজম →](08-retry-backoff-mechanism.md)
