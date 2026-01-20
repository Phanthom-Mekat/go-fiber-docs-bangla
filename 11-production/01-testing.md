# টেস্টিং (Testing)

প্রোডাকশন গ্রেড অ্যাপ্লিকেশনের জন্য টেস্টিং অপরিহার্য। Go Fiber v3 তে `fiber.Ctx` ইন্টারফেস হওয়ার কারণে টেস্টিং এখন আরও সহজ এবং ফ্লেক্সিবল।

## Unit Testing

ইউনিট টেস্টিংয়ে আমরা নির্দিষ্ট ফাংশন বা হ্যান্ডলারকে আইসোলেশনে টেস্ট করি।

### Handler Testing

Fiber v3 তে আমরা `app.Test` মেথড ব্যবহার করে সম্পূর্ণ সার্ভার রান না করেই হ্যান্ডলার টেস্ট করতে পারি।

```go
package tests

import (
    "net/http/httptest"
    "testing"
    "time"

    "github.com/gofiber/fiber/v3"
    "github.com/stretchr/testify/assert"
)

func Test_UserRoute(t *testing.T) {
    // ১. অ্যাপ ইনিশিয়ালাইজ করুন
    app := fiber.New()
    
    // ২. রাউট ডিফাইন করুন
    app.Get("/user", func(c fiber.Ctx) error {
        return c.Status(200).JSON(fiber.Map{"name": "John"})
    })

    // ৩. টেস্ট রিকোয়েস্ট তৈরি করুন
    req := httptest.NewRequest("GET", "/user", nil)
    
    // ৪. অ্যাপ টেস্ট রান করুন (v3 স্টাইল)
    // Timeout কনফিগার করা জরুরি
    resp, err := app.Test(req, fiber.TestConfig{
        Timeout:       2 * time.Second,
        FailOnTimeout: true,
    })

    // ৫. অ্যাসারশন (Assertion)
    assert.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}
```

## Integration Testing

ইন্টিগ্রেশন টেস্টিংয়ে আমরা ডেটাবেস বা অন্যান্য ডিপেন্ডেন্সিসহ পুরো ফ্লো টেস্ট করি। টিপিক্যাল Go প্রোজেক্টে আমরা `testcontainers-go` ব্যবহার করতে পারি রিয়েল ডেটাবেস স্পিন আপ করার জন্য।

```go
func Test_CreateUser_Integration(t *testing.T) {
    // সেটআপ (Mock DB বা Test Container)
    db := setupTestDB()
    defer db.Close()
    
    app := fiber.New()
    userHandler := handlers.NewUserHandler(db)
    app.Post("/users", userHandler.Create)

    payload := strings.NewReader(`{"name":"Test User","email":"test@example.com"}`)
    req := httptest.NewRequest("POST", "/users", payload)
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req)
    
    assert.NoError(t, err)
    assert.Equal(t, 201, resp.StatusCode)
}
```

## Mocking `fiber.Ctx`

যেহেতু v3 তে `fiber.Ctx` একটি ইন্টারফেস, তাই আমরা `mockery` বা কাস্টম মক ব্যবহার করে সহজেই কনটেক্সট মক করতে পারি। এটি মিডলওয়্যার বা ইউটিলিটি ফাংশন টেস্ট করার জন্য খুব কাজের।

```go
// go run github.com/vektra/mockery/v2@latest --name=Ctx --dir=$(go list -m -f '{{.Dir}}' github.com/gofiber/fiber/v3)
type MockCtx struct {
    fiber.Ctx
    // আপনার প্রয়োজনীয় মেথডগুলো ওভাররাইড করুন
}

func (m *MockCtx) Method() string {
    return "GET"
}
```

## বেস্ট প্র্যাকটিস (Best Practices)

1.  **Table Driven Tests:** Go-এর ইডিয়াম্যাটিক স্টাইল ফলো করুন।
2.  **Test Coverage:** `go test -cover` দিয়ে কভারেজ চেক করুন।
3.  **Race Condition Check:** `go test -race` ফ্ল্যাগ ব্যবহার করে কনকারেন্সি বাগ ধরুন।
4.  **Helper Functions:** টেস্ট সেটআপ এবং টিয়ারডাউনের জন্য হেল্পার ফাংশন ব্যবহার করুন।

---
[← পেছনে](../09-advanced/06-deployment.md) | [সূচি](../README.md) | [সামনে: লগিং ও অবজার্ভেবিলিটি →](02-logging-observability.md)
