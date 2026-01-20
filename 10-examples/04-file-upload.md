# ফাইল আপলোড (File Upload)

Fiber-এ ফাইল আপলোড, ইমেজ প্রসেসিং এবং ক্লাউড স্টোরেজ ইন্টিগ্রেশন শিখুন।

## প্রজেক্ট স্ট্রাকচার

```
file-upload-api/
├── main.go
├── handlers/
│   └── upload.go
├── utils/
│   ├── file.go
│   └── image.go
├── uploads/          # লোকাল স্টোরেজ
│   ├── images/
│   └── documents/
└── .env
```

## ধাপ ১: ডিপেন্ডেন্সি ইনস্টল করুন

```bash
go get github.com/gofiber/fiber/v3
go get github.com/google/uuid
go get github.com/disintegration/imaging  # ইমেজ রিসাইজিং
```

## ধাপ ২: File Utility

**utils/file.go**

```go
package utils

import (
    "fmt"
    "mime/multipart"
    "os"
    "path/filepath"
    "strings"

    "github.com/google/uuid"
)

// অনুমোদিত ফাইল টাইপ
var (
    AllowedImageTypes = []string{".jpg", ".jpeg", ".png", ".gif", ".webp"}
    AllowedDocTypes   = []string{".pdf", ".doc", ".docx", ".txt"}
    MaxFileSize       = int64(10 * 1024 * 1024) // 10MB
)

// ফাইল এক্সটেনশন চেক করুন
func IsAllowedFileType(filename string, allowedTypes []string) bool {
    ext := strings.ToLower(filepath.Ext(filename))
    for _, allowed := range allowedTypes {
        if ext == allowed {
            return true
        }
    }
    return false
}

// ইউনিক ফাইলনেম জেনারেট করুন
func GenerateUniqueFilename(originalFilename string) string {
    ext := filepath.Ext(originalFilename)
    return fmt.Sprintf("%s%s", uuid.New().String(), ext)
}

// ফাইল সেভ করুন
func SaveFile(file *multipart.FileHeader, destination string) error {
    // ডিরেক্টরি তৈরি করুন যদি না থাকে
    dir := filepath.Dir(destination)
    if err := os.MkdirAll(dir, 0755); err != nil {
        return err
    }

    // ফাইল সেভ করুন
    src, err := file.Open()
    if err != nil {
        return err
    }
    defer src.Close()

    dst, err := os.Create(destination)
    if err != nil {
        return err
    }
    defer dst.Close()

    _, err = dst.ReadFrom(src)
    return err
}

// ফাইল ডিলিট করুন
func DeleteFile(filepath string) error {
    return os.Remove(filepath)
}

// ফাইল সাইজ ফরম্যাট করুন
func FormatFileSize(size int64) string {
    const unit = 1024
    if size < unit {
        return fmt.Sprintf("%d B", size)
    }
    div, exp := int64(unit), 0
    for n := size / unit; n >= unit; n /= unit {
        div *= unit
        exp++
    }
    return fmt.Sprintf("%.1f %cB", float64(size)/float64(div), "KMGTPE"[exp])
}
```

## ধাপ ৩: Image Processing Utility

**utils/image.go**

