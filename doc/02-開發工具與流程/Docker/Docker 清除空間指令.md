你可以使用以下指令來刪除無效的網路和建置快取：

1. **刪除未使用的網路**：
    
    ```sh
    docker network prune
    ```
    
    這個指令會刪除所有未使用的 Docker 網路。
    
2. **刪除建置快取**：
    
    ```sh
    docker builder prune
    ```
    
    這個指令會刪除所有未使用的建置快取，以釋放磁碟空間。
    

如果你想要更徹底地清理 Docker 的所有未使用資源，可以使用：

```sh
docker system prune -a
```

這將移除所有未使用的影像、停止的容器、無效的網路以及建置快取。
未使用是指未啟動 container 的都視為未使用。

---

Docker `builder` 提供了一系列參數來控制建置過程，讓你可以更靈活地管理映像的建構方式。以下是一些常見的 `docker builder` 參數及其用途：

### **常見參數**

1. **`--build-arg <ARG_NAME>=<value>`**
    
    - 用途：傳遞建置時的變數，影響 Dockerfile 中的 `ARG` 指令。
    - 使用方式：
        
        ```sh
        docker build --build-arg NODE_ENV=production -t my-app .
        ```
        
    - 在 Dockerfile 中：
        
        ```dockerfile
        ARG NODE_ENV
        RUN echo "Building for environment: $NODE_ENV"
        ```
        
2. **`--file, -f <Dockerfile>`**
    
    - 用途：指定要使用的 Dockerfile（預設為 `Dockerfile`）。
    - 使用方式：
        
        ```sh
        docker build -f ./custom-dir/MyDockerfile -t custom-image .
        ```
        
3. **`--tag, -t <name:tag>`**
    
    - 用途：為建構出的映像打上標籤，以便於管理。
    - 使用方式：
        
        ```sh
        docker build -t my-app:latest .
        ```
        
4. **`--no-cache`**
    
    - 用途：禁用快取，確保所有步驟都重新執行。
    - 使用方式：
        
        ```sh
        docker build --no-cache -t my-app .
        ```
        
5. **`--target <stage>`**
    
    - 用途：在多階段建構中，指定要建構的特定階段。
    - 使用方式：
        
        ```sh
        docker build --target builder -t my-app-builder .
        ```
        
    - 在 Dockerfile 中：
        
        ```dockerfile
        FROM node:16 AS builder
        RUN npm install
        FROM node:16 AS production
        COPY --from=builder /app /app
        ```
        
6. **`--network <mode>`**
    
    - 用途：設定建構時的網路模式，例如 `default`、`none`、`host`。
    - 使用方式：
        
        ```sh
        docker build --network host -t my-app .
        ```
        
7. **`--progress <mode>`**
    
    - 用途：控制建構進度顯示方式，例如 `auto`、`plain`、`tty`。
    - 使用方式：
        
        ```sh
        docker build --progress=plain -t my-app .
        ```
        

這些參數可以幫助你更有效率地建構 Docker 映像，並根據不同需求調整建構方式。

---

Docker `builder` 指令主要用於管理建置快取和建置上下文，並提供一些有用的參數來控制建置行為。以下是一些常見的 `docker builder` 相關參數及其用途：

### **常見 `docker builder` 參數**

1. **`docker builder prune`**
    
    - 用途：刪除未使用的建置快取，以釋放磁碟空間。
    - 使用方式：
        
        ```sh
        docker builder prune
        ```
        
    - 如果要刪除所有建置快取（包括共享快取），可以加上 `--all`：
        
        ```sh
        docker builder prune --all
        ```
        
2. **`docker builder build`**
    
    - 用途：使用 BuildKit 來建構映像，類似於 `docker build`，但提供更強大的功能。
    - 使用方式：
        
        ```sh
        docker builder build -t my-app .
        ```
        
    - 可以使用 `--progress` 來控制建置進度顯示：
        
        ```sh
        docker builder build --progress=plain -t my-app .
        ```
        
3. **`docker builder inspect`**
    
    - 用途：檢視建置器的詳細資訊，例如快取狀態。
    - 使用方式：
        
        ```sh
        docker builder inspect
        ```
        
4. **`docker builder create`**
    
    - 用途：建立新的 BuildKit 建置器（適用於進階使用）。
    - 使用方式：
        
        ```sh
        docker builder create --name my-builder
        ```
        

這些指令可以幫助你更有效率地管理 Docker 的建置過程，特別是在使用 BuildKit 進行建置時。

---

Docker `builder` 提供多種指令來管理建置過程，其中 `bake`、`build` 和 `create` 各有不同的用途：

### **1. `docker builder bake`**

- **用途**：用於批量建置多個映像，並支援多平台建置。
- **特點**：
    - 使用 `docker-bake.hcl` 或 JSON/YAML 檔案來定義建置目標。
    - 簡化多階段建置，適合 CI/CD 工作流程。
    - 支援並行建置，提高效率。
- **使用方式**：
    
    ```sh
    docker buildx bake
    ```
    
    這會根據 `docker-bake.hcl` 檔案中的設定來執行建置.

### **2. `docker builder build`**

- **用途**：類似 `docker build`，但使用 BuildKit 來建構映像。
- **特點**：
    - 提供更強大的快取機制。
    - 支援多平台建置。
    - 可使用 `--progress` 來控制建置輸出格式。
- **使用方式**：
    
    ```sh
    docker builder build -t my-app .
    ```
    
    這與 `docker build` 類似，但能更靈活地管理建置過程.

### **3. `docker builder create`**

- **用途**：建立新的 BuildKit 建置器，適用於進階使用。
- **特點**：
    - 允許建立多個建置器，並在不同建置器間切換。
    - 適合需要不同建置環境的情境。
- **使用方式**：
    
    ```sh
    docker builder create --name my-builder
    ```
    
    這會建立一個新的建置器，並可透過 `docker builder use my-builder` 來使用它.

這三個指令各有用途，`bake` 適合批量建置，`build` 用於一般建置，而 `create` 則用於管理不同的建置環境。