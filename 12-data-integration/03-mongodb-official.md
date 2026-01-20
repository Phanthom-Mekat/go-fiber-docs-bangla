# MongoDB ইন্টিগ্রেশন (MongoDB Integration)

Go-তে MongoDB ব্যবহার করার জন্য অফিশিয়াল ড্রাইভার `go.mongodb.org/mongo-driver` ব্যবহার করা স্ট্যান্ডার্ড।

## ১. সেটআপ

```go
package database

import (
    "context"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

var Collection *mongo.Collection

func Connect() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    client, _ := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))

    db := client.Database("fiber_app")
    Collection = db.Collection("users")
}
```

## ২. মডেল (BSON Tags)

MongoDB-তে JSON এর পরিবর্তে BSON (Binary JSON) ব্যবহৃত হয়। তাই আমাদের স্ট্রাক্টে `bson` ট্যাগ দিতে হবে।

```go
type User struct {
    ID    string `json:"id,omitempty" bson:"_id,omitempty"`
    Name  string `json:"name" bson:"name"`
    Email string `json:"email" bson:"email"`
}
```

## ৩. CRUD অপারেশন

```go
package main

import (
    "github.com/gofiber/fiber/v3"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "my-app/database"
)

func CreateUser(c fiber.Ctx) error {
    user := new(User)
    
    if err := c.Bind().Body(user); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    
    // ম্যানুয়ালি ID জেনারেট করা (অথবা MongoDB কে করতে দেওয়া)
    user.ID = primitive.NewObjectID().Hex()

    // কনটেক্সট পাস করা জরুরি
    _, err := database.Collection.InsertOne(c.UserContext(), user)
    if err != nil {
        return c.Status(500).SendString("Could not create user")
    }

    return c.Status(201).JSON(user)
}

func GetUsers(c fiber.Ctx) error {
    var users []User = make([]User, 0)
    
    cursor, err := database.Collection.Find(c.UserContext(), bson.M{})
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    
    // কার্সর ইটারেট করা
    if err := cursor.All(c.UserContext(), &users); err != nil {
        return c.Status(500).SendString(err.Error())
    }

    return c.JSON(users)
}
```

## পারফরমেন্স টিপস

*   **ইনডেক্সিং:** MongoDB তে কুয়েরি ফাস্ট করতে ফিল্ডগুলোতে ইনডেক্স তৈরি করা নিশ্চিত করুন।
*   **Projections:** পুরো ডকুমেন্ট ফেচ না করে শুধু প্রয়োজনীয় ফিল্ডগুলো আনুন (Projection ব্যবহার করে)।

---
[← আগের: SQLC](02-sql-builder-sqlc.md) | [সূচি](../README.md) | [পরবর্তী: Redis ক্যাশিং →](04-redis-caching-advanced.md)
