# ইমেইল পাঠানো (Sending Emails)

অ্যাপ্লিকেশনে ইমেইল (ভেরিফিকেশন, পাসওয়ার্ড রিসেট) পাঠানো একটি সাধারণ রিকোয়ারমেন্ট। আমরা স্ট্যান্ডার্ড SMTP এবং SendGrid API ব্যবহার করে এটি দেখব।

## ১. SMTP ব্যবহার করে (gomail)

ছোট প্রোজেক্টের জন্য জিমেইল বা নিজস্ব SMTP সার্ভার ব্যবহার করতে পারেন। লাইব্রেরি: `gopkg.in/gomail.v2`

```go
package email

import (
    "gopkg.in/gomail.v2"
)

func SendWelcomeEmail(to string, name string) error {
    m := gomail.NewMessage()
    m.SetHeader("From", "noreply@myapp.com")
    m.SetHeader("To", to)
    m.SetHeader("Subject", "Welcome to Fiber App!")
    m.SetBody("text/html", "<h1>Hello " + name + "</h1><p>Thanks for joining.</p>")

    // SMTP কনফিগারেশন (Mailtrap বা Gmail)
    d := gomail.NewDialer("smtp.mailtrap.io", 2525, "user", "pass")

    return d.DialAndSend(m)
}
```

## ২. SendGrid API ব্যবহার করে

প্রোডাকশনে হাই ভলিউম এবং ডেলিভারিবিলিটির জন্য SendGrid বা AWS SES রিকমেন্ডেড।

```go
package email

import (
    "github.com/sendgrid/sendgrid-go"
    "github.com/sendgrid/sendgrid-go/helpers/mail"
    "os"
)

func SendWithSendGrid(toEmail string) error {
    from := mail.NewEmail("My App", "noreply@myapp.com")
    subject := "Sending with SendGrid is Fun"
    to := mail.NewEmail("User", toEmail)
    plainTextContent := "and easy to do anywhere, even with Go"
    htmlContent := "<strong>and easy to do anywhere, even with Go</strong>"
    
    message := mail.NewSingleEmail(from, subject, to, plainTextContent, htmlContent)
    
    client := sendgrid.NewSendClient(os.Getenv("SENDGRID_API_KEY"))
    _, err := client.Send(message)
    return err
}
```

## অ্যাসিনক্রোনাস ইমেইল (Best Practice)

হ্যান্ডলারের ভেতর সরাসরি ইমেইল পাঠাবেন না। এটি রিকোয়েস্ট স্লো করে দেয়। ইমেইল পাঠানোর কাজটি **Task Queue** (পরের অধ্যায়ে আলোচনা করা হয়েছে) বা অন্তত একটি `goroutine` এ করা উচিত।

```go
// সাধারণ গোরুটিন প্যাটার্ন (সিম্পল ইউসকেস)
app.Post("/register", func(c fiber.Ctx) error {
    // ... ইউজার সেভ ...

    // ব্যাকগ্রাউন্ডে ইমেইল পাঠানো
    go func() {
         email.SendWelcomeEmail(user.Email, user.Name)
    }()

    return c.Status(201).SendString("User created")
})
```

---
[← আগের: S3 স্টোরেজ](05-s3-minio-storage.md) | [সূচি](../README.md) | [পরবর্তী: টাস্ক কিউ (Asynq) →](07-task-queues-asynq.md)
