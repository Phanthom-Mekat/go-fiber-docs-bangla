# টাস্ক কিউ (Task Queues with Asynq)

ভারী কাজগুলো (ইমেইল পাঠানো, ইমেজ প্রসেসিং, রিপোর্ট জেনারেশন) HTTP রিকোয়েস্টের ভেতর না করে ব্যাকগ্রাউন্ডে করা উচিত। Go-তে Redis-based টাস্ক কিউ `hibiken/asynq` খুবই জনপ্রিয় (লারাভেল Horizon বা Celery এর মত)।

## ১. সেটআপ

```bash
go get -u github.com/hibiken/asynq
```

আমাদের দুটি অংশ দরকার:
1.  **Client:** টাস্ক তৈরি করে কিউতে জমা দেওয়ার জন্য (Fiber Handler ব্যবহার করবে)।
2.  **Server (Worker):** কিউ থেকে টাস্ক নিয়ে প্রসেস করার জন্য (আলাদা প্রসেস হিসেবে রান হবে)।

## ২. Asynq Client (Fiber এর ভেতর)

```go
package tasks

import (
    "github.com/hibiken/asynq"
)

var Client *asynq.Client

func Init() {
    Client = asynq.NewClient(asynq.RedisClientOpt{Addr: "localhost:6379"})
}

// টাস্ক পেলোড ডিফাইন করা
const TypeEmailDelivery = "email:deliver"

type EmailPayload struct {
    UserID int
}
```

**হ্যান্ডলার থেকে টাস্ক এনকিউ করা:**

```go
app.Post("/signup", func(c fiber.Ctx) error {
    // ... ইউজার তৈরি ...
    
    // ১. পেলোড তৈরি
    payload, _ := json.Marshal(tasks.EmailPayload{UserID: 123})
    
    // ২. টাস্ক এনকিউ
    task := asynq.NewTask(tasks.TypeEmailDelivery, payload)
    
    // অপশনসহ (যেমন: ১০ সেকেন্ড পর রান হবে)
    info, err := tasks.Client.Enqueue(task, asynq.ProcessIn(10*time.Second))
    
    if err != nil {
        return c.Status(500).SendString("Job failed")
    }
    
    return c.SendString("Job enqueued: " + info.ID)
})
```

## ৩. Asynq Worker (আলাদা `main.go` বা গোরুটিন)

```go
func RunWorker() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{Concurrency: 10},
    )

    mux := asynq.NewServeMux()
    
    // টাস্ক হ্যান্ডলার রেজিস্টার
    mux.HandleFunc(tasks.TypeEmailDelivery, HandleEmailDelivery)

    if err := srv.Run(mux); err != nil {
        log.Fatal(err)
    }
}

func HandleEmailDelivery(ctx context.Context, t *asynq.Task) error {
    var p tasks.EmailPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    
    log.Printf("Sending email to User ID: %d", p.UserID)
    // আসল ইমেইল পাঠানোর লজিক এখানে...
    return nil
}
```

## কেন Asynq?

*   **Reliability:** টাস্ক ফেইল করলে এটি অটোমেটিক রি-ট্রাই (Exponential backoff) করে।
*   **Priority:** ক্রিটিক্যাল টাস্কগুলোর জন্য প্রায়োরিটি কিউ সেট করা যায়।
*   **Scheduling:** ভবিষ্যতে রান করার জন্য শিডিউল করা যায় (Delay Queue)।

---
[← আগের: ইমেইল](06-email-smtp-sendgrid.md) | [সূচি](../README.md) | [পরবর্তী: অথোরাইজেশন (Casbin) →](08-rbac-casbin.md)
