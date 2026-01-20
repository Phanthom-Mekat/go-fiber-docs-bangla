# ইভেন্ট সোর্সিং (Event Sourcing)

ইভেন্ট সোর্সিং প্যাটার্নে আমরা অ্যাপ্লিকেশনের "স্টেইট" (State) সরাসরি সংরক্ষণ না করে, "ঘটনাগুলো" (Events) সংরক্ষণ করি যা ওই স্টেইট তৈরি করেছে।

উদাহরণস্বরূপ, ব্যাংক অ্যাকাউন্টের ব্যালেন্স সংরক্ষণ না করে আমরা প্রতিটি `Deposit` এবং `Withdrawal` ট্রানজেকশন সংরক্ষণ করি। বর্তমান ব্যালেন্স জানতে হলে শুরু থেকে সব ইভেন্ট রিপ্লে (Replay) করতে হয়।

## ইভেন্ট স্টোর (Event Store)

ইভেন্ট স্টোর হলো একটি ডাটাবেস যা শুধুমাত্র অ্যাপেন্ড-অনলি (Append-only) ইভেন্ট রাখে।

```go
type Event struct {
    ID        string
    Type      string    // "MoneyDeposited"
    Payload   []byte    // JSON data
    Timestamp time.Time
    Version   int
}
```

## ইমপ্লিমেন্টেশন কনসেপ্ট

### ১. এগ্রিগেট (Aggregate)

```go
type BankAccount struct {
    ID      string
    Balance float64
}

// ইভেন্ট অ্যাপ্লাই করে স্টেট আপডেট করা
func (a *BankAccount) Apply(event Event) {
    switch event.Type {
    case "MoneyDeposited":
        var p DepositPayload
        json.Unmarshal(event.Payload, &p)
        a.Balance += p.Amount
    case "MoneyWithdrawn":
        var p WithdrawPayload
        json.Unmarshal(event.Payload, &p)
        a.Balance -= p.Amount
    }
}
```

### ২. কমান্ড হ্যান্ডলিং

```go
func HandleDeposit(cmd DepositCmd) error {
    // ১. ইভেন্ট স্টোর থেকে সব পুরনো ইভেন্ট লোড করা
    events := eventStore.GetEvents(cmd.AccountID)
    
    // ২. বর্তমান স্টেট রিবিল্ড করা
    account := &BankAccount{}
    for _, e := range events {
        account.Apply(e)
    }
    
    // ৩. ভ্যালিডেশন
    if account.Balance < 0 {
        return errors.New("invalid state")
    }
    
    // ৪. নতুন ইভেন্ট তৈরি ও সেভ করা
    newEvent := Event{
        Type: "MoneyDeposited",
        Payload: json.Marshal(DepositPayload{Amount: cmd.Amount}),
    }
    eventStore.Save(newEvent)
}
```

## Fiber এর কাজ

Fiber এখানে শুধু কমান্ড রিসিভ করে।

```go
app.Post("/account/:id/deposit", func(c fiber.Ctx) error {
    // কমান্ড তৈরি এবং হ্যান্ডলারে পাঠানো
    // ...
    return c.SendString("Deposit initiated")
})
```

## চ্যালেঞ্জ

*   **Complexity:** এটি ইমপ্লিমেন্ট করা এবং মেইনটেইন করা সাধারণ CRUD এর চেয়ে অনেক কঠিন।
*   **Snapshots:** হাজার হাজার ইভেন্ট থাকলে রিপ্লে করা স্লো হতে পারে। তখন প্রতি ১০০ ইভেন্ট পর পর একটি "Snapshot" বা বর্তমান অবস্থা সেভ করে রাখা হয়।

---
[← আগের: CQRS](05-cqrs-pattern.md) | [সূচি](../README.md) | [পরবর্তী: সার্কিট ব্রেকার →](07-resilience-circuit-breaker.md)
