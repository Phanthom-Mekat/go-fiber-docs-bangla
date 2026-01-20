# JWT Authentication

JWT (JSON Web Token) দিয়ে সম্পূর্ণ অথেনটিকেশন সিস্টেম তৈরি করুন।

## প্রজেক্ট স্ট্রাকচার

```
auth-api/
├── main.go
├── config/
│   ├── database.go
│   └── jwt.go
├── models/
│   └── user.go
├── handlers/
│   └── auth.go
├── middleware/
│   └── auth.go
├── utils/
│   └── token.go
└── .env
```

## ধাপ ১: ডিপেন্ডেন্সি ইনস্টল করুন

```bash
go get github.com/gofiber/fiber/v3
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto/bcrypt
go get gorm.io/gorm
go get gorm.io/driver/postgres
go get github.com/joho/godotenv
```

## ধাপ ২: JWT Configuration

**config/jwt.go**

```go
package config

import (
    "os"
    "time"
)

type JWTConfig struct {
    Secret          string
    ExpirationHours int
}

func GetJWTConfig() *JWTConfig {
    return &JWTConfig{
        Secret:          os.Getenv("JWT_SECRET"),
        ExpirationHours: 24 * 7, // 7 দিন
    }
}

func GetJWTExpiration() time.Duration {
    return time.Hour * time.Duration(GetJWTConfig().ExpirationHours)
}
```

## ধাপ ৩: User Model

**models/user.go**

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Name      string         `gorm:"size:100;not null" json:"name"`
    Email     string         `gorm:"size:100;uniqueIndex;not null" json:"email"`
    Password  string         `gorm:"not null" json:"-"`
    Role      string         `gorm:"size:20;default:'user'" json:"role"` // user, admin
    IsActive  bool           `gorm:"default:true" json:"is_active"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

type RegisterRequest struct {
    Name     string `json:"name" validate:"required,min=3"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=6"`
}

type LoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}

type LoginResponse struct {
    Token string `json:"token"`
    User  *User  `json:"user"`
}

type ChangePasswordRequest struct {
    OldPassword string `json:"old_password" validate:"required"`
    NewPassword string `json:"new_password" validate:"required,min=6"`
}
```

## ধাপ ৪: JWT Token Utility

**utils/token.go**

```go
package utils

import (
    "auth-api/config"
    "auth-api/models"
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type JWTClaims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// JWT টোকেন জেনারেট করুন
func GenerateToken(user *models.User) (string, error) {
    jwtConfig := config.GetJWTConfig()

    claims := JWTClaims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(config.GetJWTExpiration())),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "auth-api",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtConfig.Secret))
}

// JWT টোকেন ভেরিফাই করুন
func VerifyToken(tokenString string) (*JWTClaims, error) {
    jwtConfig := config.GetJWTConfig()

    token, err := jwt.ParseWithClaims(
        tokenString,
        &JWTClaims{},
        func(token *jwt.Token) (interface{}, error) {
            return []byte(jwtConfig.Secret), nil
        },
    )

    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*JWTClaims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }

    return claims, nil
}

// Refresh Token জেনারেট করুন (optional)
func GenerateRefreshToken(user *models.User) (string, error) {
    jwtConfig := config.GetJWTConfig()

    claims := JWTClaims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Hour * 24 * 30)), // 30 দিন
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "auth-api",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtConfig.Secret))
}
```

## ধাপ ৫: Auth Middleware

**middleware/auth.go**

```go
package middleware

import (
    "auth-api/utils"
    "strings"

    "github.com/gofiber/fiber/v3"
)

// JWT Authentication Middleware
func AuthRequired() fiber.Handler {
    return func(c fiber.Ctx) error {
        // Authorization হেডার পান
        authHeader := c.Get("Authorization")
        if authHeader == "" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Authorization header required",
            })
        }

        // Bearer টোকেন এক্সট্র্যাক্ট করুন
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Invalid authorization format. Use: Bearer <token>",
            })
        }

        tokenString := parts[1]

        // টোকেন ভেরিফাই করুন
        claims, err := utils.VerifyToken(tokenString)
        if err != nil {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Invalid or expired token",
            })
        }

        // Claims কনটেক্সটে সেভ করুন
        c.Locals("userID", claims.UserID)
        c.Locals("email", claims.Email)
        c.Locals("role", claims.Role)

        return c.Next()
    }
}

// Role-based Authorization Middleware
func RequireRole(roles ...string) fiber.Handler {
    return func(c fiber.Ctx) error {
        userRole := c.Locals("role").(string)

        for _, role := range roles {
            if userRole == role {
                return c.Next()
            }
        }

        return c.Status(fiber.StatusForbidden).JSON(fiber.Map{
            "error": "Insufficient permissions",
        })
    }
}

