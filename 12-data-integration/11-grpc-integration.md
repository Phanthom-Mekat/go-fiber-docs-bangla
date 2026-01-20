# gRPC ইন্টিগ্রেশন (Microservices)

মাইক্রোসারভিস আর্কিটেকচারে Fiber সাধারণত একটি **Gateway** বা **BFF (Backend For Frontend)** হিসেবে কাজ করে, যা ক্লায়েন্ট থেকে HTTP/JSON রিকোয়েস্ট নেয় এবং ইন্টার্নাল gRPC সার্ভারের সাথে কথা বলে।

## ১. প্রোটোবাফ (Protobuf) ডেফিনিশন

ধরা যাক আমাদের একটি `UserService` আছে। `user.proto`:

```protobuf
syntax = "proto3";
package user;
option go_package = "./proto";

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}

message UserRequest {
  string id = 1;
}

message UserResponse {
  string name = 1;
  string email = 2;
}
```

`protoc` দিয়ে Go কোড জেনারেট করুন।

## ২. gRPC ক্লায়েন্ট তৈরি

Fiber অ্যাপ gRPC ক্লায়েন্ট হিসেবে কাজ করবে।

```go
package rpc

import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pb "my-app/proto"
)

var UserClient pb.UserServiceClient

func Init() {
    // ইন্টার্নাল সার্ভিস (যেমন user-service:50051) এ কানেক্ট করা
    conn, err := grpc.NewClient("localhost:50051", 
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        panic(err)
    }
    
    UserClient = pb.NewUserServiceClient(conn)
}
```

## ৩. Fiber হ্যান্ডলার

HTTP রিকোয়েস্ট -> gRPC কল -> HTTP রেসপন্স

```go
func GetUserProfile(c fiber.Ctx) error {
    id := c.Params("id")
    
    // gRPC কল (Fiber এর কনটেক্সট পাস করা ভালো প্র্যাকটিস)
    // যাতে ক্লায়েন্ট রিকোয়েস্ট ক্যান্সেল করলে gRPC কলও ক্যান্সেল হয়
    req := &pb.UserRequest{Id: id}
    res, err := rpc.UserClient.GetUser(c.UserContext(), req)
    
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    
    // gRPC রেসপন্সকে JSON এ কনভার্ট করে ক্লায়েন্টকে পাঠানো
    return c.JSON(res)
}
```

এটি একটি খুব শক্তিশালী প্যাটার্ন যা আপনাকে Go-এর হাই-পারফরম্যান্স HTTP (Fiber) এবং ইন্টার-সার্ভিস কমিউনিকেশন (gRPC) এর সুবিধা একসাথে দেয়।

---
[← আগের: GraphQL](10-graphql-gqlgen.md) | [সূচি](../README.md) | [পরবর্তী: ওয়েবহুকস →](12-webhooks-hmac.md)
