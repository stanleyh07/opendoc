# Docker 容器終端機自動啟動詳細設定說明

## 設定目標

讓在這個專案中開啟 VSCode 終端機時，自動進入 Docker 容器環境，無需手動執行任何命令。

## 已建立的設定檔案

### 1. dev-shell.sh - Docker 容器啟動腳本

**路徑**: `<Project>/dev-shell.sh`

**功能**:
- 自動檢查 Docker 服務狀態
- 檢查容器是否存在及運行狀態
- 必要時啟動或創建容器
- 使用 `exec` 命令無縫進入容器終端機

**關鍵設定**:
```bash
#!/bin/bash
set -e

# 設定變數
CONTAINER_NAME="ai-plan-generator"
COMPOSE_FILE="docker/docker-compose.yml"

# 檢查 Docker 服務
if ! docker info > /dev/null 2>&1; then
    echo "錯誤: Docker 服務未運行，請先啟動 Docker"
    exit 1
fi

# 檢查容器映像檔是否存在
if docker images --format "{{.Repository}}" | grep -q docker-"$CONTAINER_NAME"; then
    # 檢查容器是否正在運行
    if ! docker ps --format "{{.Names}}\t{{.Status}}" | grep -q "$CONTAINER_NAME"; then
        echo "啟動容器..."
        docker compose -f $COMPOSE_FILE up -d
        sleep 5
    fi
else
    echo "容器不存在，創建並啟動容器..."
    docker compose -f $COMPOSE_FILE up -d --build
    sleep 5
fi

# 直接進入容器終端機
exec docker exec -it $CONTAINER_NAME /bin/bash
```

**重要技術細節**:
- 使用 `exec` 命令取代當前 shell 進程，實現無縫切換
- 靜默模式運行，減少終端機輸出干擾
- 自動處理容器狀態檢查和啟動

### 2. .vscode/settings.json - VSCode 工作區設定

**路徑**: `<Project>/.vscode/settings.json`

**功能**:
- 設定預設終端機為自訂的 Docker 容器設定檔
- 定義終端機設定檔指向我們的啟動腳本

**關鍵設定**:
```json
{
  "terminal.integrated.defaultProfile.linux": "Docker Container",  
  "terminal.integrated.profiles.linux": {
    "Docker Container": {
      "path": "/bin/bash",
      "args": ["-c", "./dev-shell.sh"],
      "icon": "terminal-bash",
      "overrideName": true
    }
  },  
  "terminal.integrated.cwd": "${workspaceFolder}"
}
```

**設定說明**:
- `terminal.integrated.defaultProfile.linux`: 設定 Linux 系統下的預設終端機設定檔
- `terminal.integrated.profiles.linux`: 定義自訂終端機設定檔
  - `path`: 指向我們的啟動腳本完整路徑
  - `icon`: 設定終端機圖示
  - `overrideName`: 覆蓋終端機名稱顯示
- `terminal.integrated.cwd`: 設定終端機的初始工作目錄

## 工作原理

### 終端機開啟流程

1. **使用者動作**: 在 VSCode 中開啟終端機 (Ctrl+`)
2. **VSCode 處理**: 讀取工作區設定，發現預設終端機設定為 "Docker Container"
3. **執行腳本**: 執行 `./dev-shell.sh`
4. **容器檢查**: 腳本檢查 Docker 容器狀態
5. **容器啟動**: 如果容器未運行，自動啟動容器
6. **進入容器**: 使用 `docker exec -it` 進入容器終端機
7. **無縫體驗**: 使用者直接看到容器內的終端機環境

### 技術實現細節

**使用 `exec` 命令的優勢**:
- 取代當前 shell 進程，而不是創建子進程
- 保持終端機會話的連續性
- 退出容器時會正確關閉終端機

**容器狀態管理**:
- 使用 `docker ps -a` 檢查容器是否存在
- 使用 `docker ps` 檢查容器是否運行中
- 使用 `docker-compose up -d` 後台啟動容器

## 驗證設定

### 檢查設定是否生效

1. **開啟 VSCode 終端機**: 按下 `Ctrl+``
2. **觀察終端機標題**: 應該顯示為 "Docker Container"
3. **檢查提示符號**: 應該顯示容器內的提示符號（通常是 `root@容器ID:/app#`）
4. **驗證工作目錄**: 執行 `pwd` 應該顯示 `/app`

### 測試功能

1. **檢查 Python 環境**:
   ```bash
   python --version
   pip list
   ```

2. **檢查專案檔案**:
   ```bash
   ls -la
   ```

3. **運行測試命令**:
   ```bash
   python -c "print('Hello from Docker container')"
   ```

## 故障排除

### 常見問題

1. **Docker 服務未運行**
   ```bash
   # 檢查 Docker 狀態
   systemctl status docker
   # 啟動 Docker 服務
   sudo systemctl start docker
   ```

2. **權限問題**
   ```bash
   # 確保腳本有執行權限
   chmod +x dev-shell.sh
   ```

3. **容器啟動失敗**
   ```bash
   # 檢查容器日誌
   docker-compose -f docker/docker-compose.yml logs
   ```

4. **設定未生效**
   - 重啟 VSCode
   - 檢查 `.vscode/settings.json` 語法是否正確
   - 確認路徑設定正確

### 手動測試腳本

如果自動設定有問題，可以手動測試：
```bash
./dev-shell.sh
```

## 自訂設定

### 修改容器名稱

如果需要修改容器名稱，編輯 `dev-shell.sh`:
```bash
CONTAINER_NAME="your-new-container-name"
```

注意使用 docker-compose.yml 建立的 docker name 會根據 yml 檔案定義再加上檔案所在的目錄名稱，例如
```bash
# 所在目錄名稱為docker，名稱會變成 docker-"your-name"
CONTAINER_NAME=docker-"your-new-container-name"
```

### 修改 Docker Compose 檔案路徑

如果需要使用不同的 compose 檔案:
```bash
COMPOSE_FILE="path/to/your/docker-compose.yml"
```

### 修改終端機設定

如果需要調整終端機行為，編輯 `.vscode/settings.json`:
(注意不一定有效，因為已經進入docker container 的終端機，非原來 host 端的終端機)
```json
{
  "terminal.integrated.profiles.linux": {
    "Docker Container": {
      "path": "/home/arbor/source/RFQ/dev-shell.sh",
      "args": [],  // 可添加額外參數
      "icon": "terminal-bash",
      "color": "terminal.ansiGreen",  // 設定顏色
      "overrideName": true
    }
  }
}
```

## 恢復原始設定

如果需要恢復到正常的終端機行為：

1. **刪除或重命名設定檔案**:
   ```bash
   mv .vscode/settings.json .vscode/settings.json.backup
   ```

2. **或修改設定使用預設終端機**:
   ```json
   {
     "terminal.integrated.defaultProfile.linux": null
   }
   ```

這個設定確保了在這個專案中開啟終端機時，會自動在正確的 Docker 容器環境中運行，為開發和測試提供一致的環境。
