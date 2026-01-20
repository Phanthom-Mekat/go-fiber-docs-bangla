# সার্ভারলেস Fiber (AWS Lambda)

Fiber কে আপনি ট্র্যাডিশনাল সার্ভারের বদলে AWS Lambda বা Google Cloud Function এও রান করতে পারেন। এতে স্কেলিং নিয়ে ভাবতে হয় না এবং খরচও কম (Pay-per-use)।

AWS Lambda এর জন্য আমরা `awslabs/aws-lambda-go-api-proxy` ব্যবহার করব।

## ১. সেটআপ

```bash
go get github.com/aws/aws-lambda-go/lambda
go get github.com/awslabs/aws-lambda-go-api-proxy/fiber
```

## ২. অ্যাডাপ্টার তৈরি (main.go)

```go
package main

import (
    "context"
    "log"

    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    fiberadapter "github.com/awslabs/aws-lambda-go-api-proxy/fiber"
    "github.com/gofiber/fiber/v2" 
    // নোট: v3 এর অ্যাডাপ্টার এখনো অফিসিয়ালি রিলিজ নাও হতে পারে, 
    // তবে v2 অ্যাডাপ্টার সাধারণত কাজ করে অথবা কাস্টম র‍্যাপার লেখা যায়।
)

var fiberLambda *fiberadapter.FiberLambda

// init() ফাংশনে অ্যাপ একবারই ইনিশিয়ালাইজ হবে (Cold Start)
func init() {
    log.Println("Fiber cold start")
    app := fiber.New()

    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello form AWS Lambda!")
    })
    
    app.Get("/user/:name", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"user": c.Params("name")})
    })

    fiberLambda = fiberadapter.New(app)
}

// Handler যা Lambda ইভেন্ট রিসিভ করে Fiber এ পাঠায়
func Handler(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // যদি কোনো প্রক্সি রিকোয়েস্ট না হয়, সরাসরি রেসপন্স দিতে পারেন
    return fiberLambda.ProxyWithContext(ctx, req)
}

func main() {
    // Lambda রান করা
    lambda.Start(Handler)
}
```

## ৩. ডিপ্লয়মেন্ট

কোডটি লিনাক্স বাইনারি হিসেবে কম্পাইল করে জিপ করতে হবে এবং AWS Lambda তে আপলোড করতে হবে। হ্যান্ডলার হিসেবে `main` (বাইনারির নাম) সেট করুন।

```bash
GOOS=linux GOARCH=amd64 go build -o main main.go
zip main.zip main
```

এভাবে আপনি আপনার পুরো Fiber অ্যাপকে (রাউটিং, মিডলওয়্যার সহ) সার্ভারলেস ফাংশন হিসেবে চালাতে পারেন।

---
[← আগের: mTLS](11-mutual-tls-mtls.md) | [সূচি](../README.md)
