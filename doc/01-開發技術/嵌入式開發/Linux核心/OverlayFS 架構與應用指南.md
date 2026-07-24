---
title: OverlayFS 架構與應用指南
tags:
  - linux
  - overlayfs
  - embedded
  - ota
  - rootfs
  - version-control
  - multi-layer
created: 2026-07-10
modified: 2026-07-23
aliases:
  - OverlayFS rootfs
  - OTA RootFS 架構
  - OverlayFS 版本管理
  - 多層 OverlayFS
---

# OverlayFS 架構與應用指南

本文件涵蓋 OverlayFS 在嵌入式 Linux 中的三大應用場景：**兩層 OTA 系統更新架構**、**多層 OTA 架構（RootFS + App + Data）** 與 **開發版控管理**。

---

## 第一部分：OverlayFS 核心概念

OverlayFS 的核心邏輯是將多個目錄疊加組合，向使用者呈現一個虛擬的合併目錄。

### 目錄角色

| 層級 | 角色 | 說明 |
|------|------|------|
| **`lowerdir`** | 底層（唯讀） | 乾淨的系統基底（Golden Image） |
| **`upperdir`** | 上層（可讀寫） | 所有新增、修改、刪除的變更 |
| **`workdir`** | 工作目錄 | 核心內部原子操作暫存區，須與 `upperdir` 同分割區 |
| **`merged`** | 合併層 | 最終呈現給系統的根目錄 |

---

## 第二部分：OTA 更新架構

利用 OverlayFS 與 `initramfs`，實現唯讀系統與用戶資料分離，確保 OTA 更新時保留使用者設定。

### 磁碟分割規劃

| 分割區 | 屬性 | 功用 |
|--------|------|------|
| **Boot** (`/dev/mmcblk0p1`) | 可讀寫 | Bootloader, Kernel, Device Tree, initramfs |
| **RootFS** (`/dev/mmcblk0p2`) | **唯讀** | OverlayFS 的 `lowerdir` |
| **Data** (`/dev/mmcblk0p3`) | **可讀寫** | OverlayFS 的 `upperdir` 與 `workdir` |

### initramfs 啟動腳本

```bash
#!/bin/sh

# 1. 掛載虛擬檔案系統
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

# 2. 建立掛載點
mkdir -p /mnt/lower /mnt/data /mnt/overlay

# 3. 掛載底層系統
mount -o ro /dev/mmcblk0p2 /mnt/lower
mount -o rw /dev/mmcblk0p3 /mnt/data

# 4. 建立 OverlayFS 目錄
mkdir -p /mnt/data/upper /mnt/data/work

# 5. 組裝 OverlayFS
mount -t overlay overlay -o lowerdir=/mnt/lower,upperdir=/mnt/data/upper,workdir=/mnt/data/work /mnt/overlay

# 6. 轉移虛擬檔案系統
mount --move /sys /mnt/overlay/sys
mount --move /proc /mnt/overlay/proc
mount --move /dev /mnt/overlay/dev

# 7. 切換根目錄
exec switch_root /mnt/overlay /sbin/init
```

### `mount --move` 與 `switch_root` 說明

- **保留狀態**：保留核心早期建立的裝置節點，避免硬體失明
- **釋放記憶體**：允許 `switch_root` 刪除舊的 initramfs
- **原子操作**：在核心掛載樹中瞬間改變指標位置

### OTA 更新流程

1. 背景下載全新 RootFS 映像檔
2. 直接寫入 RootFS 分割區（或切換 A/B Partition）
3. 重新開機
4. initramfs 掛載新系統層，疊加既有 Data 分割區，完成無痛升級

---

## 第三部分：多層 OTA 架構（RootFS + App + Data）

當系統需要將 **基礎系統、應用程式、使用者資料** 三者完全分離，並各自獨立更新或重置時，可利用 OverlayFS 支援多層 `lowerdir` 的特性，構建三層堆疊架構。

### 磁碟分割規劃

