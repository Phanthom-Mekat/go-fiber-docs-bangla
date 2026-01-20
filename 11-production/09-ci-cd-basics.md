# কন্টিনিউয়াস ইন্টিগ্রেশন ও ডিপ্লয়মেন্ট (CI/CD)

একটি আধুনিক টিমে ম্যানুয়ালি `go build` বা `scp` করে সার্ভারে কোড পাঠানো উচিত নয়। CI/CD পাইপলাইন অটোমেটিক্যালি টেস্ট রান করে এবং সফল হলে কোড ডিপ্লয় করে।

## GitHub Actions (উদাহরণ)

আপনার রিপোজিটরির `.github/workflows/go.yml` ফাইলে নিচের কনফিগারেশনটি ব্যবহার করতে পারেন।

### বিল্ড এবং টেস্ট পাইপলাইন

```yaml
name: Go Build & Test

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.24'

    - name: Install dependencies
      run: go mod download

    - name: Verify
      run: go vet ./...

    - name: Run Tests
      # -race ফ্ল্যাগ দিয়ে রেস কন্ডিশন চেক করা ভালো
      run: go test -race -v ./...

    - name: Build
      run: go build -v ./...
```

## লিন্টিং (Linting)

Go কোডের কোয়ালিটি বজায় রাখার জন্য `golangci-lint` ব্যবহার করা ইন্ডাস্ট্রি স্ট্যান্ডার্ড।

```yaml
    - name: GolangCI-Lint
      uses: golangci/golangci-lint-action@v4
      with:
        version: latest
```

## ডিপ্লয়মেন্ট স্ট্র্যাটেজি

CI টেস্ট পাস করার পর CD (Continuous Deployment) স্টেজে আপনি:
1.  **Docker Push:** নতুন ইমেজ বিল্ড করে Docker Hub বা AWS ECR এ পুশ করতে পারেন।
2.  **SSH Deploy:** সার্ভারে SSH করে `git pull` এবং `go build` কমান্ড চালাতে পারেন (ছোট প্রজেক্টের জন্য)।
3.  **Kubernetes:** Helm চার্ট আপডেট করে নতুন ইমেজ রোল আউট করতে পারেন।

---
[← আগের: ডকার](08-docker-container.md) | [সূচি](../README.md) | [পরবর্তী: স্কেলিং →](10-scaling.md)
