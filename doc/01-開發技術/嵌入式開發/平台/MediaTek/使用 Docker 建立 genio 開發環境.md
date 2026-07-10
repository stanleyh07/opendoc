
## 一、專案目標

1. 以 Ubuntu 22.04 為基底，撰寫 Dockerfile，建立 MediaTek Genio 系列產品的 Yocto 開發環境映像。
    
2. 容器啟動時能支援韌體燒錄所需的硬體裝置映射（USB、UART、GPIO 等），並讓非 root 使用者可正常執行 `genio-flash`。
    

---

## 二、最初範例 Dockerfile 架構

- **安裝系統套件**：Yocto build（gawk、wget、git、diffstat…）、燒錄工具（android-tools-adb/fastboot）、Python 及相關套件。
    
- **安裝 `repo`**：下載官方 `repo` 到 `/root/bin`，並軟連結至 `/usr/local/bin`。
    
- **安裝 Genio Tools**：`pip3 install genio-tools`，內含 `genio-flash`、`genio-config` 等工具。
    
- **設定 udev 規則**（原先在 Dockerfile 內用 `echo … > /etc/udev/rules.d/` 寫入兩份規則檔：
    
    - `72-aiot.rules`：針對 Genio USB（0e8d:201c/0003）、FTDI UART（0403）及 GPIO，MODE=0660、TAG+=uaccess。        
    - `96-rity.rules`：針對 Genio USB（0e8d:201c），MODE=0660、GROUP=plugdev。
        
- **建立使用者**：新增 GID 與 Host 相同的 `plugdev`、`dialout` 群組，並建立 `dev` 使用者加入這兩組。
    
- **預設啟動**：以 `dev` 身份進入 `/home/dev/workspace`。


**72-aiot.rules 範例**
```
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", ATTR{idProduct}=="201c", MODE="0660", TAG+="uaccess"
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", ATTR{idProduct}=="0003", MODE="0660", TAG+="uaccess"
SUBSYSTEM=="usb", ATTR{idVendor}=="0403", MODE="0660", TAG+="uaccess"
SUBSYSTEM=="gpio", MODE="0660", TAG+="uaccess"

```

**96-rity.rules 範例**
```
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", ATTR{idProduct}=="201c", MODE="0660", $ GROUP="plugdev"
```


---
## 三、udev 規則與 `genio-flash` 的互動機制

1. **Kernel → udevd → libudev/pyudev → genio-flash**
    
    - Linux kernel 收到 USB 插拔後發送 netlink 事件；
        
    - Host 的 `udevd` 根據規則更新 `/dev`，並轉發事件給註冊的 libudev client（包含容器內的 pyudev）；
        
    - `genio-flash` 在啟動時透過 pyudev 註冊 callback，等待對應事件觸發後開始燒錄。
        
2. **在容器內只映射 `/dev/bus/usb`，無法收到熱插拔事件**
    
    - 容器沒有啟動 udevd，host 的 udevd 雖建好節點，但不會把 hotplug netlink 事件傳到容器的 netlink namespace，導致 `genio-flash` 卡在「waiting for device」。
        

---

## 四、解決方案比較

| 方案                                 | 描述                                                                                                                        | 優點                      | 風險/缺點                        |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ----------------------- | ---------------------------- |
| **1. Host 管理 udev + netlink 事件映射** | `--net=host` 並掛載 `/run/udev/control:ro`，容器可直接接收 Host 的 netlink 與 udev control。<br>udev rules 設定放在 Host，容器只接收跟註冊不直接處理 udev | 即插即用，不需在容器啟動 udevd。     | 容器對 Host netlink 開放，安全隔離性較低。 |
| **2. 在容器啟動 udevd**                 | 安裝並執行 `systemd-udevd` 或 `/lib/systemd/systemd-udevd`，並 reload/trigger。                                                    | 完整自主的 udev 環境，不仰賴 Host。 | 容易與 Host 衝突，且需啟 systemd，複雜。  |
| **3. 先插裝置再啟動容器（跳過 hotplug）**       | 開發板插好後再 `docker run`，容器內直接掃描現有 `/dev/bus/usb`。                                                                            | 簡單易行，無需熱插拔監聽。           | 不支援運行中插拔自動燒錄場景。              |

**總結**

- 容器內若不啟動 `udevd`，就不會套用裡面的 udev 規則檔，但也不需擔心跟 host 干擾──這時候全部交由宿主機的 udev 管理，再把 `/dev` 用 `--device` 或 `-v` 掛過去就行。
    