| 分割區 | 裝置路徑 | 屬性 | OverlayFS 角色 | 功用 |
|--------|----------|------|----------------|------|
| **Boot** | `/dev/mmcblk0p1` | 可讀寫 | — | Bootloader、Kernel、Device Tree、initramfs |
| **RootFS** | `/dev/mmcblk0p2` | **唯讀** | `lowerdir`（底層） | 基礎系統 Golden Image |
| **App** | `/dev/mmcblk0p3` | **唯讀** | `lowerdir`（中層） | 應用程式，獨立 OTA 更新 |
| **Data** | `/dev/mmcblk0p4` | **可讀寫** | `upperdir` + `workdir` | 使用者資料、runtime 變更 |

> [!info] 多層 lowerdir 優先級
> `lowerdir=/mnt/app:/mnt/rootfs` 中，**左側優先級高**：當 App 與 RootFS 有同路徑檔案時，App 層的版本會覆蓋 RootFS 層。

### initramfs 啟動腳本

```bash
#!/bin/sh

# 1. 掛載虛擬檔案系統
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

# 2. 建立掛載點
mkdir -p /mnt/rootfs /mnt/app /mnt/data /mnt/merged

# 3. 掛載各分割區
mount -o ro /dev/mmcblk0p2 /mnt/rootfs    # 底層：base rootfs（唯讀）
mount -o ro /dev/mmcblk0p3 /mnt/app        # 中層：app（唯讀）
mount -o rw /dev/mmcblk0p4 /mnt/data       # 頂層：data（可讀寫）

# 4. 建立 OverlayFS 工作目錄
mkdir -p /mnt/data/upper /mnt/data/work

# 5. 組裝多層 OverlayFS
#    lowerdir = app（中層）: rootfs（底層），左側優先級高
#    upperdir = data/upper，所有寫入進入 Data 分割區
mount -t overlay overlay \
  -o lowerdir=/mnt/app:/mnt/rootfs,upperdir=/mnt/data/upper,workdir=/mnt/data/work \
  /mnt/merged

# 6. 轉移虛擬檔案系統
mount --move /sys  /mnt/merged/sys
mount --move /proc /mnt/merged/proc
mount --move /dev  /mnt/merged/dev

# 7. 切換根目錄
exec switch_root /mnt/merged /sbin/init
```

### 四大情境運作機制

#### 情境一：OTA 更新 RootFS

| 項目 | 說明 |
|------|------|
| 操作 | 直接 dd 或 flash `/dev/mmcblk0p2`（RootFS 分割區） |
| 影響範圍 | 僅替換最底層 lowerdir |
| App / Data | 不動。重開機後 OverlayFS 重新堆疊，App 檔案繼續覆蓋 RootFS 同名路徑，Data 保留所有 runtime 變更 |
| 注意事項 | 新版 RootFS 不得移除 App 依賴的底層 library，否則 App 層無法補救 |

#### 情境二：Runtime 變更（正常開機使用）

| 項目 | 說明 |
|------|------|
| 行為 | 所有寫入操作（修改設定、新增檔案、安裝套件等）自動進入 `upperdir`（Data 分割區） |
| RootFS / App | 兩層 lowerdir 均為唯讀，完全不受影響 |
| 機制 | OverlayFS 原生行為，無需額外處理 |

#### 情境三：Factory Reset

| 項目 | 說明 |
|------|------|
| 操作 | 刪除 Data 分割區的 `upper/` 目錄內容，或整顆格式化 |
| 指令範例 | `rm -rf /upper/*` 或 `mkfs.ext4 /dev/mmcblk0p4` |
| 結果 | 所有 runtime 變更消失，系統回到 RootFS + App 的初始堆疊狀態 |
| RootFS / App | 完全不受影響 |

> [!tip] 預設初始狀態
> 若需要 Factory Reset 後保留預設設定值，可將預設組態放在 RootFS 的 `/etc/defaults/`，由 init 腳本在首次啟動或 Reset 後複製到 `upperdir`。

