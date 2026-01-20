# অ্যাডভান্সড Redis ক্যাশিং (Advanced Redis Caching)

`middleware/cache` বেসিক HTTP ক্যাশিংয়ের জন্য ভালো, কিন্তু কাস্টম ডেটা ক্যাশ করা বা "Cache-Aside" প্যাটার্ন ইমপ্লিমেন্ট করার জন্য আমাদের সরাসরি Redis ক্লায়েন্ট ব্যবহার করা উচিত।

আমরা `github.com/redis/go-redis/v9` প্যাকেজটি ব্যবহার করব (এটি ইন্ডাস্ট্রি স্ট্যান্ডার্ড)।

## ১. Redis Client সেটআপ

```go
package cache

import (
    "context"
    "github.com/redis/go-redis/v9"
)

var Rdb *redis.Client

func Init() {
    Rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })
}
```

## ২. Cache-Aside প্যাটার্ন

এই প্যাটার্নে আমরা:
1.  প্রথমে ক্যাশে (Redis) ডেটা খুঁজি।
2.  পেলে সাথে সাথে রিটার্ন করি (Cache Hit)।
3.  না পেলে (Cache Miss), ডেটাবেস থেকে ডেটা আনি।
4.  ডেটাবেসের ডেটা ক্যাশে সেভ করি এবং রেসপন্স দেই।

```go
func GetUserProfile(c fiber.Ctx) error {
    userID := c.Params("id")
    cacheKey := "user:" + userID
    ctx := c.UserContext()

    // ১. ক্যাশ চেক
    val, err := cache.Rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        // Cache Hit! Redis থেকে স্ট্রিং পেয়েছি, এখন JSON আনmarshal করতে পারি
        return c.SendString(val) // অথবা JSON হিসেবে পার্স করে পাঠানো
    }

    // ২. Cache Miss -> ডেটাবেস কল (সিমুলেটেড)
    user, err := db.FindUser(userID)
    if err != nil {
        return c.Status(404).SendString("User not found")
    }

    // ৩. ক্যাশে সেভ (JSON হিসেবে)
    jsonBytes, _ := json.Marshal(user)
    
    // TTL: 10 মিনিট পর এক্সপায়ার হবে
    err = cache.Rdb.Set(ctx, cacheKey, jsonBytes, 10*time.Minute).Err()
    if err != nil {
        // লগ করুন, কিন্তু রিকোয়েস্ট ফেইল করবেন না
        log.Println("Redis set error:", err)
    }

    return c.JSON(user)
}
```

## ৩. Pub/Sub (রিয়েল-টাইম ইভেন্ট)

Redis Pub/Sub ব্যবহার করে আমরা বিভিন্ন সার্ভার ইনস্ট্যান্সের মধ্যে মেসেজ পাস করতে পারি (যেমন চ্যাট অ্যাপে)।

```go
// সাবস্ক্রাইবার (আলাদা গোরুটিনে রান হবে)
func SubscribeMessages() {
    pubsub := cache.Rdb.Subscribe(context.Background(), "chat_channel")
    ch := pubsub.Channel()

    for msg := range ch {
        fmt.Println("Received message:", msg.Payload)
        // এখানে WebSocket দিয়ে ক্লায়েন্টকে মেসেজ পাঠানো যেতে পারে
    }
}

// পাবলিশার (Fiber হ্যান্ডলার)
func SendMessage(c fiber.Ctx) error {
    msg := c.FormValue("msg")
    err := cache.Rdb.Publish(c.UserContext(), "chat_channel", msg).Err()
    return err
}
```

---
[← আগের: MongoDB](03-mongodb-official.md) | [সূচি](../README.md) | [পরবর্তী: S3 স্টোরেজ →](05-s3-minio-storage.md)