```go
package utils

import (
    "fmt"
    "image"
    "path/filepath"

    "github.com/disintegration/imaging"
)

type ImageSize struct {
    Width  int
    Height int
    Name   string
}

var (
    // প্রিডিফাইন্ড সাইজ
    ThumbnailSize = ImageSize{Width: 150, Height: 150, Name: "thumbnail"}
    MediumSize    = ImageSize{Width: 500, Height: 500, Name: "medium"}
    LargeSize     = ImageSize{Width: 1200, Height: 1200, Name: "large"}
)

// ইমেজ রিসাইজ করুন
func ResizeImage(srcPath string, dstPath string, size ImageSize) error {
    // ইমেজ ওপেন করুন
    img, err := imaging.Open(srcPath)
    if err != nil {
        return fmt.Errorf("failed to open image: %w", err)
    }

    // রিসাইজ করুন (aspect ratio বজায় রেখে)
    resized := imaging.Fit(img, size.Width, size.Height, imaging.Lanczos)

    // সেভ করুন
    err = imaging.Save(resized, dstPath)
    if err != nil {
        return fmt.Errorf("failed to save image: %w", err)
    }

    return nil
}

// মাল্টিপল সাইজ জেনারেট করুন
func GenerateImageSizes(srcPath string, sizes []ImageSize) (map[string]string, error) {
    results := make(map[string]string)
    dir := filepath.Dir(srcPath)
    ext := filepath.Ext(srcPath)
    baseName := filepath.Base(srcPath[:len(srcPath)-len(ext)])

    for _, size := range sizes {
        dstPath := filepath.Join(dir, fmt.Sprintf("%s_%s%s", baseName, size.Name, ext))
        
        if err := ResizeImage(srcPath, dstPath, size); err != nil {
            return nil, err
        }
        
        results[size.Name] = dstPath
    }

    return results, nil
}

// ইমেজ ডাইমেনশন পান
func GetImageDimensions(filepath string) (int, int, error) {
    img, err := imaging.Open(filepath)
    if err != nil {
        return 0, 0, err
    }

    bounds := img.Bounds()
    return bounds.Dx(), bounds.Dy(), nil
}

// ইমেজ ক্রপ করুন
func CropImage(srcPath string, dstPath string, x, y, width, height int) error {
    img, err := imaging.Open(srcPath)
    if err != nil {
        return err
    }

    cropped := imaging.Crop(img, image.Rect(x, y, x+width, y+height))
    return imaging.Save(cropped, dstPath)
}

// ওয়াটারমার্ক যোগ করুন
func AddWatermark(srcPath string, watermarkPath string, dstPath string) error {
    // মূল ইমেজ
    img, err := imaging.Open(srcPath)
    if err != nil {
        return err
    }

    // ওয়াটারমার্ক
    watermark, err := imaging.Open(watermarkPath)
    if err != nil {
        return err
    }

    // ওয়াটারমার্ক রিসাইজ করুন (মূল ইমেজের 20%)
    bounds := img.Bounds()
    watermarkWidth := bounds.Dx() / 5
    watermark = imaging.Resize(watermark, watermarkWidth, 0, imaging.Lanczos)

    // নিচে ডানে পজিশন করুন
    offset := image.Pt(
        bounds.Dx()-watermark.Bounds().Dx()-20,
        bounds.Dy()-watermark.Bounds().Dy()-20,
    )

    // ওভারলে করুন
    result := imaging.Overlay(img, watermark, offset, 0.5)

    return imaging.Save(result, dstPath)
}
```

## ধাপ ৪: Upload Handler

**handlers/upload.go**

