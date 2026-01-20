# স্কেলিং (Scaling)

আপনার অ্যাপ যখন জনপ্রিয় হবে, তখন একটি সার্ভার আর ট্রাফিক সামাল দিতে পারবে না। তখন আমাদের স্কেলিং নিয়ে ভাবতে হবে।

## ভার্টিক্যাল বনাম হরাইজন্টাল

1.  **Vertical Scaling (Scale Up):** সার্ভারের RAM/CPU বাড়ানো। এটি সহজ কিন্তু এর একটা লিমিট আছে এবং ব্যয়বহুল।
2.  **Horizontal Scaling (Scale Out):** সার্ভারের সংখ্যা বাড়ানো (Load Balancing)। এটি অসীম পর্যন্ত স্কেল করা সম্ভব।

Fiber যেহেতু স্টেলেস (Stateless) হতে পারে, তাই এটি হরাইজন্টাল স্কেলিংয়ের জন্য আদর্শ।

## লোড ব্যালেন্সিং (Load Balancing)

Nginx, HAProxy বা ক্লাউড লোড ব্যালেন্সার (AWS ALB) ট্রাফিককে বিভিন্ন অ্যাপ সার্ভারে ডিস্ট্রিবিউট করে দেয়।

**Nginx কনফিগারেশন উদাহরণ:**

```nginx
upstream fiber_app {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://fiber_app;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## স্টেট ম্যানেজমেন্ট (Redis)

হরাইজন্টাল স্কেলিংয়ের সবচেয়ে বড় চ্যালেঞ্জ হলো "স্টেট" (Session, Cache)। অ্যাপ সার্ভার ১-এ ইউজার লগইন করলে, সেশন যদি লোকাল মেমরিতে থাকে, তাহলে পরবর্তী রিকোয়েস্ট অ্যাপ সার্ভার ২-এ গেলে ইউজার লগআউট হয়ে যাবে।

সমাধান: **Shared Storage** (Redis)।

### Fiber Session with Redis

```go
import (
    "github.com/gofiber/fiber/v3/middleware/session"
    "github.com/gofiber/storage/redis/v3"
)

func main() {
    // ১. Redis স্টোরেজ ইনিশিয়ালাইজ
    store := redis.New(redis.Config{
        Host: "localhost",
        Port: 6379,
    })

    // ২. সেশন মিডলওয়্যার কনফিগার
    sess := session.New(session.Config{
        Storage: store, // লোকাল মেমরি বা ফাইলের বদলে Redis ব্যবহার
    })

    app := fiber.New()

    app.Get("/", func(c fiber.Ctx) error {
        // সেশন পান বা তৈরি করুন
        s, _ := sess.Get(c)
        defer s.Save()

        s.Set("name", "john")
        return c.SendString("Session saved in Redis")
    })
}
```

এভাবে কনফিগার করলে আপনি হাজার হাজার অ্যাপ ইনস্ট্যান্স চালালেও সেশন হারাবে না, কারণ সব ইনস্ট্যান্স একই Redis থেকে ডেটা পড়বে।

## ক্লাস্টারিং (Clustering)

যদি আপনি একই মেশিনে একাধিক প্রসেস চালাতে চান, তাহলে Fiber-এর `Prefork: true` ব্যবহার করুন। এটি OS-এর `SO_REUSEPORT` ব্যবহার করে অটোমেটিক লোড ব্যালেন্সিং করে। তবে ভিন্ন ভিন্ন মেশিনে স্কেল করার জন্য উপরের Nginx + Redis মেথডটিই স্ট্যান্ডার্ড।

---
[← আগের: CI/CD](09-ci-cd-basics.md) | [সূচি](../README.md)