// Optional Auth (টোকেন থাকলে ভেরিফাই করবে, না থাকলে এগিয়ে যাবে)
func OptionalAuth() fiber.Handler {
    return func(c fiber.Ctx) error {
        authHeader := c.Get("Authorization")
        if authHeader == "" {
            return c.Next()
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) == 2 && parts[0] == "Bearer" {
            claims, err := utils.VerifyToken(parts[1])
            if err == nil {
                c.Locals("userID", claims.UserID)
                c.Locals("email", claims.Email)
                c.Locals("role", claims.Role)
            }
        }

        return c.Next()
    }
}
```

## ধাপ ৬: Auth Handler

**handlers/auth.go**

```go
package handlers

import (
    "auth-api/config"
    "auth-api/models"
    "auth-api/utils"

    "github.com/gofiber/fiber/v3"
    "golang.org/x/crypto/bcrypt"
)

type AuthHandler struct{}

func NewAuthHandler() *AuthHandler {
    return &AuthHandler{}
}

// POST /api/auth/register
func (h *AuthHandler) Register(c fiber.Ctx) error {
    var req models.RegisterRequest

    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // চেক করুন ইমেইল আগে থেকে আছে কিনা
    var existingUser models.User
    if err := config.DB.Where("email = ?", req.Email).First(&existingUser).Error; err == nil {
        return c.Status(fiber.StatusConflict).JSON(fiber.Map{
            "error": "Email already registered",
        })
    }

    // পাসওয়ার্ড হ্যাশ করুন
    hashedPassword, err := bcrypt.GenerateFromPassword(
        []byte(req.Password),
        bcrypt.DefaultCost,
    )
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to hash password",
        })
    }

    // নতুন ইউজার তৈরি করুন
    user := models.User{
        Name:     req.Name,
        Email:    req.Email,
        Password: string(hashedPassword),
        Role:     "user",
        IsActive: true,
    }

    if err := config.DB.Create(&user).Error; err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to create user",
        })
    }

    // JWT টোকেন জেনারেট করুন
    token, err := utils.GenerateToken(&user)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to generate token",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "message": "User registered successfully",
        "data": models.LoginResponse{
            Token: token,
            User:  &user,
        },
    })
}

// POST /api/auth/login
func (h *AuthHandler) Login(c fiber.Ctx) error {
    var req models.LoginRequest

    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // ইউজার খুঁজুন
    var user models.User
    if err := config.DB.Where("email = ?", req.Email).First(&user).Error; err != nil {
        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
            "error": "Invalid email or password",
        })
    }

    // চেক করুন ইউজার অ্যাক্টিভ কিনা
    if !user.IsActive {
        return c.Status(fiber.StatusForbidden).JSON(fiber.Map{
            "error": "Account is deactivated",
        })
    }

    // পাসওয়ার্ড ভেরিফাই করুন
    if err := bcrypt.CompareHashAndPassword(
        []byte(user.Password),
        []byte(req.Password),
    ); err != nil {
        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
            "error": "Invalid email or password",
        })
    }

    // JWT টোকেন জেনারেট করুন
    token, err := utils.GenerateToken(&user)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to generate token",
        })
    }

    return c.JSON(fiber.Map{
        "message": "Login successful",
        "data": models.LoginResponse{
            Token: token,
            User:  &user,
        },
    })
}

// GET /api/auth/me (Protected)
func (h *AuthHandler) GetProfile(c fiber.Ctx) error {
    userID := c.Locals("userID").(uint)

    var user models.User
    if err := config.DB.First(&user, userID).Error; err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "User not found",
        })
    }

    return c.JSON(fiber.Map{
        "data": user,
    })
}

// PUT /api/auth/change-password (Protected)
func (h *AuthHandler) ChangePassword(c fiber.Ctx) error {
    userID := c.Locals("userID").(uint)

    var req models.ChangePasswordRequest
    if err := c.Bind().JSON(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // ইউজার খুঁজুন
    var user models.User
    if err := config.DB.First(&user, userID).Error; err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "User not found",
        })
    }

    // পুরাতন পাসওয়ার্ড ভেরিফাই করুন
    if err := bcrypt.CompareHashAndPassword(
        []byte(user.Password),
        []byte(req.OldPassword),
    ); err != nil {
        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
            "error": "Old password is incorrect",
        })
    }

    // নতুন পাসওয়ার্ড হ্যাশ করুন
    hashedPassword, err := bcrypt.GenerateFromPassword(
        []byte(req.NewPassword),
        bcrypt.DefaultCost,
    )
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to hash password",
        })
    }

    // পাসওয়ার্ড আপডেট করুন
    user.Password = string(hashedPassword)
    if err := config.DB.Save(&user).Error; err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to update password",
        })
    }

    return c.JSON(fiber.Map{
        "message": "Password changed successfully",
    })
}