- 若真的一定要在容器內跑 udev，務必注意 `--privileged` 的副作用，且最好在容器啟動後執行 `udevadm control --reload-rules && udevadm trigger` 來重讀規則。

---

## 五、最終推薦 Dockerfile

```dockerfile
# 1. 基礎映像及非互動安裝
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

# 2. 安裝 Yocto build 與 Flash 工具
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      gawk wget git diffstat unzip texinfo \
      gcc build-essential chrpath socat cpio \
      python3 python3-pip python3-pexpect xz-utils \
      debianutils iputils-ping python3-git python3-jinja2 \
      libegl1-mesa libelf-dev libsdl1.2-dev lz4 pylint \
      xterm python3-subunit mesa-common-dev libstdc++-12-dev \
      android-tools-adb android-tools-fastboot curl sudo && \
    rm -rf /var/lib/apt/lists/*

# 3. 安裝 repo 工具
RUN curl https://storage.googleapis.com/git-repo-downloads/repo \
      -o /usr/local/bin/repo && \
    chmod a+rx /usr/local/bin/repo

# 4. 安裝 Genio Tools
RUN pip3 install --no-cache-dir --upgrade pip && \
    pip3 install --no-cache-dir genio-tools

# 5. 建立與 Host 同 GID 的 plugdev (46) / dialout (20) 群組，並新增 dev 使用者
RUN groupadd -g 46 plugdev && \
    groupadd -g 20 dialout && \
    useradd -m -u 1000 -G plugdev,dialout dev && \
    echo "dev ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# 6. 切換到非 root 使用者
USER dev
ENV HOME=/home/dev PATH=/home/dev/bin:${PATH}
WORKDIR /home/dev/workspace

CMD ["/bin/bash"]
```

---

## 六、建構與執行指令

1. **建構映像**
    
    ```bash
    docker build -t mediatek-genio-dev:latest .
    ```
    
2. **執行容器＋支援即插即燒（推薦）**
   genio-flash 會嘗試存取 /dev/ttyACM0 與 /dev/ttyUSB0, 但 ttyACM0 與 ttyUSB0 可能不存在或是還未建立, 因此需要將整個 /dev map 進去
    
    ```bash
    docker run -it --rm \
      --privileged \
      --net=host \
      -v /run/udev/control:/run/udev/control:ro \
      -v /dev/bus/usb:/dev/bus/usb \
      -v /dev:/dev \
      -v $(pwd)/workspace:/home/dev/workspace \
      mediatek-genio-dev:latest
    ```
    
3. **執行容器＋先插再跑（跳過 hotplug）**
    
    ```bash
    docker run -it --rm \
      --privileged \
      -v /dev/bus/usb:/dev/bus/usb \
      --device /dev/ttyUSB0 \
      --device /dev/ttyACM0 \
      -v $(pwd)/workspace:/home/dev/workspace \
      mediatek-genio-dev:latest
    ```
    


---

### 七、注意事項與風險

- **`--privileged`** 會提升容器對 Host 的控制權限，請於可信環境使用。
    
- **`--net=host`** 與掛載 `/run/udev/control` 讓容器可接收 Host udev 事件，但會降低網路隔離。
  為了讓容器內的 `genio-flash` 能接收到 **USB 插入事件**（透過 `pyudev` → `libudev` → `netlink`），**必須使用 `--net=host`**。但這個設定 **同時會讓容器共享 host 的整個網路堆疊**，例如：
	- 容器會與 host 使用 **相同的 IP 位址**
	- 容器內開啟的任何 port（像是 22, 80, 443）會**直接綁到 host**，無法隔離
	- 容器可能能掃描 host 網路上其他介面（安全性較弱）。
	- 群組 GID (plugdev/dialout) 必須與 Host 同步，否則容器內使用者可能無法存取對應裝置。
	- 若要完全隔離，可考慮在容器內啟動 udevd，但需額外設定，且可能容器與Host可能互相干擾，風險較高。
	- 注意因為容器是接收 Host udev 事件去處理, 因為 Host 端需要設定 `72-aiot.rules` 與 `96-rity.rules`,同時也要安裝 `android-tools-adb` `android-tools-fastboot` 來讓 Host 可以辨認到裝置，並且觸發建立將相關的行為
  
- 使用 -v 去 map 整個目錄，使用 --device 去 map 單一個裝置，當需要如 /dev/bus/usb 或是 /dev 這樣整個目錄時, 就要使用 -v
  
- 如果你只想給容器存取權限，但不想給太多 host 能力，可以改用 `--cap-add` 只加必要 capability，例如：
``` bash
--cap-add=SYS_ADMIN \
--cap-add=MKNOD
```

