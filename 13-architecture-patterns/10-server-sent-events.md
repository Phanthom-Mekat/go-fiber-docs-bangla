# সার্ভার-সেন্ট ইভেন্টস (Server-Sent Events - SSE)

রিয়েল-টাইম আপডেটের জন্য (যেমন: নোটিফিকেশন, লাইভ স্কোর) ওয়েবসকেট (WebSockets) এর চেয়ে SSE সহজ এবং হালকা, কারণ এটি সাধারণ HTTP প্রোটোকল ব্যবহার করে এবং ফায়ারওয়ল ফ্রেন্ডলি। তবে এটি শুধুমাত্র **Server -> Client** (একমুখী) যোগাযোগ করতে পারে।

## Fiber এ SSE ইমপ্লিমেন্টেশন

Fiber এ SSE হ্যান্ডেল করার জন্য স্ট্রিমিং রেসপন্স ব্যবহার করা হয়।

```go
package main

import (
    "bufio"
    "fmt"
    "time"
    "github.com/gofiber/fiber/v3"
    "github.com/valyala/fasthttp"
)

func main() {
    app := fiber.New()

    app.Get("/sse", func(c fiber.Ctx) error {
        // ১. হেডার সেট করা যা ব্রাউজারকে বলে এটি ইভেন্ট স্ট্রিম
        c.Set("Content-Type", "text/event-stream")
        c.Set("Cache-Control", "no-cache")
        c.Set("Connection", "keep-alive")
        c.Set("Transfer-Encoding", "chunked")

        // ২. স্ট্রিম রাইটার শুরু
        c.Context().SetBodyStreamWriter(fasthttp.StreamWriter(func(w *bufio.Writer) {
            ticker := time.NewTicker(1 * time.Second)
            defer ticker.Stop()

            for i := 0; i < 10; i++ {
                // ইভেন্ট ফরম্যাট: "data: <message>\n\n"
                fmt.Fprintf(w, "data: Message %d at %v\n\n", i, time.Now())
                
                // ফ্লাশ করা জরুরি যাতে ক্লায়েন্ট সাথে সাথে পায়
                if err := w.Flush(); err != nil {
                    return // ক্লায়েন্ট সংযোগ বিচ্ছিন্ন করেছে
                }
                
                <-ticker.C
            }
        }))

        return nil
    })

    app.Listen(":3000")
}
```

## ক্লায়েন্ট সাইড (JavaScript)

```javascript
const eventSource = new EventSource('/sse');

eventSource.onmessage = function(event) {
    console.log("New message:", event.data);
};

eventSource.onerror = function() {
    console.log("Connection lost");
    eventSource.close();
};
```

SSE স্বয়ংক্রিয়ভাবে সংযোগ বিচ্ছিন্ন হলে রিকানেক্ট করে, যা ওয়েবসকেটের ক্ষেত্রে ম্যানুয়ালি করতে হয়।

---
[← আগের: ফিচার ফ্ল্যাগ](09-feature-flags.md) | [সূচি](../README.md) | [পরবর্তী: mTLS →](11-mutual-tls-mtls.md)
