# ফাইল আপলোড এবং S3/MinIO স্টোরেজ

লোকাল ডিস্কে ফাইল রাখা স্কেলেবল নয় (সার্ভার নষ্ট হলে ফাইল হারাবে)। প্রোডাকশনে অ্যামাজন S3 বা সেলফ-হোস্টেড MinIO (S3 compatible) ব্যবহার করা উচিত।

আমরা অফিশিয়াল `github.com/aws/aws-sdk-go-v2` ব্যবহার করব।

## ১. AWS SDK কনফিগারেশন

```go
package storage

import (
    "context"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

var S3Client *s3.Client

func Init() {
    cfg, _ := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1"),
    )
    S3Client = s3.NewFromConfig(cfg)
}
```

## ২. ফাইল আপলোড হ্যান্ডলার

Fiber দিয়ে ফাইল রিসিভ করে সরাসরি S3 তে স্ট্রিম করা মেমরি এফিশিয়েন্ট।

```go
func UploadAvatar(c fiber.Ctx) error {
    // ১. ফর্ম ফাইল পড়া
    file, err := c.FormFile("avatar")
    if err != nil {
        return c.Status(400).SendString("File required")
    }

    // ২. ফাইল খোলা (Multipart stream)
    src, err := file.Open()
    if err != nil {
        return err
    }
    defer src.Close()

    // ৩. ফাইলের নাম জেনারেট (ইউনিক হওয়া চাই)
    filename := "avatars/" + uuid.New().String() + ".jpg"

    // ৪. S3 তে আপলোড
    _, err = storage.S3Client.PutObject(c.UserContext(), &s3.PutObjectInput{
        Bucket: aws.String("my-bucket-name"),
        Key:    aws.String(filename),
        Body:   src, // স্ট্রিম হিসেবে যাচ্ছে
        ContentType: aws.String(file.Header.Get("Content-Type")),
    })

    if err != nil {
        return c.Status(500).SendString("Upload failed: " + err.Error())
    }

    // ৫. পাবলিক URL রিটার্ন
    fileURL := "https://my-bucket-name.s3.amazonaws.com/" + filename
    return c.JSON(fiber.Map{"url": fileURL})
}
```

## ৩. MinIO সেটআপ (লোকাল S3)

MinIO ব্যবহার করলে কনফিগারেশনে শুধু `EndpointResolver` পরিবর্তন করতে হয়।

```go
cfg, _ := config.LoadDefaultConfig(context.TODO(),
    config.WithEndpointResolverWithOptions(aws.EndpointResolverWithOptionsFunc(
        func(service, region string, options ...interface{}) (aws.Endpoint, error) {
            return aws.Endpoint{
                URL: "http://localhost:9000", // MinIO URL
            }, nil
        },
    )),
)
```

## সিকিউরিটি টিপস

*   **Presigned URLs:** ক্লায়েন্টকে সরাসরি আপলোড করার অনুমতি দিতে Presigned URL ব্যবহার করুন। এতে আপনার সার্ভারের ব্যান্ডউইথ বাঁচে।
*   **Validation:** ফাইল সাইজ এবং `Content-Type` অবশ্যই ভ্যালিডেট করবেন। `c.FormFile` কল করার আগে ম্যাজিক বাইট চেক করা ভালো প্র্যাকটিস।

---
[← আগের: Redis](04-redis-caching-advanced.md) | [সূচি](../README.md) | [পরবর্তী: ইমেইল সার্ভিস →](06-email-smtp-sendgrid.md)
