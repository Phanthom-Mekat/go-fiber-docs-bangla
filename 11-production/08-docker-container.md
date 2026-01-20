# ডকার ও কনটেইনারাইজেশন (Docker & Containerization)

অ্যাপ্লিকেশন পোর্টেবিলিটির জন্য ডকার স্ট্যান্ডার্ড। Go অ্যাপ্লিকেশন কম্পাইল্ড বাইনারি হওয়ায় ইমেজ সাইজ অনেক ছোট রাখা সম্ভব।

## বেস্ট প্র্যাকটিস: মাল্টি-স্টেজ বিল্ড (Multi-Stage Build)

আমরা "Builder" ইমেজে কোড কম্পাইল করব এবং শুধুমাত্র বাইনারিটি একটি হালকা "Runner" ইমেজে কপি করব।

### Dockerfile (Distroless)

গুগলের `distroless` ইমেজ ব্যবহার করলে ইমেজে কোনো শেল (shell) থাকে না, যা সিকিউরিটি অনেক বাড়িয়ে দেয়।

```dockerfile
# Stage 1: Build
FROM golang:1.24-alpine AS builder

WORKDIR /app

# ক্যাশ লেয়ার এর সুবিধা নিতে আগে go.mod কপি করুন
COPY go.mod go.sum ./
RUN go mod download

# সোর্স কোড কপি ও বিল্ড
COPY . .
# CGO_ENABLED=0 স্ট্যাটিক বাইনারির জন্য জরুরি
# -ldflags="-s -w" ডিবাগ ইনফো রিমুভ করে সাইজ কমায়
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server ./cmd/api

# Stage 2: Runtime
# gcr.io/distroless/static-debian12 ব্যবহার করছি (অথবা alpine)
FROM gcr.io/distroless/static-debian12

# অ্যাপ রান করার জন্য নন-রুট ইউজার
USER nonroot:nonroot

WORKDIR /
COPY --from=builder --chown=nonroot:nonroot /app/server /server

# পোর্ট এক্সপোজ
EXPOSE 3000

# রান কমান্ড
CMD ["/server"]
```

## Docker Compose (লোকাল ডেভেলপমেন্ট)

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

## টিপস

1.  **.dockerignore:** `.git`, `node_modules`, বা কোনো সিক্রেট ফাইল যাতে ইমেজে না যায় তা নিশ্চিত করতে `.dockerignore` ব্যবহার করুন।
2.  **PID 1:** আপনি যদি Fiber-এর `Prefork` ব্যবহার করেন, তাহলে `dumb-init` বা `tini` এন্ট্রি পয়েন্ট হিসেবে ব্যবহার করতে হবে (যা পারফরমেন্স অধ্যায়ে আলোচনা করেছি)। Distroless ইমেজে এটি নাও থাকতে পারে, সেক্ষেত্রে `alpine` বেইস ইমেজ ব্যবহার করে `apk add dumb-init` করে নিতে হবে।

---
[← আগের: মনিটরিং](07-monitoring-health.md) | [সূচি](../README.md) | [পরবর্তী: CI/CD →](09-ci-cd-basics.md)
