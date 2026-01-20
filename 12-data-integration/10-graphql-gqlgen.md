# GraphQL (Gqlgen) ইন্টিগ্রেশন

REST API এর পাশাপাশি GraphQL এখন জনপ্রিয়। Go-তে স্কিমা-ফার্স্ট অ্যাপ্রোচের জন্য `99designs/gqlgen` সেরা লাইব্রেরি। Fiber-এর `adaptor` ব্যবহার করে আমরা সহজেই এটি ইন্টিগ্রেট করতে পারি।

## ১. Gqlgen ইনিশিয়ালাইজেশন

প্রোজেক্ট ফোল্ডারে:
```bash
go run github.com/99designs/gqlgen init
```
এটি `graph` ফোল্ডার এবং প্রয়োজনীয় ফাইল জেনারেট করবে।

## ২. Fiber এর সাথে কানেক্ট করা

Gqlgen একটি `http.Handler` রিটার্ন করে। Fiber v3 এর `adaptor` প্যাকেজ দিয়ে আমরা এটি ব্যবহার করতে পারি।

```go
package main

import (
    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/adaptor"
    
    "my-app/graph"
    "my-app/graph/generated"
)

func main() {
    app := fiber.New()

    // ১. GraphQL সার্ভার তৈরি
    srv := handler.NewDefaultServer(
        generated.NewExecutableSchema(generated.Config{Resolvers: &graph.Resolver{}}),
    )

    // ২. Query হ্যান্ডলার (POST /query)
    // Adaptor ব্যবহার করে standard handler কে fiber handler এ রূপান্তর
    app.Post("/query", adaptor.HTTPHandler(srv))

    // ৩. Playground (ব্রাউজারে টেস্ট করার জন্য)
    app.Get("/", adaptor.HTTPHandler(playground.Handler("GraphQL playground", "/query")))

    app.Listen(":3000")
}
```

## ৩. Context ম্যানেজমেন্ট

Fiber এর `UserContext` এবং Gqlgen এর কনটেক্সট আলাদা হতে পারে। আপনি যদি Fiber মিডলওয়্যার থেকে ডেটা (যেমন User ID) GraphQL রিজলভারে পাস করতে চান, তাহলে একটি র‌্যাপার মিডলওয়্যার লাগবে।

```go
// মিডলওয়্যার যা Fiber কনটেক্সট ভ্যালুকে স্ট্যান্ডার্ড Go কনটেক্সটে ইনজেক্ট করে
app.Use(func(c fiber.Ctx) error {
    ctx := context.WithValue(c.UserContext(), "user_id", c.Locals("user_id"))
    c.SetUserContext(ctx)
    return c.Next()
})
```

---
[← আগের: Swagger](09-api-docs-swagger.md) | [সূচি](../README.md) | [পরবর্তী: gRPC →](11-grpc-integration.md)
