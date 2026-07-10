
當然，這種方式通常稱為 **multi-stage builds**，主要用於減少最終映像檔的大小，並且只包含執行應用程式所需的部分，而不是所有的編譯工具和依賴項。以下是示範如何使用兩段式的 Dockerfile：

```dockerfile
# 第一階段：建置應用程式
FROM golang:1.20 AS builder

# 設定工作目錄
WORKDIR /app

# 複製原始碼
COPY . .

# 編譯應用程式
RUN go build -o myapp

# 第二階段：建立精簡的執行環境
FROM alpine:latest

# 設定工作目錄
WORKDIR /root/

# 複製第一階段的執行檔
COPY --from=builder /app/myapp .

# 指定容器啟動時執行的指令
CMD ["./myapp"]
```

在這個例子中：

1. **第一階段 (`builder`)**：使用 `golang:1.20` 來編譯 Go 程式碼。
2. **第二階段 (`alpine:latest`)**：我們只從 `builder` 階段取出編譯好的執行檔，而不包含 Go 編譯器，減少映像檔的大小。

這樣的方式不僅讓最終的 Docker 映像檔更輕量，也能提高安全性，避免暴露不必要的工具或文件。希望這個示範對你有幫助！你打算在什麼樣的應用場景使用這個技術呢？ 🚀