以下是根據 NVIDIA 官方開發者指南整理的 **在 Jetson Thor 平台製作 USB 開機磁碟的完整步驟**，我已用 Markdown 格式排版，方便操作與閱讀。

---

# 如何製作 Jetson Thor 平台的 USB 開機磁碟

## 📋 前置準備

- 一台主機電腦（Linux，建議 Debian/Ubuntu 系統）
- 高品質 USB-C 或 micro-USB 線材（連接主機與 Jetson Thor）
- 至少 **64 GB** 的 USB 隨身碟（若使用 32 GB，需修改分區設定）
- 已下載並解壓縮的 **Linux_for_Tegra BSP 套件**
- 必要目錄：
    - `bootloader`：包含 TegraFlash、CFG、BCT 等工具
    - `kernel`：包含 `Image`、DTB、kernel modules
    - `rootfs`：下載的根檔系統，需先填入 sample rootfs
    - `nv_tegra`：使用者空間程式與範例

安裝依賴套件：

```bash
cd Linux_for_Tegra
sudo tools/l4t_flash_prerequisites.sh
```

---

## 🔌 連接與進入 Recovery 模式

1. 使用 USB 線連接主機與 Jetson Thor 的 **Recovery Port**。
2. 開機並按住 **RECOVERY** 按鈕，再按下 **RESET** 按鈕，使裝置進入 **Force Recovery Mode**。

---

## ⚙️ 建立 USB 開機磁碟

1. 確認 USB 隨身碟的裝置名稱：
    
    ```bash
    sudo lsblk -p -d | grep sd
    ```
    
    假設結果為 `/dev/sdb`。
    
2. 執行 initrd flash 指令：
    
    ```bash
    sudo ./l4t_initrd_flash.sh \
      -c tools/kernel_flash/flash_l4t_t264_nvme.xml \
      --external-device sda1 \
      --direct sdb \
      jetson-agx-thor-devkit external
    ```
    
    - `-c` 指定 USB 分區配置檔
    - `--external-device sda1` 指定 USB 根檔系統分區
    - `--direct sdb` 指定主機偵測到的 USB 裝置名稱
    - `jetson-agx-thor-devkit` 為目標板配置
3. 若 USB 為 **32 GB**，需修改分區設定：
    
    - 編輯 `flash_l4t_t264_nvme.xml`，將 `num_sectors * 512 = 32 GiB`
    - 指令需加上 `-S 20GiB`：
        
        ```bash
        sudo ./l4t_initrd_flash.sh \
          -c tools/kernel_flash/flash_l4t_t264_nvme.xml \
          -S 20GiB \
          --external-device sda1 \
          --direct sdb \
          jetson-agx-thor-devkit external
        ```
        

---

## 🚀 啟動 Jetson Thor

1. 將 USB 隨身碟插入 Jetson Thor。
2. 開機或重新啟動。
3. 若仍從內部儲存開機，需修改 **UEFI boot order**：
    - 使用 `efibootmgr` 或 UEFI GUI 介面設定。

---

## ✅ 成功驗證

- 開機後進入 UEFI 選單，確認 USB 裝置可被選為開機來源。
- 系統成功載入 rootfs，即完成 USB 開機磁碟製作。

---

這份流程涵蓋了 **完整可操作的步驟**，並包含了針對不同容量 USB 的調整方式。要不要我幫你再整理一份「快速命令清單」版本，讓你在操作時可以直接複製貼上？