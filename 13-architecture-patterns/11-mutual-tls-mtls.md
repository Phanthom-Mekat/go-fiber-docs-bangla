# মিউচুয়াল TLS (mTLS)

সাধারণ TLS/SSL এ সার্ভার তার আইডেন্টিটি প্রমাণ করে (সার্টিফিকেট দিয়ে)। mTLS এ **সার্ভার এবং ক্লায়েন্ট উভয়ই** একে অপরকে সার্টিফিকেট দিয়ে ভেরিফাই করে। এটি "Zero Trust" আর্কিটেকচার এবং মাইক্রোসারভিস সিকিউরিটির জন্য গোল্ড স্ট্যান্ডার্ড।

## Fiber এ mTLS কনফিগারেশন

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "log"
    "github.com/gofiber/fiber/v3"
)

func main() {
    app := fiber.New()
    
    app.Get("/", func(c fiber.Ctx) error {
        return c.SendString("Hello Secure World!")
    })

    // ১. CA সার্টিফিকেট লোড করা (যা দিয়ে ক্লায়েন্ট সার্টিফিকেট সাইন করা হয়েছে)
    caCert, err := ioutil.ReadFile("ca-cert.pem")
    if err != nil {
        log.Fatal(err)
    }
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    // ২. TLS কনফিগারেশন
    tlsConfig := &tls.Config{
        // ক্লায়েন্ট সার্টিফিকেট চাইবে এবং ভেরিফাই করবে
        ClientAuth: tls.RequireAndVerifyClientCert,
        ClientCAs:  caCertPool,
    }

    // ৩. লিসেনার তৈরি
    ln, err := tls.Listen("tcp", ":443", tlsConfig)
    if err != nil {
        log.Fatal(err)
    }

    // ৪. কাস্টম লিসেনার দিয়ে অ্যাপ রান করা
    // সার্ভারের নিজের সার্টিফিকেট ও কি (Key) লোড করতে হবে
    log.Fatal(app.Listener(ln, fiber.ListenConfig{
        CertFile: "server-cert.pem",
        KeyFile:  "server-key.pem",
    }))
}
```

## ক্লায়েন্ট (Go)

ক্লায়েন্টকেও তার নিজের সার্টিফিকেট পাঠাতে হবে।

```go
cert, _ := tls.LoadX509KeyPair("client-cert.pem", "client-key.pem")

client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{
            Certificates: []tls.Certificate{cert},
            RootCAs:      caCertPool, // সার্ভার ভেরিফাই করার জন্য
        },
    },
}
```

mTLS নিশ্চিত করে যে শুধুমাত্র ভ্যালিড সার্টিফিকেটধারী সার্ভিসগুলোই আপনার API কল করতে পারবে। পাসওয়ার্ড বা টোকেন চুরি হলেও হ্যাকার অ্যাক্সেস পাবে না।

---
[← আগের: SSE](10-server-sent-events.md) | [সূচি](../README.md) | [পরবর্তী: সার্ভারলেস Fiber →](12-serverless-fiber-lambda.md)