```go
package handlers

import (
    "fmt"
    "path/filepath"
    "file-upload-api/utils"

    "github.com/gofiber/fiber/v3"
)

type UploadHandler struct {
    uploadDir string
}

func NewUploadHandler(uploadDir string) *UploadHandler {
    return &UploadHandler{uploadDir: uploadDir}
}

// POST /api/upload/single - সিঙ্গেল ফাইল আপলোড
func (h *UploadHandler) UploadSingle(c fiber.Ctx) error {
    // ফাইল পান
    file, err := c.FormFile("file")
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "No file uploaded",
        })
    }

    // ফাইল সাইজ চেক করুন
    if file.Size > utils.MaxFileSize {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": fmt.Sprintf("File too large. Max size: %s", 
                utils.FormatFileSize(utils.MaxFileSize)),
        })
    }

    // ফাইল টাইপ চেক করুন
    if !utils.IsAllowedFileType(file.Filename, utils.AllowedImageTypes) {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid file type. Allowed: jpg, jpeg, png, gif, webp",
        })
    }

    // ইউনিক ফাইলনেম জেনারেট করুন
    filename := utils.GenerateUniqueFilename(file.Filename)
    destination := filepath.Join(h.uploadDir, "images", filename)

    // ফাইল সেভ করুন
    if err := utils.SaveFile(file, destination); err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to save file",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "message": "File uploaded successfully",
        "data": fiber.Map{
            "filename":     filename,
            "original_name": file.Filename,
            "size":         utils.FormatFileSize(file.Size),
            "url":          fmt.Sprintf("/uploads/images/%s", filename),
        },
    })
}

// POST /api/upload/multiple - মাল্টিপল ফাইল আপলোড
func (h *UploadHandler) UploadMultiple(c fiber.Ctx) error {
    // মাল্টিপার্ট ফর্ম পান
    form, err := c.MultipartForm()
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid form data",
        })
    }

    files := form.File["files"]
    if len(files) == 0 {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "No files uploaded",
        })
    }

    // সর্বোচ্চ ফাইল সংখ্যা চেক করুন
    if len(files) > 10 {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Maximum 10 files allowed",
        })
    }

    uploadedFiles := []fiber.Map{}
    errors := []string{}

    for _, file := range files {
        // ফাইল সাইজ চেক করুন
        if file.Size > utils.MaxFileSize {
            errors = append(errors, fmt.Sprintf("%s: file too large", file.Filename))
            continue
        }

        // ফাইল টাইপ চেক করুন
        if !utils.IsAllowedFileType(file.Filename, utils.AllowedImageTypes) {
            errors = append(errors, fmt.Sprintf("%s: invalid file type", file.Filename))
            continue
        }

        // ফাইল সেভ করুন
        filename := utils.GenerateUniqueFilename(file.Filename)
        destination := filepath.Join(h.uploadDir, "images", filename)

        if err := utils.SaveFile(file, destination); err != nil {
            errors = append(errors, fmt.Sprintf("%s: failed to save", file.Filename))
            continue
        }

        uploadedFiles = append(uploadedFiles, fiber.Map{
            "filename":      filename,
            "original_name": file.Filename,
            "size":          utils.FormatFileSize(file.Size),
            "url":           fmt.Sprintf("/uploads/images/%s", filename),
        })
    }

    response := fiber.Map{
        "message": fmt.Sprintf("%d files uploaded successfully", len(uploadedFiles)),
        "data":    uploadedFiles,
    }

    if len(errors) > 0 {
        response["errors"] = errors
    }

    return c.Status(fiber.StatusCreated).JSON(response)
}

// POST /api/upload/image-resize - ইমেজ আপলোড ও রিসাইজ
func (h *UploadHandler) UploadAndResize(c fiber.Ctx) error {
    file, err := c.FormFile("file")
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "No file uploaded",
        })
    }

    // ভ্যালিডেশন
    if file.Size > utils.MaxFileSize {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "File too large",
        })
    }

    if !utils.IsAllowedFileType(file.Filename, utils.AllowedImageTypes) {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid file type",
        })
    }

    // অরিজিনাল ফাইল সেভ করুন
    filename := utils.GenerateUniqueFilename(file.Filename)
    originalPath := filepath.Join(h.uploadDir, "images", "original", filename)

    if err := utils.SaveFile(file, originalPath); err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to save file",
        })
    }

    // মাল্টিপল সাইজ জেনারেট করুন
    sizes := []utils.ImageSize{
        utils.ThumbnailSize,
        utils.MediumSize,
        utils.LargeSize,
    }

    resizedPaths, err := utils.GenerateImageSizes(originalPath, sizes)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to resize images",
        })
    }

    // URL তৈরি করুন
    urls := make(map[string]string)
    urls["original"] = fmt.Sprintf("/uploads/images/original/%s", filename)
    
    for sizeName, path := range resizedPaths {
        relPath, _ := filepath.Rel(h.uploadDir, path)
        urls[sizeName] = fmt.Sprintf("/uploads/%s", filepath.ToSlash(relPath))
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "message": "Image uploaded and resized successfully",
        "data": fiber.Map{
            "filename": filename,
            "urls":     urls,
        },
    })
}

// DELETE /api/upload/:filename - ফাইল ডিলিট করুন
func (h *UploadHandler) DeleteFile(c fiber.Ctx) error {
    filename := c.Params("filename")
    filepath := filepath.Join(h.uploadDir, "images", filename)

    if err := utils.DeleteFile(filepath); err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "File not found",
        })
    }

    return c.JSON(fiber.Map{
        "message": "File deleted successfully",
    })
}
```

## ধাপ ৫: Main Application

**main.go**

