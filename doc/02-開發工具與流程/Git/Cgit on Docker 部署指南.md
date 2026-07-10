
本專案使用 Docker Compose 與 Alpine Linux 輕量化部署 `cgit`。透過建置階段注入 Host 端的 UID/GID，完美解決 Git 2.35.2+ 的讀取權限問題，並支援透過環境變數動態修改網頁上顯示的 Clone IP。

## 專案目錄結構

請確保在你的工作目錄下建立以下四個檔案：

```
.
├── .env                 # 環境變數與 IP 設定
├── docker-compose.yml   # 容器編排設定
├── Dockerfile           # 容器建置腳本
└── cgitrc.template      # cgit 設定檔樣板
```

## 1. 準備工作 (環境設定)

### 1.1 查詢 Host 端用戶 ID

為了確保容器內的 cgit 有權限讀取 Host 端的 Git 儲存庫，請在 Host 端終端機查詢你用來管理 git 專案的帳號 UID 與 GID（例如執行 `id` 或 `id git`）。

### 1.2 建立軟連結 (Symlink)

由於我們希望 Clone URL 顯示為 `git@<IP>:/repos/...`，請在 Host 端系統根目錄建立對應的軟連結，指向實際存放 repo 的硬碟位置：

```bash
sudo ln -s /media/ssd2/repos /repos
```

### 1.3 建立 `.env` 檔案

在專案目錄下建立 `.env`，填入你查詢到的 UID/GID，以及 Host 端目前的 IP 位址：

```
# 請替換為 Host 端的實際 UID 與 GID
HOST_UID=1000
HOST_GID=1000
# 請替換為 Host 端的實際 IP 位址
HOST_IP=192.168.0.22
```

## 2. 建立設定檔

### 2.1 `docker-compose.yml`

負責定義服務、掛載唯讀的原始碼目錄與樣板檔，並將本機的 `8088` Port 映射至容器的 `8080` Port。

```yaml
services:
  cgit:
    build:
      context: .
      args:
        - HOST_UID=${HOST_UID:-1000}
        - HOST_GID=${HOST_GID:-1000}
    container_name: cgit
    # 以 Host 端對應的 UID/GID 運行，避開 git dubious ownership 檢查
    user: "${HOST_UID}:${HOST_GID}"
    ports:
      # Host Port : Container Port (若 8088 衝突請修改左側數字)
      - "8088:8080"
    environment:
      - HOST_IP=${HOST_IP}
    volumes:
      # 掛載 Host 端的 git repositories (唯讀)
      - /media/ssd2/repos:/var/git/repos:ro
      # 掛載樣板檔，啟動時會動態產生真正的 /etc/cgitrc (唯讀)
      - ./cgitrc.template:/etc/cgitrc.template:ro
    restart: unless-stopped
```

### 2.2 `Dockerfile`

基於 Alpine，安裝 `lighttpd` 與 `gettext` (提供 `envsubst`)。並在內部真實建立對應的用戶實體。

```dockerfile
FROM alpine:latest

# 接收 docker-compose 傳遞的 Build Arguments
ARG HOST_UID=1000
ARG HOST_GID=1000

# 安裝核心套件與 gettext (用於 envsubst)
RUN apk add --no-cache cgit lighttpd git gettext

# 建立與 Host 端完全相同的用戶與群組
RUN addgroup -g ${HOST_GID} gitgroup && \
    adduser -D -u ${HOST_UID} -G gitgroup gituser

# 建立 cache 資料夾並賦予權限
RUN mkdir -p /var/cache/cgit && \
    chown -R gituser:gitgroup /var/cache/cgit

# 設定 lighttpd 參數
RUN echo 'server.document-root = "/usr/share/webapps/cgit/"' > /etc/lighttpd/lighttpd.conf && \
    echo 'server.port = 8080' >> /etc/lighttpd/lighttpd.conf && \
    echo 'server.modules += ( "mod_cgi", "mod_alias", "mod_setenv" )' >> /etc/lighttpd/lighttpd.conf && \
    echo 'cgi.assign = ( "cgit.cgi" => "" )' >> /etc/lighttpd/lighttpd.conf && \
    echo 'index-file.names = ( "cgit.cgi" )' >> /etc/lighttpd/lighttpd.conf && \
    echo 'setenv.add-environment = ( "CGIT_CONFIG" => "/etc/cgitrc" )' >> /etc/lighttpd/lighttpd.conf

# 確保 gituser 對 /etc/cgitrc 有寫入權限
RUN touch /etc/cgitrc && chown gituser:gitgroup /etc/cgitrc

# 建立啟動腳本：先替換變數產生真實設定，再啟動 web server
RUN echo '#!/bin/sh' > /entrypoint.sh && \
    echo 'envsubst < /etc/cgitrc.template > /etc/cgitrc' >> /entrypoint.sh && \
    echo 'exec lighttpd -D -f /etc/lighttpd/lighttpd.conf' >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
```

### 2.3 `cgitrc.template`

Cgit 的介面與功能設定。啟動時，容器會自動將 `${HOST_IP}` 替換為 `.env` 檔案中的設定值。

```toml
css=/cgit.css
logo=/cgit.png
cache-size=1000

# 網頁標題與描述
root-title=My Git Repositories
root-desc=Local Git Hosting

# 動態替換 IP，並指向絕對路徑 /repos/
clone-url=git@${HOST_IP}:/repos/$CGIT_REPO_URL

# 自動掃描掛載進來的目錄
scan-path=/var/git/repos
```

## 3. 啟動與管理

完成上述檔案建立後，執行以下指令建置並啟動服務：

```bash
docker-compose up -d --build
```

啟動成功後，打開瀏覽器訪問 `http://<你的_Host_IP>:8088/cgit.cgi` 即可查看。

### IP 變更處理方式

如果未來 Host 端 IP 發生變化，只需要：

1. 修改 `.env` 檔案中的 `HOST_IP`。
    
2. 執行指令重啟容器：`docker-compose restart cgit`。
    
    (重啟過程會自動觸發腳本重新生成帶有新 IP 的設定檔，不需要重新 build)。
    

## 附錄：Port 衝突排解

如果在啟動時遇到 `address already in use` 錯誤，表示指定的 Port (如 8088) 已被佔用。可透過以下指令檢查 Host 端的 Port 使用狀況：

**使用 `ss` (推薦)：**

```bash
sudo ss -tulpn | grep :8088
```

**使用 `lsof`：**

```bash
sudo lsof -i :8088
```

確認衝突的服務後，可選擇關閉該服務，或者修改 `docker-compose.yml` 中的左側 Port 號，再重新啟動 `docker-compose up -d` 即可。