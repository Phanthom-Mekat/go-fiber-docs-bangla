# রিট্রাই এবং ব্যাকঅফ মেকানিজম (Retry & Backoff)

নেটওয়ার্ক গ্লিচ বা সাময়িক ডাউনটাইমের কারণে এক্সটার্নাল সার্ভিস কল ফেইল করতে পারে। সাথে সাথে হাল না ছেড়ে "Retry" করা ভালো প্র্যাকটিস। তবে রিট্রাই করার সময় "Exponential Backoff" (অপেক্ষা করার সময় ধীরে ধীরে বাড়ানো) ব্যবহার করা উচিত যাতে সার্ভার ওভারলোড না হয়।

জনপ্রিয় লাইব্রেরি: `github.com/avast/retry-go`

## ১. ব্যাসিক রিট্রাই

```go
package external

import (
    "net/http"
    "time"
    "github.com/avast/retry-go"
)

func FetchData() ([]byte, error) {
    var body []byte
    
    err := retry.Do(
        func() error {
            resp, err := http.Get("https://api.example.com/data")
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            
            // যদি 500 বা 502/503 হয়, তবেই রিট্রাই করব
            if resp.StatusCode >= 500 {
                return errors.New("server error")
            }
            
            // ... body read logic ...
            return nil
        },
        // কনফিগারেশন
        retry.Attempts(3), // মোট ৩ বার চেষ্টা করবে
        retry.Delay(1 * time.Second), // ১ সেকেন্ড গ্যাপ
        retry.OnRetry(func(n uint, err error) {
            log.Printf("Retry #%d: %s\n", n, err)
        }),
    )
    
    return body, err
}
```

## ২. Fiber মিডলওয়্যার (ক্লায়েন্টের জন্য)

Fiber ক্লায়েন্টদের জন্যও রিট্রাই সুবিধা দিতে পারে (যদি আপনার API গেটওয়ে হিসেবে কাজ করে)। `middleware/idempotency` এর সাথে এটি ব্যবহার করা নিরাপদ।

তবে সাধারণত রিট্রাই লজিক ক্লায়েন্ট সাইডেই (Frontend বা Mobile App) থাকা উচিত। ব্যাকএন্ডে আমরা শুধু `429 Too Many Requests` বা `503 Service Unavailable` দিয়ে ইঙ্গিত দিতে পারি।

## ৩. জিটার (Jitter)

সবাই যদি একই সময়ে (যেমন ১ সেকেন্ড পর) রিট্রাই করে, তাহলে "Thundering Herd" সমস্যা হতে পারে। তাই ডিলে-র সাথে র্যান্ডম সময় (Jitter) যোগ করা উচিত। `retry-go` ডিফল্টভাবে ব্যাকঅফ এবং জিটার সাপোর্ট করে।

```go
retry.DelayType(retry.BackOffDelay), // ১সে, ২সে, ৪সে...
```

---
[← আগের: সার্কিট ব্রেকার](07-resilience-circuit-breaker.md) | [সূচি](../README.md) | [পরবর্তী: ফিচার ফ্ল্যাগ →](09-feature-flags.md)
