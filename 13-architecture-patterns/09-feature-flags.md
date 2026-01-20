# ফিচার ফ্ল্যাগ (Feature Flags)

কোড ডিপ্লয় না করেই অ্যাপ্লিকেশনের কোনো ফিচার অন বা অফ করার প্রযুক্তি হলো ফিচার ফ্ল্যাগ। এটি A/B টেস্টিং এবং ক্যানারি রিলিজের জন্য অপরিহার্য।

স্ট্যান্ডার্ড: **OpenFeature**। সিম্পল ইমপ্লিমেন্টেশনের জন্য আমরা Redis বা এনভায়রনমেন্ট ভেরিয়েবল ব্যবহার করতে পারি।

## ১. সিম্পল Redis ইমপ্লিমেন্টেশন

```go
func isFeatureEnabled(ctx context.Context, featureName string) bool {
    // Redis চেক
    val, err := redisClient.Get(ctx, "feature:"+featureName).Result()
    if err == nil && val == "true" {
        return true
    }
    return false
}

// Fiber হ্যান্ডলার
func NewFeatureHandler(c fiber.Ctx) error {
    if !isFeatureEnabled(c.UserContext(), "new-dashboard") {
        return c.Status(404).SendString("Feature not available")
    }
    
    return c.SendString("Welcome to the new dashboard!")
}
```

## ২. OpenFeature (স্ট্যান্ডার্ড পদ্ধতি)

Go-তে `open-feature/go-sdk` ব্যবহার করা হয়।

```go
import (
    "github.com/open-feature/go-sdk/openfeature"
)

func main() {
    // 1. প্রোভাইডার সেটআপ (এখানে NoOp, বাস্তবে LaunchDarkly বা Flagd হবে)
    openfeature.SetProvider(openfeature.NoopProvider{})
    client := openfeature.NewClient("my-fiber-app")

    app.Get("/promo", func(c fiber.Ctx) error {
        // 2. ফ্ল্যাগ চেক করা
        // ইউজার কন্টেক্সট (targeting এর জন্য)
        evalCtx := openfeature.NewEvaluationContext(
            c.Get("X-User-ID"),
            map[string]interface{}{
                "email": c.Get("X-User-Email"),
            },
        )

        showPromo, _ := client.BooleanValue(
            context.Background(), 
            "show-summer-promo", 
            false, 
            evalCtx,
        )

        if showPromo {
            return c.JSON(fiber.Map{"promo": "50% OFF"})
        }
        return c.JSON(fiber.Map{"promo": nil})
    })
}
```

ফিচার ফ্ল্যাগ ব্যবহার করে আপনি মেইন ব্রাঞ্চে কোড মার্জ করতে পারেন কিন্তু ফিচারটি ইউজারদের থেকে লুকিয়ে রাখতে পারেন যতক্ষণ না এটি সম্পূর্ণ প্রস্তুত হয়।

---
[← আগের: রিট্রাই](08-retry-backoff-mechanism.md) | [সূচি](../README.md) | [পরবর্তী: SSE →](10-server-sent-events.md)