#### 情境四：App 獨立更新

| 項目 | 說明 |
|------|------|
| 操作 | 直接 dd 或 flash `/dev/mmcblk0p3`（App 分割區） |
| 影響範圍 | 僅替換中層 lowerdir |
| RootFS / Data | 不動。重開機後新 App 版本覆蓋 RootFS 對應路徑，Data 保留 |
| 注意事項 | 與 RootFS OTA 同理，需確保新版 App 與 RootFS 的介面相容 |

### 四大情境對照總覽

| 情境 | RootFS | App | Data | 操作方式 |
|------|--------|-----|------|----------|
| OTA 更新 RootFS | ✅ 替換 | 不動 | 不動 | dd / flash rootfs 分割區 |
| Runtime 變更 | 不動 | 不動 | ✅ 自動寫入 | 正常開機使用 |
| Factory Reset | 不動 | 不動 | ✅ 清除 | 刪除 upper 或格式化 Data |
| OTA 更新 App | 不動 | ✅ 替換 | 不動 | dd / flash app 分割區 |

### 設計注意事項

1. **App 分割區的目錄結構**：App 分割區掛載後的目錄樹必須對應到 rootfs 的目標路徑（如 `/opt/app`）。若 App 檔案散落在 `/usr/bin`、`/etc` 等多處，需在打包時確保完整的目錄結構。

2. **Data 分割區容量規劃**：所有 runtime 新增檔案都寫入 `upperdir`，需根據使用場景預留足夠空間。

3. **RootFS 與 App 的介面穩定性**：App 層無法補救 RootFS 中被移除的底層依賴（如 shared library）。兩者的介面需保持向後相容。

4. **App 更新 Rollback**：若新 App 版本有問題，可考慮 A/B 分割區方案（`/dev/mmcblk0p3a` / `/dev/mmcblk0p3b`），在 initramfs 中選擇啟用哪個 App 分割區。

5. **首次啟動處理**：可透過檢測 Data 分割區是否存在標記檔（如 `/upper/.initialized`），判斷是否為首次啟動，據此執行初始化動作。

---

## 第四部分：開發版控管理

使用 OverlayFS 實現輕量化的 rootfs 版本控制，替代 Yocto/Buildroot 重量級方案。

### 專案目錄結構

```
my_project_repo/            # git init 在此
├── .gitignore
├── build_env.sh            # 一鍵掛載/解掛載腳本
├── rootfs_base/            # 唯讀底層 (Lower)
├── work/                   # 工作目錄 (被 Git 忽略)
├── merged/                 # 合併層 (被 Git 忽略)
└── overlay/                # 變更層 (Git 追蹤)
    ├── etc/network/interfaces
    └── usr/bin/my_app
```

### `.gitignore`

```
/work/
/merged/
*.tar.gz
```

### 開發流程

```bash
# 掛載
BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
mkdir -p "${BASE_DIR}/work" "${BASE_DIR}/merged"
sudo mount -t overlay overlay \
  -o lowerdir="${BASE_DIR}/rootfs_base",upperdir="${BASE_DIR}/overlay",workdir="${BASE_DIR}/work" \
  "${BASE_DIR}/merged"

# 開發完成後提交
sync
sudo umount -l "${BASE_DIR}/merged"
cd "${BASE_DIR}/overlay"
git add .
git commit -m "feat: update configuration"
```

### 部署同步（rsync 黃金參數）

```bash
sudo rsync -avxHAX --progress /path/to/merged/ /path/to/target_deploy_dir/
```

| 參數 | 作用 |
|------|------|
| `-a` | 封存模式，保留權限、符號連結、裝置檔案 |
| `-x` | 不跨越檔案系統邊界，避免誤同步 `/proc` `/sys` |
| `-H` | 保留硬連結（BusyBox 等依賴硬連結節省空間） |
| `-A` | 保留 ACL |
| `-X` | 保留擴充屬性（SELinux labels, capabilities） |
