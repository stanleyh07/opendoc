
本文件總結了前面討論的完整設定，提供 **Dockerfile**、`docker-compose.yml` 與 `.env` 範例，並解釋各檔案用途、使用方法，以及啟動前的注意事項與環境設定。

---
## 📂 專案檔案結構

```plaintext
.
├── Dockerfile.sdkmanager    # 建立 Ubuntu 20.04 + SDK Manager 基底映像
├── docker-compose.yml       # 定義服務、掛載與使用者 UID/GID
├── .env                     # 主機使用者與環境變數
└── entrypoint.sh            # 啟動容器時調整權限並切換使用者
````

---
## 🐳 Dockerfile.sdkmanager

```dockerfile
# 以 Ubuntu 20.04 為基底
ARG UBUNTU_TAG=20.04
FROM ubuntu:${UBUNTU_TAG}

ENV DEBIAN_FRONTEND=noninteractive

# 安裝必要工具與 X11 測試套件
RUN apt-get update && apt-get install -y \
    ca-certificates gnupg wget sudo locales x11-apps \
    && rm -rf /var/lib/apt/lists/*

# Locale 設定
RUN sed -i 's/# en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

# 安裝 NVIDIA SDK Manager
RUN wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-keyring_1.1-1_all.deb && \
    dpkg -i cuda-keyring_1.1-1_all.deb && \
    rm cuda-keyring_1.1-1_all.deb && \
    apt-get update && \
    apt-get install -y sdkmanager && \
    rm -rf /var/lib/apt/lists/*

# 建立與主機相同 UID/GID 的一般使用者
ARG USERNAME
ARG UID
ARG GID
RUN groupadd -g ${GID} ${USERNAME} && \
    useradd -m -u ${UID} -g ${GID} -s /bin/bash ${USERNAME} && \
    usermod -aG sudo ${USERNAME} && \
    echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/99-${USERNAME}

# 工作目錄
WORKDIR /workspace
RUN chown -R ${UID}:${GID} /workspace

# 複製 entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["sdkmanager"]
```

---

## ⚙️ docker-compose.yml

```yaml
version: "3.9"

services:
  sdkmanager:
    build:
      context: .
      dockerfile: Dockerfile.sdkmanager
      args:
        - USERNAME=${USER}
        - UID=${UID}
        - GID=${GID}
        - UBUNTU_TAG=20.04
    image: nvidia-sdkmanager:20.04
    container_name: sdkmanager
    environment:
      - DISPLAY=${DISPLAY}
      - XAUTHORITY=/home/${USER}/.Xauthority
      - TZ=${TZ}
      - UID=${UID}
      - GID=${GID}
    # 保持 root 權限的掛載
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - ${XAUTHORITY:-$HOME/.Xauthority}:/home/${USER}/.Xauthority:ro
      - /etc/localtime:/etc/localtime:ro
      # 一般使用者可讀寫的掛載
      - sdkm-cache:/home/${USER}/.nvsdkm
      - ${HOME}/Downloads:/home/${USER}/Downloads
      - ${HOME}/nvidia:/home/${USER}/nvidia
    network_mode: host
    ipc: host
    privileged: true
    devices:
      - "/dev/bus/usb:/dev/bus/usb"

volumes:
  sdkm-cache:
```

---

## 📝 .env 範例

```bash
USER=$(id -un)
UID=$(id -u)
GID=$(id -g)
DISPLAY=${DISPLAY}
XAUTHORITY=${XAUTHORITY:-$HOME/.Xauthority}
TZ=Asia/Taipei
```

---

## 🔑 entrypoint.sh

```bash
#!/bin/bash
set -e

# 調整需要一般使用者的目錄權限
chown -R ${UID}:${GID} /home/${USERNAME}/.nvsdkm \
                       /home/${USERNAME}/Downloads \
                       /home/${USERNAME}/nvidia \
                       /workspace

# 切換成一般使用者執行
exec sudo -u ${USERNAME} "$@"
```

---

## 📖 檔案用途

- **Dockerfile.sdkmanager**
    - 建立基於 Ubuntu 20.04 的 SDK Manager 環境，並建立與主機相同的非 root 使用者。
    - 安裝 `x11-apps` 以測試 X11 轉發。
- **docker-compose.yml**
    - 定義容器服務與掛載路徑，分別處理需要 root 與一般使用者的掛載。
- **.env**
    - 儲存主機使用者名稱、UID、GID、時區與 X11 相關環境變數，方便跨平台啟動。
- **entrypoint.sh**
    - 在容器啟動時修正目錄權限，並切換成一般使用者執行 SDK Manager。

---

## 🚀 使用方法

1. **準備環境變數**
    
    ```bash
    export USER=$(id -un)
    export UID=$(id -u)
    export GID=$(id -g)
    export DISPLAY=${DISPLAY:-:0}
    export XAUTHORITY=${XAUTHORITY:-$HOME/.Xauthority}
    export TZ=Asia/Taipei
    ```
    
2. **允許 X11 存取**
    
    ```bash
    xhost +SI:localuser:$USER
    ```
    
3. **建立必要目錄**
    
    ```bash
    mkdir -p ~/.nvsdkm ~/Downloads ~/nvidia
    ```
    
4. **建置映像**
    
    ```bash
    docker compose build
    ```
    
5. **啟動 GUI 模式**
    
    ```bash
    docker compose up
    ```
    
6. **測試 X11**
    
    ```bash
    docker exec -it sdkmanager xeyes
    ```
    
7. **CLI 模式 (安裝舊版 SDK)**
    
    ```bash
    docker compose run --rm sdkmanager sdkmanager --archived-versions
    ```
    

---

## ⚠️ 注意事項

- **X11 轉發**
    - 必須在主機允許對應使用者訪問 X Server（`xhost` 設定）。
- **權限問題**
    - 若 volume 初次建立時為 root 擁有，需要 `entrypoint.sh` 調整權限。
- **USB 裝置刷機**
    - 保留 `/dev/bus/usb` 掛載與 `privileged: true` 以支援 Jetson 相關刷機流程。
- **時區同步**
    - `TZ` 參數與 `/etc/localtime` 掛載確保容器內外顯示一致時間。
- **資料持久化**
    - `sdkm-cache` 保存 SDK Manager 快取，`Downloads` 與 `nvidia` 用於交換檔案。

---

這份配置已經兼顧：

- ✅ 以一般使用者身份操作，避免 root 權限污染檔案
- ✅ 分離需要 root 與一般使用者權限的掛載
- ✅ 完整支援 X11 GUI 與 CLI 模式
- ✅ 支援 USB 刷機與舊版 SDK 安裝
