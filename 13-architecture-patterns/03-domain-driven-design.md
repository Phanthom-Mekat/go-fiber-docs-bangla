# ডোমেইন-ড্রিভেন ডিজাইন (Domain-Driven Design - DDD)

DDD কোনো নির্দিষ্ট আর্কিটেকচার নয়, এটি সফটওয়্যার ডিজাইনের একটি ফিলোসফি যা জটিল বিজনেস ডোমেইনকে কোডে ম্যাপ করতে সাহায্য করে। Go-তে DDD ইমপ্লিমেন্ট করা কিছুটা চ্যালেঞ্জিং কিন্তু খুবই ফলপ্রসূ।

## DDD এর মূল উপাদানসমূহ

### ১. Entity & Value Object

*   **Entity:** যার নিজস্ব আইডেন্টিটি (ID) আছে এবং সময়ের সাথে পরিবর্তন হয় (যেমন: User, Order)।
*   **Value Object:** যার আইডেন্টিটি নেই, ভ্যালুই তার পরিচয় এবং এটি ইমিউটেবল (যেমন: Address, Money)।

```go
// Entity
type Order struct {
    ID          string
    Items       []OrderItem
    ShippingAdd Address // Value Object
    Status      string
}

// Value Object
type Address struct {
    Street string
    City   string
    Zip    string
}
```

### ২. Aggregate & Aggregate Root

**Aggregate** হলো রিলেটেড অবজেক্টের একটি ক্লাস্টার যা একটি ইউনিট হিসেবে কাজ করে। **Aggregate Root** হলো সেই এনটিটি যার মাধ্যমে বাইরের জগত এগ্রিগেটের সাথে কথা বলে।

উদাহরণ: `Order` হলো Aggregate Root। আপনি `OrderItem` কে সরাসরি অ্যাক্সেস বা মডিফাই করবেন না, `Order` এর মেথড ব্যবহার করে করবেন।

```go
func (o *Order) AddItem(item OrderItem) error {
    if o.Status != "DRAFT" {
        return errors.New("cannot add item to confirmed order")
    }
    o.Items = append(o.Items, item)
    return nil
}
```

### ৩. Repository Interface

রিপজিটরি শুধু Aggregate Root এর জন্য তৈরি হবে।

```go
type OrderRepository interface {
    Save(order *Order) error
    FindByID(id string) (*Order, error)
}
```

### ৪. Domain Events

যখন ডোমেইনে গুরুত্বপূর্ণ কিছু ঘটে, তখন ইভেন্ট ফায়ার করা হয়। এটি সাইড-এফেক্ট (যেমন ইমেইল পাঠানো) হ্যান্ডেল করার সেরা উপায়।

```go
type OrderConfirmedEvent struct {
    OrderID   string
    Timestamp time.Time
}

func (o *Order) Confirm() {
    o.Status = "CONFIRMED"
    // ইভেন্ট অ্যাড করা
    o.events = append(o.events, OrderConfirmedEvent{OrderID: o.ID})
}
```

## Fiber এর সাথে DDD

Fiber হ্যান্ডলার হবে খুবই চিকন (Thin)। এটি শুধু কমান্ডগুলো অ্যাপ্লিকেশনে সার্ভিস লেয়ারে পাঠাবে।

```go
func ConfirmOrder(c fiber.Ctx) error {
    orderID := c.Params("id")
    
    // Application Service কল
    if err := orderService.ConfirmOrder(orderID); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    
    return c.SendStatus(200)
}
```

DDD ব্যবহার করলে আপনার বিজনেস লজিক টেকনিক্যাল ডিটেইলস (DB, HTTP) থেকে সম্পূর্ণ মুক্ত থাকে।

---
[← আগের: হেক্সাগোনাল](02-hexagonal-architecture.md) | [সূচি](../README.md) | [পরবর্তী: ডিপেন্ডেন্সি ইনজেকশন →](04-dependency-injection.md)
