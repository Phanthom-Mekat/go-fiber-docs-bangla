# কনফিগারেশন ম্যানেজমেন্ট (Configuration Management)

আধুনিক ক্লাউড নেটিভ অ্যাপ্লিকেশনের জন্য কনফিগারেশন ম্যানেজমেন্ট অত্যন্ত গুরুত্বপূর্ণ। আমরা "12-Factor App" মেথডোলজি অনুসরণ করব, যেখানে কনফিগারেশন এনভায়রনমেন্ট (Environment) ভেরিয়েবলে রাখা হয়।

Go-তে কনফিগারেশন লোড করার জন্য `Viper` বা `Knadh/koanf` জনপ্রিয়।

## Viper দিয়ে কনফিগারেশন লোড করা

Viper এনভায়রনমেন্ট ভেরিয়েবল, কনফিগ ফাইল (.env, .yaml, .json) এবং ফ্ল্যাগ থেকে কনফিগ রিড করতে পারে।

### সেটআপ

```go
package config

import (
    "log"
    "github.com/spf13/viper"
)

type Config struct {
    AppPort     string `mapstructure:"APP_PORT"`
    DatabaseURL string `mapstructure:"DATABASE_URL"`
    JWTSecret   string `mapstructure:"JWT_SECRET"`
    Environment string `mapstructure:"ENVIRONMENT"`
}

func LoadConfig() *Config {
    viper.SetConfigFile(".env") // .env ফাইল খুঁজবে
    viper.AutomaticEnv()        // এনভায়রনমেন্ট ভেরিয়েবল অটোমেটিক রিড করবে

    if err := viper.ReadInConfig(); err != nil {
        log.Println("No .env file found, using system env variables")
    }

    config := &Config{}
    if err := viper.Unmarshal(config); err != nil {
        log.Fatal("Error loading config: ", err)
    }

    return config
}
```

## ব্যবহার (Usage)

```go
package main

import (
    "log"
    "github.com/gofiber/fiber/v3"
    "my-app/config"
)

func main() {
    // ১. কনফিগ লোড
    cfg := config.LoadConfig()

    app := fiber.New(fiber.Config{
        AppName: "My Prod App",
    })

    // ২. কনফিগ ব্যবহার
    app.Get("/", func(c fiber.Ctx) error {
        return c.SendString("Running in " + cfg.Environment)
    })

    log.Fatal(app.Listen(":" + cfg.AppPort))
}
```

## Zero Allocation Configuration

হাই-পারফরম্যান্স অ্যাপের জন্য, প্রতিবার `viper.Get()` কল না করে কনফিগ স্ট্রাকট (Struct) তৈরি করে সেটা গ্লোবালি বা ডিপেন্ডেন্সি ইনজেকশনের মাধ্যমে পাস করা উচিত।

v3 তে `app.SetState` ব্যবহার করা যেতে পারে:

```go
app.SetState("config", cfg)

// হ্যান্ডলারে
cfg := c.App().GetState("config").(*Config)
```

## সিকিউরিটি চেকলিস্ট

1.  **.gitignore:** `.env` ফাইল কখনই গিট রিপোজিটরিতে আপলোড করবেন না। `.env.example` আপলোড করুন।
2.  **Secrets:** প্রোডাকশনে Docker Secret, AWS Parameter Store বা HashiCorp Vault ব্যবহার করুন।
3.  **Validation:** কনফিগারেশন লোড করার সময় ভ্যালিডেশন করুন (যেমন পোর্ট নম্বর সঠিক কিনা, ডাটাবেস ইউআরএল আছে কিনা)।

---
[← আগের: লগিং](02-logging-observability.md) | [সূচি](../README.md) | [পরবর্তী: গ্রেসফুল শাটডাউন →](04-graceful-shutdown.md)
