# অথোরাইজেশন এবং RBAC (Casbin)

জটিল পারমিশন ম্যানেজমেন্টের জন্য (যেমন: Admin সব পারবে, Manager এডিট করতে পারবে, User শুধু দেখতে পারবে) `Casbin` সেরা সমাধান।

Fiber v3 এর জন্য আমরা `github.com/gofiber/contrib/v3/casbin` ব্যবহার করব।

## ১. সেটআপ

```bash
go get -u github.com/gofiber/contrib/v3/casbin
```

## ২. Casbin মডেল ও পলিসি

দুটি ফাইল তৈরি করুন:
1.  `authz_model.conf` (Rule definition)
2.  `authz_policy.csv` (Who can do What)

**authz_model.conf:**
```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

**authz_policy.csv:**
```csv
p, admin, /users, POST
p, user, /users, GET
g, alice, admin
g, bob, user
```
(alice হলো admin, bob হলো user)

## ৩. মিডলওয়্যার ইন্টিগ্রেশন

```go
package main

import (
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/contrib/v3/casbin"
)

func main() {
    app := fiber.New()

    // Casbin Middleware লোড করা
    authz := casbin.New(casbin.Config{
        ModelFilePath: "authz_model.conf",
        PolicyFilePath: "authz_policy.csv", // অথবা ডেটাবেস অ্যাডাপ্টার
        
        // রিকোয়েস্ট থেকে সাবজেক্ট (User) বের করার ফাংশন
        Lookup: func(c fiber.Ctx) string {
            // উদাহরণস্বরূপ: JWT বা সেশন থেকে ইউজারনেম নেওয়া
            // বাস্তবে: return GetUserFromJWT(c)
            return c.Get("X-User") 
        },
    })

    // এই রাউটগুলো এখন Casbin দ্বারা সুরক্ষিত
    // Alice (admin) POST করতে পারবে, কিন্তু Bob (user) 403 পাবে
    app.Post("/users", authz, func(c fiber.Ctx) error {
        return c.SendString("User created")
    })

    // Bob শুধু এইটা এক্সেস করতে পারবে
    app.Get("/users", authz, func(c fiber.Ctx) error {
        return c.SendString("List of users")
    })

    app.Listen(":3000")
}
```

## প্রোডাকশন ব্যবহারের জন্য

CSV ফাইলের বদলে ডেটাবেস অ্যাডাপ্টার ব্যবহার করা উচিত (Gorm Adapter), যাতে রানটাইমে রোল এবং পারমিশন পরিবর্তন করা যায়।

```go
import gormadapter "github.com/casbin/gorm-adapter/v3"

a, _ := gormadapter.NewAdapterByDB(db) // GORM DB instance
authz := casbin.New(casbin.Config{
    ModelFilePath: "authz_model.conf",
    PolicyAdapter: a, // ফাইল এর বদলে DB অ্যাডাপ্টার
})
```

---
[← আগের: টাস্ক কিউ](07-task-queues-asynq.md) | [সূচি](../README.md) | [পরবর্তী: Swagger API ডক →](09-api-docs-swagger.md)
