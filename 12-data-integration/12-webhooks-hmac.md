# সিকিউর ওয়েবহুকস (Secure Webhooks with HMAC)

যখন আপনি থার্ড-পার্টি সার্ভিস (যেমন Stripe, GitHub) থেকে ওয়েবহুক রিসিভ করেন, তখন ভেরিফাই করা জরুরি যে রিকোয়েস্টটি সত্যি তাদের কাছ থেকে এসেছে কিনা। এটি সাধারণত HMAC (Hash-based Message Authentication Code) সিগনেচার দিয়ে করা হয়।

## ১. HMAC ভেরিফিকেশন লজিক

ধরা যাক সেন্ডার `X-Signature` হেডারে একটি SHA256 হ্যাশ পাঠাচ্ছে।

```go
package utils

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func CheckSignature(payload []byte, signature string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expectedSignature := hex.EncodeToString(mac.Sum(nil))
    
    // টাইমিং অ্যাটাক প্রতিরোধ করতে hmac.Equal ব্যবহার করুন
    return hmac.Equal([]byte(expectedSignature), []byte(signature))
}
```

## ২. Fiber মিডলওয়্যার হিসেবে ব্যবহার

আমরা একটি কাস্টম মিডলওয়্যার তৈরি করতে পারি যা নির্দিষ্ট রোউটের জন্য সিগনেচার চেক করবে।

> **গুরুত্বপূর্ণ:** সিগনেচার ভেরিফাই করার জন্য আমাদের রিকোয়েস্টের **Raw Body** দরকার। Fiber ডিফল্টভাবে বডি রিড করার অনুমতি দেয়।

```go
func ValidateWebhook(secret string) fiber.Handler {
    return func(c fiber.Ctx) error {
        // ১. হেডার থেকে সিগনেচার নেওয়া
        signature := c.Get("X-Signature")
        if signature == "" {
            return c.Status(401).SendString("Missing signature")
        }

        // ২. বডি রিড করা (Fiber এর c.Body() ক্যাশ করা থাকে)
        payload := c.Body()

        // ৩. ভেরিফাই করা
        if !utils.CheckSignature(payload, signature, secret) {
            return c.Status(401).SendString("Invalid signature")
        }

        // ভেরিফাইড হলে সামনে আগাও
        return c.Next()
    }
}
```

## ৩. রাউটে অ্যাপ্লাই করা

```go
func main() {
    app := fiber.New()
    secret := "my-webhook-secret"

    // শুধুমাত্র এই গ্রুপে ভেরিফিকেশন এপ্লাই হবে
    webhooks := app.Group("/webhooks")
    webhooks.Use(ValidateWebhook(secret))

    webhooks.Post("/payment-success", func(c fiber.Ctx) error {
        // এখানে আসলে আমরা নিশ্চিত যে রিকোয়েস্টটি জেনুইন
        return c.SendString("Payment processed")
    })

    app.Listen(":3000")
}
```

এটি সিকিউরিটি হ্যাক এবং স্পুফিং রিকোয়েস্ট থেকে আপনার সিস্টেমকে রক্ষা করবে।

---
[← আগের: gRPC](11-grpc-integration.md) | [সূচি](../README.md)