// POST /api/auth/logout (Optional - client-side টোকেন ডিলিট করবে)
func (h *AuthHandler) Logout(c fiber.Ctx) error {
    return c.JSON(fiber.Map{
        "message": "Logout successful",
    })
}
```

## ধাপ ৭: Main Application

**main.go**

```go
package main

import (
    "log"
    "os"

    "auth-api/config"
    "auth-api/handlers"
    "auth-api/middleware"
    "auth-api/models"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/gofiber/fiber/v3/middleware/recover"
    "github.com/joho/godotenv"
)

func main() {
    // .env লোড করুন
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found")
    }

    // ডাটাবেস কানেক্ট করুন
    config.ConnectDB()
    config.MigrateDB(&models.User{})

    // Fiber অ্যাপ তৈরি করুন
    app := fiber.New(fiber.Config{
        AppName: "Auth API v1.0",
    })

    // মিডলওয়্যার
    app.Use(recover.New())
    app.Use(logger.New())
    app.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"*"},
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
        AllowCredentials: true,
    }))

    // Handler ইনিশিয়ালাইজ করুন
    authHandler := handlers.NewAuthHandler()

    // রাউট
    api := app.Group("/api")

    // Public রাউট
    auth := api.Group("/auth")
    auth.Post("/register", authHandler.Register)
    auth.Post("/login", authHandler.Login)

    // Protected রাউট (Authentication প্রয়োজন)
    auth.Get("/me", middleware.AuthRequired(), authHandler.GetProfile)
    auth.Put("/change-password", middleware.AuthRequired(), authHandler.ChangePassword)
    auth.Post("/logout", middleware.AuthRequired(), authHandler.Logout)

    // Admin-only রাউট
    admin := api.Group("/admin")
    admin.Use(middleware.AuthRequired())
    admin.Use(middleware.RequireRole("admin"))
    admin.Get("/users", func(c fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "message": "Admin access granted",
        })
    })

    // সার্ভার শুরু করুন
    port := os.Getenv("PORT")
    if port == "" {
        port = "3000"
    }

    log.Fatal(app.Listen(":" + port))
}
```

## API টেস্ট করুন

### 1. রেজিস্টার করুন

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "রহিম আহমেদ",
    "email": "rahim@example.com",
    "password": "secure123"
  }'
```

**Response:**
```json
{
  "message": "User registered successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": 1,
      "name": "রহিম আহমেদ",
      "email": "rahim@example.com",
      "role": "user",
      "is_active": true
    }
  }
}
```

### 2. লগইন করুন

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "rahim@example.com",
    "password": "secure123"
  }'
```

### 3. প্রোফাইল দেখুন (Protected)

```bash
curl http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

### 4. পাসওয়ার্ড পরিবর্তন করুন

```bash
curl -X PUT http://localhost:3000/api/auth/change-password \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "old_password": "secure123",
    "new_password": "newsecure456"
  }'
```

## Security Best Practices

### 1. Environment Variables
```env
JWT_SECRET=use-a-very-long-random-string-here-min-32-chars
```

### 2. Password Hashing
```go
// bcrypt ব্যবহার করুন (cost 10-12)
hashedPassword, _ := bcrypt.GenerateFromPassword(
    []byte(password), 
    bcrypt.DefaultCost, // 10
)
```

### 3. Token Expiration
```go
// Short-lived access token (15 min - 1 hour)
ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Hour))

// Long-lived refresh token (7-30 days)
ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Hour * 24 * 7))
```

### 4. HTTPS Only
```go
// Production-এ শুধু HTTPS ব্যবহার করুন
app.Use(func(c fiber.Ctx) error {
    if c.Protocol() != "https" {
        return c.Redirect("https://" + c.Hostname() + c.OriginalURL())
    }
    return c.Next()
})
```

### 5. Rate Limiting
```go
import "github.com/gofiber/fiber/v3/middleware/limiter"

app.Use(limiter.New(limiter.Config{
    Max:        5,
    Expiration: 1 * time.Minute,
}))
```

---

[← আগের: ডাটাবেস ইন্টিগ্রেশন](02-database-integration.md) | [পরবর্তী: ফাইল আপলোড →](04-file-upload.md)