```go
package main

import (
    "log"
    "file-upload-api/handlers"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
)

func main() {
    app := fiber.New(fiber.Config{
        BodyLimit: 10 * 1024 * 1024, // 10MB
    })

    // মিডলওয়্যার
    app.Use(logger.New())
    app.Use(cors.New())

    // স্ট্যাটিক ফাইল সার্ভ করুন
    app.Static("/uploads", "./uploads")

    // Handler ইনিশিয়ালাইজ করুন
    uploadHandler := handlers.NewUploadHandler("./uploads")

    // রাউট
    api := app.Group("/api/upload")
    api.Post("/single", uploadHandler.UploadSingle)
    api.Post("/multiple", uploadHandler.UploadMultiple)
    api.Post("/image-resize", uploadHandler.UploadAndResize)
    api.Delete("/:filename", uploadHandler.DeleteFile)

    log.Fatal(app.Listen(":3000"))
}
```

## API টেস্ট করুন

### 1. সিঙ্গেল ফাইল আপলোড

```bash
curl -X POST http://localhost:3000/api/upload/single \
  -F "file=@/path/to/image.jpg"
```

### 2. মাল্টিপল ফাইল আপলোড

```bash
curl -X POST http://localhost:3000/api/upload/multiple \
  -F "files=@/path/to/image1.jpg" \
  -F "files=@/path/to/image2.jpg" \
  -F "files=@/path/to/image3.jpg"
```

### 3. ইমেজ আপলোড ও রিসাইজ

```bash
curl -X POST http://localhost:3000/api/upload/image-resize \
  -F "file=@/path/to/image.jpg"
```

**Response:**
```json
{
  "message": "Image uploaded and resized successfully",
  "data": {
    "filename": "550e8400-e29b-41d4-a716-446655440000.jpg",
    "urls": {
      "original": "/uploads/images/original/550e8400-e29b-41d4-a716-446655440000.jpg",
      "thumbnail": "/uploads/images/550e8400-e29b-41d4-a716-446655440000_thumbnail.jpg",
      "medium": "/uploads/images/550e8400-e29b-41d4-a716-446655440000_medium.jpg",
      "large": "/uploads/images/550e8400-e29b-41d4-a716-446655440000_large.jpg"
    }
  }
}
```

### 4. ফাইল ডিলিট করুন

```bash
curl -X DELETE http://localhost:3000/api/upload/550e8400-e29b-41d4-a716-446655440000.jpg
```

## HTML Form Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>File Upload</title>
</head>
<body>
    <h1>সিঙ্গেল ফাইল আপলোড</h1>
    <form action="http://localhost:3000/api/upload/single" 
          method="POST" 
          enctype="multipart/form-data">
        <input type="file" name="file" accept="image/*" required>
        <button type="submit">আপলোড করুন</button>
    </form>

    <h1>মাল্টিপল ফাইল আপলোড</h1>
    <form action="http://localhost:3000/api/upload/multiple" 
          method="POST" 
          enctype="multipart/form-data">
        <input type="file" name="files" accept="image/*" multiple required>
        <button type="submit">আপলোড করুন</button>
    </form>
</body>
</html>
```

## Best Practices

### 1. File Validation
```go
// সাইজ, টাইপ, এক্সটেনশন চেক করুন
if file.Size > MaxFileSize {
    return errors.New("file too large")
}
```

### 2. Unique Filenames
```go
// UUID ব্যবহার করুন collision এড়াতে
filename := uuid.New().String() + filepath.Ext(originalName)
```

### 3. Directory Structure
```
uploads/
├── images/
│   ├── original/
│   ├── thumbnail/
│   └── medium/
└── documents/
```

### 4. Security
```go
// ফাইল টাইপ ভেরিফাই করুন (শুধু এক্সটেনশন নয়)
import "net/http"

func detectContentType(file multipart.File) (string, error) {
    buffer := make([]byte, 512)
    _, err := file.Read(buffer)
    if err != nil {
        return "", err
    }
    contentType := http.DetectContentType(buffer)
    file.Seek(0, 0) // রিসেট করুন
    return contentType, nil
}
```

---

[← আগের: JWT Authentication](03-jwt-authentication.md) | [পরবর্তী: WebSocket Chat →](05-websocket-chat.md)
